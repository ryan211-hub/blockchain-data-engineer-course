# 第7课｜RPC 查询到底查到哪里：eth_getBalance、eth_getLogs 与 Node Storage 的访问路径

## 本课目标

理解常见 Ethereum RPC 请求在节点内部到底访问 History、Current State 还是 Historical State，并建立 RPC → Client → Node Database 的查询路径心智模型。

## RPC 访问路径

```text
Application / Web3
      ↓
JSON-RPC
      ↓
Ethereum Client
      ↓
History / Current State / Historical State
      ↓
Node Database / Index / State
```

RPC 本身不保存业务数据，真正回答请求的是 Ethereum Client。

## Current State

`eth_getBalance(Alice, "latest")` 查询 Alice 当前 ETH 余额，属于 Current State Query。节点已经持续维护 Current State，因此通常不需要从 Genesis 重放历史。

ETH 余额属于 Ethereum Account State：

```text
Account
├─ balance
├─ nonce
├─ storageRoot
└─ codeHash
```

## ERC-20 Balance 与 eth_call

ERC-20 余额不保存在 Alice 的 Ethereum Account `balance` 字段，而保存在 Token Contract Storage 中。

```text
USDC Contract
└─ balances[Alice] = 1000
```

因此查询 USDC 余额通常通过：

```text
eth_call → USDC.balanceOf(Alice)
```

由 Client 在指定 State 上执行只读 EVM 调用。

## Historical State

`eth_getBalance(Alice, block=10,000,000)` 查询的是 Block 10,000,000 对应的 Historical State。

即使 Historical Block 与 `stateRoot` 仍存在，如果对应 Historical State 已被 pruning，也可能无法直接回答。

```text
Historical Block available
≠
Historical State available
```

历史 ERC-20 balance 同理，需要在指定历史 State 上执行 `eth_call`。

## History 查询

- `eth_getBlockByNumber(N)` → Historical Block / History
- `eth_getTransactionReceipt(tx_hash)` → Receipt / Execution Result History
- `eth_getLogs(fromBlock..toBlock)` → Logs / History

`eth_getLogs` 是范围扫描与过滤型 Workload，而 `eth_getBalance(Alice, latest)` 更接近已知 Key 的 Current State Lookup。

Client 可以使用 Bloom Filter、Index、Cache、Storage Layout 等机制优化，但本质仍是历史范围查询。

## eth_call 的本质

`eth_call` 不是读取一条历史 Call 记录，而是在指定 State 上模拟执行一次 EVM 调用，不产生真实链上 State Change。

```text
eth_call(..., latest)
→ Current State

eth_call(..., historical_block)
→ Historical State
```

## Node Database 与 Analytics Database

Node Database 主要服务：Block validation、State execution、State lookup、History lookup。

典型 Blockchain Data Engineering Pipeline：

```text
Ethereum Node
    ↓ RPC
Indexer
    ↓
Raw / Normalized Data
    ↓
Warehouse
    ↓
SQL / API / Dashboard
```

大量 Historical RPC 不适合作为分析系统的最终查询方式。更常见做法是增量索引 History / Logs / Trace，在 Warehouse 中维护 Balance / Snapshot；Archive Node 主要用于 Backfill、Validation 与 Exceptional Query。

## 判断 RPC 类型的方法

```text
发生过什么？
→ History

现在是什么？
→ Current State

过去某一时刻是什么？
→ Historical State
```

## 理解检查结果

- `eth_getBlockByNumber(10,000,000)` → History：正确。
- `eth_getBalance(Alice, "latest")` → Current State：正确。
- `eth_getBalance(Alice, 10,000,000)` → Historical State：正确。
- 能解释 `eth_getLogs` 的 Range Scan / Filter 与 Current State Key Lookup 的成本差异。
- 能区分 ETH Account State 与 ERC-20 Contract Storage，并正确使用 `eth_call` / `balanceOf` 理解 ERC-20 balance 查询。

## 本课状态

✅ 已完成。
