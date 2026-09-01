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

- 当前阶段：第二阶段 Infrastructure
- 已完成 Module：Module 1、Module 2、Module 3、Module 4、Module 5、Module 6
- 当前 Module：Module 7 — Indexer
- Module 6 完成结论：第 1～4 课均已完成，已达到 Module Contract 结束标准。已能区分 History / Historical State，解释 Pruning、Archive 成本，并根据 Query Semantics、Derived State、规模与 TCO 判断 Archive RPC / Self-hosted / Hybrid。
- 当前 Lesson：Module 7 第 1 课｜Indexer 到底是什么：为什么 Node / RPC 之后还需要一层 Indexer
- 当前状态：⏳ 未开始
- 最近完成：Module 6 第 4 课理解检查与校准完成；Module 6 已正式结束。已掌握 Historical Data ≠ Historical State，稳定可预测的 Historical State 需求可以通过 Archive RPC Backfill 后沉淀到 Warehouse，而任意 Historical eth_call / Ad-hoc Query 更依赖通用 Archive State Queryability。
- 尚未完成：Module 7 第 1 课
- 下一步准确入口：开始 Module 7 第 1 课｜Indexer 到底是什么：为什么 Node / RPC 之后还需要一层 Indexer。建立 Node / RPC / Indexer / Warehouse 的职责边界，并理解为什么分析型数据平台不能直接把 RPC 当数据库使用。

## 目录

- [Module 4 — Blockchain Client & Node Storage](module-04-blockchain-client-node-storage/README.md)
- [Module 5 — RPC](module-05-rpc/README.md)
- [Module 6 — Archive Node](module-06-archive-node/README.md)
- [Module 7 — Indexer](module-07-indexer/README.md)

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