## Objective

输入 EVM 钱包地址，输出资产分析、行为分析和协议分析。重点将多表 SQL 建模经验迁移至 On-chain Analytics。

## Requirements

- 统一 ETH / ERC-20 Transfer、DEX Swap 与已知协议事件；数据来源清楚。
- Asset Balance / Net Flow / Token Holdings / Protocol Interactions 使用独立业务口径。
- 地址 ≠ Entity；Label 应保留证据强度。
- 处理 Token Decimals、历史定价、缺失 Token Metadata 和垃圾代币。

## Architecture

```txt
EVM Logs + Traces + Price Data
             ↓
Normalized Facts + Dimensions
             ↓
Wallet Daily Aggregates
             ↓
Query API + Reports
```

## Deliverables

SQL Models、指标字典、验证 SQL、示例钱包分析报告、查询性能与成本说明。

## Acceptance Criteria（教材中保留；发布不等于通过）

- 可解释余额快照与流量统计为何不直接相等。
- 对重复 Log、重组区块和 Late-arriving 数据有明确修复方案。
- 随机抽样与链上 Explorer / 可信数据源对账。
- Price Missing 时不伪造 USD Value。

## Review Questions（无需答题）

1. 为什么只统计 ERC-20 Transfer 不能获得完整钱包净值？
2. Wallet Label 的推断如何在报告中标注？
3. DEX Swap 与 Token Transfer 应如何避免重复计算业务成交量？
