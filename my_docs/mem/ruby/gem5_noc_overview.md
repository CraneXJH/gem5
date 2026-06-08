# gem5 NoC 入门总览

代码分类：`src/mem/ruby/network` 为主，配置侧关联 `configs/network` 和 `configs/topologies`。

## 1. 先说结论

gem5 里的 NoC 主要出现在 Ruby 这条内存系统路径上。最简单的理解是：

- 一致性协议决定“发什么消息”
- NoC 决定“这些消息怎么在片上网络里走”

所以 NoC 更像运输系统，不是协议本身。

## 2. 先认识两个关键词

### `simple`

这是更抽象的网络模型，适合先入门。它更强调“消息能不能送达”，而不是每个微结构细节。

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

NoC 不负责决定一致性行为。它不回答“该不该发这个请求”，只回答“这个请求怎么送过去”。

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
