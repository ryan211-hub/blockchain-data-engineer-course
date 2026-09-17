# Module 9 结束标准综合检查

## 综合检查结论

Module 9 — ETL 结束标准综合检查通过。

用户已经能够针对 `fact_token_transfers → dws_wallet_token_daily_flow → ads_wallet_overview` 设计完整的 Blockchain Batch ETL Pipeline，并正确说明 Source、Target、Processing Range、Grain、Unique Key、Load Strategy、Watermark / Cursor / Checkpoint、Idempotency、Validation、Late-arriving / Reorg、Backfill / Repair、DAG Dependency，以及传统 ETL 与 Blockchain ETL 的共同点和差异。

## 第一部分｜Source → Transform → Target

### 用户回答要点
- Source = `fact_token_transfers`
- Target = `dws_wallet_token_daily_flow`
- 处理 `2026-09-15` 时，Input Range = `[2026-09-15, 2026-09-16)`
- Target Grain = one row per day per wallet per token
- Unique Key = `chain_id + date + wallet_address + token_address`
- Load Strategy = Delete + Insert / Replace 当天 Partition；不为“第一次上线”设计特殊逻辑，而是始终复用同一套可重跑逻辑。

### 教师校准
一条 `Alice → Bob 100 USDC` 的 Transfer 在 wallet-token-day Grain 下应拆成 Alice 的 sent/net 和 Bob 的 received/net 两侧贡献，再按 `chain_id + date + wallet_address + token_address` 聚合。

## 第二部分｜Watermark / Cursor / Checkpoint

### 用户回答要点
- Job 在处理 09-15 过程中崩溃时，Cursor 可以指向 `2026-09-15`
- Checkpoint 仍停在 `2026-09-14`
- 重启后重新处理 `2026-09-15`
- Delete + Insert / Replace Partition 使整天重跑仍保持 Idempotency
- Watermark 定位可处理数据边界；Cursor 定位当前正在处理的位置；Checkpoint 代表已经安全完成的位置。

### 教师结论
本部分通过。

## 第三部分｜Validation + Checkpoint Advance

### 用户回答要点
- Execution Success 但 Aggregate Reconciliation 不通过时，Job 不能认定完成
- Checkpoint 不能推进
- 程序正常结束不等于数据正确
- Source / Target Grain 不一致时，不能简单用 Row Count 相等作为正确性判断
- 当前场景应使用 Aggregate Reconciliation 等符合业务语义的 Validation

### 教师结论
本部分通过。

## 第四部分｜Late-arriving / Reorg / Historical Repair

### 用户回答要点
- Provider 延迟导致 09-15 Transfer 在 09-17 才进入 Fact，属于 Late-arriving Data
- canonical block 变化属于 Reorg
- Checkpoint 表示当时可见数据条件下的安全完成点，不代表历史数据以后永远不会变化
- 对可预期 Late-arriving 优先使用 Rolling Lookback 自动吸收
- 超出 Lookback Window 的历史变化使用 Backfill / Repair；正常 ETL Checkpoint 通常不回退

### 教师校准
Late-arriving = Source Truth 没变，只是数据晚到；Reorg = canonical Source Truth 本身后来改变。`Source Changed Range != Downstream Repair Range`，真正 Repair Range 还取决于 DAG / Lineage、Processing Unit 与跨时间依赖。

## 第五部分｜DAG / Workflow Control / Blockchain ETL

### 用户回答要点
- `DWS(2026-09-15)` 应依赖对应的 `Fact(2026-09-15)` Processing Unit
- Fact / DWS / ADS Checkpoint 不一致不代表 Pipeline 错误，只表示各 Job Processing State 不同
- 上游 Fact 失败时，DWS 应是 BLOCKED / WAIT / UPSTREAM_FAILED，而不是自身 FAILED
- Scheduler / Orchestrator 负责 Workflow Control：何时调用、前置条件、成功失败后的下一步
- ETL Job 负责自身 Extract → Transform → Load 等内部数据处理逻辑
- 传统 ETL 与 Blockchain ETL 共享 Source → Extract → Transform → Load → Validation → Checkpoint → Retry / Recovery → DAG / Scheduler 结构
- Blockchain ETL 额外必须处理 Reorg，以及“已经处理过的历史事实后来可能发生变化”的特性

### 最终校准
用户最终回答：

> 这正是 Blockchain ETL 的特点，Blockchain ETL 必须接受“已经处理过的历史事实后来可能发生变化”。

该回答正确。Checkpoint 是基于当时可见且已验证数据的安全完成点；Reorg 后需要 Historical Repair，并不反向证明当时 Checkpoint 错误。

## 最终判定

Module 9 第 1～7 课与结束标准综合检查均已完成，达到 Module Contract 结束标准。

用户已经能够独立设计基础 Blockchain Batch ETL Pipeline，并解释其增量边界、状态管理、幂等、验证、失败恢复、历史修复和 DAG 调度逻辑。