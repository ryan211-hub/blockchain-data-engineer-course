# 第6课｜Backfill：Indexer 如何安全补历史数据，而不是每次从 Genesis 重跑

## Lesson Contract
【Blockchain Data Engineer 视角】
- 所属 Module：Module 7 — Indexer
- 本课核心问题：当 Indexer 缺失一段历史数据，或者新增一个 Decoder / Fact 表时，怎样补历史数据，而不是从 Genesis 把整条链重新跑一遍？
- 学完以后应能解释：Backfill 是什么；Backfill 和正常 Real-time Sync 有什么区别；为什么 Backfill 通常是一个明确的历史区间任务；为什么不能为了补一段数据就重置主 Checkpoint；Checkpoint / Cursor 在 Backfill 中怎么用；Idempotency 为什么让 Backfill 可以安全重跑；Historical Backfill 和 Real-time Indexing 怎样共存。
- 本课暂不展开：Reorg 完整回滚、分布式任务调度框架、Kafka、大规模并行调优、Warehouse 分区策略。

## 一、什么情况下会需要 Backfill？
假设 Indexer 已经稳定运行到 Block 21,000,000，数据库里已经有 18,000,000 → 21,000,000 这一段数据。后来发现 18,500,000 → 18,600,000 有一段数据因为历史故障没有完整处理。

最笨的做法是把 Checkpoint 重置到 0，从 Genesis 重新跑到 21,000,000。这显然很浪费，因为真正缺失的只有 100,001 Blocks。

所以更合理的是 Backfill。

## 二、Backfill 是什么？
Backfill 可以压缩成一句话：**对指定历史区间重新补处理。**

例如：
```text
start_block = 18,500,000
end_block   = 18,600,000
```
然后：
```text
for block in range(start_block, end_block):
    fetch
    parse
    decode
    write
```
所以：
```text
Backfill = Historical Range Processing
```

## 三、Backfill 和正常同步有什么区别？
【Indexer 运行模式视角】

正常同步：
```text
Checkpoint = N
↓
N+1
↓
N+2
↓
追 Chain Head
```
目标是持续跟上最新链。

Backfill：
```text
Start = A
End   = B
↓
处理固定历史区间
```
目标是把过去缺失或需要重新生成的数据补齐。

最直观区别：
```text
Real-time Sync = Open-ended
Backfill       = Bounded Range
```

## 四、为什么 Backfill 不能直接改主 Checkpoint？
假设当前实时 Indexer：
```text
Checkpoint = 21,000,000
```
现在要补：
```text
18,500,000 → 18,600,000
```
如果把主 Checkpoint 改成 18,500,000，实时 Indexer 会以为“我只处理到了 18,500,000”，然后重新处理 18,500,001 → 21,000,000，把大量已经处理过的 Block 再走一遍。

即使 Idempotency 能防止写坏，RPC Cost、CPU Decode、Database Write 和运行时间都会浪费。

所以 Backfill 不应该污染主实时 Checkpoint。

## 五、Real-time Checkpoint 和 Backfill Cursor 应该分开
可以有：
```text
realtime_checkpoint = 21,000,000
```
同时 Backfill 自己有：
```text
backfill_start  = 18,500,000
backfill_end    = 18,600,000
backfill_cursor = 18,530,000
```
于是：
```text
Real-time 继续追 21,000,001...
Backfill  继续补 18,530,001...
```
两套进度互不干扰。

## 六、Backfill 自己也需要 Checkpoint / Cursor
如果补 10,000,000 → 20,000,000，一千万个 Block，跑到 15,400,000 程序 Crash，没有 Backfill Cursor 就不知道从哪里恢复。

所以大型 Backfill 同样需要 Durable Progress，例如：
```text
backfill_job_id = 20260903_dex_v2
start_block = 10,000,000
end_block = 20,000,000
last_processed_block = 15,400,000
```
重启后从 15,400,001 继续。

## 七、Backfill Job 可以看成一个有边界的 Indexer
正常 Indexer：
```text
Start
↓
不断向前
↓
Latest
↓
Latest+1
↓
永不结束
```

Backfill：
```text
Start
↓
不断向前
↓
End
↓
Done
```

所以 Backfill 本质上就是一个 bounded indexer job。它同样需要 Cursor、Retry、Idempotency、Error Handling，只是有明确 end_block。

## 八、Backfill 不只是“修缺失数据”
还有一种常见场景：原来没有 dex_swaps Fact 表，后来新开发了 Uniswap Decoder。从今天开始实时 Decode 很容易，但过去三年的历史也需要补出来。

如果已经保留 Historical Raw Logs：
```text
Historical Raw Logs
↓
Backfill
↓
New Decoder
↓
dex_swaps
```

这就是为什么 Raw Layer 很重要。

## 九、有 Raw Layer 和没有 Raw Layer，Backfill 成本完全不同
情况 A：Raw Logs 已保存。
```text
Raw Logs
↓
Decoder
↓
Normalizer
↓
New Fact
```
无需重新请求 RPC。

情况 B：没有 Raw Layer。
```text
RPC Provider
↓
重新 fetch historical Block / Receipt / Log
↓
Decoder
↓
Database
```
会产生 RPC Cost、Rate Limit、Provider Dependency、历史查询时间。

所以：**Raw Layer 是 Backfill 的基础资产。** 这和第 2 课的“Raw = Rebuild Capability”一致。

## 十、Backfill 和 Reprocessing / Replay 的区别
Backfill 更强调“把过去某个区间的数据补出来”；Reprocessing / Replay 更强调“已经处理过的数据重新处理一次”。

缺数据：
```text
Block 100 → 200 没有 dex_swaps
```
这是 Backfill。

Decoder 修 Bug：
```text
Block 100 → 200 已经有 dex_swaps
但 amount 算错了
```
重新跑：
```text
Raw → Fixed Decoder → UPSERT
```
更像 Reprocessing / Replay。

工程上它们可以复用同一个框架。

## 十一、Idempotency 为什么对 Backfill 特别重要？
Backfill 18,000,000 → 19,000,000，跑到 18,700,000 Crash。如果恢复时不完全确定最后一个 Block 是否已经写完，可能从略早位置重跑，产生重叠。

如果写入不是幂等，会产生重复数据；如果有：
```text
Stable Identity
+
Unique Constraint
+
UPSERT / DO NOTHING
```
那么 Retry / overlap 都可以保持安全。

前几课形成连续关系：
```text
Checkpoint
↓
Idempotency
↓
Backfill
```

## 十二、Backfill 最典型的问题：和实时任务同时写同一张表
实时 Indexer 可能正在写 Block 21,000,001，Backfill 正在写 Block 18,500,001。这本身没有问题，因为是不同 Block。

如果 Backfill 区间与实时已经处理过的区间重叠，例如 20,900,000 → 21,000,000，那么两个任务可能碰到同一条 Fact。这时 Idempotency 是安全底座。

## 十三、实时和 Backfill 可以共用同一套 Writer
成熟设计通常不会为 Realtime 和 Backfill 分别实现两套口径不同的 Writer，而是：
```text
Realtime Source
        \
         → Parser → Decoder → Normalizer → Idempotent Writer
        /
Backfill Source
```
数据来源模式不同，但后面的处理与写入逻辑尽量复用，减少两套逻辑口径不一致。

## 十四、实时和 Backfill 真正不同的是调度方式
【Pipeline 视角】

核心 Transform 可能完全相同：
```text
Raw Block
↓
Parser
↓
Decoder
↓
Normalizer
↓
Writer
```

区别主要在入口：
```text
Realtime: next block after realtime checkpoint
Backfill: next block within [start_block, end_block]
```
可以理解为：
```text
Same Processing Engine
Different Scheduling Mode
```

## 十五、一个简单架构
```text
                   ┌─────────────────┐
Chain Head ───────→│ Realtime Cursor │
                   └────────┬────────┘
                            │
                            ▼
RPC / Raw ─────────────→ Processing Pipeline
                            ▲
                            │
                   ┌────────┴────────┐
Historical Range → │ Backfill Cursor │
                   └─────────────────┘

Processing Pipeline
↓
Parser
↓
Decoder
↓
Normalizer
↓
Idempotent Writer
↓
Database
```

## 十六、Backfill 应该有明确 Start / End
生产系统应该明确：
```text
chain_id = 1
start_block = 18,000,000
end_block = 19,000,000
pipeline = dex_swap_v2
```
这样才能回答：补什么、补哪条链、补哪个区间、用哪个处理逻辑、处理到哪、是否完成。

所以 Backfill 本质上应该是 Explicit Range Job。

## 十七、Backfill Job 表可以怎么设计？
逻辑模型：
```text
backfill_jobs
```
字段例如：
```text
job_id
chain_id
pipeline_name
start_block
end_block
last_processed_block
status
created_at
updated_at
```

例：
```text
job_id = dex_v2_20260903
chain_id = 1
pipeline_name = dex_swap
start_block = 18,000,000
end_block = 19,000,000
last_processed_block = 18,420,000
status = RUNNING
```
这是一个可恢复的历史区间任务。

## 十八、Status 有什么用？
例如：
```text
PENDING
RUNNING
FAILED
COMPLETED
```
失败时：
```text
status = FAILED
last_processed_block = 18,420,000
```
修复后 resume，而不是重新从 start_block 开始。

## 十九、Backfill 是否一定要顺序跑？
概念上 A → A+1 → A+2 最简单。但历史数据已经存在，所以理论上可以切 Chunk 并行，例如：
```text
Chunk 1: 18.0M → 18.1M
Chunk 2: 18.1M → 18.2M
Chunk 3: 18.2M → 18.3M
```
并行调度属于后续性能优化，本课只要求掌握：Backfill 是有边界、可恢复、可重跑的历史任务。

## 二十、为什么 Chunk 很自然？
一百万 Blocks 可以切成若干固定区间，每个 Chunk 都有 PENDING / RUNNING / DONE 状态。这样更容易重试、并行、观察进度和定位失败区间。

## 二十一、Backfill 是否应该更新 realtime checkpoint？
一般不应该，因为两者语义不同：
- Realtime Checkpoint：最新链同步可靠推进到哪里。
- Backfill Cursor：某个历史任务补到哪里。

例如：
```text
realtime_checkpoint = 21,000,000

backfill_job_1:
18,000,000 → 19,000,000
cursor = 18,500,000
```
两个值不能混在一起。

## 二十二、银行数据平台类比
如果每天实时处理业务流水，主批次已经到 2026-09-03，后来发现 2026-06-01 → 2026-06-10 某个历史指标算错了，不会把主生产批次日期重置回六月，而是单独启动历史补数任务。

传统数仓按 business_date，区块链天然按 block_number 推进，本质相同。

## 二十三、Backfill 最重要的两个边界
```text
Progress Isolation
+
Write Idempotency
```

Progress Isolation：
```text
Realtime Checkpoint ≠ Backfill Cursor
```

Write Idempotency：
```text
即使区间重叠或任务重跑
数据库仍然正确
```

## 二十四、如果 Backfill 写的是新表
例如新建 dex_swaps_v2：
```text
Historical Raw Logs
↓
Backfill
↓
dex_swaps_v2
```
冲突较少。

如果写现有 token_transfers，就必须考虑 Unique Key、UPSERT、DO NOTHING，否则历史重跑可能制造重复。

## 二十五、如果 Decoder 修 Bug
旧结果 amount = 1000，正确结果 amount = 100。此时 ON CONFLICT DO NOTHING 不够，因为旧数据不会被修正，应使用 UPSERT 把同一 logical fact 更新为新结果。

所以 Backfill Strategy 要根据目的决定：补缺失时 DO NOTHING 可能够；修正派生结果时 UPSERT 更合适。

## 二十六、一个 Backfill 执行流程
```text
Create Backfill Job
↓
start = A
end = B
cursor = A - 1
↓
Read cursor
↓
Fetch next Block
↓
Parse
↓
Decode
↓
Idempotent Write
↓
Advance Backfill Cursor
↓
cursor == end ?
├── No → Continue
└── Yes → COMPLETED
```
它和 Checkpoint Loop 几乎一样，只是多了一个 end boundary。

## 二十七、为什么 Backfill 不能只靠 SQL 找“缺失 block_number”？
有时候 SQL 找缺口可以解决物理缺行，但不能解决旧 Decoder 逻辑错误、新字段需要重算、新 Fact 表上线、ABI 更新后重新 Decode。

所以更广义地说：Backfill 是“用当前处理逻辑重新作用于指定历史区间”。

## 二十八、Backfill 和历史 RPC 成本
没有 Raw Layer 时，Backfill 大量 Blocks 意味着大量 eth_getBlockByNumber、eth_getTransactionReceipt、eth_getLogs 等 RPC 请求。前面已经学过 Historical Range Scan 很昂贵。

所以数据公司重视 Raw Data Retention，因为 Raw Layer 可以把未来大量 Backfill 从“重新买历史 RPC”变成“内部重放数据”。

## 二十九、Historical Raw Data 是 Reprocessing Asset
历史 Raw 数据的价值不仅是供用户查询，还包括重新 Decode、补新协议、修 Bug、生成新指标、重建 Fact。

因此 Historical Raw Data 是 Reprocessing Asset，而不只是查询存档。

## 三十、本课核心模型
```text
Realtime Indexer
Checkpoint → Chain Head
```
同时：
```text
Backfill Job
Start → Cursor → End
```
两者共用：
```text
Parser
Decoder
Normalizer
Idempotent Writer
```
最终：
```text
Realtime Progress ≠ Backfill Progress
```
但 Realtime Data + Historical Backfill Data 可以安全进入同一数据库，依赖 Stable Identity + Idempotent Write。

## 理解检查与校准
### 问题 1
当前 realtime_checkpoint = 21,000,000，现在发现 18,500,000 → 18,600,000 这一段 dex_swaps 数据缺失。为什么不应该把 realtime_checkpoint 直接改成 18,500,000？更合理的设计是什么？

**用户回答：**
> 问题一：
> 如果直接修改 real-time checkpoint 的话，就会使得 real-time index 从 1850 万高度的 block 开始重跑，一直跑到最新节点的最新 block。
>
> 原来只需处理 10 万个 block，现在却变成了处理上百万个 block，导致工作量无谓地增加，浪费node 资源。
>
> 所以更合理的设计是建立一个专门用于 backfill 的 index 实例，这样就不会影响到 real-time index 的工作。

**老师判断与校准：✅ 通过。**
核心判断正确。直接重置实时 Checkpoint 会让实时 Indexer 误认为自己只可靠处理到 18,500,000，从而重新处理到链头，浪费 RPC、Node、Decode 和 DB 资源。术语上进一步精确：不一定必须是完全独立的 Indexer 实例，也可以是同一个 Indexer 系统内独立的 Backfill Job / Worker。关键不是物理进程是否分开，而是 Progress State 必须隔离：Realtime Checkpoint ≠ Backfill Cursor。

### 问题 2
一个 Backfill Job：start_block = 10,000,000，end_block = 20,000,000，跑到 15,400,000 时程序 Crash。为什么 Backfill 自己也需要持久化 Cursor / Checkpoint？重启后应该从哪里继续？

**用户回答：**
> 问题2： Backfill Job 有了自己持久化的 Checkpoint，那么当 Job crash 的时候，重启之后，它就可以接着 Checkpoint 的位置继续它的工作。
>
> 如果 Backfill Job 自己没有持久化的 Checkpoint，那当它 crash 之后，它只好从 start_block 从头开始了

**老师判断与校准：✅ 通过。**
理解正确。Backfill 虽然是历史任务，但只要规模大、运行时间长，同样会遇到 Crash、Retry、机器重启、Worker 被替换，所以也需要 Durable Progress。当前 last_processed_block = 15,400,000 时，重启应从 15,400,001 继续，而不是重新从 10,000,000 开始。

### 问题 3
为什么 Stable Identity (chain_id, tx_hash, log_index) + Unique Constraint + Idempotent Write 能让 Real-time Indexer 和 Backfill 同时写同一张 token_transfers 表而不容易产生重复数据？即使 Backfill 区间和实时已处理区间重叠，为什么数据库仍能保持正确？

**用户回答：**
> 问题 3：
> 从 Indexer 方面来说，Real-time Indexer 和 Backfill 其实使用的是同一套 Index 数据处理逻辑。换句话说，它们在处理同一个 Block 的时候，处理出的结果是一样的。
>
> 从 token_transfers 表来说，这个表设计了 Stable Identity、Unique Constraint 和 Idempotent Write。符合这三个特性的话，不会产生重复的数据。
>
> 即使 Backfill 区间和实时已经处理过的区间发生重叠，Backfill 也只是重新再处理一遍，而不会将重复的数据插入进数据库

**老师判断与校准：✅ 通过。**
两个层次都正确。处理层面应尽量复用同一套确定性 Pipeline，使相同 Raw Input 在相同 Decoder / Normalizer 逻辑下产生相同 logical fact；写入层面，Stable Identity 把同一条链上 Transfer 映射到相同 (chain_id, tx_hash, log_index)，Unique Constraint 识别冲突，Idempotent Write 用 DO NOTHING 或 UPSERT 将“重复处理”收敛为同一行。因此区间重叠造成的是 Duplicate Processing，而不是 Duplicate Data。

## 本课重点总结
1. Backfill = 对明确历史区间重新处理，是 Bounded Range Job。
2. Real-time Checkpoint 与 Backfill Cursor 必须隔离，不能为了补历史而重置主实时进度。
3. 大型 Backfill 自己也需要 Durable Cursor / Checkpoint，才能 Crash 后恢复。
4. Realtime 与 Backfill 可以复用同一套 Parser / Decoder / Normalizer / Idempotent Writer，主要区别在调度方式。
5. Stable Identity + Unique Constraint + Idempotent Write 让区间重叠和 Retry 不容易产生重复数据。
6. Raw Layer 能显著降低 Backfill 成本，是 Reprocessing Asset。
7. 补缺失数据时 DO NOTHING 可能足够；修复历史派生结果时通常需要 UPSERT。
8. 核心模型：Realtime Progress ≠ Backfill Progress，但两者可以安全写入同一事实层。