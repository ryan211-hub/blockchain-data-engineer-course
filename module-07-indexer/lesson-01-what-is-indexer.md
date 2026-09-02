# 第1课｜Indexer 到底是什么：为什么 Node / RPC 之后还需要一层 Indexer

## Lesson Contract

【Blockchain Data Engineer 视角】

- 所属 Module：Module 7 — Indexer
- 本课核心问题：RPC 已经可以查询 Blockchain，为什么数据公司还需要自己建设 Indexer？
- 学完后应能回答 / 设计：区分 Node / RPC / Indexer / Warehouse 的职责；解释为什么 RPC ≠ Database；解释 Indexer 的核心价值是 Blockchain-oriented Data → Query-oriented Data；画出 Blockchain → Node → RPC → Indexer → Database → API / Dashboard。
- 本课明确不展开：Checkpoint 的具体实现；Reorg 处理算法；幂等写入；Backfill 调度；Parser / Decoder 的代码细节。这些会在 Module 7 后续逐课展开。

## 一、先从一个非常现实的问题开始

假设现在公司要做一个页面：

> 查询某个钱包最近 30 天所有 USDC Transfer。

你已经知道可以调用 Ethereum RPC：

```text
eth_getLogs
```

例如指定：

```text
USDC Contract
+
Transfer Topic
+
Block Range
```

理论上能得到结果。

那么问题来了：

> 既然 RPC 能查，为什么还要 Indexer？

进一步说，如果要做 Wallet History、DEX Dashboard、Token Analytics、NFT Analytics、Aave Positions，能不能全部 Frontend → RPC → Blockchain 直接完成？

技术上，部分场景能。工程上，这通常不是一个好的数据系统。

## 二、RPC 的设计目标并不是“分析数据库”

【RPC 视角】

Module 5 已经学过，RPC 本质上是一个让外部程序访问 Node 能力的接口层，例如：

```text
eth_getBlockByNumber
eth_getTransactionByHash
eth_getLogs
eth_getBalance
eth_call
```

这些 API 的设计方式本质上是在说：

```text
给我一个明确的问题
↓
Node 返回链上的原始答案
```

它很适合 point lookup、range query、raw blockchain access。

## 三、真实业务的问题长什么样？

例如一个 Analyst 会问：

> 最近 30 天每天 Uniswap 的交易量是多少？

这个问题并不是一个 Ethereum RPC 方法。没有 `eth_getUniswapDailyVolume()` 这种接口。

你必须自己做：

```text
获取 Blocks / Logs
↓
识别 Uniswap Contract
↓
识别 Swap Event
↓
Decode
↓
提取 amount
↓
获取 Token decimals
↓
获取价格
↓
计算 USD Volume
↓
按天聚合
```

RPC 给你的，是 Blockchain 原始数据；业务需要的是可查询、可理解、可聚合的业务数据。中间缺了一层，这层就是 Indexer 的核心位置。

## 四、Indexer 最核心的定义

对于 Blockchain Data Engineer：

> **Indexer 是把 Blockchain 原始数据持续读取、解析、结构化并保存成适合查询的数据的系统。**

可以压缩成：

```text
Blockchain-oriented Data
↓
Indexer
↓
Query-oriented Data
```

Blockchain 里有 Block、Transaction、Receipt、Log、Topics、Data；Indexer 把它们转成 transfers、swaps、token_mints、nft_sales、lending_positions，再存进 Postgres、ClickHouse、BigQuery 或其他数据库。

## 五、为什么叫 Indexer？

Blockchain 本身天然按照 Block 顺序组织：

```text
Block 1
├── Tx
├── Tx
└── Tx

Block 2
├── Tx
└── Tx
```

但业务可能问：

```text
Alice 所有交易
USDC 所有 Transfer
Uniswap 所有 Swap
```

因此 Indexer 实际上做了一次：

```text
Blockchain Physical Organization
↓
重新索引
↓
Business Query Organization
```

## 六、一个非常重要的“视角切换”

【协议 / Node 视角】

Blockchain 最重要的是 Block 顺序、Transaction 执行、Consensus、State。

【数据产品视角】

用户通常关心的是：我的钱包发生了什么？我今天赚了多少钱？这个 Token 有多少 Holder？Uniswap 今天交易量多少？Aave 上哪些用户快被清算了？

Indexer 位于：

```text
Blockchain World
↓
Indexer
↓
Business / Analytics World
```

## 七、拿 ERC-20 Transfer 做最简单例子

链上的 Log 可能长这样：

```text
address: 0xUSDC
topics[0]: keccak256("Transfer(address,address,uint256)")
topics[1]: 0xAlice
topics[2]: 0xBob
data: 0x...0f4240
```

Indexer 做：

```text
Raw Log
↓
Recognize Event Signature
↓
Decode Topics / Data
↓
Normalize
```

最终变成可查询字段，例如：

```text
token_address
from_address
to_address
amount_raw
tx_hash
log_index
block_number
```

再结合 Token Metadata，例如 decimals = 6，可以继续得到业务可读 amount。

## 八、所以 Indexer 最低限度做三件事

```text
Extract
↓
Parse / Decode
↓
Store
```

Extract：从 Node / RPC 获取 Block、Transaction、Receipt、Log。

Parse / Decode：把 topics、data、input 转成 from、to、amount、token、event_type。

Store：落到自己的数据库，例如 blocks、transactions、logs、token_transfers、dex_swaps。

一个最简 Indexer：

```text
Ethereum Node
↓ RPC
Python Program
↓
Parse
↓
Postgres
```

## 九、那这不就是 ETL 吗？

从数据工程角度，是的，Indexer 具有明显 ETL 特征：Extract → RPC；Transform → Parse / Decode；Load → Database。

但 Blockchain Indexer 还有一系列传统简单 ETL 不一定天然具备的问题：

```text
Block Continuous Growth
Checkpoint
Reorg
Finality
Idempotency
Backfill
Canonical Chain
```

因此当前可以理解为：

```text
Indexer
≈
Blockchain-specific ETL system
```

真正困难的地方不是 `requests.get()` 或 JSON 解析，而是如何保证链上数据持续、准确、可恢复地进入自己的数据库。

## 十、为什么不能“每次需要数据就重新 RPC 查询”？

假设 Dashboard 每次刷新都重新做过去一年 Uniswap Swap 的 eth_getLogs、Decode、Aggregate，那么每个用户都会重复执行 RPC Scan + Decode + Computation。

Indexer 的思路是：

```text
Blockchain
↓
只处理一次
↓
dex_swaps
↓
数据库保存
```

之后大量用户只需查询数据库。

核心思想：

> **把昂贵的重复计算变成一次数据加工 + 多次廉价查询。**

## 十一、Node 和 Indexer 为什么不能合并？

Node DB 的目标是验证 Blockchain、维护 State、保存 Protocol Data、支持 RPC；它的数据结构围绕 Protocol Correctness、Execution、Efficient Node Operation 设计。

Indexer DB 的目标是 Business Query、Analytics、API Serving、Aggregation、Search。

因此：

```text
Node DB
≠
Analytics DB
```

这和 Oracle OLTP vs Data Warehouse 的职责分离相似。

## 十二、银行系统类比

银行核心系统：

```text
Core Banking DB
↓
ETL / CDC
↓
Data Warehouse
↓
报表 / 风控 / 分析
```

不会让每个经营报表直接扫描核心交易系统。

Blockchain 类似：

```text
Blockchain Node
↓
RPC
↓
Indexer
↓
Analytics Database
```

Node 类似核心业务系统；Indexer 类似 CDC + ETL ingestion layer；Database / Warehouse 类似数据平台。

## 十三、Indexer 和 Warehouse 又有什么区别？

```text
Indexer
=
负责把 Blockchain Data 可靠地变成结构化数据
```

而：

```text
Warehouse
=
负责把结构化数据组织成适合分析和业务使用的数据体系
```

例如 Indexer 产生 raw_blocks、raw_logs、token_transfers、dex_swaps；Warehouse 进一步产生 daily_dex_volume、wallet_daily_balance、token_holder_stats、protocol_daily_tvl。

## 十四、一个完整的数据链路

Module 4：Node 内部如何保存和维护数据。

Module 5：外部系统如何通过 RPC 访问 Node。

Module 6：需要 Historical State 时如何使用 Archive 能力。

Module 7：如何把 Blockchain 数据持续变成自己的数据资产。

完整架构：

```text
Ethereum
↓
Execution Client
↓
RPC
↓
Indexer
↓
Raw / Normalized Database
↓
Warehouse
↓
API / Analytics / Dashboard
```

## 十五、数据公司真正拥有的资产是什么？

【CTO 视角】

两家公司都可以买到 Ethereum RPC，因此 RPC 本身不是竞争壁垒。

真正产生差异的是：你解析了什么、怎么建模、积累了多久、如何保证准确性、能否快速 Backfill、如何处理 Reorg、生成了哪些高质量 Derived Data。

因此公共 Blockchain 是数据源，而 Indexer + Data Model + Historical Dataset + Derived Dataset 才逐渐成为公司的数据资产。

## 十六、为什么 Module 7 是核心模块

Indexer 把 Block、Transaction、Receipt、Log、Event、State、RPC、Reorg、Database 串起来，同时进入 Checkpoint、Idempotency、Backfill、Schema、Recovery、Incremental Processing、Data Consistency 等真正的数据工程问题。

可以把 Module 7 看成：

```text
Blockchain Infrastructure
↓
Indexer
↓
Data Engineering
```

## 十七、本课核心模型

```text
Node
=
维护 Blockchain

RPC
=
访问 Blockchain

Indexer
=
把 Blockchain 转换成可查询的数据

Warehouse
=
把数据进一步组织成可分析、可服务的业务数据
```

以及：

```text
Blockchain
↓
Node
↓
RPC
↓
Indexer
↓
Database
↓
Warehouse
↓
API / Dashboard
```

下一课进入 Indexer 内部：

```text
Raw Block
↓
Parser
↓
Decoder
↓
Normalized Fact
↓
Database
```

## 理解检查

### 问题 1

既然 Ethereum RPC 已经提供了 `eth_getLogs`，为什么一个 DEX Dashboard 不应该让每个用户刷新页面时都直接用 RPC 查询过去一年的 Swap Logs？请从重复计算、RPC 成本、查询性能、数据复用几个角度解释。

**用户回答：**

1. 因为用这个方式的话，会有大量的重复计算，造成资源浪费
2. RPC 的使用成本变高
3. 查询性能低下
4. 数据基本没有复用

**老师判断与校准：**

✅ 通过。四个角度均覆盖：每次刷新都重复扫描和解码会浪费计算资源；RPC 调用量和费用上升；长区间查询性能差；相同历史结果无法作为数据资产复用。Indexer 的价值正是把重复的链上扫描变成一次加工、多次数据库查询。

### 问题 2

请用自己的话解释 Node、RPC、Indexer、Warehouse 分别解决什么问题，尤其解释为什么 Node 自己已经有数据库，我们仍然还需要 Indexer Database？

**用户回答：**

Node  是以太坊客户端的基本节点，和其所有的 Node 一起共同运行着以太坊区块链，维持以太网作为单点计算机。

RPC 解决了对以太坊节点访问的问题。

Index 解决了将面向区块链的数据转化为结构化数据的问题。

Warehouse 则是将结构化数据组织整理成面向主题、面向业务以及面向应用的数据。

Node 里面已经有数据库，但是这个数据库的功能和 Index 是不一样的。

Node 里面的数据库，它的首要任务是保持以太坊节点的正常运行，所以它里面的数据存储是以这个目的为核心的；而 Index Database 则是为了面向数据分析而使用。

这两个数据库的目的是不一样的

**老师判断与校准：**

✅ 通过。核心判断正确：Node DB 的首要目标是协议执行、状态维护和节点运行；Indexer Database 的目标是查询、分析和数据服务。补充一个术语校准：这里应写作 **Indexer**，不是 Index；“以太网”应理解为你想表达的 **Ethereum / 以太坊网络**。

### 问题 3

假设拿到一条 ERC-20 Transfer Log：contract_address、topics[0]、topics[1]、topics[2]、data。Indexer 最核心要做的事情是什么？最终希望把它变成什么样的数据？

**用户回答：**

Index 最核心要做的，是将这条node原生数据经过解析处理，转换成结构化的数据形式。

**老师判断与校准：**

✅ 方向正确并通过。本课需要补齐的最后一步是：Indexer 不只是“转成结构化数据”，还要把链上编码语义还原成可以直接查询的业务字段，例如 `token_address / from_address / to_address / amount_raw / tx_hash / log_index / block_number`，必要时再结合 decimals 等 metadata 转换成业务可读的 amount。也就是说，核心目标是从 Raw Blockchain Encoding 转成 Query-oriented / Business-readable Data。

## 本课总结

1. RPC 是访问 Node 的接口，不是分析数据库。
2. Indexer 的核心职责是把 Blockchain-oriented Data 持续转换为 Query-oriented Data。
3. 最小 Indexer 可以理解为 Extract → Parse / Decode → Store，但生产级 Indexer 还必须处理 Checkpoint、Reorg、Idempotency、Backfill 等问题。
4. Node DB 与 Indexer DB 的设计目标不同：前者服务协议执行和节点运行，后者服务查询、分析与 API。
5. Warehouse 在 Indexer 的结构化事实之上继续做主题化、业务化和聚合。
6. 下一课进入 Raw Block → Parser → Decoder → Normalized Fact → Database 的内部数据流。