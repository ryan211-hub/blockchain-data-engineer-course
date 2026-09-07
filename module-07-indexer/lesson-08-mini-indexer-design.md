# 第8课｜Mini Indexer 设计：把 Checkpoint / Idempotency / Backfill / Reorg 串成一个可运行系统

## Lesson Contract
【Blockchain Data Engineer 视角】
- 所属 Module：Module 7 — Indexer
- 本课核心问题：前 7 课已经分别学了 Raw / Parser / Decoder / Checkpoint / Idempotency / Backfill / Reorg。现在如果真正设计一个最小可用 Ethereum Indexer，它应该由哪些组件组成？数据怎么流？状态怎么推进？异常怎么恢复？
- 学完以后应能：画出 Mini Indexer 完整架构；明确各组件职责；设计最小数据表；设计实时同步主循环；解释 Checkpoint 为什么最后更新；设计幂等写入；设计 Backfill 独立进度；设计最小 Reorg Recovery；把前 7 课串成完整状态机。
- 本课暂不展开：Kafka、分布式 Worker、ClickHouse / Postgres 选型、高并发并行抓取、Kubernetes、完整生产监控系统、Module 8 的完整数仓建模。

## 一、任务边界：Ethereum ERC-20 Transfer Mini Indexer
目标：持续从 Ethereum 获取新 Block，提取 ERC-20 `Transfer` Event，写入数据库，并满足：Crash 可恢复；重复跑 Block 不产生重复数据；历史 Backfill 不影响实时；发生 Reorg 能修正数据库。

## 二、完整 Data Path
```text
Ethereum Node / RPC
↓
Fetcher
↓
Raw Objects
↓
Parser
↓
Block / Transaction / Receipt / Log Objects
↓
Decoder
↓
ERC20 Transfer / Swap / Mint / Burn
↓
Normalizer
↓
Normalized Fact
↓
Idempotent Writer
↓
Database
```
旁边还有 Control Path：Checkpoint、Backfill Cursor、Reorg Handling。

## 三、Data Path 与 Control Path
```text
Data Path:
Block → Parser → Decoder → Fact → Database

Control Path:
Checkpoint → Retry → Backfill → Reorg
```
Parser / Decoder 负责“数据是什么”；Checkpoint / Idempotency / Backfill / Reorg 负责“系统怎样可靠运行”。很多 Demo 只有 Data Path，没有 Control Path，所以不能安全恢复。

## 四、Fetcher
Fetcher 从 RPC 获取链上原始输入，例如 Block、Transaction、Receipt、Log。它不负责判断“这是 Transfer”，业务语义由 Decoder 处理。

## 五、Raw Layer
最小 Demo 可以不保存完整 Raw JSON，但应保留足够 Source Data / Lineage。Raw 是后续 Decoder 修复、新增 Decoder、Backfill、Reprocessing 的基础资产。

## 六、Parser / Decoder / Normalizer
Parser = Structure：把 RPC Raw JSON 解析成 Block / Tx / Receipt / Log 等结构化对象。
Decoder = Meaning：依据 topic0 / ABI 等判断 Transfer / Swap / Mint / Burn，并解析业务字段。
Normalizer = Business Data Model：把不同 Decoder / 协议输出映射到稳定统一的事实模型，例如 token_transfers、dex_swaps，而不是把所有业务事实强行塞进同一张表。

## 七、最小数据表
本课建议 6 张核心表：
```text
blocks
transactions
logs
token_transfers
indexer_checkpoints
backfill_jobs
```

### blocks
字段：chain_id, block_number, block_hash, parent_hash, block_time, canonical。
主身份：`(chain_id, block_hash)`；block_number 是位置。

### transactions
字段：chain_id, tx_hash, block_number, block_hash, transaction_index, from_address, to_address, value, status, canonical。
主键：`(chain_id, tx_hash)`。

### logs
字段：chain_id, tx_hash, log_index, block_number, block_hash, contract_address, topic0, topics, data, canonical。
唯一键：`(chain_id, tx_hash, log_index)`。

### token_transfers
字段：chain_id, tx_hash, log_index, block_number, block_hash, token_address, from_address, to_address, amount_raw, canonical。
唯一键：`(chain_id, tx_hash, log_index)`。

### indexer_checkpoints
字段：chain_id, indexer_name, block_number, block_hash, updated_at。
唯一键：`(chain_id, indexer_name)`。
语义：该 Indexer 已经可靠完成到这个具体 Block。

### backfill_jobs
字段：job_id, chain_id, indexer_name, start_block, end_block, last_processed_block, status。
Backfill Progress 与 Realtime Progress 独立。

## 八、实时同步主循环
```text
Read Checkpoint
↓
next_block = checkpoint + 1
↓
Fetch Block
↓
Check Parent Hash
↓
Parse
↓
Decode
↓
Normalize
↓
Idempotent Write
↓
Update Checkpoint
↓
Repeat
```
伪代码核心：
```python
while True:
    checkpoint = load_checkpoint()
    next_number = checkpoint.block_number + 1
    block = fetch_block(next_number)
    verify_chain_continuity(checkpoint.block_hash, block.parent_hash)
    parsed = parse(block)
    facts = decode(parsed)
    write_idempotently(parsed, facts)
    save_checkpoint(block.number, block.hash)
```

## 九、为什么 Checkpoint 最后更新
正确顺序：Write Data → Update Checkpoint。
如果先推进 Checkpoint 再写数据，Checkpoint 成功而数据写入失败会造成 Silent Data Loss。反过来，数据写成功但 Checkpoint 未更新最多造成 Retry，而 Retry 可以由 Idempotency 保护。

## 十、At-least-once + Idempotent Write
```text
Checkpoint Failure
↓
Block Replay
↓
Idempotent Writer
↓
No Duplicate Data
```
典型写法：
```sql
INSERT INTO token_transfers (...)
VALUES (...)
ON CONFLICT (chain_id, tx_hash, log_index)
DO NOTHING;
```
第一次插入；重复处理命中 Unique Constraint；最终 One Logical Fact = One Row。
如果是 Decoder Bug 修复等需要覆盖旧派生结果的场景，应使用确定性的 UPSERT，而不是一律 DO NOTHING。

## 十一、Crash Recovery
假设 Block 100 数据已写入，但 Checkpoint 仍是 99 后 Crash。重启后会再次处理 Block 100。只要 Stable Identity + Unique Constraint + Idempotent Writer 正确，最终数据库仍保持正确，这就是 Effectively Once。

## 十二、Backfill
Realtime checkpoint 不回退。单独创建 Backfill Job：start / end / cursor。Backfill Worker 复用同一 Parser / Decoder / Normalizer / Idempotent Writer，只是 Block 来源和进度状态不同。
```text
Realtime: next = realtime_checkpoint + 1
Backfill: next = backfill_cursor + 1, 且 next <= end_block
```

## 十三、Reorg Recovery
如果 Checkpoint=102A，新 Block 103B 的 parent_hash=102B，且 102B != 102A，则进入 Reorg Recovery：
```text
Detect Reorg
↓
Find Common Ancestor
↓
Mark Old Branch canonical=false
↓
Mark Old Facts canonical=false
↓
Move Checkpoint Back
↓
Replay New Canonical Branch
```
例如旧链 99→100A→101A→102A，新链 99→100B→101B→102B→103B，则共同祖先=99；旧分支失效；Checkpoint 回退到 99；再用正常 Processing Engine 处理 100B→103B。

## 十四、One Processing Engine + Multiple Execution Modes
更合理的抽象：
```text
              Execution Controller
     ┌────────────┼────────────┐
     │            │            │
 Realtime      Backfill      Reorg Replay
     │            │            │
     └────────────┼────────────┘
                  ↓
          Processing Engine
                  ↓
   Parser → Decoder → Normalizer
                  ↓
          Idempotent Writer
                  ↓
              Database
```
Realtime / Backfill / Reorg Replay 的差异主要是 Scheduling / Progress Control，不应各自复制一套 Parser / Decoder / Writer。这样可以减少口径漂移、重复开发、维护成本和修 Bug 时三套逻辑不一致的问题。

## 十五、Indexer Application State
【Indexer Application State 视角】
Indexer 至少维护三类状态：Realtime Checkpoint、Backfill Cursor、Canonical Status。这些属于 Indexer Metadata，不是 Ethereum State，也不同于 Block / Tx / Log 这类 Blockchain Data。

## 十六、项目目录与最小技术栈
一个合理目录：
```text
mini-indexer/
├── config.py
├── rpc.py
├── fetcher.py
├── parser.py
├── decoder.py
├── normalizer.py
├── writer.py
├── checkpoint.py
├── backfill.py
├── reorg.py
├── models.py
├── db.py
└── main.py
```
当前阶段优先验证 Correctness + Recoverability，不追求 Massive Scale。Python + Ethereum JSON-RPC + SQLite/Postgres 已足够，不提前引入 Kafka / Spark / Kubernetes。

## 十七、Failure Is Expected
生产系统目标不是 Never Fail，而是 Fail Safely + Recover Correctly。RPC timeout、进程 Crash、DB 断连、机器重启、Rate Limit、Reorg 都是正常工程事件，因此 Checkpoint、Idempotency、Backfill、Reorg 不是异常补丁，而是系统设计本身。

## 十八、Module 7 前 8 课整合
```text
第1课：为什么需要 Indexer
第2课：Raw → Parser → Decoder → Normalizer
第3课：Block / Tx / Receipt / Log
第4课：Checkpoint
第5课：Idempotency
第6课：Backfill
第7课：Reorg
第8课：把所有组件组成一个系统
```

## 十九、本课核心结论
1. Indexer = Data Pipeline + Stateful Controller。
2. Realtime / Backfill / Reorg Replay 应尽量复用同一 Processing Engine。
3. Checkpoint 在数据可靠写入之后推进。
4. At-least-once + Idempotent Write = Effectively Once。
5. Production Indexer 的目标不是 Never Fail，而是 Fail Safely + Recover Correctly。

## 理解检查与校准
### 问题 1
请描述从 Ethereum RPC 到 token_transfers 的核心 Data Path，并说明 Fetcher / Parser / Decoder / Normalizer / Writer 各自职责。

**用户回答：**
> 问题一：
>
> 1. Fetcher：从 RPC 获取 block 的 Raw data 原始数据
> 2. Parser：将 Raw data 解析为结构化的数据（比如 transfer 数据、receive 的数据、log 数据）
> 3. Decoder：将上一步的数据解码，比如从 log 数据里面解析出业务事件（比如 transfer、burn、mint 还是 swap 这类业务事件）
> 4. Normalizer：将上一步的数据统一整合成一个通用的数据模型，将各类业务数据存到同一个表里面
> 5. Writer：将统一的数据表写到数据库里面

**老师判断与校准：✅ 基本通过。**
Fetcher 正确。Parser 中的 `transfer` 已带业务语义，不属于 Parser；Parser 应输出 Block / Transaction / Receipt / Log 等结构化对象。Decoder 才负责从 Log 等对象中识别 Transfer / Mint / Burn / Swap。Normalizer 不是把所有业务数据塞进同一张表，而是把同类业务事实映射到稳定统一模型，例如 Transfer → token_transfers、Swap → dex_swaps。Writer 还应明确是 Idempotent Writer。记忆：`Parser = Structure`，`Decoder = Meaning`，`Normalizer = Business Data Model`。

### 问题 2
Checkpoint=100，Block 101 的 token_transfers 已全部写入，但程序在更新 Checkpoint 前 Crash。重启后应该怎么做？为什么不会重复？请串起 Checkpoint / At-least-once / Idempotency。

**用户回答：**
> 问题二：重启之后，Indexer 会先检查 Checkpoint。它会获取到 Checkpoint 等于 100，然后会继续查找 100 这个高度，获取这个位置上的 Block，然后走一遍 Indexer 流程。
>
> 因为 Indexer 具有幂等性（Idempotent）这个特性，它重复执行多遍，也会保证结果不会发生变化，更不会产生重复的数据。

**老师判断与校准：✅ 基本通过。**
关键修正：Checkpoint=100 表示 Block 100 已可靠完成，重启后应该重新处理 Block 101，而不是 Block 100。因为 101 的数据虽然已写入，但 Checkpoint 没推进，所以系统必须按 At-least-once 语义允许再次处理 101。Idempotent Writer 依靠 Stable Identity + Unique Constraint + DO NOTHING / UPSERT 保证重复处理不产生重复结果。可压缩为：Checkpoint 决定从哪里恢复；At-least-once 允许重复处理；Idempotency 保证重复处理不重复产生结果。

### 问题 3
Realtime Sync、Backfill、Reorg Replay 为什么不应该分别实现 Parser / Decoder / Writer？One Processing Engine + Multiple Execution Modes 的价值是什么？

**用户回答：**
> 因为 Realtime Sync、Backfill 和 Reorg Replay 他们在 index 这个层面其实是完全相同的，只是他们面对的工作不同：
>
> 1. Realtime Sync：处理当前最新的 block index
> 2. Backfill：回填指定历史时段的 block
> 3. Reorg Replay：当发生 reorg 情况的时候, Real-time sync index，他会先回退掉 orphan block，然后继续 index canonical block
>
> 这样设计的好处在于，只需要维护一个 index，而不需要同时维护三套 index。这可以减少开发和维护的工作量，使 index 更加简洁。

**老师判断与校准：✅ 通过。**
核心完全正确。更精确的说法不是“维护一个 index”，而是“维护一套 Processing Engine”。三种 Execution Mode 的输入范围、Cursor / Checkpoint、触发条件不同，但 Parser / Decoder / Normalizer / Writer 应复用。价值不仅是减少代码量，更重要的是避免三套业务口径漂移：Decoder Bug 修复、ABI 更新、字段模型变化只需要修一处，Realtime / Backfill / Reorg Replay 会同时获得一致逻辑。

## 本课重点总结
```text
One Processing Engine
+
Multiple Execution Modes
+
Durable Progress
+
Idempotent Write
+
Reorg Recovery
=
Minimal Reliable Indexer
```
这一课把前 7 课的分散机制整合成一个可运行、可恢复的 Stateful Indexer 设计。