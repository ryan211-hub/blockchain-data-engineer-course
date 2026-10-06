# Module 13 第 3 课｜Execution Price、Price Impact 与 Slippage

## Lesson Contract

所属 Module：Module 13 — DEX（Decentralized Exchange，去中心化交易所）

本课核心问题：

> When a trader sees one price before a swap but receives another price after execution, what exactly changed?

学完以后，你应该能够解释：

- Pool Price 是什么。
- Execution Price（成交均价）是什么。
- Price Impact（价格影响）是什么。
- Slippage（滑点）是什么。
- 为什么这三个概念经常被混在一起。
- 为什么 AMM 中大额交易天然会产生 Price Impact。
- 为什么 Slippage 不应该简单等同于“实际成交价比看到的价格差”。
- 从 Data Engineer 视角，如何设计字段避免把这几个指标混淆。

本课暂不展开：

- MEV（Maximal Extractable Value，最大可提取价值）与 Sandwich Attack 的实现细节。
- Uniswap v3 Tick / sqrtPriceX96。
- Aggregator Routing。
- LP 收益与 Impermanent Loss。
- Oracle / TWAP 的完整计算。

本课继续使用上一课的简化模型：

```text
ETH / USDC Pool

100 ETH
300,000 USDC

忽略 trading fee
忽略 gas
忽略 router
```

---

# 一、先把上一课的问题重新放到桌面上

上一课我们得到：

```text
Pool Before:

100 ETH
300,000 USDC
```

交易前 Pool Price：

```text
300,000 / 100
= 3,000 USDC / ETH
```

Alice 卖出：

```text
1 ETH
```

按照 Constant Product AMM：

```text
x * y = k
```

交易后大约变成：

```text
101 ETH
297,029.70 USDC
```

Alice 得到：

```text
≈ 2,970.30 USDC
```

这里同时出现了几个不同的“价格”：

```text
3,000
2,970.30
交易后的新 Pool Price
```

如果把这些都叫：

```text
price
```

数据一定会乱。

所以这一课的任务就是拆开它们。

---

# 二、Pool Price：交易发生前，Pool 当前状态对应的价格

先定义最基础的：

```text
Pool Price
```

假设：

```text
x = ETH reserve
y = USDC reserve
```

那么使用：

```text
USDC per ETH
```

作为报价方向时：

```text
Pool Price
= y / x
```

交易前：

```text
300,000 / 100
= 3,000 USDC / ETH
```

所以：

```text
Pool Price Before
= 3,000 USDC / ETH
```

它描述的是：

> 当前这个 Pool State 对应的瞬时价格。

注意关键词：

```text
state-based
```

也就是说：

```text
Pool State
↓
Reserve Ratio
↓
Pool Price
```

---

# 三、Execution Price：这一整笔交易实际成交的平均价格

Alice 实际：

```text
amount_in  = 1 ETH
amount_out ≈ 2,970.30 USDC
```

那么这笔交易的：

```text
Execution Price
```

可以计算为：

```text
amount_out / amount_in
```

所以：

```text
2,970.30 / 1
≈ 2,970.30 USDC / ETH
```

也就是说：

```text
Execution Price
≈ 2,970.30 USDC / ETH
```

它回答的问题不是：

> Pool 现在报价多少？

而是：

> 这整笔 Swap 平均下来，我实际以什么价格成交？

因此：

```text
Pool Price
= state-level price

Execution Price
= trade-level average price
```

这是本课第一个必须锁死的区分。

---

# 四、为什么 Execution Price 不等于交易前 Pool Price？

因为在 AMM 中，Alice 的 1 ETH 不是全部按照：

```text
3,000 USDC / ETH
```

成交的。

Swap 执行过程中，Pool State 持续从：

```text
(100 ETH, 300,000 USDC)
```

移动到：

```text
(101 ETH, 297,029.70 USDC)
```

也就是说：

```text
reserve ratio
```

在整个 Swap 过程中一直变化。

可以把它想成：

```text
第一个很小的 ETH
→ 接近 3,000 的价格

后续 ETH
→ 稍微更差的价格

再后续
→ 更差
```

最终把整笔交易平均起来：

```text
Execution Price < Initial Pool Price
```

对于 Alice 这种：

```text
sell ETH
buy USDC
```

的方向而言，就是如此。

---

# 五、Price Impact：你的交易自己把市场推走了多少

现在进入第二个概念：

```text
Price Impact
```

Price Impact 描述：

> 这笔交易由于自身规模，对 Pool Price 造成了多大影响。

它的核心原因来自：

```text
Trade Size Relative to Liquidity
```

上一课我们已经学过：

```text
same trade size
+
shallower liquidity
→ larger state movement
```

所以：

```text
larger state movement
→ larger price impact
```

如果 Alice 只卖：

```text
0.01 ETH
```

对一个 100 ETH Reserve 的 Pool 影响很小。

如果 Alice 卖：

```text
20 ETH
```

那就会显著改变 Reserve Ratio。

因此：

> Price Impact is endogenous to the trade.

这里的 endogenous 可以理解成：

```text
由这笔交易自身造成
```

---

# 六、[Protocol 视角] Price Impact 来自 AMM Curve

对于 Constant Product AMM：

```text
x * y = k
```

Swap 本质上是：

```text
(x, y)
↓
trade
↓
(x', y')
```

当 Trade Size 很大时：

```text
x' - x
```

占原 Reserve 的比例更大。

于是：

```text
y / x
```

与：

```text
y' / x'
```

差异也更明显。

因此：

> Price Impact is a direct consequence of moving along the AMM curve.

它不是系统出错，也不是交易执行异常。

这是 AMM Market Structure 的正常结果。

---

# 七、那 Slippage 又是什么？

这是最容易混淆的部分。

很多人会把：

```text
Execution Price 比页面看到的价格差
```

全部叫 Slippage。

但严格来说，这样不够精确。

Slippage 更适合描述：

> 你在提交交易时预期的执行结果，与交易真正上链执行时得到的结果之间的偏差。

重点是两个时点：

```text
Expected at submission
vs
Actual at execution
```

为什么会不同？

因为 Blockchain Transaction 不是瞬间执行：

```text
User signs transaction
↓
Transaction enters mempool / propagation
↓
Other transactions may execute
↓
Your transaction executes
```

在这段时间里：

```text
Pool State may change
```

因此你提交交易时估算：

```text
expected_amount_out = 2,970 USDC
```

但真正执行时可能只拿到：

```text
2,950 USDC
```

这部分偏差才更接近 Slippage。

---

# 八、Price Impact 与 Slippage 的核心区别

这是整课最重要的区分。

Price Impact：

```text
Your trade
↓
changes Pool State
↓
changes price
```

Slippage：

```text
You expected execution under State A
↓
transaction waits
↓
market / pool changes to State B
↓
actual execution differs from expectation
```

因此：

```text
Price Impact
= caused by your own trade size

Slippage
= deviation between expected and actual execution
```

可以先这样记。

---

# 九、一个非常关键的例子

假设当前：

```text
Pool Price
= 3,000 USDC / ETH
```

Alice 准备卖：

```text
10 ETH
```

根据当前 Pool State，前端已经算出：

```text
Expected Amount Out
≈ 27,272 USDC
```

这已经包含了：

```text
Price Impact
```

因为 10 ETH 本身会推动 AMM Curve。

于是：

```text
Expected Execution Price
≈ 2,727.2 USDC / ETH
```

注意：

```text
3,000
→ 2,727.2
```

这一大段差异主要不是 Slippage。

它首先来自：

```text
Price Impact
```

---

# 十、真正执行时又发生了什么？

Alice 签名以后，在她的 Transaction 执行之前，Bob 先进行了一笔大额 Swap。

于是 Pool State 改变。

原本 Alice 预计：

```text
27,272 USDC
```

实际执行只拿到：

```text
27,000 USDC
```

那么：

```text
27,272
→ 27,000
```

这一部分：

```text
expected execution
vs
actual execution
```

才是我们这一课所说的 Slippage。

所以完整链路是：

```text
Initial Pool Price
↓
your own trade size causes Price Impact
↓
Expected Execution Price
↓
pool changes before execution
↓
Actual Execution Price
↓
Slippage
```

---

# 十一、为什么很多产品界面还是把它们混在一起？

因为用户视角看到的往往只是：

```text
页面显示价格
vs
最后成交价格
```

而用户不一定关心差异来自：

```text
AMM curve
other trades
MEV
latency
routing
fee
```

所以产品界面可能用比较宽泛的：

```text
slippage
```

来表达“成交不如预期”。

但从：

```text
Protocol Analytics
Data Modeling
Risk Analysis
```

视角，我们应该尽量把原因拆开。

---

# 十二、Slippage Tolerance 又是什么？

DEX 前端通常允许用户设置：

```text
Slippage Tolerance
```

例如：

```text
0.5%
1%
3%
```

这不是说：

> 我希望发生 1% Slippage。

而是：

> 我最多接受实际执行结果比预期差多少。

例如：

```text
Expected Amount Out
= 10,000 USDC

Slippage Tolerance
= 1%
```

那么最低接受：

```text
Minimum Amount Out
= 9,900 USDC
```

如果执行时只能得到：

```text
9,850 USDC
```

Smart Contract 可以：

```text
revert
```

从而保护 Trader。

---

# 十三、[Protocol 视角] minimumAmountOut 是交易保护条件

很多 Router 调用会带类似语义的参数：

```text
amountOutMin
minimumAmountOut
minAmountOut
```

具体名字取决于 Protocol。

它表达：

```text
Actual Amount Out
must be >=
Minimum Acceptable Amount Out
```

否则：

```text
Transaction Reverts
```

所以：

```text
Slippage Tolerance
```

最终往往会被转换成：

```text
execution constraint
```

写入交易参数。

---

# 十四、用银行外汇兑换做类比

假设手机银行告诉你：

```text
当前 USD/SGD = 1.35
```

你准备兑换：

```text
1,000,000 USD
```

如果市场深度有限，大额订单本身可能导致不同档位成交。

这更接近：

```text
Price Impact
```

而你点击确认以后，到真正执行之前，市场价格从：

```text
1.35
```

变成：

```text
1.348
```

这部分预期与实际之间的变化，更接近：

```text
Slippage
```

虽然传统 FX Market 与 AMM 结构不同，但这个类比有助于区分：

```text
订单自身造成的影响
vs
等待执行期间市场变化
```

---

# 十五、[Data Engineer 视角] 三种 Price 必须分字段

假设我们设计：

```text
dex_swaps
```

不要只放：

```text
price
```

更合理的是明确语义，例如：

```text
pool_price_before
execution_price
pool_price_after
```

并结合：

```text
amount_in
amount_out
reserve_before
reserve_after
```

这样可以回答：

```text
交易前 Pool Price 是多少？
实际平均成交价是多少？
交易以后 Pool Price 变成多少？
```

否则一个：

```text
price = 2970
```

无法知道它到底是哪种 Price。

---

# 十六、Price Impact 应如何计算？

概念上可以写成：

```text
Price Impact
≈ difference between
initial pool price
and trade execution price
```

例如：

```text
Initial Pool Price
= 3,000

Execution Price
= 2,970.30
```

粗略比例：

```text
(3,000 - 2,970.30) / 3,000
≈ 0.99%
```

因此可以说这笔交易的 Execution 相对于 Initial Pool Price 差约：

```text
0.99%
```

不过在真实 Protocol Analytics 里，具体 Price Impact 定义可能取决于：

```text
reference price
fee treatment
token direction
protocol conventions
```

所以 Data Engineer 不应只看到字段名：

```text
price_impact
```

就假设所有协议口径完全一致。

---

# 十七、Slippage 又该如何计算？

假设交易提交时：

```text
Expected Amount Out
= 2,970 USDC
```

实际：

```text
Actual Amount Out
= 2,950 USDC
```

那么可以从 Output Amount 角度计算：

```text
Slippage
≈ (Expected - Actual) / Expected
```

即：

```text
(2,970 - 2,950) / 2,970
≈ 0.67%
```

这里比较的是：

```text
Expected Execution
vs
Actual Execution
```

不是：

```text
Initial Pool Price
vs
Execution Price
```

这就是口径上的核心差异。

---

# 十八、一个非常重要的数据限制

从纯链上历史数据中，我们通常很容易知道：

```text
Actual Amount In
Actual Amount Out
Pool State
Transaction
Logs
```

但是：

```text
Expected Amount Out at submission time
```

不一定天然存在于链上 Fact 中。

为什么？

因为：

```text
Expectation
```

可能是在：

```text
frontend
router quote
aggregator quote
wallet UI
```

阶段计算的。

所以从 Data Engineer 角度：

> Slippage is often harder to reconstruct accurately than Execution Price or Pool Price.

这一点非常重要。

---

# 十九、[Data Lineage 视角] 哪些值是 On-chain Fact，哪些是 Derived？

可以做一个初步分类。

On-chain / Protocol Evidence：

```text
amount_in
amount_out
pool
reserve state
transaction parameters
minimum amount out
logs
```

Derived：

```text
execution_price
pool_price
price_impact
```

而：

```text
expected quote
```

有时需要额外来源：

```text
frontend quote logs
aggregator API
quote service
off-chain telemetry
```

因此如果有人说：

> 只用 Swap Event，我要准确计算用户实际 Slippage。

你应该立即问：

```text
What is the reference expectation?
```

没有 Expected Quote，就很难严格定义 Slippage。

---

# 二十、Slippage Tolerance 不等于 Actual Slippage

例如用户设置：

```text
Slippage Tolerance = 1%
```

最后实际：

```text
Expected Output = 10,000
Actual Output   = 9,980
```

Actual Slippage：

```text
0.2%
```

不是：

```text
1%
```

所以：

```text
slippage_tolerance
```

是：

```text
constraint / user setting
```

而：

```text
actual_slippage
```

是：

```text
observed execution deviation
```

两者 Grain 和业务语义不同。

---

# 二十一、为什么这个区分对 Blockchain Data Engineer 很重要？

因为实际产品可能问：

```text
Which pools have the highest price impact?

Which wallets consistently suffer high slippage?

What is the average execution price?

Did users receive worse execution than quoted?

Which pools provide the best execution quality?
```

这些看起来都在问“价格”，但底层数据需求完全不同。

例如：

```text
Execution Price
→ amount_in + amount_out

Pool Price
→ pool state

Price Impact
→ reference pool price + execution price

Actual Slippage
→ expected quote + actual execution
```

这就是 Protocol Semantics 对数据模型的直接影响。

---

# 二十二、把四个概念放在一张图里

```text
Pool State Before
↓
Pool Price
↓
Trader asks for quote
↓
Expected Execution Price
    └─ already includes expected Price Impact
↓
Transaction submitted
↓
Pool may change before execution
↓
Actual Swap
↓
Execution Price
↓
Compare Expected vs Actual
↓
Actual Slippage
```

同时：

```text
Initial Pool Price
vs
Execution Price
↓
Price Impact
```

这是今天最重要的一张 Mental Model。

---

# 二十三、注意：真实世界中的边界没有这么绝对

在不同协议、钱包、DEX Analytics 产品中：

```text
Price Impact
Slippage
Execution Quality
```

具体定义可能略有不同。

尤其产品文案里：

```text
slippage
```

经常被宽泛使用。

所以作为 Data Engineer，最重要的不是死背单词，而是明确：

```text
Reference Value
Observed Value
Formula
Data Source
Timestamp / State
```

也就是说任何指标都必须问：

> What exactly are we comparing?

---

# 二十四、[Data Modeling 视角] 建议的字段语义

以后如果设计 DEX Analytics，可以考虑类似：

```text
pool_price_before
pool_price_after

amount_in
amount_out

execution_price

expected_amount_out
actual_amount_out

price_impact_pct
slippage_pct
slippage_tolerance_pct
```

但要注意：

```text
expected_amount_out
```

是否真的能获得，要看数据来源。

因此不要为了“表看起来完整”就制造不存在的字段。

---

# 二十五、从上一课到这一课的知识连接

上一课：

```text
Reserve State
↓
Pool Price
```

这一课：

```text
Pool Price
+
Trade Size
↓
Price Impact
↓
Expected Execution

then

Expected Execution
vs
Actual Execution
↓
Slippage
```

所以你现在开始从：

```text
AMM formula
```

进入：

```text
execution quality
```

这也是 DEX Analytics 的核心业务语义之一。

---

# 本课核心结论

> Pool Price is a state-based price derived from the current pool state.

> Execution Price is the average price actually obtained by a specific swap.

> Price Impact is the price movement caused by the trader's own trade relative to available liquidity.

> A large trade can have significant Price Impact even if no other transaction occurs.

> Slippage is better understood as the deviation between expected execution and actual execution.

> Expected Execution Price may already include Price Impact.

> Slippage Tolerance is a user-defined execution constraint, not the same as Actual Slippage.

> `amountOutMin`-like parameters can protect traders by reverting transactions when execution becomes worse than the accepted threshold.

> Pool Price and Execution Price can often be reconstructed from on-chain data, while precise Actual Slippage may require an off-chain expected quote.

> For a Blockchain Data Engineer, every price-related metric must explicitly define its reference value, observed value, formula, data source, and state/time context.

---

# 理解检查

## 问题一

假设：

```text
Pool Price Before
= 3,000 USDC / ETH

Alice swaps
1 ETH

Actual Amount Out
= 2,970 USDC
```

请回答：

1. `3,000 USDC / ETH` 属于什么 Price？
2. `2,970 USDC / ETH` 属于什么 Price？
3. 为什么二者不同不能简单全部叫做 Slippage？

---

## 问题二

Alice 准备卖出：

```text
10 ETH
```

前端根据当前 Pool State 计算：

```text
Expected Amount Out
= 27,272 USDC
```

Alice 提交 Transaction 后，Bob 先完成另一笔 Swap。

Alice 最后实际得到：

```text
Actual Amount Out
= 27,000 USDC
```

请回答：

1. `3,000 → 约 2,727 USDC / ETH` 这一变化主要对应什么？
2. `Expected 27,272 → Actual 27,000` 这一变化主要对应什么？
3. 为什么两者的原因不同？

---

## 问题三

从 Blockchain Data Engineer 视角，如果只有：

```text
Swap Event
amount_in
amount_out
pool state
```

请回答：

1. 能不能计算 Execution Price？
2. 能不能计算 Pool Price？
3. 能不能严格计算用户的 Actual Slippage？
4. 如果不能，还缺什么关键数据？

## 用户回答（理解检查｜问题一）

1. Pool Price
2. Execution Price
3. Pool Price 和 Execution Price 的不同，是称为 Price Impact 。 而 Slippage 是 Expected Execution 和 Actual Execution的不同

## 老师判断 / 校准（问题一）

问题一通过，而且第 3 点区分得很准确。

1. `3,000 USDC / ETH` 是 `Pool Price`。它来自交易前的 Pool State / Reserve Ratio，属于 state-level price。

2. `2,970 USDC / ETH` 是 `Execution Price`。它来自这笔 Swap 的 `amount_out / amount_in`，属于 trade-level average price。

3. 核心区分正确：

```text
Initial Pool Price
vs
Execution Price
→ Price Impact

Expected Execution
vs
Actual Execution
→ Slippage
```

再补一个小精度点：`Pool Price` 与 `Execution Price` 的差异可以用于描述 Price Impact，但真实分析时还要明确 reference price、fee treatment 和 token direction，不能只看两个数字就默认所有协议的 `price_impact_pct` 口径完全一致。

本题判定：通过。


## 用户回答（理解检查｜问题二、问题三）

问题二：

1. Price Impact
2. Slippage
3. Price Impact = caused by your own trade size

   Slippage = deviation between expected and actual execution

问题三：

1. 可以
2. 可以
3. 不能
4. 缺少提交swap瞬间，pool state的数据，因为pool state有可能发生变化

## 老师判断 / 校准（问题二、问题三）

### 问题二

整体通过。

1. `3,000 → 约 2,727 USDC / ETH` 主要对应 `Price Impact`：正确。

2. `Expected 27,272 → Actual 27,000` 主要对应 `Slippage`：正确。

3. 原因区分正确：

```text
Price Impact
= caused by your own trade size

Slippage
= deviation between expected and actual execution
```

更完整地说：

```text
Price Impact
→ the trade itself moves the pool along the AMM curve

Slippage
→ actual execution differs from the quote / expectation formed before execution
```

### 问题三

1. 能计算 Execution Price：正确。

只需要：

```text
amount_in
amount_out
```

即可得到 trade-level average execution price。

2. 能计算 Pool Price：正确。

前提是 `pool state` 中包含足够的 Reserve / Price State，并且明确 token direction。

3. 不能严格计算用户的 Actual Slippage：正确。

4. 这里需要校准。

你说缺少：

```text
提交 Swap 瞬间的 Pool State
```

这个信息确实可能有帮助，但它不是严格计算 Slippage 最核心、最直接缺少的数据。

严格定义 Slippage 时真正需要的是：

```text
Expected Execution
vs
Actual Execution
```

所以关键缺口是：

```text
Expected Quote
或
Expected Amount Out at submission time
```

例如：

```text
Expected Amount Out
= 27,272 USDC

Actual Amount Out
= 27,000 USDC
```

这样才能计算：

```text
Actual Slippage
≈ (Expected - Actual) / Expected
```

提交时的 Pool State 只能在你同时知道：

```text
routing
pricing formula
fee treatment
quote logic
```

等条件时，用来重建 Expected Quote。

因此从 Data Lineage 视角，最稳妥的说法是：

> To calculate actual slippage strictly, the missing reference is the expected quote at submission time, not merely the pool state.

本轮判定：

- 问题二：通过。
- 问题三第 1～3 点：通过。
- 问题三第 4 点：需要把“提交时 Pool State”修正为“Expected Quote / Expected Amount Out at submission time”。

完成这一点后，本课即可结课。


## 用户确认（最终校准）

好的，我知道了。

## 结课判定

Module 13 第 3 课理解检查与必要校准已完成，正式通过。

本课已经能够：

- 区分 `Pool Price` 与 `Execution Price`。
- 理解 `Price Impact` 主要来自交易自身相对于 Liquidity 的规模。
- 理解 `Slippage` 是 Expected Execution 与 Actual Execution 的偏差。
- 区分 `Slippage Tolerance` 与 `Actual Slippage`。
- 理解 `Expected Quote / Expected Amount Out at submission time` 是严格计算 Actual Slippage 的关键 reference。
- 从 Data Engineer 视角区分 on-chain fact、derived metric 与 off-chain quote data。

下一课进入：

```text
Module 13 第 4 课
Liquidity Provider：Liquidity、Fee 与 Impermanent Loss
```
