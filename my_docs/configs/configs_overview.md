# gem5 配置脚本总览

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

这一层负责“创建什么系统、用什么 CPU、什么 cache / Ruby / memory、跑什么 workload”。

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

配置脚本决定“模型长什么样”，`src` 下的 C++/Python SimObject 代码决定“模型运行时怎么行为”。

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
