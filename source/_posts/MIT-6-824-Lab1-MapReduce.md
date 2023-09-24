---
title: MIT-6.824-Lab1-MapReduce
date: 2023-09-25 01:59:30
tags: MIT-6.824, Go, MapReduce
---

完成 *Map Reduce Lab* 已经有几天了，觉得有必要记录一下实现的过程中踩过的坑，以及一些自己的心得。

## Map Reduce 介绍

在 Google 的使用场景中，存在大量的原始数据需要被处理来完成各种各样的需求：例如为了实现 Google 搜索 *Page Rank* 算法，需要对爬虫获得的大量 Web 页面进行分析；传统单机程序的性能远远无法满足这类超大数据场景，需要借助大量计算机来共同完成一件任务。

论文中提到，在过去的五年中，Google 的工程师们实现了成百上千的程序来处理海量的原始数据。为此，*Map-Reduce* 作为一种对海量数据处理方式的抽象被提了出来。

Google 将对海量数据的处理分成了两个操作原语：*Map* & *Reduce*

* Map：将原始数据的 *Key-Value* 作为输入，输出经过处理的中间数据并以 *Key-Value Pair* 的形式
* Reduce：将 Map 输出的中间数据作为输入，输出处理的结果

以论文中的 Word Count 程序为例，输入文本后，Word Count 统计每个单词出现的次数

1. *Map* 程序将输入的文本切割成 *Key-Value* 列表，*Key* 表示单词，*Value*  表示次数
2. *Reduce* 程序根据 *Key* 将中间数据的 *Key-Value* 聚合成 *Key: List[Value]* 并根据 *List* 长度来统计单词出现的次数

## Map Reduce 的挑战

Google 通过 *Map-Reduce* 这层抽象提供了一种通用的数据处理模型，但同时也引入了很多新的挑战。对于 *Map-Reduce* 的场景来说，为了快速处理数据，需要使用大量计算机来并行处理，而大规模的计算机集群使得出错无处不在：网络、内存、电源等等。为了提高容错和鲁棒性，需要在设计阶段将其考虑在内。

## Map Reduce 实现

实现 *Map-Reduce* 的过程中，有一些经验值得分享，包含如何实现 RPC 调用、任务分配、Fault Tolerance 以及如何退出

### RPC 调用

Coordinator 与 Worker 之间通过 Remote Procedure Call 来进行通信。由于 Worker 的数量、IP 以及 Port 不可知，Coordinator 只能被动接收 Worker 的消息，并作出响应。

这次的 Lab 使用 Go 自带的 RPC 以及序列化库 *gob* 来实现 RPC，Worker 需要从 Coordinator 申请 Task，比较优雅的实现是使用 Go Interface，并通过 Go Assert 对接收到的 Interface Instance 进行断言来区分是 MapTask 还是 ReduceTask。

```go
type BaseTask interface {
    GetTaskID() TaskID
    GetTaskType() TaskType
}

type MapTask struct {
    TaskID TaskID
    Input  string
    Output []string
}

func (m *MapTask) GetTaskID() TaskID {
    return m.TaskID
}

func (m *MapTask) GetTaskType() TaskType {
    return Map
}

type ReduceTask struct {
    TaskID TaskID
    Input  []string
    Output string
}

func (r *ReduceTask) GetTaskID() TaskID {
    return r.TaskID
}

func (r *ReduceTask) GetTaskType() TaskType {
    return Reduce
}
```

若要在 *gob* 中使用自定义的 Interface，则需要在序列化之前将对应的类型注册在 *gob*。

```go
gob.Register(&MapTask{})
gob.Register(&ReduceTask{})
```

### 任务分配

每个 Task 有三种状态：*Unassigned*、*Processing*、*Completed*

当 Task 被初始化后将处于 *Unassigned* 状态，等待 Coordinator 分配；分配后，*Unassigned* 状态将转换为 *Processing* 状态；当 Task 完成后则会置为 *Completed*

当 Task 满足一下两个条件中的任意一个时，则认为这个 Task 是 Assignable 的：

* Task 处于 *Unassigned*
* Task 处于 *Processing* 但 *Keep Alive* 超时

在整个系统中存在两个 ID：TaskID 与 WorkerID，之所以存在 Worker ID是因为 Worker 可能会因为某些原因失败或者 Crash。Coordinator 需要根据 Worker ID来判断请求是否合法。

#### 分配的 Workflow

1. 寻找 Assignable 的 Map Task，有过存在则直接返回
2. 若存在正在运行的 Map Task，返回空任务等待
3. 寻找 Assignable 的 Map Task，有过存在则直接返回
4. 若存在正在运行的 Reduce Task，返回空任务等待
5. 将 Done 置为 `true`， Coordinator 退出

### Fault Tolerance

* 运行超时，在 Coordinator 中保存每个 Task 的 *Last ping time*。当 Task 没有被 Keep Alive 时重新将 Task 分配给其他 Worker，并更新 Task Info 中的 Worker ID
