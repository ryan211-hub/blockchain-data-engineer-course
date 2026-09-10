# 第1课｜为什么链上数据也需要数据建模：从 Indexer Fact 到可分析模型

## Lesson Contract

【Blockchain Data Engineer 视角】

所属 Module：Module 8 — 数据建模。

本课只解决一个核心问题：

> **我们在 Module 7 已经把链上数据可靠地写入数据库了，为什么还不能直接拿这些表做分析？为什么还需要重新“建模”？**

学完这一课，你应该能够解释：

1. Indexer 数据和 Analytics Model 的区别。
2. 为什么“能存下来”不等于“适合查询”。
3. 为什么链上原始结构不能直接作为业务数据模型。
4. Raw Fact → Normalized Fact → Analytical Model 大致发生了什么变化。
5. 面对一个分析需求时，为什么应该先问“我要分析什么业务对象”，而不是先问“RPC 返回了哪些字段”。

本课暂时不深入：

- Fact / Dimension 的正式定义——后续课程；
- Grain 的系统设计——后续课程；
- Star Schema / SCD——后续课程；
- ETL Framework——Module 9；
- DEX 业务细节——Module 13。

---

## 一、我们刚刚从哪里走过来？

Module 7 中，我们建立的是这样一条链：

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

最终你的 Mini Indexer 已经能够做到：

```text
Normal Sync
Idempotent Replay
Crash Recovery
Backfill
Reorg Recovery
```

这说明一个问题：

> **数据已经能够可靠进入数据库。**

但注意，这只是第一个问题得到解决。

数据库里有数据以后，下一个问题马上出现：

> **这些数据到底应该长什么样？**

这就是 Module 8。

---

## 二、先看一个很现实的问题

假设你的 ERC-20 Indexer 最终产生这样一张表：

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

比如：

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

从 Indexer 角度，这已经相当不错。

因为我们知道：

```text
(chain_id, tx_hash, log_index)
```

能够唯一标识这个 Log。

并且：

```text
value_raw = 100000000
```

忠实保存了链上的原始值。

所以从 Module 7 的标准来看：

> 数据是正确的。

现在产品经理过来说：

> 帮我查 Alice 最近 30 天转出了多少 USDC，折合多少美元。

问题来了。

数据库里有什么？

```text
contract_address
value_raw
```

但是产品经理问的是：

```text
USDC
100 USDC
$100
```

这中间明显缺东西。

---

## 三、这里第一次出现了两个“世界”

这是 Module 8 非常重要的分界。

【链上事实视角】

链告诉我们的可能只是：

```text
contract = 0xA0b8...
value    = 100000000
```

但链不会直接告诉你：

```text
这是 USDC
Decimals = 6
100000000 = 100 USDC
USDC 属于 Stablecoin
当前价值约 $100
```

这些已经开始进入：

> **数据模型和业务语义。**

所以可以先建立一个很重要的心智模型：

```text
Blockchain Fact
      ↓
Data Model
      ↓
Business Meaning
```

Module 7 主要处理左边。

Module 8 开始处理中间。

Module 13 以后会大量处理右边。

---

## 四、“忠实保存”与“方便分析”是两个完全不同的设计目标

这点和你原来做银行数据中台其实非常接近。

假设银行核心系统有一张交易流水：

```text
account_no
txn_code
currency_code
amount
channel_code
branch_code
txn_time
```

核心系统设计它时首先关心的是：

```text
交易是否准确
账务是否正确
状态是否一致
```

但是经营分析的人问：

> 上海地区，本月手机银行渠道的个人客户转账金额是多少？

这时你往往不会让分析师自己理解：

```text
txn_code = 1527
channel_code = 03
branch_code = 310001
customer_type = ...
```

数据平台会把这些底层代码转成：

```text
transaction_type = Transfer
channel = Mobile Banking
region = Shanghai
customer_segment = Retail
```

为什么？

因为：

> **源系统的数据结构服务于业务执行，而数据模型服务于数据消费。**

区块链一模一样。

---

## 五、RPC Schema 不是 Data Model

这是这一课最需要建立的认识。

【RPC 视角】

Ethereum RPC 可能给我们：

```json
{
  "blockNumber": "...",
  "transactionHash": "...",
  "topics": [...],
  "data": "0x..."
}
```

这是：

> RPC Interface Schema

它解决的问题是：

> 节点应该如何把信息返回给客户端？

不是：

> 数据分析师应该如何查询 Token Transfer？

【Indexer 视角】

Indexer 会把它 Decoder 成：

```text
tx_hash
log_index
token_address
from_address
to_address
amount_raw
```

已经好很多。

但它主要回答：

> 链上发生了什么？

【Analytics / Data Warehouse 视角】

我们最后可能希望得到：

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

它回答的是：

> 这笔资产转移在业务上意味着什么？

因此你可以看到三层：

```text
RPC Schema
    ↓
Indexer Fact
    ↓
Analytics Model
```

它们不是同一个东西。

---

## 六、为什么不能直接把 RPC JSON 全部扔进一张表？

一个很常见的初学者思路是：

> RPC 有什么字段，我就把什么字段存下来。

比如：

```text
transactions

hash
blockNumber
from
to
input
value
gas
gasPrice
nonce
type
maxFeePerGas
...
logs
traces
...
```

字段越来越多。

最后甚至想着：

> 要不然全部塞进去，省得以后缺字段。

这是典型的：

> **Source-driven Modeling**

也就是：

```text
Source 有什么
       ↓
Table 就存什么
```

这种方式做 Raw Layer 没问题。

但是作为分析模型问题非常大。

因为真实需求通常不是：

> 给我看看 RPC 返回的数据。

而是：

> 这个 Wallet 最近交易了什么 Token？

或者：

> 这个 Wallet 最近一个月净流入多少 USDC？

或者：

> 哪些 Wallet 在 Uniswap 上 Swap 过 ETH → USDC？

或者：

> 今天 Ethereum 上 USDC Transfer Volume 是多少？

注意这些问题里的名词：

```text
Wallet
Token
Transfer
Swap
Volume
```

它们才是真正的：

> **Business Objects**

---

## 七、所以数据建模的起点发生了改变

Module 7 的思考顺序通常是：

```text
Blockchain
    ↓
RPC 返回什么？
    ↓
怎么 Parse？
    ↓
怎么 Decode？
    ↓
怎么存？
```

到了 Module 8，我们逐渐反过来：

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

这是一种非常重要的思维转换。

可以简单称为：

```text
Module 7
Source → Table

Module 8
Question → Model
```

严格来说 Module 7 也并非完全“Source → Table”，因为我们已经做过 Normalizer。

但这个对比有助于建立阶段边界。

---

## 八、举一个你已经非常熟悉的例子：Swap

Module 3 我们曾经区分过：

```text
Pool Swap
```

和：

```text
User Swap
```

假设 Alice：

```text
1000 USDC
    ↓
USDC/WETH Pool
    ↓
WETH/DAI Pool
    ↓
3000 DAI
```

底层可能产生两个：

```text
Pool Swap Event
```

因此从 Log / Pool Fact 粒度：

```text
2 rows
```

但从用户业务意图：

```text
Alice
1000 USDC → 3000 DAI
```

可能只有：

```text
1 User Swap
```

这里发生了什么？

链没有错。

Indexer 也没有错。

两张表甚至都可以是正确的。

区别只是：

> **Grain 不一样。**

虽然 Grain 会在后面正式讲，但今天你先记住一句话：

> **一行数据到底代表什么，是数据建模里最重要的问题之一。**

例如：

```text
dex_pool_swaps
一行 = 一个 Pool 执行的 Swap

user_swaps
一行 = 一个用户 Swap 意图
```

这两张表不能因为都叫“Swap”就混在一起。

---

## 九、再看 ERC-20 Transfer

假设 Alice 给 Bob 转 100 USDC。

底层 Event：

```text
Transfer(
 Alice,
 Bob,
 100000000
)
```

Indexer Fact：

```text
chain_id
tx_hash
log_index
contract_address
from_address
to_address
amount_raw
```

如果进入可分析的数据模型，可能逐渐形成：

```text
fact_token_transfers
────────────────────────
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

再通过 Token Dimension：

```text
dim_tokens
────────────────
token_id
chain_id
contract_address
symbol
name
decimals
token_type
```

于是：

```text
100000000
```

通过：

```text
decimals = 6
```

得到：

```text
amount = 100
```

通过：

```text
symbol = USDC
```

用户终于看到：

```text
100 USDC
```

注意：

Indexer 并没有“做错”。

而是：

> Indexer Fact 还没有完成 Analytics Semantic。

---

## 十、这里有一个非常重要的职责边界

【Blockchain Data Engineer 视角】

你以后设计平台时，不应该要求：

```text
Decoder
```

负责所有业务分析。

Decoder 最核心的问题是：

> 这个链上结构是什么意思？

例如：

```text
topic0
↓
Transfer(address,address,uint256)
```

然后：

```text
topics[1] → from
topics[2] → to
data      → value_raw
```

这是 Decoder。

但：

```text
USDC
Stablecoin
Decimals = 6
100 USDC
$100
```

已经逐渐进入：

```text
Dimension
Normalization
Valuation
Business Modeling
```

所以 Module 7 的那条链现在可以继续往下延伸：

```text
Node
 ↓
RPC
 ↓
Fetcher
 ↓
Parser
 ↓
Decoder
 ↓
Normalized Fact
 ↓
─────────────────────
        Module 8
─────────────────────
 ↓
Data Model
 ↓
Fact + Dimension
 ↓
DWS / Analytical Model
 ↓
ADS / Product Metric
```

这基本就是我们接下来几个月要搭起来的链路。

---

## 十一、为什么这一步对你的职业转型尤其重要

Indexer 会写的人不少。

SQL 会写的人也很多。

但是 Blockchain Data Engineer 真正比较有价值的能力在中间：

```text
懂 Blockchain
      +
懂 Data Engineering
      +
懂 Business Semantics
```

例如产品说：

> 给我统计过去 7 天活跃 Wallet。

你必须马上意识到：

“Active Wallet”不是链上原生对象。

必须定义：

```text
什么叫 Active？

发送 Transaction？
收到 Transfer？
发生 Swap？
调用 Contract？
只有成功 Transaction？
内部 Trace 算不算？
```

于是问题已经从：

```text
SELECT COUNT(DISTINCT address)
```

变成：

> **业务定义 + 数据模型设计。**

SQL 反而是最后一步。

这和银行数据平台中的“指标口径”其实非常类似。

---

## 十二、Module 8 我们真正要训练什么

Module 8 不是主要教：

```sql
CREATE TABLE ...
```

而是训练你看到一个需求之后形成这样一条思维链：

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

比如：

> 查询某个 Wallet 最近一个月 Token Transfer。

你的思考不应该首先是：

```text
我要建哪些 varchar(42)
```

而应该先是：

```text
业务对象：
Wallet
Token
Transfer

核心 Fact：
Token Transfer

Grain：
一个 canonical Token Transfer Event

Identity：
(chain_id, tx_hash, log_index)

相关 Dimension：
Token
Chain
可能还有 Wallet Label
```

表字段只是最后的物理表现。

---

## 十三、今天先建立一个最简单的判断方法

以后碰到链上数据模型，可以先问三个问题。

第一：

> **一行代表什么？**

比如：

```text
一行 = 一个 Transaction
```

还是：

```text
一行 = 一个 Transfer
```

还是：

```text
一行 = 一个 Wallet 每日余额
```

第二：

> **这个对象的唯一身份是什么？**

例如：

```text
Transaction:
(chain_id, tx_hash)

Log:
(chain_id, tx_hash, log_index)

Token:
(chain_id, contract_address)
```

第三：

> **这是链上事实，还是附加的业务语义？**

例如：

```text
amount_raw = 链上事实

decimals = Token 属性

amount = 标准化结果

symbol = Token 属性

amount_usd = 衍生业务值
```

如果这三个问题都能回答，数据模型通常已经有了骨架。

---

## 十四、把 Module 7 和 Module 8 接起来

Module 7 最后我们做的是：

```text
process_block(block)
```

不管数据来自：

```text
Realtime
Backfill
Reorg Replay
```

最终都进入统一 Processing Engine。

它解决的是：

> **同一个链上事实，无论通过什么路径进入系统，都应该得到一致的数据结果。**

Module 8 接下来继续追问：

> **这个“一致的数据结果”应该被组织成什么样的数据模型，才能被不同分析场景反复使用？**

所以 Module 8 并不是突然换了一个主题。

它实际上正好接在 Mini Indexer 后面：

```text
Module 7
可靠地产生事实

        ↓

Module 8
正确地组织事实

        ↓

Module 9
稳定地转换事实

        ↓

Module 10
实时地输送事实

        ↓

Module 11
高效地查询事实

        ↓

Module 12
保证事实可信
```

这就是第三阶段 Data Engineering 的整体主线。

---

## 本课理解检查

暂时不要去查资料，直接按你的理解回答。

**问题 1**

Indexer 已经有下面一张表：

```text
erc20_transfers

chain_id
tx_hash
log_index
contract_address
from_address
to_address
value_raw
```

为什么我不能简单地说：

> “这张表已经保存了所有需要的数据，所以数据建模已经完成了。”

你可以结合 `100000000 USDC raw value` 的例子解释。

### 用户回答 1

> 从原始数据的角度来看，这张表已经保存了最细粒度的底层数据。但是，虽然数据的颗粒度是最细、最全的，但并不能说数据建模的工作已经完成。
> 因为数据分析的视角是从业务需求出发，然后一步步拆解、分析需求内容，以及需求所需要的数据。
> 它如何从链上提供的原始数据，得到业务需求所需要的结果。比如金额（amount）来说，链上原始数据是用最细的精度来表示金额，并不能直接适用于业务含义的金额。从原始金额到业务金额，这中间需要经过一个数据处理

### 老师校准 1

问题一回答正确，而且你抓到了这一课最核心的转换：**“源数据完整”不等于“分析模型完整”。**

你这段里最重要的是两层区分。

第一层是【Indexer / Source 视角】：

`erc20_transfers` 保存了链上最接近事实的数据，例如：

```text
contract_address
from_address
to_address
value_raw
```

从“是否忠实记录链上发生了什么”来看，这张表可以已经足够正确。

第二层是【Analytics / Business 视角】：

业务并不直接消费 `value_raw = 100000000`。

以 USDC 为例，还需要知道：

```text
contract_address
      ↓
识别这是 USDC
      ↓
decimals = 6
      ↓
100000000 / 10^6
      ↓
amount = 100 USDC
```

所以你说的“中间需要经过一个数据处理”是准确的。再往后甚至还可能继续：

```text
100 USDC
   ↓
价格 / 估值数据
   ↓
amount_usd
```

这里我补一个措辞上的校准：

你说“这张表保存了最细粒度的底层数据”基本可以，但以后最好避免把“最细粒度”和“最全”直接画等号。

例如 `erc20_transfers` 对 **ERC-20 Transfer 这个业务对象** 来说已经很细，但它并没有保存整个 Transaction 的 Trace、所有 Logs、State Change 等全部链上信息。

因此更严谨的表达是：

> 这张表已经保存了 ERC-20 Transfer 这一类事实的细粒度 Normalized Fact，但还没有补齐业务分析需要的语义、标准化结果和衍生指标，因此数据建模尚未完成。

这个表述以后面试时也更稳。

问题一：**通过。**

---

**问题 2**

下面两个表：

```text
pool_swaps
一行 = 一个 Pool Swap Event
```

和：

```text
user_swaps
一行 = 一个用户完整 Swap 意图
```

假设某笔 Router 交易经过两个 Pool。

为什么这两个表一张可能保存 2 行，另一张可能保存 1 行，但它们都可以是“正确”的？

### 用户回答 2

> 因为 pool_swaps 可能有两行，是因为当 swap 的货币对没有现成的时候，需要通过中转的货币来 swap。这种情况下，它会经过多次 swap 才能得到最终的结果，要完整记录这个过程的话，需要有多行数据来表示。
> 从 user 的角度来看，user_swaps 是不用考虑中间过程的。他只要由原币种到目标币种，看结果有没有达成，这是他最重要的意图，所以用一行数据就可以表示。
> 这两种情况数据都是正确的，因为它们正确的判断标准其实是不一样的。
> 从 pool_swaps 来看，它一行数据是代表在 pool 中经过了一次 swap，它就会记录一行数据。
> 而站在用户的角度，一行数据代表的是用户发起的一次 swap。

### 老师校准 2

问题二通过。

你已经把这里最关键的概念说出来了：**两张表的“正确”，取决于它们各自定义的 Grain。**

【Pool / Protocol 执行视角】

`pool_swaps` 的一行代表：

> 某一个 Pool 实际执行了一次 Swap。

所以路径：

```text
USDC → WETH → DAI
```

如果经过两个 Pool，就可能产生：

```text
Pool A：USDC → WETH
Pool B：WETH → DAI
```

因此是 2 行。

【User / Business Intent 视角】

`user_swaps` 的一行代表：

> 用户发起的一次完整 Swap 意图。

用户关心的是：

```text
输入：USDC
输出：DAI
```

中间经过几个 Pool，是执行路径，不是用户意图本身。

所以可以是 1 行。

你最后一句尤其重要：

> “这两种情况数据都是正确的，因为它们正确的判断标准其实是不一样的。”

在数据建模里，可以把它进一步精确成：

> **同一笔链上行为，在不同 Grain 下可以形成不同的事实表；只要每张表都严格遵守自己定义的一行数据语义，它们就都可以是正确的。**

问题二：**通过。**

---

**问题 3**

下面这些字段：

```text
contract_address
amount_raw
token_symbol
decimals
amount
amount_usd
```

你先不用严格按照 Fact / Dimension 术语分类。

只需要按今天这节课的思路，尝试说明：

> 哪些更接近“链上直接事实”，哪些已经开始属于“数据模型补充的业务语义或衍生结果”？

### 用户回答 3

> 属于更接近链上直接事实的有：
> • contract_address
> • amount_raw
> • decimals
> • amount
>
> 已经属于数据模型补充的业务语义有：
> • token_symbol
> • amount_usd

### 老师校准 3

问题三需要两处校准：`decimals` 和 `amount` 不应归到“链上直接事实”。你的整体分类思路已经成立。

更准确的分类应该是：

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

关键是你误判了 `decimals` 和 `amount`。

`decimals` 虽然也是“链上的信息”，但它不是这条 Transfer Event 本身直接携带的事实。对于 ERC-20，它通常属于 Token Contract 的 metadata / state 属性。例如：

```text
USDC
decimals() = 6
```

所以从【Transfer Fact 视角】看：

```text
contract_address = Transfer Event 直接事实
amount_raw       = Transfer Event 直接事实

decimals         = Token 属性
```

这里要特别注意一个以后会反复出现的区别：

> “存在于链上” ≠ “属于这条事实记录的直接字段”。

`decimals` 可以从链上查询，但它属于 Token 维度属性，而不是某一次 Transfer 的 Event Fact。

而 `amount` 更明显是一个计算结果：

```text
amount_raw = 100000000
decimals   = 6

amount
= amount_raw / 10^decimals
= 100
```

因此：

```text
amount_raw
```

是源事实，而：

```text
amount
```

是 Normalization 后得到的衍生值。

再往后：

```text
amount_usd
= amount × token_price_usd
```

属于更进一步的估值衍生结果。

所以可以建立这一条非常有用的层次：

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

这其实已经开始提前碰到 Module 8 后续一个很重要的问题：

> 哪些字段应该存在 Fact Table，哪些应该来自 Dimension，哪些又应该在更高层的数据处理中计算出来？

这个问题我们后面会正式展开。

你的问题三虽然有两个字段需要调整，但核心判断已经建立起来了：你已经能够区分“链直接记录的数据”和“为了业务消费而补充、转换出来的数据”。

因此本课理解检查判定：**通过，校准完成。**

---

## 第 1 课最终结论

这一课真正需要留下的只有一句话：

> **Indexer 解决“如何可靠地产生链上事实”，Data Modeling 解决“如何把这些事实组织成适合业务分析的数据结构”。**

对应两种完全不同的出发点：

```text
Indexer
Blockchain / RPC
      ↓
Fact

Data Modeling
Business Question
      ↓
Business Object
      ↓
Grain
      ↓
Fact / Dimension
      ↓
Analytical Model
```

以及今天第一次正式建立的数据建模三个问题：

```text
1. 一行代表什么？
   → Grain

2. 这个对象怎么唯一识别？
   → Identity / Key

3. 哪些是事实，哪些是补充语义或衍生结果？
   → Fact / Dimension / Derived Data
```

下一课我们会正式进入这三个问题中的第一个，而且它是整个 Module 8 最重要的概念之一：

> **Module 8 第 2 课｜Grain：为什么“一行代表什么”是数据建模的第一原则。**