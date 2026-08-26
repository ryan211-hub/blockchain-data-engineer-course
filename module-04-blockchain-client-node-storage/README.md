# Module 4 — Blockchain Client & Node Storage

## 模块目标

从 Blockchain Client 的实现视角理解节点如何同步、验证、执行并持久化区块链数据，建立 Node Storage、State、State Trie、Pruning 与 Archive Node 的基础设施心智模型。

## 当前状态

- 第 1 课：已完成
- 第 2 课：已完成
- 第 3 课：已完成
- 第 4 课：已完成
- 第 5 课：学习中

## 课程索引

1. [第1课｜一台 Ethereum 节点，到底在干什么？](lesson-01-ethereum-node.md)
2. [第2课｜节点的磁盘里到底保存了什么：History、State 与 Key-Value Store](lesson-02-history-state-kv-store.md)
3. [第3课｜State Trie：Ethereum 为什么不能只用普通 map<Address, Account>？](lesson-03-state-trie.md)
4. [第4课｜Historical State、Pruning 与 Archive Node](lesson-04-historical-state-pruning.md)
5. [第5课｜Archive Node 为什么贵：历史状态查询、存储成本与数据公司的基础设施选择](lesson-05-archive-node-cost.md)

## 视角规则

本模块默认显式区分：

- Ethereum 协议视角
- Blockchain Client / 节点实现视角
- 逻辑数据模型视角
- 物理存储视角
- 历史数据 / RPC 视角
- Blockchain Data Engineer 视角
