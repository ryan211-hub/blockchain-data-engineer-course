## Lesson Contract

所属 Module：Module 10 — 实时数据

本课核心问题：

> Blockchain Streaming 和普通 Streaming 最大的不同是什么？当链发生 Reorg、数据尚未 Finalized、实时结果与最终链状态不一致时，Pipeline 如何修正？为什么最终仍然需要 Batch Backfill / Reconciliation？

学完以后，你应该能够解释：

- 为什么 Blockchain Streaming 不能把“已经收到的 Event”直接视为永远不变的事实。
- Reorg 对已经写入下游的数据意味着什么。
- Finality / Confirmation Depth 在实时数据中的作用。
- Realtime、Backfill、Reorg Replay、Reconciliation 四者如何分工。
- 为什么 Streaming 系统仍然需要 Batch。
- 为什么 Delivery Correctness 和 Canonical-chain Correctness 是两类不同问题。
- 如何设计一条同时具备实时性与最终正确性的链上数据 Pipeline。

本课不深入 Ethereum 共识算法、Finality gadget、Validator 协议细节；只讨论 Data Engineer 需要处理的数据语义和修复路径。

---

## 一、普通 Streaming 的一个隐含假设

普通业务系统中，很多 Event 一旦产生，就可以近似认为它是已经发生的事实。

例如银行系统：

transaction_id = T123
amount = 100

只要上游没有人为撤销或冲正，这条业务事实通常不会因为底层“历史重写”而消失。

但 Blockchain 不一样。

一条链上 Event 在刚出现时，可能只是：

> currently canonical

而不是：

> permanently final.

这是 Blockchain Streaming 最核心的特殊性。

---

## 二、一个实时 Pipeline 可能处理了后来变成 Orphan 的数据

假设 Indexer 收到：

Block 100
  Tx A
  Transfer Alice -> Bob 100 USDC

实时 Pipeline 很快处理：

Block 100
→ Decode Transfer
→ Kafka
→ Consumer
→ Postgres
→ Dashboard

此时 Dashboard 已经显示：

Alice -100 USDC
Bob +100 USDC

但随后发生 Reorg。

新的 canonical chain 上，旧 Block 100 被替换。

那么旧 Transfer 可能变成：

> orphaned fact.

也就是说：

Pipeline 当时没有处理错。

输入在当时也是 canonical。

但后来：

> the truth changed.

---

## 三、这和普通 Retry / Duplicate 完全不是同一个问题

上一课处理的是：

- Consumer crash
- Event retry
- duplicate delivery
- lag

这些都属于：

> Delivery / Processing Correctness.

Reorg 解决的是另一类问题：

> Canonical-chain Correctness.

可以这样区分：

Delivery Correctness：这条 Event 有没有被正确消费、写入？

Canonical Correctness：这条 Event 后来还算不算 canonical fact？

即使 Delivery 做到 perfectly exactly-once，也不能自动解决 Reorg。

---

## 四、Exactly-once 也挡不住 Reorg

假设：

Old Block 100
→ Event X
→ exactly-once write to DB

这个写入完全没有重复，也没有丢失。

后来 Reorg：

Old Block 100 becomes orphan
New Block 100 becomes canonical

那么 Event X 仍然需要被撤销或标记无效。

所以：

> Exactly-once processing does not imply canonical correctness.

这句话很重要。

---

## 五、Reorg 后实时 Pipeline 应该做什么？

你在 Module 9 已经学过基本流程：

Detect Reorg
→ Find Common Ancestor
→ Invalidate / Rollback Old Branch
→ Replay New Branch

现在把它放进 Streaming 体系。

例如：

Realtime Indexer
→ detects parent-hash mismatch
→ identifies old branch
→ emits correction / invalidation
→ processes new canonical blocks

下游必须能够接收：

- new canonical events
- orphan / invalidation signals

而不能只会 INSERT 新数据。

---

## 六、两种常见修正方式

### 方式一：Physical Delete / Rollback

旧 canonical 数据直接撤销。

例如：

DELETE FROM transfers
WHERE block_hash = old_block_hash

然后写入新链数据。

优点：查询简单。

缺点：历史变更痕迹较弱。

### 方式二：Canonical Flag / Versioning

保留旧数据，但标记：

is_canonical = false

新链写入：

is_canonical = true

优点：

- auditability
- reorg history
- debugging

对于链上数据平台，保留 canonical / orphan 状态通常更有数据工程价值。

---

## 七、为什么 block_number 不能单独作为稳定身份？

假设：

block_number = 100

Reorg 前：

BlockHash = 0xAAA

Reorg 后：

BlockHash = 0xBBB

两个都是“高度 100”。

所以：

> block number identifies height, not immutable block identity.

稳定身份更依赖：

chain_id + block_hash

对于 Log / Event 则仍要结合：

tx_hash + log_index

并理解这些 Event 所属 Block 是否 canonical。

---

## 八、Finality 是什么？

可以先从 Data Engineering 视角理解：

> Finality tells us how confident we are that a block will not be reverted.

刚产生的 Block 通常：

low confirmation / low certainty

随着更多区块确认，或者达到协议定义的 finalized state：

reorg probability becomes much lower / effectively finalized.

因此实时系统必须在：

Latency
vs
Finality Confidence

之间做 trade-off。

---

## 九、实时系统为什么不能永远等 Finality？

如果钱包余额、风险预警、交易监控都等完全 Finalized 才更新：

系统虽然更稳，但实时性会明显下降。

所以很多系统会分层：

### Fast Path

处理 latest / unfinalized data。

特点：

- low latency
- may change

### Finalized Path

处理 finalized / sufficiently confirmed data。

特点：

- higher confidence
- higher latency

这就是：

> freshness vs certainty.

---

## 十、Confirmation Depth 是一种工程折中

有些系统不会只用“latest”或“finalized”两个极端。

而是设置：

wait N blocks

例如：

Block 100
等到 6 个后续 Block
再标记为 confirmed / stable enough

注意：

这里 N 不是宇宙常数。

它取决于：

- chain behavior
- business risk tolerance
- latency requirement

所以：

> Confirmation depth is a product / risk decision as much as a technical one.

---

## 十一、Blockchain Data Engineer 需要显式建模数据状态

不要只保存：

transfer = true

更成熟的模型可能保存：

block_number
block_hash
status = unconfirmed / confirmed / finalized / orphaned

或者至少：

is_canonical

这样下游才能区分：

fresh but mutable
vs
stable canonical fact.

---

## 十二、Streaming 的职责：尽快反映最新状态

Realtime Pipeline 的核心价值是：

> low-latency propagation.

例如：

new block
→ index
→ decode
→ Kafka
→ consumer
→ dashboard / alert

它擅长：

- fast updates
- continuous processing
- incremental state change

但不代表它天然擅长：

- large historical repair
- full consistency scan
- long-range recomputation

这些通常更适合 Batch。

---

## 十三、为什么 Streaming 仍然需要 Batch？

有三个核心原因。

### 1. Retention 不一定覆盖全部历史

Kafka 只保存有限时间。

如果问题发生在更早以前，只能从链历史 / Archive Source Backfill。

### 2. Reconciliation 需要大范围对账

Streaming 很适合增量处理，但不擅长证明“整个历史都没错”。

### 3. Logic Bug 可能影响大范围历史

如果 Decoder Bug 影响过去 3 个月，直接用实时 Consumer 慢慢 Replay 可能成本很高。

Batch Backfill 更适合大范围重算。

所以：

> Streaming gives freshness; Batch gives repairability and completeness.

---

## 十四、Reconciliation 是什么？

Reconciliation 可以理解为：

> compare derived data with an authoritative or independently recomputed source.

例如：

Realtime Wallet Balance = 1000 USDC

Batch recompute from canonical transfers = 980 USDC

说明：

Realtime state drifted.

这时需要：

detect mismatch
→ identify cause
→ repair

所以 Reconciliation 的目标不是“再跑一遍 ETL”这么简单。

它是在验证：

> Is our derived state still correct?

---

## 十五、Realtime + Reconciliation 是双保险

实时 Pipeline 负责：

Fast / Incremental / Low Latency

Reconciliation 负责：

Slow / Independent / Correctness Check

可以把它理解成：

Realtime Path
→ keep data fresh

Batch Reconciliation
→ keep data honest

这在金融系统里也很常见。

例如实时交易系统先入账，日终再做总账核对。

Blockchain Data Platform 也是类似思想。

---

## 十六、Batch + Stream：不是两套逻辑

最差的设计是：

Realtime logic
≠
Backfill logic

因为两套代码很容易产生口径漂移。

更好的设计是：

Realtime
Backfill
Reorg Replay

尽可能复用：

> same normalization / decoding / process_block logic.

区别主要是：

Input Range
Execution Mode
Progress State

这和你 Module 9 已经建立的模型完全一致。

---

## 十七、Realtime、Backfill、Reorg Replay 的状态不要混用

例如：

Realtime Checkpoint / Offset

不能被 Backfill 随便修改。

Backfill 应该有自己的：

Backfill Cursor / Job State

Reorg Replay 也要有自己的 correction flow。

否则：

修历史
可能会破坏实时进度。

所以：

> shared processing logic, separate execution state.

这是很重要的设计原则。

---

## 十八、一个完整 Blockchain Streaming Pipeline

可以设计成：

Ethereum Node / Provider
↓
Indexer
↓
Kafka
↓
Realtime Consumer
↓
Serving DB / Warehouse

旁边同时存在：

Reorg Detector
→ rollback / invalidate / replay

以及：

Batch Backfill
→ historical repair

以及：

Reconciliation Job
→ compare / detect drift / repair

这才是 production-grade blockchain data pipeline。

---

## 十九、一个具体例子：Wallet Balance

Realtime：

Transfer Event
→ update wallet balance quickly

如果 Reorg：

old transfer becomes orphan
→ reverse / invalidate old effect
→ apply new canonical transfer

如果后来发现过去一个月 Decoder Bug：

Batch Backfill
→ recompute historical transfers

如果想验证余额：

Reconciliation
→ recompute canonical balance
→ compare with realtime balance

所以一个 Wallet Balance 系统真正需要：

Realtime Update
+ Reorg Correction
+ Historical Backfill
+ Reconciliation

四层能力。

---

## 二十、Late-arriving Data 也属于最终一致性问题

并不是所有 Event 都一定按你理想的时间到达。

例如：

- Provider delay
- temporary RPC gap
- consumer lag
- delayed enrichment

某条逻辑上属于旧 Block 的数据，可能晚一些才进入下游。

所以实时系统必须接受：

> event time and processing time may differ.

这也说明为什么仅靠“实时到达顺序”不能作为最终正确性的唯一依据。

---

## 二十一、Lambda / Kappa 不需要背流派

课程里不要求你死记架构名词。

只要理解实际取舍：

一种思路是：

Streaming Path
+
Batch Path

另一种思路是：

尽量通过同一 Stream / Replay 机制统一处理。

但在 Blockchain 场景，由于：

- Kafka Retention finite
- Archive Source exists
- Reorg correction needed
- historical reconciliation needed

现实中经常仍然会存在：

> Streaming + Batch Repair Path.

名字不重要，数据修复能力才重要。

---

## 二十二、Module 10 到这里形成完整闭环

Lesson 1：Streaming = Unbounded Stream

Lesson 2：Kafka = Durable Event Streaming Layer

Lesson 3：Topic / Partition / Offset

Lesson 4：Consumer Group / Ordering

Lesson 5：Delivery Semantics

Lesson 6：Retry / Replay / Backpressure

Lesson 7：Blockchain-specific correctness

最后完整模型变成：

Fast Processing
+ Reliable Delivery
+ Failure Recovery
+ Canonical Correction
+ Historical Repair
+ Reconciliation

---

## 本课核心结论

> Blockchain events can be correct when processed and become non-canonical later.

> Delivery correctness and canonical-chain correctness are different problems.

> Finality is a trade-off between freshness and certainty.

> Reorg handling requires invalidating / rolling back old branch data and replaying the new canonical branch.

> Streaming provides freshness; Batch / Backfill / Reconciliation provide repairability and completeness.

> Realtime, Backfill, and Reorg Replay should reuse processing logic but maintain separate execution state.

> A production blockchain pipeline needs both a fast path and a repair path.

---

## 理解检查

### 问题一

假设一条 Transfer Event 已经通过 Kafka 被 Consumer 成功处理并写入数据库，而且没有重复、没有丢失。

后来它所在的 Block 因为 Reorg 变成 orphan。

为什么不能说这个 Pipeline 已经“完全正确”？

这里暴露的是 Delivery Correctness 问题，还是 Canonical-chain Correctness 问题？

### 问题二

如果一个钱包系统要求“尽量实时显示余额”，但又不能把刚出现的 Block 永远当成最终事实，你会如何处理？

请从：

Freshness
vs
Finality / Confirmation

这个 trade-off 来回答。

### 问题三

为什么一个已经有 Kafka Streaming、Retry、Replay、Idempotency 的系统，仍然需要 Batch Backfill / Reconciliation？

请至少说出两个原因。

## 用户回答

问题一：这个 pipeline 还没有应对 Reorg 这种情况的处理，这里暴露的是 Canonical-chain Correctness 问题
问题二：将Streaming Path  的lag block 尽可能小， 并增加 Reorg Correction 以应对reorg的情况。
问题三：1.Retention 不一定能覆盖全部历史 2. Logic bug 可能影响了大范围的历史，所以也需要 backfill， 3.Reconciliation 需要大范围对账

## 老师判断与校准

- 问题一：通过。这里不是 Delivery Correctness 问题，因为 Event 已经成功消费、没有重复、没有丢失；真正的问题是 Reorg 使原先的 canonical fact 变成 orphan，因此暴露的是 Canonical-chain Correctness。
- 问题二：方向正确，但尚未完整通过。你已经提出两点：Streaming Path 尽量降低延迟，以获得 Freshness；同时增加 Reorg Correction，以修正后来发生的 canonical change。还需要显式补充 Finality / Confirmation 的分层：刚出现的数据可以先作为 latest / unconfirmed 展示，随着 confirmation 增加或达到 finalized，再提升为更稳定的状态。也就是说，需要同时表达“低延迟展示”和“延迟确认”两条路径。
- 问题三：通过。你列出的三个原因都正确：Kafka Retention 可能覆盖不了全部历史；Logic Bug 可能影响大范围历史，需要 Batch Backfill；Reconciliation 需要大范围独立对账来发现 realtime state drift。

## 用户补充回答

这样便于区分 fresh but mutable vs stable canonical fact

## 老师补充校准

问题二补充通过。

你的补充已经准确点出了 Finality / Confirmation 分层的意义：系统需要区分 fresh but mutable 与 stable canonical fact。前者用于低延迟展示，但仍可能因 Reorg 改变；后者经过更多确认或达到 finalized 状态，更适合作为稳定业务事实。这样既保留 Freshness，又显式表达数据的不确定性和最终稳定性。

## 结课判定

Module 10 第 7 课理解检查全部通过，正式完成。已掌握 Blockchain Streaming 的核心特殊性：Delivery Correctness 与 Canonical-chain Correctness 是不同问题；Reorg 需要回滚 / 失效旧分支并重放新 canonical branch；Finality / Confirmation 用于平衡 Freshness 与 Certainty；Streaming 负责低延迟，Batch Backfill / Reconciliation 负责历史修复、完整性检查和最终正确性。
