---
title: Hotspot源码分析(G1)——write_barrier
date: 2023-1-15 16:56:08
tags:
- G1GC-write barrier 
- Java
---
# 前言
本文将针对[ G1GC 算法篇](/2022/12/20/G1GC%E7%AE%97%E6%B3%95%E7%AF%87)中提到的 write barrier 技术基于源码进行分析，如有偏差，请指正。<!--more-->

G1 SATB 与 Rset维护都会调用到 write-barrier （ SATB 采用 pre write-barrier 完成，维护 RSet 采用 post write-barrier 完成） mutator 在 write-barrier 中仅仅将要做的事推送到队列中，然后通过另外的线程取出队列中的信息批量完成剩余的动作。

本文将分析 write-barrier 在OpenJDK8源码中解释器与C2编译中的实现。

#  write-barrier 在模板解释器的实现

在 G1GC 算法篇介绍两种 write-barrier 时提过， SATB 写屏障的目的是当并发标记时避免`漏标`，而维护的 Rset 记录的则是`跨界引用`。`漏标的产生`和`跨界引用的修改`根本原因都与对象的域被修改有关。查看如下例子：

```java
//情况 1 ：假设 G1 正在并发标记，对象 t 作为根对象已经被标记（灰色），但是 t 引用的对象还没被标记
//情况 2 ：假设对象 young 和 t 分别在不同的 Region 中
Node young = new Node();//步骤 1
t.next = young;//步骤 2
//上述两种情况都需要在步骤 2 完成前记录下相关信息
```

例子转换成字节码的核心部分如下，字节码的阅读不作详细解答，可查阅相关文档

```java
[26241] static void WriteBarrierShow.xxx(jobject, jint)
[26241]   769225     0  new 2 <Node>
[26241]   769226     3  dup
[26241]   769227     4  iload_1
[26241]   769228     5  invokespecial 3 <Node.<init>(I)V> 

//... virtual void Node.<init>(jint)

[26241] static void WriteBarrierShow.xxx(jobject, jint)
[26241]   769236     8  astore_2
[26241]   769237     9  aload_0
[26241]   769238    10  aload_2
[26241]   769239    11  putfield 4 <Node.p/LNode;> 
[26241]   769240    14  return

```

当解释执行时，会通过字节码 putfield 实现步骤2 ，所以来看一下 putfield 在模板解释器中的实现： 

```c++
//hotspot/src/cpu/x86/vm/templateTable_x86_64.cpp
void TemplateTable::putfield(int byte_no) {
  putfield_or_static(byte_no, false);
}

void TemplateTable::putfield_or_static(int byte_no, bool is_static) {
  transition(vtos, vtos);
  // ... 此处的 putfield 是将 object 放到域上，此处的 _tos_in（模板执行前的TosState）为 
  // atos （object cached）相关知识栈顶缓存
  {
    __ pop(atos);
    if (!is_static) pop_and_check_object(obj);
    // Store into the field
    do_oop_store(_masm, field, rax, _bs->kind(), false);

    __ jmp(Done);
  }
  __ bind(notObj);
  __ cmpl(flags, itos);
  __ jcc(Assembler::notEqual, notInt);
//...
}

```

事实上字节码 aastore 字节码也会调用到 do_oop_store 函数，本文仅用 putfield 举例， do_oop_store 函数会将 oop （或 NULL ）存储在 obj 描述的地址

```c++
static void do_oop_store(InterpreterMacroAssembler* _masm, Address obj, Register val, BarrierSet::Name barrier, bool precise) {
  switch (barrier) {
    // -XX:+UseG1GC 此处只关心 G1GC 会调用到的部分
    case BarrierSet::G1SATBCT:
    case BarrierSet::G1SATBCTLogging:
      {
        __ g1_write_barrier_pre(rdx /* obj */, rbx /* pre_val */, r15_thread /* thread */,  r8  /* tmp */,
                                val != noreg /* tosca_live */, false /* expand_call */);
        if (val == noreg) {
          __ store_heap_oop_null(Address(rdx, 0));
        } else {
          // G1 barrier needs uncompressed oop for region cross check.
          Register new_val = val;
          if (UseCompressedOops) {
            new_val = rbx;
            __ movptr(new_val, val);
          }
          __ store_heap_oop(Address(rdx, 0), val);
          __ g1_write_barrier_post(rdx /* store_adr */,
                                   new_val /* new_val */,
                                   r15_thread /* thread */,
                                   r8 /* tmp */,
                                   rbx /* tmp2 */);
        }
      }
      break;
    //...
  }
  //...
}
```
排除压缩指针与其他选项匹配的 barrier 代码，不难看出，当开启 G1GC 时，会在 store_heap_oop(_null) 之前调用 g1_write_barrier_pre ，之后调用 g1_write_barrier_post 函数。

<p id="g1_write_barrier_pre"></p>    

## g1_write_barrier_pre 在解释运行中的实现
回顾 G1GC 算法实现，SATB 专用写屏障的伪代码如下所示：

```
//对应JVM 在oop_store方法的赋值动作的前的pre-write barrier
def satb_write_barrier(field, newobj):
    if $gc_phase == GC_CONCURRENT_MARK:
        oldobj = *field
        if oldobj != Null:
            enqueue($current_thread.stab_local_queue, oldobj)

        *field = newobj
```

在伪代码中不难看出 SATB 专用写屏障，通过在将新对象写入域之前记录原来域对象,并滞后标记的方式来避免漏标。
在 G1GC 算法篇介绍 SATB 时提过，首先会将 old 对象放到本地线程的 SATB 队列中，所以首先来看一下 JavaThread 类的以下两个属性

```c++
//hotspot/src/share/vm/runtime/thread.hpp

  // Support for G1 barriers

  ObjPtrQueue _satb_mark_queue;          // Thread-local log for SATB barrier.
  // Set of all such queues.
  static SATBMarkQueueSet _satb_mark_queue_set;

  DirtyCardQueue _dirty_card_queue;      // Thread-local log for dirty cards.
  // Set of all such queues.
  static DirtyCardQueueSet _dirty_card_queue_set;
```

_satb_mark_queue 与 _dirty_card_queue 属于线程私有，当线程本地队列满了之后，会分别提交到全局的 _satb_mark_queue_set 与 _dirty_card_queue_set 交给 G1 线程批量处理。

 ObjPtrQueue 的父类 class PtrQueue 具有如下属性与函数
```c++
//hotspot/src/share/vm/gc_implementation/g1/ptrQueue.hpp
  // Whether updates should be logged.
  bool _active;

  // The buffer.
  void** _buf;
  // The index at which an object was last enqueued.  Starts at "_sz"
  // (indicating an empty buffer) and goes towards zero.
  size_t _index;

    // Enqueues the given "obj".
  void enqueue(void* ptr) {
    if (!_active) return;
    else enqueue_known_active(ptr);
  }
```

注释很好理解，三个属性分别是是否活跃、buffer 地址与最后一个对象的末尾索引。其中 _index 属性为 0 时代表队列满了，具体处理在后文分析。
接下来看一下 g1_write_barrier_pre 的核心实现

```c++
//hotspot/src/cpu/x86/vm/macroAssembler_x86.cpp
void MacroAssembler::g1_write_barrier_pre(Register obj,         //rdx /* obj */
                                          Register pre_val,     //rbx /* pre_val */
                                          Register thread,      //r15_thread /* thread */
                                          Register tmp,         //r8  /* tmp */
                                          bool tosca_live,      //val != noreg /* tosca_live */
                                          bool expand_call) {   //false /* expand_call */

  Label done;   //跳转标签 done 完成
  Label runtime;  //跳转标签 runtime

  //static ByteSize satb_mark_queue_offset()  { return byte_offset_of(JavaThread, _satb_mark_queue); }
  //通过类的函数获取 JavaThread 的 satb 相关信息
  Address in_progress(thread, in_bytes(JavaThread::satb_mark_queue_offset() +
                                       PtrQueue::byte_offset_of_active()));
  Address index(thread, in_bytes(JavaThread::satb_mark_queue_offset() +
                                       PtrQueue::byte_offset_of_index()));
  Address buffer(thread, in_bytes(JavaThread::satb_mark_queue_offset() +
                                       PtrQueue::byte_offset_of_buf()));


  // Is marking active? 是否处于并发标记阶段？
  if (in_bytes(PtrQueue::byte_width_of_active()) == 4) {
    cmpl(in_progress, 0);       //比较 _active 是否为 0，0需要跳转到 done ，即不需要写屏障
  } else {
    cmpb(in_progress, 0);
  }
  jcc(Assembler::equal, done);

  // Do we need to load the previous value? 空对象不需要写屏障
  if (obj != noreg) {
    load_heap_oop(pre_val, Address(obj, 0));
  }

  // Is the previous value null?    比较 pre_val 是否为空，空也不需要写屏障，直接跳转到 done
  cmpptr(pre_val, (int32_t) NULL_WORD);
  jcc(Assembler::equal, done);

  // Can we store original value in the thread's buffer?
  // Is index == 0?
  // (The index field is typed as size_t.)

  movptr(tmp, index);                   // tmp := *index_adr
  cmpptr(tmp, 0);                       // tmp == 0?
  jcc(Assembler::equal, runtime);       // If yes, goto runtime 比较 _index是否 0 。为 0 意味着当前线程的 
                                        // satb 队列满了，此时需要通过运行时调用，交给其他线程处理，具体为跳转到 bind(runtime) 处 

  // 计算 pre_val 的入队位置
  subptr(tmp, wordSize);                // tmp := tmp - wordSize  将 _index 减去 oopSize
  movptr(index, tmp);                   // *index_adr := tmp
  addptr(tmp, buffer);                  // tmp := tmp + *buffer_adr

  // Record the previous value 将 pre_val 入队
  movptr(Address(tmp, 0), pre_val);
  jmp(done);

  bind(runtime);
  // 运行时，最终会调用 SharedRuntime::g1_wb_pre ，这里会在运行时章节展开
  bind(done);
}
```
不难发现，整体实现与伪代码类似，完成了对前值的入队。

## g1_write_barrier_post 在解释运行中的实现

g1_write_barrier_post 的伪代码如下

```
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

过滤 obj 和 newobj 位于同一个区域，或者 newobj 为 Null 的时候， is_dirty_card() 用来检查参数 obj 所对应的卡片是否
为脏卡片。

```c++
void MacroAssembler::g1_write_barrier_post(Register store_addr,     // rdx /* store_adr */
                                           Register new_val,        // new_val /* new_val */
                                           Register thread,         // r15_thread /* thread */
                                           Register tmp,            // r8 /* tmp */
                                           Register tmp2) {         // rbx /* tmp2 */

  //此处获取 dirty_card 的 _index 与 _buf
  Address queue_index(thread, in_bytes(JavaThread::dirty_card_queue_offset() +
                                       PtrQueue::byte_offset_of_index()));
  Address buffer(thread, in_bytes(JavaThread::dirty_card_queue_offset() +
                                       PtrQueue::byte_offset_of_buf()));

  BarrierSet* bs = Universe::heap()->barrier_set();
  CardTableModRefBS* ct = (CardTableModRefBS*)bs;

  Label done;
  Label runtime;

  // Does store cross heap regions? 只有跨界引用需要被记录

  movptr(tmp, store_addr);
  xorptr(tmp, new_val);
  shrptr(tmp, HeapRegion::LogOfHRGrainBytes);
  jcc(Assembler::equal, done);

  // crosses regions, storing NULL? Null 对象不需要被记录 Null 时跳转到 done

  cmpptr(new_val, (int32_t) NULL_WORD);
  jcc(Assembler::equal, done);

  // storing region crossing non-NULL, is card already dirty? 已经脏的话，卡片不需要重复置脏

  const Register card_addr = tmp;
  const Register cardtable = tmp2;

  movptr(card_addr, store_addr);
  shrptr(card_addr, CardTableModRefBS::card_shift); //右移操作，找到当前对象对应的卡片
  movptr(cardtable, (intptr_t)ct->byte_map_base);
  addptr(card_addr, cardtable);

  //源码很清晰地说明了，G1 不维护从 young gen Region 出发的引用涉及到的 Rset 更新
  cmpb(Address(card_addr, 0), (int)G1SATBCardTableModRefBS::g1_young_card_val()); 
  jcc(Assembler::equal, done);

  membar(Assembler::Membar_mask_bits(Assembler::StoreLoad));  //写读屏障，相关知识指令乱序，内存序
  cmpb(Address(card_addr, 0), (int)CardTableModRefBS::dirty_card_val()); //判断卡片是否脏 
  jcc(Assembler::equal, done);


  // storing a region crossing, non-NULL oop, card is clean.
  // dirty card and log.

  movb(Address(card_addr, 0), (int)CardTableModRefBS::dirty_card_val());  //卡片置脏

  cmpl(queue_index, 0);
  jcc(Assembler::equal, runtime); //当前线程队列是否已满，已满转运行时（同 pre_barrier）
  //不满就入队
  subl(queue_index, wordSize);
  movptr(tmp2, buffer);
#ifdef _LP64
  movslq(rscratch1, queue_index);
  addq(tmp2, rscratch1);    //将 _index 减去 oopSize
  movq(Address(tmp2, 0), card_addr);  //入队
#else
  addl(tmp2, queue_index);  //将 _index 减去 oopSize
  movl(Address(tmp2, 0), card_addr);  //入队
#endif
  jmp(done);

  bind(runtime);
  // save the live input values
  push(store_addr);
  push(new_val);
  //调用运行时 SharedRuntime::g1_wb_post
#ifdef _LP64
  call_VM_leaf(CAST_FROM_FN_PTR(address, SharedRuntime::g1_wb_post), card_addr, r15_thread);
#else
  push(thread);
  call_VM_leaf(CAST_FROM_FN_PTR(address, SharedRuntime::g1_wb_post), card_addr, thread);
  pop(thread);
#endif
  pop(new_val);
  pop(store_addr);

  bind(done);
}
```

整体逻辑与伪代码类似， 源码的注释已经能够说明过程了，此处不作赘述。

#  write-barrier 在C2编译的实现

依旧是如下例子：

```java
//情况 1 ：假设 G1 正在并发标记，对象 t 作为根对象已经被标记（灰色），但是 t 引用的对象还没被标记
//情况 2 ：假设对象 young 和 t 分别在不同的 Region 中
Node young = new Node();//步骤 1
t.next = young;//步骤 2
//上述两种情况都需要在步骤 2 完成前记录下相关信息
```

当触发 C2 编译时，首先根据字节码创建理想图

```c++
//hotspot/src/share/vm/opto/parse2.cpp
void Parse::do_one_bytecode() {
  switch (bc()) {
    //...
    case Bytecodes::_putfield:
      do_putfield();
      break;
  }
}

//hotspot/src/share/vm/opto/parse.hpp
void do_putfield () { do_field_access(false, true); }

//hotspot/src/share/vm/opto/parse3.cpp
void Parse::do_field_access(bool is_get, bool is_field) {
  //...
    if (is_get) {
      (void) pop();  // pop receiver before getting
      do_get_xxx(obj, field, is_field);
    } else {
      do_put_xxx(obj, field, is_field);
      (void) pop();  // pop receiver after putting
    }
}

void Parse::do_put_xxx(Node* obj, ciField* field, bool is_field) {

  // Store the value.
  Node* store;
  if (bt == T_OBJECT) {
    const TypeOopPtr* field_type;
    if (!field->type()->is_loaded()) {
      field_type = TypeInstPtr::BOTTOM;
    } else {
      field_type = TypeOopPtr::make_from_klass(field->type()->as_klass());
    }
    store = store_oop_to_object(control(), obj, adr, adr_type, val, field_type, bt, mo);
  } else {
    store = store_to_memory(control(), adr, val, bt, adr_type, mo, is_vol);
  }

}

//hotspot/src/share/vm/opto/graphKit.hpp
  Node* store_oop_to_object(Node* ctl,
                            Node* obj,   // containing obj
                            Node* adr,   // actual adress to store val at
                            const TypePtr* adr_type,
                            Node* val,
                            const TypeOopPtr* val_type,
                            BasicType bt,
                            MemNode::MemOrd mo) {
    return store_oop(ctl, obj, adr, adr_type, val, val_type, bt, false, mo);
  }
```

创建详细过程可以读者自行理解，此处只关注调用关系，最终调用到 store_oop 

```c++
//hotspot/src/share/vm/opto/graphKit.cpp

Node* GraphKit::store_oop(/* 参数 */) {
  // ...
  pre_barrier(true /* do_load */, control(), obj, adr, adr_idx, val, val_type, NULL /* pre_val */, bt);

  Node* store = store_to_memory(control(), adr, val, bt, adr_idx, mo, mismatched);

  post_barrier(control(), store, obj, adr, adr_idx, val, bt, use_precise);
  return store;
}
```

在 store_to_memory 函数的前后分别加入了 pre_barrier 和 post_barrier 当函数需要 C2 编译运行，同时 GC 选用的是 G1GC 时，会在解析字节码时就为 write_barrier 生成相关的结点。

## pre_barrier 在C2编译中的实现

接下来追溯 pre_barrier 到底生成了哪些节点， 

```c++
//hotspot/src/share/vm/opto/graphKit.cpp
void GraphKit::pre_barrier(/*..*/) {

  BarrierSet* bs = Universe::heap()->barrier_set();
  set_control(ctl);
  //与模板解释器相同，需要根据选项选择 barrier 此处只关心 G1GC 部分
  switch (bs->kind()) {
    case BarrierSet::G1SATBCT:
    case BarrierSet::G1SATBCTLogging:
      g1_write_barrier_pre(do_load, obj, adr, adr_idx, val, val_type, pre_val, bt);
      break;
  // ...
  }
}

// G1 pre/post barriers
void GraphKit::g1_write_barrier_pre(bool do_load,
                                    Node* obj,
                                    Node* adr,
                                    uint alias_idx,
                                    Node* val,
                                    const TypeOopPtr* val_type,
                                    Node* pre_val,
                                    BasicType bt) {

  // Some sanity checks
  // Note: val is unused in this routine.

  IdealKit ideal(this, true);

  Node* tls = __ thread(); // ThreadLocalStorage

  Node* no_ctrl = NULL;
  Node* no_base = __ top();
  Node* zero  = __ ConI(0);
  Node* zeroX = __ ConX(0);

  float likely  = PROB_LIKELY(0.999);
  float unlikely  = PROB_UNLIKELY(0.999);

  BasicType active_type = in_bytes(PtrQueue::byte_width_of_active()) == 4 ? T_INT : T_BYTE;
 
  // Offsets into the thread 根据 thread 得到需要值的偏移地址
  const int marking_offset = in_bytes(JavaThread::satb_mark_queue_offset() +  // 648
                                          PtrQueue::byte_offset_of_active());
  const int index_offset   = in_bytes(JavaThread::satb_mark_queue_offset() +  // 656
                                          PtrQueue::byte_offset_of_index());
  const int buffer_offset  = in_bytes(JavaThread::satb_mark_queue_offset() +  // 652
                                          PtrQueue::byte_offset_of_buf());

  // Now the actual pointers into the thread 加法节点获取计算得到的地址
  Node* marking_adr = __ AddP(no_base, tls, __ ConX(marking_offset));
  Node* buffer_adr  = __ AddP(no_base, tls, __ ConX(buffer_offset));
  Node* index_adr   = __ AddP(no_base, tls, __ ConX(index_offset));

  // Now some of the values
  Node* marking = __ load(__ ctrl(), marking_adr, TypeInt::INT, active_type, Compile::AliasIdxRaw);

  // if (!marking) 判断是否为并发标记阶段
  __ if_then(marking, BoolTest::ne, zero, unlikely); {
    BasicType index_bt = TypeX_X->basic_type();
    assert(sizeof(size_t) == type2aelembytes(index_bt), "Loading G1 PtrQueue::_index with wrong size.");
    Node* index   = __ load(__ ctrl(), index_adr, TypeX_X, index_bt, Compile::AliasIdxRaw);

    if (do_load) {
      // load original value
      // alias_idx correct??
      pre_val = __ load(__ ctrl(), adr, val_type, bt, alias_idx);
    }

    // if (pre_val != NULL) 判断域之前的值是否为空
    __ if_then(pre_val, BoolTest::ne, null()); {
      Node* buffer  = __ load(__ ctrl(), buffer_adr, TypeRawPtr::NOTNULL, T_ADDRESS, Compile::AliasIdxRaw);

      // is the queue for this thread full? 当前线程的 satb 队列是否已满，已满会调用运行时的 g1_wb_pre
      __ if_then(index, BoolTest::ne, zeroX, likely); {

        // decrement the index
        Node* next_index = _gvn.transform(new (C) SubXNode(index, __ ConX(sizeof(intptr_t))));

        // Now get the buffer location we will log the previous value into and store it 入队
        Node *log_addr = __ AddP(no_base, buffer, next_index);
        __ store(__ ctrl(), log_addr, pre_val, T_OBJECT, Compile::AliasIdxRaw, MemNode::unordered);
        // update the index
        __ store(__ ctrl(), index_adr, next_index, index_bt, Compile::AliasIdxRaw, MemNode::unordered);

      } __ else_(); {

        // logging buffer is full, call the runtime 运行时 g1_wb_pre 函数与模板解释器调用的是同一个
        const TypeFunc *tf = OptoRuntime::g1_wb_pre_Type();
        __ make_leaf_call(tf, CAST_FROM_FN_PTR(address, SharedRuntime::g1_wb_pre), "g1_wb_pre", pre_val, tls);
      } __ end_if();  // (!index)
    } __ end_if();  // (pre_val != NULL)
  } __ end_if();  // (!marking)

  // Final sync IdealKit and GraphKit.
  final_sync(ideal);
}

```

整体逻辑类似，判断 pre_val 是否空，线程队列是否满，根据判断结果调用运行时或者生成将 pre_val 入栈的节点
在生成理想图节点后，C2编译器会对理想图做优化，最后生成机器代码，详细过程如下

```c++
enum PhaseTraceId {
_t_parser, // 1. 字节码解析与理想图生成
_t_optimizer, // 2. 机器无关优化
...
_t_matcher, // 3. 指令选择
_t_scheduler, // 4. 指令调度和全局代码提出
_t_registerAllocation, // 5. 寄存器分配
...
_t_blockOrdering, // 6. 移除空基本块
_t_peephole, // 7. 窥孔优化
_t_postalloc_expand,
_t_output, // 8. 生成机器代码
...
_t_registerMethod, // 9. 用编译生成的方法代替Java方法
_t_tec,
max_phase_timers
};
```

这部分不是本文关注的重点，为了方便研究 C2 编译结果可以通过 slowdebug 版本的 hotspot ，在运行程序时添加参数 -XX:+PrintOptoAssembly 查看 Opto （也可以直接查看汇编，但是 Opto 更直观），仍以上文例子为例可以在x86平台得到如下两段 Opto 节选，分别是 G1GC 与其他不会产生 pre_barrier 的 GC

```java
//无 pre_barrier
movq    [R10 + #24 (8-bit)], RBP	# ptr ! Field: Node.next //将寄存器RBP中的新值写入内存
```

```java
   B5: #	B13 B6 <- B4  Freq: 0.999979
   	# TLS is in R15
   	movsbl  R10, [R15 + #1424 (32-bit)]	# byte                  
   	testl   R10, R10                                        
   	jne     B13  P=0.001000 C=-1.000000   //if (!marking) ->B13

   B13: #	B6 B14 <- B5  Freq: 0.000999966
   	movq    RDI, [RBX + #24 (8-bit)]	# ptr ! Field: Node.next  //pre_val
   	testq   RDI, RDI	# ptr         
   	je     B6  P=0.500000 C=-1.000000   //if (pre_val != NULL) ->B14

   B14: #	B19 B15 <- B13  Freq: 0.000499983
   	# TLS is in R15
   	movq    R10, [R15 + #1440 (32-bit)]	# long
   	testq   R10, R10
   	je,s   B19  P=0.001000 C=-1.000000  // is the queue for this thread full? full -> B19 else -> B15

   B19: #	B6 <- B14  Freq: 4.99977e-07
   	# TLS is in R15
   	movq    RSI, R15	# spill           // logging buffer is full
   	call_leaf,runtime  g1_wb_pre        // 调用运行时函数 g1_wb_pre
     No JVM State Info
     # 
   	jmp     B6

   B15: #	B6 <- B14  Freq: 0.000499483
   	# TLS is in R15
   	movq    R11, [R15 + #1432 (32-bit)]	# ptr
   	movq    [R11 + #-8 + R10], RDI	# ptr   // pre_val 入队
   	addq    R10, #-8	# long    //将 _index 减去 oopSize
   	# TLS is in R15
   	movq    [R15 + #1440 (32-bit)], R10	# long
   	jmp     B6

   B6: #	B12 B7 <- B19 B15 B13 B5  Freq: 0.999979
   	movq    [RBX + #24 (8-bit)], RBP	# ptr ! Field: Node.next  //新值写入域

```

整体过程与理想图阶段生成的节点，能一一对应。

## post_barrier 在C2编译中的实现
post_barrier 的逻辑大体与 pre_barrier 相同，如下会根据 barrier 类型 G1GC 匹配到 g1_write_barrier_post

```c++
//hotspot/src/share/vm/opto/graphKit.cpp
//相同的逻辑，此处不作赘述
  switch (bs->kind()) {
    case BarrierSet::G1SATBCT:
    case BarrierSet::G1SATBCTLogging:
      g1_write_barrier_post(store, obj, adr, adr_idx, val, bt, use_precise);
      break;
  }
void GraphKit::g1_write_barrier_post(Node* oop_store,
                                     Node* obj,
                                     Node* adr,
                                     uint alias_idx,
                                     Node* val,
                                     BasicType bt,
                                     bool use_precise) {
  // If we are writing a NULL then we need no post barrier

  if (val != NULL && val->is_Con() && val->bottom_type() == TypePtr::NULL_PTR) {
    // Must be NULL
    const Type* t = val->bottom_type();
    assert(t == Type::TOP || t == TypePtr::NULL_PTR, "must be NULL");
    // No post barrier if writing NULLx
    return;
  }

  if (!use_precise) {
    // All card marks for a (non-array) instance are in one place:
    adr = obj;
  }
  // (Else it's an array (or unknown), and we want more precise card marks.)
  assert(adr != NULL, "");

  IdealKit ideal(this, true);

  Node* tls = __ thread(); // ThreadLocalStorage

  Node* no_base = __ top();
  float likely  = PROB_LIKELY(0.999);
  float unlikely  = PROB_UNLIKELY(0.999);
  Node* young_card = __ ConI((jint)G1SATBCardTableModRefBS::g1_young_card_val());
  Node* dirty_card = __ ConI((jint)CardTableModRefBS::dirty_card_val());
  Node* zeroX = __ ConX(0);

  // Get the alias_index for raw card-mark memory
  const TypePtr* card_type = TypeRawPtr::BOTTOM;

  const TypeFunc *tf = OptoRuntime::g1_wb_post_Type();

  // Offsets into the thread 获取偏移
  const int index_offset  = in_bytes(JavaThread::dirty_card_queue_offset() +
                                     PtrQueue::byte_offset_of_index());
  const int buffer_offset = in_bytes(JavaThread::dirty_card_queue_offset() +
                                     PtrQueue::byte_offset_of_buf());

  // Pointers into the thread 与 thread 值相加计算具体地址

  Node* buffer_adr = __ AddP(no_base, tls, __ ConX(buffer_offset));
  Node* index_adr =  __ AddP(no_base, tls, __ ConX(index_offset));

  // ... 部分优化 与 值获取

  // If we know the value being stored does it cross regions? 判断是否产生跨区域

  if (val != NULL) {
    // Does the store cause us to cross regions?

    // Should be able to do an unsigned compare of region_size instead of
    // and extra shift. Do we have an unsigned compare??
    // Node* region_size = __ ConI(1 << HeapRegion::LogOfHRGrainBytes); 
    
    // 判断方式为 check = obj ^ newobj
    Node* xor_res =  __ URShiftX ( __ XorX( cast,  __ CastPX(__ ctrl(), val)), __ ConI(HeapRegion::LogOfHRGrainBytes));

    // if (xor_res == 0) same region so skip 
    __ if_then(xor_res, BoolTest::ne, zeroX); {

      // No barrier if we are storing a NULL 如果是 NULL 也不需要 barrier
      __ if_then(val, BoolTest::ne, null(), unlikely); {

        // Ok must mark the card if not already dirty

        // load the original value of the card 
        Node* card_val = __ load(__ ctrl(), card_adr, TypeInt::INT, T_BYTE, Compile::AliasIdxRaw);

        __ if_then(card_val, BoolTest::ne, young_card); {   // 判断是否是 young 区的跨界引用， young 区每次都会被收集，故不维护 Rset
          sync_kit(ideal);
          // Use Op_MemBarVolatile to achieve the effect of a StoreLoad barrier.
          insert_mem_bar(Op_MemBarVolatile, oop_store);
          __ sync_kit(this);

          Node* card_val_reload = __ load(__ ctrl(), card_adr, TypeInt::INT, T_BYTE, Compile::AliasIdxRaw);
          __ if_then(card_val_reload, BoolTest::ne, dirty_card); {//判断卡片是否已经脏
            // 调用 g1_mark_card 函数 ，该函数的作用为更新卡表，并将卡片地址入队
            g1_mark_card(ideal, card_adr, oop_store, alias_idx, index, index_adr, buffer, tf);
          } __ end_if();
        } __ end_if();
      } __ end_if();
    } __ end_if();
  } else {
    // Object.clone() instrinsic uses this path.
    g1_mark_card(ideal, card_adr, oop_store, alias_idx, index, index_adr, buffer, tf);
  }

  // Final sync IdealKit and GraphKit.
  final_sync(ideal);
}

// Update the card table and add card address to the queue
void GraphKit::g1_mark_card(IdealKit& ideal,
                            Node* card_adr,
                            Node* oop_store,
                            uint oop_alias_idx,
                            Node* index,
                            Node* index_adr,
                            Node* buffer,
                            const TypeFunc* tf) {

  Node* zero  = __ ConI(0);
  Node* zeroX = __ ConX(0);
  Node* no_base = __ top();
  BasicType card_bt = T_BYTE;
  // Smash zero into card. MUST BE ORDERED WRT TO STORE
  __ storeCM(__ ctrl(), card_adr, zero, oop_store, oop_alias_idx, card_bt, Compile::AliasIdxRaw);

  //  Now do the queue work
  __ if_then(index, BoolTest::ne, zeroX); {//如果当前线程队列未满

    Node* next_index = _gvn.transform(new (C) SubXNode(index, __ ConX(sizeof(intptr_t))));
    Node* log_addr = __ AddP(no_base, buffer, next_index);

    // Order, see storeCM.
    __ store(__ ctrl(), log_addr, card_adr, T_ADDRESS, Compile::AliasIdxRaw, MemNode::unordered);
    __ store(__ ctrl(), index_adr, next_index, TypeX_X->basic_type(), Compile::AliasIdxRaw, MemNode::unordered);

  } __ else_(); {//线程队列已满会调用运行时
    __ make_leaf_call(tf, CAST_FROM_FN_PTR(address, SharedRuntime::g1_wb_post), "g1_wb_post", card_adr, __ thread());
  } __ end_if();

}
```
理想图部分代码与模板解释器体现的逻辑也是一致的，接下来看看最终生成的 Opto

```java
   B6: #	B12 B7 <- B19 B15 B13 B5  Freq: 0.999979
   	movq    [RBX + #24 (8-bit)], RBP	# ptr ! Field: Node.next //完成对象域新值的写入
   	movq    R10, RBP	# ptr -> long
   	movq    R11, RBX	# ptr -> long
   	xorq    R10, R11	# long
   	shrq    R10, #20
   	testq   R10, R10
   	je,s   B12  P=0.500000 C=-1.000000    //if (xor_res == 0) same region so skip -> B12 ret else -> B7

   B7: #	B12 B8 <- B6  Freq: 0.499989
  	shrq    R11, #9
   	movq    RDI, 0x00007ecf56bf1000	# ptr
   	addq    RDI, R11	# ptr
   	movsbl  R10, [RDI]	# byte
   	cmpl    R10, #32
   	je,s   B12  P=0.500000 C=-1.000000  // if_then(card_val, BoolTest::ne, young_card) 不用维护 young 区域的 Rset
 
   B8: #	B12 B9 <- B7  Freq: 0.249995
   	# TLS is in R15
   	movq    R10, [R15 + #1504 (32-bit)]	# long
   	# TLS is in R15
   	movq    R11, [R15 + #1496 (32-bit)]	# ptr
   	lock addl [rsp + #0], 0	! membar_volatile // 内存序
   	movsbl  R9, [RDI]	# byte
   	testl   R9, R9
   	je,s   B12  P=0.500000 C=-1.000000   // if_then(card_val_reload, BoolTest::ne, dirty_card) 判断卡片是否已经脏

   B9: #	B11 B10 <- B8  Freq: 0.124997   
   	movb    [RDI], #0	# CMS card-mark byte 0  // Smash zero into card. MUST BE ORDERED WRT TO STORE 卡片置为脏
   	testq   R10, R10
   	jne,s   B11  P=0.500000 C=-1.000000 // if_then(index, BoolTest::ne, zeroX) 判断当前线程队列是否已满

   B10: #	B12 <- B9  Freq: 0.0624987
   	# TLS is in R15
   	movq    RSI, R15	# spill
   	call_leaf,runtime  g1_wb_post // queue full runtime
     No JVM State Info
     # 
   	jmp,s   B12

   B11: #	B12 <- B9  Freq: 0.0624987  //  Now do the queue work 入队
   	movq    [R11 + #-8 + R10], RDI	# ptr
   	addq    R10, #-8	# long  //将 _index 减去 oopSize
   	# TLS is in R15
   	movq    [R15 + #1504 (32-bit)], R10	# long

   B12: #	N1 <- B10 B11 B8 B7 B6  Freq: 0.999979
   	addq    rsp, 32	# Destroy frame
  	popq   rbp
	  testl  rax, [rip + #offset_to_poll_page]	# Safepoint: poll for GC

   	ret

```

#  write-barrier 在运行时的实现

运行时，不仅需要处理队列满后加入队列的对象，还需要将满的队列放入全局链表，并重置线程私有队列。上文频繁提及，当队列满了会调用运行时实现写屏障，接下来看一下运行时对写屏障的实现。

## g1_wb_pre 与 g1_wb_post 在运行时的实现

```c++
// G1 write-barrier pre: executed before a pointer store.
JRT_LEAF(void, SharedRuntime::g1_wb_pre(oopDesc* orig, JavaThread *thread))
  if (orig == NULL) {
    assert(false, "should be optimized out");
    return;
  }
  assert(orig->is_oop(true /* ignore mark word */), "Error");
  // store the original value that was in the field reference
  thread->satb_mark_queue().enqueue(orig);
JRT_END

// G1 write-barrier post: executed after a pointer store.
JRT_LEAF(void, SharedRuntime::g1_wb_post(void* card_addr, JavaThread* thread))
  thread->dirty_card_queue().enqueue(card_addr);
JRT_END
```
运行时的代码更直观，即将域前值/卡片地址入队，其中

```c++
  // SATB marking queue support
  ObjPtrQueue& satb_mark_queue() { return _satb_mark_queue; }
  static SATBMarkQueueSet& satb_mark_queue_set() {
    return _satb_mark_queue_set;
  }

  // Dirty card queue support
  DirtyCardQueue& dirty_card_queue() { return _dirty_card_queue; }
  static DirtyCardQueueSet& dirty_card_queue_set() {
    return _dirty_card_queue_set;
  }
```
这些值在[上文](#g1_write_barrier_pre)曾多次提及

根据上面对模板解释器和 C2 编译的分析，只有当当前线程队列满时才会调用到运行时，追溯 enqueue 的实现，

```c++
ObjPtrQueue _satb_mark_queue;          // Thread-local log for SATB barrier.
DirtyCardQueue _dirty_card_queue;      // Thread-local log for dirty cards.

class ObjPtrQueue: public PtrQueue{}
class DirtyCardQueue: public PtrQueue{}

```
enqueue 函数的实现在 ObjPtrQueue 与 DirtyCardQueue 类的共同父类 PtrQueue 中

```c++
//hotspot/src/share/vm/gc_implementation/g1/ptrQueue.hpp
  // Enqueues the given "obj".
  void enqueue(void* ptr) {
    if (!_active) return;
    else enqueue_known_active(ptr);
  }

//hotspot/src/share/vm/gc_implementation/g1/ptrQueue.cpp

void PtrQueue::enqueue_known_active(void* ptr) {

  while (_index == 0) {   //当前线程已满调用 handle_zero_index 函数
    handle_zero_index();  //handle_zero_index 函数会为线程私有队列置申请新空间，并重置 _index 为 size
  }

  // 将 _index 减去 oopSize，并将对象指针入队
  _index -= oopSize;
  _buf[byte_index_to_index((int)_index)] = ptr;
}

void PtrQueue::handle_zero_index() {
  assert(_index == 0, "Precondition.");

  // This thread records the full buffer and allocates a new one (while
  // holding the lock if there is one).
  if (_buf != NULL) {
    if (!should_enqueue_buffer()) {
      assert(_index > 0, "the buffer can only be re-used if it's not full");
      return;
    }

    if (_lock) {
      assert(_lock->owned_by_self(), "Required.");

      void** buf = _buf;   // local pointer to completed buffer
      _buf = NULL;         // clear shared _buf field

      locking_enqueue_completed_buffer(buf);  // enqueue completed buffer 将线程私有的队列加入链表

      if (_buf != NULL) return;
    } else {
    //...
    }
  }
  // Reallocate the buffer 重新为当前线程的队列申请新空间，再将 _index = _sz
  _buf = qset()->allocate_buffer();
  _sz = qset()->buffer_size();
  _index = _sz;
  assert(0 <= _index && _index <= _sz, "Invariant.");
}

void PtrQueue::locking_enqueue_completed_buffer(void** buf) {
  assert(_lock->owned_by_self(), "Required.");

  // We have to unlock _lock (which may be Shared_DirtyCardQ_lock) before
  // we acquire DirtyCardQ_CBL_mon inside enqeue_complete_buffer as they
  // have the same rank and we may get the "possible deadlock" message
  _lock->unlock();

  qset()->enqueue_complete_buffer(buf); //将线程私有的队列加入链表
  // We must relock only because the caller will unlock, for the normal
  // case.
  _lock->lock_without_safepoint_check();
}

void PtrQueueSet::enqueue_complete_buffer(void** buf, size_t index) {
  MutexLockerEx x(_cbl_mon, Mutex::_no_safepoint_check_flag);
  BufferNode* cbn = BufferNode::new_from_buffer(buf);
  cbn->set_index(index);
  if (_completed_buffers_tail == NULL) {  //链表是否为空
    assert(_completed_buffers_head == NULL, "Well-formedness");
    _completed_buffers_head = cbn;  // 链表头为 buf 
    _completed_buffers_tail = cbn;  // 链表尾为 buf
  } else {
    _completed_buffers_tail->set_next(cbn); //非空时候 buf 加入链表末尾
    _completed_buffers_tail = cbn;
  }
  _n_completed_buffers++; //链表长度 +1
  //...
}

```

<p id="references"></p>    

# 参考

jdk8源代码