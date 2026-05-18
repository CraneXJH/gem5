# gem5 CPU 到 L3 / 内存的简单流程

## 1. 先说结论

CPU 不会直接把请求发给 NoC，也不会直接和 L3 “裸连接”。真正负责协议交互的是 controller。

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

可以先把它理解成“协议状态机”：

- 它接收请求
- 它判断命中、miss、下一步动作
- 它决定是否要发消息到 NoC

### NoC

负责把 controller 之间的消息搬运过去。

## 4. 最容易记住的理解方式

如果你只想记一句话，就记这个：

`CPU 通过 controller 和 NoC 间接访问更远的缓存和内存`

这比把它想成 “CPU 直接找 L3” 更接近 gem5 里的真实结构。

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

入门阶段先知道“消息要经过网络接口和 router”就够了，不用一开始就把 flit 流水线全看完。

## 8. 如果你想顺着看代码

- `src/mem/ruby/system/Sequencer.cc`
- `src/mem/ruby/slicc_interface/AbstractController.hh`
- `src/mem/ruby/network/garnet/NetworkInterface.cc`
- `src/mem/ruby/network/garnet/Router.cc`

## 9. 一句话总结

这条链里最关键的不是 “CPU 直接连到 L3”，而是 “CPU 先进入 Ruby，再由 controller 和 NoC 把请求送到更远处”。
