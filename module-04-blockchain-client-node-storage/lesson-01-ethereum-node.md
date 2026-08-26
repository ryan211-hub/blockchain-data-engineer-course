# 第1课｜一台 Ethereum 节点，到底在干什么？

## 本课目标

从节点客户端实现视角理解 Ethereum Client 的职责，并回答：为什么区块链节点需要数据库？为什么节点既要保存 History，也要维护 Current State？

## 一、从 Module 3 切换到 Module 4

Module 3 主要站在 Blockchain Data Engineer 的观察视角理解：

```text
Block
  ↓
Transaction
  ↓
Trace / Log / Event
  ↓
State Change
```

Module 4 切换到节点客户端实现视角：Ethereum 节点自己如何得到、执行并保存这些数据。

典型调用链可以理解为：

```text
应用程序
   ↓
RPC Server
   ↓
Ethereum Client
   ↓
本地数据库
   ↓
Blockchain Network
```

Ethereum Client 不是简单的区块下载软件，而是 Ethereum 状态机的软件实现。

## 二、Ethereum Client 的核心职责

典型客户端如 Geth、Nethermind、Besu、Erigon，需要承担：

- P2P 网络通信
- 区块同步与验证
- Transaction 执行
- EVM 执行
- State 维护
- 区块数据持久化
- RPC 服务

可以把它概括为：

> Ethereum Client = 一个 Ethereum 状态机的完整软件实现。

## 三、为什么节点必须保存数据

假设节点执行完一个区块后得到：

```text
Alice    9 ETH
Bob      2.8 ETH
Charlie  0.2 ETH
```

如果程序退出后什么都不保存，那么下次启动只能从 Genesis 开始重新执行所有区块，才能重新得到当前状态。

因此节点必须把部分结果持久化到磁盘，避免重复计算。

## 四、History 与 State

### 【逻辑数据模型视角】

节点至少需要面对两类不同的数据。

History 回答：

> 发生过什么？

例如：

- Block
- Transaction
- Receipt
- Log
- Header

State 回答：

> 现在世界是什么样子？

例如：

- Account balance
- Nonce
- Contract code
- Contract storage

概念上可以表示为：

```text
          Blockchain Node
                 │
       ┌─────────┴─────────┐
       │                   │
    History              State
       │                   │
发生过什么？           现在是什么？
```

### 【视角提醒】

这里是逻辑分类，不代表磁盘里一定存在两个名为 `history.db` 和 `state.db` 的数据库。

## 五、为什么不能只保存 Ledger

理论上：

```text
Genesis State
+
全部历史 Transaction
=
Current State
```

也就是：

```text
History + 执行规则 → State
```

因此只保存 Ledger，理论上仍可以计算当前余额。

但如果每次查询余额都从 Genesis 重放整条链，代价极高。

节点因此持续维护 Current State，本质上属于：

> 空间换时间。

## 六、与传统银行系统的类比

银行系统也可以只通过交易流水计算余额，但实际系统通常会维护当前余额字段。

```text
Transaction Ledger
       ↓
Account Balance
```

对应到 Blockchain：

```text
Blockchain History
       ↓ execution
Blockchain State
```

从数据工程角度，可近似类比为：

```text
Ledger ≈ Source of Truth / Event Log
State  ≈ Materialized Result
```

但 State 不是独立于 History 的第二份真相，而是节点执行历史后维护出来的当前结果。

## 七、为什么会出现 LevelDB / RocksDB

节点需要把 Blocks、Transactions、Receipts、State、Contract Storage 等数据落到 SSD，因此需要持久化存储引擎。

LevelDB / RocksDB 可以先理解为高性能嵌入式 Key-Value Storage Engine：

```text
Key → Value
```

它们与 Oracle / PostgreSQL 的关系型数据库使用场景不同。

## 八、State Trie 与 Storage Engine 的区别

### 【Ethereum 协议 / 数据结构视角】

State Trie 用于组织 State、生成状态承诺并得到 `stateRoot`。

### 【Client 存储实现视角】

Trie 节点最终仍需要持久化到底层 Storage Engine：

```text
State Trie Nodes
      ↓
LevelDB / RocksDB
      ↓
SSD
```

因此：

```text
State Trie = 状态数据结构 / 承诺结构
LevelDB / RocksDB = 持久化存储引擎
```

二者不是同一层概念。

## 九、Module 4 总体心智模型

```text
         Ethereum Network
                │
                ▼
         Ethereum Client
                │
      ┌─────────┴─────────┐
      │                   │
Block Validation       EVM Execution
      │                   │
      └─────────┬─────────┘
                │
            State Change
                │
    ┌───────────┴───────────┐
    │                       │
 History                  State
    │                       │
    └───────────┬───────────┘
                │
         Storage Engine
                │
      LevelDB / RocksDB
                │
               SSD
```

## 十、Blockchain Data Engineer 为什么需要理解它

后续很多问题的根源都在 Node Storage Architecture：

- 为什么 `eth_getLogs` 可能很慢
- 为什么 RPC Provider 限制历史查询
- 为什么 Archive RPC 更贵
- 为什么节点 SSD IO 很高
- 为什么 Indexer 不应无节制调用 RPC
- 为什么 RPC Provider 需要 Cache
- 为什么 Historical State 查询成本高

## 十一、理解检查结果

### 问题1

只保存 History、不保存 Current State，理论上还能否计算 Alice 当前余额？

**回答正确：可以。** 因为可以从历史账本重新执行并还原每次 State Change，最终得到 Current State。

### 问题2

State Trie 是否等于 LevelDB？

**回答正确：选择 B。** State Trie 是组织 / 验证 State 的数据结构，LevelDB / RocksDB 是持久化 Storage Engine。

### 问题3

为什么 `eth_getBalance(Alice, "latest")` 不需要从 Genesis 重放全部区块？

**回答正确：** 节点已经持续维护 Current State，可以直接读取当前状态，避免大量重复计算。

## 本课状态

✅ 已完成
