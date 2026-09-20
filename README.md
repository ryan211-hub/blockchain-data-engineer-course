# Blockchain Data Engineer Course

《从传统数据工程师到 Blockchain Data Engineer》课程仓库。

## 课程目标

能够独立设计和开发一个链上数据平台，并具备进入 Web3 数据公司的能力。

## 学习组织方式

- ChatGPT：互动学习、理解检查、课程讲解
- Notion「课程总目录」：课程进度主记录与冲突裁决来源
- Notion Module：同步维护当前 Module 的可变进度镜像
- Notion Lesson：教材与学习记录
- GitHub：Markdown 教材、代码实践、项目成果、版本历史，并在 README 中同步关键可变进度

> 若 Notion「课程总目录」、Module 页面与 GitHub README 的进度发生冲突，以 Notion「课程总目录」为准并同步修正其他镜像。

## 当前学习进度

- 当前阶段：第三阶段 Data Engineering
- 已完成 Module：Module 1、Module 2、Module 3、Module 4、Module 5、Module 6、Module 7、Module 8、Module 9
- 当前 Module：Module 10 — 实时数据
- Module 6 完成结论：第 1～4 课均已完成，已达到 Module Contract 结束标准。已能区分 History / Historical State，解释 Pruning、Archive 成本，并根据 Query Semantics、Derived State、规模与 TCO 判断 Archive RPC / Self-hosted / Hybrid。
- Module 7 完成结论：第 1～10 课与结课综合实践均已完成；Mini Indexer 五项验收 Normal Sync / Idempotent Replay / Crash Recovery / Backfill / Reorg Recovery 全部通过，已达到 Module Contract 结束标准。五项实践由用户在 Codex 中实际完成并明确确认通过。
- Module 8 完成结论：第 1～8 课与结束标准综合检查均已完成，已达到 Module Contract 结束标准。已能从 Query Semantics 推导 Business Object、Grain、Source Identity、Unique Key、Fact / Dimension、Relation 与 Layer，并区分 Source Identity 与 Business Identity。
- Module 9 完成结论：第 1～7 课与结束标准综合检查均已完成，达到 Module Contract 结束标准。已能独立设计基础 Blockchain Batch ETL Pipeline，并解释 Source / Target、Processing Range、Grain、Unique Key、Watermark / Cursor / Checkpoint、Idempotency、Validation、Late-arriving / Reorg、Lookback / Backfill / Historical Repair、DAG Dependency 与 Scheduler / Orchestrator。
- 当前 Lesson：Module 10 第 5 课｜Delivery Semantics：At-most-once、At-least-once、Exactly-once
- 当前状态：Module 10 第 4 课已完成并通过理解检查；准备进入第 5 课
- 最近完成：Module 10 第 4 课《Consumer Group 与 Ordering：如何同时获得并行处理和局部有序》理解检查通过；Canonical Lesson Content 已同步到 Notion 和 GitHub。
- 最近掌握重点：已建立 Consumer Group 心智模型：同组 Consumer 通过 Partition Assignment 分担工作；不同 Group 可独立消费同一 Topic；一个 Partition 在同一 Group 内同一时刻最多一个 active owner；Consumer Progress 可表示为 (consumer_group, topic, partition) -> offset；Rebalance 支持故障恢复与重新分配。
- 历史同步债务处理决定：用户已明确要求忽略 Module 8 第 4～6 课的既有 Notion / GitHub 同步债务与第 5 课 Canonical Drift；这些历史问题不再修复，也不再阻塞课程推进。
- 下一步准确入口：开始 Module 10 第 5 课《Delivery Semantics：At-most-once、At-least-once、Exactly-once》。

## 目录

- [Module 4 — Blockchain Client & Node Storage](module-04-blockchain-client-node-storage/README.md)
- [Module 5 — RPC](module-05-rpc/README.md)
- [Module 6 — Archive Node](module-06-archive-node/README.md)
- [Module 7 — Indexer](module-07-indexer/README.md)
- [Module 8 — 数据建模](module-08-data-modeling/README.md)
- [Module 9 — ETL](module-09-etl/README.md)
- [Module 10 — 实时数据](module-10-realtime-data/README.md)

## 同步规则

每课结束或暂停时：

1. 在 ChatGPT 中完成讲解、理解检查与必要校准。
2. 将同一份 Canonical Lesson Content 同步到 Notion Lesson。
3. 将同一份 Canonical Lesson Content 同步到 GitHub Lesson Markdown。
4. 回读 Notion Lesson 与 GitHub Lesson Markdown，确认教材正文一致。
5. 更新 Notion「课程总目录」主进度记录。
6. 同步更新当前 Notion Module 页面、GitHub 根 README、GitHub 当前 Module README 的可变进度。
7. 回读这些位置，确认当前阶段、Module、Lesson、状态、已完成 Lesson 与下一步入口一致。

Lesson 页面不维护全局可变进度；若进度镜像发生冲突，以 Notion「课程总目录」为准。