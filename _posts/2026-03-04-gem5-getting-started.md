---
layout: post
title: 'gem5 入门：环境、构建与第一次标准库仿真'
date: 2026-03-04 21:24:00 +0800
description: 从源码构建 gem5，并使用 gem5 standard library 组装处理器、缓存、内存和 workload，完成第一次仿真。
tags: gem5 simulation computer-architecture python
categories: computer-architecture
---

这篇笔记记录从源码开始安装、构建 gem5，并用 standard library 跑通一个最小系统的过程。示例基于 gem5 24.x 之后的组件化接口；后续版本的类名和 API 可能会调整，遇到差异时应以当前版本官方文档为准。

## 获取源码

```bash
git clone https://github.com/gem5/gem5
cd gem5
```

gem5 完整构建后会占用比较可观的磁盘空间，因此最好提前准备足够的空间。

## 安装构建依赖

官方常用开发环境包括 Ubuntu 22.04 和 Ubuntu 24.04。

Ubuntu 24.04 可以安装：

```bash
sudo apt install build-essential git m4 scons zlib1g zlib1g-dev \
    libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev \
    python3-dev libboost-all-dev pkg-config python3-tk clang-format-15
```

Ubuntu 22.04：

```bash
sudo apt install build-essential git m4 scons zlib1g zlib1g-dev \
    libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev \
    python3-dev libboost-all-dev pkg-config python3-tk clang-format
```

如果服务器上没有管理员权限，而 GCC、Git、Python 等基础环境已经具备，可以先通过用户级 pip 安装 SCons：

```bash
python3 -m pip install --user scons
python3 -m pip install --user -r requirements.txt
```

如果 `~/.local/bin` 不在 `PATH` 中：

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

检查：

```bash
scons --version
```

## 第一次构建

gem5 常见构建形式为：

```bash
scons build/{ISA}/gem5.{variant} -j $(nproc)
```

初学阶段可以直接构建 ALL：

```bash
scons build/ALL/gem5.opt -j $(nproc)
```

`gem5.opt` 是日常研究中很常见的优化构建版本，在性能和调试能力之间做了平衡。

## 用 Standard Library 组装系统

较新的 gem5 standard library 把 processor、memory、cache hierarchy、board、resource 和 simulator 都封装成组件，可以比传统手写 `System` 配置更快地搭建实验系统。

先导入组件：

```python
from gem5.components.boards.simple_board import SimpleBoard
from gem5.components.processors.simple_processor import SimpleProcessor
from gem5.components.cachehierarchies.ruby.mesi_two_level_cache_hierarchy import (
    MESITwoLevelCacheHierarchy,
)
from gem5.components.memory.single_channel import SingleChannelDDR4_2400
from gem5.components.processors.cpu_types import CPUTypes
from gem5.isas import ISA
from gem5.resources.resource import obtain_resource
from gem5.simulate.simulator import Simulator
```

### Cache hierarchy

```python
cache_hierarchy = MESITwoLevelCacheHierarchy(
    l1d_size="16KiB",
    l1d_assoc=8,
    l1i_size="16KiB",
    l1i_assoc=8,
    l2_size="256KiB",
    l2_assoc=16,
    num_l2_banks=1,
)
```

### Memory

```python
memory = SingleChannelDDR4_2400()
```

### Processor

```python
processor = SimpleProcessor(
    cpu_type=CPUTypes.TIMING,
    isa=ISA.ARM,
    num_cores=1,
)
```

### Board

```python
board = SimpleBoard(
    clk_freq="3GHz",
    processor=processor,
    memory=memory,
    cache_hierarchy=cache_hierarchy,
)
```

Board 把 processor、memory 和 cache hierarchy 组织到一起，省去了很多手动连接 SimObject 的工作。

## 设置 workload

可以通过 gem5 resources 获取官方维护的 workload：

```python
board.set_workload(obtain_resource("arm-gapbs-bfs-run"))
```

## 创建并运行 Simulator

```python
simulator = Simulator(board=board)
simulator.run()
```

把脚本保存为例如 `configs/tutorial/components.py` 后运行：

```bash
./build/ALL/gem5.opt configs/tutorial/components.py
```

能够进入 event queue 并最终完成 workload，就说明第一次 standard library 仿真已经跑通。

## Standard Library 源码在哪里

组件实现主要位于：

```text
src/python/gem5/components/
├── boards/
├── cachehierarchies/
├── memory/
└── processors/
```

当某个组件的参数或连接方式不清楚时，我更倾向于直接阅读这里的 Python 源码。相比只看 API 名称，源码能更快说明它到底实例化了哪些 SimObject、暴露了哪些端口，以及默认参数是什么。

## 下一步

跑通这一层之后，可以继续学习：

- 自定义 `SimObject`；
- gem5 的事件驱动模型；
- 自定义 cache / memory hierarchy；
- RISC-V full-system 仿真；
- 统计信息、功耗和可靠性模型接入。

对我来说，真正理解 gem5 的关键不是“会运行一个脚本”，而是逐步弄清楚 Python 配置层如何生成 C++ SimObject 图，以及这些对象如何通过 event queue 推进整个仿真。
