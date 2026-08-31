# 第1课｜RPC 到底是什么：从 dApp 请求到 Node Client 的接口层

## Lesson Contract
- 所属 Module：Module 5 — RPC
- 本课核心问题：RPC 在 Blockchain Client、Node、dApp / Wallet / Indexer 之间到底处于什么位置？
- 学完后应能回答：Node、RPC、RPC Provider 有什么区别；为什么应用通过 RPC 而不是直接读取 Node Storage；Indexer 经 Provider 访问链上数据时完整路径是什么。
- 本课明确不展开：Load Balancer、Rate Limit、Cache、Archive Node、多节点集群与 Provider 商业架构深挖。

## 一、RPC 是什么
RPC = Remote Procedure Call，远程过程调用。它让客户端以类似调用函数的方式，请求远程机器执行一个操作。

Ethereum 常见 JSON-RPC 方法：
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

【接口协议视角】Web3.py 等 SDK 会把本地方法调用转换为 JSON-RPC Request，再发送给远端 RPC Endpoint。

## 二、Blockchain Client、Node、RPC、RPC Provider

### Blockchain Client
【软件视角】Geth、Nethermind、Besu、Erigon 等是 Ethereum Client 软件。它们包含 P2P、Block Processing、EVM Execution、State Management、Database、RPC Server 等能力。RPC 只是 Client 的一个对外能力。

### Node
【部署 / 运行实例视角】一台服务器运行 Ethereum Client、连接 Ethereum Network、同步并验证链上数据后，形成一个运行中的 Ethereum Node。

### RPC Interface
【接口视角】RPC 是 Blockchain Client 对外暴露的调用接口 / 协议机制，例如 `eth_getBalance`、`eth_getLogs`、`eth_call`。RPC 本身不是 Node。

### RPC Provider
【基础设施服务视角】Alchemy、QuickNode、Helius 等 RPC Provider 是提供 RPC 基础设施服务的公司 / 平台。它们通常自己运营和管理一批 Node 或其他后端基础设施，再给用户提供统一、稳定的 Endpoint。

核心关系：
```text
RPC Provider ≠ RPC ≠ Node
```

使用 Provider 时的简化逻辑路径：
```text
Application / Wallet / Indexer
        ↓
RPC Provider
        ↓
RPC Interface
        ↓
Blockchain Client
        ↓
Node Storage
```

## 三、为什么应用不直接读取 Node Database
Node Storage 属于 Client 内部实现细节。Geth、Nethermind、Besu、Erigon 的数据库布局、Key 编码、State 存储实现可以不同。

如果应用直接依赖某个 Client 的 DB Layout，就会产生强耦合；通过稳定 RPC Interface，可以把应用与 Client 内部实现隔离：
```text
Application
    ↓
Stable RPC Interface
    ↓
Client Implementation
    ↓
Internal Storage
```

【软件工程视角】RPC 在这里充当抽象层，降低上层 Web3 应用与节点内部存储实现之间的耦合。

## 四、Provider 实际帮用户省掉什么
用户提出疑问：自己作为用户，本来只需要在浏览器或 Python 程序里提交 RPC 请求给 RPC Server，Provider 在流程里到底有什么作用？Provider 在 Ethereum 外部，又怎么知道 Node 的负载？

关键澄清：如果用户自己运行 Geth，并直接开放 `http://your-node:8545`，确实不需要 Provider：
```text
Python
  ↓
your-node:8545
  ↓
Geth RPC Server
  ↓
Blockchain Client
  ↓
Node Storage
```

Provider 的价值是把“自己运行和维护 Node”这件事服务化。Provider 通常不是在外面观察陌生 Node，而是自己运营、监控和调度后端 Node / Node Pool。

因此它可以采集普通基础设施指标，例如：CPU、Memory、Disk I/O、RPC QPS、Latency、Error Rate、Queue Length、Sync Status、Block Height、Peer Count 等，并据此做健康检查、负载均衡和故障转移。

Provider 更接近：
```text
Python / dApp
    ↓
Provider Endpoint / Gateway
    ↓
认证 / Rate Limit / 路由
    ↓
Node A / Node B / Archive Node ...
    ↓
Blockchain Client
    ↓
Node Storage
```

Provider 帮用户省掉的不是“写 RPC 请求”，而是：Node 部署、链同步、硬件和 SSD、Client 升级、节点监控、故障处理、高可用、负载均衡、流量峰值和不同类型 Node 的维护。

一句话：RPC = 怎么问 Node；Node = 真正执行并持有链上数据的实例；RPC Provider = 帮用户运营和管理一批 Node，并提供稳定入口。

## 五、Ethereum Network 里的 Node 与 Provider Node
用户进一步确认：Ethereum Network 里有各种各样的 Node，那么 Alchemy / QuickNode 是否运营这些 Node？

澄清：Ethereum Network 里的 Node 并不都由 Provider 公司运营。个人、交易所、DeFi 项目、Staking 机构、RPC Provider 等都可以运行自己的 Node。

```text
Ethereum Network
│
├── 个人运行的 Node
├── 交易所运行的 Node
├── DeFi 项目运行的 Node
├── Validator / Staking 机构运行的 Node
├── Alchemy 运行的 Node 集群
└── QuickNode 运行的 Node 集群
```

Alchemy 的 Node 与个人自建 Geth Node 在“是否属于 Ethereum Node”这件事上没有本质区别；差别主要在于 Alchemy 会大规模运营、监控并把访问能力商业化。

## 六、自建 Node 后能否自定义 RPC
用户提出：如果自己部署 Ethereum Client Node，再自定义一套个性化 RPC 接口，是否可行？

结论：完全可行，但要区分两种方式。

更常见、推荐的方式：
```text
Application
    ↓
自定义 API / RPC Service
    ↓
标准 Ethereum JSON-RPC
    ↓
Geth / Nethermind
```

也可以修改 Client 源码，增加自定义 RPC Method，但维护和升级成本更高。

重要边界：可以自定义“访问接口”，但不能因为自定义 RPC 就改变 Ethereum 协议、共识或链上 State。自定义 RPC 只是自己的服务接口，不会自动成为 Ethereum 标准 RPC。

## 七、Ethereum Foundation 与 Node 运营
用户询问 Ethereum Network 最大节点运营方是否是 Ethereum Foundation。

课程中的核心结论：Ethereum Foundation 不是 Ethereum Network 的中央节点运营商，也不是“运营大多数 Node”的机构。Ethereum 的设计目标是大量独立运营者共同运行节点。EF 更主要承担协议研究、生态支持、资助和协调等角色。

## 八、Node 的不同角色
用户询问 RPC Node、Validator Node 是否是不同类型 Node，功能是否不同。

结论：这些标签通常按不同维度描述 Node 的主要角色，并不是互斥的严格协议分类。

- Full Node：同步、验证当前链。
- RPC Node：主要面向 dApp / Wallet / Indexer 提供 RPC 查询能力。
- Validator Node：参与 PoS 共识，关注 attestation、block proposal、finality 等。
- Archive Node：按数据保留策略分类，保留完整 Historical State。
- Boot Node：用于 P2P Peer Discovery。

重要视角区分：
```text
RPC Node       = 按服务用途分类
Validator Node = 按共识角色分类
Archive Node   = 按数据保留策略分类
```

同一个运行环境可以同时具备多个角色，例如 Full Node + RPC，或 Archive RPC Node；但生产环境常因资源隔离而拆分不同 workload。

## 九、Blockchain Data Engineer 视角
RPC 是 Indexer 获取链上 Raw Data 的常见入口：
```text
Ethereum
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

Infrastructure 主线：
```text
Module 4：Node 内部怎么保存数据？
        ↓
Module 5：外部程序怎么访问 Node？
        ↓
Module 6：普通 Node 为什么无法满足所有历史查询？
        ↓
Module 7：怎么把 RPC / Node Data 持续搬进数据平台？
```

## 理解检查
### 问题 1
有人说：“Alchemy 就是一台 Ethereum Node，我把 RPC 请求发给它，它帮我查区块链数据。”这句话哪里不准确？请分别说明 Node、RPC、RPC Provider 是什么，以及三者关系。

### 用户回答
“Node 是指运行着 Ethereum Client 的一个节点。RPC 是远程过程调用，它是一种接口。RPC Provider 是指像 Alchemy 这类提供 RPC 服务的节点。然后说 Alchemy 就是一台 Ethereum Node，我把 RPC 请求发给它，因为它作为 RPC Provider 接收到 RPC 请求之后，会在它自己的这个 Node 里面查询区块链的数据，这好像没什么问题。”

### 老师校准
前两点基本正确；关键需要修正 `RPC Provider = Node`。Alchemy 更准确是提供 RPC 基础设施服务的公司 / 平台，通常管理一批 Node / Node Pool，而不是“一台提供 RPC 服务的 Node”。方向上“Provider 收到请求后使用自己运营的节点查询”是对的，但应理解为后端的一组 Node 或合适的基础设施。

### 问题 2
为什么 Web3 应用通常通过 `eth_getBalance` 等 RPC 接口访问节点，而不是直接打开 Geth / Nethermind 的底层 Node Database？

### 用户回答
“从软件工程的角度，不同层面的应用或者说协议，应该通过接口来互相交流，来降低耦合性。这样的话，如果在一个层面的内部有修改，就不会影响到另一个层面。所以，这里 Web3 应用和 Ethereum Node 的直连可以说是属于两套系统。它们通过 RPC 接口来发送和接收信息，当然会比直接查询 Node 底层的数据库更便于维护。”

### 老师判断
✅ 通过。核心抓住了稳定接口隔离系统、降低耦合这一原则。

### 问题 3
使用 Alchemy 的 Ethereum Indexer，至少包含 RPC Provider、RPC Interface、Blockchain Client、Node Storage，请给出正确顺序。

### 用户回答
“正确的顺序应该是：1. RPC Interface 2. RPC Provider 3. Blockchain Client 4. Node Storage。”

### 老师校准
顺序需要调整为：
```text
Python Indexer
      ↓
RPC Provider
      ↓
RPC Interface
      ↓
Blockchain Client
      ↓
Node Storage
```
`RPC Interface` 是逻辑职责层，不一定对应一台独立服务器；Provider Endpoint 先接收请求，再路由给后端能够处理标准 RPC 的 Node / Client。

## 重点总结
1. Blockchain Client 是软件；Node 是运行中的节点实例；RPC 是 Client 的对外调用接口；RPC Provider 是运营和包装节点基础设施的服务平台。
2. Provider 不是 Ethereum 协议必需组件，自建 Node 可以直接开放 RPC。
3. Provider 的商业价值来自 Node 运维、高可用、监控、路由、负载均衡等基础设施能力，而不只是提供一个 URL。
4. 应用通过 RPC 而不是直接读 Node DB，核心原因是抽象和解耦。
5. RPC Node、Validator Node、Archive Node 是不同维度的角色标签，可能重叠。
6. 自建 Node 可以提供自定义 RPC / API，但不会改变 Ethereum 协议或共识。
