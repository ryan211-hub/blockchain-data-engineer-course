# Module 13 第 6 课｜Uniswap v3：Concentrated Liquidity、Tick 与 Position

## Lesson Contract

所属 Module：Module 13 — DEX（Decentralized Exchange，去中心化交易所）

**本课核心问题：**

> Why did Uniswap v3 replace “one LP share of the whole pool” with price ranges, ticks and individual positions?

学完以后，你应该能够解释：

1. Uniswap v2 与 v3 的 Liquidity Model 为什么不同。
2. Concentrated Liquidity（集中流动性）解决了什么问题。
3. Tick 是什么，为什么 v3 要把 Price Space 离散化。
4. LP Position 为什么不再等于“全池固定比例”。
5. 为什么 v3 的 Position 往往使用 NFT 表示，而不是统一的 Fungible LP Token。
6. 什么叫 Active Liquidity。
7. 为什么某个 Position 在价格区间之外时可能只剩单边资产。
8. v3 的 Pool Identity 为什么除了 Token Pair 还需要 Fee Tier。
9. 从 Data Engineer 视角，Pool、Tick、Position、Swap 应该如何分 Grain。
10. 为什么不能把 v2 的 `reserve0 / reserve1 + LP share` 模型直接套到 v3。

本课不展开：

- Tick Math 的完整数学证明。
- `sqrtPriceX96` 的定点数编码细节。
- Tick Bitmap 的底层存储实现。
- Fee Growth Inside / Outside 的完整计算公式。
- NFT Position Manager 合约源码。
- Uniswap v3 的完整 PnL / APR 计算。

这些细节以后真正做 v3 Indexer 或 LP Analytics 时再深入。

---

# 一、先回顾 Uniswap v2 的 LP 模型

上一课的 v2 Pool 可以理解成：

```text
Pool
├── token0 reserve
├── token1 reserve
└── LP shares
```

LP 提供资金后，拥有：

```text
x% of the entire pool
```

例如：

```text
Pool:
100 ETH
300,000 USDC

Alice:
10% ownership
```

那么 Alice 的 Position 始终代表：

```text
10% of whatever the entire pool currently holds
```

也就是说，v2 的一个核心特点是：

> Liquidity is spread across the entire possible price curve.

无论 ETH 价格是：

```text
1,000 USDC
3,000 USDC
10,000 USDC
100,000 USDC
```

Alice 的流动性理论上都在整个曲线上参与做市。

---

# 二、v2 的问题：Capital Efficiency 不高

假设 ETH 当前市场价格长期在：

```text
2,900 ~ 3,100 USDC
```

但 v2 的 Liquidity 仍然覆盖整个价格空间：

```text
0
↓
1,000
↓
3,000   ← current market area
↓
10,000
↓
100,000
↓
∞
```

问题是：

大量资本实际上被分配到了：

```text
非常远离当前市场价格
```

的区域。

这些 Liquidity 很少被实际交易使用。

所以 v2 的问题不是“没有 Liquidity”，而是：

> A large portion of liquidity may be economically idle at the current market price.

这就是：

```text
Capital Efficiency
```

问题。

---

# 三、Uniswap v3 的核心创新：Concentrated Liquidity

Concentrated Liquidity（集中流动性）允许 LP 指定：

```text
Price Range
```

例如 Alice 认为 ETH 大部分时间会在：

```text
2,500 ~ 3,500 USDC
```

之间交易。

那么她可以只把 Liquidity 放在：

```text
[2500, 3500]
```

这个价格区间。

而 Bob 可以选择：

```text
[1500, 6000]
```

Carol 可以选择：

```text
[2950, 3050]
```

于是，同一个 Pool 里可能存在很多不同的 LP Position。

---

# 四、为什么集中以后 Capital Efficiency 会提高？

假设 Alice 有相同价值的资产。

v2：

```text
Liquidity spread across entire curve
```

v3：

```text
Liquidity concentrated near current price
```

如果市场价格就在 Alice 的 Range 内：

```text
2500 ~ 3500
```

那么她的资金全部集中在最可能发生交易的位置。

于是，同样资本可能提供更大的：

```text
effective liquidity
```

更深的 Liquidity 意味着：

```text
smaller price impact
better execution
more fee-generating activity
```

因此：

> Concentrated liquidity trades capital efficiency for range risk and position complexity.

这里一定要注意后半句。

v3 并不是“v2 的免费升级”。

它提高 Capital Efficiency 的同时，也把 LP Position 变得复杂很多。

---

# 五、用银行外汇做市做类比

假设你负责银行 USD/CNY 做市。

方案 A：

```text
从 1 USD = 1 CNY
一直准备到
1 USD = 20 CNY
```

这相当于把资金分散到极大价格范围。

方案 B：

你认为当前市场主要在：

```text
7.0 ~ 7.4
```

于是把更多资金集中在这里。

这更像 v3。

结果：

```text
same capital
↓
more depth around relevant prices
↓
better execution locally
```

但如果市场突然跑到：

```text
8.0
```

你的：

```text
7.0 ~ 7.4
```

做市区间就不再有效。

这就是 v3 LP 新增的重要风险。

---

# 六、[Protocol 视角] 什么叫 Price Range？

一个 v3 LP Position 至少要表达：

```text
token0
token1

lower price boundary
upper price boundary

liquidity
```

例如：

```text
ETH / USDC

Lower Price = 2,500
Upper Price = 3,500
```

这代表：

> 只有当市场价格位于这个区间附近时，这份 Liquidity 才处于 Active 状态并参与 Swap。

所以 v3 Position 不再只是：

```text
I own 10% of the pool
```

而是：

```text
I provide L units of liquidity
between price A and price B
```

这就是 v2 与 v3 最根本的数据模型差异。

---

# 七、Tick 是什么？

如果允许 LP 任意输入无限精度价格：

```text
2500.123456789...
```

协议管理 Liquidity 会非常复杂。

因此 v3 把 Price Space 划分成离散的价格点：

```text
Tick
```

可以先把 Tick 理解成：

> A discrete index representing a price point on the Uniswap v3 price curve.

也就是说：

```text
tick
↓
maps to
a specific price
```

Tick 本身不是价格金额。

它是一个：

```text
price index
```

---

# 八、Tick 与 Price 的关系

Uniswap v3 使用指数关系：

```text
price ≈ 1.0001 ^ tick
```

这里先只理解概念。

例如：

```text
tick = A
↓
price = P1

tick = A + 1
↓
price ≈ P1 × 1.0001
```

所以 Tick 每移动一个单位，Price 会按一个很小的比例变化。

这使整个连续价格空间被离散成：

```text
... tick -2
tick -1
tick 0
tick 1
tick 2 ...
```

而 LP Range 可以表示为：

```text
lower_tick
upper_tick
```

---

# 九、为什么不用直接存 Lower Price / Upper Price？

从产品界面看，你可能看到：

```text
2500 USDC
3500 USDC
```

但 Protocol 内部更核心的是：

```text
tick_lower
tick_upper
```

原因是：

```text
Tick
```

才是协议执行 Liquidity Accounting 的离散边界。

所以 Data Engineer 应该保留：

```text
tick_lower
tick_upper
```

并把：

```text
price_lower
price_upper
```

作为 Derived / Human-readable 字段。

这和之前保存：

```text
amount_raw
+
amount
```

的原则非常相似。

---

# 十、什么是 Active Liquidity？

假设当前 ETH Price：

```text
3,000 USDC
```

有三个 Position：

```text
Alice:
2500 ~ 3500

Bob:
1500 ~ 2500

Carol:
3500 ~ 5000
```

当前 3,000 位于 Alice 的 Range：

```text
2500 < 3000 < 3500
```

因此 Alice：

```text
Active
```

Bob：

```text
Out of Range
```

Carol：

```text
Out of Range
```

于是当前真正参与 Swap 的 Liquidity 主要来自：

```text
Active Positions
```

这叫：

```text
Active Liquidity
```

---

# 十一、v3 的 Pool Liquidity 不是所有 Position Liquidity 的简单总和

这是非常重要的数据工程点。

假设：

```text
Alice liquidity = 100
Bob liquidity   = 200
Carol liquidity = 300
```

不能直接说：

```text
Current Active Liquidity = 600
```

因为不同 Position 的 Range 不同。

真正影响当前价格交易的是：

```text
current tick 所在区间内 active 的 liquidity
```

因此：

```text
Total Position Liquidity
≠
Current Active Liquidity
```

这是 v3 与 v2 非常不同的地方。

---

# 十二、价格移动时 Active Liquidity 会发生什么？

假设当前：

```text
Price = 3,000
Current Tick = T
```

Alice 的 Position：

```text
2500 ~ 3500
```

当价格上涨并穿过：

```text
3500
```

这个 Upper Boundary 时：

```text
Alice liquidity
↓
becomes inactive
```

同时另一些以 3500 为 Lower Boundary 的 Position 可能：

```text
become active
```

所以价格移动时，v3 的有效 Liquidity 会发生分段变化。

可以理解为：

```text
Price moves
↓
crosses tick
↓
active liquidity changes
↓
swap continues with new liquidity depth
```

---

# 十三、[Protocol 视角] Tick 为什么是 Liquidity Boundary？

LP Position：

```text
tick_lower
tick_upper
```

定义 Liquidity 生效区间。

当价格跨过某个 Initialized Tick 时，Protocol 需要知道：

```text
How much liquidity becomes active?
How much liquidity becomes inactive?
```

所以 Tick 不只是“价格标签”。

它还是：

```text
liquidity transition boundary
```

这就是 Tick 在 v3 数据里非常重要的原因。

---

# 十四、什么是 Initialized Tick？

并不是每一个可能的 Tick 都真的有 LP Position Boundary。

只有当某个 Tick 上有 Liquidity Position 开始或结束时，它才需要被协议记录为：

```text
Initialized Tick
```

例如：

```text
Alice:
[2500, 3500]

Bob:
[3000, 4000]
```

可能对应若干：

```text
lower_tick
upper_tick
```

这些 Tick 成为 Liquidity Boundary。

所以：

```text
all possible ticks
```

与：

```text
initialized ticks
```

不是一回事。

---

# 十五、为什么 v3 Position 不再适合普通 Fungible LP Token？

回到 v2。

如果 Alice 和 Bob 都是：

```text
10% of same pool
```

他们的 LP Share 在经济上是同质的。

因此可以用：

```text
fungible LP token
```

表示。

但 v3 中：

```text
Alice:
2500 ~ 3500

Bob:
2000 ~ 5000
```

即使两个人投入相同价值的 Token：

```text
their positions are not economically identical
```

因为：

```text
range different
liquidity different
fee exposure different
token composition different
```

所以不能简单用同一种 ERC-20 LP Token 表示。

---

# 十六、v3 Position 为什么常用 NFT 表示？

Uniswap v3 常见用户 Position 通过：

```text
NFT
```

表示。

NFT（Non-Fungible Token，非同质化代币）的特点就是：

```text
Position A
≠
Position B
```

每个 Position 可以携带自己的：

```text
token0
token1
fee tier
tick_lower
tick_upper
liquidity
fee accounting state
```

所以：

> v3 Position is position-specific, not a uniform share of the entire pool.

这就是 NFT Position 的业务含义。

---

# 十七、[Data Engineer 视角] Pool 和 Position 的 Grain 完全不同

在 v3 中：

```text
Pool
```

可以理解为：

```text
one token pair
+
one fee tier
```

而：

```text
Position
```

是：

```text
one owner / position identifier
+
one pool
+
one tick range
+
one liquidity amount
```

所以数据建模至少需要分开：

```text
dim_dex_pool_v3
fact_or_snapshot_lp_position_v3
```

不能像 v2 一样只保存：

```text
wallet
pool
lp_share_pct
```

因为：

```text
share_pct
```

已经不能完整表达 Position。

---

# 十八、为什么 Fee Tier 会成为 Pool Identity 的一部分？

在 Uniswap v2 中，同一 Token Pair 通常对应一个主要 Pair Contract。

但 v3 支持不同 Fee Tier。

例如同样：

```text
USDC / WETH
```

可能存在不同 Pool：

```text
USDC / WETH / 0.05%
USDC / WETH / 0.30%
USDC / WETH / 1.00%
```

这些是不同 Pool。

因此不能只用：

```text
token0 + token1
```

识别 v3 Pool。

概念上至少需要：

```text
chain_id
pool_address
token0
token1
fee_tier
```

其中：

```text
chain_id + pool_address
```

仍然可以作为链上 Pool Identity。

但业务维度里：

```text
fee_tier
```

必须保留。

---

# 十九、为什么同一 Token Pair 要有多个 Fee Tier？

因为不同资产的交易特征不同。

例如：

```text
USDC / USDT
```

价格波动通常较小。

而：

```text
ETH / smaller token
```

波动可能明显更大。

LP 愿意承担的 Inventory Risk 不同，因此适合的 Fee Compensation 也可能不同。

所以 Fee Tier 可以理解成：

```text
different market-making market
for the same token pair
```

从 Data Analytics 角度：

```text
USDC/WETH 0.05%
```

和：

```text
USDC/WETH 0.30%
```

不能直接当作同一个 Pool。

---

# 二十、Position 在 Range 内时是什么资产结构？

假设 Alice Position：

```text
ETH / USDC
Range:
2500 ~ 3500
```

当前 Price：

```text
3000
```

Alice 的 Position 通常同时包含：

```text
ETH
+
USDC
```

因为当前价格处于 Range 中间。

随着价格在 Range 内移动：

```text
ETH quantity changes
USDC quantity changes
```

这和 v2 的自动再平衡直觉类似。

---

# 二十一、Position 跑出 Range 会怎样？

这是 v3 最重要的直觉之一。

假设：

```text
Alice Range:
2500 ~ 3500

Current Price:
3000
```

如果 ETH Price 一直上涨，最终超过：

```text
3500
```

那么 Alice Position 会：

```text
Out of Range
```

并趋向某一种单边资产。

概念上可以理解为：

```text
price rises through range
↓
pool gradually sells one asset
↓
position reaches boundary
↓
position becomes one-sided
↓
liquidity stops being active
```

反方向跌破 Lower Boundary 时，则可能转化为另一侧资产。

---

# 二十二、Out of Range 不等于 Position 消失

非常重要。

如果 Position：

```text
out of range
```

它并没有消失。

仍然存在：

```text
position ownership
liquidity configuration
underlying token value
previously earned fees
```

只是：

```text
current active liquidity contribution = 0
```

因此：

```text
Position Exists
≠
Position Is Active
```

数据模型必须能够表达这种状态。

---

# 二十三、Out of Range 时还赚新的 Swap Fee 吗？

通常：

```text
No active liquidity
→ no participation in swaps at current price
→ no new LP fee from those swaps
```

所以 LP 选择很窄的 Range：

```text
narrow range
```

虽然可以提高 Capital Efficiency，但更容易：

```text
go out of range
```

从而停止赚取当前交易 Fee。

因此存在一个核心权衡：

```text
Narrow Range
→ higher capital efficiency
→ higher out-of-range risk

Wide Range
→ lower capital efficiency
→ stays active across more prices
```

---

# 二十四、这和 Impermanent Loss 有什么关系？

上一课我们说：

```text
LP Strategy
vs
HODL Strategy
```

会产生相对收益差异。

v3 中这个问题更加复杂。

因为 LP 不仅承担：

```text
price rebalancing
```

还主动选择：

```text
price range
```

Range 决定：

```text
什么时候做市
什么时候停止做市
什么时候变成单边资产
```

所以 v3 LP Analytics 不能只看：

```text
token price change
```

还必须考虑：

```text
range
active time
fees earned
position composition
```

---

# 二十五、[Protocol 视角] v3 Pool 的核心 State

为了本课的数据理解，可以先记住几个核心字段：

```text
token0
token1
fee

sqrtPriceX96
tick
liquidity
```

其中：

```text
sqrtPriceX96
```

表示当前价格的协议级编码。

```text
tick
```

表示当前价格对应的 Tick 区域。

```text
liquidity
```

在 Pool Current State 中更接近：

```text
current active liquidity
```

而不是：

```text
sum of all LP positions
```

这一点必须特别注意。

---

# 二十六、sqrtPriceX96 是什么？

这里只讲 Data Engineer 必须知道的程度。

v3 没有像我们教学例子一样直接把：

```text
price = 3000
```

作为核心 State 存储。

它使用：

```text
sqrtPriceX96
```

也就是：

```text
square-root price
encoded as fixed-point integer
```

Data Engineer 可以通过公式把它转换为：

```text
token1 / token0 price
```

再根据：

```text
token decimals
```

调整成 Human-readable Price。

本课不要求记公式。

只要求记住：

> `sqrtPriceX96` is protocol state; human-readable price is derived data.

---

# 二十七、为什么 v3 Swap 不能只看 Reserve Ratio？

这是 v2 到 v3 的重要变化。

v2 常用：

```text
reserve1 / reserve0
```

理解 Spot Price。

但 v3 当前价格更直接来自：

```text
sqrtPriceX96
```

并结合：

```text
tick
active liquidity
```

理解当前执行环境。

所以：

```text
v2 mental model:
reserves → price

v3 mental model:
sqrt price + tick + active liquidity → current execution state
```

不能把 v2 的：

```text
reserve0
reserve1
x * y = k across entire pool
```

原封不动套到 v3 Position 模型。

---

# 二十八、Uniswap v3 的 Swap Event 有什么不同？

v3 Pool 的 Swap Event 除了：

```text
sender
recipient
amount0
amount1
```

还包含执行后的重要 State，例如：

```text
sqrtPriceX96
liquidity
tick
```

这对 Data Engineer 非常重要。

因为一条 Swap Event 不只告诉你：

```text
what token amounts changed
```

还可以告诉你执行后：

```text
current price
current active liquidity
current tick
```

因此 v3 Swap Fact 的 State Context 比 v2 更丰富。

---

# 二十九、amount0 / amount1 为什么可能有正负号？

v2 的 Swap Event 使用：

```text
amount0In
amount1In
amount0Out
amount1Out
```

方向非常显式。

v3 常见 Swap Event 则使用：

```text
amount0
amount1
```

并通过符号表示 Pool Balance Change 的方向。

可以先记：

```text
one side positive
one side negative
```

典型 Swap 中，一种 Token 流入 Pool，另一种 Token 流出 Pool。

因此 Data Engineer 需要通过：

```text
sign
+
token0/token1 mapping
```

推导：

```text
token_in
token_out
amount_in
amount_out
```

这和 v2 Decoder 的字段语义不同。

---

# 三十、[Data Engineer 视角] v2 与 v3 Swap Normalization

虽然 Protocol Event 不同，但 Analytics 层希望统一成：

```text
pool_address
token_in
token_out
amount_in
amount_out
execution_price
trader / sender context
```

所以：

```text
v2 Swap Event
        \
         → Normalized DEX Swap Fact
        /
v3 Swap Event
```

这就是你之前学过的：

```text
Protocol-specific Raw / Normalized
```

分层思想。

Raw 层必须保留：

```text
v2 original event fields
v3 original event fields
```

Normalized 层再统一业务语义。

---

# 三十一、v3 Position 的核心字段

从数据模型角度，一个 Position 至少要考虑：

```text
position_id
owner

pool_address

token0
token1
fee_tier

tick_lower
tick_upper

liquidity
```

另外真实 Analytics 还会涉及：

```text
deposited amount0
deposited amount1
withdrawn amount0
withdrawn amount1

fees collected
current amount0
current amount1
```

但本课不深入完整 PnL。

最重要的是：

```text
Position ≠ Pool Share
```

---

# 三十二、Position ID 与 Owner 为什么也要分开？

因为 Position 可以：

```text
transfer
```

如果 Position 通过 NFT 表示，那么：

```text
position_id
```

标识 Position 本身。

而：

```text
owner
```

表示当前拥有这个 Position 的 Wallet。

因此：

```text
Position Identity
≠
Current Owner Identity
```

这和：

```text
NFT token_id
≠
wallet address
```

是同一种数据建模问题。

如果只用：

```text
wallet + pool
```

做 Position Key，就可能错误。

---

# 三十三、同一个 Wallet 在同一个 Pool 可以有多个 Position 吗？

可以。

Alice 可以同时拥有：

```text
Position A
USDC/WETH 0.3%
2500 ~ 3500

Position B
USDC/WETH 0.3%
2900 ~ 3100
```

两者：

```text
same wallet
same pool
different range
```

所以：

```text
wallet + pool
```

不能唯一标识 v3 LP Position。

这正是 Grain 设计时非常关键的一点。

---

# 三十四、[Data Modeling 视角] v3 至少有哪些核心对象？

可以先拆成：

```text
Pool
Tick
Position
Swap
```

分别代表：

```text
Pool
→ trading market / current state context

Tick
→ discrete price & liquidity boundary

Position
→ one LP's range-specific liquidity object

Swap
→ executed trade event
```

这四种 Grain 不应混成一张大表。

---

# 三十五、一个简单的 v3 数据模型

Pool Dimension：

```text
dim_dex_pool_v3

grain:
one pool per chain
```

Position：

```text
fact_or_snapshot_lp_position_v3

grain:
one position_id at one state/time
```

Tick State：

```text
fact_dex_tick_state

grain:
one pool + one tick at one state/time
```

Swap Fact：

```text
fact_dex_swap_v3

grain:
one successful Pool Swap Event
```

这些表之后可以被统一到更上层：

```text
dex_swaps_normalized
```

---

# 三十六、为什么 Tick State 不应该塞进 Pool 表？

因为一个 Pool 可能存在很多 Initialized Tick。

关系是：

```text
One Pool
→ Many Ticks
```

而一个 Tick 又有自己的 Liquidity Accounting State。

如果把所有 Tick 展平进 Pool 表：

```text
one pool
×
many ticks
```

会造成：

```text
Pool row duplication
```

并破坏 Pool Grain。

这和数据库建模中的一对多关系完全一样。

---

# 三十七、为什么 Position 也不应该直接塞进 Pool 表？

同理：

```text
One Pool
→ Many Positions
```

并且：

```text
One Wallet
→ Many Positions
```

如果把：

```text
pool
tick
position
swap
```

全部揉成一张表，会产生大量重复和难以定义的 Grain。

因此 Blockchain Protocol Analytics 很重要的一条原则就是：

> Model protocol objects according to their natural grain.

---

# 三十八、v2 与 v3 的核心数据模型对比

v2：

```text
Pool
├── reserve0
├── reserve1
└── fungible LP shares
```

v3：

```text
Pool
├── current sqrt price
├── current tick
├── active liquidity
│
├── Tick A
├── Tick B
├── Tick C
│
├── Position 1 [tick A, tick C]
├── Position 2 [tick B, tick D]
└── Position 3 [tick E, tick F]
```

所以 v3 增加的不是几个字段而已。

它引入了新的：

```text
business objects
```

这才是 Data Engineer 真正需要理解的地方。

---

# 三十九、从业务问题反推数据需求

如果产品问：

> 当前 USDC/WETH Pool 的价格是多少？

需要：

```text
Pool Current State
sqrtPriceX96
decimals
```

如果问：

> Alice 的 Position 是否处于 Active 状态？

需要：

```text
position.tick_lower
position.tick_upper
pool.current_tick
```

如果问：

> 当前价格附近有多少 Active Liquidity？

需要：

```text
current tick
tick liquidity transitions
pool active liquidity
```

如果问：

> Alice 在这个 Position 赚了多少 Fee？

还需要：

```text
Position Fee Accounting
Collect Events
Fee Growth State
```

最后一个本课不展开。

---

# 四十、本课真正要建立的 Mental Model

v2 可以先想成：

```text
one pool
one shared curve
LP owns proportional share of whole curve
```

v3 应该想成：

```text
one pool
many LP-defined price ranges
ranges overlap
↓
current price selects active ranges
↓
active liquidity changes as price crosses ticks
```

因此：

```text
Price
+
Tick
+
Range
+
Position
+
Active Liquidity
```

成为 v3 的核心语义组合。

---

# 本课核心结论

> Uniswap v3 introduces Concentrated Liquidity so LP capital can be allocated to selected price ranges instead of the entire price curve.

> A Tick is a discrete price index and also serves as a boundary where liquidity can become active or inactive.

> Active Liquidity is the liquidity currently available around the current price; it is not simply the sum of all Position liquidity.

> A v3 LP Position is range-specific and position-specific, so it cannot be represented adequately as a simple proportional share of the whole pool.

> v3 Positions are commonly represented as NFTs because different ranges make positions non-fungible.

> A Position can exist while being Out of Range; in that case it is not currently providing active liquidity or earning new swap fees at the current price.

> The same wallet can own multiple Positions in the same Pool, so `wallet + pool` is not a sufficient Position identity.

> v3 Pool Identity must preserve Fee Tier semantics because the same Token Pair can have multiple pools with different fee configurations.

> `sqrtPriceX96` is protocol-level price state; human-readable price is derived data.

> Pool, Tick, Position and Swap are different protocol objects with different grains and should be modeled separately.

---

# 理解检查

## 问题一：为什么要有 Concentrated Liquidity？

假设 ETH 当前价格：

```text
3,000 USDC
```

Alice：

```text
Range = 2,500 ~ 3,500
```

Bob：

```text
Range = 1,000 ~ 10,000
```

两人投入相同价值资产。

请回答：

1. 如果价格长期在 2,900 ~ 3,100 附近，谁的 Capital Efficiency 通常更高？为什么？
2. Alice 的 Range 更窄，她承担了什么额外风险？
3. 如果 ETH 涨到 4,000，Alice Position 是否消失？它当前还是 Active Liquidity 吗？

## 问题二：Tick 与 Active Liquidity

假设有：

```text
Position A:
tick range = [100, 200]

Position B:
tick range = [150, 300]

Current Tick = 175
```

请回答：

1. 当前哪些 Position 是 Active？
2. 如果 Current Tick 从 175 上升并跨过 200，会发生什么 Liquidity 变化？
3. 为什么不能简单把 A + B 的全部 Liquidity 永远当作 Pool Current Active Liquidity？

## 问题三：Data Modeling

现在你要设计 v3 数据模型。

请回答：

1. 为什么不能继续使用 v2 的 `wallet + pool + share_pct` 来完整表示 LP Position？
2. 同一个 Wallet 能不能在同一个 Pool 有多个 Position？
3. Position 应该至少保存哪些能够表达 Range 的核心字段？
4. 为什么 `Pool`、`Tick`、`Position`、`Swap` 应设计成不同 Grain？
5. `sqrtPriceX96` 属于 Protocol State 还是 Human-readable Price？最终给 Analytics 使用的 Price 属于原始字段还是 Derived Field？

## 用户回答（理解检查｜问题一、问题二）

问题一：

1. Alice 的 capital efficiency 会更高，因为 Alice 的 liquidity range 更集中。
2. Alice 她能接受的 ETH 涨跌的波动更小。如果 ETH 的涨跌超出了这个 range，那么 Alice 的 liquidity 就不会有任何的收益
3. Alice's position 并不会消失，但是它会变成 inactive 状态

问题二：

1. 当前 position A 和 B 都是处于 active 状态
2. Position A 会变成 inactive 状态，Position B 的状态不会变，还是处于 active 状态
3. 因为 position A 和 B 的 tick range 并不完全相同，所以它们不能当做同样的 liquidity来处理

## 老师判断 / 校准（问题一、问题二）

### 问题一

整体通过，第 2 点做一个精度校准。

1. Alice 的 Capital Efficiency 更高：正确。因为在相同资本下，她把 Liquidity 集中在更窄、且更接近当前市场价格的区间，因此当前价格附近能够提供更大的 Effective Liquidity。

2. 你指出 Narrow Range 带来更高 Out-of-Range Risk：方向正确。

但“超出 Range 后就不会有任何收益”需要改得更准确：

```text
Out of Range
→ no current active liquidity
→ usually no new swap fee from trades at current price
```

Position 本身仍存在，而且仍然可能包含：

```text
underlying token value
previously accrued fees
position ownership
```

所以更准确的说法是：

> Out of Range 后，Position 通常停止参与当前价格下的 Swap，因此不再赚取新的 Swap Fee，而不是“完全没有任何收益”。

3. Position 不会消失，只是变成 Inactive / Out of Range：正确。

可以固定记成：

```text
Position Exists
≠
Position Is Active
```

### 问题二

三点都正确。

1. Current Tick = 175 时：

```text
Position A [100, 200]
Position B [150, 300]
```

175 同时位于两个 Range 内，所以 A、B 都是 Active。

2. 当 Current Tick 向上跨过 200：

```text
Position A
→ becomes inactive

Position B
→ remains active
```

判断正确。

3. 你的核心判断正确：不同 Position 的 Tick Range 不同，因此它们对 Current Active Liquidity 的贡献取决于 Current Tick 是否落在各自 Range 内。

更精确地说：

```text
Total Position Liquidity
≠
Current Active Liquidity
```

Current Active Liquidity 只统计当前 Tick 所在价格区间内处于 Active 状态的 Liquidity。

本轮判定：

- 问题一：通过，完成 1 个精度校准。
- 问题二：通过。


## 用户回答（理解检查｜问题三）

1. V3 的 LP position 它并不像 V2 的 share percent 一样，LP position 每个都是不一样的，它需要通过 position range 来显示地标识具体 position 的范围。
2. 可以
3. token0 / token1 / fee tier / tick_lower / tick_upper / liquidity / fee accounting state
4. 因为他们四个都有不同的业务含义，所以应该设计成不同的 group，而不应该混用
5. `sqrtPriceX96` 是 Human-readable Price，Price 属于 Derived Field

## 老师判断 / 校准（问题三）

问题三前 4 点整体正确，第 5 点需要校准。

1. 正确。v3 Position 不再是 v2 那种“整个 Pool 的统一 share_pct”。不同 Position 可以有不同的 Range、Liquidity 和 Fee Exposure，因此必须使用 Position-specific 字段表达。

2. 正确。同一个 Wallet 可以在同一个 Pool 中拥有多个 Position，因此 `wallet + pool` 不足以唯一标识一个 v3 Position。

3. 你列出的字段是合理的。对于表达 Range 和 Position Identity，本课最关键的是：

```text
position_id
owner
pool_address
fee_tier
tick_lower
tick_upper
liquidity
```

`token0 / token1` 可以通过 Pool 维度关联获得，也可以在宽表或 Analytics 模型中冗余保存；`fee accounting state` 则属于更完整的 Position Analytics 所需字段。

4. 核心判断正确，但术语应更精确为不同的 **Grain / Protocol Object**，而不是简单说不同 group：

```text
Pool
→ one trading market / pool state

Tick
→ one discrete price & liquidity boundary

Position
→ one LP range-specific liquidity object

Swap
→ one executed trade event
```

自然 Grain 不同，所以应该分开建模。

5. 这里需要校准：

```text
sqrtPriceX96
= Protocol State / encoded price representation

Human-readable Price
= Derived Field
```

`sqrtPriceX96` 是协议内部保存的 square-root price fixed-point encoding，不是最终面向用户或 Analytics 的价格。

Data Engineer 需要：

```text
sqrtPriceX96
+
token0 decimals
+
token1 decimals
↓
derive
↓
human-readable token price
```

因此你后半句“Price 属于 Derived Field”是正确的，但前半句应改为：

> `sqrtPriceX96` belongs to Protocol State; human-readable Price is derived from it.

本题当前判定：

- 第 1～4 点：通过。
- 第 5 点：需要完成上述概念校准。

校准确认题：

`sqrtPriceX96` 和 `3,000 USDC / ETH`，哪一个属于 Protocol State，哪一个属于 Derived / Human-readable Price？
