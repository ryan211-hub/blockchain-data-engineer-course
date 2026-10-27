## Objective

创建类似 Dune 的可复核 On-chain Dashboard，面向一个明确业务问题，例如 Uniswap Pool 活跃度或 NFT Marketplace 成交趋势，而不是随意堆砌指标。

## Requirements

- 定义指标 Grain、数据来源、刷新频率及公式。
- 至少包含日趋势、核心 KPI、实体/协议拆分和一个数据质量提示。
- 分别标记 Transaction Count、Transfer Count、Swap Count 或 Sale Count，避免混淆。
- 提供 SQL 与数据字典，说明 Token Decimals、USD Pricing 和 Reorg 修正口径。

## Architecture

```txt
Facts → SQL Transform → Mart / OLAP
      → Dashboard + Quality Report
```

## Deliverables

SQL 文件、可分享的 Dashboard 或静态复现方案、数据字典、更新策略、对账报告和 README 截图。

## Acceptance Criteria（教材中保留；发布不等于通过）

- 至少一次与独立数据源抽样对账。
- 指标显示具体日期/时间范围与 Chain ID。
- 可以解释当前窗口和历史回填的数据差异。
- 失效或缺失数据不默认为 0。

## Review Questions（无需答题）

1. 为什么 Dashboard 必须公布指标定义？
2. 一个 NFT Transfer 能否计入 NFT Sale？
3. Late-arriving 数据到达后，过去的图表值是否允许变化？
