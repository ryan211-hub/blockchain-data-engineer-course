# Module 13 第 4 课｜Liquidity Provider：Liquidity、Fee 与 Impermanent Loss

## Lesson Contract

所属 Module：Module 13 — DEX（Decentralized Exchange，去中心化交易所）

本课核心问题：

> Why would someone put their own assets into an AMM pool, and what exactly do they earn and risk?

学完以后，你应该能够解释：

- LP（Liquidity Provider，流动性提供者）到底提供了什么。
- 为什么 AMM 没有 LP 就无法正常交易。
- Liquidity、Reserve、LP Position 三者有什么区别。
- Trading Fee 为什么会成为 LP 收益来源。
- Fee 收益为什么不是“无风险利息”。
- Impermanent Loss（无常损失）到底是什么。
- 为什么 Token Price 变化越大，LP 越可能产生 Impermanent Loss。
- 为什么 Impermanent Loss 不能只理解成“账面亏损”。
- 从 Data Engineer 视角，如何区分 Pool-level 与 Position-level 数据。

本课暂不展开：

- Uniswap v3 Concentrated Liquidity。
- Tick、Range Position。
- LP APR / APY 的精细年化计算。
- MEV 对 LP 的高级影响。
- Stable-swap AMM 的特殊机制。
- LP Hedge / Delta-neutral Strategy。

本课继续以经典 Constant Product AMM 为主。

---

# 一、先回答一个最基本的问题：Pool 里的钱从哪里来？

上一课我们一直使用：

```text
ETH / USDC Pool

100 ETH
300,000 USDC
```

但这些资产不会凭空出现。

它们来自：

```text
Liquidity Provider
```

也就是 LP。

LP 把自己的资产存入 Pool，例如：

```text
10 ETH
30,000 USDC
```

如果当前 Pool Price 是：

```text
1 ETH = 3,000 USDC
```

那么这两种资产的价值大致相等：

```text
10 ETH
≈ 30,000 USDC
```

LP 的作用就是：

> 把可以被 Trader 交易的资产库存放进 AMM Pool。

---

# 二、为什么 AMM 必须有 LP？

AMM 的交易对象不是另一个具体 Trader，而是：

```text
Liquidity Pool
```

Trader：

```text
sell ETH
buy USDC
```

意味着：

```text
ETH enters pool
USDC leaves pool
```

如果 Pool 里根本没有 USDC：

```text
USDC reserve = 0
```

那么 Trader 就没有 USDC 可以拿走。

因此：

```text
No Liquidity
→ No meaningful swap capacity
```

所以 LP 实际提供的是：

```text
Market Inventory
```

也可以理解为：

```text
Trading Capacity
```

---

# 三、[Protocol 视角] Liquidity 是 AMM 可以做市的基础

在传统 Order Book Market 中，市场深度来自：

```text
Buy Orders
Sell Orders
Market Makers
```

而经典 AMM 中，市场深度来自：

```text
Pool Reserves
```

所以：

```text
LP deposits assets
↓
Pool reserves increase
↓
Liquidity becomes deeper
↓
Trades cause smaller relative reserve changes
↓
Execution quality improves
```

这和上一课已经学到的：

```text
Trade Size Relative to Liquidity
```

直接连接起来。

---

# 四、Liquidity、Reserve、LP Position 不要混为一谈

这三个概念很容易混淆。

## 1. Reserve

Reserve 是：

> 当前 Pool 合约中持有的 Token 数量。

例如：

```text
ETH reserve  = 100
USDC reserve = 300,000
```

这是 Pool-level state。

---

## 2. Liquidity

Liquidity 是一个更宽泛的市场概念：

> Pool 可以支持多少交易，以及交易对价格造成多大影响。

Reserve 是 Liquidity 的基础，但：

```text
Liquidity ≠ 单纯某一个 Reserve 数字
```

因为还要考虑：

```text
token value
pool composition
trade size
price range
protocol design
```

在本课的 v2-style 简化模型中，可以先把较大的 Reserve 理解为更深的 Liquidity。

---

## 3. LP Position

LP Position 是：

> 某一个 Liquidity Provider 在 Pool 中拥有的权益份额。

例如整个 Pool：

```text
100 ETH
300,000 USDC
```

Alice 提供了其中 10%。

那么她大致拥有：

```text
10% of pool ownership
```

她并不只是“存了 10 ETH”。

她拥有的是：

```text
share of the pool
```

---

# 五、LP 存入资产后，拿到的是什么？

在 Uniswap v2 这类经典 AMM 中，LP 存入资产以后，会得到代表 Pool 份额的：

```text
LP Token
```

例如：

```text
LP deposits:
10 ETH
30,000 USDC

↓
receives LP tokens
```

LP Token 代表：

```text
ownership share
```

当 LP 想退出时：

```text
burn / redeem LP tokens
↓
withdraw corresponding share of pool reserves
```

所以 LP Token 不是“利息”。

它更像：

```text
Pool Share Certificate
```

---

# 六、用基金份额做类比

可以把经典 LP Position 暂时类比成基金份额。

例如某基金资产：

```text
100 ETH
300,000 USDC
```

你拥有基金的：

```text
10%
```

那么你不是拥有固定：

```text
10 ETH
30,000 USDC
```

永远不变。

你拥有的是：

```text
10% of whatever assets the pool currently holds
```

这一点非常关键。

因为 Trader 不断交易后：

```text
Pool composition changes
```

于是你的 Position 里实际对应的 ETH 和 USDC 数量也会变化。

---

# 七、为什么 LP 愿意提供资产？

如果 LP 只是把资产放进去，又承担价格波动和资产结构变化，那为什么要做？

核心激励是：

```text
Trading Fee
```

每当 Trader 进行 Swap 时，Protocol 通常收取一定比例的交易费。

例如简化地假设：

```text
Trading Fee = 0.3%
```

Trader 输入：

```text
1,000 USDC
```

那么大约：

```text
3 USDC
```

会成为交易费用的一部分。

在经典 AMM 设计中，这些 Fee 通常会惠及 LP。

---

# 八、Fee 是怎么进入 LP 收益的？

这里要特别注意。

很多初学者会想象：

```text
Trader pays fee
↓
Protocol sends fee directly to LP wallet
```

实际经典 AMM 里通常不是这样。

更接近的是：

```text
Trader pays fee
↓
fee remains economically inside the pool
↓
pool value increases relative to no-fee case
↓
LP's share becomes more valuable
```

也就是说：

> LP 收益很多时候体现在 Pool Reserve / Pool Value 的增长中，而不是每笔交易都单独给 LP 转账。

---

# 九、[Protocol 视角] 为什么有 Fee 时 k 会增长？

上一课为了简化，我们说：

```text
x * y = k
```

交易前后：

```text
k stays constant
```

但真实 Uniswap v2 存在交易费。

假设 Trader 输入 1 ETH。

其中只有一部分有效参与定价，例如：

```text
effective_input < actual_input
```

因为一部分作为 fee 留在 Pool。

于是：

```text
actual reserves after trade
```

相对于无 fee 模型会略多一些。

最终：

```text
x' * y' > x * y
```

也就是：

```text
k increases
```

这不是 invariant 失效。

而是：

> Fee-adjusted swap logic preserves the pricing constraint while fees accumulate inside the pool.

---

# 十、LP 收益的第一部分：Trading Fee

因此 LP 收益的最直接来源是：

```text
Trading Volume
×
Fee Rate
×
LP Ownership Share
```

概念上可以写成：

```text
LP Fee Revenue
≈ Pool Trading Fees
× LP Share
```

例如：

```text
Daily Volume = 10,000,000 USDC

Fee Rate = 0.3%
```

那么：

```text
Total Trading Fee
≈ 30,000 USDC
```

如果 Alice 拥有：

```text
1% of pool
```

她经济上大约对应：

```text
300 USDC
```

的 Fee 份额。

实际协议计算还可能更复杂，但这个 Mental Model 是对的。

---

# 十一、是不是 Volume 越大，LP 一定赚得越多？

只看 Fee：

```text
higher volume
→ more fee revenue
```

但 LP 的最终收益不能只看 Fee。

因为 LP 同时暴露在：

```text
Token Price Movement
```

以及：

```text
Pool Rebalancing
```

之下。

这就引出：

```text
Impermanent Loss
```

---

# 十二、Impermanent Loss 到底是什么？

Impermanent Loss，简称：

```text
IL
```

全称：

```text
Impermanent Loss
```

中文通常叫：

```text
无常损失
```

但这个中文名称容易让人误解。

更准确的核心定义是：

> LP 持有 Pool Position 的价值，相对于“如果当初什么都不做、只是单独持有这些 Token”的价值差异。

也就是说，它比较的是：

```text
LP Strategy
vs
HODL Strategy
```

---

# 十三、Impermanent Loss 不是“资产价格下跌”

这是第一处关键区分。

假设 ETH 从：

```text
3,000 USDC
```

跌到：

```text
2,000 USDC
```

即使你根本没有做 LP，只是持有 ETH：

```text
portfolio value
```

也会下跌。

这部分叫：

```text
market loss
```

不是 Impermanent Loss。

IL 比较的是：

```text
If I had simply held the original assets
vs
If I had provided liquidity
```

两者谁的最终价值更高。

---

# 十四、为什么 LP 会和单纯持币产生差异？

因为 AMM 会自动改变你的资产组合。

假设初始 Pool：

```text
ETH / USDC

1 ETH
3,000 USDC
```

当前价格：

```text
1 ETH = 3,000 USDC
```

你提供：

```text
1 ETH
3,000 USDC
```

假设你拥有整个 Pool，便于理解。

如果后来外部市场 ETH 涨到：

```text
6,000 USDC
```

套利者会发现：

```text
AMM ETH cheaper than external market
```

于是他们会：

```text
buy ETH from pool
pay USDC into pool
```

结果：

```text
ETH reserve ↓
USDC reserve ↑
```

直到 AMM Price 接近外部价格。

---

# 十五、[Protocol 视角] LP 被 AMM 自动“再平衡”

所以 ETH 上涨过程中，LP Position 会发生：

```text
ETH quantity ↓
USDC quantity ↑
```

也就是说：

> 当上涨资产变得更值钱时，Pool 会逐步把上涨资产卖给套利者。

反过来，如果 ETH 下跌：

```text
ETH becomes cheaper
```

套利者会：

```text
sell ETH into pool
take USDC out
```

于是：

```text
ETH reserve ↑
USDC reserve ↓
```

所以 LP 的资产组合会被 AMM 自动再平衡。

---

# 十六、这正是 Impermanent Loss 的来源

假设 ETH 大涨。

如果你只是 HODL：

```text
1 ETH
+
3,000 USDC
```

那么你完整保留那 1 ETH 的上涨收益。

但如果你做 LP：

```text
ETH price rises
↓
arbitrage buys ETH from pool
↓
your LP position holds less ETH
↓
you participate less fully in ETH upside
```

所以：

```text
LP Position Value
<
HODL Value
```

两者之间的差额就是：

```text
Impermanent Loss
```

---

# 十七、为什么叫 Impermanent？

因为如果两种 Token 的相对价格后来又回到最初水平：

```text
price ratio returns
```

那么这部分相对损失可能缩小甚至消失。

因此历史上称：

```text
Impermanent
```

但这个词很容易误导。

因为如果 LP：

```text
withdraws liquidity
```

并在价格偏离时退出，那么这个差异就被实现。

所以：

```text
impermanent
≠ not real
```

更准确的是：

```text
relative loss depends on price path and exit point
```

---

# 十八、Impermanent Loss 不是“只要不退出就没有损失”

这是一个常见误解。

有些人会说：

> 不 withdraw 就没有 IL。

这不准确。

从资产估值视角看，只要：

```text
LP Position Value
<
HODL Benchmark Value
```

那么 IL 已经作为：

```text
mark-to-market relative underperformance
```

存在。

是否退出，只决定它是否被：

```text
realized
```

不是决定它是否存在。

---

# 十九、一个简单数学例子

假设初始：

```text
1 ETH
3,000 USDC
```

所以：

```text
k = 3,000
```

ETH 市场价格从：

```text
3,000
```

上涨到：

```text
6,000 USDC
```

新的 Pool Price 需要满足：

```text
y / x = 6,000
```

同时：

```text
x * y = 3,000
```

联立：

```text
y = 6,000x
```

所以：

```text
x * 6,000x = 3,000

6,000x² = 3,000

x² = 0.5

x ≈ 0.7071 ETH
```

于是：

```text
y ≈ 4,242.64 USDC
```

Pool 变成：

```text
0.7071 ETH
4,242.64 USDC
```

---

# 二十、比较 HODL 与 LP

## 如果只是 HODL

你仍然有：

```text
1 ETH
3,000 USDC
```

当 ETH = 6,000 USDC 时：

```text
HODL Value
= 1 × 6,000 + 3,000
= 9,000 USDC
```

---

## 如果做 LP

Pool 当前资产：

```text
0.7071 ETH
4,242.64 USDC
```

按 ETH = 6,000 计算：

```text
LP Value
≈ 0.7071 × 6,000
+ 4,242.64

≈ 8,485.24 USDC
```

所以：

```text
IL
≈ 8,485.24 - 9,000
≈ -514.76 USDC
```

比例大约：

```text
-5.72%
```

这就是经典 Constant Product AMM 中价格翻倍时大约 5.72% 的 Impermanent Loss。

---

# 二十一、这里 LP 其实还是赚钱了吗？

注意：

初始资产价值：

```text
1 ETH = 3,000
+
3,000 USDC

Total = 6,000 USDC
```

后来 LP Value：

```text
≈ 8,485 USDC
```

所以绝对金额上：

```text
6,000 → 8,485
```

LP 是赚钱的。

但 HODL：

```text
6,000 → 9,000
```

赚得更多。

所以 IL 描述的不是：

```text
absolute loss
```

而是：

```text
relative underperformance vs HODL
```

这是必须锁死的概念。

---

# 二十二、Fee 和 Impermanent Loss 是在“对抗”

现在可以理解 LP 的完整收益逻辑：

```text
LP Return
≈ Asset Price Movement
+ Trading Fee Revenue
- Impermanent Loss Effect
```

这是概念模型，不是严格会计公式。

最关键的是：

```text
Fee Revenue
```

可能抵消：

```text
Impermanent Loss
```

如果 Fee 足够高：

```text
Fee Revenue > IL
```

LP Strategy 可能优于 HODL。

如果 Fee 很低而价格变化很大：

```text
IL > Fee Revenue
```

LP 可能跑输 HODL。

---

# 二十三、这就是 LP 的商业逻辑

LP 本质上是在做一件事：

> Provide liquidity in exchange for trading fees, while accepting inventory and price-rebalancing risk.

也就是说：

```text
LP provides capital
↓
Trader gets liquidity
↓
Protocol earns / distributes fees
↓
LP receives fee compensation
↓
LP bears price-composition risk
```

这与银行提供流动性、做市商提供双边报价，在经济逻辑上有一定相似性。

---

# 二十四、用银行资金池做有限类比

假设银行有一个外汇资金池：

```text
USD
SGD
```

客户持续兑换。

银行因为：

```text
providing liquidity
```

获得：

```text
spread / fee
```

但同时银行承担：

```text
inventory risk
FX risk
```

AMM LP 类似：

```text
provide inventory
earn fee
bear inventory rebalancing risk
```

区别是：

```text
AMM
```

通过 Smart Contract + Formula 自动执行。

---

# 二十五、[Data Engineer 视角] Pool-level 和 Position-level 必须分开

这是本课最重要的数据工程结论之一。

Pool-level 数据：

```text
pool_address
token0
token1
reserve0
reserve1
total_liquidity
total_supply_lp_token
volume
fee
```

它描述整个 Pool。

Position-level 数据：

```text
wallet_address
pool_address
lp_token_balance
ownership_share
amount0_equivalent
amount1_equivalent
deposit_value
current_value
fee_share
impermanent_loss
```

它描述某个 LP 的权益。

这两个 Grain 完全不同。

---

# 二十六、不要把 LP Deposit 当成普通 Transfer

例如 Alice：

```text
send 10 ETH
send 30,000 USDC
to Pool
```

如果只看 ERC-20 Transfer，你只能看到：

```text
token moved
```

但 Protocol Semantics 可能是：

```text
Add Liquidity
```

并伴随：

```text
Mint LP Token
```

所以：

```text
Transfer
≠ Liquidity Provision
```

这和之前：

```text
Transfer
≠ Swap
```

是同一个数据工程原则。

必须结合：

```text
Protocol Event
Transaction Context
Pool Contract
LP Token Mint/Burn
```

才能还原真实业务语义。

---

# 二十七、典型 LP 生命周期

一个完整 LP Position 生命周期可以抽象为：

```text
Add Liquidity
↓
Receive LP Token
↓
Pool trades occur
↓
Fees accumulate
↓
Pool composition changes
↓
LP Position value changes
↓
Remove Liquidity
↓
Burn LP Token
↓
Receive underlying assets
```

这条链路以后会直接影响：

```text
Fact Table Design
Position Snapshot
PnL Analytics
```

---

# 二十八、Fee Data 从哪里来？

从 Data Engineer 视角，Fee 有几种可能来源：

```text
Swap Event
Protocol Fee Formula
amount_in / amount_out
Pool State Delta
Protocol-specific fields
```

但不同协议：

```text
fee rate
fee recipient
LP fee
protocol fee
```

可能不同。

因此不能简单假设：

```text
all fee = LP revenue
```

有的协议会拆：

```text
LP Fee
Protocol Fee
Treasury Fee
```

所以 Fee 也是典型的：

```text
protocol-specific semantic field
```

---

# 二十九、为什么 LP Fee 不能只靠 Volume × 0.3% 永远算？

因为真实协议中可能存在：

```text
different fee tiers
dynamic fee
protocol fee switch
multi-hop swap
token transfer tax
rebasing token
```

所以：

```text
volume × fixed fee rate
```

只能作为特定协议、特定版本下的推导方式。

从 Data Engineering 角度：

> Always bind fee calculation to protocol version and fee configuration.

---

# 三十、Impermanent Loss 数据更难

相比 Fee，IL 更难直接计算。

因为你需要知道：

```text
initial deposited assets
deposit-time token prices
current / exit token prices
current position composition
benchmark HODL value
LP position value
fees earned
```

而且还要明确：

```text
gross IL
net LP return
```

这两个不能混在一起。

---

# 三十一、Gross IL 与 Net Return

例如：

```text
Impermanent Loss
= -5.72%

Fee Revenue
= +8%
```

那么：

```text
Net LP Performance
```

可能仍然为正。

所以数据产品里最好区分：

```text
impermanent_loss_pct
fee_return_pct
net_lp_return_pct
```

否则用户会误以为：

```text
IL = final investment loss
```

这是错误的。

---

# 三十二、[Data Modeling 视角] 一个 LP Analytics 模型至少要考虑什么？

可以先建立三个核心对象：

```text
Pool
Liquidity Event
LP Position
```

其中：

```text
Pool
→ pool-level state

Liquidity Event
→ add / remove liquidity event

LP Position
→ wallet-level ownership state
```

再加：

```text
Swap
```

用于计算：

```text
volume
fee generation
pool state change
```

所以数据链路可以是：

```text
Swap Events
+
Liquidity Add / Remove Events
+
LP Token Mint / Burn
+
Pool State
+
Token Price
↓
LP Position Analytics
```

---

# 三十三、为什么这对后面的 Uniswap v2 / v3 很重要？

在 Uniswap v2 中：

```text
LP Position
≈ proportional share of whole pool
```

比较容易。

但到了 Uniswap v3：

```text
LP Position
```

不再只是一个简单全池比例。

因为 Liquidity 只在某个：

```text
Price Range
```

内有效。

所以会出现：

```text
Tick
Range
Concentrated Liquidity
NFT Position
```

这就是为什么我们必须先把 v2-style LP 概念学清楚。

---

# 三十四、[Protocol 视角] LP 的本质不是“存钱赚利息”

这是本课最后一个必须校准的直觉。

LP 不是：

```text
deposit money
↓
protocol pays interest
```

更准确是：

```text
provide market-making inventory
↓
allow traders to swap
↓
earn trading fees
↓
accept inventory rebalancing risk
```

所以：

> LP is closer to passive market making than to a bank deposit.

---

# 本课核心结论

> LP（Liquidity Provider）provides the asset inventory that allows an AMM pool to execute swaps.

> Reserve is pool-level token state; Liquidity is the pool's effective trading depth; LP Position is an individual provider's ownership share.

> In classic AMMs such as Uniswap v2, LPs receive LP Tokens representing proportional ownership of the pool.

> Trading Fees are the primary economic incentive for LPs.

> Fees often remain economically inside the pool, increasing the value attributable to LP shares.

> Impermanent Loss is not simply a token price loss; it is the relative underperformance of an LP position compared with simply holding the original assets.

> Price divergence causes arbitrage to rebalance the pool, changing the asset composition of LP positions.

> Impermanent Loss can exist even before liquidity is withdrawn; withdrawal mainly determines whether the relative loss is realized.

> LP performance should distinguish Fee Revenue, Impermanent Loss, and Net LP Return.

> From a Blockchain Data Engineer perspective, Pool-level state and Position-level state have different grains and must be modeled separately.

> Transfer evidence alone cannot prove Add Liquidity or Remove Liquidity; Protocol Events and transaction context are required.

---

# 理解检查

## 问题一

假设一个 ETH / USDC Pool 当前有：

```text
100 ETH
300,000 USDC
```

Alice 提供：

```text
10 ETH
30,000 USDC
```

请回答：

1. Alice 提供 Liquidity 后，她拥有的是固定的 `10 ETH + 30,000 USDC`，还是 Pool 的某个 Ownership Share？
2. 为什么 Trader 的 Swap 会改变 Alice Position 中实际对应的 ETH / USDC 数量？
3. LP 和普通“银行存款赚利息”的核心区别是什么？

---

## 问题二

假设 Alice 初始持有：

```text
1 ETH
3,000 USDC
```

她有两个选择：

```text
A. 直接 HODL
B. 全部放进 ETH / USDC Pool
```

后来 ETH 从：

```text
3,000
```

涨到：

```text
6,000 USDC
```

请回答：

1. 为什么 LP Position 中 ETH 数量会减少？
2. Impermanent Loss 比较的是哪两个策略？
3. 如果 LP Position 最后价值从 6,000 涨到 8,485 USDC，为什么仍然可能存在 Impermanent Loss？

---

## 问题三

从 Blockchain Data Engineer 视角：

```text
Transfer:
Alice → Pool : 10 ETH
Alice → Pool : 30,000 USDC
```

请回答：

1. 只看这两条 Transfer，能不能确认 Alice 在 Add Liquidity？
2. 还需要什么类型的证据？
3. 为什么 `Pool` 与 `LP Position` 应该设计成不同 Grain 的数据对象？