# Module 13 第 9 课｜DEX Analytics：Volume、Liquidity、Price、Fee 与 Wallet Behavior

## Lesson Contract

所属阶段：第四阶段 Protocol  
所属 Module：Module 13 — DEX（Decentralized Exchange，去中心化交易所）

**本课核心问题：**

> How can we tell whether a DEX liquidity pool is actively traded, sufficiently liquid, and economically attractive?

中文意思是：一家 Web3 数据公司想判断某个 DEX Pool 的交易是否活跃、流动性是否充足、能产生多少手续费，我们怎样利用链上数据提供可信的分析结果？

学完本课，你应能：

1. 从业务问题推导 Volume、Liquidity、Price、Fee 和 Wallet Activity 的指标。
2. 为各指标确定 Grain、数据来源和计算方法。
3. 区分 TVL（Total Value Locked，锁定总价值）、Reserve、Trading Volume、Pool Price、Execution Price。
4. 区分 Gross Swap Fee、Protocol Fee 与 LP（Liquidity Provider，流动性提供者）收益。
5. 说明为何 Wallet Address 数量不等于真实用户数量。
6. 判断一个指标是否具备可靠的数据来源、估值方法及时间边界。

**本课不展开：** 复杂 LP Position 收益归因、MEV（Maximal Extractable Value，最大可提取价值）检测算法、价格预言机实现、交易策略与 Dashboard 前端。

---

## 一、业务场景：产品经理想知道 Pool A 的经营情况

继续使用上一课的 Ethereum Uniswap v2 USDC/WETH Pool A。

上一课我们已经建立：

```text
dim_dex_pool
fact_dex_pool_swap
pool_state_change_history
daily_pool_snapshot
dim_token
```

产品经理现在不满足于查看交易明细。他希望每天生成一份 Pool Analytics Report，回答五个问题：

1. **Volume：** 今天发生了多少交易？
2. **Liquidity：** 这个 Pool 里有多少资金可用于交易？
3. **Price：** 这个 Pool 当前给出的兑换价格是多少？
4. **Fee：** 今天的交易产生了多少手续费？
5. **Wallet Behavior：** 有多少地址参与了交易？是否存在重复活跃的地址？

这些问题看起来可以直接做 SQL 聚合，但我们很快会发现：每个问题对应的数据对象和指标口径不完全一样。

为了避免不停切换场景，本课全部使用下面这一组教学数据。

### Pool A：2026-10-09（UTC）

假设在当天日末，已确认的 Canonical Pool State 是：

```text
Pool A: USDC / WETH

reserve0 = 120,000 USDC
reserve1 = 40 WETH

Reference WETH Price = 3,000 USD
Reference USDC Price = 1 USD
```

当天发生三次 Swap：

| Swap | USDC 侧成交量 | Transaction Sender |
|---|---:|---|
| Swap 1 | 3,000 USDC | Alice |
| Swap 2 | 2,000 USDC | Bob |
| Swap 3 | 3,000 USDC | Alice |

假设对应交易的美元估值均采用 `1 USDC = 1 USD`，且三次 Swap 都来自独立的用户兑换请求。

这里的 Token 余额、价格、交易地址均为教学假设，并非真实链上历史数据。

我们的任务，是从这组数据逐步构建可信的 Analytics Report。

---

## 二、第一个问题：如何判断 Pool 的交易活跃程度？

产品经理问：

> Pool A 今天的 Trading Volume 是多少？

从 `fact_dex_pool_swap` 可以得到三次 Swap：

```text
Swap 1: $3,000
Swap 2: $2,000
Swap 3: $3,000
```

因此：

```text
Pool Gross Volume = $8,000
Pool Swap Count   = 3
```

到这里似乎很简单。

但如果产品经理继续问：

> 那么 8,000 美元是否表示用户总共向 Uniswap 投入了 8,000 美元？

答案是：不一定。

原因是上一课已经建立的：

```text
Pool Execution Grain
≠
User Swap Intent Grain
```

在当前教学场景中，我们假设三次交易分别属于独立的用户兑换请求。但在一般的 Multi-hop 场景中，Pool Gross Volume 可以高于 User-level Routed Volume。

因此，第一项工程决策是：

**先明确我们统计的是 Pool Trading Activity，而不是 User-level Routed Activity。**

建议指标：

```text
pool_gross_volume_usd
pool_swap_count
```

其聚合 Grain：

```text
one UTC date
+
one chain
+
one pool
```

这一组指标可以直接从经过标准化和美元估值处理的 Pool Swap Fact 汇总。

但这只回答了交易活跃程度，还不能回答 Pool 是否拥有充足的流动性。

---

## 三、第二个问题：Trading Volume 高，是否代表 Liquidity 高？

现在产品经理问：

> Pool A 一天交易了 8,000 美元。那么它的 Liquidity 是多少？

有人可能直接回答：

```text
Liquidity = $8,000
```

这就混淆了两个不同的业务概念。

**Volume 是一段时间内发生的交易活动；Liquidity 是某个时间点的市场资金与成交能力。**

银行类比：

- 银行一天发生的转账总额，类似 Transaction Volume。
- 银行在特定时点持有的资金或可用于某项业务的头寸，是另一个概念。

一天可以发生很多笔交易，但这些交易不代表每天都需要新增相同规模的资金。

### 1. 从 Pool Reserve 推导 TVL

上一课已经保存了 Pool A 的日末状态：

```text
120,000 USDC
40 WETH
```

如果分别使用假定的独立美元参考价格：

```text
USDC = $1
WETH = $3,000
```

那么：

```text
USDC Value = 120,000 × 1
           = $120,000

WETH Value = 40 × 3,000
           = $120,000

Pool TVL   = $240,000
```

于是 Pool A 当天的两个指标是：

```text
Daily Pool Volume = $8,000
End-of-Day TVL    = $240,000
```

注意这两个指标的时间语义不同：

- Daily Volume：整天的交易量，是 **Period Metric（期间指标）**。
- End-of-Day TVL：日末时点的资金估值，是 **Point-in-time Metric（时点指标）**。

不能把一整天的 Volume 与任意时刻的 Reserve 混淆。

### 2. TVL 高是否一定代表交易体验好？

不一定。

你可能认为，TVL 越高，用户交易越不容易受到 Price Impact。

这个判断有一定直觉基础，但不能只靠 TVL 确定交易深度。

对于 Uniswap v2，Reserve 规模和比例会影响不同订单规模下的 Execution Price 与 Price Impact。

到了 Uniswap v3，还要考虑：

```text
Active Liquidity
Current Price
Tick / Position Range
```

因为 v3 的部分流动性可能位于当前价格区间之外，不能简单把整个 Pool 的 TVL 当作当前价格附近的可用深度。

所以，如果产品经理真正要比较“交易 10,000 USDC 会产生多大的 Price Impact”，更合适的分析是：

> 在明确的 Pool State 下，对指定交易规模计算或模拟报价。

而不是简单排序：

```text
ORDER BY tvl_usd DESC
```

本课先记录这个边界，不重新展开第 6 课的 Tick 数学。

官方参考：[Uniswap Concentrated Liquidity](https://developers.uniswap.org/docs/get-started/concepts/liquidity-providers/concentrated-liquidity)、[How Uniswap Works](https://developers.uniswap.org/docs/get-started/concepts/how-uniswap-works)。

---

## 四、第三个问题：Pool 当前价格是多少？

产品经理继续问：

> Pool A 中 1 WETH 大约值多少 USDC？

现在我们有：

```text
reserveUSDC = 120,000
reserveWETH = 40
```

对于本例的 Uniswap v2 Pool：

```text
Pool Spot Price
= reserveUSDC / reserveWETH
= 120,000 / 40
= 3,000 USDC per WETH
```

这里的价格是从 Pool Reserve 推导出的 **Spot Price（即时边际价格）**。

但假设 Alice 发起：

```text
Buy 1 WETH with USDC
```

她的实际成交价格未必恰好是：

```text
3,000 USDC
```

为什么？

因为执行 Swap 会改变 Reserve，形成 Price Impact；同时还存在 Swap Fee。

因此：

```text
Pool Spot Price
≠
Execution Price
```

### 价格指标又出现新的问题

如果我们需要查询：

> 2026-10-09 的 Pool Price 是多少？

这个问题不够精确。

我们至少需要明确：

- 是每天结束时的 Spot Price？
- 是每次 Swap 的 Execution Price？
- 是全天按交易量加权的成交价格（VWAP，Volume-Weighted Average Price）？
- 还是时间加权平均价格（TWAP，Time-Weighted Average Price）？

它们都可以称为 Price Metrics，但计算语义不同。

为本次 Daily Report，我们先选择：

```text
end_of_day_pool_spot_price
```

它来自日末 Pool State，而不是对当天所有 Swap Price 简单求平均。

如果产品以后要求 VWAP 或 TWAP，需要重新定义计算方式及数据来源。

这里的工程结论是：

**Price 不是一个无需说明时间、方向和计算方法的单一字段。**

另外，从 Pool Reserve 计算的价格不等于可信的外部美元参考价。对 TVL 和 USD Volume 估值时，不能未经验证就把一个可能被操纵的 Pool Spot Price 当作独立市场价格。

---

## 五、第四个问题：交易手续费是多少？

现在产品经理问：

> 既然 Pool A 今天交易了 8,000 美元，那么它创造了多少手续费？

对于本课的 Uniswap v2 示例，Swap Fee Rate 为：

```text
0.30%
```

在“每笔输入资产按美元估值后，与本例成交名义额近似一致”的简化条件下：

```text
Estimated Gross Swap Fee USD
≈ $8,000 × 0.30%
≈ $24
```

这里的 `$24` 是**示意性的 Gross Swap Fee 估值**，并不是从某个 Fee Transfer Event 直接读取的数字。

这又带出了一个设计问题。

### 1. Gross Swap Fee 是否等于 LP 收益？

不能直接画等号。

至少要区分：

```text
Gross Swap Fee
    │
    ├── LP-attributable Fee
    └── Protocol Fee（如果适用）
```

对于 Uniswap v2，交易手续费通常体现在池内资产及 LP 份额的经济机制中，不是每笔 Swap 都向 LP 钱包转一笔手续费。

而且，协议手续费机制及其启用状态也会影响 LP 与 Protocol 的经济分配。

因此：

```text
Gross Fee
≠
Protocol Revenue
≠
Individual LP Realized Earnings
```

截至本课，Uniswap 官方文档已说明 v2、v3 的 Protocol Fee 配置及分配机制；生产系统必须按 **Protocol Version、Pool、历史生效配置和具体计费机制** 处理，而不能假定每个版本、每个时间段的分配规则完全相同。

官方参考：[Uniswap Fees](https://developers.uniswap.org/docs/get-started/concepts/fees)、[Protocol Fee Configuration](https://developers.uniswap.org/docs/protocols/protocol-fee/concepts/fees)。

### 2. 我们应该怎样设计指标？

对于本次 Pool Report，可以先计算：

```text
estimated_gross_swap_fee_usd
```

但不要将它直接命名为：

```text
lp_profit_usd
```

也不能直接把每天的 Gross Fee 都称作 Protocol Revenue。

真正计算某个 LP 的收益，还需要结合 LP 在相关时间的份额或 Position、有效流动性区间、Fee Growth、提取记录及协议手续费分配规则。

这属于本课后续的扩展范围，当前先把 Metrics Definition 做正确。

---

## 六、第五个问题：今天有多少交易用户？

现在产品经理提出最后一个需求：

> Pool A 今天有几个 Unique Traders？

我们观察到：

```text
Swap 1: tx.from = Alice
Swap 2: tx.from = Bob
Swap 3: tx.from = Alice
```

如果定义的是 **Unique Transaction Senders**，那么：

```text
unique_tx_senders = 2
```

因为不同地址只有 Alice 和 Bob。

但如果产品经理真正问的是：

> 有多少个真实用户使用了这个 Pool？

我们不能立即把答案也写成 2。

### 为什么不能？

因为链上不同字段代表不同角色。

例如：

```text
tx.from
Swap.sender
Swap.recipient
```

不一定对应同一个业务主体。

一个用户可以使用多个地址；一个地址也可能代表合约、服务或多个用户操作的入口。

尤其当请求经过 Router、Aggregator、Smart Wallet 或 Relayer 时，识别最终经济交易者需要更多证据。

所以这里应区分两个指标：

```text
unique_tx_senders
```

与：

```text
unique_attributed_traders
```

前者通常容易从 Transaction 与 Swap 关联中计算。

后者需要 **Trader Attribution（交易者归因）** 规则，并对无法确定的情况保留不确定性。

更重要的是：

**Unique Wallet Addresses 不等于 Unique Human Users。**

### 进一步观察 Wallet Behavior

如果产品经理改问：

> 哪些交易地址在今天发生了多次交易？

在本教学场景下，我们可以得到：

```text
Alice: 2 swaps
Bob:   1 swap
```

这属于按 Transaction Sender 观察的地址活动，不应直接解释成“真实用户忠诚度”。

如果以后计算：

```text
daily_active_trading_addresses
repeat_trading_addresses
```

也应明确是统计 Transaction Sender、归因交易者地址，还是用户级 Route Owner。

因此，钱包行为分析的第一步不是创建用户画像，而是定义**谁被算作一个交易主体**。

---

## 七、现在我们终于能够交付一份 Daily Pool Report

回顾最初五个问题，我们得到下面这份结果。

| Pool A · USDC/WETH（2026-10-09 UTC） | 值 |
|---|---:|
| Pool Gross Volume | $8,000 |
| Pool Swap Count | 3 |
| End-of-Day TVL（按参考价格估值） | $240,000 |
| End-of-Day Pool Spot Price | 3,000 USDC per WETH |
| Estimated Gross Swap Fee（0.30%，简化估计） | ~$24 |
| Unique Transaction Senders | 2 |

该报告为假设数据；不代表真实 Uniswap 池的历史指标，也不等同于真实用户数或 LP 实际收益。

上面的结果应该如何保存？

不需要为每个指标都创建一个独立的事实表。

可以建立：

```text
dws_dex_pool_daily

Grain:
one UTC date
+
one chain
+
one pool
```

部分示意字段：

```text
trade_date_utc
chain_id
pool_address

pool_swap_count
pool_gross_volume_usd

end_of_day_tvl_usd
end_of_day_pool_spot_price

estimated_gross_swap_fee_usd
unique_tx_senders
```

但要牢记：**这些字段虽然放在同一个 Daily Summary 中，却来自不同的底层 Grain。**

具体数据血缘：

```text
fact_dex_pool_swap
       │
       ├── Daily Swap Count
       ├── Daily Gross Volume
       ├── Estimated Gross Swap Fee
       └── Transaction Sender Activity

pool_state_change_history
       │
       ▼
daily_pool_snapshot
       │
       ├── End-of-Day Reserve
       ├── End-of-Day Pool Spot Price
       └── End-of-Day TVL
                ▲
                │
       Token USD Reference Prices
```

对 `unique_tx_senders`，还需要关联 Transaction 的发送方。

这里的 Daily Summary 本质上是一个**已定义好口径、按日合并的 Analytics Product**。

它不是链上天然存在的一张表。

---

## 八、真正容易出错的是 Query Semantics

产品经理看到报告后问：

> Pool A 最近 7 天的平均 Liquidity 是多少？

此时你不能直接把过去 7 天的 `end_of_day_tvl_usd` 加起来，并把总和叫作 Liquidity。

因为那是七个时点的估值。

若业务要求“七个日末 TVL 的平均值”，则计算：

```text
AVG(end_of_day_tvl_usd)
```

如果业务要求“七天内的时间加权平均 TVL”，就需要更细粒度的状态与估值历史，仅有日末 Snapshot 未必足够。

再假设产品经理问：

> 最近 7 天的 Total Trading Volume 呢？

这时可以对符合相同指标口径的 Daily Pool Volume 求和：

```text
SUM(pool_gross_volume_usd)
```

因为 Period Metric 与 Point-in-time Metric 的聚合规则不同。

简单来说：

```text
Volume:
Can usually be summed across time.

End-of-Day TVL:
Cannot be summed across dates as if it were volume.

Unique Traders:
Cannot sum daily distinct counts to obtain weekly distinct traders.
```

最后一项尤其重要。

如果 Alice 连续七天都有交易：

```text
Daily Unique Tx Senders:
Alice counts once per day.
```

将每天的 Unique Count 直接相加，会把 Alice 统计七次。

要计算一周真正不同的 Transaction Sender 地址数，需要在周级时间范围重新做去重，或采用符合精度要求的可合并去重结构。

这些问题不是 SQL 语法错误，而是 **Aggregation Semantics（聚合语义）** 错误。

---

## 九、Reorg 和 Data Quality 会影响所有指标

我们此前学习过 Reorg（Chain Reorganization，区块链重组）。

假设前一天某个 Swap Event 随旧分支被撤销：

```text
Old Canonical Branch
   Swap Event A
         ↓
   Daily Volume includes A
```

发生 Reorg 后：

```text
Swap Event A → Non-canonical
```

那么：

- Daily Swap Count 应重新计算。
- Daily Volume 应排除失效事件并加入新主链事件。
- Fee Estimate 可能需要修正。
- Unique Transaction Senders 可能变化。
- End-of-Day Pool State、TVL 和 Price 也可能因状态重算而变化。

因此：

**Daily Analytics 不是写入一次就永远不变。**

它必须遵循之前学过的 Canonical Data Correctness、Checkpoint、Backfill 与 Reconciliation（数据核对）原则。

这也是 Blockchain Analytics 与传统静态历史报表的重要差异之一。

---

## 十、本课核心结论

本课并不是让你记住五个指标的名字。

它要建立一种分析思路：

> Start with the business question, identify the natural grain, select trustworthy source facts, define the metric, and validate the result.

对于同一个 Pool：

- **Volume** 主要来自交易执行事实，是期间指标。
- **Liquidity / TVL** 主要来自 Pool State 和估值，是时点指标。
- **Price** 需要明确 Spot、Execution 或其他计算定义。
- **Fee** 必须区分 Gross Fee、Protocol Fee 与 LP Earnings。
- **Wallet Behavior** 必须先定义地址角色和归因方式。

五种指标共同构成同一份 Pool Analytics Report，但**不能因为它们展示在同一个 Dashboard，就认为它们有相同的原始数据粒度和计算方法**。

---

# 理解检查：继续使用 Pool A

## 问题一：Volume、Liquidity 与 Price

Pool A 当天数据：

```text
Daily Gross Volume = $8,000

End-of-Day:
120,000 USDC
40 WETH

Reference:
1 USDC = $1
1 WETH = $3,000
```

请回答：

1. Pool A 的 End-of-Day TVL 是多少？需要哪些数据才能计算？
2. Pool Spot Price（以 USDC/WETH 表示）是多少？
3. 为什么不能把 Daily Volume 直接当作 Pool Liquidity？
4. 如果要判断“用 10,000 USDC 买 WETH 会有多大的 Price Impact”，只知道 TVL 是否足够？

## 问题二：Fee 与 Wallet Behavior

同一天三次 Swap：

```text
Swap 1: $3,000, tx.from = Alice
Swap 2: $2,000, tx.from = Bob
Swap 3: $3,000, tx.from = Alice
```

假设本例使用 0.30% Swap Fee，且交易名义额与计费输入的美元估值近似一致。

请回答：

1. Estimated Gross Swap Fee 大约是多少美元？
2. 能否直接把这个结果命名为 `lp_profit_usd`？为什么？
3. `unique_tx_senders` 是多少？
4. 为什么这个值不一定等于 Unique Human Traders？

## 问题三：Analytics Aggregation

某产品经理提出：

> 最近七天每天的 TVL 分别是一个数字。我想计算七天的 Liquidity，就直接 SUM(TVL)；最近七天的 Unique Traders 也直接 SUM 每天的 Unique Traders。

请回答：

1. 这个计算方案有哪些问题？
2. 对于最近七天的 Total Trading Volume，能否 SUM 每天的 `pool_gross_volume_usd`？需要什么前提？
3. 发生 Reorg 后，Daily Pool Analytics 应怎样处理？

---

本课正式内容到这里，进入理解检查阶段。可以从问题一开始逐题提交，我们继续按照课程规则逐轮校准，不提前结课。


## 用户回答（理解检查｜问题一：Volume、Liquidity 与 Price）

问题一：

1. End-of-Day TVL 计算： 120000 usdc + 40\*3000  usdc &#x20;
2. 1 WETH = $3,000USDC&#x20;
3. **Volume** 主要来自交易执行事实，是期间指标。**Liquidity / TVL** 主要来自 Pool State 和估值，是时点指标。
4. 对于 Uniswap v2 , 有了 TVL 可以分析Price Impact ， 但是Uniswap v3 ，需要在明确的 Pool State 下，对指定交易规模计算或模拟报价

## 老师判断 / 校准（问题一）

**判定：第 1、2、3 题通过；第 4 题方向正确，但需要修正一个关键概念。**

### 1. End-of-Day TVL：正确

你的计算方法正确：

\[
120{,}000\times1+40\times3{,}000=\$240{,}000
\]

计算 TVL 需要两个方面的数据：

- **Pool State**：USDC 和 WETH 的日末 Reserve。
- **Reference Prices**：这两种 Token 在对应估值时间的美元参考价格。

因此，Pool A 的 End-of-Day TVL 为 **$240,000**。

### 2. Pool Spot Price：正确

\[
\frac{120{,}000\ \text{USDC}}{40\ \text{WETH}}
=3{,}000\ \text{USDC/WETH}
\]

也就是 1 WETH = 3,000 USDC。

注意：这里是由 Pool Reserve 推导的 **Pool Spot Price**。它恰好与题目给定的参考美元价格数值一致，但两者并不是同一个数据概念。

### 3. Volume 与 Liquidity 的区别：正确

你对 Period Metric 和 Point-in-time Metric 的区分准确。

- **Daily Trading Volume**：一天中累计发生多少交易，是期间指标。
- **End-of-Day TVL**：某个时点 Pool 中资产的美元估值，是时点指标。

即使当天 Volume 很高，Pool 在日末持有的资产价值也可能较低，反之亦然。

### 4. TVL 是否足以计算 Price Impact：需要修正

你认为 Uniswap v2 有了 TVL 就可以分析 Price Impact，而 v3 需要更详细的 Pool State。

**这里的问题是：Uniswap v2 仅凭 TVL，同样无法可靠计算 Price Impact。**

例如：

```text
Pool A TVL = $240,000
Pool B TVL = $240,000
```

即使两者 TVL 相同，它们的 Token 组成、Reserve 比例及对应资产价格也可能不同。

对于 v2，计算指定方向和金额的 Swap，需要知道：

```text
reserve_in
reserve_out
amount_in
swap_fee
```

v2 的 Constant Product AMM（恒定乘积自动做市商）根据这些数据决定输出金额与 Price Impact。

对于 v3，则还需要考虑当前价格、Active Liquidity，以及订单执行过程中可能跨越的 Tick / Liquidity Range。

所以正确区别是：

**v2 与 v3 都不能只靠 TVL 计算 Price Impact；只是 v3 的流动性分布模型更加复杂。**

### 补充确认

假设两个 Uniswap v2 Pool：

```text
Pool A:
100,000 USDC + 50 WETH

Pool B:
200,000 USDC + 25 WETH
```

两者的 TVL 都按相同的独立美元参考价格估值，并且都为 $300,000。

现在用户想用 **10,000 USDC 买入 WETH**。

请判断：**两个 Pool 的 Price Impact 一定相同吗？为什么？**

你只需要回答这个问题即可。通过后，问题一就正式完成。


## 用户回答（理解检查｜问题一补充确认）

不相同， 甚至两个pool的当前weth的兑换价格都不同

## 老师判断 / 校准（问题一最终确认）

**回答正确。第 9 课问题一的补充确认通过，问题一正式完成。**

你注意到了一个重要现象：两个 Pool 即使 TVL 相同，Pool Spot Price 也可能不同。

根据题目中的 Reserve：

| | Pool A | Pool B |
|---|---:|---:|
| USDC Reserve | 100,000 | 200,000 |
| WETH Reserve | 50 | 25 |
| Spot Price | 2,000 USDC/WETH | 8,000 USDC/WETH |
| 10,000 USDC / USDC Reserve | 10% | 5% |

但这里还需要区分两个判断：

**第一，Spot Price 不同。** 这是因为两个 Pool 的资产储备比例不同。

**第二，Price Impact 不同。** 更关键的原因是，同样输入 10,000 USDC，相对于 Pool A 的 USDC Reserve 占 10%，相对于 Pool B 只占 5%。在相同手续费规则下，Pool A 的相对 Price Impact 更大。

因此，需要补充一点：**Spot Price 不同本身并不是证明相对 Price Impact 不同的充分理由；还要检查交易规模与 Reserve 的关系。**

这正是本题要建立的工程认识：

> TVL describes the value of assets in a pool, but it does not fully describe execution liquidity.

对于 Uniswap v2，要结合具体 Reserve、交易方向、输入金额和 Fee 计算 Price Impact；对于 v3，还需要考虑 Active Liquidity 和 Tick 分布。

**结论：理解检查问题一通过；问题二、三待回答，课程尚未结课。**


## 用户回答（理解检查｜问题二：Fee 与 Wallet Behavior）

问题二：

1. 8000$ \* 0.3%
2. &#x20;Gross Swap Fee != lp_profit_usd
3. 2
4. **Unique Wallet Addresses 不等于 Unique Human Users。**

## 老师判断 / 校准（问题二）

**判定：四个回答的方向都正确。第 1、3 题通过；第 2、4 题还需要补充原因。**

### 1. Estimated Gross Swap Fee：正确

你的计算式正确：

\[
\$8,000 \times 0.3\% = \$24
\]

在本题的简化估值假设下，当天的 **Estimated Gross Swap Fee 约为 $24**。

### 2. Gross Swap Fee 是否等于 LP Profit？

你回答：

`Gross Swap Fee != lp_profit_usd`

**判断正确，但还需要解释为什么。**

Gross Swap Fee 是交易活动产生的总手续费估值。

而某个 LP 的收益需要考虑其流动性份额、可能适用的 Protocol Fee、手续费归属及实现情况。如果进一步计算净利润，还需要考虑价格变化造成的损益和其他成本。

因此，不能把整个 Pool 的 $24 直接当作某个 LP 的利润。

### 3. Unique Transaction Senders：正确

三次 Swap 的发送方分别是 Alice、Bob、Alice。

因此：

```text
unique_tx_senders = 2
```

### 4. 为什么不等于 Unique Human Traders？

你回答：

> Unique Wallet Addresses 不等于 Unique Human Users。

**结论正确，但还缺少背后的原因。**

例如，一个人可以控制多个钱包地址；某个交易地址也可能是 Relayer 或其他服务的入口，而不直接代表最终经济交易者。

我们只能确定本例有两个不同的 Transaction Sender 地址，不能仅凭这一点确定有两个真实的人类用户。

### 补充确认

请简单补充两个原因即可：

1. 为什么整个 Pool 产生了 $24 的 Gross Fee，不代表某个 LP 就赚了 $24？
2. 为什么 Alice、Bob 两个不同的交易发送地址，不一定对应两个不同的真实用户？

回答后就可以完成问题二的校准。


## 学员决定（问题二校准及问题三）

这里跳过吧， 然后第三题你来回答一下

## 教师回应：问题二跳过与问题三示范解答

好的。**问题二剩余的补充校准按你的要求跳过**，不再追问。记录为“学员主动跳过”，不视为考核通过。

下面由我示范回答第 9 课的第三题。

## 问题三：Analytics Aggregation

### 1. 为什么不能 SUM 最近七天的 TVL 和 Unique Traders？

这两个指标的问题不同。

**第一，TVL 是 Point-in-time Metric（时点指标）。**

假设 Pool A 连续七天的日末 TVL 都是 $240,000。

```text
Day 1: $240,000
Day 2: $240,000
...
Day 7: $240,000
```

直接 `SUM(TVL)` 得到 $1,680,000，并不能说明这个 Pool 拥有 $1,680,000 的流动性。

如果业务要求的是“七个日末 TVL 的平均值”，可以使用：

```sql
SELECT AVG(end_of_day_tvl_usd)
FROM dws_dex_pool_daily
WHERE pool_address = :pool_address
  AND trade_date_utc BETWEEN :start_date AND :end_date;
```

如果要求真正的七天 Time-weighted Average TVL（时间加权平均 TVL），则可能需要更细粒度的状态及估值历史。

**第二，Unique Traders 是去重指标，不能跨日期简单相加。**

假设：

```text
Monday:    Alice, Bob       → 2
Tuesday:   Alice, Charlie   → 2
Wednesday: Alice, Bob       → 2
```

每日人数之和是 6，但三天实际只有三个不同地址：

```text
Alice
Bob
Charlie
```

因此，周级 Unique Traders 应在整个七天范围内重新去重，并且保持相同的 Trader Attribution 规则。

### 2. 最近七天的 Total Trading Volume 能否 SUM 每日 Volume？

**可以，但有前提。**

因为 Trading Volume 是 Period Metric（期间指标），其金额通常可以跨不重叠的时间段累加。

```sql
SELECT
    chain_id,
    pool_address,
    SUM(pool_gross_volume_usd) AS volume_7d_usd
FROM dws_dex_pool_daily
WHERE trade_date_utc BETWEEN :start_date AND :end_date
GROUP BY chain_id, pool_address;
```

必须保证数据符合相同的统计口径，包括：

- 同一个 Chain、Pool 及 Pool Gross Volume 定义。
- 每天的 UTC 日期区间没有重复或遗漏。
- USD Valuation Method 一致，金额没有重复计算。
- 使用有效的 Canonical 数据，且所有日期的数据已达到规定的完整性要求。

这里求出的是 **7-day Pool Gross Volume**，不能直接当作 7-day User Routed Volume。

### 3. 如果发生 Reorg，Daily Pool Analytics 应怎样处理？

需要执行：

```text
Detect Reorg
      ↓
Find Common Ancestor
      ↓
Invalidate Old Branch Facts
      ↓
Replay New Canonical Branch
      ↓
Recompute Affected Daily Metrics
      ↓
Reconcile and Publish Corrections
```

例如旧分支包含一笔 $3,000 的 Swap，而新 Canonical Branch 中没有这笔交易。

那么受到影响的指标可能包括：

| 指标 | 修正方式 |
|---|---|
| Pool Swap Count | 移除旧事件，统计新分支事件 |
| Pool Gross Volume | 重新汇总有效 Swap |
| Gross Swap Fee | 根据新交易集合重新估算 |
| Unique Transaction Senders | 对受影响日期重新去重 |
| End-of-Day TVL | 根据修正后的 Pool State 和估值重算 |
| End-of-Day Price | 根据修正后的日末有效 State 重算 |

尤其要注意：Unique Traders 不能简单通过 `count - 1` 修正，因为被撤销的地址可能仍有其他有效交易。

另外，如果 Reorg 改变了某个 Pool 的日末状态，依赖该状态的后续 Daily Snapshot 也可能需要修正，直到状态重新收敛或完成必要的历史重建。

**本题核心结论：**

> Aggregation rules depend on metric semantics, not merely on database column types.

即：一个指标能否 `SUM`、`AVG` 或 `COUNT(DISTINCT)`，取决于它的业务定义、Grain 和时间语义，而不是因为它恰好是数据库里的数值字段。

---

第 9 课理解检查目前的记录是：**问题一通过；问题二剩余校准主动跳过；问题三由教师示范解答。** 不将第三题记录为学员独立答题通过。
