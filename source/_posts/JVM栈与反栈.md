---
title: JVM栈与反栈
date: 2023-06-16 09:29:29
tags:
- Hotspot
- Java
- JDK Flight Recorder
- Stack frame
---

本文的出发点是分析 JFR 反栈原理，而理解反栈的前提是理解 JVM 栈结构。<!--more-->
写作本文的最初起点是希望理解 JFR MethodProfiling event 对 Java 方法采样的原理（采样过程可以参考{% post_link JFR-MethodProfiling %}）。经过分析采样过程，笔者希望与读者构建分析反栈的前置条件:

- 采样线程已发送信号使被采样线程阻塞，且拿到了被采样线程的上下文
- 采样线程在被采样线程阻塞之后，对被采样线程进行反栈
- 采样线程完成采样后，能使被采样线程回到阻塞发生的线程继续执行
- JFR MethodProfiling even 只会对 Java 方法完成采样

在这样的前提下，首先建立反栈的基础逻辑：

1. 获得正在执行的 Java 方法的栈
2. 获得当前 Java 方法名
3. 找到当前方法的父栈
  3.1 如果找到父栈则，回到 2 继续执行；
  3.2 否则结束反栈

遗留的问题也就是本文分析的问题：

1. 如何根据当前栈信息获得当前栈的方法（包括栈底定位）
2. 如何完成对 JVM 栈的反栈
3. 如何只对 Java 方法采样

# 简单认识JVM栈

本文的目的是通过反栈得到函数调用栈，所以在这里简单介绍一下反栈需要用到的栈信息

```c++
class frame {
 private:
  //src/hotspot/share/runtime/frame.hpp
  // Instance variables:
  intptr_t* _sp; // stack pointer (from Thread::last_Java_sp)
  address   _pc; // program counter (the next instruction after the call)
 
  //src/hotspot/cpu/x86/frame_x86.hpp
  intptr_t*   _fp; // frame pointer
  
 //成员函数省略...
}
```
通过上述定义，我们知道想要准确定位一个栈至少需要三个信息：
- 内存栈的栈顶（sp，低地址）
- 程序执行到哪？（pc）
- 内存栈的栈底（fp，高地址）

# 反栈源码实现

反栈的过程可以简单概括为以下 4 步：

1. 判断当前栈是不是已经是栈底了，如果是则停止反栈，否则进入 2
2. 找到当前栈的方法
3. 通过栈内保存的信息找到父栈
4. 将当前栈设置为父栈，并回到 1

代码逻辑如下

```c++
//src/hotspot/share/jfr/recorder/stacktrace/jfrStackTrace.cpp
bool JfrStackTrace::record_thread(JavaThread& thread, frame& frame) {
  // 构造函数初始化 _method 获取当前所在栈的方法
  vframeStreamSamples st(&thread, frame, false);
  u4 count = 0;
  _reached_root = true;

  _hash = 1;
  // 遍历整个栈，while 直到栈底
  while (!st.at_end()) {
    // 设置一个最大反栈深度，超过了也会结束反栈
    if (count >= _max_frames) {
      _reached_root = false;
      break;
    }
    // 拿到栈的当前方法 method
    const Method* method = st.method();
    // 确保方法有效（method 对象的 first word == vtbl pointer 得是虚函数表指针）
    if (!Method::is_valid_method(method)) {
      // we throw away everything we've gathered in this sample since
      // none of it is safe
      return false;
    }
    // 生出并获取 method 唯一编号
    const traceid mid = JfrTraceId::use(method);
    // 获得 method 的类型（解释/编译）
    int type = st.is_interpreted_frame() ? JfrStackFrame::FRAME_INTERPRETER : JfrStackFrame::FRAME_JIT;
    int bci = 0;
    // 如果当前方法是 Native 则将类型设置为 Native
    if (method->is_native()) {
      type = JfrStackFrame::FRAME_NATIVE;
    } else {
      // 否则获得相对该方法开始处的偏移量
      bci = st.bci();
    }
    const int lineno = method->line_number_from_bci(bci);
    // Can we determine if it's inlined?
    _hash = (_hash * 31) + mid;
    _hash = (_hash * 31) + bci;
    _hash = (_hash * 31) + type;
    // 将栈信息存入待返回的方法栈列表
    _frames[count] = JfrStackFrame(mid, bci, type, lineno, method);
    // 获取父方法的栈，同时获得父方法的方法放入 _method
    st.samples_next();
    count++;
  }

  _lineno = true;
  _nr_of_frames = count;
  return true;
}
```

需要关注的重点有两处：

- 不同类型栈如何找到对应方法
- 不同情况如何找到父栈

# 不同类型栈定位 method

Hotspot 中通过 fill_from_frame() 函数实现：

```c++
//src/hotspot/share/runtime/vframe.inline.hpp
inline bool vframeStreamCommon::fill_from_frame() {
  // 通过 pc 定位 是否在解释器范围 Interpreted frame
  if (_frame.is_interpreted_frame()) {
    // 解释栈定位方法
    fill_from_interpreter_frame();
    return true;
  }

  // Compiled frame
  // _cb 非空且 _cb 指向的是 CodeCache （_cb 根据 pc 获得）
  // JNI 与 JIT 都在 CodeCache 中
  if (cb() != NULL && cb()->is_compiled()) {
    // nm() 根据 pc 获取包含 pc 的 CodeBlob，再将 codeBlob 包装为 CompileMethod 类的对象
    // _method 是否为 Native
    if (nm()->is_native_method()) {
      // Do not rely on scopeDesc since the pc might be unprecise due to the _last_native_pc trick.
      // Native栈定位方法
      fill_from_compiled_native_frame();
    } else {
      PcDesc* pc_desc = nm()->pc_desc_at(_frame.pc());
      int decode_offset;
      // Should not happen...
      decode_offset = pc_desc->scope_decode_offset();
      // 编译栈定位方法 将 _mode 写为 compiled_mode
      fill_from_compiled_frame(decode_offset);
    }
    return true;
  }

  // End of stack?
  // 判断是否反栈到栈底了
  if (_frame.is_first_frame() || (_stop_at_java_call_stub && _frame.is_entry_frame())) {
    _mode = at_end_mode;
    return true;
  }

  return false;
}
```

从上述代码不难看出，定位栈对应方法的过程需要分为`解释栈`、`编译栈`、`Native方法栈`三种情况分析
（假设我们已经有当前栈的 sp fp pc）

## 解释栈定位方法

以 x86 的 Hotspot 为例，解释栈结构如下

![x86 解释栈结构](img/%E5%8F%8D%E6%A0%88/iframe.png)

通过当前栈的 fp 偏移 interpreter_frame_method_offset 地址，得到指向当前栈的 Method 的指针。

## Native 方法栈定位方法

x86 的 Hotspot 中 Native 方法存储在 CodeCache 上，只需要通过 pc 值找到 CodeCache 上包含 pc 值的 CompiledMethod 对象，就能找到对应的 Native 方法名。

## 编译定位方法

当发现当前栈既非解释栈，也不是 Native 方法栈，此时可以认定当前栈为编译栈。与 Native 方法定位不同，通过 pc 值定位方法在 CodeCache 上的位置，再通过偏移读取的方法名是包含了内敛方法的最终方法。JFR 反栈工作需要将内敛的部分也剥离出来，此时就需要依赖 CodeCache 上存储的 debug 信息实现如下

![编译栈结构](img/%E5%8F%8D%E6%A0%88/cframe.png)

如上图可知，编译方法存储了 pcs 信息。pcs 存储的是一系列 PcDesc 类的对象，通过打印参数 -XX:+PrintAssembly 可以看到某个方法的 debug 信息

![编译栈结构](img/%E5%8F%8D%E6%A0%88/pcs.png)

```c++
class PcDesc {
 private:
  int _pc_offset;           // offset from start of nmethod
  int _flags;
  // ...
}
```

图中 pc 值即为 CodeCache 中的某段指令的开始地址；offset=_pc_offset ，为这段指令的开始地址基于当前方法开始的偏移；bits=_flags，为这段指令的结果状态。下一行 `*::*` :: 前表示这段指令对应所在的类，:: 后表示这段指令对应所在的方法；`@`后为这段指令对应所在方法的字节码；括号内为这段指令对应所在方法的字节码最佳匹配源码的行号。

了解了以上内容后，当我们需要对编译栈定位方法时，只需要根据当前 pc 查找所在代码段的 pcs，再根据 pcs 记录的方法，就能得到当前编译栈的方法（同时能够记录内敛方法）。

## 栈底定位

判断当前是否为栈底的依据为同时满足如下两条：

- 当前栈为 entry_frame （判断依据 当前栈的 pc == _call_stub_return_address，即当前栈回到了 generate_call_stub 内的 return 部分）
- 当前的 entry_frame 栈是进入 Java 世界的第一个 entry_frame （判断依据 call_wrapper 中记录的 锚 anchor 的 _last_Java_sp == NULL）

结合马智的 call_stub 栈来理解一下反栈的逻辑，如下图：

![entry frame结构](img/%E5%8F%8D%E6%A0%88/entryframe.png)

图中被调用的 Java 方法的栈中存储的 return address 指向 generate_call_stub 生成的代码段的返回地址处，通过old fp ， old sp 将栈返回至 CallStub 函数栈，也就是 entry_frame。在 entry_frame 中可以拿到 call wrapper 的指针，我们知道 C 调 Java 是通过 call_helper 函数再通过函数指针方式调用 CallStub 函数完成的，如下

```c++
JavaCallWrapper link(method, receiver, result, CHECK);

      StubRoutines::call_stub()(
        (address)&link,
        // (intptr_t*)&(result->_value), // see NOTE above (compiler problem)
        result_val_address,          // see NOTE above (compiler problem)
        result_type,
        method(),
        entry_point,
        parameter_address,
        args->size_of_parameters(),
        CHECK
      );
```

这里的 call wrapper 就是 link 对象。根据源码注释，我们能够知道 JavaCallWrapper 在每个 JavaCall 之前构造，并在调用之后进行销毁。它的目的是分配一个新的句柄块，并保存恢复最后一个 Java 的 fp 和 sp。而 link 对象会创建一个 JavaFrameAnchor 类的对象 _anchor 用来记录上一个 Java 栈的 sp fp pc 值。文字描述比较抽象，考虑如下情况：

![anchor 锚](img/%E5%8F%8D%E6%A0%88/anchor.png)

如上图所示，
- 开始时。线程 Thread 的last_Java_sp、last_Java_fp、last_Java_pc 值为 NULL
- 首次进入 Java 世界时，CallStub 创建 entry_frame 存储 JavaCallWrapper 对象指针，JavaCallWrapper 对象中的成员 _anchor 锚记录的 last_Java_sp、last_Java_fp、last_Java_pc 都复制 Thread 中的值，都为为空（据此可以判断当前 entry frame 为栈底）。
- 进入 Java 世界后，运行 Java 方法（同步创建 Java栈）
- Java 调用 Native 方法，此时需要经过 Native_entry/wrapper，会通过 set_last_Java_frame 将 Thread 的last_Java_sp、last_Java_fp、last_Java_pc 值记为上一个 Java 栈的 sp fp 与 Java 方法 pc。
- 此时 Native 方法再次调用 CallStub 进入 Java，则需要再次创建 JavaCallWrapper 对象，此时 JavaCallWrapper 对象中的成员 _anchor 锚 依旧复制 Thread 中的值，指向的是进入 Native 之前的最后一个 Java 栈 fp sp 和对应 Java 方法的 pc。

等反栈确定当前为栈底后，设置 _mode = at_end_mode ，反栈函数退出 while 循环，结束反栈。

# 不同情况定位父栈

通过上面的分析可知，在拿到栈信息后可以据此拿到栈的方法名，剩下的工作就只有如何根据当前栈找到父栈了
源码的实现逻辑如下

```c++
//src/hotspot/share/jfr/recorder/stacktrace/jfrStackTrace.cpp
void vframeStreamSamples::samples_next() {
  // handle frames with inlining
  // 处理编译方法的内敛情况
  if (_mode == compiled_mode &&
    vframeStreamCommon::fill_in_compiled_inlined_sender()) {
    return;
  }

  // handle general case
  u4 loop_count = 0;
  u4 loop_max = MAX_STACK_DEPTH * 2;
  do {
    loop_count++;
    // By the time we get here we should never see unsafe but better safe then segv'd
    // 确保还有父栈
    if (loop_count > loop_max || !_frame.safe_for_sender(_thread)) {
      _mode = at_end_mode;
      return;
    }
    // 查找父栈
    _frame = _frame.sender(&_reg_map);
  } while (!fill_from_frame());
}
//src/hotspot/cpu/x86/frame_x86.cpp
frame frame::sender(RegisterMap* map) const {
  map->set_include_argument_oops(false);
  if (is_entry_frame())       return sender_for_entry_frame(map);
  if (is_interpreted_frame()) return sender_for_interpreter_frame(map);
  if (_cb != NULL) {
    return sender_for_compiled_frame(map);
  }
  // Must be native-compiled frame, i.e. the marshaling code for native
  // methods that exists in the core system.
  return frame(sender_sp(), link(), sender_pc());
}
```
定位父栈的情况一共需要分为上述 4 种情况：当前栈为 `入口栈`、`解释栈`、`编译栈`、`native 方法栈`

## entry_frame 定位父栈

此时的 Java 方法由 C 函数 call_helper 调用，定位父栈的操作变为跳转到锚 anchor 栈上。
anchor 的[ sp fp pc ]被初始化为空。父栈结构设置为 sp=NULL fp=NULL pc=RA 此处的RA为当前栈 fp[-1] 即 C 方法的程序返回 pc 处。如果此时所处的 entry frame 不是第一个，则返回到上一个 Java 栈上（结合栈底定位理解）。

```c++
frame frame::sender_for_entry_frame(RegisterMap* map) const {
  // 通过 call wrapper 获取锚 
  JavaFrameAnchor* jfa = entry_frame_call_wrapper()->anchor();
  // 锚的 pc=sp=fp=NULL walkable=false 此时是栈底，否则锚记录上个 Java 栈信息
  if (!jfa->walkable()) {
    // Capture _last_Java_pc =(address)_last_Java_sp[-1]
    jfa->capture_last_Java_pc();
  }
  map->clear();
  // 构造父栈信息
  frame fr(jfa->last_Java_sp(), jfa->last_Java_fp(), jfa->last_Java_pc());
  return fr;
}

```


## interpreted_frame 定位父栈

当前为解释栈时，由于解释栈中记录了上一个栈的栈顶 sender_sp 、上一个栈的栈底、程序返回地址 pc 。且这些信息只需要当前栈 fp 的值进行偏移即可获取，如下图：

![x86 解释栈结构](img/%E5%8F%8D%E6%A0%88/iframeb.png)

通过获取上个栈的 pc sp fp 即可定位父栈。

## compiled_frame 定位父栈

当前为编译方法栈，以 x86 为例，编译方法栈的栈底会存上一个栈的 fp，栈顶存着当前栈的返回 pc，栈结构大体如下

```c++
//   [return pc             ]   <- sp   
//   [arg6                  ]
//   [arg7                  ]
//   [arg8                  ] 
//   [arg9                  ]
//   ...
//   [old frame pointer     ]   <- fp     
```

源码分析如下：

```c++
frame frame::sender_for_compiled_frame(RegisterMap* map) const {
  // unextended_sp() 当前栈的 sp，sender_sp 为父栈的 sp
  intptr_t* sender_sp = unextended_sp() + _cb->frame_size();
  intptr_t* unextended_sp = sender_sp;

  // 在 Interl 上父栈的返回 pc 会被存在父栈的栈顶
  address sender_pc = (address) *(sender_sp-1);

  // 更新 map ...
  // 返回获得的父栈
  return frame(sender_sp, unextended_sp, *saved_fp_addr, sender_pc);
}
```

## native 方法栈 定位父栈

此时 _cb == NULL 且不是 entry_frame、解释栈、编译栈。此时需要找到 Native 方法栈上存储的 sender_sp、fp、sender_pc。以 x86 为例，C 方法栈的结构大体如下：

![x86_64栈帧结构](img/%E5%8F%8D%E6%A0%88/x86_64%E6%A0%88%E5%B8%A7%E7%BB%93%E6%9E%84.png)

可以通过当前栈的栈底 -1 获得父栈的 pc，而当前 C 栈的栈底存着父栈的栈顶，当前的栈底作为父栈的栈顶。

```c++
frame(sender_sp(), link(), sender_pc());
```

**补充： 初始栈的获取**

当 JFR 采样线程发送用户信号将被采样的目标线程暂停时，目标线程可能停在不同类型的栈中。为了正确完成反栈，需要定位到 编译栈、解释栈、entry_frame、Native 栈。如果根据pc确定所在栈类型，确定停在 stub 等特殊栈时，就不进行反栈，采样下一个线程。