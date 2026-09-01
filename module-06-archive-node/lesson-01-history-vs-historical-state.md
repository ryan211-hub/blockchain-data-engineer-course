# 第1课｜Archive Node 到底多保存了什么：History ≠ Historical State

## Lesson Contract
- 所属 Module：Module 6 — Archive Node
- 本课核心问题：Full Node 明明保存了历史 Block，为什么仍然不能稳定回答任意历史高度的余额、Storage 或 `eth_call`？Archive Node 到底多保存了什么？
- 学完后应能回答：History 与 Historical State 的区别；为什么旧 Block 可查不代表旧 State 可查；哪些 RPC 更接近历史 History 查询，哪些真正依赖 Historical State。
- 本课明确不展开：Archive Node 的完整成本模型与部署规格（后续 Lesson）；Indexer Backfill / Checkpoint（Module 7）；State Trie / Storage Engine 内核实现（Module 4 已完成）。

## 一、Block History 和 State 是两套不同的问题
【数据语义视角】

假设 Ethereum 已经到了 Block 25,000,000，而 Node 可以执行：

```text
eth_getBlockByNumber(12,000,000)
```

这只能说明它还能访问 Block 12,000,000 的历史数据，并不能推出它一定还能直接读取：

```text
eth_getBalance(Alice, 12,000,000)
```

核心区别：

```text
History
= 过去发生过什么

State
= 执行完这些事情以后，世界是什么样
```

因此：

```text
Historical Block
≠
Historical State
```

## 二、银行系统类比
如果银行保存全部交易流水，例如 Alice +100、-20、+50，这属于 Transaction History。

如果要问 Alice 在某个过去时间点的余额，则问的是 Historical State。如果系统保存了余额快照，可以直接查询；如果只保存流水和当前余额，理论上仍可从初始状态加历史流水重新推导过去余额，但这不再是简单 Point Lookup。

```text
History
→ 可以用于重新执行 / 推导

Historical State
→ 可以直接查询过去某个时间点的 State
```

## 三、普通 Node 为什么不一直保存所有 Historical State
【节点存储视角】

Ethereum 每产生一个 Block，State 都会推进：

```text
State N
  ↓
执行 Block N+1
  ↓
State N+1
```

如果 Node 想让任意历史高度都能快速查询，就必须持续保留足够的历史状态信息。由于 Ethereum State 很大且持续变化，长期全部保留会造成不断增长的存储成本。

普通节点因此可能通过 Pruning 不再保留部分旧 State 数据，于是可能出现：

```text
Block 12,000,000
仍可查询

但

State at Block 12,000,000
已经不能直接访问
```

## 四、Archive Node 多保存的核心不是“所有历史 Block”
Archive Node 真正特殊的地方，是保留足够的历史 State 数据，使过去许多 Block 高度下的 State 可以被查询。

逻辑模型：

```text
普通 / Pruned Node

History
├── Block
├── Transaction
├── Receipt
└── Log

State
└── Current / Recent State
```

Archive Node 更接近：

```text
History
├── Block
├── Transaction
├── Receipt
└── Log

State History
├── State at Block 10,000,000
├── State at Block 10,000,001
├── State at Block 10,000,002
├── ...
└── Current State
```

这只是逻辑模型，不代表磁盘上真的为每个 Block 完整复制一份整个 State。

## 五、stateRoot 不是 Historical State 本身
每个 Block Header 中都有 `stateRoot`。它是当时 State 的认证指纹 / 根承诺，而不是完整 State 数据本身。

因此：

```text
stateRoot Available
≠
Historical State Data Available
```

即使旧 Block Header 和 stateRoot 仍然存在，也不能证明构成那个 State 的旧数据仍然保存在本地。

## 六、重新分类几个 RPC
### `eth_getBlockByNumber(old_block)`
问的是“过去这个 Block 是什么”，属于 History Query。

### `eth_getTransactionByHash`
问的是过去这笔 Transaction 是什么，也属于 History Query。

### `eth_getLogs(old range)`
问的是某段历史范围内发生过哪些符合条件的 Logs，主要属于 Historical Logs / History Query。它可能因为 Range Scan 非常昂贵，但“昂贵”不代表它就是 Historical State Query。

### `eth_getBalance(address, old_block)`
问的是过去某一 Block 高度的 Account State 中 balance 是多少，属于 Historical State Query。

### `eth_getStorageAt(contract, slot, old_block)`
问的是过去某一 Block 高度的 Contract Storage Slot，属于 Historical State Query。

## 七、旧 Block 上的 `eth_call`
例如：

```text
eth_call(
    USDC.balanceOf(Alice),
    block=12,000,000
)
```

Node 需要：

```text
找到 Historical State at 12,000,000
↓
加载 Contract Code / Storage
↓
EVM Execution
↓
Result
```

所以它实际上是：

```text
Historical State
+
EVM Execution
```

## 八、请求类型对照
| RPC | 主要问题 | Archive State 是否关键 |
|---|---|---|
| `eth_getBlockByNumber(old)` | 过去发生了什么 | 通常不是核心 |
| `eth_getLogs(old range)` | 过去有哪些 Logs | 通常不是 Historical State |
| `eth_getBalance(address, old)` | 过去余额是多少 | 是 |
| `eth_call(..., old)` | 在过去 State 上执行合约 | 是 |

核心判断：

```text
Historical Block / Tx / Log
≠
Historical State
```

## 九、Archive Node 是否是数据工程师必需的
不是。很多历史分析可以主要依赖 History / Logs。例如 USDC Transfer Volume、Uniswap 历史 Swap Volume，通常主要来自 Historical Logs / Events，并不意味着每个查询都需要 Archive State。

Historical State 特别重要的场景包括：
- 查询某地址在指定历史 Block 的 ETH Balance。
- 查询历史 Contract Storage。
- 在指定历史 Block 上执行 `eth_call`。
- 需要还原某一时刻完整协议状态，而该状态无法仅从已维护的历史事实表直接获得。

## 十、Blockchain Data Engineer 视角
典型数据平台不会所有事情都直接查询 Archive Node，而会：

```text
Blockchain
↓
History / Logs
↓
Indexer
↓
Database / Warehouse
↓
自己维护历史事实和派生状态
```

如果已经维护 `wallet_balance_snapshot` 一类派生表，过去余额可以直接查 Warehouse，不必每次请求 Archive Node。具体 Indexer 与 Snapshot 设计放到 Module 7。

## 十一、本课核心模型
```text
History
= 过去发生过什么

Current State
= 现在世界是什么样

Historical State
= 过去某一个时刻，世界是什么样
```

Archive Node 的核心价值是：

```text
Historical State Queryability
```

而不是简单“保存历史 Block”。

必须记住：

```text
Old Block Available
≠
Old State Available

stateRoot Available
≠
Historical State Data Available
```

## 理解检查
### 问题 1
假设一个 Node 可以成功执行 `eth_getBlockByNumber(12,000,000)`，为什么不能由此推出它一定也能执行 `eth_getBalance(Alice, 12,000,000)`？请从 History vs Historical State 来解释。

### 用户回答
“问题一：因为 block 和 state 是不同的，Block 并不等于 State。如果要通过查询 block 数据的方式获取指定 Block 高度时的 State，那么一个可行的办法是从 Genesis Block 一直到指定高度的 Block，重新执行这个范围内所有的记录，这样就可以获取到相应 State。”

### 老师判断与校准
✅ 通过。`eth_getBlockByNumber(12,000,000)` 能成功，只说明节点还能访问这个历史 Block；它不能证明节点仍然保留 Block 12,000,000 对应的 Historical State。用户提出的“从 Genesis 一直重放到目标高度”理论上成立，但这是“可重建”，不等于“可以直接高效查询”。Archive Node 的价值之一，就是避免每次历史状态查询都进行这种昂贵的全量重放。

### 问题 2
下面哪个请求真正更依赖 Historical State？A：`eth_getLogs(fromBlock=12,000,000, toBlock=13,000,000)`；B：`eth_call(someContract, block=12,000,000)`。为什么？

### 用户回答
“问题 2，B 请求是真正更依赖 historical state。因为 B 的 request 请求的是在指定 1200 万高度的 block 时获取当时的 contract，而 A 只是获取指定范围里面的 block 数据”

### 老师判断与校准
✅ 通过。更准确地说，B 是在 Block 12,000,000 对应的 Historical State 上执行一次 EVM Call，因此访问路径是 Historical State → Contract Code / Storage → EVM Execution → Result。A 主要查询历史范围中的 Logs；即使很昂贵，也主要属于 Historical Logs / History Query，而不是 Historical State Query。

### 问题 3
假设 Block 12,000,000 的 stateRoot 仍然完整保存在 Block Header 中，为什么仍然不能据此说这个 Node 肯定可以查询 Block 12,000,000 时 Alice 的余额？

### 用户回答
“问题 3：

因为 state 和 state root 是不同的。

这个 node 查询 block 1200 万高度时的 Alice 余额，他的需求是获取当时的 state 数据。虽然我们知道 state root 完整地保存在那个 block header 里面，但是 state root 它是作为一个检验的指纹，而并非真实的 state 数据”

### 老师判断与校准
✅ 通过。`stateRoot` 是当时 State 的 Commitment / Fingerprint，不是完整 State 数据本身。即使 stateRoot 仍在，也不能证明节点本地仍然保存 Alice Account、Contract Storage、相关历史 State 数据。因此 `stateRoot Available ≠ Historical State Data Available`。

## 重点总结
1. History、Current State、Historical State 是三个不同概念。
2. Historical Block 可查询，不代表 Historical State 可查询。
3. stateRoot 是历史 State 的承诺值，不是 Historical State 数据本身。
4. `eth_getLogs(old range)` 主要是历史范围查询；`eth_getBalance(old)`、`eth_getStorageAt(old)`、`eth_call(old)` 真正依赖 Historical State。
5. Archive Node 的核心能力是 Historical State Queryability。
6. 理论上的历史重放可以重建 State，但不等于生产环境中的直接、低成本历史状态查询。