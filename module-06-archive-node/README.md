# Module 6 — Archive Node

## 模块目标

从 Blockchain Data Engineer 视角理解 Archive Node 为什么存在：区分 History 与 Historical State，理解普通 Full Node / Pruned Node 为什么不能稳定回答任意历史 State 查询，理解 Archive Node 为何昂贵，以及数据公司在什么场景需要自建或购买 Archive 能力。

## Module Contract

### 必须掌握
- History、Current State、Historical State 三者的区别
- Full / Pruned Node 与 Archive Node 在“历史状态可查询性”上的核心区别
- `eth_getBlockByNumber(old)` 与 `eth_getBalance(address, old_block)` 为什么不是同一类历史查询
- 为什么 `stateRoot` 存在不等于旧 State 仍然可读取
- Archive Node 的主要成本来源：历史状态保留、存储增长、I/O、同步与运维
- Blockchain Data Engineer 什么时候真正需要 Archive Node，什么时候可以依靠 History / Logs / 自己维护的派生状态

### 可以了解
- 不同 Ethereum Client 的 Archive / Pruning 实现差异
- Snapshot、State History 等实现思路
- Provider 对 Historical State 的产品封装

### 本 Module 不展开
- State Trie / Storage Engine 内核实现 → Module 4 已完成
- RPC Provider 的限流、缓存、商业模式 → Module 5 已完成
- Indexer Pipeline、Checkpoint、Backfill 机制 → Module 7

### 结束标准
能够解释 Archive Node 比普通 Node 多提供的核心能力，为什么它昂贵；能区分 Historical Block / Log 与 Historical State；面对具体数据需求时能判断是否真的需要 Archive Node。

## 当前学习进度

- 当前阶段：第二阶段 Infrastructure
- 当前 Module：Module 6 — Archive Node
- 已完成 Lesson：第 1 课、第 2 课、第 3 课
- 当前 Lesson：第 3 课｜Archive Node 为什么昂贵：Historical State Retention 的成本模型
- 当前状态：✅ 已完成
- 最近完成：第 3 课理解检查与校准已完成；已掌握 Archive Node 的 Storage、Read / Write I/O、Sync、Backup / Recovery、Operations 成本模型，以及 Build vs Buy 的基本判断。
- 尚未完成：第 4 课
- 下一步准确入口：开始第 4 课，判断什么时候真正需要 Archive Node：区分 Historical State、数仓派生状态，并完成 Archive RPC / 自建能力的需求决策。

> 本区块是 Notion「课程总目录」主进度记录的同步镜像；若发生冲突，以课程总目录为准。

## 课程索引

1. [第1课｜Archive Node 到底多保存了什么：History ≠ Historical State](lesson-01-history-vs-historical-state.md)（✅ 已完成）
2. [第2课｜Full / Pruned Node vs Archive Node：Pruning 到底删掉了什么？](lesson-02-pruning-full-vs-archive.md)（✅ 已完成）
3. [第3课｜Archive Node 为什么昂贵：Historical State Retention 的成本模型](lesson-03-archive-node-cost-model.md)（✅ 已完成）