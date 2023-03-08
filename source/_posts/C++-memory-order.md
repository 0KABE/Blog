---
title: C++ 内存模型 & 内存序
date: 2023-03-08 02:34:23
tags: [C++,Memory Order,Memory Fence]
toc: true
---

> One of the most important features of the C++ Standard is something most pro- grammers won’t even notice. It’s not the new syntax features, nor is it the new library facilities, but the new multithreading-aware memory model. Without the memory model to define exactly how the fundamental building blocks work, none of the facilities I’ve covered could be relied on to work. There’s a reason that most programmers won’t notice: if you use mutexes to protect your data and condition variables, futures, latches, or barriers to signal events, the details of why they work aren’t important. It’s only when you start trying to get “close to the machine” that the precise details of the memory model matter.

在C++11标准中提出了全新的多线程感知内存模型，它的出现使得提供各种多线程的基础设施成为可能。当你想要了解更深入地了解计算机底层时，理解内存模型的细节至关重要。

## Introduction

### Sequence consistent Model

### Acquire & Release Model

### Relax Model
