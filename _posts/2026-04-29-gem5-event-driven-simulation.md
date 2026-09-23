---
layout: post
title: '理解 gem5 的事件驱动仿真：Event、Callback 与 Schedule'
date: 2026-04-30 01:47:00 +0800
description: 从 event queue 出发理解 gem5 如何推进离散事件，并给自定义 SimObject 添加 EventFunctionWrapper、callback 和周期性调度。
tags: gem5 event-driven simulation c++
categories: computer-architecture
---

在创建完一个最简单的 `HelloObject` 之后，下一步就是让这个对象真正“动起来”。gem5 的核心是离散事件仿真，因此理解 `Event`、callback、`schedule()` 与 `curTick()`，几乎是理解 gem5 C++ 模型行为的入口。

## gem5 的事件队列

gem5 不需要在每一个宿主机时钟周期里把整个硬件系统从头计算一遍。相反，它维护一个按照 **仿真时间（tick）与事件优先级** 排序的事件队列。

当仿真推进时，基本过程可以理解为：

1. 从 event queue 中取出当前最早需要处理的事件；
2. 将仿真时间推进到该事件对应的 tick；
3. 执行事件对应的 callback；
4. callback 可以修改模型状态，并调度新的未来事件；
5. 重复上述过程。

因此，一次 cache miss、一次内存响应、一个 pipeline stage 的完成，都可以通过未来某个 tick 上的事件来表达。

## Callback 与 Schedule

一个事件通常至少包含两个问题：

- **发生时做什么？** —— callback；
- **什么时候发生？** —— schedule。

概念上可以写成：

```text
Event A at t0
    |
    +-- callback()
            |
            +-- schedule(Event B, t0 + latency)
```

也就是说，事件处理函数不仅可以改变当前状态，也可以继续产生新的事件，从而形成完整的时序行为。

## 给 HelloObject 添加一个事件

先在头文件中加入 callback、`EventFunctionWrapper` 和 `startup()`：

```cpp
class HelloObject : public SimObject
{
  private:
    void processEvent();

    EventFunctionWrapper event;

  public:
    HelloObject(const HelloObjectParams &p);

    void startup() override;
};
```

这里：

- `processEvent()` 是真正执行工作的 callback；
- `event` 是一个 gem5 事件对象；
- `startup()` 会在仿真启动阶段被调用，我们可以在这里安排第一个事件。

## 在构造函数中绑定 callback

```cpp
HelloObject::HelloObject(const HelloObjectParams &params) :
    SimObject(params),
    event([this] { processEvent(); }, name())
{
    DPRINTF(HelloExample, "Created the hello object\n");
}
```

`EventFunctionWrapper` 的第一个参数是一个可调用对象。这里使用 lambda：

```cpp
[this] { processEvent(); }
```

通过捕获 `this`，事件触发时就可以调用当前 `HelloObject` 实例的成员函数。

第二个参数 `name()` 用来提供事件名称，便于调试与追踪。

## 实现事件处理函数

```cpp
void
HelloObject::processEvent()
{
    DPRINTF(HelloExample, "Hello world! Processing the event!\n");
}
```

现在事件已经知道“发生时做什么”，还需要决定“什么时候发生”。

## 在 startup() 中调度第一次事件

```cpp
void
HelloObject::startup()
{
    schedule(event, 100);
}
```

这表示把 `event` 安排在 tick 100 执行。

运行时启用对应 debug flag：

```bash
build/X86/gem5.opt \
    --debug-flags=HelloExample \
    configs/learning_gem5/part2/run_hello.py
```

输出中会看到事件在指定 tick 被处理。

## 周期性重新调度

更实际的模型通常不会只触发一次事件。可以给对象增加延迟和剩余次数：

```cpp
class HelloObject : public SimObject
{
  private:
    void processEvent();

    EventFunctionWrapper event;

    const Tick latency;
    int timesLeft;

  public:
    HelloObject(const HelloObjectParams &p);

    void startup() override;
};
```

构造函数中初始化：

```cpp
HelloObject::HelloObject(const HelloObjectParams &params) :
    SimObject(params),
    event([this] { processEvent(); }, name()),
    latency(100),
    timesLeft(10)
{
    DPRINTF(HelloExample, "Created the hello object\n");
}
```

第一次事件：

```cpp
void
HelloObject::startup()
{
    schedule(event, latency);
}
```

事件处理完成后继续安排下一次：

```cpp
void
HelloObject::processEvent()
{
    timesLeft--;

    DPRINTF(
        HelloExample,
        "Hello world! Processing the event! %d left\n",
        timesLeft
    );

    if (timesLeft <= 0) {
        DPRINTF(HelloExample, "Done firing!\n");
    } else {
        schedule(event, curTick() + latency);
    }
}
```

关键就在：

```cpp
schedule(event, curTick() + latency);
```

`curTick()` 表示当前仿真时间，因此这个写法等价于“从现在开始再过 `latency` 个 tick 重新触发事件”。

## 为什么这个模型重要

一旦理解 event queue，就会发现很多 gem5 模型都可以用同一套思维理解：

- Cache lookup 在若干 tick 后产生 hit/miss 结果；
- Memory controller 在请求到达后调度后续响应；
- CPU pipeline 在未来 tick 唤醒下一阶段；
- 网络模型通过事件表达 packet/link latency；
- 周期性 monitor 可以不断自我调度。

这也是 gem5 能够在不逐门级模拟硬件的情况下，仍然表达复杂时序关系的核心机制之一。

对后续自己实现延迟、带宽、拥塞、monitor 或可靠性模型来说，`Event` 与 `schedule()` 会是最常用的基础工具。
