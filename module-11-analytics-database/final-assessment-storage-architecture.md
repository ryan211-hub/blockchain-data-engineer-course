# Module 11 · 结束标准综合检查
## 设计 Blockchain Analytics Storage Architecture

### Assessment Contract

所属 Module：Module 11 — 分析数据库

本次综合检查不再教授新的数据库知识，目标是验证你是否已经达到 Module 11 的结束标准。

你需要能够综合回答：

1. 为什么一个 Blockchain Data Platform 不能只用一种数据库解决所有问题？
2. Postgres、ClickHouse、DuckDB 分别适合什么 workload？
3. 为什么 Column Store 适合大规模分析查询？
4. 为什么 Partition / Order Key 会显著影响大表查询性能？
5. 如何为 Transfer / Swap / Wallet Analytics 选择合理的分析存储设计？
6. 为什么 Realtime Serving 与 Historical Analytics 可能需要不同 Sink？
7. 如何在 Performance、Cost、Correctness、Complexity 之间做架构取舍？

本次检查重点是：

> 从 Workload 出发完成 Architecture Reasoning，而不是背数据库产品特性。

---

## 场景背景

你需要设计一个 Ethereum Wallet Analytics Platform。

上游：

```text
Ethereum RPC
    ↓
Indexer
    ↓
Kafka
```

系统有四类主要需求。

### Requirement A：Current Balance API

API：

```text
GET /wallet/{address}/balances
```

要求：

```text
P99 < 100ms
high QPS
latest state
frequent update
point lookup
```

---

### Requirement B：Historical Wallet Dashboard

Dashboard 需要查询：

```text
过去 5 年 Wallet Transfer History
Monthly Inflow / Outflow
Token Distribution
Protocol Activity
```

数据量：

```text
fact_wallet_activity
≈ 20 billion rows
```

典型查询：

```sql
SELECT
    toStartOfMonth(block_time) AS month,
    wallet_address,
    token_address,
    SUM(inflow_usd) AS inflow_usd,
    SUM(outflow_usd) AS outflow_usd
FROM fact_wallet_activity
WHERE block_time >= now() - INTERVAL 5 YEAR
  AND wallet_address = '0xabc'
GROUP BY
    month,
    wallet_address,
    token_address;
```

---

### Requirement C：Historical Archive / Recovery

系统要求长期保存：

```text
normalized transfer / swap / wallet activity
```

用途：

```text
Backfill
Replay
Decoder Bug Repair
Audit
Research
Reconciliation
```

Kafka Retention：

```text
7 days
```

但系统必须能够修复：

```text
past 1 year
past 3 years
甚至 full history
```

---

### Requirement D：Ad-hoc Investigation

Data Engineer 偶尔需要：

```text
分析 2 TB 历史数据
验证 Decoder Bug
比较 Backfill 前后结果
检查异常 Token / Wallet
```

这些查询：

```text
temporary
low concurrency
not production serving
```

---

# 综合问题

## 问题一｜Storage Role Design

请给下面四种技术分配角色：

```text
Postgres
ClickHouse
Parquet
DuckDB
```

要求说明：

- 哪个承担 Current State / API Serving？
- 哪个承担 Historical Analytics？
- 哪个承担 Long-term Historical Storage？
- 哪个承担 Ad-hoc / Validation？

并解释：

> 为什么不是把所有需求全部交给 Postgres 或全部交给 ClickHouse？

---

## 问题二｜Row Store vs Column Store

对于 Requirement B：

```text
20 billion rows
5-year historical scan
few columns
GROUP BY
SUM
```

请解释：

为什么这个 workload 更适合 Column-oriented Analytical Storage，而不是典型 Row Store？

至少从下面三个方面解释：

```text
Column Pruning
Compression
Scan / Aggregation
```

---

## 问题三｜Partition + Order Key Design

假设 ClickHouse 表：

```text
fact_wallet_activity
```

主要 Query Pattern：

```text
wallet_address
+
time range
```

请设计一个高层的：

```text
Partition Strategy
+
Order Key Strategy
```

并解释：

1. Partition 解决什么问题？
2. Order Key 解决什么问题？
3. Data Skipping 在这里是如何产生的？
4. 为什么不应该简单地按 wallet_address 做大量 Partition？

不要求给出唯一正确 DDL，重点是设计逻辑。

---

## 问题四｜Serving Path vs Analytics Path

请补全并解释下面架构：

```text
Ethereum RPC
    ↓
Indexer
    ↓
Kafka
    ├──────────────→ ?
    │                 ↓
    │              Postgres
    │                 ↓
    │                 ?
    │
    └──────────────→ ?
                      ↓
                  ClickHouse
                      ↓
                      ?
```

请说明：

- 两条 Path 分别解决什么 workload？
- 为什么它们应该有独立 Consumer Group？
- 为什么它们应该维护独立 Checkpoint？
- 两条 Path 短时间数据不完全一致，是否一定是错误？

---

## 问题五｜Recovery Architecture

假设：

```text
Decoder Bug affected:
2026-01 → 2026-06

Kafka Retention:
7 days
```

ClickHouse 中这 6 个月的数据全部需要重建。

请设计 Recovery Flow。

至少包含：

```text
Parquet
Backfill / Replay
Validation
ClickHouse
```

并回答：

> 为什么 Parquet 在这里不只是“便宜的冷存储”，还是 Recovery Asset？

---

## 问题六｜Architecture Trade-off

假设当前系统规模还很小：

```text
10 million rows
3 internal users
simple dashboard
Postgres query latency acceptable
no API SLA issue
```

团队有人提出：

```text
立刻加入 Kafka
ClickHouse
Parquet
DuckDB
```

你是否认为现在必须拆成完整混合架构？

请从下面几个角度分析：

```text
Performance
Cost
Operational Complexity
Team Size
Workload
Future Growth
```

这里不要求回答单纯的“是 / 否”。

重点是：

> 你是否能解释什么时候 Architecture Complexity 值得引入。

---

## 问题七｜最终系统设计

最后，请你独立给出一个你认为合理的 Blockchain Analytics Storage Architecture。

至少包含：

```text
RPC / Node
Indexer
Kafka
Postgres
ClickHouse
Parquet
DuckDB
API
Dashboard
Backfill / Replay
```

可以使用文字或 ASCII 架构图。

然后用几句话解释每一个组件承担的职责。

---

# 通过标准

如果你能够完成以上问题，并体现出以下思维：

```text
Workload
→ Query Pattern
→ Storage Layout
→ Engine Choice
→ Pipeline
→ SLA
→ Cost / Complexity Trade-off
```

就说明 Module 11 已经达到结束标准。

这次综合检查最重要的不是术语完整，而是：

> 能不能从业务查询需求反推出数据架构，而不是先选数据库再找使用场景。