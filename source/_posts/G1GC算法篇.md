---
title: G1GC算法篇
index_img:
date: 2022-12-20 22:15:48
tags: 
- GC
- Java
---
# 前言

本文将通过对G1的整个回收周期的算法原理进行解读，如有偏差，请指正。<!--more-->

G1GC(Garbage First Garbage Collection)(简称G1) 自 JDK9 起，替代 Parallel Scavenge 和 Parallel Old 的组合，作为 Hotspot 在 Server 端的默认垃圾回收器，具有停顿时间可控、不产生内存碎片、返还未使用内存给堆等的特性，在 JDK9 之前可以通过 -XX:+UseG1G C的参数调用 G1 。

# 一些基本概念

在深入 G1GC 算法之前，先建立以下基本概念，如果读者已经掌握相关知识可以跳过这一部分，如果完全不了解这些概念，可以尝试简单了解一部分。

## 自动内存管理系统

在《 Java 虚拟机规范》中将 Java 堆描述为：“ The heap is the runtime data area form which memory for all class instances and arrays is allocated ” （随着 Java 语言的发展，“所有对象实例及数组都在堆上分配”这一点也不绝对）。Java 堆作为自动内存管理的内存区域，需要满足能在上面为新对象分配内存，并自动释放死亡对象所占据内存的功能，细分下就是需要回答下面三个问题：
- 如何在堆上为新对象分配内存？
- 如何识别存活对象？
- 如何回收死亡对象占据的内存？

对应到 G1GC 的算法，将从下面几点回答这三个问题：

- G1GC 堆结构（分配对象）
- G1GC 并发标记（识别存活对象）
- G1GC 跨界引用的标记（识别存活对象）
- G1GC 转移对象过程（回收死亡对象）

## 软实时性

G1GC 具有软实时性（ soft real-time ）。由于多数 GC 需要暂停应用程序，为了保证应用程序的软实时性（能保证大多数任务在最后期限之前完成，例如网络银行系统），我们要求 G1 ，在任意 1 秒的时间内，停顿不得超过 200ms 。G1 会尽量达成这个目标，它能够推算出本次要收集的大体区域，以增量的方式完成收集。

- 设置`期望暂停时间`（ -XX:MaxGCPauseMillis ，默认 200）

## GC Roots
GC 时判断哪些对象需要被回收有两种方法，引用计数法和可达性分析法。其中，G1 使用可达性分析算法通过一系列 `GC Roots` 作为起始点搜索所有通过引用可达的对象，搜索的路径称为引用链（ Reference Chain ），所有能被搜索到的对象被认为（标记为）存活对象。可作为 GC Roots 的对象包括但不限于下面几种：

- 在虚拟机栈（栈帧中的本地变量表）中引用的对象
- 在方法区中类静态属性引用的对象
- 在方法区中常量引用的对象
- 本地方法栈中 JNI（ Native 方法）引用的对象

下面是一段Java示例
```java
Obj a = new Obj();// a 为 root ，对象的引用
a.ref = new Obj(); // a 的成员ref赋值变量的引用，当前可达
//System.gc();//如果此时发生 GC ，标记过程会从 a（ GC Root ）出发标记 a 指向的对象，再标记 a.ref 指向的对象
a = null;  //a 指向的对象被回收，a.ref 指向的对象也需要被回收
```

## 三色标记法
顾名思义，通过三种颜色完成标记，
- 白色：表示对象尚未被垃圾收集器标记过。
- 黑色：表示对象已经被垃圾收集器标记过，且这个对象的所有引用都已经扫描标记过。
- 灰色：表示对象已经被垃圾收集器访问过，但这个对象上至少存在一个引用还没有被扫描过，可以理解为正在搜索的对象。

简述三色标记法的遍历过程：

1. 初始时，全部对象都是白色的
2. GC Roots 直接引用的对象变成灰色
3. 从灰色集合中获取元素：
    - 将本对象直接引用的对象标记为灰色
    - 将本对象标记为黑色
4. 重复步骤3，直到灰色的对象集合变为空
5. 结束后，仍然被标记为白色的对象就是不可达对象，视为垃圾对象

对象在并发标记阶段会被漏标的充分必要条件是（同时满足以下两点）：

- Mutator 应用程序插入了一个从黑色对象到该白色对象的新引用
- Mutator 应用程序删除了所有从灰色对象到该白色对象的直接或间接引用

<p id="Copying"></p>   

## Copying —— Cheney

当标记完成后需要将存活对象转移到特定区域中， G1 的转移逻辑都是基于 Copying 算法思想实现的，算法简略过程如下：

![Copying 算法](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/copy.gif)

# G1GC 的堆结构

G1GC 堆的内部被划分为大小相等的 `Region` ，G1GC 以 Region 为单位进行 GC ，Region 的大小 `RegionSize` 在 JVM 初始化时初始完毕且在当前 java 进程结束前不会改变。在 HotSpot 的源码中 `TARGET_REGION_NUMBER` 定义了 Region 的数量限制为 2048 个（实际上允许超过这个值）。一般 RegionSize 等于堆空间的总大小除以 2048 ，也可以用参数 -XX:G1HeapRegionSize 强制指定每个 Region 区的大小用户可以设置 Region 大小，但需要满足 RegionSize 是向上调整为 2 的指数幂，同时需要保证最小不小于 1MB ，最大不超过 32MB （集合 { 1MB , 2MB , 4MB , 8MB , 16MB , 32MB } 内）。

比如目前的堆空间总大小为 8.5GB ，RegionSize 就是 8704MB/2048 = 4.25MB ，那么最终每个 Region 的大小为 8MB 。当一个 region 剩余空间不足以满足下一个对象所需空间时， Hotspot 直接放弃 region 中剩余空间，在新分配的 region 中分配对象； OpenJ9 利用剩下的空间，让对象跨 region 存放。

## G1GC 分代分区

在 G1 收集器中逻辑依旧是分代的，依旧存在年轻代 `Eden` 区、幸存区 `Survivor` 、老年代 `Old` 区，但在物理内存上是不分代的。G1 的每个代区由物理内存不连续的 Region 集合构成，这样做的好处在于：G1 可以优先回收垃圾对象特别多的 Region 区，这样可以花费较少的时间来回收垃圾，这也就是 G1 名字的由来，即垃圾优先收集器。
在运行时，G1 会将堆空间变为如下结构：

<center>
<img src="/img/G1GC算法篇/G1堆.png" style="zoom:35%" title="分代 G1GC 堆空间划分" >
<br>
    <div style="color:orange; 
    color: #999;
    padding: 2px;">分代 G1GC 堆空间划分
    </div>
</center>

新创建的对象都会被分配到 Eden 区，对象经过第一次 YGC 后，仍然存活的会被移到 Survivor 区，多次GC后依然存活的对象会被移动到 Old 区。图中除了提到的三个分区还多了大对象区 `Humongous` 与 `未分配区` 。在 G1 中当对象大小超过单个普通 Region 区的 50% 时，认定对象为大对象，大对象会直接将其放入 Humongous 区存储，当一个 Humongous 区存不下时，可能会横跨多个 Region 区存储它。在大多数时 Humongous 区会被当做老年代看待。

## G1GC 新生代动态调整大小

G1 的设计目标是软实时，应以响应时间优先。 G1 根据基于衰减平均值的停顿预测模型统计以前发生 GC 时记录的数据来预测本次收集需要选择的分区数量。以新生代收集为例（通过添加参数 -XX:+PrintGCDetails 查看）

![新生代动态调整大小](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/%E6%96%B0%E7%94%9F%E4%BB%A3%E5%A4%A7%E5%B0%8F.png)

如上图彩色框图所示， G1 随着 GC 的发生，根据 GC 耗时动态调整新生代 Eden 区和 Survivor 区的个数，当Eden区满后触发下一次 GC 收集。通过这种方式来保证 GC 带来的最大停顿时间不超过期望暂停时间。当然上图展示的是比较理想的 YGC 过程，G1GC 的收集策略会在后续详细介绍。

# 并发标记
通常 GC 算法在标记存活对象时，为了避免在标记过程中发生引用关系改变的情况，都需要让应用程序在安全点处停下，也就是常说的`Stop The World`简称 STW 。G1 在对老年代的标记过程中引入了并发标记，并发指的是与应用程序`mutator`并发执行。并发标记并不是直接在对象上添加标记，而是在`标记位图`上添加标记。

## 标记位图

下图表示堆中的一个 region ，位图中黑色表示已标记存活，白色表示未标记为存活。

<center>
<img src="/img/G1GC算法篇/标记位图.png" style="zoom:45%" title="标记位图" >
<br>
    <div style="color:orange; 
    color: #999;
    padding: 2px;">标记位图
    </div>
</center>

需要强调下，图中为了展示对应关系，将标记位图画得同 region 相对应，实际上标记位图每一格只有 1bit ，而 region 上一块为单个对象，标记位图中的每个 bit 都对应关联 region 内的对象的开头部分。假设单个对象的大小都是 8 个字节（本文后续都如此假定），那么每 8 个字节就会对应标记位图中的 1个bit 。图中标记位图里黑色的地方表示比特值是 1 ，白色的地方表示比特值是 0 。相应地，region 内黑色的是存活对象，带有叉号的是死亡对象。

每个region有两个标记位图：
- `next`：本次标记的标记位图。
- `prev`：上次标记的标记位图，保存了上次并发标记的结果。

图中region部分：

- `bottom`：region 内众多对象的末尾
- `top`：region 内众多对象的开头（包括在并发标记过程中新分配的对象）
- `nextTAMS`：本次标记开始时的 top（TAMS-Top At Marking Start）（本次标记从 bottom 到此处）
- `prevTAMS`：上次标记开始时的 top

并发标记结束主要针对的是 G1 老年代中 region 部分的标记，根据 G1 的清理策略，并不会对所有 region 做垃圾收集（会选择垃圾最多的几个 region 作清理）。所以保存上次标记结果 prev ，用于确定上次标记的存活对象在本次标记后是否能存活，而不扫描上次已经确定死亡的对象。

下列图展示对同一块 region 两次并发标记始末状态，用来说明 prev 标记位图和 next 标记位图的作用

<center class = half>
<img src="/img/G1GC算法篇/第一次并发标记开始.png" height="143" title="第一次并发标记开始状态" >
<img src="/img/G1GC算法篇/第一次并发标记完成.png" height="143" title="第一次并发标记完成状态" >
<br>
    <div style="color:orange; color: #999;padding: 2px;">
    第一次并发标记开始状态&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;第一次并发标记完成状态
    </div>
</center>

如上图所见，到第一次并发标记完成，标记了所有 bottom 到 nextTAMS 内所有存活对象（标记完成后会在收尾阶段将标记位图 next 中的并发标记结果移动到标记位图 prev 中，再重置标记位图 next ）

<center class = half>
<img src="/img/G1GC算法篇/第二次并发标记开始.png" height="153" title="第二次并发标记开始状态" >
<img src="/img/G1GC算法篇/第二次并发标记完成.png" height="153" title="第二次并发标记完成状态" >
<br>
    <div style="color:orange; color: #999;padding: 2px;">
    第二次并发标记开始状态&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;第二次并发标记完成状态
    </div>
</center>

等到新一轮并发标记开始时时，

1. 对当前 Bottom 和 top 之间的对象创建空白的标记位图 next
2. 对 Bottom 和 prevTAMS 之间的内存对象，根据 prev 标记的存活结果重新标记存活对象（ prev 中标记死亡的对象不扫描）
3. 对 prevTAMS 和 NextTAMS 之间的内存对象进行存活标记
4. 对 NextTAMS 和 top 之间的对象，都认为是存活对象，不在标记位图上作标记
5. 完成并发标记后将标记位图 next 置为标记位图 prev ，并重置标记位图 next

注：在下文展开并发标记算法逻辑时，为了简化理解，假定region上发生的是第一次并发标记，prev为空白（如上图）。

## 执行步骤

简单介绍单次并发标记的过程，具体标记细节，会在下文展开，参考[中村成洋书中内容](#references)一共分下列五个步骤

1. **初始标记阶段**：暂停应用程序 STW ，标记可由根直接引用的对象。
2. **并发标记阶段**：与 mutator 并发进行，扫描 1 中标记的对象所引用的对象。
3. **最终标记阶段**：暂停应用程序 STW ，扫描 2 中没有标记的对象。本步骤结束后，堆内所有存活对象都会被标记。
4. **存活对象计数**：对每个 region 中被标记的对象进行计数，并发执行。
5. **收尾工作**：暂停应用程序 STW ，收尾工作，并为下次标记做准备。

## 步骤 1——初始标记阶段

![初始标记](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/%E5%88%9D%E5%A7%8B%E6%A0%87%E8%AE%B0.png)

在初始标记阶段，GC 线程首先创建空白的标记位图 next。其中 nextTAMS 是本次标记开始时 top 所在的位置。单个对象假定8字节，位图的大小是 (top-botton)/(8字节)。创建过程与 mutator 应用程序并发进行。

等所有待回收 region 的标记位图都完成创建后，暂停 mutator（ STW ）标记由根直接引用的对象（根扫描），目的是防止扫描过程中根被修改。按照三色标记算法，此时根直接引用的对象的引用还没被扫描，应该将根直接引用的对象标记为灰色，需要注意的是标记位图只能通过 0/1 的 bit 位来标记对象是否存活，并不能标记灰色，真实的位图在初始标记结束时会将存活对象在位图上标为1，而如果表现标灰的操作会在下文 SATB 部分展开。

上图状态可以用如下java代码粗略演示

```java
Obj C = new Obj();
C.ref1 = new Aobj();
C.ref2 = new Eobj();

Obj H = new Obj();
H.ref1 = new Iobj();
```
根对象 C 、 H 分别持有子对象，根对象被标记为灰色，因为持有的子对象都没被标记（白色）

## 步骤 2——并发标记阶段

在并发标记阶段，GC 线程与 mutator 并发进行，扫描在 1 阶段标记过的对象，完成对大部分存活对象的标记。

![并发标记阶段](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/%E5%B9%B6%E5%8F%91%E6%A0%87%E8%AE%B0%E9%98%B6%E6%AE%B5.png)

上图展示了并发标记的始末状态，对象 C 的子对象 A 和 E ，对象 H 的子对象 I 都被标记了。虽然 E 对应了标记位图中多个位，但是只有起始的标记位会被标记为 1 。

因为并发标记是和 mutator 应用程序并发执行，所以可能会向 region 内分配新的对象，上图中 J 和 K 即在并发标记阶段新分配的对象，记录在初始标记阶段记录的 nextTAMS 与当前 top 之间（ top 会随着分配对象而移动，指针碰撞），并不需要专门为新生成的对象创建标记位图（ J 、K 没有标记位图），这些新分配的对象（nextTAMS到top间的对象）会被当做存活对象，不在此次标记收集中处理。

但是由于这一阶段是与应用程序 mutator 并发执行， mutator 可能会改变对象之间的引用关系，造成并发标记漏标。

### SATB

在三色标记算法的基本概念中给出了并发标记漏标的充分必要条件，此处通过例子展示漏标的情况：

![](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/%E6%BC%8F%E6%A0%87.gif)

开始时，A 、C 、E 完成标记，H 由于是根直接引用的对象，在初始标记阶段标记为灰色，I 暂未被标记，然后 Mutator 执行下列代码

```java
//已知:H.ref1 = new Iobj();C.ref2 = new Eobj();
C.ref2.ref1 = H.ref1;//添加引用E->I
H.ref1 = null;//断开H->I 
```

如果不做任何处理，黑色对象 E 不会再次扫描 E 的引用，又由于断开了所有由灰色对象到对象 A 的引用，对象 A 无法被标记为存活，将在回收过程中被当作垃圾收集，造成漏标。

G1GC 使用`SATB（Snapshot At The Beginning）专用写屏障`算法，用于打破充要条件的第二点。即在断开由灰色对象到白色对象引用时，将白色对象变为灰色对象。具体的操作是在一个对象的 field 发生写操作时，这个对象 field 之前的值会被放入 SATB 本地队列。

<p id="satb"></p> 

SATB 专用写屏障的伪代码如下所示：

```
//对应JVM 在oop_store方法的赋值动作的前的pre-write barrier
def satb_write_barrier(field, newobj):
    if $gc_phase == GC_CONCURRENT_MARK:
        oldobj = *field
        if oldobj != Null:
            enqueue($current_thread.stab_local_queue, oldobj)

        *field = newobj
```

参数 field 表示被写入对象的域，参数 newobj 表示被写入域的值。第 2 行的 GC_CONCURRENT_MARK 表示并发标记阶段的标志位（flag）用于检查当前是否处于并发标记阶段。第 4 行检查被写入之前field 域的值是不是 Null。如果检查通过，则在第 5 行将 oldobj 添加到 $current_thread.stab_local_queue 中。然后，在第 7 行进行实际的写入操作。

“初始标记阶段”一节中提到并不存在位图上将对象标记为灰色的操作，所谓的标记为灰色其实是，将对象对应的标记位图bit位设置为 1 的同时，将对象加入 SATB 队列。此处的队列是 mutator 各自持有的线程本地队列，SATB 本地队列在装满（默认大小为 1 KB）之后，会被添加到全局的 SATB 队列集合中，在并发标记阶段交给 GC 线程对队列中的全部对象进行扫描和标记。

回到刚刚的例子

```java
//已知:H.ref1 = new Iobj();C.ref2 = new Eobj();
C.ref2.ref1 = H.ref1;
//C.ref2.ref1 在 region 图中为对象 E ，此处发生写操作会被写屏障函数感知
//首先判断 E.ref1 = null 所以可以直接将 H.ref1 即 对象 I 写入 E 的域
H.ref1 = null;
//H.ref1 在region图中为对象 I ，此处发生写操作会被写屏障函数感知
//由于 I 非空 需要先将对象 I 在位图上标记为存活，再将对象 I 加入待扫描的 SATB 队列
```

不难发现，通过上述过程对象 I 并不会被漏标。

细心的读者会发现，如果不存在第一步将对象 I 写入 E 的域，单纯需要将对象 I 置为垃圾，那 SATB 写屏障的存在依旧会将对象 I 标记为存活造成错标，造成浮动垃圾，但是不影响程序的正确性。G1 使用的写屏障 + SATB 产生的浮动垃圾通常比 CMS 使用的写屏障 + 增量更新（ IU ）更多。

## 步骤 3——最终标记阶段

在步骤 2 中介绍了被写屏障感知的对象会先加入 SATB 本地队列，只有当 SATB 本地队列装满之后才会被交给 GC 线程进行扫描和标记，而最终标记阶段则是暂停应用程序 mutator 扫描 SATB 本地队列存放的待扫描对象。本步骤结束后，所有的存活对象都已被标记，所有不带标记的对象都可以判定为死亡对象。

## 步骤 4——存活对象计数

存活对象计数阶段根据每个 region 的标记位图 next，统计当前 region 内存活对象的字节数，存到 region 内的 next_marked_bytes 中。下图中存活对象 A、C、E、H 和 I，一共 5 个对象，其中 E 真实大小是 16 个字节，另外 4 个对象都是 8 个字节，所以 next_marked_bytes 总共 48 个字节。

![存活对象计数结束后 region 的状态](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/%E5%AD%98%E6%B4%BB%E5%AF%B9%E8%B1%A1%E8%AE%A1%E6%95%B0.png)

同时存活对象计数阶段也是与应用程序并发出现的，如果又新创建了对象 L ，会将对象继续放在 nextTAMS 和 top 之间，被当做存活对象处理，同对象 J 、 K 都不会参与本次存活对象计数，留待下次并发标记处理。

## 步骤 5——收尾工作

收尾工作也需要暂停应用程序（ STW ），主要完成两件事情：

1. 将标记位图 next 的并发标记结果移动到标记位图 prev 中，再重置标记位图 next 为空，同时移动  prevTAMS 
2. 计算每个 region 的转移效率，并按照转移效率对 region 进行降序排序

![收尾工作完成 region 的状态](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/%E6%94%B6%E5%B0%BE%E9%98%B6%E6%AE%B5.png)

图中将标记位图 next 的并发标记结果移动到标记位图 prev 中，再重置标记位图 next 为空（会在下次并发标记第 1 步重新创建）。同时 prevTAMS 被移动到了原来 nextTAMS 的位置，表示下次并发标记会从 prevTAMS 开始， prevTAMS 之前的存活对象都被标记在标记位图 prev 中了。 nextTAMS 则会被移动到 bottom 的位置， nextTAMS 会在下次并发标记开始时，移动到 top 的最新位置。

### 转移效率

上文提到， prev 标记位图的作用是在本次标记中确定上一次标记过的活跃对象，用于优化内存管理。存在这种需求的原因是一次并发标记过后的 region 并不会马上被回收，而是选择垃圾最多（转移效率最高）的若干 region 做垃圾回收。

转移效率 = region 内死亡对象的字节数 ÷ 转移所需时间

通俗理解就是， region 内死亡对象字节数越多，存活对象字节数就越少，而存活对象字节数越少，那么转移所需的时间就越少。用更少的时间清理出一块相同大小的 region 就具有更大的转移效率。

# 跨界引用的标记

垃圾收集器在 Partial GC 时（局部收集）都会面临跨代引用的问题（如下图， GC Roots 直接引用的新生代对象 D 存在对老年代对象 C 的引用，而GC Roots 直接引用的老年代对象 B 存在对新生代对象 A 的引用）。

![跨代引用示例](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/%E8%B7%A8%E4%BB%A3%E5%BC%95%E7%94%A8.png)

G1 的问题更加复杂，由于被将堆分割为多个 region ，每个 region 与 region 之间都有可能存在跨界引用，而每次 GC 并不会对所有 region 都进行回收，但是为了准确找到所有需要回收的 region 内的存活对象，就不得不每次都扫描整个堆。为了减少扫描的代价，按照牺牲空间换时间的逻辑，引入了Remembered Set（记忆集，简称RSet）的概念，RSet是一种用于记录从非收集区域指向收集区域的指针集合的抽象结构。《深入理解 Java 虚拟机》一书将 G1 对 RSet 的实现称作“双向的卡表结构”，原始论文中更多的是直接用 Remembered Set（记忆集）来描述 RSet 。

## 卡表（ card table ）

卡表（Card Table）通过卡精度的方式实现，是元素大小为 1B 的数组，HotSpot如下实现

```c++
//share/vm/gc_implementation/g1/heapReagion.cpp
GrainBytes = (size_t)region_size;
CardsPerRegion = GrainBytes >> CardTableModRefBS::card_shift;

//share/vm/memory/cardTableModRefBS.hpp
  enum SomePublicConstants {
    card_shift                  = 9,
    card_size                   = 1 << card_shift,
    card_size_in_words          = card_size / sizeof(HeapWord)
  };

//简化逻辑为
CARD_TABLE [this address >> 9] = 0;
```

不难理解，每一块 region 内某个对象的首地址右移 9 位（除以 512 ）就是对应的卡表索引。换句话说，将 region 按照每 512 个字节作为一个卡页，只要这 512 个字节的卡页中存在一个及以上的对象有对其他区域的对象的引用，就将对应卡表元素变成 1（此处会将 1 写入某块地址占一个 byte ），也就是所谓的卡表元素变脏。在垃圾回收时，只要根据卡表中变脏的元素，找到对应的卡页，再从卡页对应内存地址包含的对象中找到跨界指针，最后将跨界指针加入 GC Roots 中一并扫描。

![卡表结构](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/card_table.png)

如上图所示，每个 region 保存了一个 points-out （“我指向谁”）的卡表，当某个卡表元素为 1 时，认为卡片脏（例如卡片 3 脏），于是只需要去对应的0x0400~0x05FF中扫描对象找到跨界引用即可。

## RSet 的实现

卡表的结构减少了整体扫描的成本， G1 在此基础上为每个 region 构造了另一张表，用于记录下哪些别的region有指向自己指针，而这些指针又分别在那些 region 的哪些 card 的范围内。通过这种方式构造了一种类似双向链表的具有索引“我指向谁”和“谁指向我”的 RSet 结构。

RSet的结构是一种哈希表，

- key ：引用本 region 内对象的其他 region 的起始地址
- value ：数组，数组元素是引用方的对象所对应的卡片索引

借用知乎文章[《关于 G1 GC 的一些研究》](#references)中的例子，如下图所示：

![G1 RSet的引用关系](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/RSet.png)

上图中的 RSet 属于 region B ，RSet 中第一项的 key 指向了 region A 的首地址，value里有index为 1,2,3... 的 card ，意思是指 region A 的 card 对应的卡页里有对象引用了 region B 内的对象。

那么整合卡表就能看清整个 RSet 的全貌了，借用[中村成洋书中案例](#references)如下：

![G1 RSet 使用例子](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/RSet%E4%BE%8B%E5%AD%90.png)

如图在 region A 保存的 RSet 中，存在以 region B 的首地址为 Key ，索引 2048 为 value 的哈希表元素。当需要确定跨界引用时会通过 Key 中 region B 的首地址找到 region B，再通过 value 2048 找到 2048 号卡表对应的 512 Bytes 的地址段，扫描地址段内的对象，找到存在跨界引用向 A 中对象 a 的 B 中对象 b 。

不难发现 RSet 记录的是 points-into 的关系（谁指向我），而 card table 记录的是 points-out 的关系（我指向谁），所以在《深入java虚拟机》一书中用“双向的卡表结构”来形容这个特殊的 RSet 。

## 维护 RSet

那么如何来维护 RSet 呢，考虑如下情况

```java
//假设对象young和old分别在不同的Region中
Object young = new Object();
old.p = young;
```

java 层面给 old 对象的 p 字段赋值 young 对象之后，jvm 底层会执行 oop_store 方法,并在赋值动作**后**使用 post-write barrier 函数。

在介绍 [SATB 伪代码](#satb)时提到过在同样的赋值动作**前**会插入 pre-write barrier 函数（ satb_write_barrier ）用于记录引用的修改，此处的 post-write barrier 函数与 pre-write barrier 函数实现方法类似，先看下方伪代码。

```
//对应JVM 在oop_store方法的赋值动作的前的post-write barrier
def evacuation_write_barrier(obj, field, newobj):
    check = obj ^ newobj
    check = check >> LOG_OF_HEAP_REGION_SIZE
    if newobj == Null:
        check = 0
    if check == 0:
        return

    if not is_dirty_card(obj):
        to_dirty(obj)
        enqueue($current_thread.rs_log, obj)

    *field = newobj
```

第 2 行到第 7 行的代码会在 obj 和 newobj 位于同一个区域，或者 newobj 为 Null 时，起到过滤的作用。第 9 行的函数 is_dirty_card() 用来检查参数 obj 所对应的卡片是否为脏卡片。该行的检查就是为了避免向转移专用记忆集合日志中添加重复的卡片。如果是净卡片，则该卡片将在第 10 行变成脏卡片，然后在第 11 行
被添加到队列 $current_thread.rs_log 中。

与 SATB 一样这里也有一个全局的队列， DirtyCardQueueSet 。更新RSet的动作则会交给多个 ConcurrentG1RefineThread 线程来并发完成，每当这个全局队列超过一定的阈值后， ConcurrentG1RefineThread 都会取出若干个队列，并且遍历队列中的记录的card并将它加到对应的 region 的 RSet 中。

# 转移

转移过程的算法原理可以参考基本概念的 [Copying](#Copying) 算法，G1 转移过程与 Copying 算法大致相同。由于加入了 RSet 优化，所以在处理根对象时略有不同，转移根对象除了需要转移`由根直接引用的对象`，还需要转移`并发标记处理中的对象`以及`由其他区域对象直接引用的回收集合内的对象`

## 回收集确定

在介绍 G1 堆结构时提到 G1 新生代会动态调整大小以满足停顿预测模型，事实上在垃圾回收过程中，G1 会记录每个region 的回收耗时，根据用户的期望停顿时间来生成回收集 Collection Set（简称CSet），一次完整的回收过程，只会转移 CSet 内的存活对象。

G1 中存在两种选定 CSet 的子模式，分别为 Young GC 与 mixed GC 

- Young GC ：选定所有的 young gen 里的 region 。通过控制 young gen 的个数来控制 young GC 的开销。
- Mixed GC ：选定所有的 young gen region ，外加根据“衰减平均值”统计得出的收益较高的若干 old gen region 。在用户指定的最大停顿时间范围内尽可能的选择收益高的old gen Region

其中第一种模式就是G1 新生代会动态调整大小的情况；第二种模式中，由于在并发标记的步骤 5 中对所有 old gen region 计算了转移效率并按照降序排列了，接下来只要依次计算各个 region 的预测暂停时间，再依次将 region 加入 CSet ，在当所有已选 region 的预测暂停时间的总和快要超过最大停顿时间时停止加入即可。

由于 young gen Region 总是在 CSet 内，因此 G1 不维护从 young gen Region 出发的引用涉及到的 Rset 更新。

## 转移过程

![YGC 的转移过程](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/YGC%E8%BD%AC%E7%A7%BB.png)

如上图，YGC 仅将所有新生代 region （包括 Eden 和 survivor ）都加入 CSet ，然后统一转移 CSet 内的对象。晋升的对象会被转移到老年代，其余的转移到 survivor 区。

![Mixed GC 的转移过程](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/MixedGC%E8%BD%AC%E7%A7%BB.png)

如上图，Mixed GC 除了所有新生代区域外，还会选择一些老年代 region 加入 CSet。

# G1GC 过程

引用[《深入探索JVM垃圾回收》](#references)书中对 G1 YGC 与 Mixed GC 状态转换图如下

![GC执行活动图](img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/GC%E6%89%A7%E8%A1%8C%E6%B4%BB%E5%8A%A8%E5%9B%BE.png)

图中对 G1 的 Young GC 、并发标记 、Mixed GC 三种状态做了展示，结合并发标记的过程和 mutator 来看，可以得到下图活动图：

![G1 垃圾回收活动图](/img/G1GC%E7%AE%97%E6%B3%95%E7%AF%87/G1%E6%B4%BB%E5%8A%A8%E5%9B%BE.png)

<p id="references"></p>    

# 参考

《深入 Java 虚拟机-JVM G1GC 的算法与实现》中村成洋/著 吴炎昌 杨文轩/译 1-6章
《深入探索JVM垃圾回收》彭成寒 4-6章  
[关于 G1 GC 的一些研究](https://zhuanlan.zhihu.com/p/463179141)  
[Part 1: Introduction to the G1 Garbage Collector](https://www.redhat.com/en/blog/part-1-introduction-g1-garbage-collector)  
[Collecting and reading G1 garbage collector logs - part 2](https://www.redhat.com/en/blog/collecting-and-reading-g1-garbage-collector-logs-part-2?source=author&term=22991)  
[(八)JVM成神路之GC分区篇：G1、ZGC、ShenandoahGC高性能收集器深入剖析](https://blog.csdn.net/weixin_45101064/article/details/123478022) G1部分