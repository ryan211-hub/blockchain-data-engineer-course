## Lesson Contract

所属 Module：Module 10 — 实时数据。

本课核心问题：

> 如果已经有 Ethereum Node / Provider、WebSocket、Indexer 和 Database，为什么实时 Pipeline 中间还需要 Kafka？

学完以后，你应该能够解释：

- Kafka 在实时数据平台中到底解决什么问题，而不是只会背 Producer / Topic / Consumer。
- 为什么 Node WebSocket 不能直接充当可靠的消息队列。
- 为什么把 Indexer 直接连接多个下游，会产生耦合、故障传播和重复读取问题。
- Kafka 如何提供 Buffer、Decoupling、Replay 和 Fan-out。
- Kafka 在整条 Blockchain Realtime Pipeline 中处于什么位置。

本课只建立 Kafka 的系统角色。Topic、Partition、Offset、Consumer Group 的内部协作机制从下一课开始系统展开；本课不进入 Broker 存储、Replication、KRaft 等 Kafka 内核实现。

---

## 一、先从一个看起来完全不需要 Kafka 的系统开始

假设我们做一个 Wallet Activity Platform。

最直接的架构是：

```text
Ethereum Node / Provider
        ↓
     WebSocket
        ↓
      Indexer
        ↓
     Postgres
        ↓
    Dashboard
```

新区块产生以后：

```text
New Block
   ↓
Indexer
   ↓
Decode Transfer
   ↓
INSERT INTO transfers
   ↓
Dashboard Query
```

这个设计并没有错。

如果系统规模很小、只有一个下游、允许偶尔重新扫描链上历史数据，那么完全可能不需要 Kafka。

所以第一条结论是：

> Kafka 不是 Blockchain Data Platform 的必选组件。

真正的问题是：当系统开始变大以后，直接连接的方式会逐渐出现什么问题？

---

## 二、第一个问题：Producer 和 Consumer 的速度不一样

假设 Indexer 正常情况下每秒产生 1,000 条 Transfer Event。

数据库 Consumer 每秒也能写 1,000 条。

这时系统很平衡：

```text
Indexer
1000 msg/s
   ↓
Database
1000 msg/s
```

但突然某个热门 Mint、空投或市场剧烈波动出现，链上事件量短时间升高：

```text
Indexer
10,000 msg/s
   ↓
Database
1,000 msg/s
```

那么剩下的 9,000 条怎么办？

如果 Indexer 直接调用数据库：

```text
Indexer
   ↓
Database
```

下游变慢，很容易反向拖慢上游。

这叫一个非常重要的工程问题：

> Backpressure。

暂时不用深入它的所有机制，只要先理解：

> 上游产生数据的速度，和下游处理数据的速度，不一定相同。

因此我们希望中间存在一个地方：

```text
Producer
   ↓
[ Buffer ]
   ↓
Consumer
```

Producer 可以先把事件写进去。

Consumer 再按照自己的处理能力逐步消费。

Kafka 就可以承担这个 Buffer。

于是系统变成：

```text
Indexer
   ↓
 Kafka
   ↓
Database Consumer
```

高峰时：

```text
Indexer
10,000 msg/s
   ↓
Kafka
████████████
   ↓
Database Consumer
1,000 msg/s
```

只要 Kafka 能保存这些尚未消费的数据，Consumer 可以稍后追上。

---

## 三、第二个问题：Indexer 不应该知道所有下游是谁

系统最开始可能只有：

```text
Indexer
   ↓
Postgres
```

后来业务增加了。

现在 Transfer Event 同时需要：

```text
Postgres
Risk Engine
Wallet Notification
Analytics
Alert System
```

如果没有消息层，很容易变成：

```text
                 → Postgres
                /
Indexer → Risk Engine
                \
                 → Notification
                  \
                   → Analytics
                    \
                     → Alert
```

Indexer 开始需要知道：

- Postgres 在哪里；
- Risk Engine API 在哪里；
- Notification Service 在哪里；
- Analytics Service 在哪里；
- 谁失败了要不要 Retry；
- 哪个系统超时；
- 哪个系统暂时下线。

Indexer 的职责本来应该是：

> 把链上事实可靠地提取和解析出来。

现在却逐渐变成：

> 管理所有下游系统。

这就是 Coupling。

---

## 四、Kafka 把这种直接依赖拆开

加入 Kafka 后：

```text
Ethereum
   ↓
Indexer
   ↓
Kafka
   ├──→ Database Consumer
   ├──→ Risk Consumer
   ├──→ Notification Consumer
   └──→ Analytics Consumer
```

Indexer 不需要知道：

```text
谁会消费？
有几个消费者？
消费者什么时候上线？
消费者处理多快？
```

Indexer 只负责：

> Produce Event。

下游只负责：

> Consume Event。

这叫：

> Decoupling。

也就是解耦。

【Blockchain Data Engineer 视角】

Indexer 的职责边界变得更清晰：

```text
Blockchain
   ↓
Extract / Decode
   ↓
Canonical Event
   ↓
Publish
```

至于 Canonical Event 后面被谁使用，不再由 Indexer 决定。

---

## 五、第三个问题：Consumer 挂掉以后怎么办？

假设：

```text
Indexer
   ↓
Risk Engine
```

14:00 Risk Engine 挂掉。

14:10 才恢复。

期间 Ethereum 仍然不断产生交易。

如果采用实时 WebSocket 直接 Push：

```text
14:01 Event A
14:02 Event B
14:03 Event C
...
```

Risk Engine 当时不在线，这些消息是否还能重新获得？

这取决于你的系统有没有另外做持久化、补数和恢复机制。

WebSocket 本身解决的是：

> 建立实时连接并推送新数据。

它并不天然等于：

> 一个可以长期保存消息、记录消费位置、允许 Consumer 以后回来继续读的可靠消息日志。

这就是 Kafka 和 WebSocket 很容易混淆的地方。

---

## 六、WebSocket 和 Kafka 的角色不同

【RPC / Provider 视角】

WebSocket Subscription 解决：

> Node / Provider 如何把“新事件出现了”快速通知给客户端。

例如：

```text
Ethereum Node
     ↓ WebSocket
Indexer
```

它主要解决：

> Push。

---

【Data Platform 视角】

Kafka 解决的是：

> 数据进入平台以后，如何被可靠地缓冲、保存、分发和重新消费。

所以：

```text
WebSocket
≈ Realtime Ingestion Channel

Kafka
≈ Durable Event Streaming Layer
```

它们不是竞争关系。

一个常见组合反而是：

```text
Ethereum Node
     ↓
 WebSocket
     ↓
  Indexer
     ↓
   Kafka
     ↓
 Consumers
```

---

## 七、第四个问题：新 Consumer 加入后，能不能读过去的数据？

假设今天系统只有：

```text
Database Consumer
```

一个月之后，公司新增：

```text
Fraud Detection Consumer
```

风控团队说：

> 我不仅需要从今天开始的数据，我还想拿过去 7 天的 Transfer Event 重新跑一次模型。

如果消息只是在实时到达时调用一次 API：

```text
Event
  ↓
API Call
  ↓
结束
```

过去的数据已经没有了。

但 Kafka 更接近一个：

> Append-only Event Log。

可以先建立一个简化心智模型：

```text
Kafka Log

Offset 0  Event A
Offset 1  Event B
Offset 2  Event C
Offset 3  Event D
Offset 4  Event E
...
```

消息写进去以后，不是 Consumer 一读就立即消失。

在 Retention 范围内，它仍然可以继续存在。

所以新的 Consumer 可以：

```text
从最新位置开始
```

也可以在允许的范围内：

```text
从较早位置 Replay
```

这就是 Kafka 非常重要的另一个能力：

> Replay。

下一课我们会正式进入 Offset。

---

## 八、这里要区分 Kafka 和传统 Queue 的直觉

很多人第一次学 Kafka，会把它想象成：

```text
Queue

Producer
   ↓
[ A B C D ]
   ↓
Consumer
```

Consumer 取走 A：

```text
[ B C D ]
```

这种理解只能解释一部分。

Kafka 更有用的心智模型其实是：

```text
Distributed Append-only Log
```

例如：

```text
0 → A
1 → B
2 → C
3 → D
4 → E
```

Consumer 不是简单地“把消息拿走”。

更接近：

> 我现在读到这个 Log 的哪个位置。

于是：

```text
Consumer A: 读到 offset 4
Consumer B: 读到 offset 2
Consumer C: 刚开始读
```

同一份 Event Log 可以服务不同 Consumer。

这也是 Kafka 能够同时做到 Fan-out 与 Replay 的基础。

---

## 九、银行系统类比

银行交易系统可以把一笔新交易发布为：

```text
Transaction Event
```

下游可能包括：

```text
                 ┌→ 反洗钱
                 ├→ 实时风控
Core Banking → Kafka → 短信通知
                 ├→ 数据仓库
                 └→ 客户画像
```

核心交易系统不应该为了新增一个“客户画像系统”，就修改自己的核心交易逻辑。

否则每增加一个系统：

```text
Core System
+ 调风控 API
+ 调短信 API
+ 调 AML API
+ 调画像 API
+ 调数据仓库 API
...
```

核心系统会越来越脆弱。

Kafka 的价值之一就是把：

> “交易发生”

与：

> “谁需要使用这笔交易”

分离开。

Blockchain Data Platform 完全是相同的问题。

---

## 十、回到 Ethereum：Kafka 放在哪里？

现在可以形成第一版完整架构：

```text
Ethereum Node / Provider
        ↓
WebSocket / RPC
        ↓
Indexer
        ↓
Decode / Normalize
        ↓
Kafka
        ↓
 ┌──────┼─────────┬──────────┐
 ↓      ↓         ↓          ↓
DB    Risk     Alert     Analytics
```

这里每一层职责不同。

### Node / Provider

负责提供链上数据。

### WebSocket / RPC

负责把数据交给 Indexer。

### Indexer

负责：

```text
Block
↓
Transaction
↓
Receipt
↓
Log
↓
Decoded Event
```

把链上的原始事实转换成平台可以理解的数据事件。

### Kafka

负责：

```text
Buffer
+
Decouple
+
Persist temporarily
+
Distribute
+
Replay
```

### Consumer

负责某个具体下游任务。

例如：

```text
Transfer Event
     ↓
Wallet Balance Consumer
     ↓
wallet_balance
```

或者：

```text
Transfer Event
     ↓
Risk Consumer
     ↓
Alert
```

---

## 十一、Kafka 解决的不是“Transform”

这一点很重要。

Kafka 本身不是因为我们需要：

```text
amount_raw
÷ 10^decimals
→ amount
```

才存在。

Transform 可以发生在：

```text
Indexer
Consumer
Flink
ETL Job
```

Kafka 核心解决的是：

> Event 如何在多个独立系统之间可靠地流动。

所以不要把：

```text
Kafka = ETL
```

也不要把：

```text
Kafka = Database
```

Kafka 在我们的课程语境里首先是一层：

> Event Streaming Infrastructure。

---

## 十二、Kafka 也不是你的最终事实库

Kafka 能保存消息，但通常不能因此得出：

> 有 Kafka 就不需要数据库。

例如最终你仍然可能需要：

```text
Postgres
ClickHouse
Data Warehouse
Object Storage
```

原因是它们解决的问题不同。

Kafka 更擅长：

```text
事件持续进入
↓
按顺序记录
↓
多个 Consumer 消费
↓
支持一段时间内 Replay
```

数据库 / Warehouse 更擅长：

```text
SELECT
JOIN
GROUP BY
历史分析
Serving Query
长期数据管理
```

所以架构不是：

```text
Kafka OR Database
```

而通常是：

```text
Kafka
  ↓
Database / Warehouse
```

Module 11 会专门讨论不同数据库的职责。

---

## 十三、为什么不让每个 Consumer 自己去读 Ethereum？

还有一种看起来很自然的方案：

```text
Ethereum
 ├→ Wallet Service
 ├→ Risk Service
 ├→ Analytics Service
 └→ Alert Service
```

也就是每个下游自己调用 RPC。

这当然也可以工作，但规模大以后会产生：

### 1. 重复工作

四个系统都在重复：

```text
eth_getLogs
Decode Transfer
Retry
Rate Limit
Reorg Handling
```

### 2. 口径漂移

Wallet Service 的 Decoder 是 v1。

Risk Service 的 Decoder 是 v2。

Analytics 又有自己的逻辑。

最后同一笔 Transfer 可能出现不同解释。

### 3. Provider 成本增加

多个系统重复读取同一份链上原始数据。

### 4. 故障处理重复建设

每个团队都需要自己解决：

```text
Retry
Checkpoint
Backfill
Reorg
Rate Limit
```

因此更合理的一种平台架构是：

```text
Ethereum
   ↓
统一 Indexer
   ↓
Canonical Event
   ↓
Kafka
   ↓
多个业务 Consumer
```

这和 Module 8 的数据建模其实也连接起来：

> 上游尽量形成统一、稳定的事实口径，下游基于同一份 Canonical Fact 做不同业务计算。

---

## 十四、Kafka 最核心的四个价值

这一课不需要先背很多 Kafka 名词。

先锁定四个词：

### 1. Buffer

生产速度和消费速度暂时不一致时，中间可以积压。

### 2. Decoupling

Producer 不需要知道所有 Consumer。

### 3. Fan-out

同一份事件可以被多个不同下游独立消费。

### 4. Replay

Consumer 可以从历史位置重新读取仍在 Retention 范围内的数据。

把它写成一张图：

```text
                 ┌→ Consumer A
Producer → Kafka ├→ Consumer B
                 └→ Consumer C
            ↑
      Buffer / Replay
```

如果你真正理解这张图，Kafka 的大框架就已经建立起来了。

---

## 十五、但 Kafka 没有自动解决所有问题

这里提前防止一个常见误区。

加入 Kafka 后：

```text
Indexer
  ↓
Kafka
  ↓
Consumer
  ↓
Database
```

仍然存在很多问题：

- 一条 Event 应该写到哪里？
- 如何保证同一个 Wallet 的事件顺序？
- Consumer A 和 Consumer B 如何各自记录处理位置？
- 两个 Consumer 实例如何并行？
- Consumer 写库成功但 Offset 没提交怎么办？
- Offset 提交了但数据库写入失败怎么办？
- 重复消费怎么办？
- Reorg 怎么处理？

这些并没有因为“用了 Kafka”自动消失。

它们正是 Module 10 后面几课的内容。

下一课我们首先进入：

```text
Topic
Partition
Offset
```

因为 Kafka 要实现刚才的 Buffer、并行、顺序和 Replay，必须先解决：

> 数据放在哪里，以及 Consumer 如何定位自己读到哪里。

---

## 十六、和上一课连接起来

上一课我们说：

```text
Streaming
=
持续处理一个 Unbounded Stream
```

于是立即出现问题：

> 这个 Stream 到底存在哪里？

如果只存在 WebSocket 连接上：

```text
连接断了
↓
中间数据怎么办？
```

Kafka 给出的思路是：

```text
Unbounded Event Stream
        ↓
先写入 Durable Log
        ↓
Consumer 按自己的位置持续读取
```

所以你可以把前两课连接成：

```text
第 1 课
为什么需要 Streaming？
        ↓
因为数据持续产生

第 2 课
Streaming 数据放在哪里、怎么让多个系统可靠消费？
        ↓
引入 Kafka / Event Log
```

---

## 本课核心结论

第一：

> Kafka 不是为了“让数据变实时”，而是为了让持续产生的事件能够在多个系统之间可靠流动。

第二：

> WebSocket 主要解决实时 Push；Kafka 主要解决 Buffer、持久化事件日志、解耦、分发和 Replay。

第三：

> Indexer 应负责形成 Canonical Event，而不应该承担所有下游系统的调用和故障管理。

第四：

> Kafka 更适合被理解为 Distributed Append-only Log，而不只是“消息取走就消失”的普通 Queue。

第五：

> Kafka 不替代 Database，也不替代 ETL / Transform；它是 Event Streaming Infrastructure。

---

## 理解检查

### 问题一

假设目前系统是：

```text
Ethereum Node
   ↓
Indexer
   ↓
Postgres
```

系统只有一个内部 Dashboard，数据量也很小。

这种情况下是否一定要加入 Kafka？

请说明你的判断依据。

### 问题二

后来系统新增三个需求：

```text
Wallet Balance
Risk Alert
Realtime Notification
```

如果 Indexer 直接分别调用三个系统，相比：

```text
Indexer
   ↓
Kafka
   ├→ Wallet Consumer
   ├→ Risk Consumer
   └→ Notification Consumer
```

后者最核心解决了什么架构问题？

请不要只回答“性能更高”。

### 问题三

Risk Consumer 从 14:00 到 14:10 宕机，但 Indexer 和 Kafka 一直正常。

14:10 Risk Consumer 恢复以后，我们希望它把这 10 分钟错过的数据继续处理。

这里 Kafka 必须具备哪两类能力，才能支持这个恢复过程？

请先用自己的话回答，不要求使用完整 Kafka 术语。

## 用户回答

问题一：这里的数据场景很简单，所以不需要加入 Kafka，也能满足需求。最主要的是可以让系统更简洁，出故障的概率也越小

问题2：最核心的是明确了 Indexer 的职责边界

问题三，这里 Kafka 具有 buffer 的功能，能把一部分的数据储存在缓存里面，供下游使用。
另一个能力是 Offset 功能，它可以记录 consumer 的进度

## 老师判断与校准

三题全部通过。

- 问题一：判断正确。Kafka 不是默认必选组件；当系统只有单一下游、数据量较小、直接链路已经能够满足可靠性与恢复要求时，不引入 Kafka 可以减少基础设施复杂度、运维成本和新的故障点。是否加入 Kafka，应由系统的解耦、缓冲、分发、恢复和扩展需求决定，而不是由“是否实时”决定。
- 问题二：判断正确，而且抓住了架构核心。加入 Kafka 后，Indexer 的职责可以稳定在“读取链上数据 → Decode / Normalize → 形成 Canonical Event → Publish”；它不再负责知道所有下游系统、逐个调用下游、处理各下游的超时与故障。更完整地说，这就是 Producer 与 Consumer 的 Decoupling。
- 问题三：整体正确，但需要一个关键术语校准。Kafka 的 Buffer 更准确地说不是普通“缓存”，而是可持久保存一段时间的 Event Log。Consumer 宕机期间，Event 仍保留在 Log 中；恢复后，再根据 Consumer 的 Offset 找到自己之前消费到的位置并继续读取。因此这里依赖的是两类能力：一是 Durable Event Retention / Buffer，二是 Consumer Position / Offset。

## 结课判定

Module 10 第 2 课理解检查通过，正式完成。已建立 Kafka 的系统角色心智模型：Kafka 不是为了“让数据变实时”，而是作为 Event Streaming Infrastructure，在 Producer 与 Consumer 之间提供 Buffer、Decoupling、Fan-out 和 Replay；WebSocket 主要承担实时 Push，Kafka 则承担可持久的事件日志与消费恢复。
