# 第2课｜Full Load、Incremental Load 与 Backfill：ETL 到底处理哪一段数据？

【Lesson Contract】

所属 Module：Module 9 — ETL。

本课核心问题是：

> 一个 ETL Job 每次运行时，到底应该处理“全部数据”，还是只处理“新增的一段数据”？

学完这一课，你应该能够：

- 区分 Full Load、Incremental Load、Backfill；
- 判断不同场景应该使用哪一种；
- 理解为什么 Incremental Load 必须有明确的增量边界；
- 理解 Backfill 和正常增量任务为什么应该复用同一套 Transform / Load 逻辑；
- 初步设计一个“按日期增量”的 Blockchain ETL Job。

本课暂时不深入：

- Watermark / Cursor / Checkpoint 的各种实现细节；
- DAG / Airflow；
- Partition 的物理存储优化；
- Reorg 修复框架。

这些后面继续展开。

---

## 一、先从上一课的问题继续

我们已经有：

```text
fact_token_transfers
        ↓
       ETL
        ↓
dws_wallet_token_daily_flow
```

Target Grain 是：

```text
chain_id
+ date
+ wallet_address
+ token_address
```

假设 `fact_token_transfers` 已经保存了 Ethereum 最近三年的 Transfer 数据。

现在到了：

```text
2026-09-15
```

我们今天只想生成：

```text
2026-09-14
```

这一天的 Wallet Token Flow。

那么问题来了。

ETL 有三种可能的做法。

方案 A：

```text
读取过去三年的全部 Transfer
↓
重新计算三年所有 daily_flow
```

方案 B：

```text
只读取 2026-09-14 的 Transfer
↓
计算 2026-09-14 daily_flow
```

方案 C：

```text
读取 2026-03-01 ~ 2026-03-31
↓
重新生成三月份 daily_flow
```

这三种做法分别对应：

```text
Full Load
Incremental Load
Backfill
```

这就是今天这一课。

---

## 二、Full Load：从头重新计算

Full Load 最容易理解。

它的含义基本就是：

> 把完整 Source 范围重新读一遍，并重新生成完整 Target。

例如：

```text
fact_token_transfers

block 0
~
latest block
```

全部读取。

然后：

```text
GROUP BY
wallet
token
date
```

最后重新生成整个：

```text
dws_wallet_token_daily_flow
```

可以画成：

```text
完整 Source
████████████████████████

        ↓

      Transform

        ↓

完整 Target
████████████████████████
```

这就是 Full Load。

---

## 三、Full Load 并不是“错误做法”

这里很容易产生一个误解：

> Full Load 性能很差，所以 Full Load 不应该使用。

这不对。

【Data Engineer 视角】

Full Load 是一种非常重要的工具。

例如表里只有：

```text
10 万行
```

你完全可以每天：

```text
TRUNCATE target
↓
重新 SELECT
↓
INSERT target
```

这种设计甚至可能比复杂的增量逻辑更可靠。

因为它最大的优势是：

```text
逻辑简单
状态少
容易理解
容易重跑
不容易漏数据
```

银行数据仓库里你应该也见过类似模式。

例如某张很小的维表：

```text
dim_branch
dim_currency
dim_product_type
```

每天直接全量刷新，完全合理。

所以：

> Incremental Load 并不天然比 Full Load “高级”。

工程设计看的不是高级不高级，而是：

```text
数据量
成本
正确性
复杂度
恢复能力
```

---

## 四、Full Load 的问题什么时候出现？

假设：

```text
Ethereum Transfer
```

已经积累：

```text
几十亿条
```

你的任务每天只是算：

```text
昨天的数据
```

如果每天都：

```text
扫描过去几年所有 Transfer
```

那会发生什么？

计算量：

```text
历史数据量不断增加
```

而你真正需要处理的数据：

```text
每天可能只有新增的 0.1%
```

这时候 Full Load 就变成：

```text
99.9% 的计算
都在重复做已经完成的工作
```

例如：

第一天：

```text
扫描 1 TB
```

一年后：

```text
扫描 5 TB
```

两年后：

```text
扫描 10 TB
```

但每天新增可能只有：

```text
20 GB
```

那么显然应该问：

> 为什么我要重新扫描那 9.98 TB？

这就是 Incremental Load 出现的原因。

---

## 五、Incremental Load：只处理新增范围

Incremental Load 的核心思想非常简单：

> 已经处理过的数据不重复处理，只处理新的 Input Range。

例如昨天已经处理到：

```text
2026-09-13
```

今天运行 Job：

```text
Input Range

2026-09-14 00:00:00
~
2026-09-15 00:00:00
```

读取：

```text
fact_token_transfers
```

只计算：

```text
2026-09-14
```

然后写入 Target：

```text
date = 2026-09-14
```

数据流变成：

```text
历史 Source
████████████████████

新增
                    ██
                     ↓
                 Transform
                     ↓
Target
██████████████████████
```

这就是 Incremental Load。

---

## 六、Incremental 的本质不是“WHERE date = yesterday”

这一点非常重要。

很多人第一次理解 Incremental ETL，会认为：

```sql
WHERE date = yesterday
```

就是 Incremental。

这只是表面。

真正的问题是：

> 系统怎么知道这一段数据应该被处理？

也就是说 Incremental Load 必须回答：

```text
上次处理到哪里？

这次从哪里开始？

这次处理到哪里结束？
```

例如：

```text
last_success_date
=
2026-09-13
```

那么：

```text
start
=
2026-09-14 00:00

end
=
2026-09-15 00:00
```

所以真正的 Incremental 模型是：

```text
Processed Range
████████████████

New Range
                ██
                ↑
              Job
```

而不是简单地：

```text
SELECT yesterday
```

后面我们学习 Watermark / Checkpoint 时，就是解决：

> “系统怎么记住处理边界？”

---

## 七、这里和你的 Mini Indexer 是同一种思想

【视角：Indexer vs ETL】

Module 7 里你已经做过：

```text
block 100
block 101
block 102
...
```

Indexer 保存：

```text
checkpoint = 102
```

下一次从：

```text
103
```

继续。

ETL 其实完全一样。

只是坐标轴不同。

Indexer 可能使用：

```text
block_number
```

ETL 可能使用：

```text
date
timestamp
ID
updated_at
partition
```

比如：

```text
Indexer

checkpoint = block 20,000,000
```

ETL：

```text
checkpoint = 2026-09-14
```

它们背后的抽象其实都是：

> 我已经可靠处理到哪里？

这是一个非常通用的数据工程概念。

---

## 八、那么 Backfill 到底是什么？

现在假设生产系统每天正常运行：

```text
09-10
09-11
09-12
09-13
09-14
```

突然发现：

```text
09-01 ~ 09-05
```

这几天的数据，因为 decoder bug 算错了。

怎么办？

显然你不能等正常 Incremental Job 慢慢再走回来。

你需要明确指定：

```text
start_date = 2026-09-01
end_date   = 2026-09-06
```

然后重新执行这段历史数据。

这就是：

> Backfill。

可以理解成：

```text
正常 Incremental

██████████████████████→

历史错误区间

      XXXXX

        ↑
     Backfill
```

所以 Backfill 本质上是：

> 对一个明确指定的历史 Input Range 重新执行数据管道。

---

## 九、Backfill 不只是“补缺失数据”

中文里“补数”容易让人误以为：

> Backfill = 数据缺了以后补进去。

其实它的范围更广。

Backfill 可能是因为：

```text
历史数据缺失
```

也可能因为：

```text
Transform 逻辑修复
```

例如以前：

```text
USDC decimals 算错
```

还可能因为：

```text
新增字段
```

例如 Target 新增：

```text
amount_usd
```

需要计算过去一年。

也可能因为：

```text
Decoder 修复
```

甚至：

```text
业务规则改变
```

因此更准确地说：

> Backfill = 对历史范围重新执行数据处理逻辑。

---

## 十、一个非常重要的设计原则

假设你有两个程序。

正常任务：

```python
run_daily_etl()
```

历史补数：

```python
run_backfill_etl()
```

然后两个程序：

```text
Transform 逻辑不同
SQL 不同
Load 逻辑不同
```

这是一个危险设计。

为什么？

因为慢慢会出现：

```text
Realtime / Daily 结果是一套逻辑

Backfill 结果是另一套逻辑
```

最后同一张表里的历史数据和新数据口径不一致。

你在 Module 7 已经碰过完全相同的问题。

当时我们强调：

```text
Realtime
Backfill
Reorg Replay
```

应该复用：

```text
process_block()
```

ETL 也是同样原则。

应该设计成：

```text
run_etl(start, end)
```

正常增量：

```text
run_etl(
    2026-09-14,
    2026-09-15
)
```

Backfill：

```text
run_etl(
    2026-03-01,
    2026-04-01
)
```

核心处理逻辑：

```text
Extract
Transform
Load
```

是一样的。

变化的只是：

```text
Input Range
```

这是这一课非常重要的工程结论：

> Normal Incremental 与 Backfill 应尽量复用同一套处理逻辑，只让处理范围不同。

---

## 十一、Full / Incremental / Backfill 真正的关系

现在我们可以重新整理。

它们并不是三个完全不同的 ETL 系统。

实际上可以统一成：

```text
run_job(start, end)
```

Full Load：

```text
start = earliest
end   = latest
```

Incremental Load：

```text
start = last_checkpoint
end   = new_boundary
```

Backfill：

```text
start = user_defined_start
end   = user_defined_end
```

你会发现：

> Transform 根本不需要知道自己是在 Full、Incremental 还是 Backfill。

它只需要知道：

```text
给我这一段 Input Range
```

我就处理。

这是非常漂亮的数据管道设计。

---

## 十二、拿银行场景对照一下

假设银行有：

```text
fact_account_transactions
```

每天生成：

```text
dws_customer_daily_transaction
```

正常每天跑：

```text
2026-09-14
```

这是：

```text
Incremental
```

后来发现：

```text
2026-08-01 ~ 2026-08-10
```

某个业务类型漏算。

重新计算：

```text
2026-08-01 ~ 2026-08-10
```

这是：

```text
Backfill
```

如果系统刚上线，要第一次生成：

```text
2020-01-01 ~ 2026-09-14
```

历史所有数据：

这可以看成：

```text
Initial Full Load
```

你会发现区块链并没有创造一种新的 ETL。

只是 Blockchain 的历史数据：

```text
量更大
增长更快
Source 可能 Reorg
```

因此对 Range、Replay 和 Idempotency 要求更高。

---

## 十三、Blockchain 场景再加一层复杂性：Block 还是 Date？

这里开始出现一个真正属于 Blockchain Data Engineer 的问题。

假设任务按：

```text
date
```

运行。

例如：

```text
2026-09-14
```

但 Blockchain 原始数据天然是：

```text
block_number
```

例如：

```text
20,000,000
20,000,001
20,000,002
```

于是存在两个不同的处理边界：

```text
Blockchain Source Boundary
=
block_number

Analytical Target Boundary
=
date
```

【Blockchain Data Engineer 视角】

这是必须区分的。

Indexer 更自然使用：

```text
block range
```

例如：

```text
20,000,000
~
20,010,000
```

而 DWS Daily ETL 更自然使用：

```text
date range
```

例如：

```text
2026-09-14
```

所以数据平台中完全可能同时存在：

```text
Indexer Checkpoint
=
block_number
```

和：

```text
ETL Checkpoint
=
date / partition
```

两者不是冲突，而是处在不同 Pipeline 层。

---

## 十四、再看一个实际 Job

我们现在设计：

```text
wallet_token_daily_flow_job
```

参数：

```text
start_date
end_date
```

例如：

```text
start_date = 2026-09-14
end_date   = 2026-09-15
```

Extract：

```sql
SELECT *
FROM fact_token_transfers
WHERE block_time >= :start_date
  AND block_time <  :end_date;
```

Transform：

```text
Transfer
↓
sender -> sent_amount
receiver -> received_amount
↓
GROUP BY
chain_id
date
wallet
token
```

Load：

```text
Target partition
=
2026-09-14
```

Unique Key：

```text
chain_id
date
wallet_address
token_address
```

正常每日任务：

```text
run_job(09-14, 09-15)
```

Backfill 一周：

```text
run_job(09-01, 09-08)
```

Initial Load：

```text
run_job(earliest_date, 09-15)
```

你现在应该已经能看到：

> Full / Incremental / Backfill 只是同一个 Pipeline 使用不同 Input Range 的三种运行方式。

---

## 十五、但是还有一个问题：一口气 Backfill 一年可以吗？

理论上可以。

例如：

```text
2025-01-01
~
2026-01-01
```

但是生产环境通常更倾向于拆成：

```text
2025-01-01
2025-01-02
2025-01-03
...
```

或者：

```text
每天
每周
每个 partition
```

原因不仅是性能。

更重要的是：

```text
失败范围更小
重跑成本更低
容易追踪
容易验证
容易并行
```

例如一次 Backfill 365 天：

```text
跑到第 364 天失败
```

如果 Job 是一个巨大事务：

```text
可能非常难恢复
```

但如果拆成：

```text
365 个 Daily Unit
```

那么：

```text
前 363 天 Success

第 364 天 Failed
```

只需要：

```text
重跑第 364 天
```

所以又出现一个重要的数据工程原则：

> 一个可靠的 Batch Job 应该有明确且足够小的 Processing Unit。

这里暂时点到为止，后面我们讲 Partition / Checkpoint 时继续。

---

## 十六、本课最核心的心智模型

以后看到任何 Batch ETL，都先画一条时间轴：

```text
历史
──────────────────────────────→ 时间

已处理
████████████████

新增范围
                ██

未来
                  ........
```

正常任务：

```text
处理新增范围
=
Incremental
```

历史某段重新执行：

```text
        ██
        ↑
      Backfill
```

全部历史重新执行：

```text
██████████████████
↑
Full Load
```

而三者最好全部复用：

```text
             Input Range
                 ↓
             Extract
                 ↓
             Transform
                 ↓
               Load
```

真正变化的只是：

```text
start
end
```

---

# 第 2 课理解检查

这一课我们先检查三个问题。

假设：

```text
fact_token_transfers
```

已经有：

```text
2026-01-01 ~ 2026-09-14
```

的数据。

每天凌晨运行：

```text
wallet_token_daily_flow_job
```

生成前一天的：

```text
dws_wallet_token_daily_flow
```

现在发生以下三个情况。

问题一：

今天是 2026-09-15。

正常每日任务只处理：

```text
2026-09-14
```

这属于 Full Load、Incremental Load 还是 Backfill？

请同时说明为什么。

问题二：

后来发现：

```text
2026-08-01 ~ 2026-08-07
```

当时 Transform 中存在一个 bug，需要重新计算这 7 天。

你认为应该：

A. 写一个全新的 `backfill_transform()`，单独实现历史数据逻辑。

B. 仍然复用正常 `run_job(start_date, end_date)`，只是把 Input Range 指定为 08-01 ~ 08-08。

选哪个？为什么？

问题三稍微重要一点。

假设当前正常 ETL 已经成功处理到：

```text
2026-09-14
```

现在你执行了历史 Backfill：

```text
2026-08-01 ~ 2026-08-07
```

那么：

> 正常 Incremental Job 的 Checkpoint 应不应该被改回 2026-08-07？

请结合你之前 Module 7 学 Backfill Cursor / Realtime Checkpoint 的经验回答。

你可以三题一次回答，也可以逐题回答；按照现在的 Canonical 增量规则，两种方式完全等价。

---

## 用户回答

问题一 ， 属于  Incremental Load ，因为job只处理了前一天的数据， dws_wallet_token_daily_flow 之前历史数据也都存在。
问题二：  B方案。 因为不需要重新写一个job，它这个 ETL 过程中，transform、load 处理逻辑都是相同的，只是 extract Input Range 不同
问题三：正常 incremental job 的 checkpoint 不应该被改回 20260807，因为 Backfill 它有自己的 cursor，可以记录自己的进度

## 老师判断与校准

三题全部正确，不需要概念性纠正。第 3 题尤其关键——Backfill 的历史处理范围不能倒退正常 Incremental 的 checkpoint；Backfill 应维护独立的 range/cursor/state，否则会让正常增量任务从历史位置重新开始。

## 结课判定

Module 9 第 2 课理解检查通过，必要校准完成，本课正式结束。