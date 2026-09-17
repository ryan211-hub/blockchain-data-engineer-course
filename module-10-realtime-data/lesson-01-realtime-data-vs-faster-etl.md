# 第1课｜实时数据到底是什么：为什么不是“把 ETL 跑快一点”

## Lesson Contract

所属 Module：Module 10 — 实时数据。

本课只解决一个核心问题：

> Batch 和 Streaming 的本质区别到底是什么？

学完以后，你应该能够解释：
- 为什么“每分钟执行一次 ETL”仍然可能是 Batch。
- 为什么 Streaming 不是单纯追求低延迟。
- 为什么实时系统开始需要“持续运行的处理状态”。
- 为什么到了 Streaming，我们会自然遇到 Kafka、Offset、Consumer、Replay 这些概念。

本课暂时不深入 Kafka Topic、Partition、Consumer Group。这些从下一课开始。

---

## 一、先从你已经熟悉的 ETL 开始

Module 9 里，我们已经建立了一套 Batch ETL 模型：

```text
确定 Processing Range
        ↓
Extract
        ↓
Transform
        ↓
Load
        ↓
Validation
        ↓
Checkpoint
```

例如每天凌晨处理昨天的数据：

```text
2026-09-16 00:00
        ↓
2026-09-17 00:00
```

这个 Job 有一个很清楚的特点：

> 它知道“这一轮我要处理哪一批数据”。

比如：

```text
start_date = 2026-09-16
end_date   = 2026-09-17
```

或者 Blockchain 里面：

```text
start_block = 23,400,000
end_block   = 23,410,000
```

处理完：

```text
Job finished
Checkpoint = 23,410,000
```

这一轮工作结束。

这就是非常典型的 Batch 思维。

## 二、如果我把它改成每分钟运行一次呢？

假设原来每天运行一次，现在改成每分钟运行一次，是不是就变成 Streaming？不是。

```text
12:00 Job
处理 11:59 ~ 12:00

12:01 Job
处理 12:00 ~ 12:01

12:02 Job
处理 12:01 ~ 12:02
```

它虽然已经非常“实时”，但运行模型仍然是：取一小批数据 → 处理 → 结束，再取下一小批数据 → 处理 → 结束。

所以它更准确地说是 Micro-batch。本质仍然属于 Batch。只是 Batch Window 从 1 day 缩小成 1 minute，甚至 5 seconds，都不改变它的基本计算模型。

## 三、Streaming 真正不同在哪里？

Streaming 的关键不是“跑得快”，而是：

> 数据被视为一个持续到达、理论上没有终点的流。

Batch 看世界：

```text
Dataset
[ A B C D E F ]
开始
↓
处理
↓
结束
```

Streaming 看世界：

```text
A → B → C → D → E → F → G → H → ...
                                ↓
                          未来还会继续来
```

Batch 问：这一批数据在哪里开始，在哪里结束？

Streaming 问：新数据来了，我接下来应该处理哪一条？

因此两者的核心抽象不同：

```text
Batch
Finite Dataset

Streaming
Unbounded Data Stream
```

即：有界数据集 vs 无界数据流。

## 四、银行系统举例

银行做“昨日银行卡交易日报”时，可以按一天的范围做 Transactions → ETL → Daily Fact → DWS → Report，这是典型 Batch。

但如果需求改成“疑似盗刷交易发生后 10 秒内产生风控告警”，每 5 秒跑一次 SQL 虽然可能做到低延迟，但会很快遇到：上一轮还没跑完怎么办、数据库持续被轮询怎么办、多个下游都需要同一批交易怎么办、消费者挂掉以后能否重新消费、一条交易是否已经处理过等问题。

这时候问题不再只是 SQL 怎么写，而是如何管理一个持续产生的数据流。这就是 Streaming Infrastructure 开始出现的地方。

## 五、Blockchain 是一个特别自然的 Streaming 场景

Ethereum 新区块不断产生：

```text
Block N
   ↓
Block N+1
   ↓
Block N+2
   ↓
Block N+3
   ↓
...
```

Transaction、Log、Transfer、Swap 也会持续产生。

Wallet Activity Platform 如果要求用户转入 USDC 后 Dashboard 很快看到，就会自然形成：

```text
New Block
   ↓
New Log
   ↓
Decode Transfer
   ↓
Update database
   ↓
Push to dashboard
```

这已经非常像 Event → Event → Event → Event 连续进入 Pipeline。

## 六、一个很重要的视角区别

【Blockchain Protocol 视角】

Ethereum 本身不断产生 Block → Transaction → Receipt → Log，这是链本身的执行和共识过程。

【Blockchain Data Engineer 视角】

我们看到的是：新链上事实不断到达 → 持续 ingest → 持续 decode → 持续 transform → 持续写入下游。

所以在数据工程师眼里，Blockchain 天生可以被看成 continuously arriving event source。

但 Blockchain 并不是 Kafka，Node WebSocket 也不是可靠消息队列。这个问题 Module 10 后面会专门讲。

## 七、Streaming 出现以后，一个概念开始改变

Batch 中 Job 通常是核心单位：Job Start → Processing Range → Job End，然后 Scheduler 再启动下一次。

Streaming 更接近长期运行的 Process：A → B → C → D → E → ...，可能运行几小时、几天、几个月，理论上没有“正常结束”这个概念。

于是 Batch 中常问“Job 成功了吗？”，Streaming 中则更多问：

> Processor 现在处理到哪里了？

## 八、这就自然引出了 Offset

假设事件流有 Event 0、1、2、3、4、5……，Consumer 已经成功处理到 3，进程 Crash。重启后必须知道“我上次处理到哪里”。

这个“流里面的位置”就是之后会重点学习的 Offset。

可以先理解成：

```text
Offset
=
我在这条事件流里面读到哪里了
```

它和 Module 9 学过的 Checkpoint 很像，但不是完全一样。后面会专门比较 Kafka Offset vs ETL Checkpoint vs Indexer Checkpoint。

## 九、Batch 和 Streaming 并排看

- Batch：有界 Dataset；一批一批处理；Job 有开始和结束；关注 Processing Range；Scheduler 启动 Job；使用 Batch Checkpoint；适合历史计算。
- Streaming：无界 Event Stream；持续处理；Processor 通常长期运行；关注 Current Stream Position；Consumer 持续消费；使用 Offset / Stream Checkpoint；适合低延迟持续处理。

但 Streaming 并不意味着应该全部替换 Batch。实际公司通常两者都会存在。

## 十、为什么 Streaming 无法取代 Batch？

实时 Wallet Pipeline 即使连续运行，如果某段历史因为 Decoder Bug 出现错误，仍然需要 Backfill 指定历史 block range 重新处理。这又回到了 Batch Processing Range。

真实数据平台经常是：Blockchain → Realtime Streaming → Serving，同时保留 Batch / Backfill / Reconciliation 路径。

## 十一、这和 Module 9 正好连接起来

Module 9 讲过 Realtime、Backfill、Reorg Replay 应尽量复用 process_block()。

它们不是三套不同业务逻辑，区别主要是输入：

```text
Realtime
Input = 持续来的新数据

Backfill
Input = 指定历史 Range

Reorg Replay
Input = 需要重新计算的 Range
```

核心 Transform 尽量保持一致：Block → Transaction → Receipt → Log → Decoded Event。

## 十二、一个重要心智模型

以后看到“实时系统”，不要首先想到 Kafka，也不要首先想到 WebSocket。先问：数据是不是持续产生？我要多快处理？消费者暂时处理不过来怎么办？消费者挂了怎么办？恢复后从哪里继续？同一条消息重复处理怎么办？历史错误怎么修？

当这些问题同时出现时，你真正面对的才是 Streaming Data Engineering。Kafka 只是解决其中一部分问题的基础设施。

## 十三、和传统数据工程经验做一个映射

```text
Module 9
“这一批数据怎么可靠处理？”
              ↓
Module 10
“数据永远不断到来时，怎么持续可靠地处理？”
```

这就是两个 Module 最核心的分界。

## 本课核心结论

第一：Streaming ≠ 更快的 Batch。

第二：Batch 面向 bounded dataset；Streaming 面向 unbounded stream。

第三：Batch 关注“这一轮处理什么范围”；Streaming 关注“当前消费到流的什么位置”。

第四：真实的数据平台通常同时需要 Streaming + Batch / Backfill / Reconciliation。

## 理解检查

### 问题一

假设一个系统每 10 秒运行一次 ETL Job：

```sql
SELECT *
FROM transfers
WHERE block_number > checkpoint
  AND block_number <= current_block;
```

处理完 Job 退出，10 秒后 Scheduler 再启动一次。它更接近 Batch 还是 Streaming？重点不是“10 秒很快还是很慢”。

### 问题二

一个 Consumer 处理 Event 100～104，在处理完 Event 102 后 Crash。重新启动后，系统最需要保存哪一种信息，才能知道应该从哪里继续？先用自己的话描述，不要求使用 Kafka 术语。

### 问题三

Wallet Realtime Pipeline 已经连续运行一个月，但后来发现 Decoder 存在 Bug，导致三天前某 20,000 个 Block 内的部分 Transfer 解码错误。为什么仅仅拥有一个 Streaming Pipeline 还不够？请联系 Module 9 回答。

## 用户回答

问题一，选A。因为这个 Job 的 Input Range 它是有边界的，虽然它每 10 秒启动一次，但它还是属于 Batch 的方式。

问题二，最需要保存的是类似 checkpoint 这个信息。这样子的话，重启之后，它会知道应该从哪里继续。

问题3，这个情况下需要修复历史数据，需要使用 backfill job，这样它不会影响当前的数据处理。

## 老师判断与校准

三题全部通过。

- 问题一：判断正确。决定它属于 Batch 的关键不是运行间隔，而是每一轮都处理一个明确有界的 Input Range，并在处理完成后退出；这更接近 Micro-batch。
- 问题二：概念方向完全正确。你说的“类似 checkpoint 的信息”，在 Kafka 场景里通常对应 consumer offset：它记录 Consumer 在事件流中已经处理到的位置。Offset 与 ETL / Indexer Checkpoint 很相似，但后续会继续区分它们的职责边界。
- 问题三：判断正确。Streaming 负责持续处理新数据，但历史错误仍需要独立的 Backfill / Historical Repair 路径重新计算指定历史范围；修复历史时不应该为了倒退历史而破坏当前 Realtime Pipeline 的处理状态。

## 结课判定

Module 10 第 1 课理解检查通过，正式完成。已建立 Batch vs Streaming 的核心心智模型：有界 Dataset vs 无界 Stream；Processing Range vs Stream Position；Streaming 与 Batch / Backfill / Reconciliation 需要并存。
