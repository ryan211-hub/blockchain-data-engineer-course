# Module 13 — DEX

## Module Contract

**第四阶段现行标准（2026-10-13）**：Module 13 以 DEX 业务理解、协议机制、参与者利益与风险、链上数据语义为主要目标；下方原偏工程的考核条目只作为历史参考。第 10 课是 3～5 道不重复既有 Lesson 题目的轻量 Business Understanding Review；Pipeline / Indexer 工程设计推迟到第五阶段。详见 Notion「AI提示词」第 14 条。

### Module 目标

进入第四阶段 Protocol，从“区块链数据如何被采集、处理、存储”进一步进入：

> What does protocol data actually mean?

Module 13 聚焦 DEX（Decentralized Exchange，去中心化交易所），目标不是背协议名词，而是建立能够支持链上数据建模、指标设计、数据质量判断和产品分析的 DEX Business Semantics。

### 必须掌握

- DEX、CEX（Centralized Exchange，中心化交易所）、Order Book 与 AMM（Automated Market Maker，自动做市商）的核心区别。
- Liquidity Pool、Swap、LP（Liquidity Provider，流动性提供者）、Fee 的业务关系。
- Constant Product AMM 的基本定价逻辑与 `x * y = k`。
- Price Impact、Slippage、Execution Price 的区别。
- Liquidity / Reserve 如何影响 Swap。
- LP 为什么提供流动性，以及 Fee 如何形成 LP 收益。
- Impermanent Loss 的基本机制与数据含义。
- Uniswap v2 / v3 在数据模型上的关键差异。
- Concentrated Liquidity、Tick、Position 的基础语义。
- Router、Multi-hop Swap 与 Aggregator 的基础数据还原。
- 如何从 Transaction / Log / Transfer / Pool State 还原可信 Swap Fact。
- 如何设计 DEX 核心 Fact / Dimension / DWS / ADS。
- 如何设计 Volume、Liquidity、Price、Fee、Trader / Pool Analytics。

### 可以了解

- TWAP（Time-Weighted Average Price，时间加权平均价格）。
- Oracle 与 DEX Price 的关系。
- MEV（Maximal Extractable Value，最大可提取价值）的基础概念。
- Arbitrage（套利）如何帮助 AMM Price 与外部市场收敛。
- Stable-swap / Curve 类 AMM 的高层差异。
- DEX Aggregator 的 Routing 逻辑。

### 本 Module 不展开

- AMM 高级数学证明。
- MEV 搜索器与 Builder 的实现细节。
- 高频量化交易策略。
- Uniswap v4 Hook 内核实现。
- 完整 DeFi 风险模型。
- Lending / Liquidation；留到 Module 14。

### 结束标准

完成本 Module 后，应能回答：

1. 为什么 AMM 会出现，它与 Order Book 的核心差异是什么？
2. Liquidity Pool 如何决定 Swap Execution？
3. `x * y = k` 如何解释价格与 Price Impact？
4. Slippage、Price Impact、Execution Price 有什么区别？
5. LP 如何赚钱，承担什么风险？
6. Uniswap v2 / v3 的数据对象为什么不同？
7. 如何识别并还原一笔 Single-hop / Multi-hop Swap？
8. 如何设计可信的 DEX Swap Fact 与核心 Analytics Metrics？
9. 如果给你 Raw Logs，如何构建一个 DEX Analytics Pipeline？

## Module 13 结束判定（第四阶段新标准）

- **状态：业务理解已完成，工程细节后置。**
- 第 1～9 课已完成，跳过和教师示范部分均按历史记录保留。
- 第 10 课共 4 道场景题：Token Launch、Arbitrage 两题学员独立通过；Trading Incentives 由教师示范，不记独立通过或失败；Slippage 校准后完成、Sandwich Attack 由教师讲解，不记整题独立通过。
- 结课依据：Notion「AI提示词」第 14 条，本阶段重业务认知与数据语义，不以完整 DEX Pipeline 或全部问题独立通过作为强制完成标准。
- 下一步：Module 14 — Lending（待开始）。工程实践后置第五阶段。

## 课程结构

1. 第 1 课｜为什么会有 DEX：从 Order Book 到 AMM
2. 第 2 课｜AMM 的定价逻辑：从 `x * y = k` 理解 Pool Price
3. 第 3 课｜Execution Price、Price Impact 与 Slippage
4. 第 4 课｜Liquidity Provider：Liquidity、Fee 与 Impermanent Loss
5. 第 5 课｜Uniswap v2：Pool / Reserve / Swap 的完整数据语义
6. 第 6 课｜Uniswap v3：Concentrated Liquidity、Tick 与 Position
7. 第 7 课｜Router、Multi-hop Swap 与 Aggregator：一笔交易为什么有多个 Swap
8. 第 8 课｜DEX Data Modeling：Swap Fact、Pool Dimension 与核心指标
9. 第 9 课｜DEX Analytics：Volume、Liquidity、Price、Fee 与 Wallet Behavior
10. 结束标准综合检查｜DEX Business Understanding Review：业务场景与核心机制

## 当前学习进度

- 当前阶段：第四阶段 Protocol
- 当前 Module：Module 13 — DEX
- 已完成 Lesson：第 1 课｜为什么会有 DEX：从 Order Book 到 AMM；第 2 课｜AMM 的定价逻辑：从 `x * y = k` 理解 Pool Price；第 3 课｜Execution Price、Price Impact 与 Slippage；第 4 课｜Liquidity Provider：Liquidity、Fee 与 Impermanent Loss；第 5 课｜Uniswap v2：Pool / Reserve / Swap 的完整数据语义；第 6 课｜Uniswap v3：Concentrated Liquidity、Tick 与 Position；第 7 课｜Router、Multi-hop Swap 与 Aggregator：一笔交易为什么有多个 Swap；第 8 课｜DEX Data Modeling：从一个真实查询需求推导数据模型（问题驱动重学版）；第 9 课｜DEX Analytics：Volume、Liquidity、Price、Fee 与 Wallet Behavior（业务理解新标准结课；剩余校准跳过、教师示范如实记录）；第 10 课｜DEX Business Understanding Review：业务场景与核心机制（综合检查已处理，2 题独立通过，2 题含教师讲解；按第四阶段新标准结课）
- 当前 Lesson：第 10 课｜DEX Business Understanding Review（已结束）
- 当前状态：Module 13 — DEX 已正式结课，按“业务理解已完成，工程细节后置”记录。第10课问题一、二独立通过，问题三教师示范，问题四部分校准与教师讲解，均按实际过程记录。
- 下一步准确入口：Module 14 — Lending（待开始），按第四阶段业务理解优先规则教学。

## 课程路径

- [第 1 课｜为什么会有 DEX：从 Order Book 到 AMM](lesson-01-why-dex-order-book-to-amm.md)
- [第 2 课｜AMM 的定价逻辑：从 `x * y = k` 理解 Pool Price](lesson-02-amm-pricing-constant-product-pool-price.md)
- [第 3 课｜Execution Price、Price Impact 与 Slippage](lesson-03-execution-price-price-impact-slippage.md)
- [第 4 课｜Liquidity Provider：Liquidity、Fee 与 Impermanent Loss](lesson-04-liquidity-provider-fee-impermanent-loss.md)
- [第 5 课｜Uniswap v2：Pool / Reserve / Swap 的完整数据语义](lesson-05-uniswap-v2-pool-reserve-swap.md)
- [第 6 课｜Uniswap v3：Concentrated Liquidity、Tick 与 Position](lesson-06-uniswap-v3-concentrated-liquidity-tick-position.md)
- [第 7 课｜Router、Multi-hop Swap 与 Aggregator：一笔交易为什么有多个 Swap](lesson-07-router-multi-hop-swap-aggregator.md)
- [第 8 课｜DEX Data Modeling：Swap Fact、Pool Dimension 与核心指标](lesson-08-dex-data-modeling-swap-fact-pool-dimension.md)
- [第 8 课（问题驱动重学版）｜DEX Data Modeling：从一个真实查询需求推导数据模型](lesson-08-problem-driven-relearn.md)
- [第 9 课｜DEX Analytics：Volume、Liquidity、Price、Fee 与 Wallet Behavior](lesson-09-dex-analytics-volume-liquidity-price-fee-wallet-behavior.md)
- [第 10 课｜DEX Business Understanding Review：业务场景与核心机制](lesson-10-dex-business-understanding-review.md)
