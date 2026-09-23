---
layout: post
title: 'gem5 学习笔记：从零创建一个自定义 SimObject'
date: 2026-01-23 15:59:00 +0800
description: 从必要的 C++ 概念出发，完成 HelloObject 的 C++ 实现、Python 包装、SCons 注册与配置脚本。
tags: gem5 simobject c++ simulation
categories: computer-architecture
---

这篇文章记录我第一次在 gem5 中创建自定义 `SimObject` 的完整过程。目标不是介绍复杂模型，而是用最小的 `HelloObject` 把 gem5 中 **C++ 模型 → Python 配置接口 → SCons 构建系统 → 仿真脚本** 这条链路跑通。

参考：[gem5 Learning gem5 — Creating a very simple SimObject](https://www.gem5.org/documentation/learning_gem5/part2/helloobject/)

## 开始前需要理解的几个 C++ 概念

### 访问修饰符

C++ 类常见的访问权限有三种：

| 修饰符 | 可访问范围 | 常见用途 |
| --- | --- | --- |
| `public` | 类内部、派生类、外部代码 | 对外接口 |
| `protected` | 类内部、派生类 | 继承体系内部使用 |
| `private` | 仅类内部 | 内部状态与实现细节 |

### 命名空间与 `::`

命名空间用于区分不同作用域中的同名符号：

```cpp
namespace demo {
int value = 1;
}

int x = demo::value;
```

`::` 是作用域解析运算符，在 gem5 源码中会频繁看到，例如 `gem5::HelloObject`。

### 构造函数与初始化列表

构造函数在对象创建时执行。初始化列表写在构造函数参数列表之后、函数体之前：

```cpp
class Line {
  public:
    Line(double len) : length(len) {}

  private:
    double length;
};
```

对 gem5 的 `SimObject` 来说，这一点非常重要，因为派生类通常需要先调用基类 `SimObject` 的构造函数。

### 常量引用

```cpp
const int &ref = value;
```

常量引用避免不必要的对象复制，同时保证调用者不能通过该引用修改原对象。gem5 生成的参数对象通常以这种形式传入 C++ 构造函数。

## 1. 定义 C++ SimObject

先创建头文件，例如：

```cpp
#ifndef __LEARNING_GEM5_HELLO_OBJECT_HH__
#define __LEARNING_GEM5_HELLO_OBJECT_HH__

#include "params/HelloObject.hh"
#include "sim/sim_object.hh"

namespace gem5
{

class HelloObject : public SimObject
{
  public:
    HelloObject(const HelloObjectParams &p);
};

} // namespace gem5

#endif // __LEARNING_GEM5_HELLO_OBJECT_HH__
```

这里最关键的两点是：

1. `HelloObject` 继承自 `SimObject`；
2. 构造函数接收 `HelloObjectParams`，这个参数类型会根据 Python 侧的 SimObject 定义由 gem5 构建系统生成。

接下来实现 `.cc` 文件：

```cpp
#include "learning_gem5/part2/hello_object.hh"

#include <iostream>

namespace gem5
{

HelloObject::HelloObject(const HelloObjectParams &params) :
    SimObject(params)
{
    std::cout << "Hello World! From a SimObject!" << std::endl;
}

} // namespace gem5
```

这里的

```cpp
SimObject(params)
```

就是在初始化列表中调用父类构造函数。

在真实 gem5 开发中通常不推荐直接使用 `std::cout`，而应该使用 gem5 的 debug flag 和 `DPRINTF`；这里为了最小示例先保留输出语句。

## 2. 创建 Python 包装类

gem5 的配置层主要由 Python 驱动，因此还需要定义 Python 侧的 SimObject：

```python
from m5.params import *
from m5.SimObject import SimObject


class HelloObject(SimObject):
    type = "HelloObject"
    cxx_header = "learning_gem5/part2/hello_object.hh"
    cxx_class = "gem5::HelloObject"
```

这一步建立了 Python 配置对象和 C++ 实现之间的对应关系。

## 3. 在 SConscript 中注册文件

gem5 使用 SCons 构建。创建或修改当前目录中的 `SConscript`：

```python
Import("*")

SimObject("HelloObject.py", sim_objects=["HelloObject"])
Source("hello_object.cc")
```

如果没有注册，源码文件和 Python SimObject 定义不会进入对应的 gem5 构建目标。

## 4. 重新构建 gem5

```bash
scons build/ALL/gem5.opt -j $(nproc)
```

也可以针对某个 ISA 构建对应目标。

## 5. 写一个最小配置脚本

```python
import m5
from m5.objects import *

root = Root(full_system=False)
root.hello = HelloObject()

m5.instantiate()

print("Beginning simulation!")
exit_event = m5.simulate()
print(
    "Exiting @ tick {} because {}".format(
        m5.curTick(), exit_event.getCause()
    )
)
```

`Root` 是 gem5 对象树的顶层对象。我们创建的 `HelloObject` 需要挂到这棵对象树上，然后通过 `m5.instantiate()` 完成 C++ SimObject 的实例化。

运行：

```bash
build/ALL/gem5.opt configs/learning_gem5/part2/run_hello.py
```

如果能够看到 `Hello World! From a SimObject!`，说明从 Python 配置到 C++ 对象实例化的完整链路已经跑通。

## 小结

创建一个自定义 SimObject 的核心流程可以压缩成四步：

1. 在 C++ 中继承 `SimObject` 并实现模型；
2. 在 Python 中声明对应的 SimObject 类型；
3. 在 `SConscript` 中注册 Python 与 C++ 文件；
4. 重新构建 gem5，并在配置脚本中实例化对象。

理解这条链路以后，再去添加参数、端口、事件和统计信息就会清晰很多。
