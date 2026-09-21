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

## 用户补充回答（问题一、问题二）

问题一：因为不同的业务职责会从不同的方面、不同的维度对同一批数据进行分析，所以我们应该将不同的业务职责划分成不同的 Consumer Group。

如果它们在同一个 Group 里面的话，就会因为竞争关系而获取不到完整的 Event 数据。
问题二：wallet activity的grain应该是one wallet activity event，而不是one transfer。

因为one transfer里面实际上会包含多个wallet的activity。比如一条transfer，因为它会有from和to，这样的话它有可能就会有两个wallet address。但是我们把wallet作为key的话，所以得需要把transfer再拆分。

## 老师补充校准（问题一、问题二）

问题一、问题二补充通过。

- Consumer Group 按独立业务职责划分，而不是按 Partition 数量划分。不同 Consumer Group 可以独立读取完整 Topic；如果多个业务逻辑放在同一个 Group 中，它们会形成 work-sharing / competition，只会分摊 Partition，而不是每套业务都完整读取全部 Event。
- wallet_activity 的 Message Grain 应定义为 one wallet activity event。因为原始 one transfer 同时包含 from 和 to，可能对应两个 Wallet；如果要使用 wallet_address 作为 Partition Key，就需要将 Transfer fan-out / transform 为面向单个 Wallet 的 activity event，使每条消息只有一个明确的 wallet_address。

结论：综合检查问题一、问题二正式通过。

## 用户回答（问题三、问题四、问题五）

问题三：

1. 重启后，Consumer 会检查 Offset 进度。因为之前 Offset 没有正常 Commit，所以它会再次把刚才的 Event 再消费一次，2
2. 这更接近 At-least-once 这种 Delivery Semantics。
3. Sink   必须支持 Idempotency（幂等性）。就是有刚才这种情况出现，因为 Consumer 有可能会多次对同一份 Event 进行写入操作，那么 Synker 它必须支持 Idempotency，这样才不会导致有数据问题。
4. 对 ERC-20 Transfer Fact，它的 Unique Key 应该是 Chain ID、Wallet Address、TX Hash、Log Index

问题是：A 应该用 retry，因为它是临时性的故障，所以用 retry 可以过会儿重试. B 应该用 replay，Replay 的三个前提是 durable retention、position，还有 idempotency。

判断是不是会发生 back pressure，主要观察的指标是 product 速度和 consumer 速度的差是不是越来越大？

主要处理的办法有：

1. Scale consumers
2. Optimize processing
3. Reduce downstream pressure

问题 5

1：这是 canonical correctness 问题？

2. pipeline 应该如何修复旧数据？

pipeline 可以再创建一个 reorg pipeline，专门用来处理这种情况。

3. block number 只是 block 的一个高度的位置，但是这个高度位置上具体的 block 内容是有可能发生变化的，所以它并不稳定，不能作为稳定的 block identity
4. 因为它把数据状态更清晰化，这样下游更能区分 fresh but mutable vs stable canonical fact

## 老师判断与校准（问题三、问题四、问题五）

### 问题三：第 1、2、3 小题通过；第 4 小题需要校准

- Consumer 在 DB write 成功但 Offset 未 Commit 时 Crash，重启后会从旧的 committed position 重新读取该 Event，因此发生 duplicate processing，这一判断正确。
- 这属于 At-least-once 语义，判断正确。
- Sink 必须支持 Idempotency，因为同一 Event 可能被重复处理；重复执行必须收敛到同一业务结果，这一点正确。
- ERC-20 Transfer Fact 的 Source Identity / Unique Key 不应包含 wallet_address。更合适的是：
  (chain_id, tx_hash, log_index)
  因为一条 Transfer Fact 本身已经由交易哈希和日志位置唯一确定；wallet_address 是该 Transfer 的业务字段之一，不是唯一身份的一部分。

### 问题四：A、B 和处理手段通过；Backpressure 指标需要校准

- A：Postgres 网络超时属于 transient failure，使用 Retry，正确。
- B：Decoder Bug 影响历史范围，应使用 Replay；Replay 需要 Retention、Position、Idempotency，正确。
- C：Producer 15,000 msg/s、Consumer 9,000 msg/s 时，会持续新增约 6,000 msg/s 的积压，属于持续 Backpressure。
- “Producer 与 Consumer 速度差”是根因 / capacity gap，但运行时最关键的观测指标应是 Consumer Lag，尤其是 Lag Trend。若 Lag 持续增长，说明 Consumer 长期追不上 Producer。
- Scale Consumers、Optimize Processing、Reduce Downstream Pressure 三类处理办法正确。需要同时记住：增加 Consumer 的有效并行度仍受 Partition Count 限制。

### 问题五：第 1、3、4 小题通过；第 2 小题需要补完整

- 第 1 小题：这是 Canonical-chain Correctness 问题，正确。
- 第 2 小题：可以有独立的 Reorg Correction Flow / Reorg Pipeline，但关键不是“多建一个 Pipeline”本身，而是它必须执行正确的修复动作：
  Detect Reorg → Find Common Ancestor → Invalidate / Rollback Old Branch → Replay New Canonical Branch。
  Realtime、Backfill、Reorg Replay 可以复用同一套 processing logic，但应维护 separate execution state。
- 第 3 小题：block_number 只表示高度，不表示 immutable block identity；同一高度在 Reorg 前后可以对应不同 block_hash，判断正确。
- 第 4 小题：区分 latest / unconfirmed 与 confirmed / finalized，是为了让下游明确 fresh but mutable 与 stable canonical fact，判断正确。

结论：问题三、问题四、问题五均已基本掌握；需要补充三点：Transfer Fact Unique Key、Backpressure 的核心运行指标 Consumer Lag、Reorg Correction 的完整修复流程。
