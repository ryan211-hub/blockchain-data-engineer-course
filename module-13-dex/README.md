# Module 13 — DEX

## Module Contract

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
10. 结束标准综合检查｜从 Raw Logs 设计 DEX Analytics Pipeline

## 当前学习进度

- 当前阶段：第四阶段 Protocol
- 当前 Module：Module 13 — DEX
- 已完成 Lesson：第 1 课｜为什么会有 DEX：从 Order Book 到 AMM
- 当前 Lesson：第 2 课｜AMM 的定价逻辑：从 `x * y = k` 理解 Pool Price
- 当前状态：第 1 课已完成；下一步进入第 2 课
- 下一步准确入口：开始 Module 13 第 2 课｜AMM 的定价逻辑：从 `x * y = k` 理解 Pool Price。

## 课程路径

- [第 1 课｜为什么会有 DEX：从 Order Book 到 AMM](lesson-01-why-dex-order-book-to-amm.md)