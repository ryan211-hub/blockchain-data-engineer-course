# 第3课｜Indexer 的输入对象：Block / Transaction / Receipt / Log 如何进入数据库

## Lesson Contract

【Blockchain Data Engineer 视角】

- 所属 Module：Module 7 — Indexer
- 本课核心问题：Indexer 真正从 Ethereum 拿到哪些基础对象？这些对象之间如何关联，又应该以什么粒度落库？
- 学完后应能回答 / 设计：解释 Block、Transaction、Receipt、Log 分别从哪里获取、一行代表什么、依靠什么字段关联、为什么不能只保存其中一种，以及怎样形成基础 Raw / Parsed 数据层。
- 本课明确不展开：ABI Decode 细节、Checkpoint / Cursor、Reorg 处理、幂等写、Backfill 调度、完整数仓建模。

## 一、先把上一课的 Pipeline 放回来

上一课建立了：

```text
Raw Blockchain Data
↓
Parser
↓
Blockchain Structure
↓
Decoder
↓
Protocol Meaning
↓
Normalizer
↓
Normalized Fact
↓
Database
```

这一课只盯住 Blockchain Structure，也就是 Block、Transaction、Receipt、Log。这是一个 Indexer 最基础的输入对象，可以先把它理解成 Indexer 的“原材料”。

## 二、先建立一张关系图

【Indexer 数据视角】

```text
Block
│
├── Transaction 1
│      │
│      └── Receipt 1
│              ├── Log 1
│              └── Log 2
│
├── Transaction 2
│      │
│      └── Receipt 2
│              └── Log 3
│
└── Transaction 3
       │
       └── Receipt 3
```

关系大致是：1 Block → N Transactions；1 Transaction → 1 Receipt；1 Receipt → N Logs。这是后面设计基础表关系的起点。

## 三、第一类对象：Block

【协议结构视角】

Block 是链的基本顺序单位。一个 Block 里通常包含 block_number、block_hash、parent_hash、timestamp、transactions 等字段。对 Indexer 来说，Block 最重要的作用有两个：第一，它提供时间顺序；第二，它提供 Transaction 的容器和上下文。

## 四、Block 表一行代表什么？

如果落一张 blocks，最基本的粒度通常是：One Row = One Block。

block_number 不应该被理解成绝对唯一身份。Module 3 已经学过 Reorg：同一个高度，可能曾经出现过不同 Block。因此从更稳健的链上身份视角，chain_id + block_hash 更接近 Block 的唯一身份；chain_id + block_number 更像“某个链高度”。

## 五、为什么 Block 表值得单独保存？

Transaction 里虽然也有 block_number，但 Block 自己有 block_hash、parent_hash、timestamp、gas_used、gas_limit、base_fee 等独立属性，而且后面处理 Reorg、Checkpoint、Canonical Chain 都高度依赖 Block。所以 Block 不是可有可无的维度信息，它是 Indexer 的链级骨架。

## 六、第二类对象：Transaction

Transaction 是用户或账户提交给 Ethereum 的一次状态转换请求。对 Indexer 来说，一条 Transaction 常见字段包括 tx_hash、block_number、block_hash、transaction_index、from_address、to_address、value、input、nonce、gas 等。

transactions 表最自然的粒度是：One Row = One Transaction。

## 七、Transaction 怎么关联 Block？

最直接的是 transaction.block_hash → blocks.block_hash。业务查询里也经常用 block_number，但要区分：block_hash 更强，因为它指向具体 Block；block_number 只表示高度。Reorg 场景下，同一个 block_number 可能先后对应不同 block_hash，所以生产级 Indexer 通常不会只依赖 block_number 做唯一关联。

## 八、transaction_index 是什么？

transaction_index 表示这笔 Transaction 在该 Block 中的位置，例如 Block N 中 Tx index 0、1、2、3。block_number + transaction_index 能描述 Block 内部顺序，但 Transaction 自己真正稳定的身份还是 tx_hash。

## 九、第三类对象：Receipt

这是很容易被初学者忽视，但 Indexer 非常依赖的对象。

【EVM 执行视角】

Transaction 被执行以后，会产生执行结果，其中一个核心结果对象就是 Receipt。Receipt 通常包括 transaction_hash、status、gas_used、cumulative_gas_used、contract_address、logs 等。

最重要的是：Receipt 描述的是 Transaction 的执行结果，而不是 Transaction 本身的请求内容。

## 十、Transaction 和 Receipt 为什么不能混在一起？

Transaction 本身告诉你 Alice 想做什么；Receipt 则告诉你执行成功还是失败、用了多少 gas、产生了哪些 logs、是否创建了新 contract。

```text
Transaction = Request / Input
Receipt = Execution Result
```

## 十一、Receipt 表一行代表什么？

如果保存 receipts，粒度通常是：One Row = One Transaction Receipt。因为 Ethereum 中 1 Transaction → 1 Receipt，所以最自然的关联键是 transaction_hash，例如 transactions.tx_hash = receipts.transaction_hash。这是 1:1 关系。

## 十二、为什么 Receipt 对数据工程师很重要？

很多业务信息不在 Transaction 里。例如 Transaction 成功还是失败，要看 receipt.status；一笔 Transaction 产生了哪些 Event，主要从 receipt.logs 进入。所以如果只存 Transaction，不存 Receipt，就会丢失大量 Execution Result。

## 十三、第四类对象：Log

Log 是 Receipt 里的多值对象。一个 Transaction 执行过程中可能产生 0、1、10、100 个 Log。因此 1 Receipt → N Logs。

## 十四、Log 表一行代表什么？

如果存 logs，最自然的粒度是：One Row = One Log。常见字段包括 block_number、block_hash、tx_hash、transaction_index、log_index、contract_address、topics、data。

## 十五、再确认一次 log_index

【Block 内顺序视角】

log_index 表示这条 Log 在整个 Block 的 Log 顺序中的位置，不是 Transaction 内的第几个 Log。

例如：

```text
Block N

Tx A
├── Log index 0
└── Log index 1

Tx B
├── Log index 2
├── Log index 3
└── Log index 4
```

所以之前学过的一个重要主键思路是 chain_id + tx_hash + log_index，用于标识一条链上 Log / Event Fact。

## 十六、为什么 Log 不能只靠 tx_hash？

因为 1 Transaction 可能产生 N Logs。tx_hash 只能定位 Transaction，不能唯一定位某条 Log，需要 log_index 进一步区分。

## 十七、四张基础表的关系

```text
blocks
  1
  ↓
  N
transactions

transactions
  1
  ↓
  1
receipts

receipts
  1
  ↓
  N
logs
```

这就是 Ethereum Indexer 最基础的数据骨架。

## 十八、是不是每个 Indexer 都必须建四张表？

不一定。

【工程实现视角】

有的系统可能把 Receipt 某些字段直接合进 transactions；有的系统可能只保存 raw_receipts JSON；还有系统可能直接保存 raw_blocks 整块 JSON，再后续拆。所以具体表结构不是协议强制的。

当前学习的是逻辑对象和粒度，而不是“生产系统必须严格建 blocks / transactions / receipts / logs 四张表”。

## 十九、逻辑模型和物理存储模型要区分

【逻辑视角】我们有 Block、Transaction、Receipt、Log 四类事实对象。

【物理实现视角】可以是 4 张表，也可以是 Raw JSON + 3 张表，甚至 Object Storage + Columnar DB。具体存储系统留到 Module 11；当前只要求建立对象边界 + 粒度 + 关联关系。

## 二十、Indexer 通常怎么获取这些对象？

一个最简单的 Ethereum Indexer 流程可能是：

```text
1. 获取 Block N
↓
2. 获取 Block 内 Transactions
↓
3. 对每个 tx_hash 获取 Receipt
↓
4. 从 Receipt 中取 Logs
↓
5. 保存基础对象
```

概念上就是 Block → Transactions → Receipts → Logs，然后 Logs → Decoder → Transfer / Swap / ...。

## 二十一、为什么很多 Indexer 从 Block 开始？

因为 Blockchain 本质上是 Block 1、Block 2、Block 3……连续增长的。所以对于同步系统，最自然的问题通常是“我处理到哪个 Block 了？”，而不是“我处理到哪个 Transaction 了？”。这也是为什么后面讲 Checkpoint 时，Block Height 会非常重要。

## 二十二、为什么基础对象必须保留链上上下文？

假设只把一条 Transfer 保存成 Alice、Bob、100 USDC，这远远不够。一个好的链上事实通常还需要 chain_id、block_number、block_hash、block_time、tx_hash、transaction_index、log_index。

因为链上数据不仅要回答“发生了什么”，还必须回答“在哪条链、哪个 Block、哪笔 Transaction、哪个执行位置发生的”。这叫 Provenance，也就是数据来源 / 可追溯性。

## 二十三、一个 Transfer 如何沿着基础对象向上追溯？

假设 token_transfers 里有一行：chain_id = 1，tx_hash = 0xABC，log_index = 27，token = USDC，from = Alice，to = Bob，amount = 100。

可以：

```text
token_transfers
↓ tx_hash + log_index
logs
↓ tx_hash
receipts
↓ tx_hash
transactions
↓ block_hash
blocks
```

这样最终可以从业务 Fact 一直追溯到 Raw Blockchain Context。

## 二十四、银行数据平台类比

业务事实可能只是“客户转账 1000 元”，上层分析也许只需要 customer_id、amount、counterparty、channel，但底层仍然会保留 source_system、batch_id、transaction_id、sequence_no、source_timestamp，因为出现问题时要追源、对账、重放、审计。

Blockchain Data Platform 一样。tx_hash / log_index / block_hash 本质上也是非常重要的 Source Lineage Keys。

## 二十五、主键应该怎么理解？

这节课先只建立基本概念，不做完整 Schema Design。

可以先记：

```text
Block → block_hash
Transaction → tx_hash
Receipt → tx_hash
Log → tx_hash + log_index
```

进入多链系统，则通常还要加 chain_id，所以更稳妥地理解为：

```text
Block: (chain_id, block_hash)
Transaction: (chain_id, tx_hash)
Receipt: (chain_id, tx_hash)
Log: (chain_id, tx_hash, log_index)
```

## 二十六、为什么 Receipt 主键也可以是 tx_hash？

因为 1 Transaction → 1 Receipt，所以 tx_hash 天然能够把两者一一关联。数据库逻辑上可以理解为 transactions PK: tx_hash；receipts PK/FK: tx_hash。当然生产系统还会考虑 chain_id。

## 二十七、Log 和 Event 再区分一次

【链上存储视角】数据库基础层可能保存 logs，也就是 address、topics、data 这种链上原始结构。

【Decoder / 业务语义视角】经过解码以后才变成 Transfer、Swap、Mint、Burn。

所以：

```text
Log
↓ Decode
Event
↓ Normalize
Fact
```

例如 raw_logs → ERC20 Transfer → token_transfers。

## 二十八、为什么不应该直接从 Transaction 生成所有业务事实？

这有时可以，但不完整。因为一笔 Transaction 可能调用 Router → 内部调用多个 Contract → 产生多个 Logs。Transaction input 只告诉你用户最外层调用了什么，而 Logs 往往告诉你执行过程中实际产生了哪些协议事件。

因此很多 Indexer 会同时使用 Transaction + Receipt + Logs，而不是只看 Transaction。

## 二十九、拿一个 Swap 举例

用户 Alice → Uniswap Router。

Transaction：tx_hash = X，to = Router，input = swap...

Receipt：status = success，logs = [...]

Logs 里面可能有 USDC Transfer、Pool Swap、WETH Transfer 等。然后 Decoder：Swap Event → dex_swaps。

所以一笔用户 Transaction 完全可能产生多个 Logs、多个 Event、多个 Fact Rows。这也是为什么 Transaction 粒度 ≠ Event 粒度。

## 三十、本课最重要的粒度表

- Block：One Block
- Transaction：One Transaction
- Receipt：One Transaction Execution Result
- Log：One Log
- token_transfers：One Token Transfer Event
- dex_swaps：One Pool Swap

## 三十一、本课核心模型

```text
Block
1
↓
N
Transaction
1
↓
1
Receipt
1
↓
N
Log
↓
Decoder
↓
Event
↓
Normalizer
↓
Fact
```

四类基础身份：

```text
Block → block_hash
Transaction → tx_hash
Receipt → tx_hash
Log → tx_hash + log_index
```

多链时再加 chain_id。

## 理解检查

### 问题 1

为什么一个 Indexer 不能只保存 transactions，而完全不保存或获取 receipts？请从 Transaction = ?、Receipt = ? 的职责区别来回答。

**用户回答：**

Transaction 里面只包含了发起请求时的一些信息，而 Receipt 保存的是关于这个请求的执行结果的信息，所以其实它们两个是不一样的。

作为 Indexer，它是要将数据解析入库（存入数据库表）的，所以这个执行结果对它来说也是个很重要的东西。结果的正确与否，直接影响着统计的口径，所以 Indexer 不能只保存 transactions，完全不保存 receipts

**老师判断与校准：**

✅ 通过。核心判断准确：Transaction = Request / Input，Receipt = Execution Result。Transaction 主要告诉我们“发起了什么请求”，Receipt 则告诉我们“这个请求执行后发生了什么”。尤其是你提到“结果的正确与否直接影响统计口径”：如果一个失败的 Transaction 也被当成成功业务行为统计，结果会直接失真。Transaction 调用了 swap() 并不等于 Swap 成功发生，必须结合 receipt.status 以及实际 Logs 才能判断执行结果。Indexer 不能只保存“用户想做什么”，还必须保存“链上最终执行成了什么”。

### 问题 2

假设一个 Block 中 Tx A 产生 2 个 Logs，Tx B 产生 3 个 Logs。那么 transactions、receipts、logs 分别增加几行？并说明为什么三张表的粒度不同。

**用户回答：**

Transactions 表增加了 2 行，Receipts 表增加了 2 行，Logs 表增加了 5 行。

这三张表的颗粒度不同：

1. Transactions 表和 Receipts 表：颗粒度是相同的，因为它们事实上是一对一的关系。从实际情况来看，一个 Transaction 的执行结果，就有相对应的一个 Receipt 来保存其结果。
2. Logs 表：情况不一样。Transaction 只是描述了发起请求时的情况，而具体实际的执行路径等信息都保存在 Logs 表里面，所以一个 Transaction 会对应多个 Logs

**老师判断与校准：**

✅ 通过。行数判断完全正确：transactions = 2 行，receipts = 2 行，logs = 5 行。你对 transactions 和 receipts 的 1:1 关系理解准确。需要校准一点：Logs 保存的是执行过程中合约主动 emit 出来的日志 / Event 记录，它能反映很多重要业务结果，但它不是完整执行路径；完整执行路径更接近 Trace / Internal Call。更精确地说：Transaction = 用户发起的请求；Receipt = 这笔请求的整体执行结果；Logs = 执行过程中产生的日志 / Event 记录。因此一笔 Transaction 可以产生多个 Logs，也可能 0 Logs，所以 logs 的粒度天然比 transactions / receipts 更细。

### 问题 3

为什么生产级链上数据表里，经常同时保存 block_number、block_hash、tx_hash、log_index，而不是只保存一个 tx_hash？请从 Block 身份、Reorg、一笔 Transaction 多个 Logs、数据追溯几个角度说明。

**用户回答：**

名词解释：
• blockNumber：指一条链上 block 的高度
• blockHash：在一条链内可以唯一确定一个 block，即 block 的身份标识符
• transactionHash：一笔交易的标识符
• logIndex：在一个 block 里面 log 的 index

在数据表里只保存一个 transactionHash 是不够的：

1. 往细了说：一笔 transaction 会有多个 logs，为了能追溯、确定是哪条 log，我们需要 logIndex
2. 往上层说：只看 transaction 不能得到 block 的信息，如果需要确定是在哪个 block 里面，就需要 blockHash 来唯一确定

为什么不能用 blockNumber 呢？因为 blockNumber 只是指代链的高度，而在这个高度位置上具体是哪一个 block，有可能会发生 reorg 的情况，所以该高度上的 block 可能会发生变化，我们不能仅靠 blockNumber 来确定 block 的身份

**老师判断与校准：**

✅ 通过。四个点已经串起来了：block_number 表示高度，block_hash 标识具体区块，tx_hash 标识交易，log_index 区分同一 Block 内的具体 Log；并且准确说明了为什么 Reorg 场景下不能只靠 block_number。工程化补充：在多链环境里，block_hash、tx_hash 这类标识最好和 chain_id 一起使用，避免跨链命名空间冲突。

## 本课总结

1. Block / Transaction / Receipt / Log 是 Indexer 最基础的链级输入对象。
2. 粒度分别是 One Block、One Transaction、One Transaction Execution Result、One Log。
3. Transaction 与 Receipt 是请求 / 执行结果的 1:1 关系；Receipt 与 Log 是 1:N。
4. block_number 是高度，block_hash 是具体区块身份；Reorg 场景下不能只靠 block_number。
5. tx_hash 只能定位 Transaction；要定位具体 Log 还需要 log_index。
6. chain_id + block_hash / tx_hash / log_index 构成多链环境下更稳妥的身份与追溯键。
7. 业务 Fact 应保留足够的链上上下文，才能支持 Provenance、对账、重放和审计。