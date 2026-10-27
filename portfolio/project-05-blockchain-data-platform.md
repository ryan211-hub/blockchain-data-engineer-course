## Objective

独立设计并开发可展示在求职作品集中的 EVM 链上数据平台，形成端到端可运行证据，而不只是架构演示。

## Target Architecture

```txt
Ethereum RPC / Node
       ↓
Indexer & Decoder
       ↓
Kafka (streaming optional in MVP)
       ↓
Postgres (operational facts)
       ↓
ClickHouse (analytics marts)
       ↓
API → Dashboard
```

## Scope and Phases

MVP：选择 ERC-20 Transfer 或 Uniswap v2 Swap 一种可控业务范围；完成历史 Backfill、增量同步、SQL 查询 API、最小 Dashboard。

扩展：Kafka、ClickHouse、Reorg Auto Repair、Metrics / Alerts、可重复部署和性能基准。

## Data Contract

事实表必须定义 Chain ID、Block Number / Hash、Tx Hash、Log Index、Contract Address、Decoded Event、Canonical Flag 与各自 Grain。Checkpoint 提交与数据写入要满足一致性要求；Backfill 与 Realtime 保持进度隔离。

## Deliverables

代码仓库、Architecture Decision Records、Docker Compose 或可重复环境、Schema / Migrations、故障恢复 Runbook、测试报告、性能/成本说明、演示视频或截图及简历项目描述。

## Acceptance Criteria（教材中保留；发布不等于通过）

- 完整链路：源数据到 API/Dashboard 可以运行复现。
- 断点恢复、重复 Replay 与指定范围 Backfill 有证据。
- Reorg 模拟：旧分支数据失效、新分支补入，报表口径正确。
- Raw/Fact/Aggregate 三层数据抽样对账。
- 指标：Processing Lag、RPC Error Rate、Decode Errors、Data Freshness 和成本。
- README 说明限制、风险、操作与面试可讲解的设计权衡。

## Review Questions（无需答题）

1. 为什么 Reorg Repair 应从 Raw Facts 传播到聚合层？
2. 为什么 Postgres 与 ClickHouse 适合承担不同角色？
3. 要证明这是可交付平台而非 Demo，最关键的可重复验证证据是什么？
