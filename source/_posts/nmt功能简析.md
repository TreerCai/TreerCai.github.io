---
title: nmt功能简析
date: 2023-07-12 08:46:47
tags:
- Hotspot
- Java
- NMT
---

JVM通常会额外分配内存。这些额外的分配，会导致java程序占用的内存，超出-Xmx的限制。此时可以通过NMT查看内存的使用情况<!--more-->

# NMT 是什么
NMT 是一项 Java Hotspot 功能，用于跟踪 JVM Hotspot 内部内存的使用情况。NMT 不跟踪第三方 Native 代码和 Oracle Java Development Kit (JDK) 类库的内存分配。本文主要参考 Oracle 提供的[手册](#references)

# 主要特征
Native Memory Tracking 与 jcmd 一起使用时，可以在不同级别下跟踪 Java 虚拟机 (JVM)内存使用情况。

NMT 支持以下功能：

- 运行程序运行时生成内存使用摘要或详细报告；
- 在运行程序开始运行的早期建立基线，用于运行程序执行一段时间后比较内存变化；
- 使用 JVM 命令行选项在 JVM 退出时打印内存使用报告。

# NMT的使用

JVM 的 NMT 默认关闭。通过 -XX:NativeMemoryTracking=[off | summary | detail] 开启不同级别的监控

- summary 仅收集聚合的内存使用情况。
- detail  收集各个调用点的内存使用情况。

注：根据 Oracle 给出的[故障排除指南](#references)中所写 jdk8 在启用 NMT时 会导致 5% -10% 的性能开销。

## 使用 jcmd 访问 NMT 数据

通过 jcmd 可以转储记录的数据，并可选择将数据与上一个基线进行比较。

**语法：**
```shell
jcmd <pid> VM.native_memory [summary | detail | baseline | summary.diff | detail.diff | shutdown] [scale= KB | MB | GB]
```
参数说明：

|jcmd NMT 选项|描述|
|:---:|:---|
|summary|打印按类别汇总的摘要。|
|detail|打印内存使用情况，按类别汇总<br > 打印虚拟内存映射<br> 打印内存使用情况，按调用者汇总|
|baseline|创建一个新的内存使用快照作为基线，用于后续比较。|
|summary.diff|根据最后一个基线打印一份新的总结报告。|
|detail.diff|根据最后一个基线打印一份新的详细报告。|
|shutdown|停止 NMT。|

### summary 级别输出示例如下：

```text
Native Memory Tracking:

Total: reserved=3622524851, committed=910833587                                 <--- total memory tracked by Native Memory Tracking
-                 Java Heap (reserved=2067791872, committed=745537536)          <--- Java Heap
                            (mmap: reserved=2067791872, committed=745537536) 
 
-                     Class (reserved=1097799251, committed=26809939)           <--- class metadata
                            (classes #3569)                                     <--- number of loaded classes
                            (  instance classes #3297, array classes #272)
                            (malloc=988755 #9613)                               <--- malloc'd memory, #number of malloc
                            (mmap: reserved=1096810496, committed=25821184) 
                            (  Metadata:   )
                            (    reserved=23068672, committed=23068672)
                            (    used=21384400)
                            (    free=1684272)
                            (    waste=0 =0.00%)
                            (  Class space:)
                            (    reserved=1073741824, committed=2752512)
                            (    used=2270784)
                            (    free=481728)
                            (    waste=0 =0.00%)
 
-                    Thread (reserved=29497864, committed=1387016)              
                            (thread #28)                                        <--- number of threads
                            (stack: reserved=29360128, committed=1249280)       <--- memory used by thread stacks
                            (malloc=106104 #170) 
                            (arena=31632 #54)                                   <--- resource and handle areas
 
-                      Code (reserved=263208110, committed=21933230)
                            (malloc=9575598 #263692) 
                            (mmap: reserved=253632512, committed=12357632) 
 
-                        GC (reserved=135893815, committed=86831927)
                            (malloc=25080631 #7158) 
                            (mmap: reserved=110813184, committed=61751296) 
 
-                  Compiler (reserved=2007114, committed=2007114)
                            (malloc=32450 #303) 
                            (arena=1974664 #11)
 
-                  Internal (reserved=824541, committed=824541)
                            (malloc=791773 #1993) 
                            (mmap: reserved=32768, committed=32768) 
 
-                     Other (reserved=135456, committed=135456)
                            (malloc=135456 #6) 
 
-                    Symbol (reserved=6324688, committed=6324688)
                            (malloc=3861456 #20058) 
                            (arena=2463232 #1)
 
-    Native Memory Tracking (reserved=4917896, committed=4917896)
                            (malloc=10744 #87) 
                            (tracking overhead=4907152)
 
-               Arena Chunk (reserved=13747712, committed=13747712)
                            (malloc=13747712) 
 
-                   Tracing (reserved=120, committed=120)
                            (malloc=120 #5) 
 
-                   Logging (reserved=6276, committed=6276)
                            (malloc=6276 #198) 
 
-                 Arguments (reserved=20264, committed=20264)
                            (malloc=20264 #515) 
 
-                    Module (reserved=184000, committed=184000)
                            (malloc=184000 #1246) 
 
-              Synchronizer (reserved=157680, committed=157680)
                            (malloc=157680 #1038) 
 
-                 Safepoint (reserved=8192, committed=8192)
                            (mmap: reserved=8192, committed=8192) 
```

结果参数说明：（下表摘自[ Native Memory Tracking Memory Categories](#references)）

|类别|描述|
|:---:|:---|
|Java Heap|对象所在的堆|
|Class|类元数据|
|Thread|线程使用的内存，包括线程数据结构、资源区、句柄区等|
|Code|生成的代码 JIT生成并缓存的汇编指令的内存占用情况|
|GC|GC使用的数据，例如卡片表，除了记忆集|
|GCCardset|GC 的记忆集使用的数据( 可选，仅限 G1)|
|Compiler|编译器在生成代码时使用的内存跟踪|
|Interna|不符合前面几类的内存，比如命令行解析器、JVMTI、属性等使用的内存|
|Other|其他类别未涵盖的内存|
|Symbol|符号占用的内存 如string table和constant pool|
|Native Memory Tracking|NMT 使用的内存|
|Arena Chunk|arena 块池中的块使用的内存 用于存放虚拟机的内部对象和数据的内存块池 |
|Logging|记录使用的内存|
|Arguments|参数占用的内存|
|Module|模块使用的内存|
|Synchronizer|用于锁同步的内存|
|Safepoint|用于安全点的内存|

NMT显示总的预留（reserved）内存、已提交（committed）内存：
```
Total: reserved=3622524851, committed=910833587
```
reserved 内存表示我们的应用程序可能使用的内存总量。committed 内存表示应用程序当前使用的内存。如果不显示单位默认为 B

### detail级别的额外输出

detail 级别的 NMT 会精确地跟踪分配内存的方法，并统计出分配最多内存的方法，如下示例：

```text
Virtual memory map:

[0x00000000a1000000 - 0x0000000800000000] reserved 30916608KB for Java Heap from
    [0x00007f5b91a2472b] ReservedHeapSpace::try_reserve_heap(unsigned long, unsigned long, bool, char*)+0x20b
    [0x00007f5b91a24de9] ReservedHeapSpace::initialize_compressed_heap(unsigned long, unsigned long, bool)+0x5a9
    [0x00007f5b91a254c6] ReservedHeapSpace::ReservedHeapSpace(unsigned long, unsigned long, bool, char const*)+0x176
    [0x00007f5b919da835] Universe::reserve_heap(unsigned long, unsigned long)+0x65

               [0x00000000a1000000 - 0x0000000117000000] committed 1933312KB from
            [0x00007f5b9132c9be] G1PageBasedVirtualSpace::commit(unsigned long, unsigned long)+0x18e
            [0x00007f5b913414d1] G1RegionsLargerThanCommitSizeMapper::commit_regions(unsigned int, unsigned long, WorkGang*)+0x1a1
            [0x00007f5b913d5c78] HeapRegionManager::commit_regions(unsigned int, unsigned long, WorkGang*)+0x58
            [0x00007f5b913d6c45] HeapRegionManager::expand(unsigned int, unsigned int, WorkGang*)+0x35

               [0x00000007fe000000 - 0x00000007fef00000] committed 15360KB from
            [0x00007f5b9132c9be] G1PageBasedVirtualSpace::commit(unsigned long, unsigned long)+0x18e
            [0x00007f5b913414d1] G1RegionsLargerThanCommitSizeMapper::commit_regions(unsigned int, unsigned long, WorkGang*)+0x1a1
            [0x00007f5b913d5c78] HeapRegionManager::commit_regions(unsigned int, unsigned long, WorkGang*)+0x58
            [0x00007f5b913d7355] HeapRegionManager::expand_exact(unsigned int, unsigned int, WorkGang*)+0xd5

```

输出包括分配内存的区域以及申请内存区域的最大深度为 4 的方法栈。

## 在 VM 退出时获取 NMT 数据
要在 VM 退出时获取上次内存使用情况的数据，请在启用本机内存跟踪时使用以下 VM 诊断命令行选项。详细程度基于跟踪级别。

-XX:+UnlockDiagnosticVMOptions -XX:+PrintNMTStatistics

# 使用 NMT 检测内存泄漏
使用 NMT 检测内存泄漏的过程步骤：

使用命令行选项启动 JVM 并进行摘要或详细信息-XX:NativeMemoryTracking=summary/detail
建立早期基线。使用 NMT 基线功能通过运行以下命令在开发和维护期间获取要比较的基线：
```shell
jcmd <pid> VM.native_memory baseline.
```
运行程序执行一段时间后通过以下命令行监视内存更改：
```shell
jcmd <pid> VM.native_memory detail.diff。
```
如果应用程序泄漏少量内存，则可能需要一段时间才能显示出来。
NMT通过+ -符号，表示这段时间内，内存占用的变化情况：
```
Native Memory Tracking:

Total: reserved=1831059KB +359KB, committed=146815KB +1383KB

-                 Java Heap (reserved=442368KB, committed=44060KB)
                            (mmap: reserved=442368KB, committed=44060KB)
 
-                     Class (reserved=1082518KB +38KB, committed=37526KB +1062KB)
                            (classes #6372 +210)
                            (malloc=1174KB +38KB #7083 +236)
                            (mmap: reserved=1081344KB, committed=36352KB +1024KB)
 
-                    Thread (reserved=29927KB, committed=29927KB)
                            (thread #30)
                            (stack: reserved=29792KB, committed=29792KB)
                            (malloc=102KB #174)
                            (arena=32KB #56)
 
-                      Code (reserved=251480KB +34KB, committed=11828KB +34KB)
                            (malloc=1880KB +34KB #3409 +137)
                            (mmap: reserved=249600KB, committed=9948KB)
 
-                        GC (reserved=1473KB, committed=181KB)
                            (malloc=25KB #126)
                            (mmap: reserved=1448KB, committed=156KB)
 
-                  Compiler (reserved=161KB +10KB, committed=161KB +10KB)
                            (malloc=28KB +10KB #334 +26)
                            (arena=133KB #5)
 
-                  Internal (reserved=11398KB +41KB, committed=11398KB +41KB)
                            (malloc=11366KB +41KB #7936 +212)
                            (mmap: reserved=32KB, committed=32KB)
 
-                    Symbol (reserved=10046KB +156KB, committed=10046KB +156KB)
                            (malloc=6969KB +156KB #62913 +1947)
                            (arena=3077KB #1)
 
-    Native Memory Tracking (reserved=1511KB +81KB, committed=1511KB +81KB)
                            (malloc=187KB +33KB #2645 +469)
                            (tracking overhead=1325KB +47KB)
 
-               Arena Chunk (reserved=177KB, committed=177KB)
                            (malloc=177KB)
```

如果是 detail 级别的基线对比可以看到如下示例：

```text
[0x00007f5b9175ea8b] MemBaseline::aggregate_virtual_memory_allocation_sites()+0x11b
[0x00007f5b9175ed68] MemBaseline::baseline_allocation_sites()+0x188
[0x00007f5b9175efff] MemBaseline::baseline(bool)+0x1cf
[0x00007f5b917d19a4] NMTDCmd::execute(DCmdSource, Thread*)+0x2b4
                             (malloc=1KB type=Native Memory Tracking +1KB #18 +18)

[0x00007f5b917635b0] MallocAllocationSiteWalker::do_malloc_site(MallocSite const*)+0x40
[0x00007f5b91740bc8] MallocSiteTable::walk_malloc_site(MallocSiteWalker*)+0x78
[0x00007f5b9175ec32] MemBaseline::baseline_allocation_sites()+0x52
[0x00007f5b9175efff] MemBaseline::baseline(bool)+0x1cf
                             (malloc=11KB type=Native Memory Tracking +10KB #156 +136)

[0x00007f5b91a2472b] ReservedHeapSpace::try_reserve_heap(unsigned long, unsigned long, bool, char*)+0x20b
[0x00007f5b91a24de9] ReservedHeapSpace::initialize_compressed_heap(unsigned long, unsigned long, bool)+0x5a9
[0x00007f5b91a254c6] ReservedHeapSpace::ReservedHeapSpace(unsigned long, unsigned long, bool, char const*)+0x176
[0x00007f5b919da835] Universe::reserve_heap(unsigned long, unsigned long)+0x65
                             (mmap: reserved=30916608KB, committed=475136KB +81920KB Type=Java Heap)

[0x00007f5b91804557] thread_native_entry(Thread*)+0xe7
                             (mmap: reserved=34868KB, committed=1224KB +68KB Type=Thread Stack)

[0x00007f5b91a23c63] ReservedSpace::ReservedSpace(unsigned long, unsigned long)+0x213
[0x00007f5b912df57c] G1CollectedHeap::create_aux_memory_mapper(char const*, unsigned long, unsigned long)+0x3c
[0x00007f5b912e4f13] G1CollectedHeap::initialize()+0x333
[0x00007f5b919da5dd] universe_init()+0xbd
                             (mmap: reserved=483072KB, committed=7424KB +1280KB Type=GC)

[0x00007f5b91a23c63] ReservedSpace::ReservedSpace(unsigned long, unsigned long)+0x213
[0x00007f5b912df57c] G1CollectedHeap::create_aux_memory_mapper(char const*, unsigned long, unsigned long)+0x3c
[0x00007f5b912e4e6a] G1CollectedHeap::initialize()+0x28a
[0x00007f5b919da5dd] universe_init()+0xbd
                             (mmap: reserved=60384KB, committed=928KB +160KB Type=GC)
```

可以看出与基线时刻对比，Java Heap 类型新 committed 的内存大小为 475136KB ，是最多的新增部分。
通过 NMT 实现内存泄漏的纠错，可以结合内存监控手段 pmap 确定是 JVM 带来的内存使用量增长还是 JNI 带来的内存使用量增长。如果是 JVM 带来的内存使用量增长，再确定是哪一部份的内存使用量涨了。

<p id="references"></p>   

# 参考
[Java Platform, Standard Edition Troubleshooting Guide——2.7 Native Memory Tracking]https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr007.html
[Native Memory Tracking]https://docs.oracle.com/javase/8/docs/technotes/guides/vm/nmt-8.html
[Native Memory Tracking Memory Categories]https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr022.html#BABHIFJC
[Troubleshooting Guide]https://docs.oracle.com/en/java/javase/20/troubleshoot/diagnostic-tools.html#GUID-FB0581EA-2F91-4093-B2FA-46687F7AB081
[Java Virtual Machine Guide——9 Native Memory Tracking]https://docs.oracle.com/en/java/javase/20/vm/native-memory-tracking.html#GUID-710CAEA1-7C6D-4D80-AB0C-B0958E329407
