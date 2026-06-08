# gem5 设备与 IO 模型总览

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

`BasicPioDevice` 的实现更简单：它把 `pio_addr` 和 `pio_size` 变成一个 `RangeSize`，所以这类设备本质上就是“固定地址窗口 + 寄存器读写函数”。

### 代表例子

- `src/dev/serial/uart.hh`
- `src/dev/serial/uart8250.cc`
- `src/dev/riscv/plic_device.cc`
- `src/dev/x86/i8254.cc`

以 UART 为例，设备内部维护一组寄存器，`read(pkt)` 和 `write(pkt)` 读写这些寄存器，并在需要时通过 platform 发中断。RISC-V 的 PLIC 设备同样是 `BasicPioDevice`，只是它的行为是中断控制器，不是串口。

## 5. DMA 设备是怎么实现的

DMA 设备的核心不在“被 CPU 读写”，而在“设备主动访问内存”。它的实现骨架是：

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

DMA 不是一次性“搬一块内存”这么简单。gem5 会把它拆成多个块来建模，因为这样才能和缓存层级、总线仲裁、响应延迟、排队冲突对上。

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
