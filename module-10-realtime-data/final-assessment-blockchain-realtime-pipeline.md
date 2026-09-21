## Module 10 结束标准综合检查

### 检查目标

本次综合检查不引入新知识，只验证是否达到 Module 10 的结束标准。

你需要能够独立设计并解释：

Ethereum / EVM Node or Provider
↓
Indexer / Event Producer
↓
Kafka Topic
↓
Consumer Group
↓
Realtime Transform / Enrichment
↓
Serving DB / Warehouse / Alert / API

同时必须覆盖：

- Message Grain
- Topic / Partition / Partition Key
- Consumer Group / Offset
- Ordering vs Parallelism
- Delivery Semantics
- Idempotency
- Retry / Replay
- Backpressure / Consumer Lag
- Reorg Correction
- Finality / Confirmation
- Batch Backfill
- Reconciliation

---

## 场景

你要为一个 Ethereum 钱包数据产品设计一条实时 Pipeline。

产品需要提供：

1. ERC-20 Transfer 实时流水。
2. Wallet Token Balance 实时更新。
3. Risk Alert：当某个钱包收到大额 USDC Transfer 时尽快告警。
4. 数据必须支持 Reorg 修正。
5. Consumer 或数据库短暂故障后必须可以恢复。
6. 如果 Decoder Bug 影响历史数据，需要支持历史修复。

假设 Indexer 已经可以从 Ethereum Provider 获取 Block / Transaction / Receipt / Log，并能解码 ERC-20 Transfer。

---

## 问题一：设计主数据流

请你从 Source 开始，设计完整主链路。

至少说明：

- Provider / Indexer 在哪里。
- Kafka 放在哪里。
- Topic 里是什么 Event。
- Consumer Group 有哪些。
- 最终写到哪些 Sink。

你可以用类似下面的格式回答，但不要求完全一致：

Provider
→ Indexer
→ Kafka Topic
→ Consumer Group
→ Transform
→ Sink

---

## 问题二：Topic、Partition 与 Ordering

假设我们有一个 wallet_activity Topic。

请回答：

1. Message Grain 你会怎么定义？
2. Partition Key 你会选择什么？
3. 为什么这样可以保证同一个 Wallet 的局部 Ordering？
4. 如果 Topic 有 8 个 Partition、Consumer Group 有 12 个 Consumer，最多有多少个 Consumer 能同时真正处理数据？为什么？

---

## 问题三：Offset、Delivery Semantics 与 Idempotency

假设 Consumer 执行：

Read Event
→ Write Postgres
→ Commit Offset

在 Postgres 写入成功、Offset Commit 之前 Consumer Crash。

请解释：

1. 重启后会发生什么？
2. 这更接近哪一种 Delivery Semantics？
3. 为什么 Sink 必须支持 Idempotency？
4. 对 ERC-20 Transfer Fact，你会用什么 Unique Key？

---

## 问题四：Retry、Replay 与 Backpressure

现在出现三个不同情况：

A. Postgres 网络超时 20 秒。
B. Decoder Bug 影响了过去 100,000 条 Event。
C. Producer 速度长期是 15,000 msg/s，Consumer 只能处理 9,000 msg/s。

请分别说明：

- A 应该用 Retry 还是 Replay？为什么？
- B 应该用 Retry 还是 Replay？Replay 安全成立需要哪些前提？
- C 会发生什么？你会观察哪个关键指标？有哪些处理办法？

---

## 问题五：Reorg 与 Finality

假设一条 Transfer 已经写入数据库，但随后所在 Block 被 Reorg 掉。

请回答：

1. 这是 Delivery Correctness 问题，还是 Canonical-chain Correctness 问题？
2. Pipeline 应该如何修正旧数据？
3. 为什么不能只依赖 block_number 作为稳定 Block Identity？
4. Wallet Balance 为什么要区分 latest / unconfirmed 与 confirmed / finalized？

---

## 问题六：为什么仍然需要 Batch

即使系统已经有：

- Kafka
- Retry
- Replay
- Idempotency
- Reorg Correction

为什么仍然需要 Batch Backfill / Reconciliation？

至少说出两个原因，并说明：

> Streaming 和 Batch 各自更擅长解决什么问题？

---

## 问题七：最后做一次完整系统设计总结

请用你自己的话，把整个系统压缩成三个部分：

### Happy Path

正常情况下 Event 如何从链上到达最终 Sink？

### Failure Path

Consumer / Sink / Processing 出错时如何恢复？

### Repair Path

Reorg、历史 Bug、数据漂移出现以后如何修复？

如果你能把这三个 Path 讲清楚，就说明你已经真正建立起 Blockchain Realtime Data Pipeline 的系统模型。

---

## 通过标准

本次不是要求术语完全一致，而是检查以下能力：

1. 能从业务需求推导 Streaming Architecture。
2. 能解释 Kafka 中 Topic / Partition / Consumer Group / Offset 的职责边界。
3. 能解释 Ordering 与 Parallelism 的 trade-off。
4. 能解释 At-least-once + Idempotency。
5. 能正确区分 Retry 与 Replay。
6. 能识别并处理 Backpressure / Consumer Lag。
7. 能区分 Delivery Correctness 与 Canonical-chain Correctness。
8. 能设计 Reorg / Finality / Backfill / Reconciliation。
9. 能形成 Happy Path + Failure Path + Repair Path 的完整生产思维。

## 用户回答（问题一、问题二）

问题一：

Provider Index 是在数据流的最前端，然后 Kafka 是 Index 的下游，接收 Index 发送过来的数据。

Topic 里面存的可以是 wallet transfer event。

Consumer group 可以设置两个，因为 partition 也设置了两个。

最终写到哪些 sink？写到 PostgreSQL。

问题二：

wallet activity topic 应该是要统计 wallet 的活动情况，那么就需要统计到 transfer，从 transfer 里面获取 wallet 的 activity。

所以：

1. message grain 应该定义成：message 应该是 Indexer 发送给 Kafka 的一个 message，那么它的 grain 应该是 one row, one transfer
2. partition key是 wallet address
3. 因为 Kafka 可以保证一个 partition 里面的 event 是 ordered，然后我们通过 wallet address 作为 partition key 的话，那同一个 wallet 就会进入同一个 partition。

那么，从同一个 partition 里面获取的 event 就是按顺序的，同样，得到的某一个钱包地址的活动顺序也是 ordered。
4. 最多有 8 个 consumer 能同时真正处理数据。

因为在 partition 分配的时候，同一个 partition 只能分给一个 consumer group 里面的一个 consumer，不能分给同一个 group 里面的多个 consumer。所以在这个 consumer group 里面，虽然它有 12 个 consumer，但 8 个 partition 只能分配给 8 个 consumer，会有 4 个 consumer 是空闲的

## 老师判断与校准（问题一、问题二）

### 问题一：部分通过，需要校准

- Provider / Indexer 位于数据流前端、Kafka 位于 Indexer 下游，这个架构判断正确。
- Topic 中承载 wallet / transfer 相关 Event，这个方向正确。
- “Consumer Group 可以设置两个，因为 Partition 也设置两个”这一点不正确。Consumer Group 的数量不是由 Partition 数量决定的。Partition 数量决定的是同一个 Consumer Group 内的最大并行度；Consumer Group 数量取决于有多少套独立消费逻辑。
- 对本题场景，更自然的设计至少可以包含：
  - Transfer Persistence Consumer Group：写 ERC-20 Transfer Fact。
  - Wallet Balance Consumer Group：更新 Wallet Token Balance。
  - Risk Alert Consumer Group：检测大额 USDC Transfer 并触发告警。
- Sink 也不一定只有 PostgreSQL。PostgreSQL 可以作为 Transfer / Balance 的 Serving Sink；Risk Alert 还可能写入 Alert Service / API / Notification Channel。用户回答中的 PostgreSQL 是正确的一类 Sink，但不完整。

### 问题二：第 2、3、4 小题通过；第 1 小题需要校准

- Partition Key = wallet_address：在 wallet_activity 这个 Topic 语义下，方向正确。
- 同一个 wallet_address 会通过相同 Partition Key 路由到同一 Partition，因此可以利用 Kafka 的 partition-level ordering 保持该 Wallet 的局部顺序，这一点正确。
- 8 个 Partition + 12 个 Consumer：最多只有 8 个 Consumer 能同时承担 Partition Processing，另外 4 个会 idle，这一点正确。
- Message Grain 不能直接定义成 one transfer，如果同时又要求 Partition Key = wallet_address。原因是一个 ERC-20 Transfer 同时有 from_wallet 和 to_wallet，一个原始 Transfer Event 无法同时以两个 wallet_address 作为 Partition Key。
- 更合理的设计是把 wallet_activity 定义成派生流：one message = one wallet activity event。这样一笔 Transfer 可以 fan-out 成两条 wallet_activity：
  - from_wallet activity
  - to_wallet activity
  每条消息只有一个明确 wallet_address，因此可以稳定用 wallet_address 作为 Partition Key。

结论：问题一尚未完全通过；问题二中第 2、3、4 小题通过，第 1 小题需要将 Grain 从 one transfer 调整为 one wallet activity event。
