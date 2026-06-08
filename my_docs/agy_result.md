# gem5 代码文档汇总 (agy_result)

> 本文件由 `my_docs/` 目录下所有文档汇总而成，按原始目录结构组织，方便一站式阅读。

---

## 目录结构

```
my_docs/
├── README.md                              → 索引与建议阅读顺序
├── arch/
│   └── riscv/
│       ├── riscv_arch_overview.md         → RISC-V 架构实现总览
│       └── riscv_memory_model_impl.md     → RISC-V 内存模型入门
├── configs/
│   └── configs_overview.md               → 配置脚本总览
├── cpu/
│   └── cpu_models_overview.md            → CPU 模型总览
├── dev/
│   └── device_io_dma_overview.md         → 设备与 IO 模型总览
├── mem/
│   ├── memory_system_overview.md         → 内存系统总览
│   └── ruby/
│       ├── gem5_cpu_l3_controller_flow.md → CPU 到 L3 / 内存的简单流程
│       └── gem5_noc_overview.md           → NoC 入门总览
└── sim/
    └── simulation_core_overview.md       → 仿真核心总览
```

---

# 1. gem5 代码分类文档索引

> 来源：`my_docs/README.md`

这个目录按 gem5 源码的大类来组织，不追求和代码目录一一对应到最细层级。目标是先给读代码的人建立入口，再从入口跳到更具体的实现文档。

## 当前分类

### `arch/riscv`

对应代码目录主要是 `src/arch/riscv`。

- RISC-V 架构实现总览（见本文第 2 节）
- RISC-V 内存模型入门（见本文第 3 节）

这一类文档关注 ISA、指令解码、CSR、异常、中断、TLB、PMP/PMA，以及 RISC-V 指令如何变成 CPU 可执行的 `StaticInst`。

### `cpu`

对应代码目录主要是 `src/cpu`。

- CPU 模型总览（见本文第 4 节）

这一类文档关注 `BaseCPU`、Simple CPU、O3 CPU、Minor CPU、KVM CPU、线程上下文、执行上下文、流水线和访存端口。

### `mem`

对应代码目录主要是 `src/mem`，其中 Ruby 相关内容放在 `mem/ruby` 下。

- 内存系统总览（见本文第 5 节）
- CPU 到 L3 / 内存的简单流程（见本文第 6 节）
- NoC 入门总览（见本文第 7 节）

这一类文档关注经典 cache、Ruby、一致性协议、NoC、memory controller、DRAM/NVM 接口、packet/request/port。

### `dev`

对应代码目录主要是 `src/dev`。

- 设备与 IO 模型总览（见本文第 8 节）

这一类文档关注 PIO/MMIO 设备、DMA 设备、PCI 设备、串口、网卡、磁盘、平台中断、设备树和 board/platform 侧的设备挂接。

### `configs`

对应代码目录主要是 `configs`。

- 配置脚本总览（见本文第 9 节）

这一类文档关注 Python 配置脚本如何创建 `System`、CPU、cache、Ruby、内存控制器、拓扑和 benchmark/workload。

### `sim`

对应代码目录主要是 `src/sim`，也会涉及 `src/python/m5`。

- 仿真核心总览（见本文第 10 节）

这一类文档关注 `SimObject`、事件队列、仿真生命周期、checkpoint、drain、statistics、clock/power domain。

## 建议阅读顺序

如果是从一次普通 gem5 仿真开始理解，可以按这个顺序看：

1. `configs`：先看脚本怎样搭系统。
2. `sim`：理解对象树、初始化和事件驱动。
3. `cpu`：看 CPU 如何执行指令和发起访存。
4. `arch/riscv`：看 RISC-V 指令语义和地址转换。
5. `mem`：看请求进入 cache、Ruby、NoC 和 memory controller 后怎么走。
6. `dev`：看外设如何通过 MMIO、DMA、PCI、中断和平台代码接入系统。

如果只关心 Ruby / NoC，可以直接从内存系统总览和 `mem/ruby` 下两篇开始。

---

# 2. RISC-V 架构实现总览

> 来源：`my_docs/arch/riscv/riscv_arch_overview.md`

## 1. 对应代码目录

RISC-V 架构相关代码主要在：

- `src/arch/riscv`
- `src/arch/riscv/isa`
- `src/arch/riscv/isa/formats`
- `src/arch/riscv/insts`
- `src/arch/riscv/regs`

这一层负责"RISC-V 指令和架构状态是什么"，不是负责 cache、内存控制器或 NoC 的实现。

## 2. 先说结论

gem5 的 RISC-V 实现可以按 4 层理解：

```text
ISA 参数和架构状态
 -> 指令解码
 -> 指令语义和 StaticInst
 -> 异常 / 中断 / TLB / PMP / PMA 等架构机制
```

CPU 模型会调用这些指令语义，但不同 CPU 模型是否乱序、如何调度、什么时候发访存请求，是 `src/cpu` 的职责。

## 3. 核心代码入口

### ISA 参数

- `src/arch/riscv/RiscvISA.py`
- `src/arch/riscv/isa.hh`
- `src/arch/riscv/isa.cc`

这里定义 RV32/RV64、向量扩展、特权模式集合、部分扩展开关，以及 CSR / misc register 的读写逻辑。

### 指令解码

- `src/arch/riscv/isa/decoder.isa`
- `src/arch/riscv/isa/bitfields.isa`
- `src/arch/riscv/isa/operands.isa`
- `src/arch/riscv/isa/main.isa`

`decoder.isa` 是理解指令分类的主入口。它把机器码按 opcode、funct、压缩指令 quadrant 等字段分派到具体指令格式。

### 指令格式和执行模板

- `src/arch/riscv/isa/formats/mem.isa`
- `src/arch/riscv/isa/formats/amo.isa`
- `src/arch/riscv/isa/formats/vector_mem.isa`
- `src/arch/riscv/isa/formats/basic.isa`
- `src/arch/riscv/isa/templates`

这些文件负责生成具体指令类的声明、构造函数、`execute()`、`initiateAcc()`、`completeAcc()` 等代码。普通算术指令、load/store、AMO、vector memory 指令会走不同模板。

### 指令类和架构对象

- `src/arch/riscv/insts`
- `src/arch/riscv/regs`
- `src/arch/riscv/types.hh`
- `src/arch/riscv/pcstate.hh`

这里放生成代码之外的一些指令基类、寄存器定义、PC 状态和架构类型。

### 地址转换和保护

- `src/arch/riscv/tlb.hh`
- `src/arch/riscv/tlb.cc`
- `src/arch/riscv/pagetable.hh`
- `src/arch/riscv/pagetable_walker.hh`
- `src/arch/riscv/pmp.hh`
- `src/arch/riscv/pma_checker.hh`

这部分处理虚拟地址到物理地址的转换，以及 PMP/PMA 权限和属性检查。CPU 发起访存时，最终会依赖 MMU/TLB 路径决定访问能否继续。

## 4. 一条执行主线

可以先按这条线理解 RISC-V 代码：

```text
机器码
 -> decoder.isa 解码
 -> 生成 StaticInst
 -> CPU 通过 ExecContext 执行 StaticInst
 -> 如果是访存指令，形成地址和内存访问标记
 -> MMU/TLB/PMP/PMA 检查
 -> 进入 CPU / memory system 的访存路径
```

关键点是：RISC-V 目录描述"指令该做什么"，CPU 目录描述"指令什么时候、以什么方式执行"。

## 5. 和内存相关的边界

RISC-V 的 load/store、fence、LR/SC、AMO 在 ISA 层有自己的语义入口，但完整内存顺序行为不是只在 `src/arch/riscv` 一个目录里实现的。

- 指令语义和访问标记在 `src/arch/riscv/isa/formats`。
- 执行、乱序、LSQ、提交等在 `src/cpu`。
- cache、一致性、NoC、memory controller 在 `src/mem`。

更具体的内存模型说明见第 3 节。

## 6. 建议阅读顺序

1. `src/arch/riscv/RiscvISA.py`
2. `src/arch/riscv/isa/decoder.isa`
3. `src/arch/riscv/isa/formats/mem.isa`
4. `src/arch/riscv/isa/formats/amo.isa`
5. `src/arch/riscv/tlb.hh`
6. `src/arch/riscv/isa.cc`

先把"指令怎样被识别、怎样生成执行代码、访存怎样进入地址转换"看明白，再深入 CSR、异常、中断和特权模式细节。

---

# 3. gem5 RISC-V 内存模型入门

> 来源：`my_docs/arch/riscv/riscv_memory_model_impl.md`

代码分类：`src/arch/riscv` 为主，同时会关联 `src/cpu` 和 `src/mem`。

## 1. 先说结论

gem5 里没有一个单独的 "RISC-V 内存模型引擎"。更容易上手的理解方式是：

- ISA 负责把指令翻译成访问行为
- CPU 模型负责决定能不能乱序
- cache / memory system 负责真正完成访问

所以，RISC-V 的内存顺序语义是分散实现的，不是集中在一个大模块里。

## 2. 先记住 4 件事

### 普通 `load/store`

普通访存最常见，也最基础。它们负责"去读什么、写什么"，但不单独决定整个系统的顺序模型。

### `fence`

`fence` 可以先理解成一条"栅栏"指令。它的作用是让前后的访存不要随便跨过去。

### `LR/SC` 和 `AMO`

这两类是原子访存：

- `LR/SC` 常用于锁和同步
- `AMO` 是原子读改写

它们走的是专门路径，不是普通 `load/store` 的简单拼接。

### 设备或不可缓存内存

这类访问通常会更保守、更严格，不会像普通缓存内存那样自由重排。

## 3. 初学时最值得认识的几类指令

### `fence`

先记住它的直觉意义就行：让程序在这里"停一停"，保证前后内存操作的相对顺序。

### `fence.i`

它和指令流更相关。入门时可以把它理解成"让 CPU 前端不要继续使用旧的指令视图"。

### `aq/rl`

这是原子操作上的顺序标记。你可以把它们理解成"给原子操作补上更强的顺序约束"。

### `LR/SC`

它是一对配套的原子指令：

- `LR` 先读，并建立 reservation
- `SC` 再尝试写，只有 reservation 还有效时才成功

### `AMO`

`AMO` 会把"读、改、写"当成一个原子动作处理。

## 4. 用一条线把它串起来

可以先按这条线理解：

`RISC-V 指令 -> CPU 执行 -> 内存请求 -> cache / memory system -> 返回结果`

其中：

- 普通访存决定"访问什么"
- `fence` 决定"什么时候不能乱序"
- `LR/SC` 和 `AMO` 决定"哪些访问必须原子完成"

## 5. 如果你想顺着看代码

入门先看下面几个文件就够了：

- `src/arch/riscv/isa/decoder.isa`
- `src/arch/riscv/isa/formats/mem.isa`
- `src/arch/riscv/isa/formats/amo.isa`
- `src/arch/riscv/isa.cc`

不用一开始就把 O3、Ruby、cache 细节全看完。先把"有哪些指令、它们大概做什么"弄清楚，后面再往下钻会轻松很多。

## 6. 一句话总结

gem5 的 RISC-V 内存模型更适合被理解成"多层一起实现"。入门阶段先掌握普通访存、`fence`、`LR/SC`、`AMO` 这四类概念，就足够建立整体感觉。

---

# 4. gem5 CPU 模型总览

> 来源：`my_docs/cpu/cpu_models_overview.md`

## 1. 对应代码目录

CPU 相关代码主要在：

- `src/cpu`
- `src/cpu/simple`
- `src/cpu/o3`
- `src/cpu/minor`
- `src/cpu/kvm`
- `src/cpu/checker`
- `src/cpu/pred`

这一层负责"CPU 怎样执行指令、怎样推进时间、怎样向内存系统发请求"。

## 2. 先说结论

gem5 里 CPU 不是一个单一模型，而是一组不同精度和用途的模型：

- Simple CPU：实现简单，适合快速理解执行和访存接口。
- O3 CPU：乱序模型，适合研究流水线、ROB、IQ、LSQ、commit。
- Minor CPU：更轻量的时序流水线模型，结构比 O3 简单。
- KVM CPU：借助宿主机虚拟化执行，偏快速运行。
- Checker CPU：用于和主 CPU 结果交叉检查。

所有 CPU 都共享一些基础概念，比如 `BaseCPU`、线程上下文、ISA/decoder、icache/dcache 端口、统计和 CPU 切换。

## 3. 公共入口

### `BaseCPU`

- `src/cpu/BaseCPU.py`
- `src/cpu/base.hh`
- `src/cpu/base.cc`

`BaseCPU` 是所有 CPU 模型的共同基础。它定义 CPU 的 SimObject 参数、CPU id、socket id、线程数、ISA、decoder、MMU、中断控制器、icache/dcache 端口和 instruction count 相关接口。

从配置脚本看，Python 文件负责声明参数；从仿真执行看，C++ 文件负责真实对象和运行时行为。

### 执行上下文

- `src/cpu/exec_context.hh`
- `src/cpu/thread_context.hh`
- `src/cpu/simple_thread.hh`
- `src/cpu/thread_state.hh`

指令语义通常不会直接操作某个具体 CPU 类型，而是通过 `ExecContext` / `ThreadContext` 读取寄存器、写回结果、发起访存、处理 fault。

这就是 ISA 和 CPU 模型之间的关键边界。

## 4. Simple CPU

主要目录：

- `src/cpu/simple`

常见模型：

- `AtomicSimpleCPU`
- `TimingSimpleCPU`
- `NonCachingSimpleCPU`

Simple CPU 的价值是入口清晰。它不追求复杂流水线细节，更适合用来理解：

- 一条指令怎样 fetch / decode / execute
- load/store 怎样调用 CPU 的访存接口
- atomic memory mode 和 timing memory mode 的差异
- CPU 怎样连接 icache/dcache port

入门时可以先看：

- `src/cpu/simple/base.hh`
- `src/cpu/simple/base.cc`
- `src/cpu/simple/atomic.cc`
- `src/cpu/simple/timing.cc`

## 5. O3 CPU

主要目录：

- `src/cpu/o3`

O3 CPU 是乱序模型。核心结构包括：

- fetch
- decode
- rename
- IEW
- commit
- ROB
- instruction queue
- LSQ
- free list
- rename map
- branch predictor

可以先从这些文件建立结构感：

- `src/cpu/o3/cpu.hh`
- `src/cpu/o3/fetch.hh`
- `src/cpu/o3/decode.hh`
- `src/cpu/o3/rename.hh`
- `src/cpu/o3/iew.hh`
- `src/cpu/o3/commit.hh`
- `src/cpu/o3/rob.hh`
- `src/cpu/o3/lsq.hh`
- `src/cpu/o3/lsq_unit.hh`

O3 的访存顺序、load/store 队列、memory dependency 和 commit 关系都在这条线里，和 `src/arch/riscv` 的指令语义、`src/mem` 的内存系统共同决定一次访存的完整行为。

## 6. Minor / KVM / Checker

### Minor CPU

- `src/cpu/minor`

Minor 是更规整的流水线模型，适合理解按 stage 推进的时序 CPU，但复杂度低于 O3。

### KVM CPU

- `src/cpu/kvm`

KVM CPU 更偏快速执行，依赖宿主机虚拟化能力。它通常不是研究微结构细节的首选入口。

### Checker CPU

- `src/cpu/checker`

Checker CPU 用于检查主 CPU 的执行结果，适合调试 CPU 模型正确性。

## 7. CPU 到内存系统的边界

CPU 通过端口和内存系统通信。最常见的边界是：

```text
CPU instruction / data access
 -> icache_port / dcache_port
 -> classic cache 或 RubyPort / Sequencer
 -> 后续内存系统
```

`BaseCPU.py` 里能看到 `icache_port` 和 `dcache_port`，具体 CPU 模型会在执行访存指令时生成请求并从这些端口发出去。

如果使用 Ruby，CPU 请求通常会先进入 `RubySequencer` / `RubyPort`，然后交给 Ruby controller 和 NoC。相关说明见第 6 节。

## 8. 建议阅读顺序

1. `src/cpu/BaseCPU.py`
2. `src/cpu/base.hh`
3. `src/cpu/exec_context.hh`
4. `src/cpu/simple/base.hh`
5. `src/cpu/simple/timing.cc`
6. `src/cpu/o3/cpu.hh`
7. `src/cpu/o3/lsq.hh`
8. `src/cpu/o3/commit.hh`

先用 Simple CPU 建立"指令怎样跑起来"的直觉，再看 O3 的乱序结构。

---

# 5. gem5 内存系统总览

> 来源：`my_docs/mem/memory_system_overview.md`

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

这一层负责"请求进入内存系统后如何被缓存、转发、一致性处理、排队和发送到内存介质"。

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

Ruby 的重点不是"一个 cache 类处理所有逻辑"，而是：

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

---

# 6. gem5 CPU 到 L3 / 内存的简单流程

> 来源：`my_docs/mem/ruby/gem5_cpu_l3_controller_flow.md`

代码分类：`src/mem/ruby` 为主，同时关联 `src/cpu` 和 Ruby 协议 controller。

## 1. 先说结论

CPU 不会直接把请求发给 NoC，也不会直接和 L3 "裸连接"。真正负责协议交互的是 controller。

## 2. 最简单的一条链

```text
CPU
 -> RubySequencer / RubyPort
 -> L1 / L2 controller
 -> NoC
 -> HNF / L3 controller
 -> SNF / memory controller
 -> Main Memory
```

返回路径大体上反过来走。

## 3. 先搞清楚谁负责什么

### CPU

负责发起 load/store/ifetch 之类的访问。

### RubySequencer / RubyPort

负责把 CPU 的请求接进 Ruby 系统。

### controller

可以先把它理解成"协议状态机"：

- 它接收请求
- 它判断命中、miss、下一步动作
- 它决定是否要发消息到 NoC

### NoC

负责把 controller 之间的消息搬运过去。

## 4. 最容易记住的理解方式

如果你只想记一句话，就记这个：

`CPU 通过 controller 和 NoC 间接访问更远的缓存和内存`

这比把它想成 "CPU 直接找 L3" 更接近 gem5 里的真实结构。

## 5. Node 和 controller 不是一回事

这点很重要。

- `RNF`、`HNF`、`SNF` 更像系统里的节点
- controller 是节点里真正负责协议逻辑的部分

所以：

- 一个节点里可以有一个或多个 controller
- 不是每个对象都直接等于 controller

## 6. 请求大概怎么走

可以先按下面这个简单流程理解：

1. CPU 发起请求。
2. Sequencer 把请求交给 Ruby。
3. 近端 controller 先判断本地能不能解决。
4. 如果不能，就通过 NoC 发给更远的 controller。
5. L3 或内存侧处理后，再把结果一路返回。

## 7. 如果底下是 Garnet

那消息通常还会经过更细的网络步骤：

- 先进入网络接口
- 再通过 router 和 link 传输
- 到目标端后再交回 controller

入门阶段先知道"消息要经过网络接口和 router"就够了，不用一开始就把 flit 流水线全看完。

## 8. 如果你想顺着看代码

- `src/mem/ruby/system/Sequencer.cc`
- `src/mem/ruby/slicc_interface/AbstractController.hh`
- `src/mem/ruby/network/garnet/NetworkInterface.cc`
- `src/mem/ruby/network/garnet/Router.cc`

## 9. 一句话总结

这条链里最关键的不是 "CPU 直接连到 L3"，而是 "CPU 先进入 Ruby，再由 controller 和 NoC 把请求送到更远处"。

---

# 7. gem5 NoC 入门总览

> 来源：`my_docs/mem/ruby/gem5_noc_overview.md`

代码分类：`src/mem/ruby/network` 为主，配置侧关联 `configs/network` 和 `configs/topologies`。

## 1. 先说结论

gem5 里的 NoC 主要出现在 Ruby 这条内存系统路径上。最简单的理解是：

- 一致性协议决定"发什么消息"
- NoC 决定"这些消息怎么在片上网络里走"

所以 NoC 更像运输系统，不是协议本身。

## 2. 先认识两个关键词

### `simple`

这是更抽象的网络模型，适合先入门。它更强调"消息能不能送达"，而不是每个微结构细节。

### `garnet`

这是更像真实 NoC 的模型。它会把消息切成 flit，还会考虑：

- VC
- credit
- router latency
- 链路仲裁

如果以后你要做性能研究，通常会更关心 `garnet`。

## 3. NoC 在系统里大概在哪

在 Ruby 路径里，你可以先把关系记成：

`controller -> NoC -> controller`

更具体一点：

- controller 产生协议消息
- NoC 负责转发这些消息
- 目标 controller 收到消息后继续处理

## 4. 先认识 3 层

### 配置层

决定用什么网络、什么拓扑。

### 网络层

负责 router、link、路由和转发。

### 协议层

controller 通过消息队列和网络交换消息。

## 5. 初学时不要混淆的一点

NoC 不负责决定一致性行为。它不回答"该不该发这个请求"，只回答"这个请求怎么送过去"。

所以你可以先记住：

- 协议层决定语义
- NoC 层决定传输

## 6. 如果你想顺着看代码

先看这几个地方最合适：

- `configs/network/Network.py`
- `configs/topologies/Mesh_XY.py`
- `src/mem/ruby/network/simple/SimpleNetwork.py`
- `src/mem/ruby/network/garnet/GarnetNetwork.py`
- `src/mem/ruby/network/garnet/README.txt`

## 7. 一句话总结

对入门者来说，先不要急着钻 router 内部细节。先把 `simple`、`garnet`、`controller -> NoC -> controller` 这几个基本关系看明白，就已经抓住主线了。

---

# 8. gem5 设备与 IO 模型总览

> 来源：`my_docs/dev/device_io_dma_overview.md`

## 1. 对应代码目录

设备和 I/O 相关代码主要在：

- `src/dev`
- `src/dev/io_device.hh`
- `src/dev/dma_device.hh`
- `src/dev/pci`
- `src/dev/serial`
- `src/dev/storage`
- `src/dev/net`
- `src/dev/arm`
- `src/dev/riscv`
- `src/dev/x86`
- `src/dev/ps2`
- `src/dev/i2c`
- `src/dev/qemu`

这一层负责外设模型，不是 CPU，也不是 cache。它模拟的是寄存器映射设备、DMA 设备、PCI 设备、控制器、总线挂接、以及平台上的中断和设备树。

## 2. 先说结论

gem5 的设备模型大体可以按这条线理解：

```text
配置脚本创建设备
 -> 设备暴露 PIO / DMA / PCI 接口
 -> 设备挂到 bus、platform 或 PCI host
 -> CPU 通过内存映射寄存器或 DMA 和设备交互
 -> 设备在事件队列里推进状态
```

其中最核心的两种访问方式是：

- PIO / MMIO：CPU 通过地址空间读写设备寄存器
- DMA：设备自己发起对内存的读写

## 3. 设备基类怎么分

### `PioDevice`

文件：

- `src/dev/Device.py`
- `src/dev/io_device.hh`
- `src/dev/io_device.cc`

`PioDevice` 是所有寄存器映射设备的基类。它有一个 `pio` 响应端口，外部请求进入后会被 `PioPort` 转成 `read(pkt)` / `write(pkt)` 调用。也就是说，设备本身不用直接处理端口协议，只需要实现三件事：

- `getAddrRanges()`：告诉系统它响应哪些地址
- `read(PacketPtr)`：处理读
- `write(PacketPtr)`：处理写

`BasicPioDevice` 是常见简化版本，直接保存：

- `pio_addr`
- `pio_size`
- `pio_latency`

它的 `getAddrRanges()` 只返回一个连续区间，适合 UART、timer、interrupt controller 这类简单寄存器设备。

### `DmaDevice`

文件：

- `src/dev/Device.py`
- `src/dev/dma_device.hh`
- `src/dev/dma_device.cc`

`DmaDevice` 在 `PioDevice` 基础上再加一个 `dma` 请求端口。它既可以有寄存器映射区，也可以主动对主存做 DMA。很多真实设备都属于这一类，例如网卡、磁盘控制器、GPU 相关控制器、HDLCD、UFS、部分 ARM/AMDGPU 设备。

### `PciDevice`

文件：

- `src/dev/pci/device.hh`
- `src/dev/pci/PciDevice.py`

`PciDevice` 继承自 `DmaDevice`，表示 PCI 设备。它除了有设备自己的寄存器或 DMA 行为，还要维护 PCI configuration space、BAR、capability list、MSI/MSI-X、PCIe capability 等结构。

## 4. PIO 设备是怎么实现的

PIO 设备的实现路径很直接：

1. 配置脚本把设备的 `pio` 端口接到某个 bus 的 `mem_side_ports`。
2. CPU 或其他主设备发起内存映射访问。
3. 访问经过 bus 到达设备的 `pio` 端口。
4. `PioPort::recvAtomic()` 把 packet 转成设备的 `read()` 或 `write()`。
5. 设备更新内部寄存器状态，返回一个延迟。

关键实现点在 `src/dev/io_device.hh`：

- `PioPort` 是 `SimpleTimingPort`
- `recvAtomic()` 里直接调用 `device->read(pkt)` 或 `device->write(pkt)`
- `getAddrRanges()` 由设备返回，用于地址匹配

`BasicPioDevice` 的实现更简单：它把 `pio_addr` 和 `pio_size` 变成一个 `RangeSize`，所以这类设备本质上就是"固定地址窗口 + 寄存器读写函数"。

### 代表例子

- `src/dev/serial/uart.hh`
- `src/dev/serial/uart8250.cc`
- `src/dev/riscv/plic_device.cc`
- `src/dev/x86/i8254.cc`

以 UART 为例，设备内部维护一组寄存器，`read(pkt)` 和 `write(pkt)` 读写这些寄存器，并在需要时通过 platform 发中断。RISC-V 的 PLIC 设备同样是 `BasicPioDevice`，只是它的行为是中断控制器，不是串口。

## 5. DMA 设备是怎么实现的

DMA 设备的核心不在"被 CPU 读写"，而在"设备主动访问内存"。它的实现骨架是：

1. 设备调用 `dmaRead()` / `dmaWrite()`。
2. `DmaPort::dmaAction()` 把一次大传输封成一个 `DmaReqState`。
3. `DmaPort` 按 cache line 大小把它拆成多个 packet。
4. 在 timing mode 下，packet 通过 `sendTimingReq()` 逐个发出。
5. 在 atomic mode 下，直接走 `sendAtomicReq()`。
6. 如果系统支持 backdoor，`sendAtomicBdReq()` 会优先利用内存后门直接读写。

关键实现点在 `src/dev/dma_device.cc`：

- `DmaPort` 负责排队、重试、完成回调
- `DmaReqState` 记录总字节数、已经完成的字节数、请求者 ID、SID/SSID、完成事件
- `sendDma()` 根据系统 memory mode 选择 timing 或 atomic 路径
- `DmaReadFifo` 用于持续预取式 DMA 读取

### 这段实现的重点

DMA 不是一次性"搬一块内存"这么简单。gem5 会把它拆成多个块来建模，因为这样才能和缓存层级、总线仲裁、响应延迟、排队冲突对上。

`DmaPort` 的工作可以理解成：

- 负责把一笔 DMA 请求拆成多个片段
- 负责在 timing/atomic 两种模式下发包
- 负责在响应返回时统计完成进度
- 负责在全部片段完成后触发 completion event

## 6. PCI 设备是怎么实现的

PCI 设备的实现比普通 PIO 设备更复杂，因为它同时管理三层东西：

1. PCI configuration space
2. BAR 映射
3. 设备本体访问

关键结构在 `src/dev/pci/device.hh`：

- `PciBar` 抽象 base address register
- `PciIoBar`、`PciMemBar`、`PciLegacyIoBar`、`PciMemUpperBar`
- `PciDevice` 管理 config space、capability、BAR 列表
- `PciEndpoint` 和 `PciType1Device` 分别表示 endpoint 和 bridge

`PciDevice` 继承 `DmaDevice`，所以 PCI 设备通常既有 config/MMIO 访问，也有 DMA 行为。`read()` 和 `write()` 是 final 入口，内部再分发到：

- `readConfig()` / `writeConfig()`：处理 PCI 配置空间
- `readDevice()` / `writeDevice()`：处理设备自身寄存器或 BAR 访问

### 典型工作流程

1. 平台或 PCI host 给设备分配 BAR。
2. BAR 写入后，`PciBar::write()` 把配置空间值翻译成真实地址。
3. 设备把 BAR 对应的地址段注册给系统。
4. CPU 访问 BAR 地址时，设备收到对应 packet。
5. 如果设备需要搬运数据，再通过 `dma` 端口发起内存访问。

### 代表例子

- `src/dev/net/Ethernet.py`
- `src/dev/amdgpu/AMDGPU.py`
- `src/dev/storage/Ide.py`
- `src/dev/pci/device.hh`

像网卡、GPU、IDE 控制器这类设备，通常都会同时涉及 PCI config、BAR、DMA 和中断。

## 7. 平台层怎么把设备接起来

设备本身只是对象，真正把它们接到系统上的通常是 platform / board 代码。

关键文件：

- `src/dev/platform.hh`
- `src/dev/platform.cc`
- `src/dev/arm/RealView.py`
- `src/dev/riscv/HiFive.py`

`Platform` 这一层负责平台相关的中断注入，比如：

- `postConsoleInt()`
- `clearConsoleInt()`
- `postPciInt()`
- `clearPciInt()`

平台脚本再把具体设备连到 bus 或 PCI host 上。例如 RISC-V `HiFive`：

- `clint` 和 `plic` 是 on-chip 设备
- `uart`、disk、rng 这类是 off-chip 设备
- `attachOnChipIO()` 和 `attachOffChipIO()` 把设备的 `pio` 接到 bus
- `generateDeviceTree()` 生成 Linux 能读的 FDT 节点

ARM 的 `RealView` 更进一步，会同时处理：

- on-chip / off-chip IO
- PCI 设备
- DMA 设备
- SMMU
- device tree

这就是为什么同一个设备类既要有 C++ 行为，也要有 Python 配置代码：Python 负责布线，C++ 负责行为。

## 8. 常见设备家族

### 串口 / 定时器 / 中断

- `src/dev/serial`
- `src/dev/x86`
- `src/dev/riscv`
- `src/dev/arm`

这些通常是 PIO 设备，寄存器少、状态机清晰，适合看设备模型的最小闭环。

### 网卡

- `src/dev/net`

典型网卡常常是 PCI 设备，带 DMA，把包数据搬到主存，再通过中断通知 CPU。

### 存储

- `src/dev/storage`

IDE、disk image、simple disk 都在这里。控制器一般是 PCI 设备，磁盘数据通常通过 DMA 进出内存。

### ARM / RISC-V 平台设备

- `src/dev/arm`
- `src/dev/riscv`

这里会有更多 SoC 级设备，比如 GIC、PLIC、CLINT、SMMU、virtio-MMIO、power controller、timer。

### QEMU / virtio / special-purpose

- `src/dev/qemu`
- `src/dev/arm/VirtIOMMIO.py`

这类通常负责把外部生态常见设备接进 gem5。

## 9. 一条从配置到运行的链

```text
configs/*
 -> 创建 Platform / Bus / Device
 -> 设备通过 pio 或 dma 接到 bus
 -> Platform 生成 FDT / 绑定中断 / 分配地址
 -> CPU 访问设备寄存器或设备发起 DMA
 -> 设备在 event queue 中推进状态
```

## 10. 建议阅读顺序

如果是第一次看设备模型，建议按这个顺序：

1. `src/dev/Device.py`
2. `src/dev/io_device.hh`
3. `src/dev/io_device.cc`
4. `src/dev/dma_device.hh`
5. `src/dev/dma_device.cc`
6. `src/dev/platform.hh`
7. `src/dev/serial/uart8250.cc`
8. `src/dev/riscv/plic_device.cc`
9. `src/dev/pci/device.hh`
10. `src/dev/arm/RealView.py` 或 `src/dev/riscv/HiFive.py`

先看 PIO，再看 DMA，再看 PCI，最后看 platform 怎么把它们挂到系统上。

---

# 9. gem5 配置脚本总览

> 来源：`my_docs/configs/configs_overview.md`

## 1. 对应代码目录

配置脚本主要在：

- `configs`
- `configs/common`
- `configs/example`
- `configs/ruby`
- `configs/network`
- `configs/topologies`
- `configs/dram`
- `configs/nvm`
- `configs/example/gem5_library`

这一层负责"创建什么系统、用什么 CPU、什么 cache / Ruby / memory、跑什么 workload"。

## 2. 先说结论

gem5 的配置脚本不是被模拟硬件本身，而是用 Python 创建 SimObject 对象树，然后交给仿真核心实例化。

可以先按这条线理解：

```text
命令行参数
 -> Python 配置脚本
 -> 创建 System / CPU / cache / memory / workload
 -> 连接 ports
 -> m5.instantiate()
 -> m5.simulate()
```

配置脚本决定"模型长什么样"，`src` 下的 C++/Python SimObject 代码决定"模型运行时怎么行为"。

## 3. 常见目录职责

### `configs/example`

这里放常用示例脚本，比如 SE/FS 模式、Ruby test、Garnet synthetic traffic 等。

常见入口：

- `configs/example/se.py`
- `configs/example/fs.py`
- `configs/example/ruby_random_test.py`
- `configs/example/garnet_synth_traffic.py`

### `configs/common`

这里放老式 example 配置脚本共享的工具函数和参数解析逻辑。

关键文件：

- `configs/common/Options.py`
- `configs/common/Simulation.py`
- `configs/common/CacheConfig.py`
- `configs/common/MemConfig.py`
- `configs/common/CpuConfig.py`
- `configs/common/ObjectList.py`

`Options.py` 定义大量命令行参数。`Simulation.py` 处理 CPU class、memory mode、checkpoint、切换 CPU、运行控制。`CacheConfig.py` 和 `MemConfig.py` 负责创建 cache 和 memory controller。

### `configs/ruby`

这里负责 Ruby 系统配置。

关键文件：

- `configs/ruby/Ruby.py`
- `configs/ruby/CHI.py`
- `configs/ruby/MESI_Two_Level.py`
- `configs/ruby/MESI_Three_Level.py`
- `configs/ruby/MI_example.py`

`Ruby.py` 负责 Ruby 通用选项、memory controller 设置和 topology 创建。具体协议脚本负责创建对应的 controller 列表。

### `configs/network` 和 `configs/topologies`

这两个目录通常和 Ruby/Garnet 一起看。

关键文件：

- `configs/network/Network.py`
- `configs/topologies/BaseTopology.py`
- `configs/topologies/Mesh_XY.py`
- `configs/topologies/Crossbar.py`
- `configs/topologies/Pt2Pt.py`

`Network.py` 定义网络相关选项，topology 脚本负责把 controller 映射成 router/link 连接结构。

### `configs/example/gem5_library`

这里是更偏 gem5 standard library 风格的示例。它通常通过 board、processor、cache hierarchy、memory 等更高层对象搭系统，代码风格和传统 `configs/example/se.py` 不一样。

## 4. 配置和源码的关系

配置脚本里的对象大多来自 `m5.objects`，这些对象对应 `src` 下的 SimObject 声明。

例子：

- `BaseCPU` 来自 `src/cpu/BaseCPU.py`
- `System` 来自 `src/sim/System.py`
- `MemCtrl` 来自 `src/mem/MemCtrl.py`
- `RubySystem` 来自 `src/mem/ruby/system/RubySystem.py`
- `GarnetNetwork` 来自 `src/mem/ruby/network/garnet/GarnetNetwork.py`

Python 配置脚本设置的是参数和连接关系；真正执行时会创建对应 C++ 对象。

## 5. 一条常见 SE 配置主线

```text
configs/example/se.py
 -> configs/common/Options.py 解析参数
 -> configs/common/Simulation.py 选择 CPU 和 memory mode
 -> configs/common/CacheConfig.py 配 cache
 -> configs/common/MemConfig.py 配 memory
 -> 创建 workload / process
 -> m5.instantiate()
 -> m5.simulate()
```

如果加上 `--ruby`，路径会转向 `configs/ruby/Ruby.py` 和具体协议脚本。

## 6. 建议阅读顺序

1. `configs/example/se.py`
2. `configs/common/Options.py`
3. `configs/common/Simulation.py`
4. `configs/common/CacheConfig.py`
5. `configs/common/MemConfig.py`
6. `configs/ruby/Ruby.py`
7. `configs/network/Network.py`
8. `configs/topologies/Mesh_XY.py`

先理解一个能跑起来的配置脚本，再看它调用的 common/ruby/network/topology helper。

---

# 10. gem5 仿真核心总览

> 来源：`my_docs/sim/simulation_core_overview.md`

## 1. 对应代码目录

仿真核心相关代码主要在：

- `src/sim`
- `src/python/m5`
- `src/base`

这一层负责"对象怎样被创建、仿真时间怎样推进、事件怎样调度、状态怎样保存和恢复"。

## 2. 先说结论

gem5 是事件驱动仿真器。配置脚本先创建 SimObject 对象树，仿真核心再实例化对象、初始化状态，并通过 event queue 推进模拟时间。

可以先按这条线理解：

```text
Python config
 -> SimObject 参数树
 -> m5.instantiate()
 -> SimObject init / regStats / initState 或 loadState / startup
 -> m5.simulate()
 -> EventQueue 按 tick 处理事件
```

## 3. SimObject

关键文件：

- `src/sim/sim_object.hh`
- `src/sim/sim_object.cc`
- `src/python/m5/SimObject.py`
- `src/python/m5/simulate.py`

`SimObject` 是 gem5 里大多数模型对象的基础。CPU、cache、memory controller、System、clock domain 等都可以理解为 SimObject 或其派生对象。

从 `src/sim/sim_object.hh` 里的说明看，常见初始化顺序是：

```text
init()
 -> regStats()
 -> initState() 或 loadState()
 -> resetStats()
 -> startup()
 -> drainResume()
```

读代码时可以把 Python 侧看成"参数和对象树声明"，C++ 侧看成"运行时对象和行为"。

## 4. 事件队列

关键文件：

- `src/sim/eventq.hh`
- `src/sim/eventq.cc`
- `src/sim/simulate.hh`
- `src/sim/simulate.cc`
- `src/sim/cur_tick.hh`

gem5 的时间推进依赖 event queue。对象不是每个 host 时间片都运行，而是在未来某个 tick 调度 event。仿真主循环不断取出下一个 event，移动当前 tick，然后调用 event 的处理逻辑。

CPU tick、cache 延迟、memory controller 响应、统计/退出事件等都可以通过事件队列表达。

## 5. System 和 Root

关键文件：

- `src/sim/System.py`
- `src/sim/system.hh`
- `src/sim/system.cc`
- `src/sim/Root.py`
- `src/sim/root.hh`
- `src/sim/root.cc`

`Root` 是整个 SimObject 树的根。`System` 描述一个被模拟系统，包括 memory mode、memory ranges、cache line size、workload、共享 backstore、DVFS handler 等。

配置脚本通常会创建一个 `System`，再把 CPU、cache、memory、bus、Ruby 等对象挂到它下面。

## 6. Checkpoint、drain 和序列化

关键文件：

- `src/sim/serialize.hh`
- `src/sim/serialize.cc`
- `src/sim/drain.hh`
- `src/sim/drain.cc`
- `src/sim/sim_events.hh`

Checkpoint 依赖对象把状态序列化到文件，再在恢复时反序列化。Drain 用于让系统进入一个可以安全切换 CPU、保存 checkpoint 或退出的稳定状态。

如果研究 CPU switching、checkpoint restore、fast-forward，这部分会很重要。

## 7. Clock、power 和统计

关键文件：

- `src/sim/clocked_object.hh`
- `src/sim/ClockedObject.py`
- `src/sim/clock_domain.hh`
- `src/sim/ClockDomain.py`
- `src/sim/stat_control.hh`
- `src/sim/stats.hh`
- `src/sim/power`

CPU、cache、Ruby、memory controller 等很多对象都继承自 clocked object 或带有时钟域。统计系统负责注册、重置和输出模拟统计数据。

## 8. 和 configs / src 的关系

配置脚本创建对象树，例如：

```text
configs/example/se.py
 -> System()
 -> CPU()
 -> Cache()
 -> MemCtrl()
```

这些 Python 对象最终会对应到 `src` 下的 C++ SimObject。仿真核心负责把配置阶段的对象树变成运行时对象，并把它们放进事件驱动的执行环境。

## 9. 建议阅读顺序

1. `src/python/m5/SimObject.py`
2. `src/python/m5/simulate.py`
3. `src/sim/sim_object.hh`
4. `src/sim/System.py`
5. `src/sim/eventq.hh`
6. `src/sim/simulate.cc`
7. `src/sim/serialize.hh`
8. `src/sim/drain.hh`

先掌握 SimObject 和 event queue，再看 checkpoint、drain、stats、power 这些辅助机制。
