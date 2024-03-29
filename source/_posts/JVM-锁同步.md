---
title: JVM 锁同步
date: 2023-05-19 17:03:15
tags:
- Hotspot
- Java
- monitor
---

Java 锁相关的资料有很多，此处是笔者的粗浅理解，如有不当之处欢迎指正。<!--more-->

# 前置知识

## synchronized 原理

synchronized 原理的解析与后文源码分析借鉴引用了[《这才叫Synchronized的源码分析》](#references)一文。

synchronized 关键字，用于方法加锁，要求任一线程运行到这个方法时，都要检查有没有其它线程正在用这个方法（或者该类的其他同步方法），有的话要等正在使用 synchronized 方法的线程运行完这个方法后再运行此线程；没有的话,锁定调用者，然后直接运行。

想要实现 synchronized 关键字的上述功能，可以将加锁的逻辑转换为：对一个对象的监视器( monitor )进行排他获取（即同一个时刻只能有一个线程获得由 synchronized 所保护对象的监视器）。在Hotspot中，是通过 ObjectMonitor 来实现，每个对象中都会内置一个 ObjectMonitor 对象

## 对象头的 Mark Word

![HotSpot虚拟机对象头的 Mark Word ](img/monitor%E5%88%86%E6%9E%90/mark.png)

这里参考马智的[《深入剖析Java虚拟机》](#references)给出64位的对象头的 Mark Word 的图解，整体与32位的对象头没有大的出入。对象头是一个Java对象在内存中的布局的一部分，当锁膨胀为重量级锁后，指向重量级锁的指针实际上就是指向对象对应的 monitor，依托 monitor 实现争抢锁的逻辑。

## 锁状态转换

Hotspot 中锁一共四种状态：

`未锁定状态` -> `偏向锁状态` -> `轻量级锁状态` -> `重量级锁状态`

锁只能升级，不能降级，状态转化及对象 Mark word 的关系图如下：

![偏向锁、轻量级锁状态转换与对象头关系图](img/monitor%E5%88%86%E6%9E%90/%E7%8A%B6%E6%80%81%E8%BD%AC%E6%8D%A2%E5%9B%BE.png)

偏向锁状态的 Mark Word 可以很自然地通过计算 hash code 转换为不可用偏向锁的无锁状态；但是被轻量级/重量级锁定状态转换为无锁状态需要恢复 Mark Word 的 age 、 cms_free 属性，这就要求有地方对这几项属性进行保存。

## Lock Record

对于偏向锁和轻量级锁，在代码即将进入同步块的时候，如果此同步对象没有被锁定，jvm 首先将在当前线程 A 的栈帧中建立一个名为锁记录（Lock Record）的空间，轻量级锁将它用于存储锁对象目前的 Mark Word 的拷贝。获取锁的过程可以简化为尝试将对象的锁拥有者（ Mark Word / _owner ） 写为当前线程 A 的过程。

锁重入：同一个线程执行外层同步方法时获得对象锁之后，对象的所有同步方法都已被上锁，内层同步方法再次获取该锁称为重入。

Displayed Mark Word 用于存储对象加锁前的 Mark Word

下图为当对象所处于偏向锁时，当前线程重入3次，线程栈帧中Lock Record记录：

![ 持有偏向锁的线程堆栈 ](img/monitor%E5%88%86%E6%9E%90/LockRecord.png)

# Hotspot 中锁的实现逻辑

想要理解锁的实现逻辑，首先需要理解这个锁被设计出来的目的以及它的作用；再通过逻辑转换图的方式理解锁的实现逻辑，同时印证锁的作用；三种锁有不同的处理逻辑，处理逻辑也包含着状态转换的逻辑。

## 偏向锁

### 偏向锁的目的作用

大多数情况下，锁不仅存在多线程竞争，当一个线程获取锁，后面还有大概率该线程还会需要继续持有这把锁。为了优化性能，于是在无竞争的情况下将同步过程移除（不需要 CAS ）。偏向锁的加锁与撤销流程，博客[《偏向锁的获取和撤销详解》](#references)一文非常详实，此处直接引用。

### 偏向锁的加锁

- 线程 A 第一次访问同步代码块时，先检查对象头 Mark Word 中锁标志位是否为 01 ，依此判断此时对象是否处于无锁状态或者偏向锁状态；

- 若锁标志位是为 01 ，然后判断偏向锁的标识是否为 1 ：
    - 如果不是，则进入轻量级锁逻辑（使用CAS竞争锁）（注意：此时不是使用CAS尝试获取偏向锁，而是直接升级为轻量级锁；原因是：当偏向锁的标识为0时，表明偏向锁在此对象上被禁用，禁用原因可能是JVM关闭了偏向锁模式，或该类刚经历过 bulk revocation ，等等。所以应该入轻量级锁逻辑）：
    - 如果是 1 ，表明此对象是偏向锁状态，则进行下一步流程。

- 判断是偏向锁时，检查对象头 Mark Word 中记录的 ThreadID 是否是当前线程 A 的 ID ：
    - 如果是，则表明当前线程 A 已经获得过该对象锁，以后线程A进入同步代码块时，不需要 CAS 进行加锁，只会往当前线程 A 的栈中添加一条 Displaced Mark Word 为空的 Lock Record ，用来统计重入的次数。
    - 如果不是，则进行 CAS 操作（比较对象 Mark Word 和 匿名的 Mark Word （ThreadID属性为 0 ） 是否一致，一致则将 当前线程写入对象 Mark Word），尝试将当前线程 A 的 ID 替换进 Mark Word ：
      - 如果当前对象锁的 ThreadID 为 0（匿名偏向锁状态），则会替换成功（将 Mark Word 中的 Thread id 由匿名 0 改成当前线程 A 的 ID ，在当前线程 A 栈中找到内存地址最高的可用 Lock Record ，将线程 A 的 ID 存入），获得到锁，执行同步代码块。
      - 如果当前对象锁的 ThreadID 不为 0 ，即该对象锁已经被其他线程 B 占用了，则会替换失败，开始进行偏向锁撤销。这也是偏向锁的特点，一旦出现线程竞争，就会撤销偏向锁。

当线程 A 对对象加偏向锁后，在同步方法退出甚至线程死亡时并不会对这个锁做传统的解锁操作，只有当另一个线程 B 也需要持有这个对象的锁，才会发生偏向锁的撤销操作。

### 偏向锁的撤销

此时持有偏向锁的线程为 A ，线程 B 尝试进入同步块（获取锁），此时会发生偏向锁的撤销
- 偏向锁的撤销需要等待全局安全点（ safe point ，代表了一个状态，在该状态下所有线程都是暂停的， stop-the-world ），到达全局安全点后，持有偏向锁的线程 A 也被暂停了。
- 检查持有偏向锁的线程 A 的状态（会遍历当前 JVM 的所有线程，如果能找到线程 A ，则说明偏向的线程 A 还存活着）：
    - 如果线程还存活，则检查线程是否还在执行同步代码块中的代码：
      - 如果是，则把该偏向锁升级为轻量级锁，且原持有偏向锁的线程 A 继续获得该轻量级锁。
      - 如果否，处理同线程未存活。
    - 如果线程未存活，或线程未在执行同步代码块中的代码（线程 A 的栈中是否存在指向该对象的 Lock Record），则进行校验是否允许重偏向：
      - 如果不允许重偏向，则将 Mark Word 设置为无锁状态（未锁定不可偏向状态），然后升级为轻量级锁，进行轻量级锁的 CAS 竞争锁操作。
      - 如果允许重偏向，设置为匿名偏向锁状态（即线程 A 释放偏向锁）。当唤醒线程后，进行 CAS 将偏向锁指向线程 B （在对象头和线程栈帧的锁记录中存储当前线程 ID ）。
- 唤醒暂停的线程，从安全点继续执行代码。

如何判断线程是否还在执行同步代码块中的代码：每次进入同步块（即执行 monitorenter ）的时候都会以从高往低的顺序在栈中找到第一个可用的 Lock Record ，并设置偏向线程 ID ；每次解锁（即执行 monitorexit ）的时候都会从最低的一个 Lock Record 移除。所以如果能找到对应的Lock Record说明偏向的线程还在执行同步代码块中的代码。

### 批量重偏向与批量撤销逻辑

默认开启批量重偏向
BiasedLockingBulkRebiasThreshold 20 偏向锁批量重偏问阈值
BiasedLockingBulkRevokeThreshold 40 偏向锁批量撤销值阈值

**批量重偏向**

- 线程 A 创建了同一个类( a.class )的 50 个对象 obj ，jvm 会在 obj 对象的`类class对象`中， 定义了一个偏向 revoke （撤销）计数器以及 epoch 偏向版本。
- 线程 A 都调用了1\~49 号对象的同步方法并退出不再进入，调用第 50 号对象的同步方法并停留，此时这 50 个对象都获取到了指向线程 A 的偏向锁。
- 线程 B 依次调用第 1\~50 号对象的同步方法，第 1\~19 号对象的偏向锁发生撤销（ a.class类对象 的偏向锁撤销计数器 +1）。
  - 由于线程 A 已经退出了 1\~49 号对象的同步块且不再进入，此时将第 1\~19 号对象的 Mark Word 设置为无锁状态（未锁定不可偏向状态），然后升级为轻量级锁。
  - 线程 B 调用第 20 号对象的同步方法，发生第 20 次偏向锁撤销，a.class 类对象的偏向撤销计数器自增为20，认为出现大规模的锁撤销动作，a.class 类对象 中的 epoch 值 +1。
  - 偏向锁撤销一定发生在安全点， 此时 jvm 会找到所有正处在同步代码块中的obj对象（第 50 号对象），让他们的 epoch 等于 a.class 类对象的 epoch。其他不在同步代码块中的 obj 对象，则不修改 epoch 。
  - 线程 B 继续获取第 20~49 号对象的锁，此时在撤销指向线程 A 的偏向锁时，判断到 obj 对象的 epoch 和 a.class 对象的 epoch 不相等（即允许重偏向状态），会将偏向锁设置为匿名偏向锁。当唤醒线程后，进行 CAS 将偏向锁直接指向线程 B （在对象头和线程栈帧的锁记录中存储当前线程 ID ）。
  - （在批量重偏向之后，B 调用 50 号对象的同步方法之前，线程 A 退出 50 号对象的同步方法）当线程 B 获取第 50 号对象锁时，发生偏向锁的撤销，由于发生批量重偏向时该对象处在同步块中，它的 epoch 更新已经等于 a.class 类对象的 epoch，属于不允许重偏向，将第 50 号对象的 Mark Word 设置为无锁状态（未锁定不可偏向状态），然后升级为轻量级锁。
  - 线程 B 释放所有对象锁退出同步块，此时状态 第 1\~19,50 号对象 Mark Word 为未锁定不可偏向状态（0|01），第 20\~49 号对象的 Mark Word 为指向线程 B 的偏向锁

**批量撤销**

- 线程 C 依次调用第 20\~39 号对象的同步方法，第 20\~39 号对象的偏向锁发生撤销（ a.class类对象 的偏向锁撤销计数器 +1）。
- 由于线程 B 已经退出了所有对象的同步块，此时将第 20\~39 号对象的 Mark Word 设置为无锁状态（未锁定不可偏向状态），然后升级为轻量级锁。
  - 线程 C 调用第 39 个对象的同步方法，发生第 40 次偏向锁撤销，a.class 类对象的偏向撤销计数器自增为40，发生批量撤销。
  - jvm 此时会将 a.class 类对象中的偏向标记（注意是类中的偏向锁开启标记，而不是对象头中的偏向锁标记）设置为禁用偏向锁。
- 后续 a.class 对应的对象的 new 操作将直接走轻量级锁的逻辑。
- 如果此时线程 B 调用第 40\~49 号对象（这几个对象的 Mark Word 为已偏向状态，偏向线程 B ），此时由于已经发生了批量撤销，获取锁的方式变换为获取轻量级锁，不再走偏向锁逻辑。

![批量重偏向与批量撤销](img/monitor%E5%88%86%E6%9E%90/%E6%89%B9%E9%87%8F%E9%87%8D%E5%81%8F%E5%90%91%E3%80%81%E6%89%B9%E9%87%8F%E6%92%A4%E9%94%80.png)

### 偏向锁过程转换图

可以结合流程图理解：

![ 偏向锁的加锁与撤销 ](img/monitor%E5%88%86%E6%9E%90/biased.png)

图中，黄色块CAS操作的成功、失败的判断依据为 Mark Word 的线程 ID 是否为匿名状态。
偏向锁的获取撤销涉及的对象头转换涵盖了上文[偏向锁、轻量级锁状态转换与对象头关系图](#锁状态转换)的左半部分

## 轻量级锁

偏向锁针对的情况是单一线程长期持有锁的情况，轻量级锁针对的情况是线程交替（不能是同时）执行同步块的情况。

### 轻量级锁的加锁

- 在代码进入同步块的时候，先检查对象头 Mark Word 中锁标志位是否为 01 ，依此判断此时对象是否处于无锁状态或者偏向锁状态；
- 若锁标志位是为 01 ，然后判断偏向锁的标识是否为 1 ：
  - 如果是 1 ，表明此对象是偏向锁状态，上文已经讨论。
  - 如果不是，则进入轻量级锁逻辑
- 由于已经有同步对象锁状态为无锁状态（锁标志位为“ 01 ”状态），于是在当前线程的栈帧中建立一个名为锁记录（ Lock Record ）的空间，用于存储锁对象目前的 Mark Word 的拷贝，官方称之为 Displaced Mark Word。
  - 拷贝对象头中的 Mark Word 复制到锁记录中。
  - 拷贝成功后，将使用 CAS 操作（比较拷贝过来的 Displaced Mark Word 与 对象头的 Mark Word 是否一致，防止拷贝期间锁被其他线程已经抢占）尝试将对象的 Mark Word 更新为指向 Lock Record 的指针，并将 Lock record 里的 owner 指针指向 object mark word 。
    - 如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，即表示此对象处于轻量级锁定状态。
    - 如果这个更新操作失败了，说明此时对象头的 Mark Word 已经被修改了（在尝试获取期间被其他线程抢到了锁），需要进入锁膨胀流程。
- 若锁标志位是为 00 ，则说明此时已经处于轻量级锁的上锁状态，判断对象的 Mark Word 是否指向当前线程的栈帧；
  - 如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行。
  - 如果不是说明多个线程竞争锁，需要进入锁膨胀流程。
- 当抢锁失败（01锁状态和00锁状态都会出现抢锁失败）之后，就需要进入锁膨胀过程，轻量级锁膨胀为重量级锁，锁标志的状态值变为“10”，Mark Word 中存储的就变成指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。 而当前线程便尝试使用自旋来获取锁，自旋就是为了不让线程阻塞，而采用循环去获取锁的过程。

### 轻量级锁的解锁

- 线程推出同步块方法后，首先获取当前线程中末尾的 Lock Record 中的 Mark Word ，判断 Mark Word 是否为 Null
  - 如果为 Null 说明是从重入锁退出，那只需要删除当前的 Lock Record 即可。
  - 如果不是 Null 说明是当前线程释放该锁，进入轻量级锁的实际解锁过程。
- 通过CAS操作（比较对象头 Mark Word 指向的地址 和 线程栈帧中 Displayed Mark Word 的地址是否一致，一致说明是轻量级锁，否则说明已经指向重量级锁了）尝试把线程中复制的 Displaced Mark Word 对象替换当前的Mark Word（回到无锁状态）。
- 如果替换成功，整个同步过程就完成了，锁完成解锁。
- 如果替换失败，说明有其他线程尝试过获取该锁（此时锁已膨胀），那就要在释放锁的同时，唤醒被挂起的线程（过程在重量级锁中展开）。

## 重量级锁

### 

### 

# 加锁过程源码分析


<p id="references"></p>    

# 参考

[这才叫Synchronized的源码分析](https://blog.csdn.net/qq_42986622/article/details/120845878)
[锁原理：偏向锁、轻量锁、重量锁1.加锁2.撤销偏向锁1.加锁2.解锁3.膨胀为重量级锁](https://cloud.tencent.com/developer/article/1036756)
[java中的锁——偏向锁，轻量级锁，重量级锁](https://blog.csdn.net/A_BCDEF_/article/details/89436705)
[偏向锁的获取和撤销详解](https://blog.csdn.net/weixin_43882265/article/details/121318419)
[Java并发编程：Synchronized底层优化（偏向锁、轻量级锁）](https://www.cnblogs.com/paddix/p/5405678.html)
[7000+字图文并茂解带你深入理解java锁升级的每个细节](https://zhuanlan.zhihu.com/p/537852119)
[死磕Synchronized底层实现--偏向锁](https://github.com/farmerjohngit/myblog/issues/13)
《深入理解Java虚拟机》周志明
《深入剖析Java虚拟机》马智