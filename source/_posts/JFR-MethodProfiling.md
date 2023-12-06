---
title: JFR MethodProfiling
date: 2023-06-05 10:33:09
tags:
- Hotspot
- Java
- JDK Flight Recorder
---

# JFR MethodProfiling Event 简介

JMC 的 “方法分析” 页用于查看特定方法的运行频率以及运行方法所需的时间。可以帮助用户通过识别执行时间过长的方法来确定方法瓶颈。<!--more-->

方法分析页面如下：

![Method Profiling 页面](img/JFRMethodProfiling/JMCmp.png)

页面中能提供的信息包括：

- 根据采样信息 JMC 分析得出的热点方法堆栈（红框）
- 被采样到的正在运行的方法，及方法计数（黄色箭头）
- 通过点击被采样到的方法，查看调用该方法的堆栈（黄色方框）

细心的读者估计可以发现，JMC 分析得出的热点方法与采样次数最多的方法并不一致。那是因为 JMC 分析的依据是所有方法，包括堆栈内的方法。

方法分析页面的数据来源是 JFR 记录的 ExecutionSample 事件信息（ NativeMethodSample 事件信息不会在当前页面展示）只记录真正执行的普通方法。通过 JFR 进行方法采样分析与通过 perf 采样分析，两者存在如下区别：

- 采样原理不同，perf 基于内核，JFR 基于 Hotspot
- 采样过程不同
- 采样对象不同，JFR 只采普通 Java 方法

perf的原理过程可以参考网上资料，本文主要分析 JFR ExecutionSample 事件的采样逻辑

# JFR MethodProfiling Event 采样算法文字描述

1. JFR 线程为 JFR 方法采样线程的对象设置采样间隔时间参数，并启动 JFR 采样线程（下文简称采样线程）。JFR 线程通过加减信号量（semaphore）_sample 的方式开关方法采样。
2. 采样线程获取当前时间，初始化`上次采样结束的时间`
3. 采样线程调用 sem_wait 尝试将信号量 _sample 减去 1  (初始由 JFR 线程将信号量 _sample + 1 -> 1)
  - 3.1 大于 0 _sample 成功被减去 1，转到 4 (采样线程获得锁可运行)
  - 3.2 小于等于 0  阻塞当前线程直到 JFR线程 将 _sample 设置为大于0
4. 采样线程对信号量 _sample + 1 ，根据`上次采样结束的时间` + `设置的间隔时间` = `下次采样的时间`，计算出下次预计的采样时间
5. 采样线程判断 `下次采样的时间` 是否大于 `当前时间`（通过 clock_gettime 获得）
  - 5.1 `下次采样的时间` 大于 `当前时间`说明还不到预计的采样时间，不能开始采样，sleep 一段时间。`睡眠时间` = `下次采样的时间` - `当前时间`
  - 5.2 `下次采样的时间` 小于 `当前时间`说明已经过了预计的采样时间可以开始采样了
6. 采样线程根据上次被采样的线程 lT（初始为 Null ），对往后固定个数(Java 5 Native 1)个 Java 线程进行采样
  - 6.1 采样线程将采样成功计数置为 0，将 Threads table 上锁，并获取 `JVM 线程列表`。
  - 6.2 采样线程 根据 上一次被采样的线程 lT ，找到` JVM 线程列表`中下一个线程 t（如果上次遍历完列表 或 这是第一次遍历，则从列表第一个线程开始），并将 lT 设置为 t
  - 6.3 如果线程 t 非空也非编译线程，则转到 6.4 尝试对该线程进行采样，否则转回到 6.2 继续寻找下一个被采样线程
  - 6.4 （通过线程 t 的 state 状态）判断当前被采样线程 t 是运行 Java 方法的线程还是运行 Native 方法的线程（这里分析的事件是 ExecutionSample 事件，只需要采样 运行 Java 方法的线程；否则调换6.4.1和6.4.2的判断条件），
    **注：**此时需要获得准确的线程状态！（细节参考文章 {% post_link UseMembar %}）
    - 6.4.1 如果线程 t 在运行 Java 方法，跳转到 6.5 继续采样
    - 6.4.2 如果线程 t 在运行 Native 方法，跳转回到 6.2 
  - 6.5 采样线程通过 pthread_kill 函数给被采样线程 t 发出异常信号，使线程 t 进入异常处理逻辑（异常处理逻辑提前被初始化为 SR_handler 函数）
    - 6.5.1 被采样线程 t 进异常处理函数时会获得传入的参数 ucontext_t（其中记录着线程 t 当前运行的程序上下文）。将上下文信息保存到线程 t 对象中后，线程 t 调用 sigsuspend 函数将自己暂停
    - 6.5.2 采样线程读取线程 t 的上下文信息 ucontext_t
  - 6.6 采样线程依据被采样线程 t 的上下文完成堆栈采样，采样线程发送 UNBLOCK 信号给被采样线程 t ，恢复被采样线程 t 的上下文。采样完成计数 + 1，并判断总采样成功计数是否大于阈值(Java 5 Native 1)。
    - 6.6.1 如果总采样成功计数小于阈值则转回到 6.2，继续采样
    - 6.6.2 如果总采样成功计数大于阈值则释放 Threads table 锁，然后转到步骤 7 
7. 采样线程一次采样结束（此时 lT 为最后被采样的线程），更新`上次采样结束的时间`（为当前时间，通过 clock_gettime 获得），并回第 3 步。

# JFR MethodProfiling Event 采样算法源码分析

## 方法采样频次控制

ExecutionSample 与 NativeMethodSample 事件属于采样事件，需要按照一定频率进行。 Hotspot 会起一个专门的线程按照一定频次完成采样任务，一次采样任务只会采集固定个数的指定类型线程，并记录最后采集的线程；等下一次采集任务发生时会根据上次采集的线程，继续采集固定个数的指定类型线程，知道线程队列都被采集过一遍，再从头开始采集。

接下来通过分析 OpenJDK11 源码的方式看一下。

```c++
//bishengjdk-11-mirror/src/hotspot/share/jfr/periodic/sampling/jfrThreadSampler.cpp
void JfrThreadSampler::run() {
  // 当前为新建的采样任务线程
  _sampler_thread = this;
  // 初始化变量，last_java_ms 与 last_native_ms 分别记录最后一次 java/native方法采样的时间
  jlong last_java_ms = get_monotonic_ms();
  jlong last_native_ms = last_java_ms;
  //死循环，采样将持续进行，直到当前应用程序结束或者手动关闭 JFR 记录
  while (true) {
    //尝试获取信号量锁，_semaphore > 0 => _semaphore -1 就不用进循环
    if (!_sample.trywait()) {
      // disenrolled
      _sample.wait();
      // 信号量 _semaphore -1 ，更新 last 时间
      last_java_ms = get_monotonic_ms();
      last_native_ms = last_java_ms;
    }
    // 设置进程信号量 _sample 的 _semaphore +1 
    _sample.signal();
    // 设置记录间隔
    jlong java_interval = _interval_java == 0 ? max_jlong : MAX2<jlong>(_interval_java, 1);
    jlong native_interval = _interval_native == 0 ? max_jlong : MAX2<jlong>(_interval_native, 1);
    // 设置当前时间
    jlong now_ms = get_monotonic_ms();
    // 还剩 next 时间可以开始下次的方法采样
    jlong next_j = java_interval + (last_java_ms - now_ms);
    jlong next_n = native_interval + (last_native_ms - now_ms);

    jlong sleep_to_next = MIN2<jlong>(next_j, next_n);
    // sleep_to_next > 0 还需等待，还不能开始
    if (sleep_to_next > 0) {
      os::naked_short_sleep(sleep_to_next);//短暂等待，更新 sleep_to_next
    }

    // <= 0 说明需要等待的时间为 "负" 可以开始采样
    // 相同的判断，分别开始采样 Java 方法 与 Native 方法
    // 最后更新 最后采样时间。
    if ((next_j - sleep_to_next) <= 0) {
      // 传入上一次采样的 Java 线程指针，传址调用，会在函数内部被修改为这次采样的最后一个线程
      task_stacktrace(JAVA_SAMPLE, &_last_thread_java);
      last_java_ms = get_monotonic_ms();
    }
    if ((next_n - sleep_to_next) <= 0) {
      task_stacktrace(NATIVE_SAMPLE, &_last_thread_native);
      last_native_ms = get_monotonic_ms();
    }
  }
  delete this;
}
```

方法采样频次的源码分析在上述源码中已经详细分析，不难看出 Java 方法的采样与 Native 方法的采样调用相同的方法，故此处只以 Java 方法的采样为例，进行分析。Native 方法的采样逻辑留待有兴趣的读者自行挖掘。

## Java 方法采样准备工作

上一节的源码部分实现的是固定间隔时间的采样频次控制，task_stacktrace 函数实现的是每次对固定个数线程反栈的操作。源码分析如下

```c++
//bishengjdk-11-mirror/src/hotspot/share/jfr/periodic/sampling/jfrThreadSampler.cpp
void JfrThreadSampler::task_stacktrace(JfrSampleType type, JavaThread** last_thread) {
  ResourceMark rm;
  // 方法采样事件组
  // MAX_NR_OF_JAVA_SAMPLES = 5 MAX_NR_OF_NATIVE_SAMPLES = 1
  // 每一次采样 Java 方法只采 5 个线程 Native 只采 1 个线程
  EventExecutionSample samples[MAX_NR_OF_JAVA_SAMPLES];
  EventNativeMethodSample samples_native[MAX_NR_OF_NATIVE_SAMPLES];
  JfrThreadSampleClosure sample_task(samples, samples_native);

  const uint sample_limit = JAVA_SAMPLE == type ? MAX_NR_OF_JAVA_SAMPLES : MAX_NR_OF_NATIVE_SAMPLES;
  uint num_samples = 0;
  JavaThread* start = NULL;

  {
    elapsedTimer sample_time;
    sample_time.start();
    { // 将 monitor Threads_lock 上锁，在括号外调用析构函数解锁
      MonitorLockerEx tlock(Threads_lock, Mutex::_allow_vm_block_flag);
      // Java 线程列表（包括在执行 Native 和 Java 方法的所有 JavaThread）
      ThreadsListHandle tlh;
      // Resolve a sample session relative start position index into the thread list array.
      // In cases where the last sampled thread is NULL or not-NULL but stale, find_index() returns -1.
      // 上一次被采样的线程在线程列表中的线程号（如果是从头遍历，*last_thread = NULL，_cur_index = -1）
      _cur_index = tlh.list()->find_index_of_JavaThread(*last_thread);
      JavaThread* current = _cur_index != -1 ? *last_thread : NULL;

      while (num_samples < sample_limit) {
        // 找到下一个要被采样的线程 
        //（如果 current 为 NULL 说明是一次新的遍历，current被更新为 0 号 Java 线程）
        current = next_thread(tlh.list(), start, current);
        // current == NULL 说明线程列表都被采样完了，这一次即使没采满指定线程数也要退出了
        if (current == NULL) {
          break;
        }
        // start == NULL 说明是从头开始的新遍历，需要初始化 start
        if (start == NULL) {
          start = current;  // remember the thread where we started to attempt sampling
        }
        // 被采样线程是编译线程也不采样
        if (current->is_Compiler_thread()) {
          continue;
        }
        // 对被采样线程进行反栈，反栈成功则采样数 +1
        if (sample_task.do_sample_thread(current, _frames, _max_frames, type)) {
          num_samples++;
        }
      }
      // 采样结束，将最后采样的线程设置为当前采样的线程
      *last_thread = current;  // remember the thread we last attempted to sample
    }
    sample_time.stop();
    log_trace(jfr)("JFR thread sampling done in %3.7f secs with %d java %d native samples",
                   sample_time.seconds(), sample_task.java_entries(), sample_task.native_entries());
  }
  if (num_samples > 0) {
    sample_task.commit_events(type);
  }
}
```

这一部分主要包括采样线程的确定，单个线程的采样过程又被包装在 do_sample_thread 函数内，do_sample_thread 函数完成对正在执行不同方法的 Java 线程进行分类（按 Native 和 Java 方法）

```c++
bool JfrThreadSampleClosure::do_sample_thread(JavaThread* thread, JfrStackFrame* frames, u4 max_frames, JfrSampleType type) {
  // 被采样线程无法被反栈的话就当没采到
  if (thread->is_hidden_from_external_view() || thread->in_deopt_handler()) {
    return false;
  }

  bool ret = false;
  // 将当前被采线程的 _suspendible_thread 状态原子的（cmpxchg）设置为 _trace_flag
  thread->set_trace_flag();
  // 对于虚拟屏障技术需要置换状态记录页的权限，确保 JavaThread 类的 _thread_state 状态是最新的 
  if (!UseMembar) {
    os::serialize_thread_states();
  }
  // 根据采样事件类型确定当前线程是否是需要被采样的
  if (JAVA_SAMPLE == type) {
    // 判断当前被采样的 JavaThread 的 _thread_state 是否为 Java 方法
    if (thread_state_in_java(thread)) {
      // 按照 Java 方法线程的方式进行反栈
      ret = sample_thread_in_java(thread, frames, max_frames);
    }
  } else {
    // 采样 Native 方法的情况
    if (thread_state_in_native(thread)) {
      ret = sample_thread_in_native(thread, frames, max_frames);
    }
  }
  clear_transition_block(thread);
  return ret;
}
```

sample_thread_in_java 函数除了包含反栈操作，还将得到的方法信息存储起来，并返回方法栈的在仓库中的 ID

```c++
bool JfrThreadSampleClosure::sample_thread_in_java(JavaThread* thread, JfrStackFrame* frames, u4 max_frames) {
  OSThreadSampler sampler(thread, *this, frames, max_frames);
  // 采样线程对被采样线程进行反栈获得函数调用栈
  sampler.take_sample();
  // 判断反栈是否成功
  if (!sampler.success()) {
    return false;
  }
  // 方法采样事件
  EventExecutionSample *event = &_events[_added_java - 1];
  // 将得到的函数调用栈信息存储到 Heap 上，获得一个 id。
  // Heap 上的信息会在程序退出或者根据启动 JFR 的 duration 参数 dump 到磁盘上。
  traceid id = JfrStackTraceRepository::add(sampler.stacktrace());
  // 方法采样时间记录 id
  event->set_stackTrace(id);
  return true;
}

void OSThreadSampler::take_sample() {
  run();
}

//class OSThreadSampler : public os::SuspendedThreadTask
void os::SuspendedThreadTask::run() {
  internal_do_task();
  _done = true;
}

//bishengjdk-11-mirror/src/hotspot/os/linux/os_linux.cpp
void os::SuspendedThreadTask::internal_do_task() {
  // suspend 被采样的线程
  if (do_suspend(_thread->osthread())) {
    // 获得线程 TaskContext
    SuspendedThreadTaskContext context(_thread, _thread->osthread()->ucontext());
    // 根据 TaskContext 获得函数调用栈信息
    do_task(context);
    // 恢复线程
    do_resume(_thread->osthread());
  }
}
```

可以看到，最后反栈的过程会通过四部曲完成 —— suspend 被采样线程 ,采样线程获得 ucontext, 采样线程 do_task 获得堆栈信息, 采样线程 resume 被采样线程

# 被采样的线程 suspend 与 resume 过程

```c++
//bishengjdk-11-mirror/src/hotspot/os/linux/os_linux.cpp
// 采样线程调用 do_suspend 函数尝试将被采样的线程 suspend
static bool do_suspend(OSThread* osthread) {

  // osthread->sr->_state 变为 SR_SUSPEND_REQUEST
  if (osthread->sr.request_suspend() != os::SuspendResume::SR_SUSPEND_REQUEST) {
    // failed to switch, state wasn't running?
    ShouldNotReachHere();
    return false;
  }
  // 向线程传入 signal 信号（信号与对应的信号处理后续补充）
  // pthread_kill(osthread->pthread_id(), SR_signum)
  if (sr_notify(osthread) != 0) {
    ShouldNotReachHere();
  }

  // managed to send the signal and switch to SUSPEND_REQUEST, now wait for SUSPENDED
  while (true) {
    // 尝试将全局对象 sr_semaphore 的信号量 _semaphore  -1
    if (sr_semaphore.timedwait(create_semaphore_timespec(0, 2 * NANOSECS_PER_MILLISEC))) {
      // 完成 -1，跳出循环
      break;
    } else {
      // 没完成即超过 2 ms的时间限制
      // timeout
      os::SuspendResume::State cancelled = osthread->sr.cancel_suspend();
      if (cancelled == os::SuspendResume::SR_RUNNING) {
        return false;
      } else if (cancelled == os::SuspendResume::SR_SUSPENDED) {
        // make sure that we consume the signal on the semaphore as well
        sr_semaphore.wait();
        break;
      } else {
        ShouldNotReachHere();
        return false;
      }
    }
  }

  guarantee(osthread->sr.is_suspended(), "Must be suspended");
  return true;
}

// pthread_kill 传递信号后，被采样的线程会进入异常处理函数，context 中会保留 CPU 上下文
static void SR_handler(int sig, siginfo_t* siginfo, ucontext_t* context) {
  // Save and restore errno to avoid confusing native code with EINTR
  // after sigsuspend.
  int old_errno = errno;
  Thread* thread = Thread::current_or_null_safe();

  OSThread* osthread = thread->osthread();

  os::SuspendResume::State current = osthread->sr.state();
  // 状态会在do_suspend 函数内被设置为 SR_SUSPEND_REQUEST
  if (current == os::SuspendResume::SR_SUSPEND_REQUEST) {
    // 将上下文信息保存到被采样的线程对象中去
    suspend_save_context(osthread, siginfo, context);

    // attempt to switch the state, we assume we had a SUSPEND_REQUEST
    // 准备将当前被采样线程暂停，需要将 state 变为 SR_SUSPENDED
    os::SuspendResume::State state = osthread->sr.suspended();
    if (state == os::SuspendResume::SR_SUSPENDED) {
      sigset_t suspend_set;  // signals for sigsuspend()
      sigemptyset(&suspend_set);
      // get current set of blocked signals and unblock resume signal
      pthread_sigmask(SIG_BLOCK, NULL, &suspend_set);
      sigdelset(&suspend_set, SR_signum);

      sr_semaphore.signal();
      // wait here until we are resumed
      while (1) {
        // 暂停线程，等待 UNBLOCK 信号唤醒
        sigsuspend(&suspend_set);

        os::SuspendResume::State result = osthread->sr.running();
        if (result == os::SuspendResume::SR_RUNNING) {
          sr_semaphore.signal();
          break;
        }
      }

    } else if (state == os::SuspendResume::SR_RUNNING) {
      // request was cancelled, continue
    } else {
      ShouldNotReachHere();
    }
    // 恢复运行，清楚记下的上下文
    resume_clear_context(osthread);
  } else if (current == os::SuspendResume::SR_RUNNING) {
    // request was cancelled, continue
  } else if (current == os::SuspendResume::SR_WAKEUP_REQUEST) {
    // ignore
  } else {
    // ignore
  }

  errno = old_errno;
}

// 采样线程完成采样后，需要调用 do_resume 函数，使被采样线程恢复运行
static void do_resume(OSThread* osthread) {
  // 将被采样线程的 state 设置为 SR_WAKEUP_REQUEST
  if (osthread->sr.request_wakeup() != os::SuspendResume::SR_WAKEUP_REQUEST) {
    // failed to switch to WAKEUP_REQUEST
    ShouldNotReachHere();
    return;
  }

  while (true) {
    // 再次向被采样线程传递信号，恢复被采样线程的运行状态
    if (sr_notify(osthread) == 0) {
      if (sr_semaphore.timedwait(create_semaphore_timespec(0, 2 * NANOSECS_PER_MILLISEC))) {
        if (osthread->sr.is_running()) {
          return;
        }
      }
    } else {
      ShouldNotReachHere();
    }
  }

}
```
# 采样线程反栈

通过信号暂停了被采样线程后，同时还获取到被采样线程的上下文信息，并且这部分上下文信息被记录到被采样线程对象中去。此时采样线程将通过这部分上下文信息完成反栈工作：

```c++
void OSThreadSampler::do_task(const os::SuspendedThreadTaskContext& context) {

  _suspend_time = JfrTicks::now();
  
  if (JfrOptionSet::sample_protection()) {
    OSThreadSamplerCallback cb(*this, context);
    os::ThreadCrashProtection crash_protection;
    // call 方法传入对象 cb ，进行反栈
    if (!crash_protection.call(cb)) {
      log_error(jfr)("Thread method sampler crashed");
    }
  } else {
    protected_task(context);
  }
}

//src/hotspot/os/posix/os_posix.cpp
bool os::ThreadCrashProtection::call(os::CrashProtectionCallback& cb) {
  sigset_t saved_sig_mask;
  // 获取 _crash_mux 锁
  Thread::muxAcquire(&_crash_mux, "CrashProtection");
  // ...
  if (sigsetjmp(_jmpbuf, 0) == 0) {
    // make sure we can see in the signal handler that we have crash protection
    // installed
    _crash_protection = this;
    cb.call();
    //...
  }
  //...
}
//src/hotspot/share/jfr/periodic/sampling/jfrThreadSampler.cpp
// 最终方法采样调用的是 OSThreadSamplerCallback 类的 call 方法
class OSThreadSamplerCallback : public os::CrashProtectionCallback {
  //...
  virtual void call() {
    // 最终的反栈函数
    _sampler.protected_task(_context);
  }
  //...
};

void OSThreadSampler::protected_task(const os::SuspendedThreadTaskContext& context) {
  // 被采样线程
  JavaThread* jth = (JavaThread*)context.thread();
  // Skip sample if we signaled a thread that moved to other state
  if (!thread_state_in_java(jth)) {
    return;
  }
  JfrGetCallTrace trace(true, jth);
  // 存储栈顶方法的栈信息 fp pc sp sender_sp
  frame topframe;
  // 获取被采样线程的函数调用栈的栈顶，放入 topframe，确保栈顶有方法
  if (trace.get_topframe(context.ucontext(), topframe)) {
    // 根据栈顶函数状态和被采样线程指针将栈信息保存到线程私有的内存内
    if (_stacktrace.record_thread(*jth, topframe)) {
      /* If we managed to get a topframe and a stacktrace, create an event
      * and put it into our array. We can't call Jfr::_stacktraces.add()
      * here since it would allocate memory using malloc. Doing so while
      * the stopped thread is inside malloc would deadlock. */
     // jfr event 记录
      _success = true;
      EventExecutionSample *ev = _closure.next_event();
      ev->set_starttime(_suspend_time);
      ev->set_endtime(_suspend_time); // fake to not take an end time
      ev->set_sampledThread(JFR_THREAD_ID(jth));
      ev->set_state(java_lang_Thread::get_thread_status(jth->threadObj()));
    }
  }
}

```

至此，JFR 方法采样工作只剩下反栈的具体操作了，这部分将在{% post_link JVM栈与反栈 %}中详细展开

**补充：** Native 方法的采样与 Java 方法采样存在以下不同：
- 采样结果方面： Native 方法采样不会在 JMC 的 Method Profiling 页面展示