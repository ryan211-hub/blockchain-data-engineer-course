# Module 9 第 1 课｜ETL 到底是什么：从“已有数据”到“可持续数据管道”

【Lesson Contract】

所属 Module：Module 9 — ETL。

本课核心问题：

> 为什么已经有了 Indexer 和数据模型之后，还需要 ETL？

学完本课，你应该能够解释：

- ETL 的 E、T、L 分别在做什么；
- 为什么 ETL 不是“一段 SQL”；
- ETL 和 Indexer 的职责有什么区别；
- 为什么一个可靠的数据平台必须考虑重复运行、失败恢复和处理范围；
- Blockchain ETL 相比传统 ETL 多了哪些特殊问题。

本课暂时不展开：

- DAG / Airflow；
- Kafka / Streaming；
- ClickHouse 等数据库选型；
- 完整 Data Quality Framework。

这些放到后续课程。

---

## 一、先从一个你已经熟悉的场景开始

假设你的 Mini Indexer 已经能够不断把 Ethereum `Transfer` Event 写进：

```text
fact_token_transfers
```

字段包括：

```text
chain_id
block_number
block_time
tx_hash
log_index
token_address
from_address
to_address
amount_raw
amount
```

Module 8 又设计了一张：

```text
dws_wallet_token_daily_flow
```

它的 Grain 是：

```text
one wallet
+ one token
+ one day
```

问题来了：

> `dws_wallet_token_daily_flow` 里的数据是谁生成的？

它不会凭空出现。

你必须定期从：

```text
fact_token_transfers
```

读取明细，然后：

```text
按 date
按 wallet
按 token
统计 received_amount
统计 sent_amount
统计 transfer_count
```

最后把结果写入：

```text
dws_wallet_token_daily_flow
```

这个过程已经非常接近 ETL。

---

## 二、ETL 最简单的定义

ETL：

```text
Extract
↓
Transform
↓
Load
```

翻译成数据工程语言：

```text
从哪里读
↓
怎么算
↓
写到哪里
```

例如：

```text
Extract
fact_token_transfers

↓

Transform
按 wallet + token + date 聚合

↓

Load
dws_wallet_token_daily_flow
```

所以 ETL 本质上不是一个神秘框架。

它首先是一种：

> 数据从 Source 经过处理后进入 Target 的工程流程。

---

## 三、为什么你以前其实已经在做 ETL？

【传统数据工程视角】

你现在银行工作里做的数据提取，其实已经包含大量 ETL 思想。

例如业务系统里有：

```text
交易流水表
客户表
账户表
```

业务要求：

> 每天统计每个客户的交易金额。

你可能写：

```text
SELECT ...
FROM transaction
JOIN account
JOIN customer
GROUP BY customer_id, date
```

然后把结果：

- 导出；
- 写入中间表；
- 写入报表表；
- 提供给下游系统。

本质上就是：

```text
Extract
业务表读取数据

Transform
JOIN / Filter / Group By

Load
写入目标表
```

所以 ETL 对你并不是全新的东西。

真正需要系统补上的，是：

> 怎么把“我会写一段 SQL 处理数据”升级成“我会设计一个可靠的数据管道”。

---

## 四、ETL 和 SQL 最大的区别在哪里？

例如你写：

```sql
INSERT INTO dws_wallet_token_daily_flow
SELECT
    ...
FROM fact_token_transfers
GROUP BY ...
```

这只是 Transform + Load 的逻辑。

但一个真正的数据工程系统还要回答：

```text
今天处理哪一天？
从哪个 block 开始？
处理到哪个 block？
昨天已经处理过了吗？
任务执行一半挂了怎么办？
重新执行会不会重复数据？
历史 6 个月要补数怎么办？
上游晚到了怎么办？
Reorg 后旧数据怎么办？
```

这些问题都不是一个 `GROUP BY` 本身能回答的。

所以：

> SQL 是数据处理逻辑的一部分；ETL 是把数据处理逻辑变成可持续运行的数据系统。

这是 Module 9 最重要的变化。

---

## 五、一次性脚本 vs 数据管道

一次性脚本可能是：

```text
读取数据
↓
计算
↓
写表
↓
结束
```

如果失败：

```text
人工重新跑
```

如果重复：

```text
可能产生重复数据
```

如果要补历史：

```text
修改代码
重新跑
```

但生产 ETL Pipeline 要考虑：

```text
Input Range
↓
Extract
↓
Transform
↓
Load
↓
Checkpoint / Watermark
↓
Success
```

失败时：

```text
Retry
```

历史数据：

```text
Backfill
```

重复执行：

```text
Idempotent
```

所以真正的数据管道需要三个额外属性：

```text
可重复运行
可恢复
可追踪
```

---

## 六、你会发现它和 Module 7 Indexer 很像

你可能已经注意到了。

Module 7 的 Indexer：

```text
RPC / Node
↓
Fetcher
↓
Parser
↓
Decoder
↓
Normalizer
↓
Writer
```

Module 9 ETL：

```text
Source Table
↓
Extract
↓
Transform
↓
Load
↓
Target Table
```

看起来确实很像。

这是正常的。

因为从更高层看，它们都属于：

> Data Pipeline。

区别主要在于输入和业务职责。

【Blockchain Data Engineer 视角】

Indexer 更关注：

```text
Blockchain Source
↓
Reliable On-chain Fact
```

例如：

```text
Raw Log
↓
ERC20 Transfer
↓
fact_token_transfers
```

ETL 更关注：

```text
Existing Data
↓
Another Data Model / Dataset
```

例如：

```text
fact_token_transfers
↓
dws_wallet_token_daily_flow
```

所以可以这样记：

```text
Indexer
解决“链上事实怎么可靠地进入数据平台”

ETL
解决“平台里的数据怎么持续流向新的数据模型”
```

两者底层工程思想会大量重合：

```text
Checkpoint
Idempotency
Backfill
Retry
Recovery
```

这不是课程重复，而是同一种 Data Engineering 原理在不同层次再次出现。

---

## 七、ETL 的 E：Extract

Extract 回答：

> 从哪里拿数据，以及这一次拿哪一段数据？

例如：

```text
Source
= fact_token_transfers
```

但还不够。

还必须知道：

```text
block 20,000,000 ~ 20,010,000
```

或者：

```text
2026-09-14 00:00:00
~
2026-09-15 00:00:00
```

所以 Extract 不只是：

> SELECT 数据。

还包括：

> 定义 Input Range。

这一点后面会非常重要。

---

## 八、ETL 的 T：Transform

Transform 回答：

> Source 数据要怎样变成 Target 所需要的语义？

例如 Source 是：

```text
one Transfer Event
```

Target 是：

```text
one wallet + one token + one day
```

那么 Transform 就需要：

```text
Transfer Event
↓
拆 sender / receiver role
↓
按 wallet + token + date
↓
SUM(amount)
COUNT(transfer)
```

这里可以明显看到 Module 8 的作用。

如果你不知道：

```text
Source Grain
Target Grain
```

Transform 很容易写错。

所以：

> 数据建模决定“数据应该长什么样”；ETL 负责把一种 Grain 稳定地变成另一种 Grain。

---

## 九、ETL 的 L：Load

Load 看起来最简单：

> 写进目标表。

但实际上也有很多问题。

例如 Target：

```text
dws_wallet_token_daily_flow
```

Unique Key：

```text
(chain_id, date, wallet_address, token_address)
```

假设同一天的 Job 重跑一次。

如果你直接：

```sql
INSERT
```

可能得到两条重复记录。

所以 Load 必须考虑：

```text
INSERT?
UPSERT?
DELETE + INSERT?
MERGE?
Overwrite Partition?
```

这些选择背后的核心依然是：

> Target Grain + Unique Key。

Module 8 在这里继续发挥作用。

---

## 十、为什么 ETL 必须知道自己的处理范围？

假设今天 Job 要计算：

```text
2026-09-14
```

那么一个好的 Job 应该明确：

```text
Input Range
= 2026-09-14 00:00
  ~
  2026-09-15 00:00

Target Partition
= 2026-09-14
```

而不是：

```text
SELECT 所有历史 Transfer
↓
重新计算全部历史
```

虽然全量跑在逻辑上可能正确，但随着数据量增加，成本会迅速上升。

这就是为什么后面我们要区分：

```text
Full Load
Incremental Load
Backfill
```

---

## 十一、传统 ETL 和 Blockchain ETL 有什么区别？

核心结构其实相同：

```text
Source
↓
Extract
↓
Transform
↓
Load
↓
Target
```

区别主要出现在 Source 的性质。

银行业务系统的数据一般更接近：

```text
已经提交的数据
→ 通常不会因为“链重组”突然消失
```

Blockchain 数据则存在：

```text
最新 Block
↓
可能 Reorg
↓
之前认为 canonical 的数据失效
```

此外还有：

```text
RPC Rate Limit
Node Lag
Missing Block
Provider Failure
Late Finality
Backfill
```

所以 Blockchain ETL 特别需要重视：

```text
可重放
可修正
可回滚
可重复执行
```

不过这些并不是一种完全新的 ETL。

更准确地说：

> Blockchain ETL = 通用数据工程原则 + Blockchain Source 的特殊一致性约束。

---

## 十二、本课先建立一个最重要的心智模型

以后看到任何 ETL Job，都先问：

```text
Source 是什么？

Input Range 是什么？

Transform 做什么？

Target 是什么？

Target Grain 是什么？

Unique Key 是什么？

重复执行会怎样？

失败以后从哪里恢复？
```

如果这些问题回答不了，这个 Job 通常还不能算一个可靠的数据管道。

---

# 第一课理解检查

先做第一题。

假设我们每天运行一个 Job：

```text
fact_token_transfers
↓
统计每个 Wallet 每种 Token 每天的收发金额
↓
dws_wallet_token_daily_flow
```

请你回答：

1. 这里的 Extract 是什么？
2. Transform 是什么？
3. Load 是什么？
4. 为什么“写一个 GROUP BY SQL”还不能算完整的 ETL Pipeline？

### 用户原回答（理解检查 1）

1.这里的 Extract 是  fact\_token\_transfers
2.Transform  是 统计每个 Wallet 每种 Token 每天的收发金额
3.Load 是dws\_wallet\_token\_daily\_flow
4.因为单独的一个 GROUP BY SQL，它只是 Transform 这个阶段的一部分。
完整的 ETL Pipeline 是一个综合的数据工程，需要稳定的输入和稳定的输出，还需要明确的输入范围，以及数据幂等性（Idempotence）。

### 老师判断与校准

通过。

你的四点判断都正确，而且第 4 点已经抓到了 ETL 和“单段 SQL”的本质区别。

有两处我帮你固定得更严谨一些。

第 1 点，`Extract = fact_token_transfers` 方向正确，但完整表达应该是：

```text
Extract
= 从 fact_token_transfers 读取本次 Job 对应的 Input Range
```

因为 Extract 不只是“数据源是谁”，还包括：

```text
这次到底读哪一段数据
```

例如：

```text
2026-09-14 00:00
~
2026-09-15 00:00
```

或者：

```text
block 20,000,000
~
20,010,000
```

所以以后可以把：

> Extract = Source + Input Range

先作为一个基础心智模型。

第 2、3 点正确：

```text
Transform
= 按 wallet + token + date
  统计 received / sent amount
  以及后续可能的 count 等指标
```

```text
Load
= 把结果写入 dws_wallet_token_daily_flow
```

第 4 点尤其重要。你已经从“SQL 逻辑”切换到了“Pipeline 工程”的视角。

单独一个：

```sql
GROUP BY
```

主要回答：

> 数据怎么算？

但一个完整 ETL Pipeline 还必须回答：

```text
从哪里读？
读哪一段？
怎么算？
写到哪里？
重复跑会怎样？
失败后怎么恢复？
历史怎么补？
什么时候算成功？
```

所以可以把你的回答进一步固定成一句话：

> **SQL 负责数据处理逻辑，ETL Pipeline 负责让这套逻辑能够稳定、可重复、可恢复地持续运行。**

这一题通过。

继续第二题。

假设每天凌晨 1 点运行一个 Job，处理“昨天”的数据：

```text
fact_token_transfers
↓
dws_wallet_token_daily_flow
```

某一天 Job 运行到一半失败了，已经写入了一部分 DWS 数据。

第二天你重新运行昨天这个 Job。

请回答：

1. 如果只是继续执行普通 `INSERT`，可能出现什么问题？
2. 为什么这里需要 Idempotency？
3. 你觉得可以用什么方式保证同一天的 Job 重跑后，最终结果仍然只有一份正确数据？

### 用户原回答（理解检查 2）

一、如果只是继续执行普通 insert，可能出现什么问题呢？



那可能会有部分数据重复，因为之前的 Job 已经写入了一部分 DWS 数据。如果只是普通 insert，那么已经写入的部分数据就会有重复的情况。

二、这里需要 idempotency，就是为了防止多次插入会有重复数据的现象。我们需要保证多次插入，但是数据结果能保持一致，而不会出现重复

3. 用什么方式保证重跑后最终结果仍只有一份正确的数据？

可以在重跑的时候检查：根据关键字来检查表里是否已经存在了数据。如果存在的话，可以先删除，然后再插入；如果不存在的话，可以直接插入。

### 老师判断与校准

这题通过。

你的三点都对，而且第 3 点已经开始进入真正的 Load Strategy 了。

第 1 点，你判断准确：普通 `INSERT` 重跑会导致“之前已经成功写入的那部分”再次写入，从而出现重复。对于 DWS 这种聚合结果，重复不是简单多一行那么轻微，它会直接让金额、次数等指标失真。

第 2 点，对 Idempotency 的理解也正确：

> 同一个 Job、同一个输入范围，重复执行一次或多次，最终 Target 状态应该一致。

这就是 ETL 里的幂等。

第 3 点，你提出的“先检查，再删除，再插入”是可行思路，但我帮你再工程化一点。

如果这个 DWS 是按天处理，例如：

```text
Target Grain
= (chain_id, date, wallet_address, token_address)
```

那么更稳妥的方式通常不是“逐行先查有没有”，而是按本次 Job 的处理范围整体处理。

例如昨天的数据：

```text
date = 2026-09-14
```

重跑时可以：

```text
DELETE FROM dws_wallet_token_daily_flow
WHERE date = '2026-09-14';

INSERT INTO dws_wallet_token_daily_flow
SELECT ...
FROM fact_token_transfers
WHERE ...
GROUP BY ...
```

这叫：

```text
Delete + Insert
```

它的优点是逻辑简单，而且非常符合“一个 Job 负责一个明确 Partition / Date Range”的思想。

另外还有几种常见方案：

```text
UPSERT / MERGE
Overwrite Partition
Unique Key + Conflict Update
```

具体选哪种，要看 Target 表、数据库能力和 Grain。

你这里最需要固定的一句是：

> 幂等不是“避免重复 insert”这么窄，而是保证相同 Input Range 重跑后，Target 最终状态仍然正确且一致。

接下来第三题。

假设这个 ETL 每天处理昨天的数据，但有一天因为故障连续停了 3 天：

```text
2026-09-10 未处理
2026-09-11 未处理
2026-09-12 未处理
```

到 2026-09-13 系统恢复。

请回答：

1. 这是正常 Incremental Load，还是 Backfill？
2. 你会让一个 Job 一次处理 3 天，还是把 3 天拆成 3 个独立处理范围？为什么？
3. 这种设计和你在 Module 7 里学过的 Backfill Cursor / Checkpoint 有什么相似之处？

### 用户原回答（理解检查 3）

1. 这是 backfill。
2. 我会让 job 把三天拆成三个独立处理范围。因为这样的话，独立处理可以把 job 简化为一次处理一天，同一个 job 就可以重复使用，达到最大范围的重复使用。
3. 相似之处在于，同一套处理逻辑可以面对不同的情况而重复使用，保持同一份口径，减少代码的复杂程度

### 老师判断与校准

前两问通过，第 3 问需要再校准一下。

1. 判断为 Backfill：正确。

因为这 3 天本来应该由正常增量任务处理，但由于故障形成了历史缺口，现在是在补历史数据。

2. 拆成 3 个独立处理范围：正确。

你的理由也成立：让同一个 Job 始终保持统一接口，例如：

```text
process(date)
```

那么：

```text
process(2026-09-10)
process(2026-09-11)
process(2026-09-12)
```

比专门写一个“处理连续 3 天”的特殊逻辑更容易复用、重试和验证。

不过还有一个更重要的工程原因：

> 独立处理范围可以形成更小的 failure domain。

例如：

```text
09-10 成功
09-11 失败
09-12 未开始
```

恢复时只需要从 `09-11` 继续，不必把前三天全部重跑。

第 3 问你回答的是“共用同一处理逻辑”，这个结论没错，而且和 Module 7 的设计原则确实相通。但这道题真正想检查的是 `Cursor / Checkpoint`。

Module 7 里：

```text
Backfill Cursor
=
当前已经处理到哪个 block
```

例如：

```text
start_block = 20,000,000
end_block   = 21,000,000
cursor      = 20,500,000
```

Module 9 的 ETL 也一样。

如果按天 Backfill：

```text
2026-09-10 ✅
2026-09-11 ✅
2026-09-12 ⏳
```

那么系统需要知道：

```text
已经成功完成到哪里
下一次应该从哪里继续
```

这个角色就是：

```text
Watermark / Cursor / Checkpoint
```

所以两者真正的共同点是：

> **都需要持久化“处理进度边界”，从而支持断点续跑，并避免重复或遗漏处理。**

再加上你说的另一个共同点：

> Realtime / Incremental / Backfill 最好复用同一套核心处理逻辑，以保持数据口径一致。

这两点合起来就完整了。

最后做一个很短的校准题：

```text
假设 09-10 和 09-11 已经成功完成，
09-12 处理一半失败。
```

请回答两点：

1. Checkpoint / Cursor 应该记录到哪里？
2. 系统恢复后，应该从哪里继续？

### 用户原回答（校准题）

1. Checkpoint course 应该记录到 0911。
2. 系统恢复后，应该从 0912号开始处理

### 老师判断与校准、结课判定

这次校准通过。

按我们当前“按天一个处理范围”的设计：

```text
Checkpoint = 2026-09-11
```

因为 `09-11` 是最后一个完整成功的处理范围。

系统恢复后：

```text
从 2026-09-12 重新开始
```

这里还有一个重要边界需要记住：

- 如果 Checkpoint 只记录到“天 / Partition”级别，那么 `09-12` 即使已经跑了一半，也要整天重跑；
- 如果以后设计了更细粒度的日内 Cursor，才可以从 `09-12` 内部某个更细的位置继续。

所以固定一句：

> **Checkpoint 记录的是最后一个确认成功的处理边界，而不是“程序曾经运行到哪里”。**

到这里，Module 9 第 1 课的理解检查已经通过，可以正式结束。

## 结课判定

四轮回答与必要校准已完成；理解检查通过。Module 9 第 1 课正式结束。