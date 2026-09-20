## Lesson Contract

所属 Module：Module 10 — 实时数据

本课核心问题：

> 一个 Topic 有多个 Partition、系统也有多个 Consumer 时，Kafka 怎么决定“谁处理哪个 Partition”？为什么这样既能 parallel processing，又不会破坏 Partition 内的 ordering？

学完以后，你应该能够解释：

- Consumer Group 为什么存在。
- 同一个 Group 内多个 Consumer 如何分工处理 Partitions。
- 为什么同一个 Partition 在同一个 Consumer Group 内，同一时刻只会分配给一个 active Consumer。
- 为什么 Consumer 数量超过 Partition 数量后，不会继续增加有效并行度。
- 为什么不同 Consumer Group 可以独立消费同一个 Topic。
- Consumer Group 如何把 load balancing 与 partition-level ordering 结合起来。
- Rebalance 是什么，以及为什么 Consumer membership 变化时需要重新分配 Partition。

本课不深入 Offset Commit 时机、At-most-once / At-least-once / Exactly-once；这些留到第 5 课。

---

## 一、先从上一课留下的问题开始

上一课我们已经知道：

~~~text
Topic = Logical Stream
Partition = Physical Shard
Offset = Position in Partition
~~~

例如：

~~~text
Topic: wallet_transfers

P0
P1
P2
P3
~~~

现在假设我们启动两个 Consumer：

~~~text
Consumer A
Consumer B
~~~

马上出现一个问题：

> Who consumes which partition?

如果两个 Consumer 都随便读取所有 Partition：

~~~text
Consumer A -> P0 P1 P2 P3
Consumer B -> P0 P1 P2 P3
~~~

那很可能出现：

- duplicate processing
- coordination problems
- ordering confusion

所以 Kafka 需要一个“消费者协作单位”。

这就是：

> Consumer Group.

---

## 二、Consumer Group：一组协作消费的 Consumer

可以先用一句英文定义：

> A Consumer Group is a set of consumers that cooperate to consume a topic.

例如：

~~~text
Consumer Group: wallet-balance-service

Consumer A
Consumer B
~~~

Topic 有 4 个 Partition：

~~~text
P0
P1
P2
P3
~~~

Kafka 可以分配：

~~~text
Consumer A -> P0, P1
Consumer B -> P2, P3
~~~

于是两个 Consumer 并行处理不同 Partition。

这就是：

> Work sharing within a group.

或者说：

> load balancing across partitions.

---

## 三、最重要的规则：一个 Partition 在一个 Group 内只能有一个 Owner

这是本课最重要的一条规则：

> Within the same consumer group, a partition is assigned to at most one active consumer at a time.

例如：

~~~text
Group G1

Consumer A -> P0, P1
Consumer B -> P2, P3
~~~

不会正常出现：

~~~text
Consumer A -> P0
Consumer B -> P0
~~~

两个人同时处理同一个 Partition。

为什么？

因为上一课已经知道：

> Ordering is guaranteed within a partition.

如果同一个 Partition 同时被两个 Consumer 随便抢：

~~~text
P0:
Event 0
Event 1
Event 2
Event 3
~~~

可能变成：

~~~text
Consumer A -> Event 0
Consumer B -> Event 1
Consumer A -> Event 2
Consumer B -> Event 3
~~~

如果处理完成时间不同：

~~~text
0
2
1
3
~~~

就会破坏这个 Partition 的 processing order。

所以 Kafka 的基本设计是：

~~~text
One Partition
      ↓
One Active Consumer
within a Consumer Group
~~~

这样 partition-level ordering 才有清晰的 ownership boundary。

---

## 四、Consumer Group 如何获得 Parallelism？

假设：

~~~text
Topic = wallet_transfers
Partitions = 4
~~~

只有一个 Consumer：

~~~text
Consumer A
  ├─ P0
  ├─ P1
  ├─ P2
  └─ P3
~~~

A 一个人处理全部数据。

如果增加到两个：

~~~text
Consumer A -> P0, P1
Consumer B -> P2, P3
~~~

并行度增加。

四个 Consumer：

~~~text
Consumer A -> P0
Consumer B -> P1
Consumer C -> P2
Consumer D -> P3
~~~

可以达到这个 Topic 在该 Group 下的 maximum partition-level parallelism。

所以有一个很重要的关系：

> Effective parallelism in a consumer group is bounded by the number of partitions.

简化理解：

~~~text
Maximum useful consumers
≈
Number of partitions
~~~

---

## 五、如果 Consumer 比 Partition 多，会发生什么？

假设：

~~~text
Partitions = 4
Consumers = 6
~~~

可能是：

~~~text
Consumer A -> P0
Consumer B -> P1
Consumer C -> P2
Consumer D -> P3
Consumer E -> idle
Consumer F -> idle
~~~

E 和 F 没有 Partition 可分。

所以：

> More consumers do not always mean more throughput.

如果 Partition 数量不增加，继续增加 Consumer 可能没有意义。

这也是为什么上一课说：

> Partition is the basic unit of parallelism.

---

## 六、如果 Consumer 比 Partition 少呢？

假设：

~~~text
Partitions = 4
Consumers = 2
~~~

一个 Consumer 可以负责多个 Partition：

~~~text
Consumer A -> P0, P1
Consumer B -> P2, P3
~~~

所以关系并不是：

~~~text
1 Consumer = 1 Partition
~~~

而是：

> One consumer can own multiple partitions, but one partition cannot be owned by multiple consumers in the same group at the same time.

这句话非常关键。

---

## 七、同一个 Topic 能不能被两个业务系统同时完整消费？

可以。

这里要区分：

> Consumers in the same group

和：

> Consumers in different groups.

假设：

~~~text
Topic = wallet_transfers
~~~

有两个业务：

~~~text
Wallet Balance Service
Risk Alert Service
~~~

如果它们需要各自看到全部 Transfer Event，就不能简单把它们放在同一个 Consumer Group 里。

为什么？

因为同一个 Group 是：

> share the work.

例如：

~~~text
Group G1

Wallet Consumer -> P0, P1
Risk Consumer   -> P2, P3
~~~

这样 Wallet Service 看不到 P2、P3，Risk Service 也看不到 P0、P1。

这显然不符合需求。

---

## 八、不同 Consumer Group：每个 Group 独立消费完整 Topic

更合理的是：

~~~text
Topic: wallet_transfers
       │
       ├── Group: wallet-balance
       │      ├── Consumer A
       │      └── Consumer B
       │
       └── Group: risk-alert
              ├── Consumer C
              └── Consumer D
~~~

这两个 Group 都可以独立消费整个 Topic。

可以这样理解：

~~~text
Same Group
= divide the work

Different Groups
= each group gets its own logical consumption
~~~

这是 Kafka Fan-out 的关键机制之一。

---

## 九、Group 的 Offset 也是独立的

上一课你已经知道：

~~~text
Offset belongs to a Partition
~~~

但 Consumer Progress 是：

~~~text
某个 Consumer / Consumer Group
对某个 Partition
处理到哪个 Offset
~~~

现在就可以更精确了。

例如：

~~~text
Topic: wallet_transfers

Group: wallet-balance
P0 -> Offset 120
P1 -> Offset 98

Group: risk-alert
P0 -> Offset 115
P1 -> Offset 96
~~~

两组消费进度不同。

这完全正常。

因为：

> Consumer offsets are tracked independently per consumer group.

所以 Risk Service 暂时慢一点，不会阻塞 Wallet Balance Service。

这正是 decoupling 的进一步体现。

---

## 十、把 Group 和上一课的 Offset 连起来

完整 key 可以先形成这样的 mental model：

~~~text
Consumer Progress
≈
(Group, Topic, Partition) -> Offset
~~~

例如：

~~~text
(wallet-balance, wallet_transfers, P0) -> 120
(risk-alert, wallet_transfers, P0)     -> 115
~~~

所以不能只说：

~~~text
Offset = 120
~~~

甚至也不能只说：

~~~text
P0 Offset = 120
~~~

如果是在讨论 Consumer Progress，还要知道：

> Which consumer group?

这一步会为下一课 Offset Commit 做准备。

---

## 十一、Rebalance：Consumer 变化以后，Partition 要重新分配

假设最初：

~~~text
P0 -> Consumer A
P1 -> Consumer A
P2 -> Consumer B
P3 -> Consumer B
~~~

后来 Consumer C 加入这个 Group。

Kafka 需要重新安排：

~~~text
P0 -> A
P1 -> B
P2 -> C
P3 -> A
~~~

这种重新分配过程叫：

> Rebalance.

再比如 Consumer B crash：

~~~text
Before:

A -> P0, P1
B -> P2, P3
~~~

B 挂掉以后，P2、P3 不能没人处理。

于是 Group 重新分配：

~~~text
After Rebalance:

A -> P0, P1, P2, P3
~~~

如果之后 B 恢复：

~~~text
A -> P0, P1
B -> P2, P3
~~~

可能再次发生 Rebalance。

---

## 十二、Rebalance 为什么需要 Offset？

这个问题很重要。

假设：

~~~text
Consumer B
P2 -> processed to Offset 500
~~~

然后 B crash。

Rebalance 后：

~~~text
Consumer A 接管 P2
~~~

A 必须知道：

> Where should I resume?

答案就是依赖这个 Consumer Group 对 P2 保存的消费进度。

例如：

~~~text
Group G1
P2 -> committed offset 500
~~~

A 就可以从相应位置继续。

所以：

~~~text
Partition Ownership
+
Consumer Group Offset
~~~

共同构成 failure recovery 的基础。

但注意：

> “Offset 到底什么时候 Commit？”

这是下一课的核心，因为 commit timing 会直接决定 duplicate 或 data loss 的风险。

---

## 十三、Ordering 为什么没有被 Consumer Group 破坏？

现在把逻辑串起来。

Producer 通过 Partition Key：

~~~text
wallet_address
~~~

保证 Alice 的事件都进入 P2：

~~~text
P2:
A1 -> A2 -> A3 -> A4
~~~

Consumer Group 又保证：

> P2 at a given time has one active owner within the group.

例如：

~~~text
Consumer B -> P2
~~~

所以 Consumer B 可以按：

~~~text
A1
A2
A3
A4
~~~

顺序消费。

这就是：

> Partition Key determines locality; Consumer Group preserves single ownership of each partition.

两者配合以后：

~~~text
Same Wallet
   ↓
Same Partition
   ↓
One Consumer Owner
   ↓
Ordered Processing
~~~

---

## 十四、但要注意一个工程细节：Kafka 保证的是 Record Delivery Order

这里稍微做一个 precision correction。

Kafka 把同一 Partition 的 Records 按顺序交给 Consumer。

但如果你的 Consumer 内部又自己开线程：

~~~text
Record 1 -> Worker A
Record 2 -> Worker B
Record 3 -> Worker C
~~~

然后：

~~~text
Worker B finishes first
Worker C finishes second
Worker A finishes last
~~~

业务处理结果仍然可能乱序。

所以更精确地说：

> Kafka preserves record order within a partition, but your application must also preserve processing order if the business requires it.

Kafka 不能替你的业务代码保证所有 downstream side effects 都按顺序完成。

这个区别在 Wallet Balance 这类 stateful processing 中尤其重要。

---

## 十五、Blockchain Example：Wallet Balance Pipeline

假设：

~~~text
Topic = wallet_transfers
Partition Key = wallet_address
Partitions = 4
~~~

Wallet Balance Service：

~~~text
Consumer Group = wallet-balance
~~~

两个 Consumer：

~~~text
WB-1 -> P0, P1
WB-2 -> P2, P3
~~~

Alice hash 到 P2：

~~~text
Alice Event 1
Alice Event 2
Alice Event 3
~~~

都进入 P2。

P2 由 WB-2 独占消费。

于是：

~~~text
Alice Event 1
      ↓
Alice Event 2
      ↓
Alice Event 3
~~~

可以保持顺序更新 Alice Balance。

与此同时 Bob 在 P1：

~~~text
Bob Events
~~~

由 WB-1 并行处理。

所以系统实现：

> ordered per wallet, parallel across wallets.

这就是非常典型的 Kafka design pattern。

---

## 十六、同一个 Topic 再给 Risk System

Risk System 也需要全部 Transfer Event。

于是建立另一个 Group：

~~~text
Consumer Group = risk-alert
~~~

例如：

~~~text
Topic: wallet_transfers

Group wallet-balance:
  WB-1 -> P0,P1
  WB-2 -> P2,P3

Group risk-alert:
  Risk-1 -> P0,P2
  Risk-2 -> P1,P3
~~~

两个 Group 的 Partition assignment 可以不同。

它们的 Offset 也可以不同。

但是都能覆盖整个 Topic。

所以：

> Partition assignment is scoped to a consumer group.

---

## 十七、银行系统类比

假设：

~~~text
Topic = card_transactions
Partition Key = account_id
~~~

两个业务系统：

~~~text
Fraud Detection
Customer Notification
~~~

应该是：

~~~text
Group: fraud-detection
  Consumer F1
  Consumer F2

Group: notification
  Consumer N1
  Consumer N2
~~~

而不是把 Fraud 和 Notification 放进同一个 Group。

原因非常简单：

同组意味着：

> They split the work.

不同组意味着：

> Each service independently consumes the stream.

---

## 十八、Consumer Group 最核心的三个价值

可以压缩成三个关键词：

### 1. Parallelism

多个 Consumer 可以并行处理不同 Partition。

### 2. Load Balancing

Partitions 在 Group 内分配给不同 Consumer。

### 3. Failure Recovery

Consumer crash 后，通过 Rebalance 把 Partition 交给其他 Consumer。

这三件事共同解决：

> How can multiple consumers cooperatively process one event stream?

---

## 十九、Partition 数量其实定义了 Scaling Ceiling

现在可以看到一个很重要的系统设计结论。

假设：

~~~text
Partitions = 8
~~~

那么一个 Consumer Group 最多大约有：

~~~text
8 active partition consumers
~~~

再增加：

~~~text
Consumer 9
Consumer 10
~~~

可能处于 idle。

所以 Partition 数量不仅影响 storage layout，也影响：

> horizontal scaling ceiling.

这就是为什么 Partition Count 是一个 architecture decision。

它不是单纯配置参数。

---

## 二十、一个容易混淆的问题：一个 Consumer 能处理多个 Partition，那顺序怎么办？

假设：

~~~text
Consumer A -> P0, P1
~~~

Kafka 保证：

~~~text
P0 内部有序
P1 内部有序
~~~

但不会保证：

~~~text
P0 Event 1
一定在
P1 Event 1
之前
~~~

所以：

> Ordering scope remains the partition.

哪怕同一个 Consumer 同时拥有多个 Partition，也不存在跨 Partition 的 global ordering guarantee。

这和上一课完全一致。

---

## 二十一、把 Topic / Partition / Consumer Group / Offset 串起来

现在 Kafka 的基础结构已经完整：

~~~text
Topic
  ↓
Partitions
  ↓
Consumer Group
  ↓
Partition Assignment
  ↓
Consumer
  ↓
Per-Partition Offset
~~~

可以翻译成五个问题：

~~~text
Topic
What stream is this?

Partition
How is the stream sharded?

Consumer Group
Which consumers cooperate?

Assignment
Who owns which partition?

Offset
How far has this group progressed?
~~~

这是 Kafka 的核心 operational model。

---

## 二十二、和前三课连接

~~~text
Lesson 1
Streaming
= Unbounded Stream

Lesson 2
Kafka
= Durable Event Streaming Layer

Lesson 3
Topic / Partition / Offset
= Stream Structure and Position

Lesson 4
Consumer Group
= Cooperative Parallel Consumption
~~~

下一课继续：

~~~text
Delivery Semantics
At-most-once
At-least-once
Exactly-once
~~~

它要回答的是：

> Consumer 处理完数据和 Commit Offset 的先后顺序不同，会发生什么？

这会把 Kafka 和 Module 9 的 Idempotency 正式连接起来。

---

## 本课核心结论

第一：

> Consumers in the same group share the partitions.

第二：

> Consumers in different groups independently consume the same topic.

第三：

> Within one consumer group, one partition has at most one active consumer owner at a time.

第四：

> Effective consumer parallelism is bounded by the number of partitions.

第五：

> Consumer progress is tracked per group, per topic, per partition.

第六：

> Rebalance redistributes partition ownership when consumer membership changes.

第七：

> Kafka preserves partition-level record ordering; the application must still preserve processing order if business semantics require it.

---

## 理解检查

### 问题一

假设：

~~~text
Topic = wallet_transfers
Partitions = 4

Consumer Group = wallet-balance
Consumers = 6
~~~

最多有几个 Consumer 能真正同时承担 Partition processing？

为什么另外的 Consumer 可能 idle？

### 问题二

Wallet Balance Service 和 Risk Alert Service 都需要完整读取 wallet_transfers Topic。

它们应该：

~~~text
A. 放进同一个 Consumer Group
~~~

还是：

~~~text
B. 使用两个不同 Consumer Group
~~~

为什么？

### 问题三

假设：

~~~text
Consumer B owns P2
and has processed P2 to Offset 500
~~~

Consumer B 突然 crash。

Rebalance 后 Consumer A 接管 P2。

为了让 A 正确继续处理，系统最关键需要知道什么信息？

请联系上一课 Offset 的概念回答。

## 用户回答

问题一：最多有 4 个 consumer 能真正同时承担 partition processing。另外的两个 consumer 可能 idle，因为在同一个 consumer group 内，一个 partition 不能同时分配给两个 consumer。
问题二：他们应该使用两个不同的 consumer group。
因为在同一个 consumer group 里面，消费者不能同时拥有同一个 partition；而在不同的 consumer group 里面，他们可以同时读取所有完整的 topic。 问题三，为了让 A 正确地继续处理，系统最关键需要知道 offset 信息。那这里的 offset 信息特指的是 consumer group， topic， partition这三个组成的 key 所对应的 offset

## 老师判断与校准

三题全部通过。

- 问题一：判断正确。4 个 Partition 决定了这个 Consumer Group 最多只有 4 个 active partition owners；第 5、6 个 Consumer 没有 Partition 可分，因此会 idle。这里体现的是：Effective consumer parallelism is bounded by the number of partitions。
- 问题二：判断正确。Wallet Balance Service 和 Risk Alert Service 需要各自完整读取整个 Topic，因此应使用两个不同的 Consumer Group。Same Group means work sharing；Different Groups means independent consumption of the same topic。
- 问题三：判断正确，而且表达得很精确。这里需要恢复的不是一个脱离上下文的 Offset 数字，而是这个 Consumer Group 在该 Topic 的该 Partition 上的消费进度。可以抽象为：(consumer_group, topic, partition) -> offset。Rebalance 后新的 Consumer 根据这一进度继续处理。

## 结课判定

Module 10 第 4 课理解检查通过，正式完成。已掌握 Consumer Group 的核心模型：同组 Consumer 通过 Partition Assignment 分担工作；不同 Group 可以独立消费同一 Topic；一个 Partition 在同一 Group 内同一时刻最多有一个 active owner；Consumer Progress 需要按 Group / Topic / Partition 维护；Rebalance 通过重新分配 Partition 支持扩缩容和故障恢复。
