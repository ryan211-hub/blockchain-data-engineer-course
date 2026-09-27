# Module 12 第 1 课｜数据质量到底是什么：为什么 Pipeline 成功不代表数据正确

## Lesson Contract

所属 Module：Module 12 — 数据质量

本课核心问题：

> A pipeline can finish successfully, but can the data still be wrong?

学完以后，你应该能够解释：

- 为什么 `Job Success`、`Pipeline Success` 和 `Data Correctness` 是三个不同概念。
- Blockchain Data Quality 为什么比传统报表校验更复杂。
- Completeness、Accuracy、Consistency、Freshness、Uniqueness、Validity 分别在链上数据里是什么意思。
- Source Quality 和 Derived Data Quality 有什么区别。
- 为什么 Reorg、Provider Gap、Decoder Bug、Duplicate Delivery 都属于数据质量问题，但错误类型不同。
- 为什么 Validation Failure 时不能继续推进相关 Checkpoint。
- 一个 Blockchain Data Engineer 应该如何建立 `Detect → Block → Diagnose → Repair → Verify` 的质量闭环。

本课不展开具体 Missing Block SQL、Duplicate Detection SQL、Aggregate Reconciliation 算法和 Reorg Repair 实现；这些分别放到后面的课程。

---

## 一、先看一个最危险的误区

假设每天凌晨有一个 ETL Job：

```text
Extract
↓
Transform
↓
Load
↓
SUCCESS
```

Scheduler 显示：

```text
Job Status = SUCCESS
Runtime = 18 min
Rows Loaded = 12,500,000
```

很多系统会因此认为：

> 今天的数据处理成功了。

但这句话其实只证明：

> The program completed without a detected execution failure.

它并没有证明：

```text
数据没有缺
数据没有重复
字段解析正确
业务含义正确
数据足够新
多套系统彼此一致
数据仍属于 canonical chain
```

所以第一条原则是：

> Pipeline Success ≠ Data Correctness.

---

## 二、先区分三个层次

### 1. Execution Correctness

这是程序运行层面的问题。

例如：

```text
Python process exited with code 0
SQL executed successfully
Kafka consumer did not crash
ClickHouse INSERT succeeded
```

它回答的是：

> Did the computation run successfully?

### 2. Processing Correctness

这是处理逻辑层面的问题。

例如：

```text
block 100 → 200 都处理了吗？
同一个 log 有没有处理两次？
checkpoint 是否正确推进？
reorg 后旧分支有没有撤销？
```

它回答的是：

> Did we process the intended input correctly?

### 3. Data Correctness

这是最终数据产品层面的问题。

例如：

```text
USDC Transfer amount 是否正确？
wallet balance 是否与事实一致？
ClickHouse 与 Postgres 是否对应同一 canonical history？
Dashboard 的 volume 是否少算了 3%？
```

它回答的是：

> Is the resulting data actually trustworthy?

这三个层次不能混在一起。

---

## 三、用银行系统做一个类比

你现在熟悉的数据提取场景很好理解这个区别。

假设银行有一个每日交易报表：

```text
核心交易系统
↓
ETL
↓
交易事实表
↓
日报
```

某天 Job 正常完成。

但上游因为接口问题少传了下午 2:00–2:10 的数据。

结果：

```text
ETL SUCCESS
SQL SUCCESS
报表生成 SUCCESS
```

但事实上：

```text
交易少了 8,000 笔
金额少了 3,000 万
```

这是典型的：

> Technically successful, semantically wrong.

Blockchain 数据工程也是这样，只是多了几个特殊风险：

```text
Reorg
Provider Gap
RPC inconsistency
Decoder Bug
Duplicate Delivery
Late-arriving data
Canonicality change
```

---

## 四、为什么 Blockchain Data Quality 更特殊？

传统数据库中的一条已经提交记录，通常不会突然变成：

```text
昨天是真的
今天变成无效历史
```

但 Blockchain 可能。

例如：

```text
Block 100
└── Tx A
    └── Transfer 100 USDC
```

Indexer 已经处理：

```text
canonical = true
```

随后发生 Reorg：

```text
Old Block 100
→ orphaned

New Block 100
→ canonical
```

此时原来那条 Transfer：

```text
不是“字段错了”
不是“程序跑失败了”
```

而是：

> Its chain truth changed.

这就是 Blockchain Data Quality 和普通企业数据质量的重要差异。

---

## 五、数据质量不是一个指标，而是一组维度

我们先建立六个核心维度。

```text
Completeness
Accuracy
Consistency
Freshness
Uniqueness
Validity
```

这一课先建立地图，后面逐个深入。

---

## 六、Completeness：该有的数据有没有全部到？

Completeness 关心：

> Is anything missing?

例如链上理论上应该处理：

```text
Block 100
Block 101
Block 102
Block 103
```

数据库却只有：

```text
100
101
103
```

那么：

```text
Block 102 missing
```

这是最明显的 Completeness 问题。

但 Blockchain 中还可能更隐蔽。

例如 Block 存在：

```text
Block 102 exists
```

Transaction 也存在，但 Provider 的 `eth_getLogs` 请求漏掉了一部分 Logs。

于是：

```text
Block complete
Transaction complete
Logs incomplete
```

所以 Completeness 必须和数据 Grain 一起看。

---

## 七、Accuracy：有数据，但值对不对？

Accuracy 关心：

> Is the value correct?

例如 ERC-20 Log：

```text
data = 0x...
```

Decoder 成功解析出：

```text
amount_raw = 100000000
```

但 token decimals 实际是：

```text
6
```

程序却错误用了：

```text
18
```

最后：

```text
amount = 0.0000000001
```

正确应该是：

```text
amount = 100
```

这里：

```text
Row exists
Log exists
No duplicate
No missing block
```

但数据仍然错误。

这是 Accuracy 问题。

---

## 八、Consistency：不同数据表示是否相互矛盾？

Consistency 关心：

> Do different representations of the same reality agree?

例如同一份 Transfer Facts：

Postgres：

```text
1,000,000 rows
```

ClickHouse：

```text
998,500 rows
```

不能立刻断言 ClickHouse 错，因为可能只是：

```text
different checkpoint
different freshness SLA
```

但如果二者都声明：

```text
processed through block 22,000,000
canonical only
same business definition
```

仍然数量不同，就出现 Consistency 问题。

所以一致性比较必须先确认：

```text
same range
same grain
same filters
same canonical boundary
same business definition
```

---

## 九、Freshness：数据可以完全正确，但已经太晚

Freshness 问的是：

> Is the data available when the product needs it?

例如：

```text
Blockchain head = 22,000,100
Postgres checkpoint = 22,000,099
```

对于 Balance API：

```text
lag = 1 block
```

可能很好。

如果：

```text
checkpoint = 21,999,000
```

数据可能没有任何字段错误，但是已经落后 1,100 个 block。

对于实时产品来说，这仍然属于：

> Data Quality Failure.

所以：

> Correct but stale data can still be unusable.

---

## 十、Uniqueness：同一个事实是不是被保存了多次？

Kafka 常见：

```text
At-least-once delivery
```

Consumer 处理完：

```text
INSERT succeeded
```

但 offset 还没 commit 就 crash。

重启以后：

```text
same event
→ delivered again
```

如果 Sink 没有 Idempotency：

```text
Transfer A
Transfer A
```

数据库出现两条。

Pipeline 仍然全部 SUCCESS。

这就是：

> Duplicate Delivery → Uniqueness Failure.

后面第 3 课会专门展开。

---

## 十一、Validity：格式正确不等于业务合法

Validity 通常关心：

> Does the value satisfy expected rules or domain constraints?

例如：

```text
chain_id = null
block_number = -1
tx_hash length invalid
log_index missing
```

这些比较容易。

但也可能有业务层规则：

```text
from_address
to_address
contract_address
```

格式都正常，但某类 Event 按协议定义不应该出现这样的组合。

那么它可能通过数据库 schema，却没有通过：

> Domain Validation.

---

## 十二、把六个维度放在一起

可以这样记：

```text
Completeness
→ 有没有少？

Accuracy
→ 值对不对？

Consistency
→ 不同系统 / 表之间是否矛盾？

Freshness
→ 是否足够新？

Uniqueness
→ 是否重复？

Validity
→ 是否符合预期规则？
```

注意：

> 一行数据可能同时在多个维度失败。

例如 Decoder Bug 导致某些 Log 没有被识别：

```text
missing decoded events
```

从输出表看是：

```text
Completeness problem
```

但根因其实是：

```text
Accuracy / Decoder Quality problem
```

所以：

> Quality symptom and root cause are not always the same thing.

---

## 十三、Source Quality 和 Derived Data Quality 必须分开

这是本 Module 很重要的视角。

### [Source Data 视角]

例如：

```text
block
transaction
receipt
log
```

我们首先问：

```text
有没有缺 block？
有没有缺 transaction？
有没有缺 receipt？
有没有缺 log？
provider 返回是否完整？
canonical 标记是否正确？
```

这是：

> Source Quality.

### [Derived Data 视角]

例如：

```text
fact_token_transfer
fact_swap
wallet_token_balance
dws_wallet_daily_flow
dashboard metrics
```

这里问：

```text
decoder 是否正确？
聚合是否正确？
余额是否正确？
DWS 是否漏算？
Dashboard 是否与 Fact 一致？
```

这是：

> Derived Data Quality.

一个非常重要的关系是：

```text
Bad Source
↓
Bad Derived Data
```

但反过来：

```text
Good Source
↓
仍然可能
↓
Bad Derived Data
```

例如：

```text
Raw Log 完整
↓
Decoder Bug
↓
Transfer Fact 错误
```

因此不能只验证 RPC 数据是否完整。

---

## 十四、Data Quality 应该贯穿整个 Pipeline

不要把 Quality 理解成最后跑几个 SQL。

更合理的架构是：

```text
RPC / Node
↓
Source Validation
↓
Indexer
↓
Parsing / Decoder Validation
↓
Kafka
↓
Delivery / Lag Monitoring
↓
Sink Load
↓
Sink Validation
↓
Reconciliation
↓
DWS / ADS
↓
Business Invariant Check
```

Quality 是：

> Cross-cutting concern.

它贯穿数据生命周期，而不是最后一个步骤。

---

## 十五、一个真实案例：Pipeline 全绿，但 Dashboard 错了

假设系统：

```text
RPC
↓
Indexer
↓
Kafka
↓
ClickHouse
↓
DEX Volume Dashboard
```

所有监控：

```text
RPC request success = 99.99%
Indexer status = healthy
Kafka lag = 0
Consumer status = running
ClickHouse insert = success
Dashboard query = success
```

看起来全部正常。

但某次代码发布时 Decoder 把：

```text
Swap amount0
```

和：

```text
Swap amount1
```

的符号逻辑写反。

结果：

```text
ETH volume
↓
under-counted 20%
```

系统层面完全健康。

数据却错了。

这说明：

> Infrastructure health monitoring is not data quality monitoring.

---

## 十六、Liveness、Pipeline Health 和 Data Quality 不一样

Module 5 我们讲过：

```text
Liveness
Readiness
```

它们回答：

```text
service alive?
service ready?
```

Kafka Lag 回答：

```text
consumer keeping up?
```

Job Status 回答：

```text
execution succeeded?
```

而 Data Quality 回答：

```text
data trustworthy?
```

所以监控体系通常应该至少分为：

```text
System Health
Pipeline Health
Data Quality
```

三个层次。

---

## 十七、Blockchain 特有的几类质量事故

后面几课基本围绕这些展开。

### 1. Missing Block / Missing Log

```text
Completeness
```

### 2. Duplicate Delivery

```text
Uniqueness
```

### 3. Decoder Bug

```text
Accuracy
```

### 4. Provider Gap

通常表现为：

```text
Completeness
Consistency
```

### 5. Reorg

可能影响：

```text
Accuracy
Consistency
Canonicality
Derived State
```

### 6. Consumer Lag

主要是：

```text
Freshness
```

### 7. Multi-sink divergence

例如：

```text
Postgres corrected
ClickHouse not corrected
```

主要是：

```text
Consistency
```

---

## 十八、Data Quality 的最终目的不是“发现错误”

发现错误只是第一步。

一个成熟的数据质量系统应该形成：

```text
Detect
↓
Block / Contain
↓
Diagnose
↓
Repair
↓
Verify
↓
Resume
```

中文可以理解为：

```text
发现
↓
阻断扩散
↓
定位
↓
修复
↓
复核
↓
继续推进
```

这才是完整的：

> Data Quality & Repair Loop.

---

## 十九、为什么 Validation Failure 不能推进 Checkpoint？

假设 ETL 正在处理：

```text
block 1000 → 1099
```

处理完成后：

```text
rows inserted successfully
```

于是准备：

```text
checkpoint = 1099
```

但 Validation 检查发现：

```text
expected transfer amount
≠
loaded transfer amount
```

如果仍然推进：

```text
checkpoint = 1099
```

系统就会认为：

> 1000–1099 已经完成。

下次从：

```text
1100
```

开始。

这样错误数据就被正式“封存”在已完成区间中。

所以正确逻辑应该是：

```text
Process
↓
Load
↓
Validate
↓
PASS?
├─ Yes → Advance Checkpoint
└─ No  → Keep Checkpoint / Fail Job
```

这和你之前 Module 9 学到的原则完全一致：

> Validation is part of completion semantics.

不是附加报告。

---

## 二十、Checkpoint 本质上是一种“完成声明”

这是理解上很关键的一层。

Checkpoint 不只是：

```text
我运行到哪里了
```

更准确地说，它意味着：

> Up to this point, the pipeline claims processing is complete according to its correctness rules.

所以如果质量验证失败，却推进 Checkpoint，相当于系统做出了错误声明：

```text
incorrect data
→ marked complete
```

这比 Job 失败更危险。

因为 Job 失败很明显。

而：

```text
SUCCESS + wrong checkpoint
```

会让错误长期隐藏。

---

## 二十一、数据质量应该有“可证明性”

作为 Data Engineer，不应该只说：

> 我觉得数据应该没问题。

应该能够给出证据。

例如：

```text
Source range complete
+
No block gaps
+
No duplicate unique keys
+
Aggregate reconciliation passed
+
Canonical boundary verified
+
Sink checkpoints aligned with SLA
```

于是你才能说：

> Data quality checks passed for this processing range.

这也是为什么数据平台需要：

```text
quality metrics
validation results
audit logs
repair history
```

---

## 二十二、不要追求“永远零错误”

真实数据系统几乎不可能保证：

```text
nothing ever goes wrong
```

更现实的工程目标是：

```text
detect quickly
limit blast radius
repair safely
prove recovery
```

也就是：

> Reliability is not the absence of failure; it is the ability to detect and recover correctly.

Blockchain Data Engineer 尤其如此，因为你不能控制：

```text
Provider
Network
Chain reorg
Upstream protocol changes
```

但你可以控制：

```text
Detection
Checkpoint semantics
Idempotency
Replay
Reconciliation
Auditability
```

---

## 二十三、CTO 视角：为什么 Data Quality 是架构问题，而不是 SQL 问题？

如果一家公司直到 Dashboard 出错才发现：

```text
过去三个月 Swap Volume 全错
```

这说明问题不只是某条 SQL 写错。

而是系统缺少：

```text
quality gates
reconciliation
versioned decoder
replay capability
historical raw data
audit trail
```

所以 CTO 更关心：

```text
Can we detect corruption?
Can we know the affected range?
Can we stop propagation?
Can we replay safely?
Can we prove the repair worked?
```

这就是为什么：

> Data Quality is part of system architecture.

---

## 二十四、和前面几个 Module 串起来

现在你可以看到 Module 12 其实不是新开一套知识，而是在收束前面的课程。

### Module 7 Indexer

```text
Checkpoint
Backfill
Reorg
Idempotency
```

在这里变成：

```text
Correctness foundation
```

### Module 9 ETL

```text
Incremental Load
Validation
DAG
Retry
Backfill
```

在这里变成：

```text
Quality control + repair
```

### Module 10 Streaming

```text
At-least-once
Lag
Replay
Reorg Correction
```

在这里变成：

```text
Uniqueness + Freshness + Recovery
```

### Module 11 Analytics DB

```text
Postgres
ClickHouse
Parquet
multiple sinks
```

在这里变成：

```text
Cross-sink consistency + reconciliation
```

所以 Module 12 本质上是在回答：

> 前面这些系统都建起来以后，我怎么知道整个数据平台值得相信？

---

## 二十五、先建立一个最小 Data Quality Framework

目前先记住这五层：

```text
1. Input Quality
   ↓
2. Processing Quality
   ↓
3. Output Quality
   ↓
4. Reconciliation
   ↓
5. Repairability
```

具体是：

```text
Input Quality
→ source 有没有缺失

Processing Quality
→ parser / decoder / transformation 是否正确

Output Quality
→ sink 中的数据是否满足预期

Reconciliation
→ 不同来源 / sink / 层次之间是否能相互验证

Repairability
→ 出错后是否可以 replay / backfill / rollback
```

如果一个系统只有前四项，没有第五项，那么发现错误以后仍然可能无能为力。

因此：

> Repairability is part of data quality architecture.

---

## 二十六、本课最重要的心智转换

以前容易这样理解：

```text
Pipeline
→ move data
```

现在应该升级为：

```text
Pipeline
→ move data
→ prove correctness
→ preserve recoverability
```

Data Engineer 不只是让数据：

```text
arrive
```

还要让数据：

```text
trustworthy
traceable
repairable
```

---

## 本课核心结论

> Pipeline Success only proves that the execution path completed; it does not prove that the resulting data is correct.

> Blockchain Data Quality must consider Completeness, Accuracy, Consistency, Freshness, Uniqueness, Validity, and canonical-chain changes.

> Source Quality and Derived Data Quality are separate layers. Correct source data can still produce incorrect derived data.

> Data quality is not a final SQL check. It must exist across ingestion, parsing, decoding, loading, reconciliation, and repair.

> A checkpoint should advance only when the pipeline has satisfied its completion and validation rules.

> A mature Blockchain Data Platform needs not only error detection, but a full Detect → Contain → Diagnose → Repair → Verify loop.

---

## 理解检查

### 问题一

假设下面这个 Pipeline：

```text
Ethereum RPC
↓
Indexer
↓
Kafka
↓
Consumer
↓
ClickHouse
```

当天所有程序都正常运行：

```text
RPC success
Indexer success
Kafka lag = 0
Consumer success
ClickHouse INSERT success
```

但由于 Decoder Bug，某一种 ERC-20 Token 的 `decimals` 被错误地当成 18，而实际是 6。

请回答：

1. 这是 Pipeline Failure 吗？
2. 主要属于哪一个 Data Quality Dimension？
3. 为什么单看系统健康监控发现不了这个问题？

### 问题二

某个 Incremental Job 正在处理：

```text
block 10,000 → 10,999
```

Load 已成功，但 Aggregate Reconciliation 失败。

此时是否应该：

```text
checkpoint = 10,999
```

为什么？

### 问题三

下面两种情况分别更接近 Source Quality 还是 Derived Data Quality？

A：

```text
Provider 漏掉 block 15,003 的部分 Logs
```

B：

```text
Raw Logs 完整，但 Swap Decoder 把 token0 / token1 的 amount 方向解析反了
```

分别说明理由。