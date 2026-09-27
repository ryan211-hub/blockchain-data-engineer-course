# Module 11 · 第 7 课
## Serving DB + Analytics DB：混合存储架构

### Lesson Contract

所属 Module：Module 11 — 分析数据库

本课核心问题：

> 一个 Blockchain Data Platform 为什么往往需要不止一种数据库？不同数据库之间到底应该怎样分工，才能同时满足低延迟 Serving 与大规模 Historical Analytics？

学完以后，你应该能够解释：

- 什么是 Serving DB，什么是 Analytics DB。
- 为什么 Current State / API Serving 和 Historical Analytics 往往不应压在同一个数据库角色上。
- 为什么 Postgres + ClickHouse 是一种典型的 Role Separation。
- 为什么 Parquet / DuckDB 可以作为补充，而不是另一个在线 Serving Sink。
- 同一份业务事实为什么可能流向多个存储引擎。
- 如何设计一条从 Indexer / Kafka 到不同 Sink 的数据路径。
- 如何理解 Freshness、Correctness、Latency、Cost、Complexity 之间的权衡。

本课不深入 CDC 内部实现、双写事务一致性协议、分布式事务、ClickHouse replication internals；只建立 Blockchain Data Engineer 需要掌握的混合存储架构模型。

---

## 一、先把前六课串起来

前面六课其实一直在回答同一个问题：

> Different workloads need different storage engines.

我们已经得到：

```text
Postgres
→ Serving / Operational

ClickHouse
→ Online Historical Analytics

DuckDB
→ Local / Ad-hoc Analytics

Parquet
→ Historical Analytical Storage
```

现在的问题是：

> 如果一个真实 Blockchain Data Platform 同时需要这些能力，应该怎么组合？

答案通常不是：

```text
Choose one database
```

而是：

```text
Use multiple engines with clear roles
```

---

## 二、什么叫 Serving DB？

[Data Engineer 视角]

Serving DB 的核心目标是：

> Serve application queries with predictable low latency.

典型查询：

```sql
SELECT balance
FROM wallet_token_balance
WHERE chain_id = 1
  AND wallet_address = '0xabc'
  AND token_address = 'USDC';
```

特点：

```text
small result
point lookup
current state
low latency
high QPS
frequent updates
```

这种 workload 更关心：

```text
latency
availability
predictability
transactional update
```

Postgres 很适合承担这类角色。

---

## 三、什么叫 Analytics DB？

Analytics DB 的核心目标是：

> Scan and aggregate large historical datasets efficiently.

典型查询：

```sql
SELECT
    toDate(block_time) AS day,
    token_address,
    SUM(amount_usd) AS volume_usd
FROM fact_token_transfer
WHERE block_time >= now() - INTERVAL 365 DAY
GROUP BY
    day,
    token_address;
```

特点：

```text
many rows
few columns
historical
aggregation
large scan
high analytical concurrency
```

它更关心：

```text
scan throughput
compression
aggregation performance
analytical concurrency
```

ClickHouse 更适合承担这类角色。

---

## 四、为什么不让一个数据库全做？

技术上可以。

但问题是：

> Can one engine serve both workloads efficiently and predictably at the same time?

假设同一个 Postgres 同时承担：

```text
Wallet Balance API
+
Historical Dashboard
```

API 要求：

```text
P99 < 100ms
```

Dashboard 同时在跑：

```text
scan billions of rows
GROUP BY
SUM
COUNT
```

两者会竞争：

```text
CPU
Memory
Disk I/O
Buffer Cache
Workers
```

于是：

```text
heavy analytics
↓
resource contention
↓
serving latency unstable
```

这就是为什么要做：

> Workload Isolation.

---

## 五、混合存储架构的核心不是“多数据库”

如果只是：

```text
Postgres
ClickHouse
DuckDB
Parquet
```

全都上了，并不代表架构好。

真正关键的是：

> Every engine must have a clear workload responsibility.

例如：

```text
Postgres
→ current state
→ operational metadata
→ API serving

ClickHouse
→ historical facts
→ dashboard
→ analytical aggregation

Parquet
→ cold history
→ archive
→ backfill output

DuckDB
→ ad-hoc validation
→ local analysis
```

所以混合架构的核心是：

> Role Separation.

不是：

> Tool Collection.

---

## 六、一个典型 Blockchain Data Platform

```text
Ethereum RPC / Node
        ↓
      Indexer
        ↓
       Kafka
        ↓
   Parsed Events
        ↓
 ┌──────────────┬──────────────┬──────────────┐
 ↓              ↓              ↓
Postgres     ClickHouse      Parquet
 ↓              ↓              ↓
Serving      Analytics      Archive
Current      Historical     Backfill
State        Facts          Research
API          Dashboard         ↓
                              DuckDB
```

这里 Kafka 之后可以有多个 Consumer Group：

```text
Serving Consumer Group
→ Postgres

Analytics Consumer Group
→ ClickHouse

Archive Consumer / Batch Job
→ Parquet
```

这就是我们 Module 10 学过的 Consumer Group 在这里的实际落地。

---

## 七、同一条 Event 为什么可以进入多个 Sink？

例如一个 ERC-20 Transfer：

```text
Alice
→ Bob
100 USDC
```

对不同系统角色，它有不同价值。

Postgres 可能更新：

```text
wallet_token_balance
```

ClickHouse 可能追加：

```text
fact_token_transfer
```

Parquet 可能长期归档：

```text
normalized_transfer_history
```

因此：

> One business event can feed multiple physical representations.

注意：

这不等于“完全复制所有表”。

而是：

> Same event, different data products for different workloads.

---

## 八、这里要区分 Fact 和 State

这是 Blockchain 场景里很重要的一层。

Transfer Event 本身是：

```text
Fact
```

例如：

```text
Alice sent 100 USDC to Bob
```

而余额：

```text
Alice current USDC balance = 500
```

是：

```text
Derived Current State
```

所以：

```text
Historical Fact
→ ClickHouse

Derived Current State
→ Postgres
```

这是一种很自然的角色分工。

---

## 九、为什么 Postgres 保存 Current State 很自然？

假设 Wallet API：

```text
GET /wallet/0xabc/balance
```

如果每次请求都去：

```text
scan all historical transfers
```

再计算余额：

当然不合理。

所以系统会维护：

```text
wallet_token_balance
```

每有新 Transfer：

```text
update balance
```

查询时：

```text
point lookup
```

这就是：

> Materialized Current State.

Postgres 非常适合这种模式。

---

## 十、为什么 ClickHouse 保存 Historical Fact 很自然？

Historical Fact：

```text
append-heavy
large volume
time-oriented
aggregation-heavy
```

例如：

```text
fact_transfer
fact_swap
fact_nft_transfer
```

Dashboard 常问：

```text
past 30 days
past 1 year
group by token
group by protocol
group by wallet cohort
```

这就是：

> Analytical Fact Store.

ClickHouse 很适合。

---

## 十一、Serving Path 和 Analytics Path

可以把系统拆成两条 Path。

### Serving Path

```text
Indexer / Kafka
      ↓
Serving Consumer
      ↓
Postgres
      ↓
API
```

目标：

```text
fresh
low latency
predictable
```

### Analytics Path

```text
Indexer / Kafka
      ↓
Analytics Consumer
      ↓
ClickHouse
      ↓
Dashboard / Analyst
```

目标：

```text
large-scale scan
aggregation
historical analysis
```

这两个 Path 的 SLA 本来就不同。

---

## 十二、这和 Module 10 的 Streaming Path 有什么关系？

我们之前讲：

> Streaming Path = fresh but mutable.

最新 Block 刚进来时：

```text
fresh
but may still reorg
```

因此 Serving / Analytics 两条路径都可能先收到：

```text
near-head data
```

之后还需要：

```text
Reorg Correction
```

所以多 Sink 并不意味着：

> 每个 Sink 自己随便处理 Reorg。

更合理的是：

```text
shared canonical processing logic
+
sink-specific correction strategy
```

---

## 十三、Reorg 对两个 Sink 的处理可能不同

例如 Reorg 撤销一个 Transfer。

Postgres Current State：

```text
rollback / recompute balance
```

ClickHouse Historical Fact：

```text
mark old row non-canonical
or insert corrected version
or rebuild affected aggregate
```

也就是说：

> Same chain correction, different sink semantics.

所以：

```text
Correction Logic
```

和：

```text
Storage Operation
```

不是完全一回事。

---

## 十四、一个很重要的问题：谁是 Source of Truth？

如果 Postgres 和 ClickHouse 数据不一致怎么办？

这里要建立一个原则：

> Serving DB and Analytics DB should not become independent business truths.

更合理的是：

```text
Blockchain / Canonical Parsed Fact
        ↓
Shared Processing Logic
        ↓
Multiple Derived Sinks
```

也就是说：

```text
Postgres
ClickHouse
```

都是：

> Derived Data Products.

而不是相互作为权威来源。

---

## 十五、不要从 Postgres 再推导所有 ClickHouse 数据

一种架构是：

```text
Indexer
↓
Postgres
↓
ETL
↓
ClickHouse
```

这在某些系统完全可以。

但如果你的实时链路已经有 Kafka：

```text
Indexer
↓
Kafka
├→ Postgres
└→ ClickHouse
```

会更容易：

```text
decouple workloads
independent scaling
independent retries
independent consumer lag
```

这也是 Kafka 的价值：

> Fan-out + decoupling.

---

## 十六、但也不要为了“架构漂亮”强行多 Sink

如果系统很小：

```text
1 million rows
5 users
simple dashboard
```

可能：

```text
Postgres only
```

就够。

加入：

```text
Kafka
ClickHouse
Parquet
DuckDB
```

可能只是增加：

```text
deployment
monitoring
consistency
failure modes
on-call burden
```

所以还是：

> Architecture complexity must be justified by workload.

---

## 十七、混合架构最大的代价：数据一致性复杂度

多 Sink 以后会出现：

```text
Postgres succeeded
ClickHouse failed
```

或者：

```text
ClickHouse lag = 2 minutes
Postgres lag = 5 seconds
```

或者：

```text
Reorg correction applied to Postgres
but ClickHouse correction delayed
```

所以你必须接受：

> Different sinks can be temporarily inconsistent.

这不是一定代表系统设计失败。

关键是：

```text
Can we detect it?
Can we reconcile it?
Can we repair it?
```

---

## 十八、每个 Sink 应该有自己的 Checkpoint

这个和 Module 9 / 10 可以直接连接。

例如：

```text
Postgres Consumer Checkpoint
= block 22,000,100

ClickHouse Consumer Checkpoint
= block 22,000,080
```

这不一定错误。

只是：

```text
Postgres path is ahead by 20 blocks
```

所以：

> Processing State belongs to each pipeline / sink.

不要强行共用一个 Checkpoint。

---

## 十九、Freshness 不一定要完全一样

Serving API 可能要求：

```text
lag < 2 blocks
```

Dashboard 可能允许：

```text
lag < 1 minute
```

Archive Parquet 可能：

```text
hourly / daily
```

因此：

```text
Same source
≠
Same freshness SLA
```

这也是多路径设计的重要理由。

---

## 二十、Consistency 也要区分“最终一致”与“实时完全一致”

假设：

```text
Postgres
→ latest balance

ClickHouse
→ historical transfer dashboard
```

两者短时间差几十秒：

可能是可接受的。

如果系统要求：

```text
every sink must commit atomically
```

复杂度会急剧上升。

大部分数据平台更常见的是：

> Eventual Consistency + Reconciliation.

即：

```text
independent processing
+
idempotency
+
checkpoint
+
replay
+
reconciliation
```

这正好和前面 Module 9 / 10 的知识连起来。

---

## 二十一、Idempotency 为什么再次重要？

Kafka 是 At-least-once。

Consumer 可能重复收到：

```text
same Transfer
```

Postgres：

```text
UPSERT / unique key
```

ClickHouse：

需要设计：

```text
dedup / version / idempotent load semantics
```

核心不是：

> Kafka 永远只发一次。

而是：

> Each sink can safely process duplicate delivery.

---

## 二十二、Parquet 在混合架构里是什么角色？

Parquet 通常不是：

```text
online serving sink
```

而更像：

```text
historical storage
cold archive
backfill output
replay source
validation dataset
```

例如：

```text
Kafka / Batch
      ↓
Parquet
      ↓
Object Storage
```

需要时：

```text
DuckDB
Spark
ClickHouse reload
```

都可以读取。

---

## 二十三、为什么保留 Parquet 可以提高可恢复性？

假设 ClickHouse 因 Bug 需要重建。

如果你只有：

```text
live Kafka retention = 7 days
```

但 Bug 影响：

```text
past 6 months
```

Kafka 已经不够。

如果你有长期 Parquet：

```text
historical normalized facts
```

就可以：

```text
Parquet
↓
Backfill / Replay
↓
Rebuild ClickHouse
```

所以 Parquet 不只是省钱。

它也是：

> Recovery Asset.

---

## 二十四、一个典型 Hot / Warm / Cold 架构

可以进一步分层：

```text
Hot
→ Postgres
→ latest/current serving

Warm
→ ClickHouse
→ recent + long historical analytics

Cold
→ Parquet/Object Storage
→ archive/replay/research
```

注意：

这不是固定标准。

只是一个常见思维框架。

---

## 二十五、银行系统类比

你可以把它类比成：

```text
Core / Online DB
→ 当前业务状态
→ 高频查询

Data Warehouse
→ 历史明细
→ 报表分析

Archive
→ 长期历史
→ 审计 / 追溯
```

Blockchain 平台：

```text
Postgres
→ Serving

ClickHouse
→ Analytics

Parquet
→ Archive

DuckDB
→ Ad-hoc Tool
```

本质上还是：

> Operational workload and analytical workload should be separated when scale justifies it.

---

## 二十六、设计一个 Wallet Analytics Platform

需求：

### A. Current Balance API

```text
P99 < 100ms
```

### B. 5 年 Wallet Flow Dashboard

```text
billions of activities
```

### C. 历史修复与 Backfill

```text
occasionally replay years of data
```

一个合理架构：

```text
Indexer
  ↓
Kafka
  ├───────────────┬────────────────┐
  ↓               ↓                ↓
Serving        Analytics         Archive
Consumer       Consumer          Pipeline
  ↓               ↓                ↓
Postgres       ClickHouse        Parquet
  ↓               ↓                ↓
Balance API    Dashboard          DuckDB
                                  Backfill
                                  Validation
```

这里没有任何一个数据库“赢了”。

每个引擎只是做它最适合的工作。

---

## 二十七、CTO 视角：为什么不追求“一库通吃”？

因为 CTO 真正关心的不是：

> Which database is coolest?

而是：

```text
Can we meet SLA?
Can we scale?
Can we recover?
Can we operate it?
What does it cost?
```

所以选型本质是：

```text
performance
+
correctness
+
operability
+
cost
+
team complexity
```

的综合权衡。

---

## 二十八、什么时候应该开始拆 Serving / Analytics？

常见信号：

```text
analytics starts hurting API latency
historical scan volume grows rapidly
dashboard concurrency rises
storage cost grows
serving SLA becomes unstable
different freshness requirements emerge
different retention requirements emerge
```

这时：

> Role Separation becomes valuable.

---

## 二十九、什么时候不应该拆？

如果：

```text
small data
low concurrency
simple workload
Postgres meets SLA
team is small
operational simplicity matters more
```

继续：

```text
Postgres only
```

可能更好。

架构演进应该是：

> driven by pain, not fashion.

---

## 三十、把 Module 11 整体串起来

第 1 课：

```text
OLTP vs OLAP
→ Workload
```

第 2 课：

```text
Row vs Column
→ Physical Storage
```

第 3 课：

```text
Postgres Boundary
→ Serving limits
```

第 4 课：

```text
ClickHouse
→ Analytical Engine
```

第 5 课：

```text
Partition / Order / Skipping
→ Physical Optimization
```

第 6 课：

```text
DuckDB + Parquet
→ File Analytics / Cold Data
```

第 7 课：

```text
Hybrid Architecture
→ Put everything together
```

这就形成了完整的：

> Blockchain Analytics Storage Architecture.

---

## 本课核心结论

> A Serving DB and an Analytics DB solve different workload problems.

> Postgres is commonly used for current state, operational metadata, point lookup, and low-latency API serving.

> ClickHouse is commonly used for historical facts, large scans, aggregation, and online analytical serving.

> Parquet provides low-cost historical storage and replay/recovery capability; DuckDB provides on-demand file analytics.

> Multiple sinks introduce temporary inconsistency and operational complexity, so each path needs independent checkpointing, idempotency, monitoring, replay, and reconciliation.

> The goal of hybrid architecture is not “use more databases”; it is “assign each workload to the right engine”.

---

## 理解检查

### 问题一

一个 Wallet 产品同时需要：

```text
A. Current Balance API
P99 < 100ms

B. 过去 5 年 Wallet Flow Dashboard
需要扫描几十亿行
```

为什么把 A 和 B 都长期压在同一个 Postgres 上可能产生架构风险？

你会怎样分配 Postgres 和 ClickHouse 的角色？

### 问题二

假设：

```text
Postgres Consumer
checkpoint = block 22,000,100

ClickHouse Consumer
checkpoint = block 22,000,080
```

这是否一定说明数据管道出错？

为什么两个 Sink 应该维护独立的 Checkpoint？

### 问题三

如果 ClickHouse 中过去 6 个月的数据因为 Decoder Bug 需要全部重建，但 Kafka Retention 只有 7 天：

为什么长期保存的 Parquet Historical Facts 在这里很重要？

请把下面这条恢复链路补完整：

```text
Parquet
→ ?
→ ?
→ ClickHouse
```

## 用户回答

问题一：A、B 都长期压在同一个 PostgreSQL 上，B 的产品有可能会对 A 产生影响。因为 B 需要扫描几十亿行数据，这个 dashboard 的占用资源会比较高。可以把产品 A 分配给 Postgres，产品 B 分配给 ClickHouse

问题二：Postgres Consumer 和 ClickHouse Consumer 它们是各自独立的。它们各自维护自己的 checkpoint 进度，所以它们不同步并不代表数据管道出错。正是因为它们有各自独立的进度情况，所以需要单独维护自己的 checkpoint

问题三：这个时候正好可以使用 DuckDB 直接查询 Parquet 历史数据，得到恢复的结果导入到 ClickHouse。
所以这里完整链路是：
Parquet 
→ DuckDB 
→ Compare aggregates  
→ ClickHouse

## 老师判断与校准

三题全部通过。

- 问题一：回答正确。把 Current Balance API 和 5 年 Historical Dashboard 都长期压在同一个 Postgres 上，会产生 Resource Contention 风险。Dashboard 的 large scan / aggregation 会竞争 CPU、Memory、Disk I/O、Buffer Cache 等资源，从而让 Serving Latency 和 SLA 变得不稳定。合理分工是 Postgres 承担 Current State / API Serving，ClickHouse 承担 Historical Facts / Dashboard / Aggregation。
- 问题二：回答正确。Postgres Consumer 与 ClickHouse Consumer 是两个独立的 processing path / sink，它们有各自的 lag、failure、retry、replay 和 freshness SLA，因此应维护独立 Checkpoint。Checkpoint 不一致只说明处理进度不同，不等于 Pipeline 出错。
- 问题三：思路正确。Parquet Historical Facts 在 Kafka Retention 不足时可以作为长期 Recovery Asset。你写的 DuckDB + Compare aggregates 很适合作为 Validation / Reconciliation 步骤，但恢复主链路更适合表达为：

```text
Parquet
→ Backfill / Replay
→ Rebuild / Load
→ ClickHouse
```

DuckDB 可以插在 Backfill / Replay 之后或 Load 之前，用于 Row Count、Aggregate Reconciliation、Distribution Check 等校验。

## 结课判定

Module 11 第 7 课理解检查全部通过，正式完成。已经能够设计 Serving DB + Analytics DB 的基础混合存储架构，并能解释 Workload Isolation、独立 Checkpoint、Eventual Consistency、Recovery Asset 与多 Sink Role Separation。
