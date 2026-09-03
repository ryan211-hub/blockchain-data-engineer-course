# 第7课｜Reorg：Indexer 如何处理 canonical / orphaned 数据与回滚重放

## Lesson Contract
【Blockchain Data Engineer 视角】
- 所属 Module：Module 7 — Indexer
- 本课核心问题：Indexer 已经把某个 Block 写入数据库以后，如果这条 Block 后来因为链重组不再属于 canonical chain，应该怎么办？
- 学完以后应能解释：Reorg 是什么；为什么 block_number 不是稳定身份；canonical / orphaned Block 的含义；Indexer 如何检测 Reorg；为什么 Checkpoint 最好同时保存 block_number 与 block_hash；Reorg 后如何 rollback / mark orphaned；为什么之后还要 replay 新 canonical blocks；以及 Idempotency、Checkpoint、Backfill 与 Reorg 如何形成完整恢复模型。
- 本课暂不展开：Ethereum 共识算法细节、Validator fork choice 内部算法、大规模深度 Reorg 修复框架、Finality 协议实现细节、Warehouse 下游完整补偿机制。

## 一、block_number 是位置，block_hash 是身份
假设 Indexer 先看到：
```text
Block Number = 100
Block Hash   = AAA
```
后来再次查询高度 100，得到：
```text
Block Hash = BBB
```
这并不是 AAA 这个具体 Block “变成”了 BBB，而是高度 100 当前对应的 canonical Block 从 AAA 切换成 BBB。
因此：
```text
block_number = Position
block_hash   = Identity
```

## 二、Reorg 是什么
Reorg = 节点原先认为 canonical 的一段链，后来被另一条链分支替代。
例如旧链：
```text
98 → 99 → 100A → 101A → 102A
```
后来 canonical chain 变成：
```text
98 → 99 → 100B → 101B → 102B → 103B
```
原来的 100A / 101A / 102A 不再属于 canonical chain。

## 三、canonical 与 orphaned
【Indexer / Data Model 视角】
- Canonical Block：当前属于主链的 Block。
- Orphaned Block：曾被观察、甚至被处理过，但后来不再属于 canonical chain 的 Block。

所以：
```text
Canonical = 当前有效链事实
Orphaned  = 历史上曾出现，但现在不属于主链
```

## 四、为什么 Checkpoint 最好保存 block_hash
仅保存：
```text
last_processed_block = 102
```
只能说明处理到了高度 102，却不知道处理的是 102A 还是 102B。
更稳健的 Checkpoint：
```text
checkpoint_block_number = 102
checkpoint_block_hash   = hash(102A)
```
重启时查询当前 canonical Block 102，如果得到 102B，hash 不一致即可识别 Checkpoint 已经过期。
所以：
```text
Checkpoint = Position + Identity
```

## 五、Indexer 如何发现 Reorg
Block Header 有：
```text
block_hash
parent_hash
```
正常情况下：
```text
current_block.parent_hash = previous_block.block_hash
```
例如数据库最新 Block 102：
```text
hash = AAA102
```
新收到 Block 103：
```text
parent_hash = BBB102
```
因为 BBB102 != AAA102，说明链连续性被打断，是典型 Reorg Signal。

## 六、发现 Reorg 后不能直接继续写新 Block
如果数据库是：
```text
100A
101A
102A
```
而新 canonical chain 是：
```text
100B
101B
102B
103B
```
直接插入 103B 会得到一条不存在的混合链：
```text
100A → 101A → 102A → 103B
```
因此必须先找到 Common Ancestor。

## 七、Common Ancestor
旧链：
```text
98 → 99 → 100A → 101A → 102A
```
新链：
```text
98 → 99 → 100B → 101B → 102B → 103B
```
最后一个共同 Block 是 99，所以：
```text
Common Ancestor = 99
```

## 八、完整 Reorg Recovery 流程
```text
Detect Reorg
↓
Find Common Ancestor
↓
Rollback Old Branch
↓
Replay New Canonical Branch
↓
Advance Checkpoint
```
这是本课核心流程。

## 九、Rollback Old Branch
旧分支 100A / 101A / 102A 已不再 canonical，它们下面的 Transactions、Receipts、Logs、Decoded Events、Normalized Facts 都不能继续被当作 canonical facts。
常见两种处理：
1. DELETE orphaned 数据。
2. 保留历史数据，但设置 canonical = false。

生产系统经常倾向第二种，因为保留 orphaned history 有利于审计、分析 Reorg、解释“交易为什么消失”、验证 Indexer 恢复是否正确。

## 十、Raw Layer 与 canonical flag
一种常见 blocks 表：
```text
block_hash | block_number | canonical
100A       | 100          | false
100B       | 100          | true
```
这体现：
```text
Preserve History
+
Filter Canonical Truth
```

## 十一、Fact 表也要有 Reorg 策略
如果 100A 中有一条 USDC Transfer，已经写入 token_transfers，后来 100A orphaned，这条 Transfer 就不能继续算作 canonical 链事实。
常见策略：
```text
canonical = false
```
或者删除 orphaned facts。

## 十二、Source Lineage 为什么重要
Fact 表应保留：
```text
chain_id
block_number
block_hash
tx_hash
log_index
```
这些字段不仅方便查询，也是 Rollback Lineage。Reorg 时可以按 orphaned block_hash 找回所有需要失效的数据。

## 十三、Reorg 与普通 Retry 的区别
Retry：
```text
Same Blockchain Fact
↓
Same Identity
↓
Idempotent Write
```
Reorg：
```text
Old Blockchain Fact 被撤销
New Blockchain Fact 替代
```
所以：
> Reorg 不是 Duplicate Processing，而是 Canonical Truth Change。

## 十四、Replay New Canonical Branch
回退到 Common Ancestor 99 后，从 100B 开始重新：
```text
Fetch
↓
Parse
↓
Decode
↓
Normalize
↓
Idempotent Write
```
直到新的 canonical head。

## 十五、为什么 Reorg 后仍需要 Idempotency
Reorg 决定“哪些事实有效”；Idempotency 保证“重复执行不会写坏”。二者互不替代。
Replay 中可能有新数据、相同数据、重叠处理，因此 Writer 仍应 Retry-safe。

## 十六、Checkpoint 在 Reorg 时可以逻辑回退
原 Checkpoint：
```text
102A
```
Common Ancestor：
```text
99
```
则 Checkpoint 应先逻辑回退到 99，再 Replay 100B → 103B，最后推进到 103B。
正常情况下 Checkpoint 单调增加，但 Reorg 时可以回退。

## 十七、Backfill 与 Reorg 的区别
```text
Backfill = 补历史处理结果，链事实本身未变化
Reorg    = 修正历史 canonical 链事实
```
两者不能混淆。

## 十八、Block / Transaction / Log / Fact 的典型 provenance 字段
Blocks：
```text
chain_id
block_number
block_hash
parent_hash
block_time
canonical
```
Block 主身份适合 `(chain_id, block_hash)`，高度用于查询。
Transactions / Logs / Facts 同样应携带 block_number、block_hash、canonical 等 lineage 字段，以支持 Reorg rollback。

## 十九、具体 Block immutable，不等于高度的 canonical assignment immutable
【具体 Block 对象视角】某个 block_hash 对应的 Block 内容是 immutable。
【高度 / canonical chain 视角】某个 block_number 在 Finalized 前对应哪个 canonical Block 可能变化。

## 二十、Finality 与 Confirmation Delay
越接近 Chain Head，数据越新，但 Reorg 风险越高；等待更多确认，稳定性更高，但延迟也更高。
```text
Near-head data = fast but reversible
Finalized data = slower but stronger
```
不同下游业务可以采用不同 confirmation policy。

## 二十一、Reorg Recovery 的关键不是只“重新跑”
真正目标是：
```text
Identify Wrong Branch
↓
Invalidate Wrong Facts
↓
Restore Correct Branch
```
只 Replay 新 branch 而不处理旧 orphaned facts，会让数据库同时存在旧事实和新事实，造成统计污染。

## 二十二、必须同时 Rollback + Replay
```text
Rollback = 撤销旧 canonical branch 的影响
Replay   = 处理新的 canonical branch
```
只做一个都不完整。

## 二十三、与前几课连接
```text
Checkpoint
↓
知道处理到哪里

Idempotency
↓
重复处理不会写坏

Backfill
↓
可以补历史区间

Reorg
↓
链事实变化时可以回滚 + 重放
```
这四项组合起来，Indexer 才从 Demo 变成 Production Pipeline。

## 二十四、最终状态机
```text
Read Checkpoint
↓
Fetch Next Block
↓
Parent Hash Match?
├── Yes
│   ↓
│ Process
│ ↓
│ Idempotent Write
│ ↓
│ Advance Checkpoint
│
└── No
    ↓
    Reorg Detected
    ↓
    Find Common Ancestor
    ↓
    Rollback / Mark Orphaned
    ↓
    Move Checkpoint Back
    ↓
    Replay Canonical Blocks
```

## 本课核心模型
> Reorg = Canonical Truth Change。

```text
block_number 相同
↓
block_hash 不同
↓
Reorg Detected
↓
Find Common Ancestor
↓
Old Branch → Orphaned
↓
Rollback Old Facts
↓
Replay New Canonical Branch
↓
Checkpoint 更新到新 canonical head
```

## 理解检查与校准
### 问题 1
Indexer 数据库当前记录 Block 100 / block_hash = AAA / canonical = true。后来 Node 查询同一个 block_number = 100 得到 block_hash = BBB。这说明什么？为什么 block_number 不能作为 Block 的稳定 Identity？

**用户回答：**
> 问题一：这说明了在 block number 等于 100 的位置，block 发生了改变，不再是原来那个 block 了。block number 没变，但是 block hash 变化了，所以 block number 不能作为 block 的稳定 identity

**老师判断与校准：✅ 通过。**
核心判断正确。更精确地说，AAA 这个具体 Block 本身没有变成 BBB；变化的是“高度 100 当前对应哪个 canonical Block”。因此 block_number 是 Position，block_hash 才是具体 Block 的 Identity。

### 问题 2
旧链：98 → 99 → 100A → 101A → 102A；新 canonical chain：98 → 99 → 100B → 101B → 102B → 103B。Common Ancestor 是谁？哪些旧 Block 变 orphaned？从哪里开始 replay？

**用户回答：**
> 1：Common ancestor 是 位置在高度 99 的 block
> 2：100A、101A、102A 这三个 block 应该变成 orphan。
> 3：Indexer 应该从 100 的高度开始 replay 新 canonical branch。

**老师判断与校准：✅ 通过。**
三个判断都正确。Checkpoint 视角下，应先把 Checkpoint 逻辑回退到 99，然后从高度 100 开始 Replay 100B → 103B。

### 问题 3
为什么 Reorg 不能只 Replay New Canonical Blocks，而必须 Rollback Old Branch + Replay New Branch？如果旧 branch 的 token_transfers 仍 canonical=true，会怎样？

**用户回答：**
> 问题 3：Reorg 处理必须同时做 Rollback old branch 加上 Replay new branch，如果不先 rollback old branch，直接加上 replay new branch 的话，那旧的 block 数据canonical = true，那后续的数据都会产生错误。

**老师判断与校准：✅ 通过。**
核心判断正确。具体来说，如果旧 branch 的 facts 仍 canonical=true，同时又把新 branch facts 写入，那么 Dashboard、Volume、Transfer Count 等统计会同时计算旧分支与新分支数据，造成重复计数、虚增交易量、错误余额或行为分析。正确流程必须先让旧分支失效，再 Replay 新 canonical branch。

## 本课重点总结
1. block_number 是高度/位置，block_hash 是具体 Block 身份。
2. Reorg = Canonical Truth Change，不是普通 Retry。
3. Reorg Detection 依赖 parent_hash / block_hash 连续性与 Checkpoint Identity。
4. Reorg Recovery 核心流程：Detect → Find Common Ancestor → Rollback / Orphan → Replay → Advance Checkpoint。
5. Raw / Fact 层保留 block_hash 与 canonical 能显著提升可审计和可恢复能力。
6. Reorg 必须同时处理旧分支失效与新分支重放，只 Replay 新分支是不完整的。
7. Checkpoint + Idempotency + Backfill + Reorg 共同构成生产级 Indexer 的可靠性基础。