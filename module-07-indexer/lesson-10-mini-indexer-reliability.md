# 第10课｜Mini Indexer 可靠性实践：Crash Recovery / Backfill / Reorg Recovery

## Lesson Contract
【Blockchain Data Engineer 视角】

本课核心问题：
> 第 9 课我们已经有了一个最小 Processing Engine。现在怎样把它升级成一个真正“可恢复”的 Indexer？

本课重点实现三件事：
```text
Crash Recovery
+
Backfill
+
Reorg Recovery
```

学完以后，你应该能够：
- 把 Realtime Checkpoint 与 Backfill Cursor 分离
- 设计一个可恢复的 Backfill Job
- 用 `parent_hash` 检测 Reorg
- 找到 Common Ancestor
- 将旧分支标记为 orphaned
- 回退 Checkpoint
- Replay 新 canonical branch
- 解释为什么 Realtime / Backfill / Reorg Replay 应复用同一个 Processing Engine

本课暂不展开：
```text
Kafka
多 Worker 并发协调
深度 Reorg 优化
Finality Strategy
完整生产级 Data Quality
```
这些属于后续 Module。

## 一、先看第 9 课还缺什么
第 9 课已经有：
```text
RPC
↓
Fetcher
↓
Parser
↓
Decoder
↓
Normalizer
↓
Idempotent Writer
↓
Database
↓
Checkpoint
```

这条主链路已经成立。

但是它还存在三个问题。

第一个：
```text
Crash 后虽然理论上可以 Replay
但我们还没有明确测试恢复流程
```

第二个：
```text
只能从 Realtime Checkpoint 往前跑
不能指定一段历史区间 Backfill
```

第三个：
```text
虽然能够 Detect Reorg
但还不能真正 Rollback + Replay
```

所以第 10 课的本质是：
> 给第 9 课的 Processing Engine 加上完整 Control Path。

## 二、先重新确认一个核心架构
第 8 课已经形成过这个结构：
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
```

这节课我们真正把它写成代码。

核心思想仍然是：
```text
One Processing Engine
+
Multiple Execution Modes
```

## 三、第一部分：Crash Recovery
Crash Recovery 其实是三种可靠性机制里最简单的。

因为第 9 课实际上已经具备了核心条件：
```text
Checkpoint
+
Stable Identity
+
Unique Constraint
+
Idempotent Writer
```

假设：
```text
Checkpoint = 100
```

开始处理：
```text
Block 101
```

Block 101 有 500 条 Transfer。

程序写到：
```text
300 / 500
```

然后 Crash。

数据库：
```text
前 300 条已存在
```

Checkpoint：
```text
仍然是 100
```

重启以后：
```text
next_block = 101
```

然后重新执行整个 Block 101。

前 300 条：
```text
UNIQUE(chain_id, tx_hash, log_index)
↓
INSERT OR IGNORE
↓
忽略
```

后 200 条：
```text
不存在
↓
INSERT
```

最后：
```text
500 / 500
```

全部完整。

再更新：
```text
Checkpoint = 101
```

## 四、Crash Recovery 本身甚至不需要“特殊恢复程序”
这个点很重要。

错误的直觉是：
> Crash 以后是不是要写一套 Recovery Code？

对于这个模型来说，不需要。

因为：
```text
Normal Processing
=
Recovery Processing
```

恢复动作就是：
```text
从 Checkpoint 重新开始正常跑
```

也就是说：
> 一个好的可靠性设计，会让“恢复路径”尽可能等于“正常路径”。

这是一种非常重要的工程设计原则。

## 五、但我们还可以做得更好：Block Atomicity
第 9 课里，每个 Transfer 都单独：
```text
INSERT
COMMIT
```

这虽然依靠幂等性可以保证最终正确，但会出现 Partial Write。

更好的设计是：
```text
一个 Block
=
一个 DB Transaction
```

例如：
```python
def write_block_atomically(
    transfers,
    block_number,
    block_hash,
):
    conn = get_connection()

    try:
        for transfer in transfers:
            conn.execute(
                """
                INSERT OR IGNORE INTO token_transfers (...)
                VALUES (...)
                """,
                ...
            )

        conn.execute(
            """
            INSERT INTO indexer_checkpoints (...)
            VALUES (...)
            ON CONFLICT(...)
            DO UPDATE SET ...
            """,
            ...
        )

        conn.commit()

    except Exception:
        conn.rollback()
        raise

    finally:
        conn.close()
```

这样：
```text
Transfer Facts
+
Checkpoint
```

可以在同一个 DB transaction 中提交。

## 六、为什么这比“Data First → Checkpoint Second”更强？
原来的设计：
```text
Write Data
↓
Commit
↓
Save Checkpoint
↓
Commit
```

它是：
```text
Correct
```

因为 Crash 后可以 Replay。

但中间存在一个窗口：
```text
Data 已写
Checkpoint 未更新
```

这就是为什么需要 Idempotency。

如果放进一个 transaction：
```text
BEGIN

写全部 Transfer
更新 Checkpoint

COMMIT
```

那么结果只能是：
```text
全部成功
```

或者：
```text
全部失败
```

这叫：
```text
Atomicity
```

## 七、但注意：Idempotency 仍然不能删除
即使已经使用 DB Transaction，也不能说：
```text
有 Atomic Transaction
→ 不需要 Idempotency
```

因为还有：
```text
RPC Retry
Backfill overlap
Manual Replay
Reorg Replay
Job 重复执行
```

所以：
```text
Atomicity
```

和：
```text
Idempotency
```

是不同层面的保护。

可以理解成：
```text
Atomicity
→ 一个 Block 内部不要半成功

Idempotency
→ 同一个 Fact 重复执行也安全
```

## 八、第二部分：Backfill
现在产品告诉你：
> 当前 Indexer 从 Block 21,000,000 开始运行，但是我们还需要补 20,000,000 → 20,100,000 的历史 Transfer。

第一个错误方案是：
```text
把 Realtime Checkpoint 改成 19,999,999
```

这样程序就会从：
```text
20,000,000
```

重新开始跑。

问题是什么？

你的 Realtime 原本已经在：
```text
21,000,000
```

现在你把 Checkpoint 改回：
```text
19,999,999
```

等于破坏了实时同步状态。

## 九、所以必须有独立 Backfill State
新增：
```text
backfill_jobs
```

例如：
```sql
CREATE TABLE backfill_jobs (
    job_id TEXT PRIMARY KEY,
    chain_id INTEGER NOT NULL,
    indexer_name TEXT NOT NULL,
    start_block INTEGER NOT NULL,
    end_block INTEGER NOT NULL,
    last_processed_block INTEGER,
    status TEXT NOT NULL
);
```

例如：
```text
job_id = bf_001
start_block = 20,000,000
end_block = 20,100,000
last_processed_block = 20,045,000
status = RUNNING
```

## 十、Realtime Checkpoint 与 Backfill Cursor 的区别
Realtime：
```text
Checkpoint = 我实时已经可靠追到哪里
```

Backfill：
```text
Cursor = 这个历史任务已经补到哪里
```

两者不是同一种状态。

所以：
```text
Realtime Checkpoint
≠
Backfill Cursor
```

## 十一、Backfill Worker 怎么写？
可以非常简单：
```python
def run_backfill(job):
    cursor = job["last_processed_block"]

    if cursor is None:
        next_block = job["start_block"]
    else:
        next_block = cursor + 1

    while next_block <= job["end_block"]:

        process_block(next_block)

        save_backfill_cursor(
            job["job_id"],
            next_block,
        )

        next_block += 1
```

注意：
```python
process_block(next_block)
```

还是第 9 课那一个函数。

没有新写：
```text
backfill_parse()
backfill_decode()
backfill_writer()
```

## 十二、这就是 One Processing Engine
Realtime：
```python
process_block(block_number)
```

Backfill：
```python
process_block(block_number)
```

未来 Reorg Replay：
```python
process_block(block_number)
```

三种模式的不同只是：
```text
Block Number 从哪里来
+
Progress 保存在哪里
```

而：
```text
Business Processing
```

完全相同。

## 十三、Backfill Crash 怎么办？
假设：
```text
Backfill range:
1000 → 2000

Cursor = 1500
```

正在处理：
```text
1501
```

写完数据后程序 Crash。

Cursor 仍然：
```text
1500
```

重启以后：
```text
next = 1501
```

重新处理。

因为 Writer 幂等：
```text
安全
```

所以 Backfill 的可靠性模型和 Realtime 几乎完全一样：
```text
Backfill Cursor
+
At-least-once
+
Idempotent Write
```

## 十四、如果 Backfill 和 Realtime 重叠怎么办？
例如：
```text
Realtime 正在跑：
2000 → 2100

Backfill：
1900 → 2050
```

那么：
```text
2000 → 2050
```

会被两边都处理。

这是不是问题？

如果有：
```text
Stable Identity
+
Unique Constraint
+
Idempotent Writer
```

那么不是问题。

例如同一条 Transfer：
```text
(chain_id, tx_hash, log_index)
```

Realtime 写一次。

Backfill 又写一次。

第二次：
```text
Conflict
↓
Ignore / Safe UPSERT
```

最终只有一条。

## 十五、所以 Backfill 的核心不是“避开重复”
更合理的设计思想是：
> 允许 Execution overlap，但保证 Write 是安全的。

因为生产系统里完全消灭 overlap 很困难。

例如：
```text
Realtime
Backfill
Retry
Manual Replay
Reorg Replay
```

都可能交叉。

所以真正可靠的底层应该是：
```text
Overlap is expected
Idempotency absorbs overlap
```

## 十六、第三部分：Reorg Recovery
现在进入最关键的一部分。

假设数据库 Checkpoint：
```text
block_number = 102
block_hash = 102A
```

然后我们 Fetch：
```text
Block 103
```

得到：
```text
hash = 103B
parent_hash = 102B
```

但：
```text
102B != 102A
```

这说明：
```text
Chain Continuity Broken
```

也就是：
```text
Reorg Detected
```

## 十七、先加入最小 Detection
可以在 Realtime Controller 里：
```python
block = get_block(next_block)

if block["parentHash"] != checkpoint["block_hash"]:
    handle_reorg(checkpoint, block)
    return
```

注意：

`process_block()` 不应该负责决定：
```text
这是 Reorg 还是正常链
```

因为这是：
```text
Execution Controller / Chain Control
```

的职责。

不是 Decoder 的职责。

## 十八、为什么 parent_hash 是最自然的检查方式？
正常链：
```text
100A
↓
101A
↓
102A
↓
103A
```

满足：
```text
101A.parent_hash = hash(100A)
102A.parent_hash = hash(101A)
103A.parent_hash = hash(102A)
```

这就是链结构本身。

所以 Indexer 只需要验证：
```text
new_block.parent_hash
==
checkpoint.block_hash
```

就能够检查连续性。

## 十九、Detect 之后不能直接处理新 Block
错误：
```text
Checkpoint = 102A
↓
收到 103B
↓
直接 process 103B
```

因为数据库里可能仍然有：
```text
100A
101A
102A
```

同时又开始加入：
```text
103B
```

这在逻辑上是一条不存在的混合链。

所以必须进入：
```text
Reorg Recovery Mode
```

## 二十、Reorg Recovery 的第一步：Find Common Ancestor
假设：
```text
旧：
99 → 100A → 101A → 102A

新：
99 → 100B → 101B → 102B → 103B
```

共同祖先：
```text
99
```

我们必须知道：
> 从哪一个 Block 开始，旧链与新链分叉？

## 二十一、要找 Common Ancestor，必须保存 blocks
这里会看到第 8 课设计：
```text
blocks
```

表的重要性。

例如：
```sql
CREATE TABLE blocks (
    chain_id INTEGER NOT NULL,
    block_number INTEGER NOT NULL,
    block_hash TEXT NOT NULL,
    parent_hash TEXT NOT NULL,
    canonical INTEGER NOT NULL,

    PRIMARY KEY(chain_id, block_hash)
);
```

必须保留：
```text
block_hash
parent_hash
```

否则你只有 Transfer 表，很难恢复链结构。

## 二十二、为什么不能只存 Checkpoint？
Checkpoint 只告诉你：
```text
我之前认为最后完成的是 102A
```

但 Reorg 时你还需要知道：
```text
101A 是谁？
100A 是谁？
99 是谁？
```

才能向后寻找共同祖先。

所以：
```text
Checkpoint
=
Progress State

blocks
=
Chain Lineage
```

这是两种不同的数据。

## 二十三、Common Ancestor 的最小算法
概念伪代码：
```python
def find_common_ancestor(old_number):
    n = old_number

    while n >= 0:
        old_block = get_db_canonical_block(n)
        new_block = get_rpc_block(n)

        if old_block["block_hash"] == new_block["hash"]:
            return n

        n -= 1
```

例如：
```text
102:
DB = 102A
RPC = 102B
Mismatch

101:
DB = 101A
RPC = 101B
Mismatch

100:
DB = 100A
RPC = 100B
Mismatch

99:
DB = 99
RPC = 99
Match
```

所以：
```text
Common Ancestor = 99
```

## 二十四、真实生产不会无限向 Genesis 扫
当前为了理解，算法可以这么简单。

生产里通常会：
```text
限制最大 Reorg Depth
使用近期 Block Cache
使用 Finalized / Confirmed Boundary
优化 DB 查询
```

但是这些不是本课重点。

本课重点是正确性模型：
```text
向后找
直到 DB 与 RPC 的 block_hash 相同
```

## 二十五、找到 Common Ancestor 之后怎么办？
假设：
```text
Common Ancestor = 99
```

旧 canonical：
```text
100A
101A
102A
```

全部需要：
```text
canonical = false
```

对应的：
```text
transactions
logs
token_transfers
```

也需要失效。

## 二十六、最简单的 rollback
例如：
```sql
UPDATE blocks
SET canonical = 0
WHERE chain_id = ?
  AND block_number > ?;
```

假设 ancestor = 99：
```text
block_number > 99
```

旧 canonical block 全部失效。

然后：
```sql
UPDATE token_transfers
SET canonical = 0
WHERE chain_id = ?
  AND block_number > ?;
```

当然生产环境应该更谨慎，用：
```text
block_hash lineage
```

限制旧 branch。

但学习模型先这样理解。

## 二十七、为什么保留 orphaned 数据而不是 DELETE？
两个方案都可以：
```text
DELETE
```

或者：
```text
canonical = false
```

当前更推荐：
```text
canonical = false
```

因为你可以保留：
```text
Reorg History
Audit
Debug
Provider Comparison
Data Lineage
```

所以：
```text
Preserve Observation
+
Filter Canonical Truth
```

是一种很常见的生产思路。

## 二十八、Dashboard 怎么处理？
如果 `token_transfers` 中同时保存：
```text
100A canonical=false
100B canonical=true
```

那么上层查询必须：
```sql
WHERE canonical = 1
```

否则：
```text
旧 Transfer
+
新 Transfer
```

会一起被计算。

这就是为什么 `canonical` 不只是 Block 表字段。

Fact 层也必须能识别链归属。

## 二十九、Rollback 后 Checkpoint 必须回退
原来：
```text
Checkpoint = 102A
```

Common Ancestor：
```text
99
```

那么：
```text
Checkpoint = 99
```

注意：

Checkpoint 平时：
```text
只向前
```

但发生 Reorg 时：
```text
允许逻辑回退
```

这是一个重要例外。

## 三十、然后 Replay 新 Branch
现在数据库状态：
```text
Checkpoint = 99
```

旧：
```text
100A / 101A / 102A
canonical=false
```

接下来：
```text
100B
101B
102B
103B
```

全部重新：
```text
process_block()
```

也就是继续使用第 9 课的 Processing Engine。

## 三十一、Reorg Replay 不需要单独 Decoder
所以：
```text
Reorg Recovery
```

实际上分两段：

第一段：
```text
Control Path
```

负责：
```text
Detect
Find Common Ancestor
Invalidate Old Branch
Rollback Checkpoint
```

第二段：
```text
Normal Processing Engine
```

负责：
```text
Fetch
Parse
Decode
Normalize
Write
```

这就是职责分离。

## 三十二、把 Reorg Controller 写成伪代码
```python
def handle_reorg(checkpoint):

    ancestor = find_common_ancestor(
        checkpoint["block_number"]
    )

    mark_orphaned_after(
        ancestor
    )

    save_checkpoint(
        CHAIN_ID,
        INDEXER_NAME,
        ancestor,
        get_db_block_hash(ancestor),
    )
```

然后正常主循环继续：
```text
next_block = ancestor + 1
```

自然就进入 Replay。

## 三十三、完整 Realtime Loop
现在主循环变成：
```python
while True:

    checkpoint = load_checkpoint(
        CHAIN_ID,
        INDEXER_NAME,
    )

    next_number = (
        checkpoint["block_number"] + 1
    )

    block = get_block(next_number)

    if (
        block["parentHash"]
        != checkpoint["block_hash"]
    ):
        handle_reorg(checkpoint)
        continue

    process_block(next_number)

    save_checkpoint(
        CHAIN_ID,
        INDEXER_NAME,
        next_number,
        block["hash"],
    )
```

这已经非常接近：
```text
Reliable Mini Indexer
```

## 三十四、现在三种 Execution Mode 已经出现
Realtime：
```text
checkpoint + 1
→ process_block()
→ update checkpoint
```

Backfill：
```text
backfill_cursor + 1
→ process_block()
→ update backfill cursor
```

Reorg Replay：
```text
rollback checkpoint
→ ancestor + 1
→ process_block()
→ update checkpoint
```

所以从代码角度看：
```text
谁决定 next block
```

不同。

但：
```text
怎么处理 block
```

完全相同。

## 三十五、这就是 Control Plane 与 Data Plane
可以进一步抽象。

【系统架构视角】

Data Plane：
```text
Processing Engine
```

负责：
```text
Block → Fact
```

Control Plane：
```text
Realtime Controller
Backfill Controller
Reorg Controller
Checkpoint / Cursor
```

负责：
```text
处理哪个 Block
什么时候处理
失败从哪里恢复
发生 Reorg 怎么回退
```

## 三十六、这和银行数据平台很像
你可以类比银行批处理。

同一个存储过程 / ETL：
```text
transform_transaction()
```

可能被：
```text
日常批次
历史补数
失败重跑
冲正重处理
```

调用。

如果：
```text
日常批次一套 ETL
补数一套 ETL
重跑一套 ETL
```

长期一定会产生：
```text
口径漂移
```

所以最合理的是：
```text
一个业务处理逻辑
多个调度入口
```

这和这里完全一样。

## 三十七、现在把 Module 7 的可靠性统一起来
Crash：
```text
same chain fact
reprocess
```

靠：
```text
Checkpoint + Idempotency
```

Backfill：
```text
historical processing
```

靠：
```text
Independent Cursor + Same Processing Engine
```

Reorg：
```text
canonical truth changed
```

靠：
```text
Detect
→ Common Ancestor
→ Invalidate
→ Rollback
→ Replay
```

## 三十八、一个非常重要的区别
不要把：
```text
Retry
Backfill
Reorg
```

看成一回事。

Retry：
```text
同一个 Fact 再执行一次
```

Backfill：
```text
原本没处理 / 需要重算的历史 Fact
```

Reorg：
```text
原来的 Fact 已经不是 canonical truth
```

所以：
```text
Retry
→ Duplicate Processing

Backfill
→ Historical Processing

Reorg
→ Truth Correction
```

这是三种完全不同的语义。

## 三十九、但它们最终都依赖同一个底层能力
底层必须有：
```text
Stable Identity
Source Lineage
Idempotent Write
Block Hash
Checkpoint / Cursor
```

如果这些设计正确：
```text
Retry
Backfill
Reorg Replay
```

都容易处理。

如果这些设计错误：
```text
后面所有可靠性功能都会很痛苦
```

## 四十、本课核心总结
第一：
```text
Crash Recovery
≈ 从 Checkpoint 重新执行
```

前提：
```text
Idempotent Write
```

第二：
```text
Realtime Checkpoint
≠
Backfill Cursor
```

第三：
```text
Realtime / Backfill / Reorg Replay
共用同一个 Processing Engine
```

第四：
```text
Reorg Detection
=
new_block.parent_hash
!=
checkpoint.block_hash
```

第五：
```text
Reorg Recovery
=
Find Common Ancestor
→ Invalidate Old Branch
→ Rollback Checkpoint
→ Replay New Branch
```

第六：
```text
block_number = Position
block_hash = Identity
parent_hash = Lineage
canonical = Current Truth
```

第七：
```text
Production Indexer
=
Data Plane
+
Control Plane
```

## 理解检查
### 问题 1
假设：
```text
Realtime Checkpoint = 20,000,000
```
现在产品要求 Backfill：
```text
18,000,000 → 19,000,000
```
为什么不能把 Realtime Checkpoint 直接改成：
```text
17,999,999
```
然后重新跑？

正确设计应该是什么？

请重点解释：
```text
Realtime Checkpoint
vs
Backfill Cursor
```

### 问题 1：用户回答
> 问题一：如果直接修改 Realtime Checkpoint，那么它会影响到 Realtime Index 的执行。本来 Index 是处理最新的 block 数据，但是你把 Checkpoint 修改了之后，下一个它又会从历史的 block 开始执行，所以 Backfill 需要单独的一个 Index。
>
> 我们需要指定的关键字段有：
>
> 1. Chain ID
> 2. Index Name
>
> 针对 Backfill 的特殊情况，还需要指定 Start 和 End block，然后保存 Backfill Cursor。Backfill Cursor 等同于 Realtime Checkpoint，它用于标识 Backfill 当前处理的 block

### 问题 1：老师判断与校准
问题 1 通过。主线判断是对的。

第一个术语校准：
> “Backfill 需要单独的一个 Index”

更准确的说法是：Backfill 需要单独的 Progress State / Cursor，不一定需要单独一套 Indexer Processing Engine。

也就是说：
```text
Realtime
→ 有自己的 Checkpoint

Backfill
→ 有自己的 Backfill Cursor
```
但二者都可以调用同一个：
```text
process_block()
```
所以：
```text
独立进度
≠
独立处理逻辑
```

第二个术语校准：
> “Backfill Cursor 等同于 Realtime Checkpoint”

更准确地说，两者在“记录处理进度”这个作用上相似，但语义不同。

Realtime Checkpoint 表示：
```text
实时任务已经可靠追到哪个 Block
```

Backfill Cursor 表示：
```text
这个有边界的历史补数任务已经处理到哪个 Block
```

一个 Backfill Job 最小可以有：
```text
job_id
chain_id
indexer_name
start_block
end_block
last_processed_block
status
```
其中 `last_processed_block` 就是 Backfill Cursor；`chain_id + indexer_name` 说明是哪条链、哪个 Indexer 逻辑的任务，但不足以唯一表达某一个 Backfill Job，因为同一个 Indexer 可以同时有多个不同历史区间任务，所以通常还需要 `job_id`。

压缩成一句：
```text
Realtime Checkpoint 管实时进度，
Backfill Cursor 管某个历史区间任务的进度，
两者进度隔离，但复用同一 Processing Engine。
```

### 问题 2
假设：
```text
Checkpoint:
102A

新获取：
103B

103B.parent_hash = 102B
```
而：
```text
102A != 102B
```

请你完整描述 Indexer 接下来应该做的步骤。

至少包含：
```text
Detect Reorg
Find Common Ancestor
Invalidate Old Branch
Rollback Checkpoint
Replay New Branch
```

### 问题 2：用户回答
> Indexer 接下来应该做的有：
> • Detect reorg
> • Find common ancestor
> • Rollback checkpoint
> • Invalidate old branch
> • Replay new branch

### 问题 2：老师判断与校准
问题 2 基本通过，步骤都列全了，但顺序上需要校准。

更标准的顺序是：
```text
1. Detect Reorg
2. Find Common Ancestor
3. Invalidate Old Branch
4. Rollback Checkpoint
5. Replay New Branch
```

例如：
```text
旧链：
99 → 100A → 101A → 102A

新链：
99 → 100B → 101B → 102B → 103B
```
发现：
```text
Checkpoint = 102A
103B.parent_hash = 102B
102A != 102B
→ Detect Reorg
```
接着找到 Common Ancestor = 99，将 100A / 101A / 102A 标记为 `canonical=false`，再将 Checkpoint 从 102A 回退到 99，最后 Replay 100B / 101B / 102B / 103B。

可以记成：
```text
Detect
→ Find Ancestor
→ Invalidate Old
→ Rollback Progress
→ Replay New
```

另外要区分两个“Rollback”对象：
```text
Rollback old branch facts
→ canonical = false

Rollback checkpoint
→ 回到 Common Ancestor
```

### 问题 3
为什么：
```text
Realtime
Backfill
Reorg Replay
```
应该共用：
```text
process_block()
```
而不是分别写：
```text
process_realtime_block()
process_backfill_block()
process_reorg_block()
```

请从：
```text
业务口径一致性
维护成本
Bug 修复
```
三个角度回答。

### 问题 3：用户回答
> 因为 realtime backfill 还有 reorg replay，他们在 process_block 这个处理单元所做的工作内容是同样的，所以应该用同一套处理逻辑，符合业务口径的一致性。这样的话，更加便于维护和 bug 修复

### 问题 3：老师判断与校准
问题 3 通过。

三种 Execution Mode 在 `process_block()` 这个处理单元所做的事情相同：都要把同样的 Block 经过 Fetch / Parse / Decode / Normalize / Idempotent Write 转换成同样的业务事实。

因此共用同一个 Processing Engine 有三方面价值：

1. **业务口径一致性**：Realtime、Backfill、Reorg Replay 使用同一套 Decoder / Normalizer，不会因为三份代码逐渐分叉而出现同一个 Transfer 在不同模式下得到不同结果。
2. **维护成本**：只维护一套核心 Processing Engine。不同 Execution Mode 只负责输入范围、触发条件、Checkpoint / Cursor 等 Control Plane 状态。
3. **Bug 修复**：如果 Decoder、Normalizer 或 Writer 有 bug，只需要在一处修复，Realtime、Backfill、Reorg Replay 会同时获得修复；如果是三套实现，就需要同步修改三处，并承担漏改和逻辑漂移风险。

可以压缩成：
```text
One Processing Engine
+
Multiple Execution Modes
=
Same Business Semantics
+
Lower Maintenance Cost
+
One-place Bug Fix
```
