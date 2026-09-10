# Module 7 结课综合实践｜Mini Indexer 集成与验收

## Practice Contract

【Blockchain Data Engineer 视角】

本实践用于验证 Module 7 的结束标准：把第 9～10 课的 Processing Engine、Realtime Controller、Backfill Cursor 与 Reorg Recovery 整合成一套可运行的 Ethereum ERC-20 Transfer Mini Indexer。

验收项目：

1. Normal Sync
2. Idempotent Replay
3. Crash Recovery
4. Backfill
5. Reorg Recovery

## 验收结果

用户确认：

> Mini Indexer 五项验收已经全部通过，我在codex上已经完成，可以结束 Module 7了

因此本次结课综合实践判定为：**✅ 通过**。

说明：五项实践是在 Codex 中实际完成并由用户明确确认通过；本页只记录课程验收结论，不虚构未在当前聊天中提供的具体运行日志、Block Number、SQL 输出或代码细节。

## Module 7 结束判定

结合前 10 课的理解检查与本次综合实践，Module 7 的 Module Contract 结束标准已满足：

- 能够解释 Indexer 与 Node / RPC / Warehouse 的边界
- 能够说明 `Raw Block → Parser → Decoder → Normalized Fact → Database` 的完整链路
- 能够设计 Checkpoint / Cursor 与 Idempotent Write
- 能够处理 Backfill 的独立进度
- 能够解释并实现 Reorg Detection / Common Ancestor / Invalidate / Rollback / Replay
- 能够让 Realtime / Backfill / Reorg Replay 复用同一个 Processing Engine
- 已完成可运行 Mini Indexer 的五项验收

结论：**Module 7 — Indexer 正式完成。**