# Module 12 — 数据质量

## Module Contract

### Module 目标

把前面已经掌握的 Indexer、ETL、Streaming、Backfill、Reorg 与多 Sink 架构，统一到一个核心问题：

> How do we know blockchain data is correct, and how do we repair it when it is not?

本 Module 不把“数据质量”理解成几条 SQL 校验规则，而是建立 Blockchain Data Engineer 需要掌握的完整 Quality / Detection / Repair 心智模型。

### 必须掌握

- Data Quality 的几个核心维度：Completeness、Accuracy、Consistency、Freshness、Uniqueness、Validity。
- Source Quality 与 Derived Data Quality 的区别。
- Blockchain 特有的数据质量问题：Reorg、Late-arriving、Missing Block / Log、Decoder Bug、Provider Gap、Duplicate Delivery。
- Validation、Reconciliation、Invariant Check 的区别。
- Row Count 为什么经常不够，以及何时应使用 Aggregate Reconciliation。
- 如何检测 Missing Range、Duplicate、Unexpected Gap。
- 如何设计 Reorg Detection、Rollback、Replay 与 Correction。
- 如何使用 Backfill / Replay 修复历史错误。
- 如何在多 Sink 系统中发现并修复 Postgres / ClickHouse / Parquet 之间的不一致。
- 为什么 Data Quality Job 失败时不能推进相关 Checkpoint。
- 如何设计一套基础 Blockchain Data Quality & Repair Framework。

### 可以了解

- Schema / Contract Testing。
- Anomaly Detection 的基础思路。
- Data Observability 与 Quality Metrics。
- Quarantine / Dead-letter Data 的高层用途。
- Golden Dataset / Reference Dataset 的概念。

### 本 Module 不展开

- 复杂机器学习异常检测。
- 完整 Data Governance / Data Catalog 产品体系。
- 分布式事务一致性协议。
- 商业 Data Observability 产品采购。
- Protocol-level 共识证明与形式化验证。

### 结束标准

完成本 Module 后，应能回答：

1. Blockchain Data Quality 为什么不仅是“SQL 跑成功”？
2. Completeness / Accuracy / Consistency / Freshness / Uniqueness 分别如何落到链上数据？
3. 如何检测 Missing Block / Log、Duplicate、Provider Gap 与 Decoder Bug？
4. Row Count、Aggregate Reconciliation、Invariant Check 各适合什么场景？
5. Reorg 如何影响 Fact、State、DWS / ADS 与多 Sink？
6. 如何安全地执行 Backfill / Replay / Historical Repair？
7. 为什么 Validation Failure 不应推进 Checkpoint？
8. 如何设计一套可检测、可追溯、可重放、可修复的 Blockchain Data Quality Pipeline？

## 课程结构

1. 第 1 课｜数据质量到底是什么：为什么 Pipeline 成功不代表数据正确
2. 第 2 课｜Completeness：Missing Block、Missing Log 与 Provider Gap
3. 第 3 课｜Uniqueness & Idempotency：Duplicate Delivery 与重复事实
4. 第 4 课｜Accuracy & Decoder Quality：解析正确不等于业务语义正确
5. 第 5 课｜Reconciliation：Row Count、Aggregate Check 与 Invariant
6. 第 6 课｜Freshness & Lag：数据正确但太晚也可能不可用
7. 第 7 课｜Reorg Quality：Canonical / Orphan、Rollback 与 Replay
8. 第 8 课｜Historical Repair：Backfill、Replay 与多 Sink 修复
9. 第 9 课｜Data Quality Framework：检测、告警、阻断、修复、审计
10. 结束标准综合检查｜设计 Blockchain Data Quality & Repair Pipeline

## 当前学习进度

- 当前阶段：第三阶段 Data Engineering
- 当前 Module：Module 12 — 数据质量
- 已完成 Lesson：第 1 课｜数据质量到底是什么：为什么 Pipeline 成功不代表数据正确；第 2 课｜Completeness：Missing Block、Missing Log 与 Provider Gap；第 3 课｜Uniqueness & Idempotency：Duplicate Delivery 与重复事实；第 4 课｜Accuracy & Decoder Quality：解析正确不等于业务语义正确；第 5 课｜Reconciliation：Row Count、Aggregate Check 与 Invariant；第 6 课｜Freshness & Lag：数据正确但太晚也可能不可用；第 7 课｜Reorg Quality：Canonical / Orphan、Rollback 与 Replay
- 当前 Lesson：第 8 课｜Historical Repair：Backfill、Replay 与多 Sink 修复
- 当前状态：第 7 课已完成；下一步进入第 8 课
- 下一步准确入口：开始 Module 12 第 8 课｜Historical Repair：Backfill、Replay 与多 Sink 修复。

## 课程路径

- [第 1 课｜数据质量到底是什么：为什么 Pipeline 成功不代表数据正确](lesson-01-data-quality-foundation.md)
- [第 2 课｜Completeness：Missing Block、Missing Log 与 Provider Gap](lesson-02-completeness.md)
- [第 3 课｜Uniqueness & Idempotency：Duplicate Delivery 与重复事实](lesson-03-uniqueness-idempotency.md)
- [第 4 课｜Accuracy & Decoder Quality：解析正确不等于业务语义正确](lesson-04-accuracy-decoder-quality.md)
- [第 5 课｜Reconciliation：Row Count、Aggregate Check 与 Invariant](lesson-05-reconciliation.md)
- [第 6 课｜Freshness & Lag：数据正确但太晚也可能不可用](lesson-06-freshness-lag.md)
- [第 7 课｜Reorg Quality：Canonical / Orphan、Rollback 与 Replay](lesson-07-reorg-quality.md)