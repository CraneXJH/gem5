# gem5 代码分类文档索引

这个目录按 gem5 源码的大类来组织，不追求和代码目录一一对应到最细层级。目标是先给读代码的人建立入口，再从入口跳到更具体的实现文档。

## 当前分类

### `arch/riscv`

对应代码目录主要是 `src/arch/riscv`。

- [RISC-V 架构实现总览](arch/riscv/riscv_arch_overview.md)
- [RISC-V 内存模型入门](arch/riscv/riscv_memory_model_impl.md)

这一类文档关注 ISA、指令解码、CSR、异常、中断、TLB、PMP/PMA，以及 RISC-V 指令如何变成 CPU 可执行的 `StaticInst`。

### `cpu`

对应代码目录主要是 `src/cpu`。

- [CPU 模型总览](cpu/cpu_models_overview.md)

这一类文档关注 `BaseCPU`、Simple CPU、O3 CPU、Minor CPU、KVM CPU、线程上下文、执行上下文、流水线和访存端口。

### `mem`

对应代码目录主要是 `src/mem`，其中 Ruby 相关内容放在 `mem/ruby` 下。

- [内存系统总览](mem/memory_system_overview.md)
- [CPU 到 L3 / 内存的简单流程](mem/ruby/gem5_cpu_l3_controller_flow.md)
- [NoC 入门总览](mem/ruby/gem5_noc_overview.md)

这一类文档关注经典 cache、Ruby、一致性协议、NoC、memory controller、DRAM/NVM 接口、packet/request/port。

### `dev`

对应代码目录主要是 `src/dev`。

- [设备与 IO 模型总览](dev/device_io_dma_overview.md)

这一类文档关注 PIO/MMIO 设备、DMA 设备、PCI 设备、串口、网卡、磁盘、平台中断、设备树和 board/platform 侧的设备挂接。

### `configs`

对应代码目录主要是 `configs`。

- [配置脚本总览](configs/configs_overview.md)

这一类文档关注 Python 配置脚本如何创建 `System`、CPU、cache、Ruby、内存控制器、拓扑和 benchmark/workload。

### `sim`

对应代码目录主要是 `src/sim`，也会涉及 `src/python/m5`。

- [仿真核心总览](sim/simulation_core_overview.md)

这一类文档关注 `SimObject`、事件队列、仿真生命周期、checkpoint、drain、statistics、clock/power domain。

## 建议阅读顺序

如果是从一次普通 gem5 仿真开始理解，可以按这个顺序看：

1. `configs`：先看脚本怎样搭系统。
2. `sim`：理解对象树、初始化和事件驱动。
3. `cpu`：看 CPU 如何执行指令和发起访存。
4. `arch/riscv`：看 RISC-V 指令语义和地址转换。
5. `mem`：看请求进入 cache、Ruby、NoC 和 memory controller 后怎么走。
6. `dev`：看外设如何通过 MMIO、DMA、PCI、中断和平台代码接入系统。

如果只关心 Ruby / NoC，可以直接从 `mem/memory_system_overview.md` 和 `mem/ruby` 下两篇开始。
