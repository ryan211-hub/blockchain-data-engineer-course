# Module 13 第 1 课｜为什么会有 DEX：从 Order Book 到 AMM

## Lesson Contract

所属 Module：Module 13 — DEX（Decentralized Exchange，去中心化交易所）

本课核心问题：

> Why does a decentralized exchange need a different market structure from a traditional exchange?

学完以后，你应该能够解释：

- DEX（Decentralized Exchange，去中心化交易所）到底解决什么问题。
- CEX（Centralized Exchange，中心化交易所）和 DEX 的核心差异。
- 为什么传统 Order Book（订单簿）模型在链上早期并不理想。
- AMM（Automated Market Maker，自动做市商）为什么会出现。
- Liquidity Pool（流动性池）到底在做什么。
- LP（Liquidity Provider，流动性提供者）为什么愿意提供资产。
- Swap 为什么本质上是在“与一个 Pool 交易”。
- 从 Blockchain Data Engineer 视角，DEX 最重要的数据对象是什么。
- 为什么后续做 DEX Analytics 时，不能只看 Transaction。

本课暂不展开：

- Constant Product Formula `x * y = k` 的数学推导；
- Impermanent Loss；
- Concentrated Liquidity；
- Uniswap v3 Tick；
- MEV（Maximal Extractable Value，最大可提取价值）；
- Aggregator / Router；
- DEX 收益率模型。

这些放到后续课程。

---

## 一、先回到最根本的问题：为什么需要交易所？

假设世界里只有 Token A 和 Token B。

Alice 有：

```text
100 Token A
```

她想换成：

```text
Token B
```

最原始的方法是找 Bob：

```text
Alice:
I want to sell 100 A.

Bob:
I want to buy 100 A.
```

如果双方：

```text
price
amount
timing
```

全部匹配，

交易才能发生。

这就是最原始的：

> Peer-to-peer matching.

问题是效率很低。

所以传统金融里产生了 Exchange：

> Bring buyers and sellers into one market.

---

## 二、CEX 是怎么解决这个问题的？

CEX（Centralized Exchange，中心化交易所），例如传统意义上的 Binance / Coinbase 这类平台，通常维护一套中心化市场。

你看到：

```text
BTC / USDT
```

背后可能有：

```text
Buy Orders
Sell Orders
```

也就是：

```text
Order Book
```

例如：

```text
Sell:
$60,100  1 BTC
$60,050  2 BTC

Buy:
$60,000  3 BTC
$59,950  5 BTC
```

交易所负责：

```text
receive orders
match orders
update balances
settle internally
```

核心是：

> Buyer and seller are matched by the exchange.

---

## 三、Order Book 的本质是什么？

Order Book 其实就是：

> A collection of trading intentions.

例如：

```text
Alice:
buy 1 ETH at $3,000

Bob:
sell 1 ETH at $3,000
```

当价格匹配：

```text
trade executes
```

所以传统市场结构可以简化为：

```text
Buyer
↓
Order

Order Book

Sell Order
↑
Seller
```

然后：

```text
Matching Engine
```

负责撮合。

---

## 四、为什么 Blockchain 上不能直接照搬？

技术上可以。

事实上，现在也有 On-chain Order Book。

但在早期 Ethereum 上，它有明显问题。

### 1. Every order modification costs money

如果 Alice：

```text
place order
cancel order
change order
```

这些操作如果全部上链，

都可能意味着：

```text
Transaction
→ Gas Fee
```

这对于高频挂单非常昂贵。

---

## 五、传统 Market Maker 的行为很“高频”

Market Maker（做市商）可能不断修改：

```text
buy price
sell price
amount
```

例如：

```text
10:00:00
ETH bid = 3000

10:00:01
ETH bid = 3001

10:00:02
ETH bid = 2999
```

在传统交易所，

这是服务器里的高速更新。

但在 Ethereum 早期，如果每次修改都变成：

```text
on-chain transaction
```

成本会非常高。

所以出现一个关键问题：

> Can we create liquidity without constantly maintaining an on-chain order book?

AMM 就是对这个问题的一种回答。

---

# 六、AMM 到底改变了什么？

AMM（Automated Market Maker，自动做市商）改变了交易对手的结构。

传统 Order Book：

```text
Trader
↕
Other Traders / Market Makers
```

AMM：

```text
Trader
↕
Liquidity Pool
```

也就是说：

> You are not directly waiting for another trader to take the opposite side.

而是：

> You trade against a pool of assets.

这是理解 DEX 最关键的一步。

---

## 七、Liquidity Pool 是什么？

假设有一个：

```text
ETH / USDC Pool
```

里面有：

```text
100 ETH
300,000 USDC
```

那么这个 Pool 本质上就是一个 Smart Contract（智能合约）持有的资产集合。

可以先抽象为：

```text
ETH Reserve
+
USDC Reserve
```

Trader 来进行 Swap：

```text
give ETH
receive USDC
```

或者：

```text
give USDC
receive ETH
```

Pool 的资产余额随交易发生变化。

---

## 八、一个最简单的 Swap

假设：

```text
Pool:
100 ETH
300,000 USDC
```

Alice 想：

```text
sell ETH
buy USDC
```

从资金流看：

```text
Alice
↓ ETH

Liquidity Pool

↓ USDC
Alice
```

所以从数据工程角度：

```text
token_in = ETH
token_out = USDC
```

但是注意：

> Swap is not merely a Transfer pair.

因为它还有：

```text
price
fee
pool state
liquidity
execution path
```

这些 Business Semantics（业务语义）必须从 Protocol 角度理解。

---

# 九、那价格从哪里来？

这是 AMM 的核心。

Order Book 中：

```text
price
```

来自买卖双方挂单。

例如：

```text
seller asks $3000
buyer bids $3000
```

而 AMM 没有传统意义上的订单簿。

它需要通过：

```text
mathematical rule
```

根据 Pool 当前状态自动决定：

```text
how much token_out
```

这就是为什么叫：

> Automated Market Maker

“Automated”的核心不是“自动交易”，而是：

> Market-making price and execution are determined algorithmically.

---

## 十、[Protocol 视角] AMM 是一个 State Machine

你之前已经学过：

```text
Transaction
→ Execution
→ State Change
```

现在放进 DEX：

交易前：

```text
Pool State
ETH   = 100
USDC  = 300,000
```

Alice Swap。

交易后：

```text
Pool State
ETH   = 101
USDC  = some lower number
```

所以一次 Swap 本质上是：

> A state transition of a liquidity pool.

这个视角对 Blockchain Data Engineer 非常重要。

---

# 十一、那谁把钱放进 Pool？

LP（Liquidity Provider，流动性提供者）。

例如 Bob：

```text
deposit ETH
+
deposit USDC
```

到 Pool。

他不是来 Swap。

他的角色是：

> Provide inventory for other traders to trade against.

也就是提供 Liquidity（流动性）。

---

## 十二、LP 为什么这么做？

因为 Trader 每次 Swap 往往会支付：

```text
Trading Fee
```

部分费用会归 LP。

所以基本经济关系是：

```text
LP
provides capital
↓
Pool gets liquidity
↓
Traders can swap
↓
Traders pay fees
↓
LP earns fees
```

所以 DEX 的核心角色可以简化为：

```text
Trader
→ wants execution

LP
→ provides liquidity

Protocol
→ defines market rules
```

---

# 十三、这和 CEX 最大的结构差异是什么？

CEX 更接近：

```text
Buyer
+
Seller
+
Order Book
+
Matching Engine
```

经典 AMM DEX 更接近：

```text
Trader
+
Liquidity Pool
+
Pricing Formula
+
Smart Contract
```

所以可以把两者放一起：

```text
CEX
Trader
↓
Order
↓
Order Book
↓
Matching Engine
↓
Trade
```

对比：

```text
AMM DEX
Trader
↓
Swap Transaction
↓
Pool Contract
↓
Pricing Formula
↓
Pool State Change
```

---

# 十四、[Data Engineer 视角] DEX 的数据对象开始变了

前面做通用 Blockchain Data 时，我们关注：

```text
Block
Transaction
Log
Transfer
```

但进入 DEX Protocol 后，

你必须开始关注：

```text
Pool
Swap
Liquidity
Token Pair
LP Position
Fee
Price
Volume
```

这是第四阶段和前三阶段的明显区别。

前三阶段更关注：

> How does blockchain data infrastructure work?

现在开始关注：

> What does the business event actually mean?

---

## 十五、为什么只解析 Transaction 不够？

假设一笔 Transaction：

```text
Alice calls Router
```

里面可能发生：

```text
ETH → USDC
USDC → DAI
```

也可能包含：

```text
Transfer
Swap
Fee
Liquidity movement
```

如果你只看：

```text
Transaction.from
Transaction.to
Transaction.value
```

你无法准确回答：

> Alice 到底完成了什么 Swap？

因此 DEX Data Engineering 必须进入：

```text
Event / Log
+
Protocol Semantics
```

---

# 十六、Swap Event 为什么重要？

以 AMM Protocol 为例，

Smart Contract 通常会发出类似：

```text
Swap
```

Event。

里面可能包含：

```text
sender
recipient
amount_in
amount_out
pool
token information
```

具体字段取决于 Protocol。

从 Data Engineer 视角：

> Event is often the most convenient structured evidence of a protocol action.

但仍然要记得你之前已经学过的原则：

> Event is evidence emitted by protocol execution, not the whole execution itself.

---

## 十七、Swap 和 Transfer 的 Grain 不一样

这是很重要的数据建模问题。

一笔 Swap：

```text
ETH → USDC
```

底层可能产生多个 Transfer。

例如：

```text
Alice → Pool: ETH

Pool → Alice: USDC

Pool → Fee Recipient: fee
```

所以：

```text
Transfer Grain
≠
Swap Grain
```

不能看到两个 Token Transfer 就自动认为：

```text
one swap
```

---

## 十八、[Data Modeling 视角] Swap Fact 的 Grain

一个典型 `dex_swaps` Fact Table，首先要明确 Grain：

> One row represents one protocol-level swap event.

可能的核心字段：

```text
chain_id
block_number
block_time
tx_hash
log_index
protocol
pool_address
token_in
token_out
amount_in
amount_out
trader
recipient
```

后面还可以加入：

```text
amount_usd
price
fee
router
```

但 Grain 必须先确定。

---

# 十九、DEX 的核心 Business Object

现在先建立一张地图：

```text
DEX Protocol
│
├── Pool
│
├── Token Pair
│
├── Swap
│
├── Liquidity
│
├── LP
│
└── Fee
```

后续几课就是把这些对象逐步拆开。

---

## 二十、Pool 为什么是核心对象？

因为对于 AMM：

```text
Price
Liquidity
Swap Capacity
Fee Generation
```

基本都围绕 Pool。

所以很多 DEX Analytics 都以：

```text
pool_address
```

作为非常重要的 Dimension（维度）。

例如：

```text
daily volume by pool
fee revenue by pool
liquidity by pool
swap count by pool
```

---

# 二十一、Protocol 与 Pool 不要混淆

Uniswap 可以是：

```text
Protocol
```

但：

```text
ETH / USDC 0.3% Pool
```

是：

```text
Pool
```

一个 Protocol 可以有很多 Pool。

例如：

```text
Uniswap
├── ETH / USDC
├── ETH / USDT
├── WBTC / ETH
└── ...
```

所以：

```text
Protocol Grain
≠
Pool Grain
```

---

## 二十二、DEX 为什么对 Blockchain Data Engineer 特别重要？

因为它几乎把之前学过的所有能力都连接起来：

```text
RPC
↓
Indexer
↓
Log Decoder
↓
Swap Fact
↓
Kafka
↓
Postgres
↓
ClickHouse
↓
DWS / ADS
↓
Volume / Price / Liquidity Dashboard
```

同时还有：

```text
Reorg
Backfill
Decoder Bug
Multi-sink Repair
```

所以 DEX 是非常合适的真实业务训练场。

---

# 二十三、从公司角度看，DEX 数据有什么价值？

一个 DEX 数据平台可能需要回答：

```text
How much volume did Uniswap process today?

Which pools have the most liquidity?

What price did this wallet receive?

How much fee did LPs earn?

Which tokens are gaining trading activity?

How much slippage are users experiencing?
```

这类问题直接对应：

```text
analytics
risk
trading
wallet
research
market data
```

产品。

---

## 二十四、[Career 视角] 为什么这一阶段很关键？

Infrastructure Engineer 可以告诉你：

```text
I can parse logs.
```

Blockchain Data Engineer 还应该能够说：

```text
I know which logs represent swaps,
what the grain of a swap fact is,
how pool state affects price,
and how to build trustworthy DEX metrics.
```

区别在于：

> Protocol semantics turn raw blockchain data into business data.

这正是 Module 13 开始要训练的能力。

---

# 二十五、本课先建立一个完整 DEX Mental Model

先不要碰公式。

把 DEX 想成：

```text
LPs
↓
Provide Assets
↓
Liquidity Pool
↑
Protocol Rules / Pricing Formula
↓
Trader Swap
↓
Pool State Changes
↓
Fees Generated
```

再从数据侧看：

```text
Transaction
↓
Contract Execution
↓
Swap Event / Transfers
↓
Pool State Change
↓
Decoder
↓
DEX Swap Fact
↓
Analytics
```

这两张图分别是：

```text
Business View
```

和：

```text
Data Engineer View
```

---

# 本课核心结论

> DEX means Decentralized Exchange.

> CEX means Centralized Exchange.

> Traditional order books match buyers and sellers through a matching engine.

> On-chain order books can be expensive when every order update requires an on-chain transaction.

> AMM means Automated Market Maker.

> AMMs replace continuous order matching with algorithmic trading against a Liquidity Pool.

> LP means Liquidity Provider; LPs provide assets so traders have liquidity to trade against.

> A Swap is a protocol-level business event and usually changes Pool State.

> Swap Grain and Transfer Grain are not the same.

> For a Blockchain Data Engineer, understanding protocol semantics is necessary to transform raw Logs and Transfers into trustworthy DEX business facts.

> Module 13 marks a shift from “how blockchain data infrastructure works” to “what protocol data means”.

---

# 理解检查

## 问题一

传统 Order Book Exchange：

```text
Buyer
+
Seller
+
Order Book
+
Matching Engine
```

AMM DEX：

```text
Trader
+
Liquidity Pool
+
Pricing Formula
```

请回答：

1. 两种模式最核心的 Market Structure 差异是什么？
2. 为什么 Ethereum 早期直接把传统高频 Order Book 完整搬上链成本很高？
3. AMM 主要解决了哪个问题？

---

## 问题二

假设：

```text
Alice swaps
1 ETH → USDC
```

底层看到：

```text
Transfer A
Alice → Pool

Transfer B
Pool → Alice
```

请回答：

1. 能不能简单认为“两个 Transfer = 一个 Swap”？
2. 为什么？
3. 对 `dex_swaps` Fact Table 来说，合理的 Grain 应该是什么？

---

## 问题三

请从两个视角分别描述一次 Swap：

### [Protocol 视角]

它对 Pool 做了什么？

### [Data Engineer 视角]

你需要从链上数据中还原哪些核心 Business Semantics？