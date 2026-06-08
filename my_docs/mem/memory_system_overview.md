# gem5 内存系统总览

## 1. 对应代码目录

内存系统相关代码主要在：

- `src/mem`
- `src/mem/cache`
- `src/mem/ruby`
- `src/mem/ruby/protocol`
- `src/mem/ruby/network`
- `src/mem/ruby/system`
- `src/mem/slicc`
- `src/mem/qos`

这一层负责“请求进入内存系统后如何被缓存、转发、一致性处理、排队和发送到内存介质”。

## 2. 先说结论

gem5 的内存系统可以先分成两条主线：

- classic memory system：常规 cache / bus / memory controller 路径。
- Ruby memory system：协议 controller + message buffer + NoC + SLICC 协议路径。

两条路径都使用 `Request`、`Packet`、`Port` 这些基础抽象，但 Ruby 会把一致性协议和片上网络建模得更明确。

## 3. 基础抽象

先认识这些基础文件：

- `src/mem/request.hh`
- `src/mem/packet.hh`
- `src/mem/port.hh`
- `src/mem/abstract_mem.hh`
- `src/mem/physical.cc`

可以粗略理解为：

- `Request` 描述一次访问的意图，比如地址、大小、requestor、flags。
- `Packet` 是在内存系统里传输的消息载体。
- `Port` 是 SimObject 之间发送和接收 packet 的连接点。

CPU、cache、xbar、RubyPort、memory controller 之间主要通过这些抽象衔接。

## 4. Classic cache 路径

主要目录：

- `src/mem/cache`
- `src/mem/cache/tags`
- `src/mem/cache/replacement_policies`
- `src/mem/cache/prefetch`
- `src/mem/cache/compressors`

关键入口：

- `src/mem/cache/base.hh`
- `src/mem/cache/base.cc`
- `src/mem/cache/cache.hh`
- `src/mem/cache/cache.cc`
- `src/mem/cache/mshr.hh`
- `src/mem/cache/mshr_queue.hh`
- `src/mem/cache/write_queue.hh`

Classic cache 可以先按这条线理解：

```text
CPU port
 -> cache cpu-side port
 -> tag lookup
 -> hit 直接响应
 -> miss 分配 MSHR
 -> 向下一级 cache / xbar / memory controller 发请求
```

这一条路径更适合看 cache 命中、miss、MSHR、write buffer、replacement、prefetcher 这些传统 cache 行为。

## 5. Ruby 路径

主要目录：

- `src/mem/ruby/system`
- `src/mem/ruby/slicc_interface`
- `src/mem/ruby/protocol`
- `src/mem/ruby/network`
- `src/mem/ruby/structures`
- `src/mem/slicc`

Ruby 的重点不是“一个 cache 类处理所有逻辑”，而是：

```text
Sequencer / RubyPort
 -> protocol controller
 -> MessageBuffer
 -> Ruby network
 -> target controller
 -> cache / directory / memory side structure
```

关键入口：

- `src/mem/ruby/system/RubySystem.py`
- `src/mem/ruby/system/RubyPort.hh`
- `src/mem/ruby/system/Sequencer.hh`
- `src/mem/ruby/slicc_interface/AbstractController.hh`
- `src/mem/ruby/protocol`
- `src/mem/ruby/network/Network.hh`

Ruby 协议通常用 SLICC 描述，生成 controller 相关代码。不同协议目录文件名里会出现 `L1cache.sm`、`L2cache.sm`、`dir.sm`、`msg.sm` 等。

已有更具体说明：

- [CPU 到 L3 / 内存的简单流程](ruby/gem5_cpu_l3_controller_flow.md)
- [NoC 入门总览](ruby/gem5_noc_overview.md)

## 6. Memory controller 和介质接口

主要文件：

- `src/mem/MemCtrl.py`
- `src/mem/mem_ctrl.hh`
- `src/mem/mem_ctrl.cc`
- `src/mem/MemInterface.py`
- `src/mem/DRAMInterface.py`
- `src/mem/dram_interface.hh`
- `src/mem/qos`

`MemCtrl` 是单通道、单端口的 memory controller 模型。它维护读写队列，处理调度策略、读写切换阈值、前后端固定延迟，并通过具体 memory interface 表示 DRAM/NVM 等介质。

配置脚本里的 `--mem-type`、`--mem-channels`、`--mem-size` 等参数最终会影响这些对象如何被实例化。

## 7. Classic 和 Ruby 的区别

可以先记住这个简化对比：

```text
classic: CPU -> cache/xbar 层级 -> memory controller
ruby:    CPU -> RubyPort/Sequencer -> protocol controller -> network -> controller -> memory
```

classic 更直接，适合看常规 cache 层级。Ruby 更适合研究一致性协议、controller 状态机和 NoC 传输。

## 8. 建议阅读顺序

如果想先理解普通 cache：

1. `src/mem/request.hh`
2. `src/mem/packet.hh`
3. `src/mem/port.hh`
4. `src/mem/cache/base.hh`
5. `src/mem/cache/cache.cc`
6. `src/mem/cache/mshr.hh`
7. `src/mem/MemCtrl.py`

如果想理解 Ruby：

1. `src/mem/ruby/system/Sequencer.hh`
2. `src/mem/ruby/system/RubyPort.hh`
3. `src/mem/ruby/slicc_interface/AbstractController.hh`
4. `src/mem/ruby/protocol`
5. `src/mem/ruby/network/Network.hh`
6. `src/mem/ruby/network/garnet`
