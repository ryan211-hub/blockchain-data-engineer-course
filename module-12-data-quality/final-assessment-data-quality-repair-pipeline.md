# Module 12 结束标准综合检查｜设计 Blockchain Data Quality & Repair Pipeline

## Assessment Contract

所属 Module：Module 12 — 数据质量

本次不是新课，而是 Module 12 的综合验收。

核心问题：

> Can you design a Blockchain Data Quality & Repair Pipeline that can detect problems, block unsafe progress, repair historical data, and verify that all sinks have converged?

你不需要追求“标准架构图”，重点是能否把前 9 课的知识连成一个完整 operational model。

---

## 一、场景

你负责 Ethereum 的链上数据平台。

当前架构：

```text
Ethereum RPC
↓
Indexer
↓
Kafka
↓
Consumer
↓
Postgres Fact
↓
DWS
↓
ADS
↓
Dashboard
```

同时还有：

```text
Postgres Fact
↓
ClickHouse

Raw / Historical Data
↓
Parquet
```

也就是说，主要 Sink 包括：

```text
Postgres
ClickHouse
Parquet
```

其中：

- RPC（Remote Procedure Call，远程过程调用）用于获取链上数据；
- DWS（Data Warehouse Summary，数据仓库汇总层）负责汇总；
- ADS（Application Data Service，应用数据服务层）面向产品 / Dashboard；
- DQ（Data Quality，数据质量）Framework 负责检测和控制数据质量。

---

## 二、突然出现一个生产事故

系统当前：

```text
chain head = 21,500,000
realtime checkpoint = 21,499,980
```

你发现：

```text
block 21,400,000 → 21,420,000
```

存在 Decoder Bug。

Bug 内容：

```text
ERC-20 amount_raw
```

解析正确，

但是：

```text
decimals
```

读取错误。

因此：

```text
amount
```

被错误放大 1000 倍。

Bug 已经持续两天。

---

## 三、当前影响范围

进一步调查后发现：

```text
Raw Logs
→ correct

Postgres Fact
→ wrong amount

ClickHouse Fact
→ wrong amount

Parquet decoded history
→ wrong amount

DWS
→ wrong aggregate

ADS
→ wrong aggregate

Dashboard
→ wrong
```

Realtime 新数据已经部署新 Decoder，因此：

```text
new blocks
→ correct
```

但历史两天仍然错误。

---

## 四、同时发现另一个问题

你在检查过程中发现：

```text
Postgres checkpoint = 21,499,980
ClickHouse checkpoint = 21,499,500
```

ClickHouse 明显比 Postgres 落后。

但两边共同范围：

```text
<= 21,499,500
```

的数据，在修复前本身逻辑一致。

---

# 综合问题

## 问题一｜Problem Classification

请先对问题分类。

1. Decoder Bug 最直接属于哪个 Data Quality Dimension？
2. `amount_raw` 正确、`amount` 错误，说明 Source Quality 和 Derived Data Quality 分别是什么状态？
3. ClickHouse 比 Postgres 少处理 480 blocks，这首先属于 Accuracy 问题，还是 Freshness / Processing Progress 问题？

---

## 问题二｜Checkpoint Decision

当前 Realtime 新 Block：

```text
21,499,981
```

已经使用修复后的 Decoder，并且：

```text
Completeness = PASS
Accuracy = PASS
Uniqueness = PASS
Reconciliation = PASS
```

但历史：

```text
21,400,000 → 21,420,000
```

仍然错误。

请回答：

1. 是否应该把 Realtime Checkpoint 回退到 `21,400,000`？
2. Realtime Checkpoint 是否可以继续向前推进？
3. 为什么 Historical Repair 和 Realtime Processing 应该维护独立状态？

---

## 问题三｜Repair Scope

请为这次事故定义一个合理的 Repair Scope。

至少说明：

```text
chain
block range
affected contract / event
decoder version
affected datasets
affected sinks
downstream dependencies
```

不需要写具体字段值，但要说明哪些维度必须被限定。

---

## 问题四｜Repair Source

你现在有：

```text
Raw Logs = correct
Kafka retention = 7 days
Bug happened 2 days ago
Parquet decoded history = wrong
```

请回答：

1. 你优先选择什么作为 Repair Source？
2. 是否必须重新访问 RPC？
3. 是否必须经过 Kafka Replay？
4. 为什么？

---

## 问题五｜Repair Write Semantics

Postgres Fact 中已有错误行：

```text
unique key
=
chain_id + tx_hash + log_index
```

现在重新计算得到相同 key、正确 amount。

请回答：

1. 为什么 `ON CONFLICT DO NOTHING` 不合适？
2. 更合适的写入方式是什么？
3. 为什么 Repair 仍然必须满足 Idempotency？

---

## 问题六｜Downstream Repair

假设 Postgres Fact 已经修复。

请回答：

1. 能不能立刻宣布 Incident Resolved？
2. DWS / ADS 应该怎么处理？
3. ClickHouse 和 Parquet 应该怎么处理？
4. 为什么修复顺序通常应该是：

```text
source-near layer
↓
Fact
↓
DWS
↓
ADS
↓
Serving
```

---

## 问题七｜Reconciliation

修复完成后，你准备做：

```text
Postgres vs ClickHouse
```

Reconciliation。

但：

```text
Postgres checkpoint = 21,499,980
ClickHouse checkpoint = 21,499,500
```

请回答：

1. 能不能直接比较两边所有最新数据？
2. 应该选择什么共同 Processing Boundary？
3. 为什么不对齐 Boundary 会产生 false mismatch？

---

## 问题八｜DQ Framework Decision

假设 Historical Repair 完成后：

```text
Postgres = VERIFIED
ClickHouse = VERIFIED
Parquet = VERIFIED
DWS = VERIFIED
ADS = VERIFIED
```

但此时：

```text
Freshness SLO FAIL
block_lag = 600
```

这里 SLO（Service Level Objective，服务水平目标）表示数据时效目标。

请回答：

1. 是否一定要停止 Checkpoint？
2. 这个 Freshness Failure 更适合：
   - BLOCK
   - WARN / ALERT
   - QUARANTINE
   中的哪一种？
3. 为什么 Freshness Failure 和 Reconciliation Failure 的 Blocking Policy 不一定相同？

---

## 问题九｜Incident Closure

一个完整 Incident 在关闭前，至少应该确认哪些条件？

请从下面几个角度回答：

```text
Root Cause
Blast Radius
Repair
Validation
Reconciliation
Multi-sink Status
Audit
```

最终说明：

> What conditions must be true before `Incident = CLOSED`?

---

## 问题十｜系统设计总结

最后请你用自己的方式，把整个 Blockchain Data Quality & Repair Pipeline 串起来。

可以参考这个骨架，但不要求完全一样：

```text
Detect
↓
Classify
↓
Decide
↓
Contain
↓
Repair
↓
Rebuild
↓
Reconcile
↓
Verify
↓
Resume
↓
Audit
```

你需要说明：

- 哪些失败应该 Block Checkpoint；
- 哪些失败可以继续 Processing；
- Reorg、Decoder Bug、Missing Data、Duplicate、Freshness 分别如何进入这套 Framework；
- Historical Repair 为什么要独立于 Realtime Pipeline。

---

## 通过标准

如果你能比较稳定地回答以下几个核心点，就可以认为 Module 12 已达到结束标准：

```text
1. 能区分不同 Data Quality Dimensions
2. 能判断 Checkpoint 是否应该推进
3. 能设计 Reorg Correction
4. 能设计 Historical Repair
5. 能处理 Multi-sink Divergence
6. 能设计 Validation / Reconciliation
7. 能区分 Blocking vs Non-blocking Failure
8. 能设计完整 Incident → Repair → Verification 流程
```

这次综合检查你可以一次全部回答，也可以分几次回答。