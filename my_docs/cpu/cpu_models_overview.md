# gem5 CPU 模型总览

## 1. 对应代码目录

CPU 相关代码主要在：

- `src/cpu`
- `src/cpu/simple`
- `src/cpu/o3`
- `src/cpu/minor`
- `src/cpu/kvm`
- `src/cpu/checker`
- `src/cpu/pred`

这一层负责“CPU 怎样执行指令、怎样推进时间、怎样向内存系统发请求”。

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

如果使用 Ruby，CPU 请求通常会先进入 `RubySequencer` / `RubyPort`，然后交给 Ruby controller 和 NoC。相关说明见 `my_docs/mem/ruby`。

## 8. 建议阅读顺序

1. `src/cpu/BaseCPU.py`
2. `src/cpu/base.hh`
3. `src/cpu/exec_context.hh`
4. `src/cpu/simple/base.hh`
5. `src/cpu/simple/timing.cc`
6. `src/cpu/o3/cpu.hh`
7. `src/cpu/o3/lsq.hh`
8. `src/cpu/o3/commit.hh`

先用 Simple CPU 建立“指令怎样跑起来”的直觉，再看 O3 的乱序结构。
