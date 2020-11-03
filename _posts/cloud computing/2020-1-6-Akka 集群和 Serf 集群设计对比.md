# Akka 集群和 Serf 集群设计对比及 CDCF 集群设计思考

[Akka](https://akka.io) 是 Java 分布式 Actor 模型实现，Akka 集群是其中的一个模块。

[Serf](https://www.serf.io) 是一个独立的集群编排工具，其他程序通过 RPC 的方式向其调用集群功能。

Akka 集群和 Serf 集群设计的核心是对集群的 Availability 、Consistency 和 Latency 在不同情况下根据需求所作的取舍。

## PACELC 定理

[PACELC 定理](https://en.wikipedia.org/wiki/PACELC_theorem#cite_note-ctmddsd-3)是针对分布式系统的一个理论：

> 在分布式系统中，当发生 Partition 时，系统必须在 Availability 和 Consistency 之间选择其一，当没有发生 Partition 正常运行时，系统必须在 Latency 和 Consistency 之间选择其一。

> It states that in case of network partitioning (P) in a distributed computer system, one has to choose between availability (A) and consistency (C) (as per the CAP theorem), but else (E), even when the system is running normally in the absence of partitions, one has to choose between latency (L) and consistency (C).

可以这样直觉性地简单理解这个定理：

想象一个由两个节点组成的分布式系统，当两个节点之间发生 Partition 时，

如果要保持每个节点的 Availability，即两个节点同时可读取，那么由于 Partition 的存在，节点之间数据的 Consistency 一定会被破坏；

如果要保持节点间用户读取到的数据的 Consistency，则至少有一个节点必须停止读写，破坏了 Availability。

当系统没有发生 Partition 时，Availablility 和 Consistency 可以同时维持，但分布式系统的 Availability 同时隐含了数据必须在系统中得到复制的先决条件，

当维持 Consistency 的情况下进行数据复制，则系统必须在数据复制到一定数量的节点后才响应，牺牲了系统的 Latency；

当系统以低 Latency 响应时，则系统中多份复制数据的 Consistency 得不到保证。

## Partition 未发生时 Akka 和 Serf 的设计对比

### Akka 和 Serf 的设计的相同之处

1. 根据 PACELC 定理，Akka 和 Serf  在未 Partition 时均选择了倾向于 Latency 而部分牺牲了 Consistency，它们都采用了 Eventual Consistency 模型，即数据写入时，只需写入集群中一个节点即可，集群之后再将数据复制到其他节点，在数据复制的过程中，节点之间的数据会不一致。
2. Akka 和 Serf 在集群中维护的数据都是集群的节点信息及其运行状态，每个节点都有一份集群中所有节点的列表和其运行状态信息，这份数据需要最终达到一致。
3. Akka 和 Serf 在数据复制时，均选择了 Gossip 协议的数据分发模型。

```
Gossip 过程是由种子节点发起，当一个种子节点有状态需要更新到网络中的其他节点时，它会随机的选择周围几个节点散播消息，收到消息的节点也会重复该过程，直至最终网络中所有的节点都收到了消息。
```

### Akka 和 Serf 的设计的不同之处

| 类别 | Akka | Serf |
| ------ | ------ | ------ |
| Gossip 消息内容 | 整个集群所有节点的地址和状态 | 集群中单个节点的地址和状态 |
| 区分 Gossip 消息的工具 | Vector Clock                 | Incarnation Number |

Incarnation Number 其实就是一个从 0 开始每次加 1 的整型，使用它只能标识单个节点不同消息的先后顺序。

Vector Clock 则是一个复杂得多的数据结构，使用它可以标识整个集群不同节点间消息的先后顺序。

节点收到消息后，会将消息中的信息与本地的节点信息进行合并：

Serf 消息中只含有单个节点的信息，因为 Gossip 消息不保证到达顺序，因此合并时可能有歧义的地方只会是该节点消息的先后顺序，Incarnation Number 被添加到每个消息中以达到消歧义的目的。

Akka 消息含有所有节点的信息，消息不属于任意一个节点，因此需要使用 Vector Clock 才能消歧义。

## Partition 发生时 Akka 和 Serf 的设计对比

根据 PACELC 定理，Akka 和 Serf 在 Partition 发生时采取了不同的取舍。

### Akka 选择 Consistency，放弃 Availability

Akka 设计了节点选举机制，使得系统在 Partition 发生时，每个 Partition 不能再加入退出节点，不同 Partition 节点查询到的节点列表维持不变，但会有不同的节点状态。

```
Akka 选举机制

每个新节点向集群请求加入（或离开）时，不会立即进入运行（或退出）状态，要等加入的消息传递到所有节点后，选出一个 Leader 节点，由 Leader 节点决定新节点由加入转变为运行状态，因此只要集群发生 Partition（包括任意节点宕机的情况），则集群不再能够加入或退出节点。
```

由于 Leader 的选出需要所有节点参与，该机制依赖于 Gossip 消息中的所有节点信息和 Vector Clock。

### Serf 选择 Availability，放弃 Consistency

系统发生 Partition 后，每个 Partition 可以分别正常加入退出节点，因此从不同 Partition 查到的节点信息可能会有较大不同。

## Partition 检测

分布式系统在 Partition 前后有各种取舍，而 Partition 状态发生本身，也需要检测方法。

检测方法在选定方案后再研究。

## CDCF 集群设计思考

可以观察到，除了因 Gossip 机制不同而带来的 Eventual Consistency 性能不同外，Akka 和 Serf 的外在表现最大的不同就是在 Partition 发生时，Akka 选择 Consistency 而 Serf 选择 Availability。

从用户需求看，CDCF 主要会用到集群功能的是 Cluster-Aware Router，Router 在 Partition 发生时只关心自己所在 Partition 的节点情况，对于跨 Partition 的 Consistency 并不关心。

再考虑到 Akka 的架构远比 Serf 复杂，而每个方案的整体性很高，很难选择其中一部分加以组合，因此建议集群设计采用 Serf 的方案。

## 参考资料

1. Akka 集群文档：https://doc.akka.io/docs/akka/current/typed/index-cluster.html
2. Serf 集群文档：https://www.serf.io/docs/internals/gossip.html
3. Serf 集群论文：https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf
4. PACELC 定理介绍：http://dbmsmusings.blogspot.com/2010/04/problems-with-cap-and-yahoos-little.html