# gem5 仿真核心总览

## 1. 对应代码目录

仿真核心相关代码主要在：

- `src/sim`
- `src/python/m5`
- `src/base`

这一层负责“对象怎样被创建、仿真时间怎样推进、事件怎样调度、状态怎样保存和恢复”。

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

读代码时可以把 Python 侧看成“参数和对象树声明”，C++ 侧看成“运行时对象和行为”。

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
