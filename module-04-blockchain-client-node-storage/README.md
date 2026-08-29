# Module 4 — Blockchain Client & Node Storage

## 模块目标

从 Blockchain Data Engineer 所需的基础设施视角理解 Blockchain Client：为什么节点需要本地数据库，History 与 State 如何区分，KV Storage 与 State Trie 各自解决什么问题，以及 Ethereum Account State 与 Bitcoin UTXO Set 的核心差异。目标是建立足够支撑后续 RPC、Archive Node 与 Indexer 学习的节点心智模型，而不是深入培养 Client / Storage Engine 内核开发能力。

## 当前状态

✅ 已完成。

- 第 1 课：已完成
- 第 2 课：已完成
- 第 3 课：已完成
- 第 4 课：已完成
- 第 5 课：已完成
- 第 6 课：已完成
- 第 7 课：已完成
- 第 8 课：已完成
- 第 9 课：已完成
- 第 10 课：已完成

原“节点数据库故障与恢复”调整为扩展阅读，不计入主线完成条件。

## Module Contract

### 必须掌握

- Blockchain Client 的核心职责
- 为什么节点需要本地数据库
- History 与 Current State 的区别
- Node Database 与 Key-Value Store 的基本访问模型
- State Trie / stateRoot 的作用与 Storage Engine 的层级区别
- Full Sync / Snap Sync 的基本思想
- Bitcoin UTXO Set 与 Ethereum Account State 的核心差异

### 可以了解

- Historical State 与 Pruning 的基本概念
- LevelDB / RocksDB 为什么适合节点的基本原因
- LSM Tree、MemTable、SSTable、Compaction、Write Amplification 作为补充心智模型

### 本 Module 不展开

- RPC Provider 架构、RPC 查询优化、eth_getLogs 深入 → Module 5 RPC
- Archive Node 的商业价值、历史状态基础设施与成本模型 → Module 6 Archive Node
- Indexer Pipeline、Checkpoint、数据落库 → Module 7 Indexer
- WAL Recovery、Crash Consistency、数据库 Repair、Storage Engine 内核调优 → 扩展阅读

## 课程索引

1. [第1课｜一台 Ethereum 节点，到底在干什么？](lesson-01-ethereum-node.md)
2. [第2课｜节点的磁盘里到底保存了什么：History、State 与 Key-Value Store](lesson-02-history-state-kv-store.md)
3. [第3课｜State Trie：Ethereum 为什么不能只用普通 map<Address, Account>？](lesson-03-state-trie.md)
4. [第4课｜Historical State、Pruning 与 Archive Node](lesson-04-historical-state-pruning.md)
5. [第5课｜Archive Node 为什么贵：历史状态查询、存储成本与数据公司的基础设施选择](lesson-05-archive-node-cost.md)
6. [第6课｜节点同步：为什么新节点不必总从 Genesis 重放？Full Sync、Snap Sync 与状态恢复](lesson-06-node-sync.md)
7. [第7课｜RPC 查询到底查到哪里：eth_getBalance、eth_getLogs 与 Node Storage 的访问路径](lesson-07-rpc-node-storage.md)
8. [第8课｜Logs Bloom、索引与缓存：为什么 eth_getLogs 仍然可能很慢？](lesson-08-logs-bloom-index-cache.md)
9. [第9课｜LevelDB / RocksDB 为什么适合节点：LSM Tree、Compaction 与写放大](lesson-09-lsm-tree-compaction.md)
10. [第10课｜UTXO Set：Bitcoin 如何维护可花费状态，与 Ethereum Account State 有什么不同？](lesson-10-utxo-set.md)

## 结束说明

Module 4 主线已完成。最终检查中 LevelDB / RocksDB 与 KV Storage 的两个校准点已提示，用户选择跳过，不作为进入 Module 5 的阻塞项。

## 视角规则

本模块默认显式区分：

- Ethereum 协议视角
- Blockchain Client / 节点实现视角
- 逻辑数据模型视角
- 物理存储视角
- 历史数据 / RPC 视角
- Blockchain Data Engineer 视角
