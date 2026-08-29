# 第1课｜RPC 到底是什么：从 dApp 请求到 Node Client 的接口层

## Lesson Contract

- 所属 Module：Module 5 — RPC
- 本课核心问题：RPC 在 Blockchain Client、Node、dApp / Wallet / Indexer 之间到底处于什么位置？
- 学完后应能回答：Node、RPC、RPC Provider 有什么区别；为什么应用通过 RPC 而不是直接读取 Node Storage；Indexer 经 Provider 访问链上数据时完整路径是什么。
- 本课明确不展开：Load Balancer、Rate Limit、Cache、Archive Node、多节点集群与 Provider 商业架构深挖。

## 一、RPC 是什么

RPC = Remote Procedure Call，远程过程调用。它让客户端以类似调用函数的方式，请求远程机器执行一个操作。

Ethereum 常见的是 JSON-RPC，例如：

```text
eth_getBalance
eth_getBlockByNumber
eth_getTransactionByHash
eth_getLogs
eth_call
```

典型请求：

```json
{
  "jsonrpc": "2.0",
  "method": "eth_getBalance",
  "params": ["0xAlice...", "latest"],
  "id": 1
}
```

## 二、Node / RPC / RPC Provider

### Node

Node 是运行 Blockchain Client 的节点实例，参与网络、同步区块并持有本地链上数据和状态。

### RPC

RPC 是 Blockchain Client 对外暴露的调用接口 / 协议机制。RPC 不是 Node 本身。

### RPC Provider

RPC Provider 是建立在节点与 RPC 之上的基础设施服务商，例如 Alchemy、QuickNode、Helius。它提供稳定 Endpoint，并替用户承担节点部署、同步、升级、监控与可用性等工作。

所以：

```text
RPC Provider ≠ RPC ≠ Node
```

## 三、调用路径

不使用商业 Provider 时：

```text
Application / Wallet / Indexer
        ↓
RPC Interface
        ↓
Blockchain Client
        ↓
Node Storage
```

使用商业 Provider 时：

```text
Application / Wallet / Indexer
        ↓
RPC Provider
        ↓
RPC Infrastructure / RPC Interface
        ↓
Blockchain Client
        ↓
Node Storage
```

## 四、为什么应用不直接读取 LevelDB / RocksDB

Node Storage 属于 Client 内部实现细节。不同 Client（例如 Geth、Nethermind、Besu、Erigon）内部数据库布局可以不同。

如果应用直接依赖某个 Client 的 DB Layout，就会与实现强耦合；RPC 提供稳定抽象层：

```text
Application
    ↓
Stable RPC Interface
    ↓
Client Implementation
    ↓
Internal Storage
```

因此应用只需要关心 `eth_getBalance` 等稳定接口，而不需要理解内部 State Trie、KV Layout 或 Storage Engine 的具体组织。

## 五、RPC Provider 的价值

Provider 卖的不是一个 URL，而是稳定访问 Blockchain Infrastructure 的能力。它替用户承担：

- 服务器与节点部署
- Blockchain Client 同步
- SSD 与存储维护
- Client 升级
- 节点监控与运维
- 后续还会涉及可用性、限流、缓存、多节点与多区域等能力

这些后续在 Module 5 继续展开。

## 六、Blockchain Data Engineer 视角

RPC 是 Indexer 获取链上 Raw Data 的常见入口：

```text
Blockchain
    ↓
RPC
    ↓
Raw Block / Transaction / Log
    ↓
Parser
    ↓
Decoder
    ↓
Database
```

Infrastructure 模块之间的分工：

```text
Module 4：Node 内部怎么保存数据？
        ↓
Module 5：外部程序怎么访问 Node？
        ↓
Module 6：为什么普通 Node 的历史能力不够？
        ↓
Module 7：怎么持续把 Node 数据搬进数据平台？
```

## 理解检查

1. “Alchemy 就是一个 Ethereum Node。”这句话有什么问题？请从 Node / RPC / RPC Provider 三个层级解释。
2. 为什么应用程序通常通过 `eth_getBalance` 查询余额，而不是直接打开节点里的 LevelDB / RocksDB 查询？
3. 补完整数据路径：Indexer → RPC Provider → RPC Interface → Blockchain Client → Node Storage。

## 当前状态

🟡 学习中。讲解已完成，理解检查尚未回答。
