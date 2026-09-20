# Module 10 — 实时数据

## Module Contract

### Module 目标

把 Module 9 已经掌握的 Batch ETL 心智模型推进到 Streaming / Real-time Data Pipeline：理解为什么实时系统不是“把 ETL 跑得更频繁”，并能够从 Blockchain Data Engineer 视角设计一条可恢复、可重放、可校准的链上实时数据管道。

### 必须掌握

- Batch 与 Streaming 的根本差异：有限数据集 vs 持续到达的数据流。
- Event-driven / Message-driven Pipeline 的基本结构。
- Kafka 在实时数据平台中的角色：Producer、Topic、Partition、Consumer、Consumer Group、Offset。
- Ordering、Partition Key 与并行消费之间的关系。
- At-most-once、At-least-once、Exactly-once 的含义及工程边界。
- Consumer Offset / Checkpoint 与数据处理状态之间的关系。
- Idempotency、Retry、Replay、Backpressure 与故障恢复。
- Blockchain 实时数据的特殊约束：WebSocket / RPC 不是可靠消息队列；Reorg、Finality、Late-arriving Data、Reconciliation 仍然存在。
- Lambda / Kappa / Batch + Streaming 混合架构的基本取舍，不追求流派名词记忆，而能解释实际数据流。
- 能够设计一条 Wallet / Transfer / Swap 实时处理 Pipeline，并明确 Source、Message Grain、Partition Key、Consumer、Sink、Offset、幂等键和修复路径。

### 可以了解

- Kafka Broker、Replication、ISR、Retention、Compaction 的基本概念。
- CDC、Flink、Spark Structured Streaming 等技术在整体架构中的位置。
- Real-time Analytics 与 Operational Real-time 的差异。

### 本 Module 不展开

- Kafka Broker 内部存储实现、Controller / KRaft 协议细节。
- Flink Runtime、Checkpoint Barrier、State Backend 等框架内核实现。
- 分布式一致性算法的证明与实现。
- DEX / Lending 等协议业务细节；留到后续 Protocol Module。
- ClickHouse / DuckDB / Postgres 的系统性选型；留到 Module 11。

### 结束标准

完成本 Module 后，应能独立设计并解释：

```text
Ethereum / EVM Node or Provider
        ↓
Indexer / Event Producer
        ↓
Kafka Topic
        ↓
Consumer Group
        ↓
Realtime Transform / Enrichment
        ↓
Serving DB / Warehouse / Alert / API
```

并能回答：

1. 为什么实时处理不能简单理解为“每 1 秒跑一次 ETL”？
2. Kafka 为什么需要 Partition、Offset 和 Consumer Group？
3. 如何在并行吞吐与事件顺序之间做设计？
4. Consumer 在写库成功前后分别提交 Offset，会产生什么故障语义？
5. 为什么 At-least-once 通常要求 Sink 幂等？
6. 链上发生 Reorg 时，实时 Pipeline 如何修正已经下发的旧 canonical 数据？
7. 为什么 Streaming 系统仍然需要 Batch Backfill / Reconciliation？

## 课程结构

1. 第 1 课｜实时数据到底是什么：为什么不是“把 ETL 跑快一点”
2. 第 2 课｜Kafka 在实时 Pipeline 中到底解决什么问题
3. 第 3 课｜Topic、Partition、Offset：数据流如何被切分和定位
4. 第 4 课｜Consumer Group 与 Ordering：如何同时获得并行处理和局部有序
5. 第 5 课｜Delivery Semantics：At-most-once、At-least-once、Exactly-once
6. 第 6 课｜Retry、Replay、Backpressure 与实时故障恢复
7. 第 7 课｜Blockchain Streaming：Reorg、Finality、Reconciliation 与 Batch + Stream
8. 结束标准综合检查｜设计一条完整 Blockchain Realtime Data Pipeline

## 当前学习进度

- 当前阶段：第三阶段 Data Engineering
- 当前 Module：Module 10 — 实时数据
- 已完成 Lesson：第 1 课｜实时数据到底是什么：为什么不是“把 ETL 跑快一点”；第 2 课｜Kafka 在实时 Pipeline 中到底解决什么问题；第 3 课｜Topic、Partition、Offset：数据流如何被切分和定位；第 4 课｜Consumer Group 与 Ordering：如何同时获得并行处理和局部有序；第 5 课｜Delivery Semantics：At-most-once、At-least-once、Exactly-once；第 6 课｜Retry、Replay、Backpressure 与实时故障恢复
- 当前 Lesson：第 7 课｜Blockchain Streaming：Reorg、Finality、Reconciliation 与 Batch + Stream
- 当前状态：第 7 课开始
- 下一步准确入口：开始 Module 10 第 7 课｜Blockchain Streaming：Reorg、Finality、Reconciliation 与 Batch + Stream。
