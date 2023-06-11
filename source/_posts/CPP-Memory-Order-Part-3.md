---
title: C++ 内存模型 - Part 3
date: 2023-05-31 02:11:27
tags:
---

Part 2 中介绍了 C++ 中的内存模型以及它们的主要区别，在 Part 3 中，我将会介绍 C++ 内存模型在生产环境中的使用案例。这些案例有一部份来自于 C++ STL 库，还有一些是经典案例。

## 内存模型案例

### Mutex

在看过了前面 Part 1 & Part 2 的内容之后，你也许会意识到 Mutex 作为多线程开发中常见的基础设施，也包含了特殊的 Memory Order 语义：我在持有 Mutex 期间做的所有操作，在下一次 Mutex 被持有时必须可见，是非常经典的 Acquire-Release 语义。

在了解内存一致性模型之前，把这种一致性当作是编程语言/硬件默认的保证是一种错误的认识。

### Shared Pointer

C++ 11 中引入了 Smart Pointers（智能指针）， `std::shared_ptr<T>` 与 `std::weak_ptr<T>` 增加 Control Block，并在 Control Block 中维护着对象的 Reference Count，实现当没有其他对象引用它时会自动销毁的功能。虽然 C++ 标准中没有要求智能指针需要保证线程安全，但是标准中提到 Smart Pointers 中的 Control Block 不能出现 Data Race。为了在保证不出现 Data Race 的前提下拥有足够的性能，就需要引入内存模型并选择合适的语义。

#### Reference Count Increment

典型的引用计数器实现会在增加引用计数时使用 `std::memory_order_relaxed` & `fetch_add` 自增。

在经过思考之后，你也许会提出一些疑问：

Q：**如果 Reference Count Increment 被 Delay 到另外一个线程的 Reference Count Decrement 之后，会出现 Reference Count 在非预期情况下被减至 0 吗？**

A：不会，这个问题需要结合 Shared Pointer 的实际应用场景来进行分析。Increment 的触发时机只会出现在 Shared Pointer 被拷贝时，这时候至少存在一个 Shared Pointer 没有被释放，即不可能在这种情况下 Reference Count 为 0。

#### Reference Count  Decrement

自减引用计数，当引用计数为 0 时释放持有的指针。`fetch_sub` 操作需要满足 `std::memory_order_acq_rel` 语义。

可能的实现代码如下：

```c++
if (cnt.fetch_sub(1, std::memory_order_acq_rel) == 1) {
   delete p;
}
```

使用 `std::memory_order_acq_rel` 语义的原因在于 `delete p` 会与 `fetch_sub` 前的 Store 操作出现 Data Race，需要使用 Acquire-Release 语义建立 Memory Barrier。

#### 代码示例

下面这份玩具代码实现了`std:shared_ptr`  的主要功能，并且保证 RefCount 是线程安全的。但其与标准库STL的实现相比非常简单，缺少了许多功能：自定义 Deleter、支持 WeakPtr、支持 Shared Pointer Aliasing、各种构造函数、`std::make_shared<T>` 方法中的 Placement New 优化等等。

```c++
class ControlBlock {
 public:
  using Count = unsigned;

  // fetch_add() can be std::memory_order_relaxed, because the atomic variable no need to have
  // synchronize-with relationship
  Count IncreaseShared() noexcept { return shared_count_.fetch_add(1, std::memory_order_relaxed); }

  // fetch_sub() must be std::memory_order_acq_rel, because the atomic variable needs to
  // synchronize-with itself
  Count DecreaseShared() noexcept { return shared_count_.fetch_sub(1, std::memory_order_acq_rel); }

  [[nodiscard]] Count GetCount() const noexcept {
    return shared_count_.load(std::memory_order::relaxed);
  }

 private:
  std::atomic<Count> shared_count_{1};  // Default ref count is 1
};

template <typename T>
class SharedPtr {
 public:
  constexpr SharedPtr() = default;

  constexpr SharedPtr(nullptr_t) noexcept : SharedPtr(){};  // NOLINT(google-explicit-constructor)

  explicit SharedPtr(T* ptr) : ptr_(ptr), control_block_(new ControlBlock) {}

  SharedPtr(const SharedPtr& shared_ptr) : SharedPtr(shared_ptr.ptr_, shared_ptr.control_block_) {
    if (control_block_) {
      control_block_->IncreaseShared();
    }
  }

  SharedPtr(SharedPtr&& shared_ptr) noexcept : SharedPtr() { Swap(shared_ptr); }

  SharedPtr& operator=(const SharedPtr& shared_ptr) {
    if (this != &shared_ptr) {  // Handle self-assignment case
      Release();
      ptr_ = shared_ptr.ptr_;
      control_block_ = shared_ptr.control_block_;

      if (control_block_) {
        control_block_->IncreaseShared();
      }
    }

    return *this;
  }

  SharedPtr& operator=(SharedPtr&& shared_ptr) noexcept {
    SharedPtr null_ptr;
    null_ptr.Swap(shared_ptr);
    null_ptr.Swap(*this);
    return *this;
  }

  bool operator==(const SharedPtr& shared_ptr) const { return ptr_ == shared_ptr.ptr_; }

  auto operator<=>(const SharedPtr& shared_ptr) const { return ptr_ <=> shared_ptr.ptr_; }

  operator bool() const { return ptr_ != nullptr; }  // NOLINT(google-explicit-constructor)

  T& operator*() const noexcept { return *ptr_; }

  T* operator->() const noexcept { return ptr_; }

  T* Get() const noexcept { return ptr_; }

  void Swap(SharedPtr& shared_ptr) {
    std::swap(ptr_, shared_ptr.ptr_);
    std::swap(control_block_, shared_ptr.control_block_);
  }

  void Reset(T* ptr) {
    Release();
    *this = SharedPtr(ptr);
  }

  void Release() {
    // pointer and control block must be both either nullptr or not at the same time
    assert(!ptr_ == !control_block_);

    if (ptr_ && control_block_ && control_block_->DecreaseShared() == 1) {
      delete ptr_;
      delete control_block_;
    }

    ptr_ = nullptr;
    control_block_ = nullptr;
  }

  [[nodiscard]] ControlBlock::Count UseCount() const noexcept { return control_block_->GetCount(); }

  ~SharedPtr() { Release(); }

 private:
  SharedPtr(T* ptr, ControlBlock* control_block) : ptr_(ptr), control_block_(control_block) {}

  T* ptr_{nullptr};
  ControlBlock* control_block_{nullptr};
};
```

### Singleton

单例模式作为一种常见的设计模式被许多地方使用，并发从不同的线程获取实例会出现数据竞争导致需要注意线程安全问题。

由于单例模式中的实例在整个进程生命周期中只会初始化一次，因此极大部份是对实例的读操作，只有一次真正的写操作。为了优化性能，工程师们提出了一种 DCLP 模式（Double Check Locking Pattern）。

这种模式的优点在于只有当指针为空时才会引入锁来初始化实例，其余的情况只会有一次指针的判空开销。

在 C++ 中正确实现单例模式需要涉及到内存屏障、CPU 多流水线指令重排、编译器指令重排等知识，而这正是 C++ 内存模型所需要解决的。

下面的示例代码是单例模式的一种实现，可以很容易发现它是非线程安全的，因为可能存在多线程同时读写 `instance_` 变量等问题。

```c++
template <typename T>
class Singleton {
 public:
  static T& GetInstance() {
    if (instance_ != nullptr) {
      instance_ = new T();
    }
    return instance_;
  }

 private:
  static T* instance_{nullptr};
};
```

下面的示例代码使用锁来互斥 `instance_` 的写操作，保证 `instace_` 只会被初始化一次，但这仍旧是非线程安全的。
原因在于：`instance_` 可能在对象初始化完成前被赋值，导致其他线程返回未初始化完成的实例。这可能是由于 CPU 指令重排，也可能是由于编译器指令重排，在不引入内存屏障的情况下无法解决该问题。

```c++
template <typename T>
class Singleton {
 public:
  static T& GetInstance() {
    if (instance_ != nullptr) {
      std::lock_guard _(mutex_);
      if (instance_ != nullptr) {
        instance_ = new T();
      }
    }
    return instance_;
  }

 private:
  static T* instance_{nullptr};
  static std::mutex mutex_;
};
```

这份示例代码是一个线程安全的实现，C++ 内存模型会在对应的位置插入内存屏障，避免上面的问题。A 与 C 之间建立 *Acquire-Release* 关系，保证当 A 在观察到指针为非空时，实例已经被初始化完成。

B 处使用 `std::memory_order::relaxed` 的原因在于此处读取 `instance_` 是互斥的，深层一些的解释是互斥锁本身就包含有 *Acquire-Release* 语义，保证上一个线程对临界区的任何改动对于下一个进入临界区的线程一定是可见的，因此也就不需要通过 `instance_` 来与 C 处的写入建立同步关系了。

```c++
template <typename T>
class Singleton {
 public:
  static T& GetInstance() {
    T* instance = instance_.load(std::memory_order::acquire);  // A

    if (instance != nullptr) {
      std::lock_guard _(mutex_);
      instance = instance_.load(std::memory_order::relaxed);  // B

      if (instance != nullptr) {
        instance = new T;
        instance_.store(instance, std::memory_order::release);  // C
      }
    }

    return instance_.load(std::memory_order::acquire);
  }

 private:
  static std::atomic<T*> instance_{nullptr};
  static std::mutex mutex_;
};
```

### Lock-free queue

To be continued
