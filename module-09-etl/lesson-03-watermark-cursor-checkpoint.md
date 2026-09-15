# 第3课｜Watermark / Cursor / Checkpoint：ETL 如何记住处理到哪里

【Lesson Contract】

所属 Module：Module 9 — ETL。

本课核心问题：

> 一个 Incremental ETL Job 怎么知道“上次已经处理到哪里”，以及失败后应该从哪里继续？

学完本课，你应该能够：

- 区分 Watermark、Cursor、Checkpoint；
- 理解它们为什么经常看起来很像；
- 判断一个 ETL Job 应该用什么字段作为增量边界；
- 理解为什么 Checkpoint 只能在“这一段处理完整成功”以后推进；
- 理解为什么 Backfill 的状态不能污染正常 Incremental 的状态。

本课暂时不展开：

- Scheduler / Airflow；
- Kafka Offset；
- Reorg Repair Framework；
- Exactly-once Streaming。

这些属于后续内容。

---

## 一、先从上一课留下的问题开始

上一课我们已经有：

```text
run_job(start, end)
```

正常 Incremental：

```text
start = last_checkpoint
end   = new_boundary
```

问题是：

> `last_checkpoint` 从哪里来？

程序总不能每次启动以后猜：

```text
上次好像处理到 09-14？
```

它必须把自己的处理进度保存下来。

例如：

```text
wallet_token_daily_flow_job
```

已经成功处理：

```text
2026-09-14
```

那么系统至少要记住：

```text
processed_until = 2026-09-15 00:00
```

下一次才能知道：

```text
start = 2026-09-15 00:00
```

所以 Incremental ETL 本质上一定需要：

> 一种“进度状态”。

而 Watermark、Cursor、Checkpoint，本质上都和这个问题有关。

---

## 二、为什么三个词很容易混淆？

你在 Module 7 已经接触过：

```text
Checkpoint
Cursor
```

现在又出现：

```text
Watermark
```

它们经常都会长成一个值：

```text
20000000
```

或者：

```text
2026-09-14
```

所以很容易觉得：

> 这三个词是不是其实完全一样？

不能简单说一样。

更准确地说：

> 它们可能保存相同的“边界值”，但强调的语义不同。

这是本课最重要的地方。

---

## 三、先理解 Watermark

【Data Engineering 视角】

Watermark 更强调：

> 数据处理的“边界”。

例如 Source：

```text
fact_token_transfers
```

里面有：

```text
block_time
```

你设计一个增量 ETL：

```sql
WHERE block_time >= :start
  AND block_time <  :end
```

如果系统记录：

```text
watermark = 2026-09-15 00:00
```

它表达的是：

> 在这个边界之前的数据，我认为已经进入了已处理范围。

所以可以画成：

```text
时间轴

───────────────|────────────→
               ↑
           Watermark
```

左边：

```text
已处理范围
```

右边：

```text
尚未处理范围
```

所以 Watermark 主要解决的是：

> “Incremental Boundary 在哪里？”

---

## 四、Watermark 不一定是时间

这一点很重要。

Watermark 只是“有序边界”。

可以是：

```text
timestamp
```

例如：

```text
2026-09-15 00:00
```

也可以是：

```text
block_number
```

例如：

```text
20,000,000
```

也可以是：

```text
auto_increment_id
```

例如：

```text
customer_id = 938475
```

还可能是：

```text
updated_at
```

所以 Watermark 的关键不是“是不是时间”。

而是：

> 这个字段能不能定义一个稳定的处理顺序和增量范围？

---

## 五、Blockchain ETL 里的 Watermark

假设 Source 是：

```text
fact_token_transfers
```

你可以有两种常见 Watermark。

方案一：

```text
block_number
```

例如：

```text
watermark = 20,000,000
```

下一批：

```text
20,000,001 ~ 20,010,000
```

方案二：

```text
block_time
```

例如：

```text
watermark = 2026-09-15 00:00
```

下一批：

```text
2026-09-15
~
2026-09-16
```

选择哪个，要看 Target Grain 和 Job 的处理方式。

例如：

```text
daily_flow
```

更自然按：

```text
date
```

处理。

而：

```text
Indexer
```

更自然按：

```text
block_number
```

处理。

---

## 六、Cursor 是什么？

Cursor 更强调：

> “当前执行走到哪里了。”

举一个最简单的 Backfill。

你要处理：

```text
2026-01-01
~
2026-06-01
```

按天执行：

```text
01-01
01-02
01-03
...
```

跑到：

```text
2026-03-18
```

程序挂了。

那么你可能保存：

```text
cursor = 2026-03-18
```

Cursor 表达的是：

> 当前这个任务实例，已经推进到什么位置。

所以：

```text
Backfill Range
01-01 ───────────────── 06-01
            ↑
          Cursor
```

Cursor 更偏：

```text
execution position
```

而 Watermark 更偏：

```text
processed boundary
```

实际工程里，两者经常会存同一个值。

但语义不同。

---

## 七、用数据库游标理解 Cursor

你以前学 Oracle，Cursor 这个词应该很熟。

数据库 Cursor 也是：

> 当前扫描位置。

例如：

```text
Row 1
Row 2
Row 3
Row 4
```

处理到：

```text
Row 3
```

Cursor 指向：

```text
Row 3
```

ETL Cursor 是同一种抽象。

只是它扫描的不一定是：

```text
row
```

可能是：

```text
block
date
partition
page
batch
```

所以可以简单理解：

> Cursor = 当前任务“走到哪里”。

---

## 八、Checkpoint 又是什么？

Checkpoint 强调的是：

> 一个可以安全恢复的“已完成点”。

这是三个概念里面工程语义最强的一个。

假设：

```text
09-14
```

这个 Job 执行：

```text
Extract
↓
Transform
↓
Load
```

如果刚做完 Transform：

```text
checkpoint = 09-14
```

可以吗？

不可以。

因为 Target 还没写成功。

如果程序这时挂了，下一次看到：

```text
checkpoint = 09-14
```

可能会认为：

```text
09-14 已经完成
```

于是直接处理：

```text
09-15
```

那么：

```text
09-14
```

就永久丢了。

所以 Checkpoint 必须代表：

> 到这里为止，完整数据处理已经成功完成，并且可以安全从后面继续。

---

## 九、Checkpoint 的推进时机

正确顺序通常是：

```text
1. Determine Input Range

2. Extract

3. Transform

4. Load

5. Verify Success

6. Advance Checkpoint
```

也就是说：

```text
Checkpoint
```

永远最后推进。

例如：

```text
checkpoint = 09-14
```

准备处理：

```text
09-15
```

执行中：

```text
Extract ✓
Transform ✓
Load ✗
```

那么：

```text
checkpoint
```

仍然必须保持：

```text
09-14
```

下一次：

```text
重新处理 09-15
```

这就是上一课你已经掌握的：

> Checkpoint 只记录最后完整成功的边界。

---

## 十、这和数据库事务有一点相似

你可以把 Checkpoint 理解成：

```text
数据处理任务的 commit point
```

注意这里只是类比，不代表技术实现一定是数据库 Transaction。

例如：

```text
处理 09-15
```

整个过程：

```text
read source
↓
calculate
↓
write target
↓
checkpoint = 09-15
```

只有最后一步完成以后，系统才承认：

```text
09-15 Done
```

否则：

```text
09-15 Not Done
```

这就是恢复能力的基础。

---

## 十一、Watermark、Cursor、Checkpoint 放在一起

现在我们可以做一个语义区分。

Watermark：

```text
处理边界在哪里？
```

Cursor：

```text
当前任务执行到哪里？
```

Checkpoint：

```text
哪个位置已经确认安全完成？
```

可以写成：

```text
Watermark
= logical boundary

Cursor
= execution position

Checkpoint
= durable recovery point
```

在简单系统里：

```text
watermark
cursor
checkpoint
```

可能最后全部保存成：

```text
2026-09-14
```

所以看起来像一个东西。

但是在复杂系统里，它们就可能分开。

---

## 十二、一个例子：Backfill 100 天

假设你 Backfill：

```text
2026-01-01
~
2026-04-11
```

一共大约 100 天。

你按天执行。

当前：

```text
cursor = 2026-03-18
```

但假设 03-18 正在执行，还没有完成。

那么：

```text
cursor = 03-18
```

但：

```text
checkpoint = 03-17
```

因为：

```text
03-17
```

才是最后安全完成的 processing unit。

这个例子可以很好地区分：

```text
Cursor
!=
Checkpoint
```

---

## 十三、Watermark 甚至可能比 Checkpoint 更“逻辑化”

再看一个例子。

你的 ETL 按：

```text
updated_at
```

增量读取。

Source 当前最大时间是：

```text
10:35:42
```

理论 Watermark：

```text
10:35:42
```

但为了避免 late-arriving data，你可能故意只处理到：

```text
10:30:00
```

于是：

```text
Source high watermark = 10:35:42
```

但是：

```text
safe processing checkpoint = 10:30:00
```

这时两个概念就明显不同。

我们今天不深入 Late-arriving，只让你看到：

> Watermark 和 Checkpoint 并不是天然完全相同。

---

## 十四、为什么 “MAX(timestamp)” 很危险？

很多简单 ETL 会这样写：

```sql
SELECT MAX(block_time)
FROM target;
```

然后：

```text
从这个时间继续
```

看起来很聪明。

但其实可能危险。

为什么？

假设 Target 中：

```text
09-14
09-15
09-17
```

但：

```text
09-16
```

因为任务失败缺失。

这时候：

```text
MAX(date) = 09-17
```

系统会认为：

```text
已经处理到 09-17
```

但实际上中间有洞。

所以：

> “目标表最大值”不一定等于“可靠处理进度”。

Checkpoint 最好是：

```text
显式状态
```

而不是：

```text
从业务数据推断状态
```

---

## 十五、Checkpoint 表怎么设计？

一个最简单的 ETL 状态表可能是：

```text
etl_checkpoint
```

字段：

```text
job_name
checkpoint_value
updated_at
status
```

例如：

```text
job_name
=
wallet_token_daily_flow

checkpoint_value
=
2026-09-14

status
=
SUCCESS
```

如果是多链：

```text
chain_id
job_name
checkpoint_value
```

例如：

```text
1
wallet_token_daily_flow
2026-09-14
```

因为：

```text
Ethereum
Base
Arbitrum
```

可能处理进度不同。

---

## 十六、Checkpoint 的 Key 应该是什么？

这里开始和 Module 7 对上了。

你之前在 Indexer 里已经学过：

```text
chain_id
+ index_name
```

可以标识一个 Realtime Checkpoint。

ETL 也是同样思路。

例如：

```text
chain_id
job_name
```

作为 checkpoint identity。

例如：

```text
chain_id = 1
job_name = wallet_token_daily_flow
```

Checkpoint：

```text
2026-09-14
```

如果另一个 Job：

```text
dex_volume_daily
```

就有自己独立的 Checkpoint。

所以：

> Checkpoint 属于 Pipeline / Job，不属于整个系统。

这一点非常重要。

---

## 十七、Backfill 为什么不能改正常 Checkpoint？

这个你上一课已经答对了。

假设：

```text
Normal Checkpoint
=
2026-09-14
```

现在 Backfill：

```text
2026-08-01
~
2026-08-07
```

如果 Backfill 把：

```text
checkpoint
```

修改成：

```text
2026-08-07
```

那么下次正常任务就会认为：

```text
从 08-08 开始
```

结果：

```text
08-08 ~ 09-14
```

全部重新跑一遍。

所以正常 Incremental 与 Backfill 必须：

```text
state isolation
```

例如：

```text
realtime/incremental checkpoint
=
09-14
```

Backfill：

```text
backfill_start = 08-01
backfill_end   = 08-08
backfill_cursor = 08-05
```

两套状态完全分开。

---

## 十八、一个更完整的状态模型

可以设计：

```text
etl_job_state
```

正常 Incremental：

```text
job_name
chain_id
mode = incremental
checkpoint
```

Backfill：

```text
job_name
chain_id
mode = backfill
range_start
range_end
cursor
status
```

例如：

```text
wallet_token_daily_flow
1
backfill
2026-08-01
2026-08-08
2026-08-05
RUNNING
```

这样就非常清楚：

```text
Incremental State
```

和：

```text
Backfill State
```

互不影响。

---

## 十九、失败恢复怎么发生？

假设：

```text
checkpoint = 09-14
```

准备处理：

```text
09-15
```

流程：

```text
09-15
↓
Extract
↓
Transform
↓
Load
```

Load 中途失败。

Checkpoint 仍然：

```text
09-14
```

服务重启后：

```text
read checkpoint
=
09-14
```

于是重新运行：

```text
09-15
```

这里立刻又出现一个问题：

> 如果 09-15 已经写进去一半怎么办？

答案就是上一课和 Module 7 已经反复出现的：

```text
Idempotency
```

所以你现在应该看到：

```text
Checkpoint
+
Idempotent Load
```

其实是一对。

Checkpoint 解决：

```text
从哪里重跑
```

Idempotency 解决：

```text
重跑会不会把数据搞坏
```

---

## 二十、为什么 Checkpoint 单独存在还不够？

假设：

```text
checkpoint = 09-14
```

09-15 Job：

```text
写了一半 target
↓
crash
```

重启：

```text
checkpoint 还是 09-14
```

很好。

它会重新处理：

```text
09-15
```

但是如果你的 Load 是：

```sql
INSERT
```

那么之前已经写进去的那一半：

```text
又 INSERT 一次
```

可能产生重复。

所以可靠 ETL 必须同时具备：

```text
Checkpoint
+
Idempotent Load
```

两者缺一不可。

这一点和你之前的 Mini Indexer 是完全一样的。

---

## 二十一、Checkpoint 应该按 Row 更新，还是按 Batch 更新？

假设一天有：

```text
10,000,000 Transfer
```

是不是每处理一条：

```text
checkpoint++
```

通常不一定。

如果你的 Processing Unit 是：

```text
one day
```

那么更自然的是：

```text
09-15 整天完成
↓
checkpoint = 09-15
```

而不是每处理一个 row 更新一次。

原因是：

```text
状态写入成本
复杂度
恢复语义
```

都更简单。

所以 Checkpoint 的粒度应该和：

> Processing Unit

保持一致。

上一课我们已经讲过：

```text
Daily Unit
Weekly Unit
Partition Unit
```

现在你可以把它们连起来：

```text
Processing Unit
↓
Checkpoint Unit
```

---

## 二十二、Blockchain ETL 常见的两级 Checkpoint

一个典型链上数据平台可能同时存在：

```text
Indexer Checkpoint
```

例如：

```text
block_number = 20,000,000
```

和：

```text
DWS ETL Checkpoint
```

例如：

```text
date = 2026-09-14
```

数据流：

```text
Blockchain
↓
Indexer
↓
fact_token_transfers
↓
Daily ETL
↓
dws_wallet_token_daily_flow
```

所以状态是：

```text
Indexer Checkpoint
=
20,000,000
```

另一个：

```text
Daily ETL Checkpoint
=
2026-09-14
```

【Blockchain Data Engineer 视角】

一定要记住：

> 每一层 Pipeline 都维护自己的 processing state。

而不是整个系统共用一个：

```text
global checkpoint
```

---

## 二十三、本课最重要的心智模型

以后看到增量 ETL，先问四件事：

```text
1. Boundary 是什么？
   block_number?
   timestamp?
   date?
   ID?

2. 当前任务执行到哪里？
   Cursor

3. 哪个位置已经安全完成？
   Checkpoint

4. 下一次从哪里开始？
   checkpoint → next range
```

可以简化成：

```text
Watermark
= 边界

Cursor
= 当前走到哪里

Checkpoint
= 已确认完成到哪里
```

---

## 二十四、把 Module 7 和 Module 9 串起来

你现在应该能看到一个很重要的规律。

Indexer：

```text
block range
↓
process_block
↓
writer
↓
checkpoint
```

ETL：

```text
date range
↓
transform
↓
load
↓
checkpoint
```

本质上都是：

```text
Ordered Input
↓
Processing Unit
↓
Durable Output
↓
Advance Progress State
```

所以：

> Checkpoint 不是 Blockchain 专属概念，它是 Data Pipeline 的通用恢复机制。

Module 7 是你第一次在链上 Indexer 场景看到它。

Module 9 则把它提升成通用 Data Engineering 抽象。

---

# 第 3 课理解检查

先做三题。

问题一：

一个 Daily ETL 当前：

```text
checkpoint = 2026-09-14
```

现在正在处理：

```text
2026-09-15
```

Transform 已完成，但 Load 执行到一半程序崩溃。

请回答：

1. Checkpoint 应该是 `09-14` 还是 `09-15`？
2. 重启以后应该重新处理哪一天？
3. 为什么还必须要求 Load 是 Idempotent？

问题二：

一个 Backfill：

```text
start = 2026-01-01
end   = 2026-06-01
```

目前正在处理：

```text
2026-03-18
```

但是 03-18 尚未完成。

这时候：

```text
cursor
```

和：

```text
checkpoint
```

你认为分别可以是什么？

请解释两者区别。

问题三：

系统里有两个 Pipeline：

```text
Ethereum Indexer
```

以及：

```text
wallet_token_daily_flow ETL
```

Indexer 当前：

```text
checkpoint = block 20,000,000
```

Daily ETL 当前：

```text
checkpoint = 2026-09-14
```

为什么它们不应该共用一个 Checkpoint？

请从“Pipeline State”角度回答。

---

## 用户回答

问题一：1.Checkpoint 应该定在 0914
2.重启以后应该处理 0915 这一天
3. 因为之前报错的时候，09-15 已经处理过一部分了，所以表里是有部分数据的。如果直接 insert 的话，会导致有重复的数据，所以必须要求 load 是 idempotent（幂等的）
问题二：

这时候 Cursor 应该是 0318，Checkpoint 应该是 0317。因为 Checkpoint 代表的是已经完成的进度，而 Cursor 指向的是当前正在进行的进度
问题三：因为它们是不同的两个 pipeline，它们都有自己维护的 checkpoint，不应该共用一个 checkpoint

## 老师判断与校准

三题都正确，可以正式通过。第 2 题对 Cursor 与 Checkpoint 的区分尤其准确：Cursor 表示“当前执行位置”，Checkpoint 表示“最后安全完成位置”。

第 1 题也完整抓住了 Checkpoint 与 Idempotency 的组合关系：Checkpoint 决定失败后从哪里重跑，Idempotent Load 保证重跑不会因为前一次已写入的部分结果而产生重复或破坏目标数据。

第 3 题结论正确。更精确地说，Checkpoint 是某一个 Pipeline / Job 自己的 durable processing state；Ethereum Indexer 与 `wallet_token_daily_flow` ETL 的输入坐标系、Processing Unit 和推进条件都不同，因此必须分别维护状态，不能使用一个全局 Checkpoint。

## 结课判定

Module 9 第 3 课理解检查通过，必要校准完成，本课正式结束。