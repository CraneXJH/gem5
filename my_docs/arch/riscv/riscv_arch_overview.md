# RISC-V 架构实现总览

## 1. 对应代码目录

RISC-V 架构相关代码主要在：

- `src/arch/riscv`
- `src/arch/riscv/isa`
- `src/arch/riscv/isa/formats`
- `src/arch/riscv/insts`
- `src/arch/riscv/regs`

这一层负责“RISC-V 指令和架构状态是什么”，不是负责 cache、内存控制器或 NoC 的实现。

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

关键点是：RISC-V 目录描述“指令该做什么”，CPU 目录描述“指令什么时候、以什么方式执行”。

## 5. 和内存相关的边界

RISC-V 的 load/store、fence、LR/SC、AMO 在 ISA 层有自己的语义入口，但完整内存顺序行为不是只在 `src/arch/riscv` 一个目录里实现的。

- 指令语义和访问标记在 `src/arch/riscv/isa/formats`。
- 执行、乱序、LSQ、提交等在 `src/cpu`。
- cache、一致性、NoC、memory controller 在 `src/mem`。

更具体的内存模型说明见 [RISC-V 内存模型入门](riscv_memory_model_impl.md)。

## 6. 建议阅读顺序

1. `src/arch/riscv/RiscvISA.py`
2. `src/arch/riscv/isa/decoder.isa`
3. `src/arch/riscv/isa/formats/mem.isa`
4. `src/arch/riscv/isa/formats/amo.isa`
5. `src/arch/riscv/tlb.hh`
6. `src/arch/riscv/isa.cc`

先把“指令怎样被识别、怎样生成执行代码、访存怎样进入地址转换”看明白，再深入 CSR、异常、中断和特权模式细节。
