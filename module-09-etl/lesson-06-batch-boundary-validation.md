# 第6课｜Batch Job 的输入输出边界与可验证性：怎么证明这一批数据真的处理对了

【Lesson Contract】
所属 Module：Module 9 — ETL。
本课核心问题：
> 一个 Batch ETL Job 显示 SUCCESS，为什么仍然不能证明数据处理正确？我们怎样定义一批任务的 Input Boundary、Output Boundary 和 Validation，让一次 ETL 真正“可验证”？

学完本课，你应该能够：
- 明确定义一个 Batch Job 的 Input Range；
- 明确一个 Processing Unit 应该产生什么 Target Range；
- 理解“程序执行成功”与“数据正确”不是一回事；
- 区分 Execution Success 和 Data Validation；
- 设计最基础的 Row Count、Range、Key、Aggregate Validation；
- 理解为什么 Checkpoint 必须在 Validation 之后推进；
- 为 `dws_wallet_token_daily_flow` 设计一套最小可验证 Batch Job。

本课不展开：完整 Data Quality Framework、异常检测和统计学监控、SLA / SLO、Great Expectations / dbt tests 等具体框架、自动 Repair System。这些主要留到 Module 12。

---

## 一、先看一个“任务成功但数据错误”的例子

假设 Daily ETL 处理 2026-09-15，Extract / Transform / Load 都成功，exit code = 0，Scheduler 显示 SUCCESS，于是 checkpoint = 09-15。

但第二天分析师发现 09-15 USDC transfer volume 只有平时的 20%。后来检查发现 Indexer 当时只同步到当天 18:00，所以 ETL 实际只处理 00:00 ~ 18:00，缺了 18:00 ~ 24:00。

这里程序没有异常，SQL 没有报错，数据库也成功写入，但数据是错的。

> Job Success ≠ Data Correctness。

## 二、Execution Success 只证明“程序跑完了”

【Data Engineer 视角】Execution Success 通常只能证明：程序没有 Crash、SQL 没有报错、RPC 没有返回 fatal error、Load transaction 成功。它不能自动证明输入完整、Transform 正确、输出完整、没有重复、没有缺失、业务数值合理。

所以生产 ETL 还需要另一层：Validation。

## 三、先明确 Input Boundary

一个 Batch Job 如果不能回答“这次到底处理了哪一段 Source？”，它就很难验证。

例如：

```text
run_job(
    start = 2026-09-15 00:00:00,
    end   = 2026-09-16 00:00:00
)
```

Input Range = `[start, end)`，也就是：

```text
>= 09-15 00:00
<  09-16 00:00
```

最好使用半开区间 `[start, end)`，因为相邻 Processing Unit 可以自然衔接而不重叠。

## 四、Blockchain ETL 的 Input Boundary 不一定是时间

【Blockchain Data Engineer 视角】Input Range 可能是 block_number，例如 `[20,000,000, 20,010,000)`；也可能是 block_time，例如 `[2026-09-15, 2026-09-16)`；甚至可以同时保存 start_block / end_block / start_time / end_time。

Block Range 更接近 Blockchain Source 的天然顺序；Time Range 更接近 Daily Warehouse 的业务分区。所以不同 Pipeline 的 Processing Coordinate 可以不同。这和 `Indexer checkpoint = block`、`Daily ETL checkpoint = date` 是同一个思想。

## 五、Input Boundary 必须是显式参数

不建议写：

```sql
WHERE date = CURRENT_DATE - 1
```

因为真实 Input Range 是隐含且动态的，对 Retry、Backfill、Audit、Replay 都不友好。

更好的方式是：

```text
run_job(start, end)
```

Scheduler 只负责决定今天传什么 start/end，真正的数据处理逻辑只认 Input Range。

## 六、这和 Backfill 为什么天然统一？

正常增量和 Backfill 底层 Extract / Transform / Load 可以完全相同，区别只是 Input Range。

> Normal Incremental 和 Backfill 不应该写两套 Transform / Load。

## 七、有了 Input Boundary，还要有 Output Boundary

假设 Input 是 `fact_token_transfers`，Input Range 是 `block_time >= 09-15` 且 `< 09-16`，Target 是 `dws_wallet_token_daily_flow`，那么 Output Boundary 通常是：

```text
date = 09-15
```

这就是明确的 Input → Output Mapping。

## 八、为什么 Output Boundary 也必须明确？

因为 Load 时必须知道“这次 Job 允许修改 Target 的哪一部分？”例如 `run_job(09-15)` 理论上应该只修改 `target date = 09-15`。如果 09-14、09-13 的数据也变化，可能就是 Bug。

所以 Output Boundary 既是 Load Scope，也是 Validation Scope。

## 九、一个可靠 Batch Job 应该有“边界合同”

一次 Job 可以看成：

```text
Input Contract
+
Transform Contract
+
Output Contract
```

例如：

```text
Job:
dws_wallet_token_daily_flow

Input:
fact_token_transfers

Input Range:
block_time ∈ [09-15, 09-16)

Transform:
按 chain_id / wallet / token / date 聚合

Output:
dws_wallet_token_daily_flow

Output Range:
date = 09-15
```

## 十、接下来才谈 Validation

Validation 应该尽量 Machine-checkable：Job 自己能够判断这一批输出至少是否满足基本正确性条件。

## 十一、Validation 1：Range Validation

输出有没有超出允许范围？例如处理 09-15，那么预期 `MIN(date) = 09-15`、`MAX(date) = 09-15`。如果出现 09-14 或 09-16，说明 Output Boundary 可能被污染。

## 十二、Validation 2：Row Count

Row Count 是非常便宜的 sanity check。如果正常每日约 120,000 行，某天突然只有 8,000 行，即使 Job SUCCESS，也应该引起怀疑。

## 十三、Row Count 为什么不能单独证明正确？

因为数量正确，内容仍然可能错误。例如本来 A / B / C，结果 A / A / C，数量仍然是 3。

> Row Count 是必要但不充分的验证方式。

## 十四、Validation 3：Unique Key Validation

`dws_wallet_token_daily_flow` 的 Unique Key 可以是：

```text
chain_id
+ date
+ wallet_address
+ token_address
```

重复键检查正确结果应该是 0 rows。这能验证 Load 没有破坏 Target Grain / Unique Key。

## 十五、Validation 4：Aggregate Reconciliation

这是比 Row Count 更强的验证。根据数据模型定义，在 Source 和 Target 之间建立明确的守恒关系或对账关系。银行数据工程中通常称为 Reconciliation / 对账。

## 十六、银行系统类比

如果银行当天 Source 有 1,000,000 笔交易、总金额 20 亿元，而生成的 customer_daily_flow 总金额只有 12 亿元，即使 Job SUCCESS，也说明数据处理有问题。Blockchain ETL 完全一样。

## 十七、Source Count 和 Target Count 往往不能直接相等

Source `fact_token_transfers` 有 100,000 rows，Target `dws_wallet_token_daily_flow` 可能只有 15,000 rows，这不是错误。因为 Transform 改变了 Grain：Source Grain 是 one transfer event，Target Grain 是 one chain + wallet + token + day。

所以 `Source Count != Target Count` 很正常。

## 十八、Validation 必须理解 Grain

如果不知道 Source Grain 和 Target Grain，就没办法设计正确的验证。真正应该验证的是：Transform 后应该满足什么不变量？这叫 Invariant。

## 十九、什么是 Invariant？

Invariant 是：在 Transform 前后，虽然数据结构变化，但某些业务事实必须保持成立。例如总 Token Amount 的守恒关系、每个 Source Transfer 对钱包维度的贡献、Unique Key 不重复。

## 二十、一个 wallet flow 的验证例子

一笔 Alice → Bob 的 100 USDC Transfer，Source 是 1 row；如果 DWS 采用 wallet perspective，可能产生 Alice sent=100 和 Bob received=100 两个贡献。

因此不能直接验证 `source amount = target sent + target received`。正确的验证应该按业务语义分别设计，例如：

```text
SUM(sent_amount) = Source transfer amount
SUM(received_amount) = Source transfer amount
```

Validation 必须建立在数据模型语义上，而不是随便比较两个 SUM。

## 二十一、验证本质上是在证明 Contract

```text
Given:
Input Range R

When:
Transform T

Then:
Output Range O
must satisfy
Invariant I
```

Validation 不是 ETL 外面的附加功能，而是 Job Contract 的一部分。

## 二十二、Checkpoint 为什么必须放在 Validation 后面？

```text
Extract
↓
Transform
↓
Load
↓
Validate
↓
Advance Checkpoint
```

如果 Load 成功，但 Validation 发现 Target row_count = 0 或 duplicate unique keys = 5,000，checkpoint 不能推进，因为从数据正确性角度，这个 Processing Unit 并没有完成。

```text
Load Success
≠
Processing Unit Complete
```

真正应该是：

```text
Load Success
+
Validation Pass
=
Processing Unit Complete
```

然后才能 Advance Checkpoint。

## 二十三、这会改变我们对 Checkpoint 的定义

> Checkpoint = 最后一个已经成功执行并通过必要 Validation 的 Processing Unit 边界。

## 二十四、一个完整的 Batch Run Record

生产环境里通常值得记录一次 Job Run 的元数据，例如：

```text
job_name
run_id
input_start
input_end
output_partition
source_row_count
target_row_count
validation_status
started_at
finished_at
status
```

这样以后可以回答：09-15 当时到底处理了什么？

## 二十五、为什么 run_id 很有价值？

同一个 Processing Unit 可能执行很多次：第一次 FAILED，第二次 SUCCESS，之后因为 Late-arriving 又 REPROCESS。

Checkpoint 是当前状态，而 Job Run History 是执行历史。

## 二十六、Checkpoint Table 和 Job Run Table 是不同概念

```text
Checkpoint = State
Job Run = History
```

这个区别与 State vs History 的思路相同。

## 二十七、Blockchain Job 还可以保存 Block Boundary

Daily ETL 虽然按日期处理，但可以额外记录 min_block / max_block。以后 Reorg / Repair 时，可以更容易把 changed block 映射到 Affected Job Run / Partition。

## 二十八、不要只保存“成功 / 失败”

更好的日志至少应该回答：处理了什么？处理多少？输出到哪里？验证了什么？验证结果怎样？即：Input / Output / Metrics / Validation / Status。

## 二十九、最小可验证 ETL 模型

```text
Read Checkpoint
      ↓
Determine Input Range
      ↓
Record Job Run START
      ↓
Extract
      ↓
Record Source Metrics
      ↓
Transform
      ↓
Idempotent Load
      ↓
Record Target Metrics
      ↓
Validate
      ↓
PASS ?
 ├─ No
 │   ↓
 │ FAILED
 │ Checkpoint 不推进
 │
 └─ Yes
     ↓
   Advance Checkpoint
     ↓
   Job Run SUCCESS
```

## 三十、针对 `dws_wallet_token_daily_flow` 设计一次 Job

Target：`dws_wallet_token_daily_flow`
Processing Unit：one day
Input：`fact_token_transfers`
Input Range：`block_time >= 2026-09-15 00:00` 且 `< 2026-09-16 00:00`
Transform：GROUP BY `chain_id / date / wallet_address / token_address`
Output Boundary：`date = 2026-09-15`
Load：Replace 09-15 Partition

Validation 至少可以做：
1. Target date 只能是 09-15
2. Unique Key 不重复
3. target_row_count > 0（如果业务允许 0，需要另定规则）
4. SUM(sent_amount) 与 Source outbound amount 对账
5. SUM(received_amount) 与 Source inbound amount 对账

全部通过后，checkpoint 才从 09-14 推进到 09-15。

## 三十一、为什么这是“可重跑 + 可验证”？

因为 Input Range 明确，所以可以重放；Load Idempotent，所以重放安全；Output Boundary 明确，所以知道允许修改哪里；Validation 明确，所以知道结果是否满足 Contract；Checkpoint 在验证后推进，所以失败不会被误认为完成。

## 三十二、到这里 Module 9 的核心链条已经很完整

```text
Lesson 1
ETL 是长期 Pipeline

Lesson 2
Full / Incremental / Backfill
→ 处理哪一段

Lesson 3
Watermark / Cursor / Checkpoint
→ 处理到哪里

Lesson 4
Idempotent Load
→ 重跑是否安全

Lesson 5
Late-arriving / Reorg
→ 历史为什么需要重新打开

Lesson 6
Input / Output Boundary + Validation
→ 怎么证明这一批处理正确
```

---

## 本课理解检查

问题一：

```text
run_job(
  start = 2026-09-15 00:00,
  end   = 2026-09-16 00:00
)
```

Source：`fact_token_transfers`
Target：`dws_wallet_token_daily_flow`

1. Input Boundary 是什么？
2. Output Boundary 应该是什么？
3. 为什么不能只写成 `WHERE date = yesterday`？

问题二：

```text
Extract ✓
Transform ✓
Load ✓
exit code = 0
```

但是 Validation 得到：

```text
duplicate unique keys = 3,000
```

1. 这个 Job 是否应该算 Processing Unit 完成？
2. Checkpoint 能不能推进？
3. 为什么？

问题三：Source 是 100,000 token transfer events，Target 是 18,000 wallet-token-day rows。为什么不能简单判断 `source_row_count != target_row_count` 所以 ETL 出错了？请从 Grain 和 Transform 的角度解释。

---

## 用户回答

问题 1：

1. Input Boundary 是 20260915
2. Output Boundary 是 20260915
3. 因为 yesterday 是个动态的，它会根据每天的日期不同动态改变。所以如果需要指定日期的话，最好是用明确的日期数字。

问题二：

1. 这个 Job 不应该算 Processing Unit 完成
2. Checkpoint 不能推进
3. 因为虽然程序正常执行完成、没有报错，但在数据验证上它还是有问题的。所以 Checkpoint的推进不只看 Job 正常完成，也需要数据能验证通过

问题 3：

Source Row Count 和 Target Row Count 不相等，并不代表 ETL 出错了。

因为 Source 和 Target 它们的 Grain 是不一样的，而且 Source 表里面的数据需要经过 Transform 这一整个 ETL 的过程，并且经过Aggregated操作之后， Load 到 Target 表里面，并不能用数据量的比较来判断 ETL 的过程是否正确，二是需要分析 transform 的过程，统计相关指标来验证数据是否正确。

---

## 教师判断与校准

三题均回答正确。

问题一做一个精度校准：Input Boundary 不是单独“2026-09-15”，而是显式半开区间 `[2026-09-15 00:00, 2026-09-16 00:00)`；Output Boundary 才是 `date = 2026-09-15`。这一区分很重要，因为 Input Boundary 描述 Source Range，Output Boundary 描述 Target Scope。

问题二判断正确：Execution Success 不等于 Processing Unit Complete；必须在必要 Validation 通过后，才能推进 Checkpoint。

问题三判断正确：Source 与 Target Grain 不同，Transform 中的 Aggregate 会改变行数，所以不能用 Source Row Count 与 Target Row Count 是否相等来证明正确性；应根据数据模型语义设计 Invariant / Reconciliation 指标。

**正式结课判断：Module 9 第 6 课理解检查通过，必要校准完成，本课正式结束。**