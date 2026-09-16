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
- 当前 Lesson：第 6 课｜Batch Job 的输入输出边界与可验证性：怎么证明这一批数据真的处理对了
- 当前状态：第 6 课已完成；Module 9 继续进行中
- 已完成 Lesson：第 1 课｜ETL 到底是什么：从“已有数据”到“可持续数据管道”；第 2 课｜Full Load、Incremental Load 与 Backfill：ETL 到底处理哪一段数据？；第 3 课｜Watermark / Cursor / Checkpoint：ETL 如何记住处理到哪里；第 4 课｜Idempotent Load 与失败恢复：为什么 ETL 必须可安全重跑；第 5 课｜Late-arriving Data 与 Reorg：为什么已经处理的数据后来还会变化；第 6 课｜Batch Job 的输入输出边界与可验证性：怎么证明这一批数据真的处理对了
- 已完成 Module：Module 1～8
- 最近掌握重点：Batch Job 显式定义 Input / Output Boundary；Input Range 使用 `[start, end)` 半开区间；Execution Success 不等于 Processing Unit Complete；Validation 通过后才能推进 Checkpoint；Source / Target Grain 不同时应设计 Range、Unique Key、Invariant 与 Reconciliation 验证；Checkpoint 是 State，Job Run 是 History。
- 下一步准确入口：开始 Module 9 第 7 课｜DAG / Dependency / Scheduler：多个 ETL Job 如何组成一条可运行的数据管道。

## Lessons

- [第 1 课｜ETL 到底是什么：从“已有数据”到“可持续数据管道”](lesson-01-what-is-etl.md)
- [第 2 课｜Full Load、Incremental Load 与 Backfill：ETL 到底处理哪一段数据？](lesson-02-full-incremental-backfill.md)
- [第 3 课｜Watermark / Cursor / Checkpoint：ETL 如何记住处理到哪里](lesson-03-watermark-cursor-checkpoint.md)
- [第 4 课｜Idempotent Load 与失败恢复：为什么 ETL 必须可安全重跑](lesson-04-idempotent-load-failure-recovery.md)
- [第 5 课｜Late-arriving Data 与 Reorg：为什么已经处理的数据后来还会变化](lesson-05-late-arriving-reorg.md)
- [第 6 课｜Batch Job 的输入输出边界与可验证性：怎么证明这一批数据真的处理对了](lesson-06-batch-boundary-validation.md)