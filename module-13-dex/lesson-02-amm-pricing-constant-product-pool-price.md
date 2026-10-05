# Module 13 第 2 课｜AMM 的定价逻辑：从 `x * y = k` 理解 Pool Price

## Lesson Contract

所属 Module：Module 13 — DEX（Decentralized Exchange，去中心化交易所）

本课核心问题：

> If an AMM has no order book, how does the pool decide how much token a trader receives?

学完以后，你应该能够解释：

- Constant Product AMM（恒定乘积自动做市商）为什么需要 `x * y = k`。
- `x`、`y`、`k` 分别代表什么。
- Pool Price 为什么可以从 Reserve Ratio（储备比例）理解。
- Trader 买入 / 卖出为什么会改变 Pool Price。
- 为什么交易量越大，对 Pool State 的推动越明显。
- 为什么 Liquidity 越深，同样一笔 Swap 对价格的影响越小。
- 如何从 `reserve_before → swap → reserve_after` 理解一次 AMM Swap。
- 从 Blockchain Data Engineer 视角，哪些 Pool State 对价格解释最重要。

本课暂不展开：

- Execution Price、Price Impact、Slippage 的严格区分；下一课专门讲。
- Impermanent Loss。
- LP Fee 收益模型。
- Uniswap v3 Concentrated Liquidity。
- Tick / sqrtPriceX96。
- Arbitrage 的完整机制。
- AMM 高级数学证明。

本课先使用一个简化模型：

```text
不考虑 trading fee
不考虑 gas
不考虑 router
只有一个 ETH / USDC Pool
```

这样可以先把 Constant Product AMM 的核心逻辑看清楚。

---

## 一、上一课留下的问题：没有 Order Book，价格从哪里来？

传统 Order Book Exchange 中：

```text
Seller asks 3,000 USDC / ETH
Buyer bids 3,000 USDC / ETH
```

价格来自市场参与者提交的订单。

但经典 AMM 中没有这样一张：

```text
bid / ask order book
```

Trader 面对的是：

```text
Liquidity Pool
```

例如：

```text
ETH / USDC Pool

100 ETH
300,000 USDC
```

于是问题变成：

> Pool 如何知道 1 ETH 应该换多少 USDC？

Constant Product AMM 给出的答案是：

```text
x * y = k
```

---

# 二、`x * y = k` 分别是什么？

假设：

```text
x = ETH reserve
y = USDC reserve
```

Pool 当前有：

```text
x = 100 ETH
y = 300,000 USDC
```

那么：

```text
k = x * y
  = 100 * 300,000
  = 30,000,000
```

所以当前 Pool State 可以写成：

```text
100 * 300,000 = 30,000,000
```

其中：

```text
x
= reserve of token X

y
= reserve of token Y

k
= constant product
```

在这个简化模型里，一次 Swap 执行后，需要让新的 Reserve 满足：

```text
x' * y' = k
```

也就是：

> Trader 可以改变 `x` 和 `y`，但不能随意决定输出数量；输出必须受到 invariant（不变量）的约束。

这里的 invariant 就是：

```text
x * y = k
```

---

## 三、先不要把 `k` 理解成“价格”

这是一个非常容易混淆的地方。

```text
k
```

不是：

```text
ETH price
```

也不是：

```text
USDC price
```

它描述的是两个 Reserve 之间必须满足的约束关系。

真正更接近 Pool 当前价格的是：

```text
reserve ratio
```

对于：

```text
x = ETH
y = USDC
```

如果我们要表达：

```text
USDC per ETH
```

那么可以先近似理解为：

```text
price of ETH in USDC
≈ y / x
```

当前：

```text
300,000 / 100
= 3,000
```

所以 Pool 当前价格约为：

```text
1 ETH ≈ 3,000 USDC
```

---

# 四、[视角：Protocol] Price 是 Pool State 的函数

这里要特别建立一个新的视角。

Order Book 中，价格更多来自：

```text
orders
```

Constant Product AMM 中，价格来自：

```text
pool reserves
```

也就是：

```text
Pool State
    ↓
Reserve Ratio
    ↓
Pool Price
```

因此：

> In a constant-product AMM, price is state-dependent.

这句话很重要。

对于 Blockchain Data Engineer 来说，这意味着：

```text
Price
```

不是一个完全独立的数据对象。

它与：

```text
reserve_x
reserve_y
```

直接相关。

---

# 五、Alice 卖出 1 ETH，会发生什么？

当前 Pool：

```text
100 ETH
300,000 USDC
```

Invariant：

```text
x * y = 30,000,000
```

Alice：

```text
token_in  = 1 ETH
token_out = USDC
```

她把 1 ETH 放入 Pool。

那么新的 ETH Reserve：

```text
x' = 101 ETH
```

为了保持：

```text
x' * y' = 30,000,000
```

新的 USDC Reserve 必须是：

```text
y'
= 30,000,000 / 101
≈ 297,029.70 USDC
```

Pool 原来有：

```text
300,000 USDC
```

交易后剩：

```text
297,029.70 USDC
```

因此 Alice 获得：

```text
300,000
- 297,029.70
≈ 2,970.30 USDC
```

所以整个过程是：

```text
Before:

100 ETH
300,000 USDC

Alice adds:

1 ETH

After:

101 ETH
297,029.70 USDC

Alice receives:

≈ 2,970.30 USDC
```

---

## 六、为什么 Alice 没拿到 3,000 USDC？

交易前 Pool Ratio 是：

```text
300,000 / 100
= 3,000 USDC / ETH
```

直觉上可能会认为：

```text
1 ETH
→ 3,000 USDC
```

但实际按照 Constant Product Formula：

```text
1 ETH
→ ≈ 2,970.30 USDC
```

原因不是 Pool “少给了钱”。

真正原因是：

> Alice 的交易本身改变了 Pool State。

交易过程中：

```text
ETH reserve ↑
USDC reserve ↓
```

也就是说：

```text
ETH becomes more abundant in the pool
USDC becomes scarcer in the pool
```

因此继续从 Pool 卖 ETH、买 USDC，会越来越“不划算”。

这就是 AMM 自动调整价格的方法。

它不需要一个人不断修改报价。

Pool State 本身就在改变报价条件。

---

# 七、AMM 的“自动做市”究竟自动在哪里？

上一课我们说：

> Automated Market Maker 的 Automated，不只是“自动交易”。

现在可以进一步精确：

```text
Trader changes reserves
        ↓
Formula constrains new reserves
        ↓
Reserve ratio changes
        ↓
Available exchange rate changes
```

所以它自动完成的是：

> Pricing and inventory adjustment through protocol rules.

传统 Market Maker 可能不断修改：

```text
bid
ask
quantity
```

而 AMM 通过：

```text
Pool State + Formula
```

自动产生类似效果。

---

# 八、如果 Alice 卖很多 ETH，会怎样？

现在仍然从：

```text
100 ETH
300,000 USDC
```

开始。

如果 Alice 不是卖：

```text
1 ETH
```

而是卖：

```text
10 ETH
```

那么：

```text
x' = 110
```

新的 USDC Reserve：

```text
y'
= 30,000,000 / 110
≈ 272,727.27
```

Alice 得到：

```text
300,000
- 272,727.27
≈ 27,272.73 USDC
```

平均每 1 ETH 得到：

```text
27,272.73 / 10
≈ 2,727.27 USDC
```

明显低于交易前看到的：

```text
3,000 USDC / ETH
```

因此我们可以先得到一个重要直觉：

> The larger the trade relative to pool liquidity, the more the pool state moves.

注意，本课先建立这个直觉。

下一课再正式区分：

```text
Pool Price
Execution Price
Price Impact
Slippage
```

---

# 九、为什么 Liquidity 越深，价格越稳定？

现在比较两个 Pool。

Pool A：

```text
100 ETH
300,000 USDC
```

Pool B：

```text
1,000 ETH
3,000,000 USDC
```

它们的 Reserve Ratio 都是：

```text
3,000 USDC / ETH
```

也就是说交易前的 Pool Price 相同。

但是如果同样卖：

```text
1 ETH
```

对于 Pool A：

```text
1 ETH / 100 ETH
= 1%
```

对于 Pool B：

```text
1 ETH / 1,000 ETH
= 0.1%
```

同样一笔交易，对 Pool B 的 Reserve Structure 改变更小。

所以：

> Deeper liquidity means a trade is smaller relative to the pool.

结果就是：

```text
same trade size
+
deeper liquidity
→
smaller state movement
```

这也是为什么：

```text
Liquidity
```

不仅代表“Pool 里有多少钱”。

它还直接影响：

```text
trade capacity
price stability
execution quality
```

---

# 十、[银行系统类比] Pool Reserve 类似“库存”，但不是普通库存

可以用银行外汇兑换做一个有限类比。

假设一个兑换柜台有：

```text
USD inventory
SGD inventory
```

如果大量客户不断：

```text
sell USD
buy SGD
```

那么柜台会逐渐：

```text
USD inventory ↑
SGD inventory ↓
```

现实中的做市商可能主动调整报价，避免某一种货币库存越来越失衡。

AMM 的特殊之处是：

```text
inventory imbalance
```

不是由人工交易员观察后再调报价，

而是由：

```text
mathematical formula
```

直接把 Reserve State 映射成新的交易条件。

所以这个类比最重要的部分是：

> Inventory state affects price.

但 AMM 的具体定价机制仍然由 Smart Contract Protocol Rules 决定。

---

# 十一、[Data Engineer 视角] 一笔 Swap 应该保存哪些状态？

如果我们只记录：

```text
amount_in
amount_out
```

可以知道：

```text
Alice swapped 1 ETH for 2,970.30 USDC
```

但如果想进一步解释：

> 为什么她拿到的是 2,970.30，而不是 3,000？

就需要 Pool State。

例如：

```text
reserve_x_before
reserve_y_before

amount_in
amount_out

reserve_x_after
reserve_y_after
```

于是数据链路就变成：

```text
Pool State Before
↓
Swap
↓
Pool State After
```

这让你能够检查：

```text
x_before * y_before

x_after * y_after
```

以及 Reserve Ratio 如何变化。

---

# 十二、Pool Price 的方向一定要写清楚

这是数据工程里很常见的错误。

假设：

```text
x = ETH
y = USDC
```

那么：

```text
y / x
```

表示：

```text
USDC per ETH
```

例如：

```text
300,000 / 100
= 3,000 USDC / ETH
```

但：

```text
x / y
```

表示的是：

```text
ETH per USDC
```

即：

```text
100 / 300,000
≈ 0.000333 ETH / USDC
```

两者数学上互为倒数，但业务含义不同。

所以一个 Price 字段不能只叫：

```text
price
```

而不说明：

```text
base_token
quote_token
price direction
```

更清楚的表达可能是：

```text
base_token  = ETH
quote_token = USDC

price
= quote_token per base_token
= USDC per ETH
```

---

# 十三、[Data Modeling 视角] Price 必须有 Quote Convention

如果 Fact Table 中直接保存：

```text
price = 3000
```

这个数字本身是不完整的。

你至少需要知道：

```text
3000 what per what?
```

因此一个更完整的语义是：

```text
base_token_address
quote_token_address
price_quote_per_base
```

例如：

```text
base_token  = ETH
quote_token = USDC

price_quote_per_base
= 3000
```

这表示：

```text
1 ETH = 3000 USDC
```

如果 Token 顺序反过来：

```text
base_token  = USDC
quote_token = ETH
```

那么 Price 就应该变成：

```text
≈ 0.000333 ETH / USDC
```

这一点以后做 DEX Analytics 非常重要。

否则不同 Pool、不同 Token Pair 的 Price 很容易被混在一起。

---

# 十四、`x * y = k` 为什么形成一条曲线？

公式：

```text
x * y = k
```

可以改写成：

```text
y = k / x
```

这意味着：

当：

```text
x ↑
```

那么：

```text
y ↓
```

而且不是线性下降。

例如固定：

```text
k = 30,000,000
```

几个可能的 Pool State：

```text
x = 100
y = 300,000

x = 110
y ≈ 272,727

x = 150
y = 200,000

x = 200
y = 150,000
```

这些点都满足：

```text
x * y = 30,000,000
```

所以 AMM 不是在一个固定汇率下交换。

它沿着：

```text
constant product curve
```

移动。

Trader 的 Swap 本质上就是：

> Move the pool from one point on the curve to another point.

---

# 十五、[Protocol 视角] Swap = 沿着 AMM Curve 改变 State

上一课你已经回答：

> A Swap is a state transition of a liquidity pool.

这一课可以把它进一步具体化。

对于 Constant Product AMM：

```text
State Before
(x, y)

↓

Swap

↓

State After
(x', y')

where

x * y ≈ x' * y'
```

在我们目前“不考虑 fee”的简化模型中：

```text
x * y = x' * y'
```

因此：

> The AMM formula constrains valid pool state transitions.

这比仅仅说：

```text
Alice transferred ETH
Pool transferred USDC
```

更接近 Protocol Semantics。

---

# 十六、现实中的 Uniswap v2 为什么会比这个稍复杂？

到这里先只需要知道一件事：

真实交易通常存在：

```text
Trading Fee
```

所以真实的 Uniswap v2 Swap 计算不会简单等于：

```text
(x + amount_in) * (y - amount_out) = k
```

它会对用于定价计算的输入数量做 fee adjustment。

因此现实中：

```text
actual input
```

和：

```text
effective input used for pricing
```

会有差异。

Fee 的细节我们后面再处理。

本课只需要先锁定核心骨架：

```text
Reserve State
+
Invariant
→
Swap Output
→
New Reserve State
```

---

# 十七、那外部市场价格变化怎么办？

例如：

```text
Uniswap ETH price = 3,000 USDC

Binance ETH price = 3,100 USDC
```

AMM 自己并不会主动读取 Binance，然后说：

```text
我要把价格改成 3,100
```

经典 Constant Product Pool 的价格由：

```text
its own reserve state
```

决定。

那么为什么实际 DEX Price 往往会接近外部市场？

一个重要原因是：

```text
Arbitrage
```

套利者会发现价格差，然后进行交易，改变 Pool Reserve，推动 Pool Price 向外部市场靠近。

本课不展开 Arbitrage Strategy。

只保留这个关系：

```text
External Price Difference
↓
Arbitrage Trades
↓
Pool Reserve Changes
↓
AMM Price Moves
```

因此：

> The formula defines how price moves; arbitrage helps decide where the pool is pushed.

---

# 十八、[Data Engineer 视角] 从链上还原 Pool Price 有两种思路

第一种：

```text
State-based
```

使用 Pool Reserve：

```text
reserve_y / reserve_x
```

得到某个状态点的 Pool Price。

第二种：

```text
Trade-based
```

使用某笔 Swap 的：

```text
amount_in
amount_out
```

得到该笔交易实际形成的交换比率。

这两个值不是同一个概念。

现在只先记住：

```text
Reserve Ratio
→ pool state price

Swap Amount Ratio
→ trade execution result
```

下一课我们会正式拆开：

```text
Pool Price
Execution Price
Price Impact
Slippage
```

---

# 十九、为什么不能只看 `amount_in / amount_out` 当 Pool Price？

因为一笔 Swap：

```text
changes the pool while it executes
```

例如交易前：

```text
Pool Price
≈ 3000 USDC / ETH
```

但 Alice 卖 1 ETH 后实际获得：

```text
≈ 2970.30 USDC
```

如果你简单把：

```text
amount_out / amount_in
```

定义成：

```text
pool price
```

那么你把：

```text
trade-level execution result
```

和：

```text
pool-state price
```

混在了一起。

这是 DEX Analytics 里非常关键的语义区别。

---

# 二十、一个 Blockchain Data Engineer 应该如何理解 `x * y = k`

不要只把它背成：

```text
Uniswap formula
```

更重要的是建立这套 Mental Model：

```text
Liquidity Pool
contains reserves

↓

Reserves define Pool State

↓

Protocol invariant constrains
valid state transitions

↓

Trader changes reserves

↓

Reserve ratio changes

↓

Available price changes
```

所以 `x * y = k` 同时连接了：

```text
Protocol Semantics
Pool State
Swap Execution
Price
Liquidity
Data Modeling
```

这也是为什么它是理解 AMM 的入口。

---

# 二十一、把今天的内容放回完整数据链路

链上：

```text
Transaction
↓
Router / Pool Contract Execution
↓
Swap
↓
Token Transfers
↓
Pool Reserve State Changes
```

Indexer：

```text
Logs / State Evidence
↓
Decoder
↓
Swap Fact
```

数据模型：

```text
amount_in
amount_out
pool_address
token_in
token_out
reserve_before
reserve_after
```

Analytics：

```text
Pool Price
Execution Price
Volume
Liquidity
Price Impact
```

这里你会看到：

> AMM mathematics is not isolated math; it determines the business meaning of DEX data.

---

# 本课核心结论

> Constant Product AMM uses the invariant `x * y = k` to constrain pool state transitions.

> `x` and `y` represent token reserves; `k` is the product invariant, not the token price.

> For an ETH / USDC pool, `USDC reserve / ETH reserve` can be used to understand the current pool price in `USDC per ETH`.

> A trader changes the reserves when executing a swap, so the pool price changes as part of the transaction.

> The larger a trade is relative to pool liquidity, the more strongly it moves the reserve state.

> Deeper liquidity generally means the same trade causes a smaller change in pool state.

> A swap in a constant-product AMM can be understood as moving the pool from `(x, y)` to `(x', y')` along the invariant curve.

> From a Data Engineer perspective, Reserve State and Swap Amounts describe different semantics: state-based price and trade-level execution result must not be confused.

> A price field is incomplete unless its base token, quote token, and price direction are clearly defined.

---

# 理解检查

## 问题一

一个 ETH / USDC Pool 当前有：

```text
100 ETH
300,000 USDC
```

暂时忽略 Fee。

请回答：

1. `x`、`y`、`k` 分别是什么？
2. 如果用 `USDC per ETH` 表示 Pool Price，当前价格是多少？
3. `k` 是不是 ETH 的价格？为什么？

---

## 问题二

还是这个 Pool：

```text
100 ETH
300,000 USDC
```

Alice 向 Pool 卖出：

```text
1 ETH
```

交易后：

```text
ETH reserve = 101
```

请你不用追求小数特别精确，说明：

1. 为什么 USDC Reserve 必须下降？
2. 为什么 Alice 得到的 USDC 会少于 3,000？
3. 这笔 Swap 从 Protocol 视角，本质上改变了什么？

---

## 问题三

有两个 Pool：

```text
Pool A
100 ETH
300,000 USDC

Pool B
1,000 ETH
3,000,000 USDC
```

两者交易前价格都是：

```text
3,000 USDC / ETH
```

现在同样有人卖：

```text
1 ETH
```

请回答：

1. 哪个 Pool 的价格变化会更小？
2. 为什么？
3. 这说明 Liquidity 和交易价格稳定性之间是什么关系？

## 用户回答（理解检查）

问题一：

1. X和 Y 代表两种 token，K 代表它们的积数
2. 1 eth = 300,000 USDC
3. K 不是 ETH 的价格，它是 ETH 和 USDC 两种 token 的积数，它代表了当前这个 pool 的深度

问题二：

1. 因为 USDC 确实是被兑换出了，所以它减少了
2. 根据恒定乘积公式计算，得到的交易价格是少于 3000 的，所以 Alice 得到的 USDC 少于 3000
3. 从这笔 swap 从 protocol 视角，改变了 AMM 曲线的状态

问题三：

1. pool B的变化更小
2. 因为根据恒定乘积公式，Pool 的 K 值会更大，说明 Liquidity 的深度更大，所以交易对它的影响会更小。
3. Liquidity 深度越深，交易价格越稳定。

## 老师判断 / 校准（本轮）

本轮暂不判定整课通过。问题二、问题三的核心方向正确；问题一有一个数值错误，同时需要校准 `k` 与 Liquidity Depth 的关系。

### 问题一

1. **`x`、`y`、`k` 的含义：基本正确。**

更精确地说：

```text
x = token X reserve
y = token Y reserve
k = x * y
```

在本题中：

```text
x = 100 ETH
y = 300,000 USDC
k = 30,000,000
```

2. **Pool Price：此处有数量级错误。**

你写的是：

```text
1 ETH = 300,000 USDC
```

正确计算是：

```text
USDC per ETH
= y / x
= 300,000 / 100
= 3,000
```

所以：

```text
1 ETH ≈ 3,000 USDC
```

这里要注意 Price 是一个 ratio，而不是直接取某一个 Reserve。

3. **`k` 不是 ETH 的价格：正确；但“`k` 代表 Pool 深度”需要校准。**

`k` 的直接含义是：

> Constant Product Invariant，也就是两个 Reserve 乘积形成的不变量。

在本题这两个 Pool 具有相同 Token Pair、相同价格比例，而且 Pool B 的两个 Reserve 都按 10 倍放大，所以：

```text
Pool A:
k = 100 * 300,000

Pool B:
k = 1,000 * 3,000,000
```

此时更大的 `k` 确实伴随更深的 Liquidity。

但不能一般化成：

> `k` 就是 Liquidity Depth。

原因是 `k` 受 Token 数量单位、Decimals 和 Reserve Scale 影响。更稳妥的工程表述是：

> Liquidity Depth 应从 Reserve Size、资产价值以及交易规模相对于 Reserve 的比例来理解；在同一 Token Pair、相同单位和相同比例下，更大的 Reserve 通常也会对应更大的 `k`。

### 问题二

整体通过。

1. **为什么 USDC Reserve 下降：正确。**

Alice 向 Pool 输入 ETH，并从 Pool 取走 USDC，因此：

```text
ETH reserve ↑
USDC reserve ↓
```

更进一步，从 Protocol 约束看，新的 Reserve 还必须满足：

```text
x' * y' = k
```

2. **为什么得到少于 3,000 USDC：正确。**

你的核心理解是对的：Swap 本身改变了 Reserve State，所以不能按交易前的 `3,000 USDC / ETH` 固定兑换整笔交易。

这里暂时不要把它进一步定名为 Price Impact / Execution Price；下一课会专门拆开这些概念。

3. **Protocol 视角：基本正确。**

你说“改变了 AMM 曲线的状态”，更精确地可以说：

> Swap moves the pool from one state point `(x, y)` to another state point `(x', y')` along the constant-product curve.

也就是：

```text
Pool State Before
→ Swap
→ Pool State After
```

### 问题三

结论通过，理由需要轻微校准。

1. **Pool B 价格变化更小：正确。**

2. 你用更大的 `k` 来解释，方向在这个具体例子里成立，但更关键的原因是：

```text
same trade = 1 ETH

Pool A:
1 / 100 = 1%

Pool B:
1 / 1,000 = 0.1%
```

也就是同样 1 ETH 的交易，在 Pool B 中只占 ETH Reserve 的 0.1%，因此对 Reserve Ratio 的改变更小。

更推荐记住：

> Trade Size Relative to Liquidity，比单独看 `k` 更能直接解释价格为什么移动。

3. **Liquidity 越深，交易价格越稳定：正确。**

更完整地说：

> For the same trade size, deeper liquidity usually causes a smaller reserve-ratio change and therefore a smaller price movement.

## 本轮校准结论

你已经掌握了本课最核心的三条逻辑：

```text
Reserve State
→ determines Pool Price

Swap
→ changes Reserve State

Deeper Liquidity
→ same trade causes smaller state movement
```

还需要修正两个点后即可判定本课通过：

1. 本题 Pool Price 是：

```text
1 ETH ≈ 3,000 USDC
```

不是 300,000 USDC。

2. 不要直接把：

```text
k = Liquidity Depth
```

作为一般规则。更准确的是：

```text
Liquidity Depth
→ 看 Reserve Size / Reserve Value
→ 看 Trade Size relative to Reserve
```

在相同 Token Pair、相同单位和相同比例的比较中，更大的 Reserve 往往同时产生更大的 `k`，所以两者会相关，但概念上不是同一个东西。


## 用户回答（校准确认）

1. 3000 usdc
2. k 是 Constant Product Invariant（恒定乘积不变量）；`Liquidity Depth`  表示 Trade Size Relative to Liquidity

## 老师判断 / 最终校准

第 1 点正确：

```text
1 ETH ≈ 3,000 USDC
```

第 2 点核心方向已经修正，但还需要做一个术语层面的最终区分：

`Liquidity Depth` 本身不是 “Trade Size Relative to Liquidity”。

更准确地说：

```text
Liquidity Depth
→ describes how much usable liquidity / reserve depth the pool has

Trade Size Relative to Liquidity
→ describes how large a particular trade is compared with that liquidity
```

在解释“为什么同一笔交易对深池子的价格影响更小”时，真正直接有用的是：

```text
Trade Size Relative to Liquidity
```

例如：

```text
Pool A:
1 ETH / 100 ETH = 1%

Pool B:
1 ETH / 1,000 ETH = 0.1%
```

所以可以把最终关系记成：

```text
k
= Constant Product Invariant

Liquidity Depth
= pool reserve / usable liquidity depth

Trade Size Relative to Liquidity
= how large this trade is relative to that depth
```

三者相关，但不是同一个概念。

## 结课判定

Module 13 第 2 课理解检查与必要校准已完成，正式通过。

已经能够：

- 正确解释 `x`、`y` 与 `k` 的含义。
- 使用 Reserve Ratio 计算 Pool Price，并明确 Price Direction。
- 理解 `k` 是 Constant Product Invariant，而不是 Token Price，也不直接等同于 Liquidity Depth。
- 从 Protocol 视角理解 Swap 是 Pool 从 `(x, y)` 到 `(x', y')` 的 State Transition。
- 理解 Swap 会改变 Reserve Ratio，因此会改变 Pool Price。
- 理解同样 Trade Size 下，更深的 Liquidity 会产生更小的 State Movement。
- 使用 Trade Size Relative to Liquidity 解释不同 Pool 的价格稳定性差异。

下一课进入：

```text
Module 13 第 3 课
Execution Price、Price Impact 与 Slippage
```
