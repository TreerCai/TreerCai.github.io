---
title: JFR概述
date: 2023-5-12 17:25:27
tags:
- Hotspot
- Java
- JDK Flight Recorder
- JDK Mission Control
---

# JFR 简述

JDK Flight Recorder ( JDK 飞行记录器, JFR ) 是内置于 JVM 中的用于提供时间分析和诊断的引擎。JDK Mission Control ( JDK 任务控制, JMC ) 是用于查看 JFR 生成的记录的客户端工具。<!--more-->

基于 [OpenJDK wiki](#references) 中关于 JFR 的介绍可知，JFR 内置于 JVM 中用于记录 JVM 运行过程中若干行为事件的相关数据，具有以下功能：

- 更快地解决程序问题
- 查找应用程序中的瓶颈
- 查找 ISV 提供的应用程序中的瓶颈

JFR 与其他类似工具对比，JFR具有以下关键特点：

- 极低的开销
    - JFR 由开发 JVM 的人构建到 JVM / JDK 中
    - 高性能飞行记录引擎和高性能数据收集
    （时间戳基于固定的 Time Stamp Counter (TSC)，线程本地本机缓冲区( thread local native buffers )，访问运行时中已收集的数据，更准确的方法分析（分析方法的数据甚至有来自外部安全点的），更快、更准确的分配分析）
- 安全可靠
    - 作为 JVM/JDK 测试的一部分，在所有平台上进行了测试
    - 低开销意味着对正在运行的应用程序影响很小
- 低开销 + 可靠性 = 始终开启
    - 在出现问题时转储记录数据，并查看问题发生之前、放生时和发生之后的运行时在做什么
    - 即使在 JVM 进程崩溃时，过去几分钟的飞行记录数据也会在转储中以便解决问题

# 开启 JFR 记录

利用 JFR 监控应用程序的启动方式分为三种，分别是`通过 JVM 启动参数启动`、`通过 jcmd 命令启动`、``

## 通过 JVM 启动参数启动

启动JFR涉及的参数仅仅只有两个，一个负责启动，一个用于配置。JDK 8 中需要额外通过FlightRecorder 打开 FlightRecorder 状态位在 JDK 11 之后不再需要了，这里本文通过下面例子展示：

```bash
java -XX:+FlightRecorder \
     -XX:StartFlightRecording=name=myrecording,filename=recording.jfr,settings=profile \
     test.Main
```

这条命令表示 启动应用程序同时启动一个名为 myrecording 的文件记录，dump 文件名为 recording.jfr 配置文件为 profile.jmc

核心就是 -XX:StartFlightRecording，关于参数配置可以参考博客
[《深度探索JFR - JFR详细介绍与生产问题定位落地 - 1. JFR说明与启动配置》](https://zhanghaoxin.blog.csdn.net/article/details/105241064)

## 通过 jcmd 命令启动

jcmd 命令相关的参数与 JVM 参数涉及的配置参数原理一致，还是通过例子简单展示如何开启如下

### 1. jcmd <pid> JFR.start
启动 JFR 记录，参数和 -XX:StartFlightRecording 一致，注意通过空格分割参数

示例：
```bash
jcmd xxx JFR.start name=myrecording filename=recording.jfr
```
这个就代表启动一个名称为 myrecording文件记录 xxx 为进程号

### 2. jcmd <pid> JFR.stop
停止 JFR 记录，需要传入名称，例如如果要停止上面打开的，则执行：
```bash
jcmd xxx JFR.stop name=profile_online
```
xxx 为进程号

其他 jcmd 功能命令可以参考博客 
[《深度探索JFR - JFR详细介绍与生产问题定位落地 - 1. JFR说明与启动配置》](https://zhanghaoxin.blog.csdn.net/article/details/105241064)


JRF 的记录文件为 *.jfr 格式，通过 JMC 打开 JFR 能够获取记录的详细信息。

# JFR 基本原理

## JFR 的生命周期

博客[通过 JFR 与日志深入探索 JVM - 2. JFR 基本原理以及快慢因素](#references)描述了如下的JFR记录过程
![JFR 记录](img/JFR%E6%A6%82%E8%BF%B0/JFR_record.png)
JFR 记录开始：每个 JVM 进程可以同时启用多个 JFR 记录采集，可以在 JVM 启动的时候利用 JVM 启动参数启用 JFR 记录，也可以通过`jcmd`动态开启 JFR 记录采集，也可以在程序内通过代码开启采集。

JFR 记录结束：可以启动时指定在采集多久后结束，也可以通过jcmd动态关闭 JFR 记录采集，也可以在程序内通过代码结束采集。在结束时，可以指定让 JFR 记录 dump 到一个文件中。JFR 记录也会随着 JVM 的结束而结束。

JFR 记录分析：可以随时通过 jcmd 动态将 JFR 记录 dump 到一个文件中，或者通过代码程序中执行 dump，进行后续分析。注意 dump 并不会结束一个 JFR 记录，并且 dump 最多只能 dump 出上次 dump 到现在的所有记录。

JFR 记录实时分析：可以通过 JFR Stream 实现对于 JFR 记录的实时消费与处理。

## JFR 的记录对象 —— event

在 JFR 记录中，以事件(Event)为记录单位，每个 Event 记录特定粒度的 JVM 行为。

例如：类加载是一个 Event，对应 ClassLoad Event ，开始进入安全点是一个 Event，对应 SafepointBegin Event 等等。根据 oracle 网站提供的[JDK Mission Control User Guide](#references)给出的分类如下：

- 持续时间事件（Duration events）：在特定持续时间内发生，具有特定的开始时间和停止时间。例如有 GC 发生的时候的 GC Event
- 即时事件（Instant events）：立即发生并立即记录，例如，线程被阻塞。
- 示例事件（Sample events）：定期发生（按照一定的频率采集）以检查系统的整体运行状况。例如，每分钟打印一次堆诊断。
- 自定义事件（Custom events）：使用 JMC 或 API 创建的用户定义事件。

按照事件类型，以 JDK11 为例，可以将 Event 分为 5 大类:
|Event 类型|该类 Event 对应 JVM 行为|补充描述|
|:-:|:-:|:-:|
|Flight Recorder|JFR记录监控|包括：JFR数据丢失、开始记录、配置、记录原因等信息|       
|Java Application|Java应用监控|包括：锁与同步、文件IO、网络IO、Java异常与错误、类加载、TLAB分配统计 等信息| 
|Java Development Kit|JDK信息|包括：安全检查、X509证书及验证等信息| 
|Java Virtual Machine|JVM信息|包括：类加载、代码高速缓存、JVM配置运行信息、运行时（安全点、模块化操作）、GC、编译、方法堆栈、偏向锁等信息| 
|Operating System|操作系统信息|包括：内存、网络利用率、线程、CPU占用等信息|

这些 Event 在某些特定的时间点或者特定的场景下产生，每个 Event 都由 `Event 类型`，`开始时间`，`结束时间`，`发生事件的线程`，`事件发生的线程堆栈`还有 `Event 数据体`组成。

当 JFR 开启记录时，不会在事件发生时立即将其写入磁盘。相反，Event 会先将数据存储到自己的线程 JFR 缓冲（Thread Buffer）中；然后在这个 Buffer 满了之后，会将 Buffer 的内容刷入全局 JFR 缓冲（Global Buffer ）中；当 Global Buffer 存储到达上限之后，根据配置选择丢弃或者写入磁盘。数据存储到本地线程 JFR 缓冲这一操作，消除了对每个事件在线程之间进行同步的需要，从而提高了吞吐量。当程本地缓冲区被填满，将数据传输到全局缓冲区这种情况发生时，线程之间需要同步，但是由于不同的线程本地缓冲区以不同的速率填充，因此很少发生争用。JFR 在写入磁盘时通过写入格式紧凑二进制格式文件，增强应用程序的读写。

也可以配置 JFR，使其不向磁盘写入任何数据。在这种模式下，全局缓冲区充当循环缓冲区，当缓冲区满时，将删除最旧的数据。这种情况下全局缓冲区中永远保存最新的数据，当操作或监视系统检测到问题时，再根据需要将信息写入磁盘。

## JFR event 基础信息采集

已知 Event 由 `Event 类型`，`开始时间`，`结束时间`，`发生事件的线程`，`事件发生的线程堆栈`还有 `Event 数据体`组成。

其中 `Event 类型` 由 Event 自身确定。`开始时间` 与 `结束时间` 在支持 Read Time-Stamp Counter (RDTSC)的x86架构上，通过RDTSC获取时间值；在其他（除x86）架构，通过操作系统 OS 时钟获取时间值（接口为 time.h 的 clock_gettime 函数）。由于Java可用多个时钟可能带来的时钟漂移可以参考文章[How to Tell Time in Java’s Garbage Collection](#references)

由于 Event 是由事件发生线程各自记录，所以 Event 的 `发生事件的线程` 信息通过 `Thread::current()` 方法直接获取。同时，在获取到当前线程后先判断是否为 Java 线程，再通过构造 vframeStream 遍历获取`事件发生的线程堆栈`（当开启 OldObjectSample 事件分析时，会追踪内存泄漏，此时会额外在线程 buffer 处记录堆栈。这种情况下堆栈的获取不需要遍历）

最后`Event 数据体`中会记录与关注 Event 相关的信息，这部分信息可以通过读取 JVM 行为产生的变量数据直接获取。这部分数据及数据的描述（除Java Development Kit相关的 Event 信息）都可以在`metadata.xml`中获取

以 `EventThreadSleep` 为例，`metadata.xml`中的描述如下

```xml
  <Event name="ThreadSleep" category="Java Application" label="Java Thread Sleep" thread="true" stackTrace="true">
    <Field type="long" contentType="millis" name="time" label="Sleep Time" />
  </Event>
```
最终通过JMC打开查看可以得到如下信息（JMC作为单独的项目，会在后续文章中进行介绍）
![EventThreadSleep](img/JFR%E6%A6%82%E8%BF%B0/EventThreadSleep.png)

可以很清晰地看出`EventThreadSleep`记录了 `Event 类型`，`开始时间`，`结束时间`（作差即持续时间），`发生事件的线程`，`事件发生的线程堆栈`还有 `Event 数据体`（ Sleep Time ）这几项。

# JMC 分析 JFR

JDK Mission Control (JMC)是一套用于管理、监视、分析和故障排除 Java 应用程序的高级工具。JMC 支持对代码性能、内存和延迟等领域进行高效和详细的数据分析，而不会引入通常与分析和监视工具相关的性能开销。随着发展 JMC 已发展成为应用程序故障排除的一站式商店，它现在（插件形式）集成了许多其他 Java 性能监控实用程序和 Java 虚拟机 (JVM) 调整工具。主要包括：用于托管各种有用的Java工具的框架，用于可视化Java飞行记录内容以及内容自动分析结果的工具，JMX 控制台，堆废物分析工具。

## 自动结果分析页

JMC 从 JFR 记录中提取并分析数据，然后在“自动分析结果”页上显示彩色编码的报告日志，界面如下：

![自动结果分析页面](img/JFR%E6%A6%82%E8%BF%B0/%E8%87%AA%E5%8A%A8%E7%BB%93%E6%9E%9C%E5%88%86%E6%9E%90.png)

问题主要分为以下三个方面呈现：

- Java Application —— Java 应用程序
- JVM internal —— JVM 内部
- Environment —— 环境

这三块也对应 Outline 视图中的三个大类，还有额外的一类 Event Browser 这一块存储了所有 JFR 采集到的单独事件信息。针对每一个得分可以通过点击 `+` 符号下拉获取 JMC 针对某个分数的分析，如下

![分数分析](img/JFR%E6%A6%82%E8%BF%B0/%E5%88%86%E6%95%B0%E5%88%86%E6%9E%90.png)

默认情况下，只显示黄色和红色分数的结果即可能是潜在问题的点。一共四种颜色，灰色 `Not Applicable` 绿色 `OK` 黄色 `Information` 红色 `Warning`，状态依次从未使用到警告事项。可以通过页面右上角`OK`按钮查看每一类包括的所以得分；也可以通过 `Table` 按钮，查看 JMC 计算的所有分数信息，如下

![所有得分](img/JFR%E6%A6%82%E8%BF%B0/score.png)

Table 页面展示了所有的 result 的 score,name,ID,Page ，其中 score 为严重性得分，Page 为该项结果产生依赖的数据页面，对应 Outline 视图下三大分类下的页面，如下

![](img/JFR%E6%A6%82%E8%BF%B0/page-result.png)

可以通过双击条目直接去往产生结果的页面。

## Java 应用程序 分析

Java 应用程序页面显示 Java 应用程序的整体健康状况。需要注意可以在 JMC 工具的 Window -> show view -> Results ，通过这种方式附加显示结果信息。这部分结果信息对应自动结果分析页面展示的 Java 应用程序 类的结果，如下图展示

![Java 应用程序分析](img/JFR%E6%A6%82%E8%BF%B0/Java%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E5%88%86%E6%9E%90.png)

其中红色方框内为这一页计算出的结果，与自动结果分析页面中结果对应。红框内的结果分析依据与当前页面内提供的信息对应印证。

分析页的上半部分是文字形式的线程信息，下半部分为图形界面。图形界面中展示了：中断信息、整体的 CPU 利用率（分机器整体 和 JVM+应用）、堆使用情况、方法概要分析、分配情况、异常抛出信息和活跃线程。当单独选中某一/几个线程时，整体信息会聚焦在某一线程上（CPU 利用率、堆使用情况依旧为全局状态，不做改变），可以通过鼠标悬停获得额外信息

![](img/JFR%E6%A6%82%E8%BF%B0/%E6%82%AC%E5%81%9C.png)

对于本文给出的 jfr 文件中的结果可以如下图寻找印证现象

![](img/JFR%E6%A6%82%E8%BF%B0/%E7%BB%93%E6%9E%9C%E5%AF%B9%E7%85%A7.png)

通过选中 halt 中断信息，找到持续中断时间片最长的，查看对应线程和 VM Operation 可以定位到产生中断的事件是 JFR 本身，结果与分析情况一致。在图中其他的中断可以留给读者自行分析（多数情况为 GC ）。

### Threads 页

“线程”页面提供属于 Java 应用程序的所有线程的快照。它揭示了有关应用程序的线程活动的信息，这些信息可以用于诊断问题并优化应用程序和 JVM 性能。线程在一个表中表示，每一行都有一个关联的图形，图形用于识别有问题的执行模式，它提供了关注区域的上下文信息。

![Threads 页面](img/JFR%E6%A6%82%E8%BF%B0/thread.png)

如图中所示，图中展示了线程运行的状态，页面不与任何结果关联。当前页面还存在下拉页，但不具有更多有意义信息，故不再赘述。

可以通过页面给的 `Thread State Selection` 选项个性化定制在当前页面显示的 `Event` 如下

![](img/JFR%E6%A6%82%E8%BF%B0/TSS.png)

当选中线程和时刻后可以在页面中分析当前时刻的线程堆栈（当打开 jfr 堆栈跟踪后可见）如下

![](img/JFR%E6%A6%82%E8%BF%B0/StackTrace.png)

### Memmory 页

检测 Java 应用程序性能问题的另一种方法是查看它在运行时如何使用内存。在 Memory 页面中，下图表示 Java 应用程序的堆内存使用情况。

![](img/JFR%E6%A6%82%E8%BF%B0/Memory.png)

每个周期由一个 Java 堆增长阶段组成，该阶段表示堆内存分配的周期，随后是一个表示垃圾收集的短暂下拉，然后循环重新开始。从图中可以得出的重要推论是，内存分配是短期的，因为垃圾收集器在每个周期都会将堆推到开始位置。选中 `Garbage Collection` 复选框，可以在图中查看垃圾收集的暂停时间。它表示垃圾收集器在暂停期间停止 Java 应用程序来完成垃圾收集。一般长暂停时间会导致 Java 应用程序性能较差。

与 Memory 页面对应的 Results 分析可以在 Results 页面查看。上图中有一个红色的警告结果分析为物理内存偏小，这一点可以通过看 `Used size` 和 `Total size` 得出。但是需要注意的是自动分析结果只能程序化地得出一些结论，只能作为参考，真实原因可能不在现象处。

上图中很多结果没有分数是因为在 JFR 记录中没有打开相应的 Event 记录。

### Method Profiling 页

“方法分析”页用于查看特定方法的运行频率以及运行方法所需的时间。通过识别执行时间过长的方法来确定方法瓶颈。

由于分析会生成大量数据，因此默认情况下不会打开。通过验证堆栈跟踪中的详细信息、检查代码、验证内存分配是否集中在特定对象上的方式，JFR 找到特定行号的特定问题并输出为结果。

![Method Profiling 页面](img/JFR%E6%A6%82%E8%BF%B0/MethodProfiling.png)

上图中自动分析结果信息显示占用CPU最多的方法为 `void org.spec.jbb.core.collections.HashMultiSet.update(Object, int)` 这一点从方法部分无法准确感知，但是通过分析多个方法堆栈能够得出，`update` 方法的占用确实是最高的。方法分析页面的数据来源是 ExecutionSample 事件，是通过周期线程堆栈的方式搜集方法频次。

### 其他页面

除了上述比较重要的页面，Java 应用程序分析还包括：锁实例页面、File I/O、Socket I/O、Exceptions、Thread Dumps

其中，锁实例页面提供有关指定锁信息（额外显示监视类、统计独立线程数与争用总次数）的线程的详细信息，即线程是否尝试获取锁或等待锁上的通知。如果线程已经采用任何锁，那么会在堆栈跟踪中显示详细信息。

![lock instance 页面](img/JFR%E6%A6%82%E8%BF%B0/lock.png)

File I/O、Socket I/O用于检测文件和 socket 的读写持续时间，将记录持续读写时间超过阈值的记录，同时可以选择记录堆栈。Exceptions 用于记录测试过程中抛出的异常和警告信息。这部分页面与其对应的结果分析方式类似之前的页面，同时都需要在 JFR 中指定打开对应的事件采集。

Thread Dumps 页面对应线程转储事件，这里包括某一时刻的所有线程堆栈信息，可以用于细节分析线程阻塞等问题。

![Thread Dumps 页面](img/JFR%E6%A6%82%E8%BF%B0/ThreadDump.png)

## JVM Internals 分析

“JVM内部”页面提供了有关 JVM 传入参数、默认 flags 值的信息。

![JVM Internals 页面](img/JFR%E6%A6%82%E8%BF%B0/JVMInternals.png)

针对参数的不合理设置会给出结果分析。

### 垃圾收集

JVM Internals 一块最重要参数之一是 `Garbage Collections` 垃圾收集。垃圾收集是一个删除未使用对象的过程，这样空间就可以用于分配新对象。垃圾收集相关的信息通过三页来展示，分别为垃圾收集详细信息、垃圾收集配置信息、垃圾收集汇总信息。

#### GC Summary

界面如下图所示：

![JVM Internals 页面](img/JFR%E6%A6%82%E8%BF%B0/GCSummary.png)

GC Summary 页面不与任何结果先关联，分块输出不同对象类型垃圾收集的信息，包括 GC 次数、平均 GC 时间、最大 GC 时间和总 GC 时间。需要用户根据经验判断 GC 是否存在问题。

#### GC Configuration

界面如下图所示：

![JVM Internals 页面](img/JFR%E6%A6%82%E8%BF%B0/GCConfiguration.png)

GC Configuration 页面只关联是否使用压缩指针这一结果，主页面输出垃圾收集配置信息、Flags、堆配置信息、年轻代配置信息。这些配置信息展现了 JVM 运行过程中的真实信息，辅助用户作出调整。

#### Garbage Collections

Garbage Collections 页保有垃圾收集相关的细节信息，通过图的方式展现。这些图表显示了堆使用情况与（最长、平均、总和）暂停时间的比较以及在指定时间段内的变化情况。这一页还列出了记录过程中发生的所有垃圾回收事件。

![GC 页面](img/JFR%E6%A6%82%E8%BF%B0/GC.png)

安装用户手册的提示，观察堆上最长的停顿时间，如果在应用程序运行期间暂停时间越来越长，则表明垃圾收集释放的堆空间在变少。这种情况说明可能存在内存泄漏情况。

### Compilations 页

“编译”页提供了有关代码编译以及编译持续时间、代码大小、内敛代码大小的详细信息。柱状图按编译时间统计了方法数，有利于精准定位编译耗时较长的方法。下表还可以切换为编译失败的方法，此处不做展示。

![Compilations 页面](img/JFR%E6%A6%82%E8%BF%B0/compile.png)

在大型应用程序中，可能有许多已编译的方法，并且可能会耗尽内存，从而导致性能问题。通过 Compilations 页可以获取相关信息

### Class Loading 页

“类加载”页图部分显示随时间类加载与类卸载的统计情况。下表中展示的是具体相关的类加载器、初始类加载器、类加载线程与类加载持续时间。

![Class Loading 页](img/JFR%E6%A6%82%E8%BF%B0/classload.png)

结果分析对应类加载压力与类泄露两个得分，需要开启相关事件记录才能获取得分。

### VM Operations 页

VM 操作页面统计以 VM 操作为单位的信息，上表信息包括统计时间段内的触发次数、暂停时间、总持续时间、最长持续时间。

![VM Operations 页面](img/JFR%E6%A6%82%E8%BF%B0/VMO.png)

通过选中单个 VM Operation 可以分别查看三张图表：根据时间线查看当前操作的持续时间、根据持续时间统计持续时间范围内的操作个数、所有时间组成的表。

![](img/JFR%E6%A6%82%E8%BF%B0/VMOtimeline.png)![](img/JFR%E6%A6%82%E8%BF%B0/VMOduration.png)![](img/JFR%E6%A6%82%E8%BF%B0/VMOeventlog.png)

### TLAB Allocations

当开始分析 TLAB 相关功能时可以先进入 summary 标签页查看一些汇总信息，分 TLAB 内/外 统计包括：

- 新建 TLAB 计数
- 最大 TLAB 大小
- 最小 TLAB 大小
- 平均 TLAB 大小
- TLAB 总大小的估计值

![TLAB Allocations summary](img/JFR%E6%A6%82%E8%BF%B0/TLABsummary.png)

当具体分析 TLAB 时，可以根据不同情况选择 By Thread / Top Method / Class 三种情况分析

![By Thread](img/JFR%E6%A6%82%E8%BF%B0/TLABThread.png)

当选中不同情况时只会改变上表中第一列的属性，再重新统计表格。以 By Thread 为例，表中后续列分别为

- 某个 Thread / Method / Class 在 TLAB 内申请的大小
- 某个 Thread / Method / Class 在 TLAB 内申请的大小在本次统计到的TLAB 内申请的大小的占比
- 某个 Thread / Method / Class 在 TLAB 外申请的大小
- 某个 Thread / Method / Class 在 TLAB 外申请的大小在本次统计到的TLAB 外申请的大小的占比

下半部分图显示的是：某一时间段内，JVM在 TLAB 内/外 申请的空间大小

## Environment 分析

“环境”页面统计的是当前 JAVA 应用程序运行的环境信息，包括 CPU 类型、核心数量、硬件线程数、插槽数、CPU 说明、可用内存、操作系统版本

![Environment 页面](img/JFR%E6%A6%82%E8%BF%B0/Environment.png)

### 系统信息

Processes “进程”页统计正在运行的并发进程以及这些进程的竞争 CPU 情况。如果许多进程使用 CPU 和其他系统资源，将影响被测试应用程序的性能。

![Processes 页面](img/JFR%E6%A6%82%E8%BF%B0/Processes.png)

如页面中所示，上半部分为图，分别是

- 随时间的 CPU 占用率图（机器总 CPU 占用、JVM+应用程序占用、JVM+应用程序（内核）占用）
- 随时间的并发进程图

下半部分是记录某一时刻的进程信息表，包括进程号、指令、（第一/最后）取样时间

系统信息还包括以下3页：

- Environment Variables “环境变量”页顾名思义统计机器的环境变量
- Native Libraries “本地库”页存储的是本地库的 首地址 与 基地址
- System Properties “系统属性页”记录 Java 系统信息包括如下

![System Properties 键](img/JFR%E6%A6%82%E8%BF%B0/SystemProperties.png)

### Recording 页

Recording “记录”页记录了本次 JFR 测试的相关信息，如下：

![Recording 页面](img/JFR%E6%A6%82%E8%BF%B0/Recording.png)

呈现的信息包括 JFR 开启时间、结束时间、事件总数、记录时间长度、事件开启的个数。以及在页面最下方的表中记录了开启记录的事件配置信息。

## Event Browser

Event Browser “事件浏览器”页面用于查看所有 JFR Event 事件类型的统计信息。可以使用“事件浏览器”页创建自定义页。从 Event Type Tree 中选择所需的事件类型，然后使用页面右上角的 Select Event Type 按钮单击 Create a new page。自定义页在事件浏览器页下列为新的事件页

# JFR + JMC 调优分析

无论是 JVM 的调优过程还是应用程序的调优过程，首先需要做的是性能瓶颈的定位。两者的区别在于性能瓶颈的产生位置不同，其中大多数情况下 JMC 分析的直接结果能够由于指导应用程序的问题寻找；而调优 JVM 则需要进一步根据 JFR 的 Event 测试区间定位问题细节。我们首先来分析 JMC 给出的有用信息

首先在测试平台上稳定运行应用程序，通过 jcmd 命令获取应用程序进程号，开启 JFR 测试并输出 jfr 文件。使用 JMC 离线打开 jfr 文件，首先查看如下页面自动分析页面：

![案例结果自动分析 页面](img/JFR%E6%A6%82%E8%BF%B0/eg1AAR.png)

通过自动分析页面的结果和结果分析可以看出，当前应用程序的瓶颈可能出在 4 个黄色 information 级别的模块上。
首先分析严重程度最高的 Context Switches 问题上，根据分析文字可以得知应用程序的问题可能出在锁的争用上。 点击 Table 按钮切换到 Table 视图，可以得知 Context Switches 问题的产生是通过分析 Lock Instances 页面得出的。

![](img/JFR%E6%A6%82%E8%BF%B0/CSLock.png)

我们需要切换到 Lock Instances 页面:

![案例 Lock Instances 页面](img/JFR%E6%A6%82%E8%BF%B0/egLI.png)

通过页面分析确实可以看出应用程序对 
```java
com.sun.xml.bind.v2.runtime.JAXBContextImpl
```
的争用很多，且导致了很高的总停顿时间，怀疑这里是性能的瓶颈。打开堆栈视图，针对 Monitor Class 分析函数调用情况

![](img/JFR%E6%A6%82%E8%BF%B0/egST.png)

可以看出最终调用的是函数
```java
Encoded[] com.sun.xml.bind.v2.runtime.JAXBContextImpl.getUTF8NameTable()

//bishengjdk-8/jaxws/src/share/jaxws_classes/com/sun/xml/internal/bind/v2/runtime/JAXBContextImpl.java
//not in openjdk11
    public synchronized Encoded[] getUTF8NameTable() {
        if(utf8nameTable==null) {
            Encoded[] x = new Encoded[nameList.localNames.length];
            for( int i=0; i<x.length; i++ ) {
                Encoded e = new Encoded(nameList.localNames[i]);
                e.compact();
                x[i] = e;
            }
            utf8nameTable = x;
        }
        return utf8nameTable;
    }
```

可以得到的结论是，getUTF8NameTable() 同步函数导致了严重的锁争用情况，从而导致了线程的等待性能的下降。
查看 Properities 视图如下

![](img/JFR%E6%A6%82%E8%BF%B0/Properties.png)

从图中能够得知，Lock Instances 页面的 Monitor Class 信息来源是 JFR 的 Java Monitor Blocked 事件记录的。将视图切换到 Event Browser 页面的 Java Monitor Blocked 项如下

![案例 Event Browser 页面](img/JFR%E6%A6%82%E8%BF%B0/egEB.png)

基于 JMC + JFR 的信息分析到此为止，作为应用程序的开发人员可以基于此处给出的函数堆栈优化函数调用，当然也可能需要像 JVM 调优工作者一样再次将目光聚焦在 Java Monitor Blocked 的事件记录上。

<p id="references"></p>   

# 参考

[JFR Overview](https://wiki.openjdk.org/display/jmc/Overview) 
[通过 JFR 与日志深入探索 JVM - 2. JFR 基本原理以及快慢因素](https://blog.csdn.net/zhxdick/article/details/111562928)
[JDK Mission Control User Guide](https://docs.oracle.com/en/java/java-components/jdk-mission-control/8/user-guide/index.html)
[Java Platform, Standard Edition Java Flight Recorder Runtime Guide](https://docs.oracle.com/javacomponents/jmc-5-5/jfr-runtime-guide/toc.htm)
[How to Tell Time in Java’s Garbage Collection](https://devblogs.microsoft.com/java/how-to-tell-time-in-javas-garbage-collection/)
[Advanced Java Diagnostics and Monitoring Without Performance Overhead](https://www.oracle.com/java/technologies/jdk-mission-control.html)
