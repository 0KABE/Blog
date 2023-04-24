---
title: CPP-Memory-Order-Part-2
date: 2023-04-25 02:31:50
tags:
---

在 Part 1 中主要介绍了内存模型的发展历史，引入了 *DRF* 概念 (Data Race Free)。C++ 内存模型的所有一致性保证全部以程序满足 *DRF* 要求为前提，对于不满足 *DRF* 的多线程程序，程序的行为是未定义的（Undefined Behavior）。

Part 2 将介绍 C++ 中涉及到的前置概念，并介绍 Memory Order 的具体内容。我期望通过 Sample Code 能让它有一个直观的认识。

如果想要知道更多细节，可以阅读 Boehm 的 [Memory Model Rationales](https://open-std.org/jtc1/sc22/wg21/docs/papers/2007/n2176.html#undefined) 和 Boehm & Adve 的 [Foundations of the C++ Concurrency Memory Model](https://www.hpl.hp.com/techreports/2008/HPL-2008-56.pdf) 

## C++ 内存模型概念

### Sequenced Before

Sequenced Before 就是**相同线程**内代码的求值顺序，C++ 中对求值顺序进行了详细的定义： [Evaluation Order](https://en.cppreference.com/w/cpp/language/eval_order)

### Synchronized With

同 *DRF* 中介绍的那样，程序通过 *Synchronization* 进行同步操作，这个操作在 C++ 中对应的就是 *Synchronized With* 关系。C++ 中往往通过 *Synchronization Atomic* 来建立同步关系。对于原子变量来说，*Atomic Store* 与 *Atomic Load* 间会建立同步关系；除此之外，C++ 还提供了 *Fence* 基础设施用于建立同步关系。

### Happens Before

通过在多线程程序中建立的 *Sequenced Before* 与 *Synchronized With* 关系，还可以推导出 *Happens Before* 关系。

*Happens Before* 关系不仅可以帮助分析代码的可见性顺序，还可以通过它分析多线程程序中是否存在 *Data Racing*，继而确定是否符合 *DRF* 规范。

由于不是任意代码之间都能建立 *Happens Before* 关系，所以它本质上是一个偏序关系。

- - - -

## Memory Order

C++11 标准中定义了4种 C++ *Memory Order*：*Sequential Consistency*，*Acquire-Release*，*Consume-Release* 以及 *Relax*。

### Sequential Consistency

与 [[C++ 内存模型 - Part 1]] 中介绍的硬件内存模型非常接近，同样基于 *DRF-SC* 模型。

* *SC* 相比其他的弱一致性模型最大的区别在于 ***SC* 保证存在一个全局可见的内存修改顺序**，下面的 Case 可以说明这一点：

```c++
// Litmus Test: Store Buffering 
// Can this program see r1 = 0, r2 = 0?
std::atomic<int> x, y;
int r1, r2;
// Thread 1
x.store(1, std::memory_order_seq_cst);
r1 = y.load(std::memory_order_seq_cst);
// Thread 2
y.store(1, std::memory_order_seq_cst);
r2 = x.load(std::memory_order_seq_cst);
// On C++11 (sequentially consistent atomics): no.
```

* C++ 11 中的 *SC* 只能在 *SC* Model 的原子变量中生效，对于非原子变量或非 *SC* 操作的原子变量无效。例如下面的 Case：

```c++
std::atomic<int> x, y, z;
int r1, r2, r3;

// Thread 1
x.store(1, std::memory_order_seq_cst);
r2 = y.load(std::memory_order_acquire);
r3 = z.load(std::memory_order_seq_cst);
// Thread 2
y.store(1, std::memory_order_release);
r1 = x.load(std::memory_order_seq_cst);
z.store(1, std::memory_order_seq_cst);

// Assertation
assert(r1 != 0 || r2 != 0);  // maybe be failed
assert(r1 != 0 || r3 != 0);  // always successful
assert(r2 != 0 || r3 != 0);  // maybe be failed
```

### Acquire-Release

在 C++ 11 中，Acquire-Release 是一种常用的**弱同步内存一致性模型**。

* Acquire 保证在其之后的读写不会被重排到其之前（不允许出现 Load-Store、Load-Load 乱序）
* Release 保证在其之前的读写不会被重排到其之后（不允许出现 Store-Store、Load-Store乱序）

这种模型有一种不严谨的说法：**在 Acquire 与 Release 建立同步关系的前提下，Acquire 之后的操作能够观察到 Release 之前的所有副作用**。

```c++
std::atomic<int> x{0};
int y{0}, r1{0}, r2{0};

// Thread 1
y = 1;
x.store(1, std::memory_order_release);
// Thread 2
r1 = x.load(std::memory_order_acquire);
r2 = y;

// Assertation
assert(!(r1 == 1 && r2 == 0)); // r1 must be 1 if r1 is 1
```

对于没有建立同步关系的 Acquire/Release 操作，会出现意外情况。

```c++
// Litmus Test: Store Buffering 
// Can this program see r1 = 0, r2 = 0?
std::atomic<int> x, y;
int r1, r2;
// Thread 1
x.store(1, std::memory_order_release);
r1 = y.load(std::memory_order_acquire);
// Thread 2
y.store(1, std::memory_order_release);
r2 = x.load(std::memory_order_acquire);
// On C++11 (sequentially consistent atomics): no.
// On C++11 (acquire/release atomics): yes!
```

### Consume-Release

*Consume-Release* 是一种更宽松的内存模型，其思想是通过数据依赖关系来提供更弱的内存一致性保证。

**到 2015 年 2 月为止没有产品编译器跟踪依赖链：均将消费操作提升为获得操作。目前释放消费顺序的规范正在修订中，暂时不鼓励使用 *Consume-Release* 。（C++17起）**

### Relax

*Relax* 是所有 C++ 中最弱的 Memory Order，甚至**不能**通过它建立 *Synchronized-With* 关系。

*Relax* 只提供两个保证：

1. 原子性
2. 修改顺序一致性（不同线程观察该变量只能得到相同的修改顺序）

Relax 在使用中很容易出现令人匪夷所思的现象，其根本原因是它无法建立 *Synchrozied-With* 关系：

```c++
// Can this program see r1 = 1, r2 = 1?
std::atomic<int> x{0}, y{0};
int r1{0}, r2{0};
// Thread 1
r1 = y.load(std::memory_order_relaxed);
x.store(r1, std::memory_order_relaxed);
// Thread 2
r2 = x.load(std::memory_order_relaxed);
y.store(1, std::memory_order_relaxed);
// On C++11 (relax atomics): yes!
```

- - - -

## Memory Fence

### Fence V.S. Atomic Operation

Fence 与 原子变量同样都能够起到同步的作用，在许多场景下它们可以相互代替。例如下面的示例中 `f()` 与 `g()` 是等价的。

```c++
std::atomic<int> var(1);
int a = 1;

void f() {
  a = 123;
  var.store(0, std::memory_order_release);
}

void g() {
  a = 123;
  std::atomic_thread_fence(std::memory_order_release);
  var.store(0, std::memory_order_relaxed);
}
```

> `atomic_thread_fence` 强加的同步制约强于带同一 Memory Order 的原子存储操作。在原子存储释放操作阻止所有前驱写入被移动到存储释放之后的同时，带 `memory_order_release` 顺序的 `atomic_thread_fence` 还阻止所有前驱写入被移动到后继存储之后。  

- - - -

## FAQ

### 与 `volatile` 的区别

C++ 的 `volatile` 语义与 Java 中的不太相同，C++ 的 `volatile` 不提供原子性、同步或内存顺序，所以并不能将其用于建立多线程间同步。C++ 的 `volatile` 主要包含两种作用：

* 禁止编译期乱序重排
* 总是从内存中读取最新的值

> `voaltile` 的经典用法是模仿映射于内存的 I/O 端口  
> Visual Studio 中的 volatile 变量存在Acquire/Release语义，尽管C++标准中 volatile 不可用于多线程开发 - [std::memory_order - cppreference.com](https://en.cppreference.com/w/cpp/atomic/memory_order)  

- - - -

## 参考资料

[std::memory_order - cppreference.com](https://en.cppreference.com/w/cpp/atomic/memory_order)
[The Synchronizes-With Relation](https://preshing.com/20130823/the-synchronizes-with-relation/)
[C++ 内存模型](https://paul.pub/cpp-memory-model/)
[Memory Models by Russ Cox](https://research.swtch.com/mm)
