# Module 9 — ETL

## 模块目标

从 Blockchain Data Engineer 视角，把已经存在的链上 Raw / Normalized Fact / Dimension 模型组织成可持续运行、可重跑、可恢复、可回填的数据管道，理解传统 ETL 与 Crypto ETL 的共同点和特殊约束。

## Module Contract

### 必须掌握
- ETL / ELT 的核心含义，以及 Extract / Transform / Load 各自负责什么
- 数据管道为什么不能只是“一次性脚本”
- Source → Staging / Raw → Transform → Target 的基本数据流
- Full Load / Incremental Load / Backfill 的区别
- Watermark / Cursor / Checkpoint 在 ETL 中的角色
- 幂等、重复运行、失败恢复与断点续跑
- Late-arriving / Reorg 等链上数据对 ETL 的特殊影响
- Schema / Grain / Unique Key 如何约束 Transform 与 Load
- 批处理 Job 的输入范围、输出范围和可验证性

### 可以了解
- ETL 与 ELT 的工程取舍
- DAG / Dependency 的基础概念
- Partition-based processing
- Scheduler / Orchestrator 的基本职责

### 本 Module 不展开
- Kafka / Streaming 实时管道 → Module 10
- ClickHouse / DuckDB / Postgres 选型 → Module 11
- 系统性 Data Quality / Repair Framework → Module 12
- 协议业务深挖 → Module 13 以后

### 结束标准

能够针对一个链上数据模型设计批处理数据管道：明确 Source、Transform、Target、处理范围、增量边界、Checkpoint / Cursor、幂等策略、Backfill 与失败恢复方式；能够解释传统 ETL 与 Blockchain ETL 的共同结构和链上特殊约束。

## 当前学习进度

- 当前阶段：第三阶段 Data Engineering
- 当前 Module：Module 9 — ETL
- 当前 Lesson：第 1 课｜ETL 到底是什么：从“已有数据”到“可持续数据管道”
- 当前状态：第 1 课已完成；Module 9 继续进行中
- 已完成 Lesson：第 1 课｜ETL 到底是什么：从“已有数据”到“可持续数据管道”
- 已完成 Module：Module 1～8
- 最近掌握重点：Extract = Source + Input Range；Transform 将 Transfer Event Grain 汇总为 wallet + token + date Grain；Load 按 Target Grain / Unique Key 保证幂等；Backfill 按独立日期范围执行，Checkpoint 只记录最后完整成功的处理边界。
- 下一步准确入口：规划并开始 Module 9 第 2 课（按 Module Contract 继续学习 ETL）。

## Lessons

- [第 1 课｜ETL 到底是什么：从“已有数据”到“可持续数据管道”](lesson-01-what-is-etl.md)