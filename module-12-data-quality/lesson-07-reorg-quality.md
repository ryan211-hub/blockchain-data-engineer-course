# Module 12 第 7 课｜Reorg Quality：Canonical / Orphan、Rollback 与 Replay

## Lesson Contract

所属 Module：Module 12 — 数据质量

本课核心问题：

> When previously accepted blockchain data becomes non-canonical, how should a data platform detect, correct, and propagate that change?

学完以后，你应该能够解释并设计：

- 什么是 Canonical Block / Orphan Block。
- 为什么 Reorg 是 Blockchain Data Quality 的特殊问题。
- 为什么“已经写入数据库的数据”后来仍可能变错。
- 如何识别 Reorg。
- 为什么仅依赖 `block_number` 不足以识别区块身份。
- Common Ancestor 在 Reorg Repair 中的作用。
- 为什么需要 Rollback old branch。
- 为什么需要 Replay new branch。
- Reorg 如何影响 Raw / Fact / DWS / ADS / 多 Sink。
- 为什么 Reorg Repair 不能只修一张 Fact Table。
- Realtime Path 与 Canonical Path 如何分工。
- Reorg 发生时 Checkpoint 应如何处理。
- 如何让 Reorg Correction 具备 Idempotency。
- 如何验证修复后的 Canonical Correctness。

本课暂不展开：

- Ethereum 共识协议内部细节；
- Fork Choice Rule 数学细节；
- Validator 行为与 Slashing；
- 大规模分布式事务；
- Protocol-level Finality 证明；
- 完整跨链重组处理。

---

## 一、为什么 Blockchain Data Quality 特别难？

传统数据库里，一个已经提交的事实通常不会因为“系统后来发现另一条历史更有效”而整体失效。

区块链不一样。

你可能已经处理：

```text
Block 100
Block 101A
Block 102A
```

然后链发生重组：

```text
Block 100
Block 101B
Block 102B
Block 103B
```

此时原来的：

```text
101A
102A
```

可能不再属于 Canonical Chain。

所以：

> Previously correct data can later become non-canonical.

这就是 Reorg Quality 的核心难点。

---

## 二、[Protocol 视角] Canonical 和 Orphan 是什么？

### Canonical Block

当前被网络接受为主链的一部分。

例如：

```text
100 → 101B → 102B → 103B
```

这里：

```text
101B
102B
103B
```

属于 Canonical Chain。

### Orphan / Non-canonical Block

曾经被看到、甚至被处理过，但后来不再属于主链。

例如：

```text
101A
102A
```

它们可能已经被你的系统写入数据库，但现在必须被标记为：

```text
non-canonical
```

---

## 三、[Data Engineer 视角] Reorg 不是“重复数据问题”

Reorg 和 Duplicate Delivery 不一样。

Duplicate Delivery：

```text
same canonical fact
arrives twice
```

Reorg：

```text
old fact was once accepted
but is no longer canonical
```

所以 Reorg 的问题不是：

> same fact twice

而是：

> historical truth changed from the perspective of the canonical chain.

---

## 四、为什么 block_number 不能作为稳定区块身份？

假设：

```text
block_number = 101
```

在旧分支里：

```text
101A
hash = 0xAAA
```

在新分支里：

```text
101B
hash = 0xBBB
```

两者：

```text
block_number same
block_hash different
```

所以：

> Block Number identifies position; Block Hash identifies block identity.

这是一个非常重要的区分。

---

## 五、Block Table 应该至少保存什么？

一个可做 Reorg Detection 的最小设计通常需要：

```text
chain_id
block_number
block_hash
parent_hash
timestamp
canonical_status
```

关键字段是：

```text
block_hash
parent_hash
```

因为它们让你能够重建：

```text
parent-child relationship
```

---

## 六、Reorg 如何被发现？

最常见的检测方式之一：

你当前已经处理：

```text
checkpoint block = 102A
hash = 0xA102
```

然后收到新 block：

```text
block = 103B
parent_hash = 0xB102
```

但你数据库里：

```text
block 102 hash = 0xA102
```

发现：

```text
new_block.parent_hash
!=
local_tip.block_hash
```

这就是：

> Parent Hash Mismatch.

通常意味着：

```text
possible reorg
```

---

## 七、Parent Hash Mismatch 之后不能直接删最新块

为什么？

因为你还不知道 Reorg 深度。

可能只是：

```text
1-block reorg
```

也可能是：

```text
5-block reorg
```

所以需要继续向后查：

> Where is the common ancestor?

---

## 八、Common Ancestor 是什么？

假设旧链：

```text
100
↓
101A
↓
102A
↓
103A
```

新链：

```text
100
↓
101B
↓
102B
↓
103B
↓
104B
```

两条链最后一个共同区块：

```text
100
```

这就是：

> Common Ancestor.

---

## 九、为什么 Common Ancestor 是 Reorg Repair 的边界？

因为：

```text
<= 100
```

仍然是双方共同认可的历史。

而：

```text
101A → 103A
```

需要失效。

```text
101B → 104B
```

需要重新处理。

所以修复边界是：

```text
common_ancestor + 1
```

---

## 十、完整 Reorg Repair 流程

核心流程：

```text
Detect Reorg
↓
Find Common Ancestor
↓
Mark / Rollback Old Branch
↓
Reset Processing Boundary
↓
Replay New Canonical Branch
↓
Rebuild Derived Data
↓
Reconcile
↓
Resume
```

这和你在 Mini Indexer 里已经做过的：

```text
Detect reorg
Find common ancestor
Rollback
Replay
```

是同一个核心模型。

---

## 十一、Rollback 到底 rollback 什么？

很多人会误以为：

> 把 checkpoint 改回去就完成了。

不是。

Checkpoint 只是：

```text
processing state
```

真正需要修的是：

```text
data state
```

如果旧分支已经写入：

```text
blocks
transactions
logs
token_transfers
swaps
DWS
ADS
```

这些数据都可能需要处理。

---

## 十二、[关键视角] Checkpoint Rollback ≠ Data Rollback

例如：

```text
checkpoint = 103
```

Reorg 后 common ancestor：

```text
100
```

你把 checkpoint 改成：

```text
100
```

如果数据库里仍然保留：

```text
101A
102A
103A
```

并且它们还被当成 canonical 数据使用，

系统仍然是错的。

所以：

> Rollback processing state and rollback data state are separate responsibilities.

---

## 十三、Old Branch 怎么处理？

有两种常见思路。

### 方案 A：Delete

把旧分支数据物理删除。

例如：

```text
delete blocks 101A–103A
delete derived facts
```

优点：

```text
simple serving semantics
```

缺点：

```text
loss of branch history
harder auditing
```

### 方案 B：Mark Non-canonical

保留旧数据，但加：

```text
is_canonical = false
```

优点：

```text
auditability
reorg analysis
historical branch retention
```

缺点：

```text
all downstream queries must respect canonical filter
```

---

## 十四、哪种方式更适合数据平台？

对于专业链上数据平台，

通常更倾向：

```text
retain + canonical flag
```

尤其 Raw / Bronze 层。

因为你可能以后需要：

```text
audit
debug
reorg analysis
provider comparison
```

但 Serving / DWS 层通常更强调：

```text
only canonical data
```

所以不同 Layer 可以采用不同策略。

---

## 十五、Raw Layer 和 Fact Layer 的职责不同

### Raw Layer

更适合保留：

```text
all observed blocks
canonical + orphan
```

### Fact Layer

可以：

```text
keep all with canonical flag
```

或者：

```text
serve canonical only
```

### DWS / ADS

通常应该反映：

> current canonical business truth.

因此 Reorg 后要修正聚合结果。

---

## 十六、一个 Transfer 的 Reorg 例子

旧链：

```text
Block 101A
Alice → Bob 100 USDC
```

你已经写入：

```text
fact_token_transfer
Alice -100
Bob +100
```

DWS：

```text
Alice outflow +100
Bob inflow +100
```

然后 Reorg。

新链：

```text
Block 101B
Alice → Carol 50 USDC
```

那么原来的：

```text
Alice → Bob 100
```

不再是 canonical fact。

---

## 十七、只插入新链数据会发生什么？

如果你只是：

```text
insert Alice → Carol 50
```

却没有撤销：

```text
Alice → Bob 100
```

那么数据库会出现：

```text
old orphan fact
+
new canonical fact
```

Dashboard 可能显示：

```text
Alice total outflow = 150
```

但 canonical truth 实际是：

```text
50
```

所以：

> Reorg correction must remove or invalidate old branch effects before applying the new branch.

---

## 十八、这和 Additive State 特别冲突

如果 DWS 更新方式是：

```text
current_total += delta
```

Reorg 就很麻烦。

因为旧分支：

```text
+100
```

已经加进去了。

新分支：

```text
+50
```

再加一次就变成：

```text
150
```

所以需要：

```text
reverse old delta
+
apply new delta
```

这正是为什么之前一直强调：

> Set-based / rebuildable state is often safer than blind additive mutation.

---

## 十九、Reorg 对 DWS / ADS 的两种修复方式

### 方式 A：Compensating Update

旧分支影响：

```text
-100
```

撤销后再应用：

```text
+50
```

适合：

```text
small reorg
well-modeled deltas
```

### 方式 B：Recompute affected partition

例如重算：

```text
date + wallet + token
```

受影响分区。

适合：

```text
complex aggregates
multiple dependent metrics
```

通常更容易保证正确。

---

## 二十、Reorg 的 Blast Radius 不只是 Block Range

假设 Reorg 范围：

```text
101–103
```

但影响可能扩散到：

```text
Fact
DWS daily
ADS dashboard
wallet balance cache
risk metrics
API cache
ClickHouse aggregates
Parquet partition
```

所以需要考虑：

> Dependency Blast Radius.

---

## 二十一、Reorg 和 Multi-sink 的问题

例如：

```text
Postgres
ClickHouse
Parquet
```

旧分支已经全部写入。

Reorg 后：

```text
Postgres corrected
ClickHouse not yet corrected
Parquet not rebuilt
```

那么系统内部就出现：

> Consistency Failure.

所以 Reorg Correction 必须有：

```text
sink-by-sink repair state
```

---

## 二十二、为什么各 Sink 应维护独立修复进度？

因为不同 Sink：

```text
storage model different
write latency different
repair mechanism different
```

例如：

```text
Postgres
→ UPSERT / DELETE

ClickHouse
→ replace / mutation / partition rebuild

Parquet
→ rewrite files / partitions
```

所以不能假设：

```text
one correction action fixes all sinks
```

---

## 二十三、Reorg Detection 和 Reorg Repair 是两个阶段

### Detection

回答：

```text
Did canonical history change?
```

### Repair

回答：

```text
How do we make all derived data converge to the new canonical history?
```

检测成功不代表数据已经正确。

---

## 二十四、Reorg Correction Path

可以把系统设计成：

```text
Realtime Path
↓
fresh data

Reorg Detector
↓
correction events

Correction Path
↓
invalidate old branch
replay new branch
repair downstream
```

也就是说：

> Reorg is not an exception outside the architecture; it should be part of the normal correction architecture.

---

## 二十五、为什么 Realtime Path 必须接受 Mutable Truth？

因为靠近 Chain Head：

```text
fresh
```

但：

```text
not fully stable
```

所以实时链路中的事实应理解为：

> currently canonical, but potentially mutable.

而 finalized / stable 层则更接近：

> stable canonical truth.

---

## 二十六、[Perspective] “正确”要带时间前提

在 `t1`：

```text
101A is canonical
```

你的系统处理 101A：

```text
correct at t1
```

在 `t2` Reorg 后：

```text
101A becomes orphan
101B becomes canonical
```

于是：

```text
101A no longer represents current canonical truth
```

这说明：

> Blockchain correctness is time-relative near the chain head.

---

## 二十七、Checkpoint 在 Reorg 中怎么处理？

假设：

```text
checkpoint = 103A
```

common ancestor：

```text
100
```

通常不能继续：

```text
104B
```

直接往前走。

需要先把 processing state 回到安全边界：

```text
checkpoint = 100
```

然后：

```text
replay 101B → ...
```

---

## 二十八、为什么不能保留原 checkpoint？

因为 Checkpoint 的含义是：

> validated completion on the canonical processing path.

如果：

```text
101A–103A
```

已经变成 non-canonical，

那么原来的：

```text
checkpoint = 103A
```

已经不再代表有效的 canonical completion。

---

## 二十九、Reorg Replay 为什么应该复用原 process_block？

你之前已经掌握这个设计。

Realtime：

```text
process_block(block)
```

Backfill：

```text
process_block(block)
```

Reorg Replay：

```text
process_block(block)
```

原因：

> One business logic, multiple input modes.

这样可以减少：

```text
logic drift
bug divergence
repair inconsistency
```

---

## 三十、Reorg Replay 为什么需要 Idempotency？

因为 Correction 过程也可能：

```text
crash
retry
restart
```

例如：

```text
101B
102B
```

已经 replay，

然后系统崩溃。

重启后：

```text
101B
102B
```

可能再次处理。

如果 Sink 不是 Idempotent，

就会产生：

```text
duplicate facts
duplicate aggregates
```

所以：

> Reorg Replay must be replay-safe.

---

## 三十一、Stable Unique Key 仍然重要

对于 Log-derived Fact：

```text
chain_id + tx_hash + log_index
```

仍然是稳定 Source Identity。

但注意：

旧分支和新分支：

```text
tx_hash
```

通常本来就不同。

所以需要同时保存：

```text
block_hash
canonical_status
```

以识别事实属于哪个分支。

---

## 三十二、为什么 Fact 最好保留 block_hash？

如果 Fact 只保存：

```text
block_number
tx_hash
log_index
```

虽然通常能定位事件，

但在 Reorg 调试时，

你仍然很难快速判断：

> This fact came from which exact block branch?

保留：

```text
block_hash
```

会显著提高 Reorg Auditability。

---

## 三十三、Reorg Validation 应验证什么？

修复后不能只说：

```text
Replay finished
```

还需要验证：

```text
old branch no longer canonical
new branch fully present
canonical block continuity
parent_hash chain consistent
fact keys match canonical logs
DWS aggregates reconcile
multi-sink state converged
checkpoint points to canonical branch
```

这才是：

> Reorg Quality Verification.

---

## 三十四、一个 Canonical Invariant

非常重要的系统 Invariant：

```text
For each canonical block N > genesis:
block[N].parent_hash
=
block[N-1].block_hash
```

如果这个关系断裂：

```text
possible gap
possible reorg
possible bad ingest
```

这是一个很强的链结构检查。

---

## 三十五、另一个 Invariant

同一个：

```text
chain_id + block_number
```

在 Serving Canonical View 中，

应该只有：

```text
one canonical block
```

可以写成：

```text
one block height
→ one current canonical block
```

但 Raw 历史层可以保留：

```text
multiple observed blocks at same height
```

只要只有一个：

```text
is_canonical = true
```

---

## 三十六、为什么 “DELETE WHERE block_number >= X” 有风险？

如果你不先找到 Common Ancestor，

直接：

```text
DELETE WHERE block_number >= 100
```

可能：

```text
delete too much
```

或者：

```text
delete too little
```

更严重的是：

如果系统已经处理到更高范围，

你可能破坏无关历史。

所以：

> Reorg repair boundary should come from chain ancestry, not arbitrary block ranges.

---

## 三十七、为什么 Historical Reorg Repair 和 Backfill 很像？

两者都会：

```text
process historical range again
```

但目的不同。

Backfill：

```text
fill / recompute historical data
```

Reorg Replay：

```text
replace invalidated branch with new canonical branch
```

所以：

```text
same processing engine
different control semantics
```

---

## 三十八、[银行系统类比] Reorg 更像冲正，不像重复交易

银行里如果一笔交易后来被判定无效，

不能简单再插一笔新交易就结束。

通常需要：

```text
reverse old effect
apply corrected effect
```

Reorg 也是类似：

```text
invalidate old branch effect
apply new canonical branch effect
```

这比“去重”更接近：

> correction / reversal.

---

## 三十九、Reorg Incident 的标准状态流

可以设计成：

```text
DETECTED
↓
ANCESTOR_FOUND
↓
OLD_BRANCH_INVALIDATED
↓
NEW_BRANCH_REPLAYING
↓
DOWNSTREAM_REBUILDING
↓
RECONCILING
↓
VERIFIED
↓
CLOSED
```

这样方便：

```text
audit
retry
monitoring
incident recovery
```

---

## 四十、本课核心心智模型

可以压缩成：

```text
Chain Head Changes
↓
Parent Hash Mismatch
↓
Find Common Ancestor
↓
Invalidate Old Branch
↓
Rollback Processing State
↓
Replay New Canonical Branch
↓
Repair Downstream
↓
Reconcile
↓
Resume
```

最核心的两个概念是：

```text
Canonical Truth Can Change Near Head
```

以及：

```text
Rollback State + Replay Data
```

---

## 本课核心结论

> Reorg means previously accepted data may later become non-canonical.

> Block Number is a position; Block Hash is block identity.

> Parent Hash Mismatch is a key Reorg Detection signal.

> Common Ancestor defines the repair boundary.

> Checkpoint Rollback and Data Rollback are different responsibilities.

> Reorg correction must invalidate old-branch effects before applying the new canonical branch.

> Raw layers often benefit from retaining orphan history with canonical flags.

> DWS / ADS must reflect the current canonical truth, so they may need recomputation after Reorg.

> Reorg correction is a multi-layer and multi-sink repair problem.

> Reorg Replay should reuse the same deterministic `process_block` logic as realtime and backfill.

> Reorg Replay must be idempotent and replay-safe.

> After correction, Reconciliation and Invariant Checks are required before declaring recovery complete.

---

## 理解检查

### 问题一

旧链：

```text
100
↓
101A
↓
102A
```

新链：

```text
100
↓
101B
↓
102B
↓
103B
```

请回答：

1. Common Ancestor 是哪个 block？
2. 哪些 block 属于 Old Branch，应该被 invalidated / rolled back？
3. 哪些 block 属于 New Canonical Branch，需要 replay？

### 问题二

你的系统：

```text
checkpoint = 102A
```

数据库里已经有：

```text
101A
102A
```

对应的 Transfer Fact 和 DWS Aggregate。

现在发生 Reorg，Common Ancestor 是：

```text
100
```

请回答：

1. 为什么不能只把 Checkpoint 改回 100 就结束？
2. 除了 Checkpoint，还需要修哪些 Data State？
3. 为什么新分支 Replay 之前，要先 invalidated / rollback Old Branch？

### 问题三

你的 Raw Layer 保留：

```text
101A
101B
```

它们：

```text
block_number 都是 101
block_hash 不同
```

请回答：

1. 为什么 `block_number` 不能作为稳定区块身份？
2. Serving Canonical View 中，同一个 `chain_id + block_number` 应该允许几个 `is_canonical = true`？
3. 为什么 Fact Table 里保留 `block_hash` 对 Reorg Audit 很重要？

## 用户回答（问题一）

问题1：

1.Common Ancestor 的block是100

2.101A、102A block 属于 Old Branch，应该被 invalidated / rolled back

3.101B 、 102B 、 103B  block 属于 New Canonical Branch，需要 replay

## 老师判断 / 校准（问题一）

通过。

1. **Common Ancestor = 100**：正确。
2. **Old Branch = 101A、102A**：正确，这些 block 在 Reorg 后不再属于 Canonical Chain，应被 invalidated / rolled back。
3. **New Canonical Branch = 101B、102B、103B**：正确，需要从 Common Ancestor 之后重新 Replay。

核心边界是：

```text
common ancestor = 100

old branch:
101A → 102A

new canonical branch:
101B → 102B → 103B
```

因此 Repair Boundary 从：

```text
common_ancestor + 1
```

也就是 block 101 开始。

## 用户回答（问题二）

问题2：

1.只修改 Checkpoint 并不能修复全部的问题数据。像中表中已经存在的orphans数据，无法通过只修改 Checkpoint 来修复。

2. fact 、dws、 ads等 Data State&#x20;
3. 因为当前表中已经存在Old Branch，如果不处理，会影响downstream 的数据。

## 老师判断 / 校准（问题二）

通过。

1. **只回退 Checkpoint 不够**：正确。Checkpoint 只代表 Processing State；数据库中已经写入的 Old Branch / Orphan Data 仍然存在，必须单独修复 Data State。
2. **需要修复的 Data State**：正确。包括 Fact、DWS、ADS，以及其他受 Old Branch 影响的下游状态。
3. **为什么要先处理 Old Branch**：正确。需要再明确一步：不仅是“Old Branch 还存在”，更关键的是它的业务影响已经传播到 Downstream。

正确顺序应是：

```text
Invalidate / Rollback Old Branch
↓
Remove / Reverse Old Branch Effects
↓
Replay New Canonical Branch
↓
Rebuild / Reconcile Downstream
```

如果直接 Replay New Branch 而不撤销 Old Branch：

```text
old orphan effect
+
new canonical effect
```

就会同时存在，造成错误聚合。

核心结论：

> Checkpoint Rollback ≠ Data Rollback.

> Reorg correction must invalidate old-branch effects before applying the new canonical branch.

## 用户回答（问题三）

问题3：

1.`block_number`   是block pos的identity，block_hash 才是block 的identity。

2.is_canonical = true 只能有一个。

3.block_hash 是block 的identity，在fact table中保留这个字段，便于定位transfer数据是属于哪个block

## 老师判断 / 校准（问题三）

通过，但第 1 小问术语需要精确一点。

1. **为什么 block_number 不能作为稳定区块身份**：核心理解正确。更准确地说：

```text
block_number
→ block height / position

block_hash
→ block identity
```

不应把 block_number 称为 position identity，因为同一个 block height 在 Reorg 前后可以对应不同 block。
2. **Serving Canonical View 中 is_canonical = true 的数量**：正确。同一个 `chain_id + block_number` 当前只能有一个 canonical block。
3. **Fact Table 为什么保留 block_hash**：正确。它可以明确指出该 Fact 来自哪一个具体 Block Branch，提高 Reorg Audit / Debug / Repair 的可追溯性。

核心结论：

> Block Number identifies position; Block Hash identifies block identity.

> One block height should have only one current canonical block.

## 结课判定

Module 12 第 7 课理解检查全部通过，正式完成。

已经能够：
- 区分 Canonical Block 与 Orphan / Non-canonical Block。
- 理解 Reorg 会让“曾经正确”的历史事实后来变为 non-canonical。
- 理解 Block Number 表示 height / position，而 Block Hash 表示 block identity。
- 使用 Parent Hash Mismatch 识别可能的 Reorg。
- 使用 Common Ancestor 定义 Reorg Repair Boundary。
- 区分 Checkpoint Rollback 与 Data Rollback。
- 理解 Reorg Repair 必须先撤销 / 失效 Old Branch，再 Replay New Canonical Branch。
- 理解 Raw Layer、Fact、DWS、ADS 在 Reorg 中承担不同修复职责。
- 理解 Reorg Blast Radius 可以扩散到多个 Derived Layer 和多个 Sink。
- 理解 Reorg Replay 应复用 deterministic `process_block`。
- 理解 Reorg Correction 也必须满足 Idempotency / Replay-safe。
- 理解 Fact 保留 `block_hash` 与 `canonical_status` 的 Reorg Audit 价值。
- 理解修复后仍需执行 Reconciliation / Invariant Check，验证 Canonical Correctness。