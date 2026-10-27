## Objective

构建对原始 Block/Transaction/Event 数据进行可恢复索引的 Mini Indexer。课程原目录偏向解析 Solana Block、Transaction、Instruction；可保留双链可选实现，优先复用已经完成的 Mini Indexer 项目经验，不把旧验收结果覆盖为新实践。

## Requirements

- 定义 Indexer Scope、区块范围、Normalized Data Model 与 Unique Key。
- Fetch Block / Tx / Event、解析、写入数据库。
- 持久化 Cursor / Checkpoint，处理失败恢复与幂等 Replay。
- 如采用 EVM，需显式处理 Reorg、Canonical Chain 与 rollback；如采用 Solana，考虑 Commitment 等最终性语义。

## Architecture

```txt
RPC → Block Fetcher → Decoder → Facts → Postgres
                           ↕
                   Checkpoint / Replay
```

## Deliverables

Schema、CLI、Config、Runbook、Metrics、Integration Tests、Reorg/Replay 故障演练报告。

## Acceptance Criteria（教材中保留；发布不等于通过）

- Normal Sync；Idempotent Replay；Crash Recovery；Backfill；Reorg Recovery（适用于 EVM）；数据校验。
- 在指定样本范围能证明记录数和唯一键正确。
- 项目文档说明 Backfill Cursor 与 Realtime Checkpoint 独立。

## Review Questions（无需答题）

1. 为什么 Checkpoint 不能在数据库写入之前推进？
2. 如何区分 Reorg Replay 与 Historical Backfill？
3. 为什么 Raw Event 与业务 Fact 的 Grain 不应混淆？
