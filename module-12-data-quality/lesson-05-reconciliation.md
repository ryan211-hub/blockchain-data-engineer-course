# Module 12 第 5 课｜Reconciliation：Row Count、Aggregate Check 与 Invariant

## Lesson Contract

所属 Module：Module 12 — 数据质量

本课核心问题：

> How do we compare two representations of data and decide whether they are consistent enough to trust?

学完以后，你应该能够解释并设计：

- 什么是 Reconciliation。
- Validation 和 Reconciliation 的区别。
- 为什么 Row Count 经常不够。
- 什么是 Aggregate Reconciliation。
- 什么是 Key-level Reconciliation。
- 什么是 Invariant Check。
- 为什么不同 Grain 的表不能直接比较 Row Count。
- 如何选择适合的 Reconciliation Metric。
- 如何定位 mismatch 的 Blast Radius。
- 为什么 Reconciliation Failure 不应被简单当成“SQL 不一致”。
- 如何把 Reconciliation 用在 Source vs Fact、Fact vs DWS、Postgres vs ClickHouse、Realtime vs Backfill。

本课暂不展开：

- 完整 Reorg Repair；
- Freshness SLA；
- 多维异常检测模型；
- 完整 Data Observability 平台；
- 分布式事务一致性协议。

---

## 一、Reconciliation 到底是什么？

Reconciliation 可以理解为：

> 把两个本来应该在某种规则下保持一致的数据集合进行核对。

例如：

```text
Raw Logs
vs
Decoded Transfer Facts
```

或者：

```text
Postgres
vs
ClickHouse
```

或者：

```text
Fact
vs
DWS Aggregate
```

你不是简单问：

> 数据有没有？

而是在问：

> These two representations should correspond in some way — do they actually reconcile?

---

## 二、[Data Quality 视角] Validation 和 Reconciliation 不完全一样

### Validation

Validation 更像：

> 单独检查某个数据集是否满足预定义规则。

例如：

```text
amount_raw >= 0
tx_hash IS NOT NULL
no duplicate unique key
block range continuous
```

### Reconciliation

Reconciliation 更像：

> 比较两个数据集合之间的关系是否符合预期。

例如：

```text
Raw Transfer Logs count
=
Decoded Transfer Facts count
```

或者：

```text
SUM(Fact amount)
=
DWS daily transfer amount
```

所以：

```text
Validation
→ one dataset against rules

Reconciliation
→ one dataset against another dataset / representation
```

---

## 三、为什么需要 Reconciliation？

因为很多错误单看一张表看不出来。

例如：

```text
fact_token_transfer
```

里面：

```text
no null
no duplicate
schema valid
```

看起来完全正常。

但实际上少了 10% 的 Transfer。

如果你只检查 Fact 本身：

```text
PASS
```

但如果拿 Raw Logs 对比：

```text
Raw matching logs = 10,000
Fact rows         = 9,000
```

就发现问题。

这就是 Reconciliation 的价值：

> Detect silent divergence.

---

## 四、最直观的 Reconciliation：Row Count

最简单的方式：

```text
Dataset A row count
vs
Dataset B row count
```

例如：

```text
Raw Transfer Logs = 10,000
Fact Transfers    = 10,000
```

看起来：

```text
PASS
```

但问题是：

> Row Count equal does not prove the rows are correct.

---

## 五、Row Count 为什么经常不够？

假设 Raw：

```text
A
B
C
D
```

Fact：

```text
A
B
C
X
```

两边都是：

```text
4 rows
```

所以：

```text
Row Count = PASS
```

但：

```text
D missing
X unexpected
```

真实数据仍然错误。

所以：

> Equal count does not imply equal content.

---

## 六、Row Count 还有一个更严重的问题：Grain 不一致

假设：

```text
transactions
grain = one transaction
```

而：

```text
token_transfers
grain = one decoded Transfer log
```

一笔 Transaction 可以产生：

```text
0
1
5
20
```

条 Transfer。

所以：

```text
transaction_count
```

和：

```text
transfer_count
```

天然不应该相等。

此时如果做：

```text
Row Count Reconciliation
```

本身就是错误设计。

---

## 七、[Data Modeling 视角] Reconciliation 前先确认 Grain

这是本课第一条基本原则：

> Reconciliation must respect Grain.

在比较之前先问：

```text
What does one row represent in Dataset A?
What does one row represent in Dataset B?
```

如果 Grain 不同，就不能直接：

```text
COUNT(*) = COUNT(*)
```

必须先找到：

> 两者之间真正应该成立的业务关系。

---

## 八、一个 Grain 不同但可以核对的例子

例如：

```text
Raw Logs
grain = one log
```

而：

```text
fact_token_transfer
grain = one decoded ERC-20 Transfer event
```

不是所有 Log 都是 Transfer。

所以应该先过滤：

```text
Raw Logs
WHERE topic0 = Transfer Signature
```

再比较：

```text
matching Transfer logs
vs
decoded Transfer facts
```

此时 Grain 才对齐。

---

## 九、Key-level Reconciliation 比 Row Count 更强

假设双方 Grain 已对齐：

```text
chain_id + tx_hash + log_index
```

作为 Source Identity。

可以比较：

```text
Source Keys
vs
Fact Keys
```

从而发现：

```text
Missing Keys
Unexpected Keys
```

例如：

```text
Source:
A B C D

Fact:
A B C X
```

结果：

```text
Missing in Fact:
D

Unexpected in Fact:
X
```

这比单纯 Row Count 更有诊断价值。

---

## 十、SQL 怎么做 Key-level Reconciliation？

例如 Source：

```text
raw_transfer_logs
```

Fact：

```text
fact_token_transfer
```

可以查 Source 有、Fact 没有：

```sql
SELECT
    s.chain_id,
    s.tx_hash,
    s.log_index
FROM raw_transfer_logs s
LEFT JOIN fact_token_transfer f
    ON  s.chain_id = f.chain_id
    AND s.tx_hash = f.tx_hash
    AND s.log_index = f.log_index
WHERE f.tx_hash IS NULL;
```

这回答：

> Which expected facts are missing?

反过来：

```sql
SELECT
    f.chain_id,
    f.tx_hash,
    f.log_index
FROM fact_token_transfer f
LEFT JOIN raw_transfer_logs s
    ON  f.chain_id = s.chain_id
    AND f.tx_hash = s.tx_hash
    AND f.log_index = s.log_index
WHERE s.tx_hash IS NULL;
```

回答：

> Which facts have no matching source evidence?

---

## 十一、Row Count 的正确用途是什么？

Row Count 并不是没用。

它适合：

```text
cheap first-line signal
```

例如：

```text
expected 10,000
actual   7,000
```

明显异常。

所以：

```text
Row Count
```

更适合作为：

> Coarse-grained check.

而不是：

> Proof of correctness.

---

## 十二、Aggregate Reconciliation 是什么？

Aggregate Reconciliation 是：

> 不逐行比较，而是比较聚合后的业务指标。

例如：

```text
SUM(amount_raw)
```

或者：

```text
COUNT(DISTINCT wallet)
```

或者：

```text
SUM(volume_usd)
```

比较：

```text
Source-derived aggregate
vs
Target-derived aggregate
```

---

## 十三、为什么 Aggregate Reconciliation 很重要？

有些上下游 Grain 不同。

例如：

```text
Fact Transfer
grain = one transfer
```

DWS：

```text
grain = one day + wallet + token
```

Row Count 不可能相等。

但可以比较：

```text
SUM(Fact.amount)
```

和：

```text
SUM(DWS.inflow + outflow)
```

如果业务定义一致，就可以核对。

所以：

> Different Grain often requires Aggregate Reconciliation.

---

## 十四、你之前已经遇到过这个问题

例如：

```text
Fact
→ many transfer rows
```

而：

```text
DWS
→ daily wallet-token summary
```

如果直接比较：

```text
Fact row count
vs
DWS row count
```

没有意义。

应该比较：

```text
same business measure
under same filter and scope
```

比如：

```text
date
chain
token
wallet population
canonical status
```

必须对齐。

---

## 十五、Aggregate Reconciliation 的前提：口径必须一致

这是最容易犯错的地方。

例如 Fact 统计：

```text
all transfers
```

DWS 统计：

```text
canonical transfers only
```

或者 Fact：

```text
UTC date
```

DWS：

```text
Singapore local date
```

即使两个系统都正确：

```text
aggregates can differ
```

所以 Reconciliation 前必须对齐：

```text
scope
filter
time boundary
canonical rule
token universe
decimal rule
business definition
```

---

## 十六、[银行系统类比] 对账不是简单 COUNT

银行里“对账”通常也不是只看：

```text
交易笔数
```

而是同时看：

```text
笔数
金额
账户
交易类型
日期
渠道
```

例如：

```text
核心系统：
10,000 笔
金额 100,000,000

数据仓库：
10,000 笔
金额 95,000,000
```

Row Count 一样。

但金额明显不一致。

所以：

> Count reconciliation can pass while financial reconciliation fails.

---

## 十七、Blockchain 里常见的 Aggregate Reconciliation 指标

例如 Transfer：

```text
row_count
SUM(amount_raw)
COUNT(DISTINCT tx_hash)
COUNT(DISTINCT wallet)
per-token count
per-block count
```

Swap：

```text
swap_count
volume_token0
volume_token1
volume_usd
unique_traders
```

不同业务对象应该选择：

> meaningful measures.

---

## 十八、Checksum / Hash 可以做什么？

有时可以将某一范围按稳定顺序构造：

```text
key + selected attributes
```

然后生成：

```text
checksum / hash
```

两边比较。

优点：

```text
compact
fast comparison
```

但缺点：

```text
hash mismatch
```

只能告诉你：

> something differs.

不能直接告诉你：

> which row differs.

所以 Checksum 更适合：

> Fast detection.

Key-level diff 更适合：

> Diagnosis.

---

## 十九、Invariant Check 和 Reconciliation 有什么区别？

Invariant Check 问：

> 某个本来应该永远成立的关系是否成立？

它不一定要求有第二张表。

例如：

```text
block_number increasing
```

或者：

```text
unique source identity
```

或者某协议中：

```text
reserve relationship
```

所以：

```text
Reconciliation
→ compare representations

Invariant
→ verify structural/business truth
```

---

## 二十、一个简单 Invariant

例如 Transfer Fact：

```text
chain_id + tx_hash + log_index
```

应该唯一。

那么：

```sql
SELECT
    chain_id,
    tx_hash,
    log_index,
    COUNT(*) AS cnt
FROM fact_token_transfer
GROUP BY
    chain_id,
    tx_hash,
    log_index
HAVING COUNT(*) > 1;
```

这里检查的是：

> Uniqueness Invariant.

这不是在和另一张表比较。

---

## 二十一、Invariant 可以跨层

例如：

```text
matching raw Transfer logs
=
decoded Transfer facts
```

严格来说既可以看成：

```text
Reconciliation
```

也可以看成一个系统 invariant：

> Every matching raw Transfer log should produce exactly one Transfer Fact.

这说明现实工程里这些概念会重叠。

关键不是术语边界，而是：

> 你到底在验证什么关系？

---

## 二十二、Reconciliation 应该在哪些层做？

成熟数据链路通常不是只在最后做一次。

例如：

```text
Provider / Raw
↓
Raw vs Parsed
↓
Parsed vs Decoded Fact
↓
Fact vs DWS
↓
DWS vs ADS
↓
Postgres vs ClickHouse
```

每一层都可以有不同 reconciliation rule。

---

## 二十三、Source vs Fact Reconciliation

例如：

```text
matching Transfer logs
vs
fact_token_transfer
```

可以检查：

```text
count
keys
amount_raw
contract address
```

这主要验证：

```text
Completeness
Uniqueness
Decode Accuracy
```

---

## 二十四、Fact vs DWS Reconciliation

例如：

```text
fact_token_transfer
```

到：

```text
dws_wallet_token_daily_flow
```

这时通常不能比 row count。

应该比：

```text
per date
per wallet
per token
```

的：

```text
SUM(inflow)
SUM(outflow)
```

或者净流量：

```text
SUM(inflow) - SUM(outflow)
```

---

## 二十五、Postgres vs ClickHouse Reconciliation

假设：

```text
Postgres
```

用于 serving，

```text
ClickHouse
```

用于 analytics。

两边由独立 Consumer 写入。

由于：

```text
checkpoint independent
```

短时间内不同步并不一定是错误。

所以比较时必须先确认：

```text
same committed processing boundary
```

例如：

```text
Postgres watermark = block 20,000,000
ClickHouse watermark = block 19,999,500
```

这时不能直接拿最新全量结果比较。

---

## 二十六、[Streaming 视角] Boundary Alignment 很重要

这是你之前 Kafka 课程的延伸。

Reconciliation 必须明确：

```text
compare up to which block?
```

例如：

```text
min(
    postgres_checkpoint,
    clickhouse_checkpoint
)
```

作为共同边界。

否则你看到的 mismatch 可能只是：

> processing lag difference

而不是：

> data corruption.

---

## 二十七、Realtime vs Backfill 也需要 Reconciliation

假设：

```text
Realtime Pipeline
```

已经处理某段数据。

后来：

```text
Backfill Pipeline
```

重算同一范围。

你可以比较：

```text
Realtime output
vs
Backfill output
```

如果逻辑相同、输入相同、canonical state 相同：

```text
results should converge
```

这可以帮助发现：

```text
logic drift
configuration drift
historical bug
```

---

## 二十八、什么是 Reconciliation Window？

Reconciliation 通常不会每次比较：

```text
all history
```

因为成本太高。

常见做法：

```text
hourly window
daily partition
block range
recent N blocks
```

例如：

```text
block 20,000,000 → 20,009,999
```

作为 reconciliation window。

---

## 二十九、Window 的好处

它让问题更容易：

```text
detect
locate
repair
re-run
audit
```

如果你只知道：

```text
过去两年总金额差了 0.1%
```

定位非常困难。

如果你知道：

```text
2026-09-30 14:00–15:00
block 21,123,000–21,123,999
```

差异出现，

修复范围清晰很多。

---

## 三十、Mismatch 不应该只有 PASS / FAIL

更成熟的设计会记录：

```text
expected
actual
difference
difference_ratio
window
metric
source
target
status
```

例如：

```text
metric = transfer_count
expected = 10000
actual = 9980
diff = -20
diff_ratio = -0.2%
```

这样才能做：

```text
alerting
trend analysis
repair audit
```

---

## 三十一、Threshold 应该怎么理解？

不是所有 mismatch 都一定能要求：

```text
difference = 0
```

例如某些：

```text
late-arriving
eventual consistency
price enrichment
external reference data
```

可能允许短时间偏差。

所以规则可以是：

```text
exact match
```

或者：

```text
tolerance
```

例如：

```text
difference_ratio < 0.01%
```

但要注意：

> Tolerance must come from business semantics, not convenience.

---

## 三十二、Blockchain Core Fact 通常更适合 Exact Match

对于：

```text
chain_id + tx_hash + log_index
```

这种链上基础事实，

如果比较：

```text
same canonical range
same decoder version
same source definition
```

通常应该追求：

```text
exact match
```

因为：

```text
one missing log
```

就是一条真实事实缺失。

而不是“允许 1% 误差”。

---

## 三十三、Enrichment 数据可能允许 Tolerance

例如：

```text
amount_usd
```

不同价格源：

```text
Provider A
Provider B
```

可能略有差异。

所以：

```text
exact equality
```

未必合理。

可能比较：

```text
relative difference
```

例如：

```text
abs(a - b) / reference < threshold
```

这属于：

> Semantic tolerance.

---

## 三十四、Reconciliation Failure 后怎么办？

不能只是：

```text
send alert
```

然后结束。

标准流程应该是：

```text
Detect mismatch
↓
Classify mismatch
↓
Locate range / keys
↓
Find root cause
↓
Repair
↓
Rebuild downstream if needed
↓
Re-run reconciliation
↓
Close incident
```

---

## 三十五、先判断是哪一种 mismatch

例如：

```text
Count mismatch
```

可能代表：

```text
missing rows
duplicate rows
filter mismatch
boundary mismatch
late-arriving
reorg
```

```text
Amount mismatch
```

可能代表：

```text
decoder error
decimals error
duplicate fact
missing fact
business logic drift
```

所以：

> Metric mismatch is a symptom, not automatically the root cause.

---

## 三十六、一个完整例子

假设某天：

```text
Raw Transfer logs:
100,000 rows

Fact:
100,000 rows
```

Row Count：

```text
PASS
```

但是：

```text
SUM(raw amount_raw)
=
5,000,000,000

SUM(fact amount_raw)
=
5,100,000,000
```

Aggregate：

```text
FAIL
```

进一步 Key-level 检查发现：

```text
100 source keys missing
100 unexpected keys present
```

所以：

```text
same count
but wrong membership
```

这正说明：

> Row Count alone is insufficient.

---

## 三十七、再看另一个例子

Fact：

```text
10,000 transfer rows
```

DWS：

```text
2,500 wallet-token-day rows
```

Row Count：

```text
not comparable
```

但：

```text
SUM(Fact inflow)
=
SUM(DWS inflow)
```

且：

```text
SUM(Fact outflow)
=
SUM(DWS outflow)
```

那么：

```text
Aggregate Reconciliation = PASS
```

这就是 Grain 不同情况下正确的核对方式。

---

## 三十八、Checkpoint 与 Reconciliation Failure

假设当前 Batch：

```text
block 25,100,000 → 25,100,999
```

ETL 已写完 Fact。

但是 Reconciliation 发现：

```text
matching raw logs = 10,000
fact rows         = 9,950
```

那么：

```text
checkpoint should not advance
```

因为：

> Completion includes successful reconciliation.

这继续强化你之前的理解：

> Checkpoint = validated completion state.

---

## 三十九、历史 Reconciliation Failure 怎么办？

如果今天才发现：

```text
三个月前某个 daily aggregate
```

不一致，

Realtime Checkpoint 已经很远。

那么：

```text
do not rewind realtime checkpoint
```

而是：

```text
identify affected partition
↓
recompute / backfill
↓
replace
↓
reconcile again
```

也就是 Historical Repair。

---

## 四十、本课核心心智模型

可以压缩为：

```text
Same business truth
↓
Different representations
↓
Align grain / scope / boundary
↓
Choose reconciliation metric
↓
Compare
↓
Locate mismatch
↓
Repair
↓
Reconcile again
```

---

## 本课核心结论

> Reconciliation compares two representations that should satisfy a defined relationship.

> Validation checks one dataset against rules; Reconciliation checks relationships between datasets.

> Row Count is useful as a coarse signal, but equal counts do not prove equal content.

> Reconciliation must respect Grain. Different Grain usually requires Aggregate Reconciliation rather than direct Row Count comparison.

> Key-level reconciliation identifies missing and unexpected facts and is more diagnostic than count alone.

> Aggregate Reconciliation compares meaningful business measures under aligned scope, filters and boundaries.

> Invariant Check verifies a relationship that should always hold; it may or may not involve two datasets.

> Multi-sink reconciliation requires boundary alignment, otherwise processing lag can look like corruption.

> A reconciliation mismatch is a symptom. Root cause may be missing data, duplicates, decoder bugs, boundary mismatch, reorg, late-arriving data or logic drift.

> Reconciliation Failure in the current processing range should block checkpoint advancement.

---

## 理解检查

### 问题一

现在有两张表：

```text
transactions
grain = one transaction
```

```text
token_transfers
grain = one Transfer event
```

某一天：

```text
transactions = 10,000 rows
token_transfers = 25,000 rows
```

请回答：

1. 能不能因为 Row Count 不相等，就判断数据质量失败？
2. 为什么？
3. 如果要做 Reconciliation，第一步应该先确认什么？

### 问题二

某个 block range 中：

```text
matching raw Transfer logs = 10,000
fact_token_transfer rows   = 10,000
```

Row Count 完全相同。

但你仍然怀疑数据有问题。

请回答：

1. Row Count 相等能不能证明数据正确？
2. 接下来你会做哪一种更强的 Reconciliation？
3. 如果 Source Keys 是 `A B C D`，Fact Keys 是 `A B C X`，这说明什么？

### 问题三

现在：

```text
fact_token_transfer
grain = one Transfer
```

而：

```text
dws_wallet_token_daily_flow
grain = one date + wallet + token
```

请回答：

1. 为什么不能直接比较两张表的 Row Count？
2. 更适合比较什么？
3. 如果当前 Batch 写完 Fact 后，Aggregate Reconciliation FAIL，Checkpoint 是否应该推进？为什么？

## 用户回答（问题一）

问题1 ：
1.不能
2.transactions 和 token_transfers  的grain不同，不能直接比较。
3.先确认业务语义是否一致。

## 老师判断 / 校准（问题一）

基本通过。

1. **不能因为 Row Count 不相等就判断数据质量失败**：正确。
2. **原因是 Grain 不同**：正确。transactions 的 Grain 是 one transaction，而 token_transfers 的 Grain 是 one Transfer event。一笔 Transaction 可以产生 0、1 或多条 Transfer，因此两张表的 Row Count 天然没有一一对应关系。
3. **第一步应该先确认什么**：你的“先确认业务语义是否一致”方向是对的，但还需要更精确。第一步应先确认：

```text
What does one row represent in Dataset A?
What does one row represent in Dataset B?
```

也就是先确认两边的 **Grain / row meaning**。

在 Grain 对齐或明确映射关系之后，再继续确认：

```text
business definition
scope
filter
time boundary
canonical rule
```

因此，这道题最重要的原则是：

> Reconciliation must respect Grain.

以及：

> Before comparing counts, first define what one row means on each side.

## 用户回答（问题二）

问题2：
1.不能
2.接下来可以对比 unique key 是否一致。
3.说明Source Keys  和 Fact Keys   的数据不一致。需要进一步检查Source Keys 中的D 是从哪里来的，而Fact Keys中x怎么丢失了

## 老师判断 / 校准（问题二）

基本通过，但第 3 小问的方向需要修正。

1. **Row Count 相等不能证明数据正确**：正确。相同数量只能说明两边行数一致，不能证明成员集合一致，也不能证明字段值一致。
2. **下一步做更强的 Reconciliation**：正确。可以做 **Key-level Reconciliation**，比较双方的 Stable Unique Key / Source Identity。
3. **Source Keys = A B C D，Fact Keys = A B C X**：你已经判断出双方数据不一致，这一点正确。但具体方向应是：

```text
D
→ exists in Source
→ missing in Fact
→ Missing Key / Missing Fact

X
→ exists in Fact
→ no matching Source
→ Unexpected Key / Unexpected Fact
```

所以不是“检查 Source 中的 D 从哪里来”，而是应该检查：

```text
为什么 D 没有进入 Fact？
为什么 Fact 中出现了没有 Source Evidence 的 X？
```

这正是 Key-level Reconciliation 比 Row Count 更有诊断价值的原因。

核心结论是：

> Equal Row Count does not imply equal membership.

以及：

> Key-level reconciliation should identify both Missing Keys and Unexpected Keys.

## 用户回答（问题三）

问题3：
1.因为两个表的grain不同，直接比较没有意义。
2.限定好数据范围，可以比较 sum(amount)
3.Checkpoint  不应该推进。Reconciliation Failure in the current processing range should block checkpoint advancement

## 老师判断 / 校准（问题三）

通过。

1. **不能直接比较 Row Count**：正确。两张表 Grain 不同：

```text
fact_token_transfer
→ one Transfer

dws_wallet_token_daily_flow
→ one date + wallet + token
```

因此 Row Count 没有直接可比性。

2. **更适合比较什么**：正确。应使用 **Aggregate Reconciliation**，例如比较：

```text
SUM(amount)
SUM(inflow)
SUM(outflow)
net flow
```

但在比较前，必须确保双方口径一致，包括：

```text
scope
filter
time boundary
token
canonical rule
decimals / normalization rule
business definition
```

否则 Aggregate mismatch 可能只是口径不同，并不一定是数据错误。

3. **Checkpoint 是否推进**：正确。当前处理范围内如果 Aggregate Reconciliation FAIL，则 Checkpoint 不应推进，因为该范围尚未完成 validated completion。

核心结论：

> Different Grain requires Aggregate Reconciliation rather than direct Row Count comparison.

以及：

> Reconciliation Failure in the current processing range should block checkpoint advancement.

## 结课判定

Module 12 第 5 课理解检查全部通过，正式完成。

已经能够：
- 区分 Validation 与 Reconciliation。
- 理解 Reconciliation 前必须先确认 Grain / row meaning。
- 理解 Row Count 只能作为 coarse-grained signal，不能证明内容正确。
- 使用 Key-level Reconciliation 识别 Missing Keys 与 Unexpected Keys。
- 理解 Equal Row Count does not imply equal membership。
- 在 Grain 不同的上下游之间使用 Aggregate Reconciliation。
- 理解 Aggregate Reconciliation 前必须对齐 scope / filter / time boundary / canonical rule / business definition。
- 区分 Reconciliation 与 Invariant Check。
- 理解 Multi-sink Reconciliation 必须先做 processing boundary alignment。
- 理解 Reconciliation mismatch 是 symptom，而不是自动等于 root cause。
- 理解当前处理范围 Reconciliation Failure 必须阻止 Checkpoint 推进。