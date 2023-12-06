---
title: JFR可靠性分析
date: 2023-05-15 15:44:27
tags:
- Hotspot
- Java
- JDK Flight Recorder
---

# JFR Event 记录源码分析

JDK Flight Recorder ( JFR ) 记录以 Event 为单位，每个 Event 都有`事件类型`，`开始时间`，`结束时间`，`发生事件的线程`，`事件发生的线程堆栈`还有`Event 数据体组成`。<!--more-->

下面我们通过源码的方式理解 JFR 记录的过程

# 开启记录并提交

以 JavaMonitorEnter 为例，首先查看 JavaMonitorEnter 的 Event 数据体（大多数 Event 的数据体描述都记录在 metadata.xml 文件中，category="Java Development kit" 的 Event 数据体不在里面 ）

```xml
<!--bishengjdk-11-mirror/src/hotspot/share/jfr/metadata/metadata.xml-->
  <Event name="JavaMonitorEnter" category="Java Application" label="Java Monitor Blocked" thread="true" stackTrace="true">
    <Field type="Class" name="monitorClass" label="Monitor Class" />
    <Field type="Thread" name="previousOwner" label="Previous Monitor Owner" />
    <Field type="ulong" contentType="address" name="address" label="Monitor Address" relation="JavaMonitorAddress" />
  </Event>

```
通过 metadata.xml 中的描述可知 JavaMonitorEnter Event 的数据体包括 `monitorClass` `previousOwner` `address`，相应的描述也在 metadata.xml 中。当开启 JFR 对 JavaMonitorEnter Event 采集时，会在 ObjectMonitor::enter 函数内额外执行以下部分

```c++
//bishengjdk-11-mirror/src/hotspot/share/runtime/objectMonitor.cpp
void ObjectMonitor::enter(TRAPS) {
    //...
    EventJavaMonitorEnter event;
    if (event.should_commit()) {
    event.set_monitorClass(((oop)this->object())->klass());
    event.set_address((uintptr_t)(this->object_addr()));
    }
    //...
    if (event.should_commit()) {
    event.set_previousOwner((uintptr_t)_previous_owner_tid);
    event.commit();
    }

```

代码抽象出来看其实包括4个部分：

1. 某个 Event 对象初始化
2. 判断是否开启某个 Event 的检测
3. 设置某个 Event 对应的`数据体`
4. event.commit()

这部分代码首先回答了`Event 数据体`信息的采集过程，另一方面 commit 会完成剩余的`开始时间`，`结束时间`，`发生事件的线程`，`事件发生的线程堆栈`信息的采集（`事件类型`由 Event 类确定），下面通过源码方式理解

```c++
//bishengjdk-11-mirror/build/linux-x86_64-normal-server-release/hotspot/variant-server/gensrc/jfrfiles/jfrEventClasses.hpp
class EventJavaMonitorEnter : public JfrEvent<EventJavaMonitorEnter>
{
    //初始化对象，当timing=TIMED时，需要记下当前时刻的时间作为测试开始时间
    EventJavaMonitorEnter(EventStartTime timing=TIMED) : JfrEvent<EventJavaMonitorEnter>(timing) {}
    //...
    //使用父类的commit函数
    using JfrEvent<EventJavaMonitorEnter>::commit;
    //...
}
//bishengjdk-8/hotspot/src/share/vm/jfr/recorder/service/jfrEvent.hpp
template <typename T>
class JfrEvent {
  private:
  jlong _start_time;
  jlong _end_time;
  bool _started;

  protected:
  //初始化函数，当TIMED == timing && !T::isInstant时会设置当前 Event 对象的开始时间（为当前）
  JfrEvent(EventStartTime timing=TIMED) : _start_time(0), _end_time(0), _started(false)  {
    if (T::is_enabled()) {
      _started = true;
      if (TIMED == timing && !T::isInstant) {
        set_starttime(JfrTicks::now());
      }
    }
  }
  //当完成 Event 数据体采集后调用 commit 函数
  void commit() {
    //判断是否需要commit
    if (!should_commit()) {
      return;
    }
    // 针对不同 Event 有不同采集需求，有的需要同时记录开始结束时间，有的只需要一个开始时间
    // 如果初始化没有设置 _start_time，则 _start_time 为当前时间
    if (_start_time == 0) {
      set_starttime(JfrTicks::now());
    } else if (_end_time == 0) {
    //正常情况下在初始化对象时设置完_start_time，在commit函数内设置_endtime为当前时间
      set_endtime(JfrTicks::now());
    }
    // 记录线程信息与堆栈信息
    if (should_write()) {
      write_event();
    }
  }
  //...
  void write_event() {
  //获取当前线程  
  Thread* const event_thread = Thread::current();
  //获取当前线程的 jfr 的线程私有信息
  JfrThreadLocal* const tl = event_thread->jfr_thread_local();
  //找到线程私有内存中的 jfr buffer
  JfrBuffer* const buffer = tl->native_buffer();
  //buffer 为空直接返回
  if (buffer == NULL) {
    // most likely a pending OOM
    return;
  }
  JfrNativeEventWriter writer(buffer, event_thread);
  //将当前 Event 的Id写入buffer
  writer.write<u8>(T::eventId);
  //将当前 Event 的开始时间写入buffer
  writer.write(_start_time);
  if (!(T::isInstant || T::isRequestable) || T::hasCutoff) {
  //将当前 Event 的持续时间写入buffer
    writer.write(_end_time - _start_time);
  }
  if (T::hasThread) {
  //将当前 Event 的线程号写入buffer
    writer.write(tl->thread_id());
  }
  //将当前 Event 的堆栈写入buffer
  if (T::hasStackTrace) {
    if (is_stacktrace_enabled()) {
    if (tl->has_cached_stack_trace()) {
      writer.write(tl->cached_stack_trace_id());
    } else {
      writer.write(JfrStackTraceRepository::record(event_thread));
    }
    } else {
        writer.write<traceid>(0);
      }
    }
  // payload
  static_cast<T*>(this)->writeData(writer);
  }
}
```

通过上述方式，将 Event 的`事件类型`，`开始时间`，`持续时间`，`发生事件的线程`，`事件发生的线程堆栈`还有`Event 数据体组成`都写入线程私有的 Buffer 内存中，后续会在 buffer 满之后，将 buffer 内的数据提交到 global buffer 内。

# 时间获取的方式

通过上文的源码分析可以发现，JFR 获取时间戳的方式为 JfrTicks::now() 函数

```c++
//bishengjdk-11-mirror/src/hotspot/share/jfr/utilities/jfrTime.hpp
typedef TimeInstant<CounterRepresentation, FastUnorderedElapsedCounterSource> JfrTicks;
//bishengjdk-11-mirror/src/hotspot/share/utilities/ticks.hpp
template <template <typename> class Rep, typename TimeSource>
class TimeInstant : public Rep<TimeSource> {
    //...
  void stamp() {
    this->_rep = TimeSource::now();
  }
  static TimeInstant<Rep, TimeSource> now() {
    TimeInstant<Rep, TimeSource> temp;
    temp.stamp();
    return temp;
  }
};
//bishengjdk-11-mirror/src/hotspot/share/utilities/ticks.cpp
FastUnorderedElapsedCounterSource::Type FastUnorderedElapsedCounterSource::now() {
#if defined(X86) && !defined(ZERO)
  static bool valid_rdtsc = Rdtsc::initialize();
  if (valid_rdtsc) {
    return Rdtsc::elapsed_counter();
  }
#endif
  return os::elapsed_counter();
}
```
不难看出 x86 对应 JVM 获取时间的方式是通过调用 Rdtsc::elapsed_counter() 函数获取，而非 x86 操作系统对应的 JVM 获取时间的方式是通过调用 os::elapsed_counter() 函数实现的。

os::elapsed_counter()函数最终依赖的是操作系统库函数的 clock_gettime 函数

```c++
//bishengjdk-11-mirror/src/hotspot/os/linux/os_linux.cpp
jlong os::elapsed_counter() {
  return javaTimeNanos() - initial_time_count;
}
//bishengjdk-11-mirror/src/hotspot/os/linux/os_linux.cpp
jlong os::javaTimeNanos() {
  //return Linux::_clock_gettime != NULL;
  if (os::supports_monotonic_clock()) {
    struct timespec tp;
    int status = Linux::clock_gettime(CLOCK_MONOTONIC, &tp);
    jlong result = jlong(tp.tv_sec) * (1000 * 1000 * 1000) + jlong(tp.tv_nsec);
    return result;
  } else {
    //...
  }
}
//bishengjdk-11-mirror/src/hotspot/os/linux/os_linux.hpp
static int clock_gettime(clockid_t clock_id, struct timespec *tp) {
return _clock_gettime ? _clock_gettime(clock_id, tp) : -1;
}
//bishengjdk-11-mirror/src/hotspot/os/linux/os_linux.cpp
void os::Linux::clock_init() {
  void* handle = dlopen("librt.so.1", RTLD_LAZY);
  if (handle == NULL) {
    handle = dlopen("librt.so", RTLD_LAZY);
  }

  if (handle) {
    int (*clock_getres_func)(clockid_t, struct timespec*) =
           (int(*)(clockid_t, struct timespec*))dlsym(handle, "clock_getres");
    int (*clock_gettime_func)(clockid_t, struct timespec*) =
           (int(*)(clockid_t, struct timespec*))dlsym(handle, "clock_gettime");
    if (clock_getres_func && clock_gettime_func) {
      struct timespec res;
      struct timespec tp;
      if (clock_getres_func (CLOCK_MONOTONIC, &res) == 0 &&
          clock_gettime_func(CLOCK_MONOTONIC, &tp)  == 0) {
        // yes, monotonic clock is supported
        _clock_gettime = clock_gettime_func;
        return;
      } else {
        // close librt if there is no monotonic clock
        dlclose(handle);
      }
    }
  }
}
```

 x86 对应 JVM 的 Rdtsc::elapsed_counter() 函数最终通过指令 __rdtsc获取时间周期数
 
```c++
//bishengjdk-11-mirror/src/hotspot/cpu/x86/rdtsc_x86.cpp
bool Rdtsc::initialize() {
  static bool initialized = false;
  if (!initialized) {
     VM_Version_Ext::initialize();
    bool result = initialize_elapsed_counter(); // init hw
    if (result) {
      result = ergonomics(); // check logical state
    }
    rdtsc_elapsed_counter_enabled = result;
    initialized = true;
  }
  return rdtsc_elapsed_counter_enabled;
}

jlong Rdtsc::elapsed_counter() {
  return os::rdtsc() - _epoch;
}
//bishengjdk-11-mirror/src/hotspot/os_cpu/linux_x86/os_linux_x86.inline.hpp
inline jlong os::rdtsc() {
#ifndef AMD64
  // 64 bit result in edx:eax
  uint64_t res;
  __asm__ __volatile__ ("rdtsc" : "=A" (res));
  return (jlong)res;
#else
  uint64_t res;
  uint32_t ts1, ts2;
  __asm__ __volatile__ ("rdtsc" : "=a" (ts1), "=d" (ts2));
  res = ((uint64_t)ts1 | (uint64_t)ts2 << 32);
  return (jlong)res;
#endif // AMD64
}
```