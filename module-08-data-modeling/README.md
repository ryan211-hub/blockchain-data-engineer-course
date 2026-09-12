# Module 8 — 数据建模

## 模块目标

从 Blockchain Data Engineer 视角，把 Indexer 已经产出的链上事实组织成稳定、可查询、可分析、可复用的数据模型，为后续 ETL、实时数据、分析数据库和业务分析打基础。

## Module Contract

### 必须掌握
- 链上数据建模为什么不能直接等同于 RPC / Indexer 原始结构
- Fact / Dimension 的基本划分
- Grain（粒度）与主键 / Unique Key 的设计
- Wallet、Transaction、Token、Transfer、Swap 等核心对象如何建模
- Source Identity 与 Business Identity 的区别
- Raw / Normalized Fact / DWS / ADS 等层级之间的数据职责边界
- 多链环境下 `chain_id + address` 的命名空间意识
- Token Metadata、Decimal、Symbol 等维度属性与事实表的关系
- 面向查询场景设计模型，而不是把所有字段堆进一张大表

### 可以了解
- Star Schema / Snowflake Schema
- SCD 等经典数仓维度技术
- 宽表与窄表的取舍
- 不同链之间的数据模型统一方式

### 本 Module 不展开
- 通用 ETL Framework → Module 9
- Kafka / Streaming → Module 10
- ClickHouse / DuckDB / Postgres 选型 → Module 11
- 系统性 Data Quality / Reorg Repair → Module 12
- DEX / Lending 等协议业务深挖 → Module 13 以后

### 结束标准

能够根据一个链上分析需求，先确定业务对象和 Grain，再设计 Fact / Dimension 表、主键与关联关系；能够解释为什么某字段属于事实、维度或更高层聚合；能够设计一个支持 Wallet / Token / Transfer / Swap 查询的最小链上数据模型。

## 当前学习进度

- 当前阶段：第三阶段 Data Engineering
- 当前 Module：Module 8 — 数据建模
- 已完成 Lesson：第 1 课｜为什么链上数据也需要数据建模：从 Indexer Fact 到可分析模型；第 2 课｜Grain：为什么“一行代表什么”是数据建模的第一原则；第 3 课｜Fact 与 Dimension：哪些数据是事实，哪些数据是维度；第 4 课｜Source Identity 与 Business Identity；第 5 课｜Raw / Normalized Fact / DWS / ADS：数据职责边界；第 6 课｜核心对象最小模型：Wallet / Token / Transfer / Swap
- 当前 Lesson：第 7 课｜Transaction 建模：Transaction Fact 与 Transfer / Swap 的 Grain 边界
- 当前状态：第 7 课理解检查中
- 已完成 Module：Module 1～7
- 下一步准确入口：继续 Module 8 第 7 课理解检查：Transaction / Transfer / Pool Swap Grain、transaction.value 语义、Transaction-level Measure 与 Mixed Grain。

## Lessons

- [第 1 课｜为什么链上数据也需要数据建模：从 Indexer Fact 到可分析模型](lesson-01-why-blockchain-data-modeling.md)
- [第 2 课｜Grain：为什么“一行代表什么”是数据建模的第一原则](lesson-02-grain.md)
- [第 3 课｜Fact 与 Dimension：哪些数据是事实，哪些数据是维度](lesson-03-fact-and-dimension.md)
- [第 4 课｜Source Identity 与 Business Identity](lesson-04-source-identity-and-business-identity.md)
- [第 5 课｜Raw / Normalized Fact / DWS / ADS：数据职责边界](lesson-05-raw-normalized-dws-ads.md)
- [第 6 课｜核心对象最小模型：Wallet / Token / Transfer / Swap](lesson-06-core-object-minimal-model.md)
- [第 7 课｜Transaction 建模：Transaction Fact 与 Transfer / Swap 的 Grain 边界](lesson-07-transaction-modeling.md)