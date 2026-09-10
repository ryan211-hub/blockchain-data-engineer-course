# Module 7 — Indexer

## 模块目标

从 Blockchain Data Engineer 视角掌握链上 Indexer 的完整数据路径：从 Raw Block / Transaction / Receipt / Log 获取原始链上事实，经 Parser / Decoder 规范化为可查询的数据模型，并稳定、可恢复地写入数据库；最终完成一个 Mini Indexer。

## Module Contract

### 必须掌握
- Indexer 为什么存在，以及它与 Node / RPC / Warehouse 的职责边界
- Raw Block → Parser → Decoder → Normalized Fact → Database 的完整数据流
- Block、Transaction、Receipt、Log / Event 在 Indexer 中的输入与落库关系
- Checkpoint / Cursor：如何连续推进并从中断位置恢复
- Idempotency：重复处理同一 Block / Log 时如何避免重复写入
- Backfill：如何安全补历史数据，而不是每次从 Genesis 重跑
- Reorg：canonical / orphaned 数据如何识别、回滚或重放
- 数据唯一键与幂等写入的基本设计
- 实时同步与历史 Backfill 如何共存
- 能设计并实现一个最小可用 Mini Indexer

### 可以了解
- 不同 Indexer Framework / Provider 的产品差异
- 并行抓取、批量写入、队列化等性能优化思路
- 多链 Indexer 的统一抽象

### 本 Module 不展开
- 更完整的数据仓库分层与维度建模 → Module 8
- 通用 ETL Framework → Module 9
- Kafka / Streaming 架构 → Module 10
- ClickHouse / DuckDB / Postgres 选型 → Module 11
- 系统性数据质量与大规模 Reorg 修复 → Module 12

### 结束标准

能够从零画出一个可靠 Indexer 的数据流和状态机；能够解释 Checkpoint、Idempotency、Backfill、Reorg 为什么是生产级 Indexer 的必要能力；能够完成一个 Mini Indexer，将链上原始数据稳定解析并写入数据库。

## 当前学习进度

- 当前阶段：第三阶段 Data Engineering
- 当前 Module：Module 7 — Indexer（✅ 已完成）
- 已完成 Lesson：第 1 课、第 2 课、第 3 课、第 4 课、第 5 课、第 6 课、第 7 课、第 8 课、第 9 课、第 10 课
- 当前 Lesson：结课综合实践｜Mini Indexer 集成与验收
- 当前状态：✅ Module 7 已完成
- 最近完成：结课综合实践五项验收全部通过：Normal Sync、Idempotent Replay、Crash Recovery、Backfill、Reorg Recovery。验收由用户在 Codex 中实际完成并明确确认通过。
- 尚未完成：本 Module 无。已达到 Module Contract 结束标准。
- 下一步准确入口：Module 8 第 1 课｜为什么链上数据也需要数据建模：从 Indexer Fact 到可分析模型。

> 本区块是 Notion「课程总目录」主进度记录的同步镜像；若发生冲突，以课程总目录为准。

## 课程索引

1. [第1课｜Indexer 到底是什么：为什么 Node / RPC 之后还需要一层 Indexer](lesson-01-what-is-indexer.md)（✅ 已完成）
2. [第2课｜Indexer 内部数据流：Raw Block → Parser → Decoder → Normalized Fact → Database](lesson-02-indexer-internal-data-flow.md)（✅ 已完成）
3. [第3课｜Indexer 的输入对象：Block / Transaction / Receipt / Log 如何进入数据库](lesson-03-indexer-input-objects.md)（✅ 已完成）
4. [第4课｜Checkpoint / Cursor：Indexer 如何知道自己处理到哪个 Block](lesson-04-checkpoint-cursor.md)（✅ 已完成）
5. [第5课｜Idempotency：Indexer 重复处理同一个 Block 为什么不会重复写数据](lesson-05-idempotency.md)（✅ 已完成）
6. [第6课｜Backfill：Indexer 如何安全补历史数据，而不是每次从 Genesis 重跑](lesson-06-backfill.md)（✅ 已完成）
7. [第7课｜Reorg：Indexer 如何处理 canonical / orphaned 数据与回滚重放](lesson-07-reorg.md)（✅ 已完成）
8. [第8课｜Mini Indexer 设计：把 Checkpoint / Idempotency / Backfill / Reorg 串成一个可运行系统](lesson-08-mini-indexer-design.md)（✅ 已完成）
9. [第9课｜Mini Indexer 实践：用 Python + Ethereum JSON-RPC 实现最小 ERC-20 Transfer Indexer](lesson-09-mini-indexer-practice.md)（✅ 已完成）
10. [第10课｜Mini Indexer 可靠性实践：Crash Recovery / Backfill / Reorg Recovery](lesson-10-mini-indexer-reliability.md)（✅ 已完成）
11. [结课综合实践｜Mini Indexer 集成与验收](module-07-final-mini-indexer-validation.md)（✅ 五项验收通过）
