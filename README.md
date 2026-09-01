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
- 已完成 Module：Module 1、Module 2、Module 3、Module 4、Module 5
- 当前 Module：Module 6 — Archive Node
- Module 6 已完成：第 1 课｜Archive Node 到底多保存了什么：History ≠ Historical State；第 2 课｜Full / Pruned Node vs Archive Node：Pruning 到底删掉了什么？；第 3 课｜Archive Node 为什么昂贵：Historical State Retention 的成本模型
- 当前 Lesson：第 3 课｜Archive Node 为什么昂贵：Historical State Retention 的成本模型
- 当前状态：✅ 已完成
- 最近完成：Module 6 第 3 课理解检查与校准完成；已掌握 Archive Node 的成本不是单纯增加磁盘容量，而是 Historical State Retention 对 Storage、Read / Write I/O、Sync、Backup / Recovery 与 Operations 的连锁放大。
- 尚未完成：Module 6 第 4 课
- 下一步准确入口：开始 Module 6 第 4 课，判断什么时候真正需要 Archive Node：区分 Historical State、数仓派生状态，并完成 Archive RPC / 自建能力的需求决策。

## 目录

- [Module 4 — Blockchain Client & Node Storage](module-04-blockchain-client-node-storage/README.md)
- [Module 5 — RPC](module-05-rpc/README.md)
- [Module 6 — Archive Node](module-06-archive-node/README.md)

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