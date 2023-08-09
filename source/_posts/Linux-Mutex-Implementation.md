---
title: Linux Mutex 实现原理
date: 2023-08-10 01:07:16
tags:
---

## 数据结构

在 Linux 的 [**Mutex** 结构体](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h#L64)中实际上定义了其他的字段，这里将无关紧要的成员变量做了删减，只保留最关键的部分。

```c
struct mutex {
    atomic_long_t       owner;
    raw_spinlock_t      wait_lock;
    struct list_head    wait_list;
    struct optimistic_spin_queue osq; /* Spinner MCS lock */
};
```

这里对 Field 的用途进行简单的解释：

* `owner`： 表示当前 **Mutex** 的持有者，为空说明没有进程持有，非空则说明当前有进程持有
* `wait_lock`：保护 `wait_list` 的自旋锁
* `wait_list`：等待 **Mutex** 的进程队列
* `osq`：MCS 锁，保证只有一个线程正在乐观自旋

其中 `owner` 的 64 Bits 被拆分成了两个部分：

* 63rd - 3rd Bit表示 `task_struct` 地址
* 2nd - 0 Bit表示锁的状态，分别用于表示 `wait_list` 是否为空，以及用于 **Handoff** 机制

## 流程介绍

为了提高性能，**Mutex** 加锁存在 3 种路径，当快速 & 中速路径条件不满足时，才会使用慢速路径。[^1]

### Fast-Path（快速路径）

Fast-Path 的实现非常简单高效：使用 CAS 指令判断 `owner` 是否为 0，如果为 0 则将当前的线程/Task ID 写入，这样就完成了一次上锁过程。

### Mid-Path（中速路径）

Mid-Path 的核心思想是通过乐观自旋来减少无效的 CPU 上下文切换。

在不带有乐观自旋的版本中，当线程 A 获取一个正在被进程 B 持有的 **Mutex** 时，会直接进入睡眠状态，等待 **Mutex** 被释放。
其实直接进入睡眠状态是不必要的，因为持有 **Mutex** 的线程 B 往往很快就会释放，完全可以多等待一段时间来避免多余的睡眠唤醒。如果此时线程 B 失去了 CPU，则线程 A也会取消自旋，同样进入睡眠。

通过 `osq` ，保证同时只有一个线程进行乐观自旋，避免大量线程蜂拥而至。

### Slow-Path（慢速路径）

将当前线程加入到 `wait_list` 后睡眠等待线程被重新调度唤醒。

## 性能优化

### Handoff机制

> 当 **Mutex** 支持乐观自旋时，那么会存在这样的情况：`wait_list` 中有一些进程在睡眠并等待被唤醒拿锁，同时还有一些进程不在wait_list中且不断的自旋等锁或乐观自旋等锁。`wait_list` 上的进程大概率是抢不过自旋拿锁的进程的，这是因为调度延时的存在。当被唤醒的进程真正获得 CPU 时，锁早就被自旋等锁的进程给偷走了(偷锁)，这会在一定程度的造成 `wait_list` 的等待者长时间“饥饿”。于是社区的peterz大神为 **Mutex** 增加了 Handoff 机制用来防止这种情况的发生。具体实现利用了`owner` 中的 flags field(2bit至0bit)。[^2]

### Futex 实现

**Futex** 全称为：*Fast Userspace Mutexes*

**Futex** 的核心机制思路很简单：由于系统调用的代价很大，通过避免在无竞争的场景下总是进行系统调用陷入内核态来优化性能。

在实现思路上，在不同的进程间 *mmap* 共享一段内存，将 *Futex* 保存在这段内存中。*Futex* 中保存着一个整数 Counter，需要上锁时对 Counter *fetch_sub* 操作。如果 *fetch_sub* 前 Counter = 0，说明此时没有进程正在上锁，反之则说明当前有其他进程正在上锁，需要调用 `futex_wait()` 陷入内核态将自己加入到等待队列中。

[^1]: [Linux Mutex机制分析 - LoyenWang - 博客园](https://www.cnblogs.com/LoyenWang/p/12826811.html)
[^2]: [浅析mutex实现原理](https://zhuanlan.zhihu.com/p/390107537)
