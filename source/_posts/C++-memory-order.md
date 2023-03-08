---
title: C++ 内存模型 & 内存序
date: 2023-03-08 02:34:23
tags: [C++,Memory Order,Memory Fence]
toc: true
---

> One of the most important features of the C++ Standard is something most programmers won’t even notice. It’s not the new syntax features, nor is it the new library facilities, but the new multithreading-aware memory model. Without the memory model to define exactly how the fundamental building blocks work, none of the facilities I’ve covered could be relied on to work. There’s a reason that most programmers won’t notice: if you use mutexes to protect your data and condition variables, futures, latches, or barriers to signal events, the details of why they work aren’t important. It’s only when you start trying to get “close to the machine” that the precise details of the memory model matter. [^1]

在C++11标准中提出了多线程感知内存模型，它的出现使得提供各种多线程的基础设施成为可能。当你想要了解更深入地了解计算机底层时，理解内存模型的细节至关重要。

::: tip
*此处的内存模型指是指多线程感知方面，而不是C++对象内存布局之类*
:::

## 优化是把双刃剑

### 编译器优化

### CPU优化

[^1]: C++ Concurrency in Action 2nd
