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
- 已完成 Lesson：无
- 当前 Lesson：第 1 课｜Archive Node 到底多保存了什么：History ≠ Historical State
- 当前状态：🟡 学习中
- 下一步准确入口：开始第 1 课，先区分“过去发生过什么”与“过去某一高度的完整 State”，再理解 Archive Node 为什么存在。

> 本区块是 Notion「课程总目录」主进度记录的同步镜像；若发生冲突，以课程总目录为准。

## 课程索引

1. 第1课｜Archive Node 到底多保存了什么：History ≠ Historical State（当前学习中）