# Module 12 · 第 1 课
## 数据质量到底是什么：为什么 Pipeline 成功不代表数据正确

### Lesson Contract

所属 Module：Module 12 — 数据质量

本课核心问题：

> 一个 ETL / Streaming Job 明明成功执行、没有报错，为什么产出的 Blockchain Data 仍然可能是错的？

学完以后，你应该能够解释：

- 为什么 `Job Success ≠ Data Correctness`。
- Data Quality 和 Pipeline Reliability 的区别。
- Blockchain Data Quality 常见的几个维度。
- Source Error、Processing Error、Business Logic Error 分别是什么。
- 为什么链上数据特别需要 Validation / Reconciliation。
- 为什么“数据能查出来”不代表“数据可信”。
- 为什么 Data Quality Failure 有时应该阻止 Checkpoint 推进。

本课不深入具体 Reorg Repair、Missing Block Detection、Decoder Bug Repair；这些会在后续课程展开。

---

## 一、先从你已经熟悉的 ETL 开始

假设一个 Job：

```text
Extract
↓
Transform
↓
Load
```

运行结果：

```text
Job Status = SUCCESS
```

很多传统系统会下意识认为：

> 数据处理成功了。

但从 Data Quality 角度看，这个结论太早。

因为：

```text
Job Success
```

只能说明：

> 程序按它自己的逻辑执行完了。

它并不能证明：

```text
输入数据完整
转换逻辑正确
输出数据没有重复
业务口径正确
链上数据仍然 canonical
```

所以首先要建立一个最重要的区别：

> Pipeline Reliability 和 Data Correctness 是两件事。

---

## 二、Pipeline Reliability 是什么？

Pipeline Reliability 更关注：

```text
Job 有没有启动
有没有报错
有没有超时
有没有崩溃
有没有重试成功
有没有写入 Target
```

例如：

```text
RPC Request = success
Decoder = no exception
Insert = success
Checkpoint = updated
```

从系统运行角度看：

> Everything worked.

但这只证明：

```text
system execution path
```

正常。

不代表：

```text
business data
```

正确。

---

## 三、一个最简单的错误例子

假设 ERC-20 Transfer Decoder 写错了。

正确逻辑：

```text
topics[1] = from
topics[2] = to
data      = amount
```

Bug 版本：

```text
topics[1] = to
topics[2] = from
data      = amount
```

程序会不会报错？

可能完全不会。

结果：

```text
Job Success = true
Insert Success = true
Checkpoint Advanced = true
```

但：

```text
from / to 全部反了
```

这就是：

> Technically successful, semantically wrong.

---

## 四、所以 Data Quality 关心什么？

Data Quality 更关心：

> Is the data fit for its intended use?

也就是：

```text
数据完整吗？
数据准确吗？
有没有重复？
不同层之间一致吗？
够新吗？
字段是否合法？
业务规则有没有被破坏？
```

常见可以拆成几个维度：

```text
Completeness
Accuracy
Consistency
Freshness
Uniqueness
Validity
```

---

## 五、Completeness：数据完整吗？

例如你应该同步：

```text
block 22,000,000
→
block 22,001,000
```

理论上：

```text
1001 blocks
```

但实际只有：

```text
998 blocks
```

Job 可能仍然成功。

比如 Provider 某几个 Block 请求失败后被错误跳过。

那么：

```text
Pipeline Success
```

仍然可能是：

```text
SUCCESS
```

但：

> Data is incomplete.

这就是 Completeness 问题。

---

## 六、Accuracy：数据解析对吗？

假设原始 Log：

```text
amount_raw = 1000000
decimals = 6
```

正确：

```text
amount = 1 USDC
```

如果 Decoder 错误使用：

```text
decimals = 18
```

那么：

```text
amount
```

会被算错。

所有 Block 都可能完整。

没有重复。

Checkpoint 也正常。

但结果仍然错误。

这就是：

> Accuracy issue.

---

## 七、Uniqueness：有没有重复事实？

Kafka At-least-once 下：

```text
same Transfer
```

可能被重复消费。

如果 Sink 没有 Idempotency：

```text
tx_hash = 0xabc
log_index = 10
```

可能出现两次。

结果：

```text
Transfer Volume × 2
```

Job 仍然可以完全成功。

这就是：

> Uniqueness issue.

---

## 八、Freshness：数据对，但太旧

假设 Dashboard 数据：

```text
correct through block 22,000,000
```

链头已经：

```text
22,001,000
```

相差：

```text
1000 blocks
```

数据本身可能完全正确。

但业务要求：

```text
lag < 5 blocks
```

那么这份数据仍然不可用。

所以：

> Correct but stale data can still be bad data.

这就是 Freshness。

---

## 九、Consistency：不同系统是否一致？

假设：

```text
Postgres wallet balance
= 500 USDC
```

ClickHouse 根据 Historical Transfer 聚合：

```text
= 450 USDC
```

Parquet 重放结果：

```text
= 500 USDC
```

这里就出现：

```text
cross-system inconsistency
```

可能原因：

```text
ClickHouse lag
missing event
duplicate correction
reorg handling failure
```

所以多 Sink 架构里：

> Consistency 本身就是 Data Quality 问题。

---

## 十、Validity：字段值是否合法？

例如：

```text
block_number < 0
```

显然不合法。

或者：

```text
token_address = NULL
```

但对于一个要求有 token contract 的 ERC-20 Transfer，这可能不合法。

或者：

```text
amount_raw < 0
```

也可能违反业务定义。

这就是：

> Validity / Domain Rule.

---

## 十一、Blockchain 为什么比传统 ETL 更麻烦？

传统数据仓库常假设：

> 已经写入的历史事实基本稳定。

Blockchain 不是完全这样。

因为：

```text
Reorg
Late-arriving
Provider Gap
Decoder Bug
Replay
Backfill
Duplicate Delivery
```

都可能让：

> 已经处理过的数据后来需要修正。

所以 Blockchain Data Quality 不只是：

```text
before load validation
```

还包括：

```text
after load detection
historical reconciliation
repair
replay
rollback
```

---

## 十二、一个关键视角：Source 也不一定“可靠”

[Data Engineer 视角]

很多人会觉得：

> Blockchain 本身是确定的，所以链上数据源一定可靠。

这个说法太粗。

Protocol 层的 canonical chain 和你拿到的数据之间，还隔着：

```text
Node
RPC Provider
Indexer
Parser
Decoder
Kafka
Consumer
Sink
```

任何一层都可能造成：

```text
missing
duplicate
delay
wrong decode
wrong enrichment
wrong aggregation
```

所以：

> Blockchain may be the source of truth, but your data pipeline can still misrepresent it.

---

## 十三、Source Error、Processing Error、Business Logic Error

可以把错误分成三类。

### 1. Source / Ingestion Error

例如：

```text
RPC Provider missing block
timeout 被错误忽略
log range 少抓一段
```

结果：

```text
missing data
```

### 2. Processing Error

例如：

```text
Decoder 把 from / to 解析反
decimals 用错
reorg correction 没执行
```

结果：

```text
wrong data
```

### 3. Business Logic Error

例如：

```text
DWS 把 inflow / outflow 口径写反
某类 Internal Transfer 被错误计入 Volume
```

底层 Fact 可能没错。

但业务结果错了。

---

## 十四、所以 Data Quality 不能只做一层

你不能只检查：

```text
Raw
```

也不能只检查：

```text
ADS
```

更合理的是：

```text
Source / Raw
↓
Normalized Fact
↓
DWS
↓
ADS
```

不同层做不同类型 Validation。

例如：

```text
Raw
→ block continuity

Normalized
→ unique key / decoded fields

DWS
→ aggregate reconciliation

ADS
→ business rule / serving consistency
```

---

## 十五、Row Count 为什么不够？

假设 Raw Logs：

```text
1000 rows
```

Normalized Transfer：

```text
1000 rows
```

Row Count 一样。

是否说明正确？

不一定。

可能：

```text
100 rows解析错
但总行数没变
```

或者：

```text
50 rows missing
+ 50 rows duplicated
```

最后还是：

```text
1000 rows
```

所以：

> Equal Row Count does not imply equal data quality.

这也是为什么后面我们要学：

```text
Aggregate Reconciliation
Invariant Check
```

---

## 十六、Validation 和 Reconciliation 有什么区别？

Validation 更像：

> Is this dataset internally valid?

例如：

```text
primary fields not null
block_number >= 0
unique key not duplicated
```

Reconciliation 更像：

> Does this dataset agree with another trusted representation?

例如：

```text
Raw Transfer Amount Sum
vs
Normalized Transfer Amount Sum
```

或者：

```text
Parquet replay result
vs
ClickHouse result
```

所以：

```text
Validation
= internal check

Reconciliation
= cross-source / cross-layer comparison
```

---

## 十七、为什么 Validation Failure 不应该推进 Checkpoint？

假设：

```text
block 22,000,000
→
22,000,100
```

ETL 已处理完。

但是 Validation 发现：

```text
Missing 3 blocks
```

如果你仍然推进：

```text
checkpoint = 22,000,100
```

系统就会认为：

> 这个范围已经安全完成。

下次从：

```text
22,000,101
```

继续。

那么缺失数据可能永久留在历史中。

所以更合理：

```text
Process
↓
Validate
↓
If pass
    advance checkpoint
Else
    stop / retry / repair
```

这里的关键是：

> Checkpoint should represent validated completion, not merely attempted processing.

---

## 十八、这和你之前学的 Checkpoint 有什么关系？

Module 9 里我们说：

> Checkpoint = 已完成处理的位置。

现在要进一步升级：

> A production-grade checkpoint should mean safely and validly completed.

不是：

```text
程序跑到哪里
```

而是：

```text
哪一段数据已经通过要求，可以被视为完成
```

这就是 Data Quality 和 Processing State 的连接。

---

## 十九、一个银行系统类比

假设银行日终 Job：

```text
extract transactions
↓
calculate balances
↓
load report
```

Job 全绿。

但最终：

```text
借贷不平
```

银行不会说：

> Job 成功，所以报表正确。

而是会做：

```text
balance reconciliation
control total
accounting invariant
```

Blockchain Data Engineering 也是一样。

Job Success 只是：

```text
技术执行状态
```

Data Quality 才是：

```text
业务可信状态
```

---

## 二十、数据质量的真正目标

不是：

> 永远不出错。

现实系统做不到。

更成熟的目标是：

```text
Detect
↓
Understand
↓
Contain
↓
Repair
↓
Verify
↓
Audit
```

也就是说：

> A good data platform is not one that never has bad data; it is one that can detect and repair bad data reliably.

---

## 二十一、为什么 Blockchain 特别需要“可重放”？

因为很多错误不是立刻发现。

例如：

```text
Decoder Bug introduced:
2026-01-01

Detected:
2026-03-01
```

已经影响两个月。

这时候：

```text
Retry latest block
```

没有意义。

需要：

```text
Historical Replay / Backfill
```

所以：

> Replayability is part of Data Quality Architecture.

---

## 二十二、一个完整质量闭环

可以建立：

```text
Ingest
↓
Process
↓
Validate
↓
Load
↓
Monitor
↓
Detect Issue
↓
Backfill / Replay / Repair
↓
Reconcile
↓
Confirm Correctness
```

这才是完整的数据质量生命周期。

---

## 二十三、Module 12 要解决的核心问题

接下来我们会逐步回答：

```text
How do we detect missing data?
How do we detect duplicates?
How do we know decoded data is accurate?
How do we reconcile different layers?
How do we measure freshness?
How do we handle reorg?
How do we repair historical data?
How do we build a repeatable quality framework?
```

这就是 Module 12 的主线。

---

## 本课核心结论

> Job Success does not imply Data Correctness.

> Pipeline Reliability describes whether processing executed successfully; Data Quality describes whether the resulting data is trustworthy and fit for use.

> Blockchain Data Quality commonly includes Completeness, Accuracy, Consistency, Freshness, Uniqueness, and Validity.

> Errors can originate from ingestion, processing, or business logic.

> Row Count alone is not enough to prove correctness.

> Validation and Reconciliation are different but complementary.

> A Checkpoint should advance only after the processed range satisfies required validation conditions.

> Replayability and repairability are first-class parts of Blockchain Data Quality Architecture.

---

## 理解检查

### 问题一

一个 ETL Job：

```text
Status = SUCCESS
Checkpoint = block 22,000,100
```

但后来发现其中缺失了 3 个 Block。

为什么这里不能说“Pipeline 已成功，所以数据质量没问题”？

你认为这里暴露的是哪个 Data Quality 维度的问题？

### 问题二

下面两个场景分别更偏向什么问题？

A：

```text
同一笔 Transfer 被写入两次
```

B：

```text
所有 Transfer 都完整，但 amount 因 decimals 使用错误而算错
```

请分别说明对应的 Data Quality Dimension。

### 问题三

为什么在生产级 Blockchain ETL 中，更合理的流程是：

```text
Process
↓
Validate
↓
Advance Checkpoint
```

而不是：

```text
Process
↓
Advance Checkpoint
↓
Validate
```

请从“Checkpoint 代表什么”这个角度解释。