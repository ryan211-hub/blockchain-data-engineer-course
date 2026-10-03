# Module 12 第 8 课｜Historical Repair：Backfill、Replay 与多 Sink 修复

## Lesson Contract

所属 Module：Module 12 — 数据质量

本课核心问题：

> When we discover that historical blockchain data is wrong, missing, or inconsistent, how do we repair it safely without damaging the realtime pipeline?

学完以后，你应该能够解释并设计：

- Historical Repair 为什么不能等同于“重新跑一次 Job”。
- Retry、Replay、Backfill、Recompute 的区别。
- 为什么 Historical Repair 不应该随意回退 Realtime Checkpoint。
- 如何定义 Repair Scope / Blast Radius。
- 为什么 Repair Job 需要自己的 Cursor / Checkpoint / Status。
- 如何选择 Raw Layer、Parquet、RPC 作为 Repair Source。
- 为什么同一套 `process_block` / transform logic 应复用。
- 为什么 Historical Repair 必须是 Idempotent。
- 为什么 `DO NOTHING` 经常不适合修复历史错误。
- 如何修复 Fact、DWS、ADS 和多个 Sink。
- 为什么 Postgres 修好了不代表 ClickHouse / Parquet 已经修好。
- 如何通过 Validation / Reconciliation 判断 Repair 是否真的完成。
- 如何让 Historical Repair 可追踪、可重跑、可审计。

本课暂不展开：

- 完整 Data Quality Framework；
- 自动 Incident Management；
- 高级 Workflow Engine；
- 大规模分布式事务；
- 跨链统一 Repair Framework；
- 商业 Data Observability 平台。

---

## 一、为什么需要 Historical Repair？

最简单的场景：

今天发现一个 Decoder Bug。

它已经存在：

```text
2 months
```

代码今天修好了。

但过去两个月的数据：

```text
still wrong
```

这意味着：

> Code Fix ≠ Historical Data Repair.

修代码解决的是：

```text
future processing
```

而 Historical Repair 解决的是：

```text
already materialized wrong history
```

---

## 二、Historical Repair 处理哪些问题？

常见情况包括：

```text
Missing Block / Log
Decoder Bug
Wrong Decimals
Business Mapping Bug
Duplicate Facts
Provider Gap
Reorg Correction
DWS Aggregation Bug
Multi-sink Divergence
```

这些问题有一个共同特点：

> Wrong data has already crossed the processing boundary.

也就是说，数据已经落入历史。

---

## 三、[Data Engineer 视角] Repair 和 Normal Processing 的区别

Normal Processing：

```text
process new data
advance forward
```

Historical Repair：

```text
revisit old range
recompute / replace existing result
verify convergence
```

所以 Historical Repair 的关键不是：

> Can I run the code?

而是：

> Can I safely change already-materialized historical state?

---

## 四、先区分 Retry、Replay、Backfill

这三个词很容易混。

### Retry

目标：

> 重新执行刚才失败的同一次处理。

例如：

```text
DB connection timeout
```

当前 block 没处理成功。

那么：

```text
retry same block
```

属于 Retry。

---

## 五、Replay 是什么？

Replay：

> 对已经处理过的数据重新执行处理逻辑。

例如：

```text
decoder bug fixed
```

然后重新处理：

```text
block 20,000,000 → 20,100,000
```

这些 block 过去已经处理过。

现在重新执行：

```text
process_block()
```

这是 Replay。

---

## 六、Backfill 是什么？

Backfill 更强调：

> 针对一个历史范围补数据或重建数据。

例如：

```text
2026-08-01 → 2026-08-10
```

因为当时漏采某些 logs。

你创建一个历史任务：

```text
start_block
end_block
cursor
```

然后重新填补这个范围。

---

## 七、Replay 和 Backfill 为什么经常一起出现？

现实系统里两者边界并不绝对。

例如 Decoder Bug：

```text
historical range exists
but decoded result is wrong
```

你会：

```text
Backfill historical range
+
Replay corrected logic
```

所以工程上更重要的是理解：

```text
Historical Repair
=
historical scope
+
correct processing logic
+
safe overwrite semantics
+
validation
```

---

## 八、Recompute 又是什么？

Recompute 通常用于 Derived Data。

例如：

```text
Fact repaired
```

然后重新计算：

```text
DWS
ADS
daily aggregates
wallet metrics
```

这更接近：

> derive again from corrected upstream state.

所以可以形成：

```text
Replay Fact
↓
Recompute DWS
↓
Recompute ADS
```

---

## 九、Historical Repair 第一件事不是“开始跑”

第一件事应该是：

> Define the Blast Radius.

也就是明确：

```text
which chain?
which block range?
which contracts?
which event types?
which decoder version?
which tables?
which sinks?
which downstream products?
```

如果 Repair Scope 不清楚，

很容易：

```text
repair too little
```

或者：

```text
repair too much
```

---

## 十、一个 Decoder Bug 的 Blast Radius

假设：

```text
decoder_version = v1.4
```

在：

```text
block 20,000,000 → 20,500,000
```

对某 ERC-20 Contract 的 decimals 处理错误。

那么 Repair Scope 可以定义为：

```text
chain_id = 1
contract = 0xABC...
decoder_version = v1.4
block_range = 20,000,000 → 20,500,000
```

这比：

```text
replay last 2 months
```

精确得多。

---

## 十一、为什么不能直接回退 Realtime Checkpoint？

假设 Realtime：

```text
checkpoint = 21,000,000
```

你发现：

```text
20,000,000 → 20,500,000
```

有历史错误。

如果把 Realtime Checkpoint 改成：

```text
20,000,000
```

那么实时 Pipeline 会从历史位置重新开始。

结果可能：

```text
new blocks stop being processed
freshness collapses
historical repair competes with realtime
checkpoint semantics become confused
```

所以：

> Historical Repair should not hijack the realtime checkpoint.

---

## 十二、正确方式：独立 Repair Job

Realtime：

```text
realtime_checkpoint
```

继续处理最新数据。

Historical Repair：

```text
repair_job_id
start_block
end_block
repair_cursor
repair_status
```

独立执行。

这就是之前你已经掌握的设计：

> Realtime Pipeline and Backfill Pipeline maintain independent processing state.

---

## 十三、Repair Cursor 和 Realtime Checkpoint 的职责不同

例如：

```text
Realtime Checkpoint
= 21,000,000
```

Repair Job：

```text
range = 20,000,000 → 20,500,000
cursor = 20,200,000
```

两者同时存在完全正常。

Realtime 表示：

```text
latest validated processing state
```

Repair Cursor 表示：

```text
how far this historical repair job has progressed
```

---

## 十四、Repair Job 应该保存什么状态？

一个基础设计可以包含：

```text
repair_job_id
repair_type
chain_id
start_block
end_block
cursor
status
reason
logic_version
created_at
started_at
finished_at
validation_status
```

如果涉及多个 Sink，还可以有：

```text
postgres_status
clickhouse_status
parquet_status
```

---

## 十五、Repair Source 从哪里来？

常见来源：

```text
Raw / Bronze
Parquet archive
RPC Provider
Archive Node
Kafka retained data
```

不同来源的优先级取决于：

```text
availability
cost
retention
trust
re-decodability
```

---

## 十六、为什么 Raw Layer 很重要？

假设 Raw Logs 被完整保存。

Decoder Bug 修复后：

```text
Raw Logs
↓
New Decoder
↓
Correct Fact
```

你不需要重新访问 RPC。

优势：

```text
faster
cheaper
deterministic
less provider dependency
```

所以：

> Raw retention is a repair capability.

---

## 十七、Parquet 为什么适合 Historical Repair？

如果历史 Raw Data 已经落到：

```text
Parquet
```

那么可以：

```text
DuckDB / Spark / Python
↓
read historical partitions
↓
recompute
↓
write repaired result
```

特别适合：

```text
large historical scans
partition-based repair
offline reconciliation
```

---

## 十八、什么时候必须回 RPC？

例如：

```text
Raw Layer 当时没保存
Kafka retention 已过期
Parquet 也没有
```

那么只能重新从：

```text
RPC / Archive Node
```

获取历史数据。

这时会面临：

```text
rate limit
cost
provider gap
historical availability
latency
```

所以 Raw Storage 的价值，不只是“留备份”。

---

## 十九、[Architecture 视角] Repairability 是架构属性

好的数据平台不仅要回答：

```text
How do we process data?
```

还应该回答：

```text
How do we repair data?
```

如果系统只能正常运行，

一旦历史错误就只能人工改数据库，

说明：

> Repairability is weak.

---

## 二十、为什么 Replay 应复用原逻辑？

Realtime：

```text
process_block(block)
```

Backfill：

```text
process_block(block)
```

Historical Replay：

```text
process_block(block)
```

最好共用核心逻辑。

否则很容易出现：

```text
realtime logic = v2
repair logic = another implementation
```

最终修复的数据反而和新数据口径不同。

---

## 二十一、但“复用代码”还不够

还需要固定：

```text
logic version
decoder version
metadata version
price source
business rule version
```

因为 Historical Repair 必须回答：

> Which interpretation are we repairing history to?

例如：

```text
decoder v2
```

和：

```text
decoder v3
```

可能产生不同结果。

---

## 二十二、Historical Repair 为什么必须 Idempotent？

Repair Job 也可能：

```text
crash
restart
retry
partial success
```

例如：

```text
block 20,000,000 → 20,100,000
```

已经处理，

系统崩溃。

重启后可能再次处理。

如果 Repair 不是 Idempotent：

```text
duplicate facts
double-counted aggregates
```

就会产生新的 Data Quality Incident。

所以：

> Repair itself must be replay-safe.

---

## 二十三、Stable Unique Key 是 Repair 的基础

例如 Transfer Fact：

```text
chain_id
+
tx_hash
+
log_index
```

它让你能够说：

> This repaired record is the same business/source fact as the old record.

然后才能：

```text
UPDATE
UPSERT
REPLACE
```

而不是盲目 Insert。

---

## 二十四、为什么 DO NOTHING 经常不适合 Repair？

假设旧数据：

```text
key = K1
amount = 1000   -- wrong
```

修复后：

```text
key = K1
amount = 100    -- correct
```

如果：

```sql
ON CONFLICT DO NOTHING
```

那么数据库仍然保留：

```text
1000
```

修复失败。

所以 Historical Repair 经常需要：

```text
DO UPDATE
REPLACE
DELETE + INSERT
partition overwrite
```

---

## 二十五、[关键视角] Idempotency ≠ DO NOTHING

Idempotency 的目标是：

> Repeated execution converges to the same correct final state.

它并不意味着：

```text
never update existing rows
```

在 Repair 场景：

```text
same key
wrong old attributes
```

必须允许：

```text
replace incorrect state with correct state
```

---

## 二十六、Repair Fact 后为什么还没结束？

因为错误 Fact 可能已经被：

```text
DWS
ADS
Dashboard
Risk Metrics
API Cache
ClickHouse MV
Parquet Aggregate
```

使用。

所以：

```text
Fact Correct
```

不自动意味着：

```text
Downstream Correct
```

---

## 二十七、Dependency Blast Radius

需要问：

```text
Which downstream datasets were derived from the affected facts?
```

例如：

```text
fact_token_transfer
↓
dws_wallet_token_daily_flow
↓
ads_wallet_dashboard
```

如果 Fact 的 8 月 10 日数据被修复，

那么至少需要判断：

```text
8 月 10 日 DWS 是否要重算？
ADS 是否要重算？
缓存是否要失效？
```

---

## 二十八、多 Sink Repair 更复杂

假设同一 Fact 同时写：

```text
Postgres
ClickHouse
Parquet
```

你修好了：

```text
Postgres
```

但：

```text
ClickHouse still wrong
Parquet still wrong
```

那么系统内部仍然存在：

> Consistency Failure.

所以 Historical Repair 不能只记录：

```text
repair_job = success
```

而应该知道：

```text
which sink has converged?
```

---

## 二十九、Multi-sink Repair State

例如：

```text
repair_job_id = R123

Postgres  = VERIFIED
ClickHouse = REPAIRING
Parquet   = PENDING
```

此时不能说：

```text
Repair Complete
```

只有：

```text
all required sinks verified
```

才能真正关闭。

---

## 三十、不同 Sink 可以使用不同修复策略

### Postgres

常见：

```text
UPSERT
DELETE + INSERT
transactional replacement
```

### ClickHouse

常见：

```text
partition rebuild
ReplacingMergeTree strategy
mutation
reload corrected partitions
```

### Parquet

常见：

```text
rewrite affected files / partitions
```

所以：

> Same logical repair, different physical repair mechanism.

---

## 三十一、Repair 应该先修哪一层？

通常遵循：

```text
source-near layer
↓
fact
↓
DWS
↓
ADS
↓
serving/cache
```

原因：

> Downstream should be rebuilt from corrected upstream truth.

如果先手工修 ADS，

而 Fact 仍然错误，

下一次重新计算 ADS 时又会变错。

---

## 三十二、[银行系统类比] 不要只改报表

银行场景：

核心交易明细金额错误。

如果你只修改：

```text
月报表
```

但底层交易明细仍然错误，

下一次月报重跑：

```text
wrong again
```

正确方式应该：

```text
repair transaction fact
↓
rebuild summary
↓
rebuild report
```

链上数据完全一样。

---

## 三十三、Repair 完成后必须 Validation

Repair Job 执行成功只是：

```text
processing success
```

不是：

```text
data correctness
```

至少要检查：

```text
row count
key-level match
aggregate reconciliation
invariant
canonical status
expected range completeness
```

---

## 三十四、一个 Decoder Repair 验证例子

修复：

```text
USDC decimals bug
```

可以检查：

```text
amount_raw unchanged
amount corrected
same source keys
no duplicate keys
affected range complete
DWS aggregate reconciles with Fact
```

这样才能证明：

> Repair converged to intended state.

---

## 三十五、Historical Repair 的标准状态流

可以设计：

```text
DETECTED
↓
SCOPED
↓
READY
↓
REPAIRING
↓
UPSTREAM_REPAIRED
↓
DOWNSTREAM_REBUILDING
↓
RECONCILING
↓
VERIFIED
↓
CLOSED
```

如果失败：

```text
FAILED
```

然后：

```text
retry / resume
```

---

## 三十六、为什么要保存 Repair Audit？

几个月后有人问：

> 为什么 8 月 10 日的数据在 10 月 3 日发生了变化？

系统应该能够回答：

```text
repair_job_id
reason
bug ticket
affected range
logic version
before / after
operator / trigger
validation result
completion time
```

这就是：

> Repair Auditability.

---

## 三十七、Repair 不一定意味着整段全量重跑

例如 Bug 只影响：

```text
one contract
one event signature
one decoder version
```

那么可以做：

```text
targeted repair
```

而不是：

```text
rebuild entire chain history
```

所以：

> Smaller verified blast radius is usually safer and cheaper.

---

## 三十八、什么时候应该扩大 Repair Scope？

如果验证发现：

```text
mismatch outside expected range
```

或者：

```text
same bug exists in more contracts
```

那么需要重新：

```text
expand blast radius
```

也就是说 Repair Scope 本身也可能在诊断过程中变化。

---

## 三十九、Historical Repair 与 Realtime Pipeline 如何共存？

理想状态：

```text
Realtime Pipeline
→ continues processing new blocks

Historical Repair Pipeline
→ repairs old range independently
```

但两者可能写同一张表。

所以必须保证：

```text
stable keys
idempotent writes
clear ownership
safe overwrite semantics
resource isolation
```

否则 Repair 会干扰 Realtime。

---

## 四十、资源隔离为什么重要？

一个 3 TB Historical Repair 如果和 Realtime 共用所有资源，

可能造成：

```text
DB saturation
ClickHouse overload
RPC throttling
Kafka backlog
freshness degradation
```

所以常见做法：

```text
rate limit repair
separate worker pool
off-peak execution
priority scheduling
```

核心目标：

> Repair history without breaking the present.

---

## 四十一、什么时候 Historical Repair 可以直接用 Raw / Parquet，不经过 Kafka？

完全可以。

例如：

```text
Parquet Raw History
↓
DuckDB / Repair Job
↓
Corrected ClickHouse Partition
```

不一定需要：

```text
Parquet → Kafka → Consumer → ClickHouse
```

因为 Repair 的目标不是模拟 Realtime Transport，

而是：

> produce the same correct target state.

只要逻辑口径一致即可。

---

## 四十二、那什么时候 Replay Kafka 有价值？

如果你希望验证：

```text
full streaming consumer path
```

或者：

```text
downstream consumer logic itself had bug
```

可能会使用 Kafka Replay。

前提：

```text
retention still covers range
```

并且：

```text
sink remains idempotent
```

---

## 四十三、Kafka Retention 不足怎么办？

例如 Bug 影响：

```text
3 months ago
```

Kafka 只保留：

```text
7 days
```

那就不能依赖 Kafka Replay。

需要：

```text
Raw DB
Parquet
RPC
Archive Node
```

这正是为什么：

> Replay capability depends on durable historical source.

---

## 四十四、Historical Repair 的完整流程

可以压缩为：

```text
Detect Historical Issue
↓
Define Blast Radius
↓
Choose Trusted Repair Source
↓
Freeze Logic / Version
↓
Create Independent Repair Job
↓
Replay / Backfill Fact
↓
Update / Replace Existing State
↓
Recompute Downstream
↓
Repair Every Required Sink
↓
Validate / Reconcile
↓
Audit
↓
Close
```

---

## 四十五、本课核心心智模型

最核心的模型是：

```text
Historical Error
≠
Realtime Failure
```

因此：

```text
Do not rewind realtime blindly
```

而应该：

```text
Independent Repair Scope
+
Independent Repair State
+
Shared Deterministic Logic
+
Idempotent Overwrite
+
Downstream Rebuild
+
Multi-sink Verification
```

---

## 本课核心结论

> Code Fix does not repair already-materialized historical data.

> Retry handles transient execution failure; Replay reprocesses already-seen data; Backfill targets historical ranges; Recompute rebuilds derived state.

> Historical Repair should usually run independently from the Realtime Checkpoint.

> Repair must begin with a clearly defined Blast Radius.

> Raw / Bronze retention and Parquet archives provide durable reprocessing capability.

> Historical Repair should reuse deterministic business logic while recording the exact logic / decoder version used.

> Repair must be idempotent, but idempotency does not mean `DO NOTHING`.

> When old rows are wrong, `DO UPDATE / Replace / overwrite` is often required.

> Fixing Fact does not automatically fix DWS / ADS or every Sink.

> Multi-sink repair must track convergence independently per Sink.

> Repair is not complete until Validation / Reconciliation proves the corrected historical state.

> A production data platform should be designed not only for processing, but also for repairability.

---

## 理解检查

### 问题一

Realtime Pipeline 当前：

```text
checkpoint = 21,000,000
```

你发现 Decoder Bug 影响：

```text
block 20,100,000 → 20,200,000
```

请回答：

1. 为什么不应该直接把 Realtime Checkpoint 改成 20,100,000？
2. 更合理的 Repair 设计是什么？
3. Historical Repair Job 至少应该有哪几个关键 Processing State 字段？

### 问题二

Fact Table 已经存在错误数据：

```text
key = chain_id + tx_hash + log_index
amount = 1000   -- wrong
```

修复后的正确结果是：

```text
same key
amount = 100
```

请回答：

1. 为什么 `ON CONFLICT DO NOTHING` 不能完成这次 Repair？
2. 更适合哪类写入语义？
3. 为什么这种 Repair 仍然可以做到 Idempotent？

### 问题三

一个 Bug 同时影响：

```text
Postgres Fact
ClickHouse Fact
Parquet History
DWS
ADS
```

现在只有 Postgres 已经修复并验证。

请回答：

1. 能不能把 Repair Job 标记为 Completed？
2. 为什么？
3. 更完整的 Repair Completion 条件应该是什么？

## 用户回答（问题一）

问题一：
1.直接修改Realtime Checkpoint 的话，会影响 realtime block的 process，另外 一个Checkpoint 同时用与realtime和repair的处理，导致语义模糊
2.repair设计单独的一个job，但是 process logic 可以使用相同的。
3.unique key（chain_id ， tx_hash ， log_index ） ，start_block , end_block ， cursor 等

## 老师判断 / 校准（问题一）

基本通过，第 3 小问需要校准。

1. **为什么不应该直接修改 Realtime Checkpoint**：正确。直接回退 Realtime Checkpoint 会让 Realtime Pipeline 从历史位置重新开始，影响最新 Block 的处理；同时让同一个 Checkpoint 同时承担 Realtime Processing State 和 Historical Repair State，语义会混乱。

2. **更合理的 Repair 设计**：正确。应创建独立的 Historical Repair Job / Backfill Job，维护自己的 Processing State；核心处理逻辑可以继续复用同一套 `process_block` / transform logic，从而避免 Realtime 与 Repair 产生业务逻辑漂移。

3. **Historical Repair Job 的关键 Processing State 字段**：你提到的 `start_block`、`end_block`、`cursor` 是正确的；但：

```text
chain_id + tx_hash + log_index
```

属于 Fact Table 的 Stable Unique Key，不属于 Repair Job 自身的 Processing State。

Repair Job 更典型的状态字段是：

```text
repair_job_id
chain_id
start_block
end_block
cursor
status
repair_type
logic_version
validation_status
```

必要时还可以包含：

```text
reason
created_at
started_at
finished_at
postgres_status
clickhouse_status
parquet_status
```

其中：

```text
start_block / end_block
→ Repair Scope

cursor
→ 当前修复到哪里

status
→ Job 当前生命周期状态

validation_status
→ 修复结果是否已经通过验证
```

而：

```text
chain_id + tx_hash + log_index
```

更适合用于：

```text
Fact Identity
Idempotent Write
UPDATE / UPSERT / REPLACE
```

核心结论：

> Realtime Checkpoint and Historical Repair State should be maintained independently.

> Fact Unique Key and Repair Job Processing State solve different problems.