# 第1课｜为什么链上数据也需要数据建模：从 Indexer Fact 到可分析模型

## Lesson Contract
【Blockchain Data Engineer 视角】

所属 Module：Module 8 — 数据建模。

本课核心问题：我们在 Module 7 已经把链上数据可靠地写入数据库了，为什么还不能直接拿这些表做分析？为什么还需要重新“建模”？

学完本课应能够解释：
- Indexer 数据和 Analytics Model 的区别。
- 为什么“能存下来”不等于“适合查询”。
- 为什么链上原始结构不能直接作为业务数据模型。
- Raw Fact → Normalized Fact → Analytical Model 大致发生了什么变化。
- 面对分析需求时，为什么应该先问“我要分析什么业务对象”，而不是先问“RPC 返回了哪些字段”。

本课不展开：Fact / Dimension 的正式定义、Grain 的系统设计、Star Schema / SCD、ETL Framework、DEX 业务细节。

---
## 一、从 Module 7 到 Module 8
Module 7 建立的数据链路：
```text
Ethereum Node
     ↓
    RPC
     ↓
 Raw Block
     ↓
 Parser
     ↓
 Decoder
     ↓
 Normalizer
     ↓
 Database
```
Mini Indexer 已经能够完成 Normal Sync、Idempotent Replay、Crash Recovery、Backfill、Reorg Recovery。这说明“数据能够可靠进入数据库”已经解决，但新的问题是：**这些数据到底应该长什么样？**

Module 7 主要解决可靠地产生事实；Module 8 开始解决如何正确组织事实。

---
## 二、Indexer Fact 正确，不代表 Analytics Model 已完成
例如：
```text
erc20_transfers

chain_id
block_number
block_hash
tx_hash
log_index
contract_address
from_address
to_address
value_raw
```
一条 USDC Transfer 可能是：
```text
chain_id       = 1
block_number   = 21000000
tx_hash        = 0xabc...
log_index      = 37
contract       = 0xA0b8...   -- USDC
from_address   = 0xAlice
to_address     = 0xBob
value_raw      = 100000000
```
从 Indexer 角度，`(chain_id, tx_hash, log_index)` 可以唯一标识这个 Log，`value_raw` 忠实保存链上原始值，因此数据可以是正确的。

但产品需求可能是：“查 Alice 最近 30 天转出了多少 USDC，折合多少美元。”这时 `contract_address` 和 `value_raw` 还不是用户直接需要的业务结果。

---
## 三、Blockchain Fact → Data Model → Business Meaning
【链上事实视角】链可能只直接给出：
```text
contract = 0xA0b8...
value    = 100000000
```
而业务需要逐步得到：
```text
这是 USDC
decimals = 6
100000000 = 100 USDC
当前价值约 $100
```
因此建立心智模型：
```text
Blockchain Fact
      ↓
Data Model
      ↓
Business Meaning
```

---
## 四、“忠实保存”与“方便分析”是两个设计目标
银行核心系统的交易流水可能保存 `txn_code`、`currency_code`、`channel_code`、`branch_code` 等字段，核心系统首先服务交易执行和账务准确；数据平台则需要把代码解释成 Transfer、Mobile Banking、Shanghai、Retail 等业务语义。

同理：**源系统的数据结构服务业务执行，而数据模型服务数据消费。**

---
## 五、RPC Schema 不是 Data Model
【RPC 视角】RPC 返回的是节点接口结构，例如 `blockNumber`、`transactionHash`、`topics`、`data`，它解决“节点如何返回信息”。

【Indexer 视角】Decoder 后可以得到：
```text
tx_hash
log_index
token_address
from_address
to_address
amount_raw
```
它主要回答“链上发生了什么”。

【Analytics / Data Warehouse 视角】分析模型进一步可能需要：
```text
chain_id
block_time
tx_hash
wallet_from
wallet_to
token_id
token_symbol
amount
amount_usd
```
它回答“这笔资产转移在业务上意味着什么”。

所以：
```text
RPC Schema
    ↓
Indexer Fact
    ↓
Analytics Model
```

---
## 六、不要把 Source-driven Modeling 当成最终分析模型
Raw Layer 可以按“Source 有什么，Table 就存什么”保留完整源字段，但业务分析需求通常围绕 Wallet、Token、Transfer、Swap、Volume 等 **Business Objects**。

因此 Module 8 的思考顺序开始反过来：
```text
用户要分析什么？
       ↓
业务对象是什么？
       ↓
一行数据代表什么？
       ↓
需要哪些 Fact？
       ↓
需要哪些 Dimension？
       ↓
底层链上数据怎么提供这些信息？
```
可以简化为：
```text
Module 7：Source → Table
Module 8：Question → Model
```

---
## 七、Swap 示例：同一行为可以有不同 Grain
一笔 Router Swap：
```text
USDC
 ↓
Pool A
 ↓
WETH
 ↓
Pool B
 ↓
DAI
```
从 Pool 执行粒度看，可以有 2 行 `pool_swaps`；从用户意图看，可以只有 1 行 `user_swaps`。

核心：**一行数据到底代表什么，是数据建模里最重要的问题之一。**

```text
dex_pool_swaps
一行 = 一个 Pool 执行的 Swap

user_swaps
一行 = 一个用户 Swap 意图
```

---
## 八、Transfer Fact、Token 属性与衍生结果
一个可分析的 Token Transfer 模型可能逐步形成：
```text
fact_token_transfers
chain_id
block_number
block_time
tx_hash
log_index
token_id
from_address
to_address
amount_raw
amount
```
Token 维度可能提供：
```text
dim_tokens
token_id
chain_id
contract_address
symbol
name
decimals
token_type
```
从而 `100000000` 结合 `decimals = 6` 得到 `amount = 100`，再结合价格得到 `amount_usd`。

【Blockchain Data Engineer 视角】Decoder 的职责是解释链上结构本身；Token Symbol、Decimals、标准化金额、USD 估值等逐渐进入 Dimension、Normalization、Valuation、Business Modeling。

---
## 九、Module 8 的设计思维链
```text
需求
 ↓
Business Object
 ↓
Grain
 ↓
Fact
 ↓
Dimension
 ↓
Identity
 ↓
Primary / Unique Key
 ↓
Relationships
 ↓
Query Model
```
面对链上模型，先问三个问题：
1. **一行代表什么？** → Grain
2. **这个对象的唯一身份是什么？** → Identity / Key
3. **这是链上事实，还是附加的业务语义？** → Fact / Dimension / Derived Data

---
## 十、与后续 Module 的关系
```text
Module 7：可靠地产生事实
        ↓
Module 8：正确地组织事实
        ↓
Module 9：稳定地转换事实
        ↓
Module 10：实时地输送事实
        ↓
Module 11：高效地查询事实
        ↓
Module 12：保证事实可信
```

---
## 理解检查与用户回答
### 问题 1
为什么不能说 `erc20_transfers` 已保存所有需要的数据，所以数据建模已经完成？

用户回答：
> 从原始数据的角度来看，这张表已经保存了最细粒度的底层数据。但是，虽然数据的颗粒度是最细、最全的，但并不能说数据建模的工作已经完成。
> 因为数据分析的视角是从业务需求出发，然后一步步拆解、分析需求内容，以及需求所需要的数据。
> 它如何从链上提供的原始数据，得到业务需求所需要的结果。比如金额（amount）来说，链上原始数据是用最细的精度来表示金额，并不能直接适用于业务含义的金额。从原始金额到业务金额，这中间需要经过一个数据处理。

老师判断：通过。

校准：`erc20_transfers` 对 ERC-20 Transfer 这一业务对象可以是细粒度 Normalized Fact，但“最细粒度”不等于“整个链上信息最全”。更严谨地说，它保存了 ERC-20 Transfer 事实的细粒度数据，但尚未补齐业务语义、标准化结果与衍生指标。

### 问题 2
为什么 Router 经过两个 Pool 时，`pool_swaps` 可能 2 行、`user_swaps` 可能 1 行，但两者都正确？

用户回答：
> 因为 pool_swaps 可能有两行，是因为当 swap 的货币对没有现成的时候，需要通过中转的货币来 swap。这种情况下，它会经过多次 swap 才能得到最终的结果，要完整记录这个过程的话，需要有多行数据来表示。
> 从 user 的角度来看，user_swaps 是不用考虑中间过程的。他只要由原币种到目标币种，看结果有没有达成，这是他最重要的意图，所以用一行数据就可以表示。
> 这两种情况数据都是正确的，因为它们正确的判断标准其实是不一样的。
> 从 pool_swaps 来看，它一行数据是代表在 pool 中经过了一次 swap，它就会记录一行数据。
> 而站在用户的角度，一行数据代表的是用户发起的一次 swap。

老师判断：通过。

校准：同一笔链上行为，在不同 Grain 下可以形成不同事实表；只要每张表严格遵守自己定义的一行数据语义，它们都可以是正确的。

### 问题 3
将 `contract_address / amount_raw / token_symbol / decimals / amount / amount_usd` 区分为链上直接事实与业务语义 / 衍生结果。

用户回答：
> 属于更接近链上直接事实的有：contract_address、amount_raw、decimals、amount。
> 已经属于数据模型补充的业务语义有：token_symbol、amount_usd。

老师判断：需要两处校准后通过。

正确分类：
```text
更接近链上直接事实
- contract_address
- amount_raw

数据模型补充的语义 / 属性 / 衍生结果
- decimals
- token_symbol
- amount
- amount_usd
```
`decimals` 虽然可从链上 Token Contract 查询，但不是这条 Transfer Event 本身直接携带的事实；它属于 Token 属性。`amount = amount_raw / 10^decimals` 是标准化后的衍生值，`amount_usd = amount × token_price_usd` 是进一步的估值结果。

建立层次：
```text
链上事实
contract_address
amount_raw
        ↓
维度 / Metadata
decimals
token_symbol
        ↓
标准化结果
amount
        ↓
业务衍生结果
amount_usd
```

---
## 本课结论
**Indexer 解决“如何可靠地产生链上事实”，Data Modeling 解决“如何把这些事实组织成适合业务分析的数据结构”。**

本课三个核心问题：
```text
1. 一行代表什么？
   → Grain

2. 这个对象怎么唯一识别？
   → Identity / Key

3. 哪些是事实，哪些是补充语义或衍生结果？
   → Fact / Dimension / Derived Data
```

理解检查全部完成并通过必要校准，本课正式结束。
