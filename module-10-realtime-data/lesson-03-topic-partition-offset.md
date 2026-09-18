## Lesson Contract

所属 Module：Module 10 — 实时数据

本课核心问题：

> Kafka 里的 data 到底放在哪里？为什么一个 Topic 还要切成 Partition？Offset 到底表示什么？

学完以后，你应该能够解释：

- Topic 是什么，以及为什么它更像一个 logical stream / category，而不是一张数据库表。
- Partition 为什么存在，以及它和 parallelism、ordering、scalability 的关系。
- Offset 是什么，它为什么只在单个 Partition 内有意义。
- 为什么 Kafka 的 ordering guarantee 通常是 partition-level ordering，而不是 global ordering。
- 为什么 Consumer 的 progress 本质上要表示为“每个 Partition 读到哪个 Offset”。

本课不展开 Consumer Group rebalance、delivery semantics、exactly-once。这些会在后续课程继续。

---

## 一、先把 Kafka 想成一个 Log System，而不是 Queue

上一课我们已经建立了一个重要 mental model：

> Kafka is a distributed append-only log.

也就是说，Event 写进去以后，更接近：

```text
0 -> Event A
1 -> Event B
2 -> Event C
3 -> Event D
...
```

而不是传统 Queue 那种：

```text
A 被取走
↓
A 消失
```

但这里马上出现一个 engineering question：

> 如果所有 Event 都写进一条无限增长的 Log，会发生什么？

比如一个大型 Blockchain Data Platform 每秒可能产生大量：

- Transfer Event
- Swap Event
- NFT Event
- Lending Event
- Wallet Activity

如果所有东西都混在一起：

```text
Kafka
  ↓
[ Transfer, Swap, NFT, Transfer, Lending, Swap, ... ]
```

管理会非常困难。

于是 Kafka 首先需要一个 logical classification。

这就是 Topic。

---

## 二、Topic：一种 Logical Stream

可以先把 Topic 理解成：

> A named stream of related events.

例如：

```text
erc20_transfers
dex_swaps
nft_transfers
wallet_activity
```

它们代表不同 category of events。

例如：

```text
Indexer
   ↓
Kafka
   ├── erc20_transfers
   ├── dex_swaps
   └── nft_transfers
```

这里的 Topic 更像一个“逻辑数据流名称”。

它回答的是：

> What kind of events are these?

例如：

```text
Topic = erc20_transfers
```

意味着：

> 这条 logical stream 中保存的是 ERC-20 Transfer Event。

---

## 三、Topic 和数据库表不是一回事

这个地方容易产生类比误导。

你可以暂时类比：

```text
Kafka Topic
≈ Event Category

Database Table
≈ Queryable Stored Relation
```

但不能直接认为：

```text
Topic = Table
```

因为它们的 primary purpose 不同。

数据库 Table 主要为：

```text
SELECT
JOIN
GROUP BY
UPDATE
INDEX
```

服务。

Kafka Topic 主要为：

```text
Produce
Consume
Stream
Replay
```

服务。

【Data Engineering 视角】

数据库关心：

> What data do I have, and how do I query it?

Kafka 更关心：

> What events are arriving, and how do consumers process them continuously?

---

## 四、如果一个 Topic 只有一条 Log，会有什么问题？

假设：

```text
Topic: erc20_transfers
```

只有一条 Log：

```text
0
1
2
3
4
5
6
7
8
...
```

只有一个 Consumer 从头顺序处理。

那么：

```text
Producer
   ↓
Single Log
   ↓
Single Consumer
```

如果一秒进来 100,000 条 Event，而一个 Consumer 只能处理 10,000 条：

系统就会不断产生 lag。

你可能会说：

> 那我开 10 个 Consumer 不就行了吗？

问题是：

如果大家同时读同一条严格有序 Log：

```text
Event 0
Event 1
Event 2
Event 3
...
```

谁处理哪一条？

如果多个 Consumer 随便抢，会破坏 ordering。

Kafka 需要一种结构来同时解决：

- parallelism
- scalability
- ordering boundary

这就是 Partition。

---

## 五、Partition：Topic 的 Physical Shard

Topic 是 logical stream。

Partition 是：

> A physical shard of that stream.

例如：

```text
Topic: erc20_transfers
```

可以被拆成：

```text
Partition 0
Partition 1
Partition 2
Partition 3
```

于是结构变成：

```text
Topic: erc20_transfers

P0: 0 -> 1 -> 2 -> 3 -> ...
P1: 0 -> 1 -> 2 -> 3 -> ...
P2: 0 -> 1 -> 2 -> 3 -> ...
P3: 0 -> 1 -> 2 -> 3 -> ...
```

注意一个非常重要的点：

> Each partition has its own log.

也就是说，每个 Partition 都是一条独立的 append-only log。

---

## 六、为什么 Partition 能提升吞吐？

如果只有一个 Partition：

```text
P0
 ↓
Consumer A
```

处理能力有限。

如果有四个 Partition：

```text
P0 -> Consumer A
P1 -> Consumer B
P2 -> Consumer C
P3 -> Consumer D
```

现在可以 parallel processing。

所以：

> Partition is the basic unit of parallelism in Kafka.

这是一个非常重要的英文表达。

可以直接记：

```text
More Partitions
      ↓
More Parallelism
      ↓
Higher Throughput Potential
```

但不是 Partition 越多越好。

因为 Partition 越多，也意味着更多 metadata、更多 state、更多 coordination cost。

所以它是 scalability trade-off，而不是“越多越快”。

---

## 七、那么一条 Event 到底进入哪个 Partition？

现在有：

```text
erc20_transfers
├── P0
├── P1
├── P2
└── P3
```

Producer 产生一条：

```text
Transfer Event
from = Alice
to   = Bob
token = USDC
```

它必须被放进某一个 Partition。

Kafka 可以依据一个 key 来决定。

例如：

```text
key = wallet_address
```

或者：

```text
key = contract_address
```

然后通过 partitioning logic 把相同 key 的 Event 路由到同一个 Partition。

例如：

```text
hash(wallet_address) % partition_count
```

这是一个 simplified model。

核心不是公式本身，而是：

> Partition Key determines event locality.

---

## 八、为什么 Partition Key 很重要？

假设我们做 Wallet Balance Consumer。

Alice 连续发生三笔交易：

```text
Event A: Alice +100 USDC
Event B: Alice -20 USDC
Event C: Alice +50 USDC
```

如果这三条 Event 分到不同 Partition：

```text
P0 -> Event A
P1 -> Event B
P2 -> Event C
```

三个 Consumer 并行处理时，processing order 可能变成：

```text
B
C
A
```

那你就无法依赖原始顺序。

如果我们希望：

> All events for the same wallet stay ordered.

就可以选择：

```text
Partition Key = wallet_address
```

于是 Alice 的所有 Event 都进入同一个 Partition：

```text
P2:
A -> B -> C
```

这样 Kafka 能提供：

> ordering within a partition.

注意，不是全局有序。

---

## 九、Kafka 的 Ordering Guarantee 是局部的

这是本课最重要的概念之一。

Kafka 通常保证：

> Messages are ordered within a partition.

也就是：

```text
P0:
A -> B -> C
```

这个顺序可以保持。

但：

```text
P0: A -> C
P1: B -> D
```

Kafka 不保证：

```text
A < B < C < D
```

这样的 global ordering。

因为多个 Partition 本来就是为了 parallelism。

所以存在一个经典 trade-off：

```text
Global Ordering
      vs
Parallelism
```

如果你要求所有 Event 严格 global order，最简单的方法往往是：

```text
1 Partition
```

但吞吐就会受到限制。

如果你需要 high throughput：

```text
Many Partitions
```

就通常只能获得 partition-level ordering。

---

## 十、Blockchain 场景如何选择 Partition Key？

【Blockchain Data Engineer 视角】

Partition Key 必须由 downstream semantics 决定。

不是“随便选一个字段”。

例如 Wallet Balance：

```text
key = wallet_address
```

因为你希望同一个 wallet 的 balance updates 保持顺序。

例如 Token Analytics：

```text
key = token_contract
```

可能更合理。

例如一个按 Pool 维护实时状态的 DEX Consumer：

```text
key = pool_address
```

因为同一个 Pool 的 Swap / Liquidity events 需要局部有序。

所以 Partition Key 的问题，本质上是：

> Which entity requires ordered processing?

这不是 Kafka configuration question。

这是 data model + business semantics question。

---

## 十一、Offset：Partition 里的 Position

现在来看 Offset。

假设 Partition 0：

```text
P0

Offset 0 -> Event A
Offset 1 -> Event B
Offset 2 -> Event C
Offset 3 -> Event D
Offset 4 -> Event E
```

Offset 就是：

> The position of a record within a partition.

最核心的两个限定词：

```text
within a partition
```

也就是说：

```text
P0 Offset 5
```

和：

```text
P1 Offset 5
```

是两个完全不同的位置。

不能只说：

```text
Offset = 5
```

而不说它属于哪个 Partition。

---

## 十二、Offset 不是 Event 的 Global ID

这是很容易混淆的点。

不要把：

```text
Offset
```

理解成类似：

```text
tx_hash
log_index
primary_key
```

Offset 不是业务唯一标识。

它只是：

> position in a partition log.

例如：

```text
Topic = erc20_transfers
Partition = 3
Offset = 928381
```

表示：

> 这条 Record 位于 erc20_transfers Topic 的 Partition 3 的第 928381 位置附近。

它是 log position，不是 business identity。

这和你之前学过的：

```text
(chain_id, tx_hash, log_index)
```

完全不是同一层概念。

---

## 十三、Producer Offset 和 Consumer Progress 要分开

这里需要非常精确。

Kafka record 写入某个 Partition 后，会有对应 Offset。

例如：

```text
P0:
0 1 2 3 4 5 6 7 8
```

这是 Kafka Log 本身的位置。

Consumer 可能只处理到：

```text
5
```

所以：

```text
Log End Offset = 8
Consumer Position = 5
```

中间差：

```text
6, 7, 8
```

还没有处理。

这就是一种 Consumer Lag。

所以 Offset 本身是 Log position。

而：

> Consumer has processed up to offset 5.

才是 Consumer progress。

---

## 十四、这和你学过的 Checkpoint 有什么关系？

Module 9 你已经很熟悉：

```text
Checkpoint
=
已完成到哪里
```

Kafka 中可以做一个 mapping：

```text
ETL Checkpoint
≈ completed processing boundary

Kafka Consumer Offset
≈ consumed / committed position in a partition
```

两者非常相似。

但 Kafka 多了一个维度：

> Partition.

所以 Consumer progress 通常不是一个数字，而更像：

```text
P0 -> Offset 120
P1 -> Offset 98
P2 -> Offset 143
P3 -> Offset 101
```

这才是完整状态。

这和你之前理解多个 Pipeline 各自维护 Checkpoint 很像。

只是现在变成：

> each partition has its own progress position.

---

## 十五、为什么 Offset 能支持 Replay？

假设 Consumer 当前：

```text
P0 -> Offset 1000
```

后来发现：

```text
Offset 900 ~ 1000
```

处理逻辑有 Bug。

如果这些 Record 仍在 Retention 范围内，就可以把 Consumer position 调回：

```text
900
```

重新消费。

这就是：

> Replay from an earlier offset.

所以 Replay 的基础并不是神秘机制。

本质就是：

```text
Durable Log
+
Position
```

有数据还在。

也知道从哪里重新读。

就可以 Replay。

---

## 十六、一个完整例子：ERC-20 Transfer Topic

现在把三个概念连起来。

假设：

```text
Topic = erc20_transfers
```

4 个 Partition：

```text
P0
P1
P2
P3
```

Partition Key：

```text
wallet_address
```

Alice hash 到 P2。

Bob hash 到 P1。

那么：

```text
P1:
offset 0 -> Bob Event 1
offset 1 -> Bob Event 2
offset 2 -> Bob Event 3

P2:
offset 0 -> Alice Event 1
offset 1 -> Alice Event 2
offset 2 -> Alice Event 3
```

这样有两个好处：

第一：

Alice 的事件在 P2 内保持 local ordering。

第二：

Alice 和 Bob 可以 parallel processing。

这就是 Kafka 非常经典的设计思想：

> Preserve order where it matters; parallelize where it does not.

这句话建议直接记下来。

---

## 十七、银行系统类比

假设银行实时交易流：

```text
Topic = card_transactions
```

如果风险规则要求：

> 同一个 Account 的交易必须按顺序判断。

那么：

```text
Partition Key = account_id
```

于是：

```text
Account A -> P2
Account B -> P0
Account C -> P3
```

同一个 Account 内保持顺序。

不同 Account 可以并行。

这和 Blockchain Wallet 完全相同：

```text
account_id
≈
wallet_address
```

---

## 十八、Topic、Partition、Offset 三者关系

现在可以把三者压缩成一句话：

> Topic defines the logical stream, Partition splits the stream for scalability, Offset identifies a position inside one partition.

对应中文：

```text
Topic
= 这是什么数据流？

Partition
= 这条数据流被切成哪几个并行 shard？

Offset
= 我在某个 shard 里面的哪个位置？
```

再画一次：

```text
Topic: erc20_transfers

Partition 0
  Offset 0
  Offset 1
  Offset 2

Partition 1
  Offset 0
  Offset 1
  Offset 2

Partition 2
  Offset 0
  Offset 1
  Offset 2
```

---

## 十九、一个必须建立的工程心智模型

以后看到 Kafka，不要把它想成：

```text
一个大 Queue
```

更准确的是：

```text
Topic
  ↓
Partitions
  ↓
Append-only Logs
  ↓
Offsets
```

即：

```text
Logical Stream
      ↓
Physical Shards
      ↓
Ordered Records
      ↓
Position
```

这是 Kafka 后续所有概念的基础。

Consumer Group、parallel consumption、rebalance、delivery semantics 都建立在这个结构上。

---

## 二十、和前两课连接

到现在，Module 10 的逻辑已经形成：

```text
Lesson 1
Streaming = Unbounded Data Stream

        ↓

Lesson 2
Kafka = Durable Event Streaming Layer

        ↓

Lesson 3
Topic = Logical Stream
Partition = Parallel Shard
Offset = Position in Partition
```

下一课就可以继续回答：

> 多个 Consumer 到底怎么协作？

也就是：

```text
Consumer Group
+
Ordering
+
Parallel Consumption
```

---

## 本课核心结论

第一：

> A Topic is a logical stream of related events.

第二：

> A Partition is a physical shard and the basic unit of parallelism.

第三：

> Ordering is guaranteed within a partition, not across the entire topic.

第四：

> An Offset is a position within a partition, not a global event ID.

第五：

> Consumer progress must be tracked per partition.

第六：

> Partition Key should be chosen according to the entity whose event order must be preserved.

---

## 理解检查

### 问题一

假设：

```text
Topic = wallet_transfers
Partitions = 4
```

如果我们要求：

> 同一个 Wallet 的 Transfer 必须按顺序处理。

你会优先选择什么作为 Partition Key？

为什么？

### 问题二

假设：

```text
P0 -> Offset 120
P1 -> Offset 95
P2 -> Offset 143
```

Consumer 说：

> “我的 Offset 是 120。”

这句话为什么不够准确？

### 问题三

为什么 Kafka 不默认保证整个 Topic 的 global ordering？

请从：

```text
Ordering
vs
Parallelism
```

这个 trade-off 来解释。