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
- Module 7 已完成：第 1 课、第 2 课、第 3 课、第 4 课、第 5 课、第 6 课
- 当前 Lesson：Module 7 第 6 课｜Backfill：Indexer 如何安全补历史数据，而不是每次从 Genesis 重跑
- 当前状态：✅ 已完成
- 最近完成：Module 7 第 6 课理解检查与校准完成；已掌握 Backfill = Bounded Range Job、Realtime Checkpoint 与 Backfill Cursor 的进度隔离、Backfill 自身的 Durable Progress，以及与 Real-time 共用 Idempotent Writer 的设计。
- 尚未完成：Module 7 第 7 课
- 下一步准确入口：开始 Module 7 第 7 课｜Reorg：Indexer 如何处理 canonical / orphaned 数据与回滚重放。重点理解同一 block_number 为什么会对应不同 block_hash、如何识别 canonical chain，以及 Reorg 后数据如何回滚与重放。

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