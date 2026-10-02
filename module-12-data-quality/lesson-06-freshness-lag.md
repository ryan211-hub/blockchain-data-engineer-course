# Module 12 第 6 课｜Freshness & Lag：数据正确但太晚也可能不可用

## Lesson Contract

所属 Module：Module 12 — 数据质量

本课核心问题：

> If data is correct but arrives too late, is it still good data?

学完以后，你应该能够解释并设计：

- 什么是 Freshness。
- Freshness 和 Accuracy / Completeness 的区别。
- 什么是 Data Lag。
- Event Time、Processing Time、Availability Time 的区别。
- 为什么“最终正确”不代表“当前可用”。
- 如何定义 Freshness SLA / SLO。
- 如何用 Block Lag、Time Lag、Consumer Lag 描述实时链路状态。
- 为什么不同 Pipeline / Sink 可以有不同 Freshness。
- 如何区分 temporary lag、systematic lag、stalled pipeline。
- Freshness Failure 是否应该阻止 Checkpoint。
- Realtime、Near-realtime、Batch 三种链路该如何设计不同 Freshness 目标。
- 为什么 Freshness 问题经常是 Operational Quality 和 Data Quality 的交界。

本课暂不展开：

- Kafka 内部机制复习；
- 完整监控平台设计；
- Reorg Correctness；
- SLA 合同管理；
- 自动扩缩容策略；
- 高级流处理窗口语义。

---

## 一、Freshness 到底是什么？

Freshness 问的是：

> How up-to-date is the data?

也就是：

> 这份数据离“现实世界当前状态”有多远？

例如链上已经产生到：

```text
block 21,000,000
```

而你的数据平台只处理到：

```text
block 20,999,500
```

那么：

```text
block lag = 500
```

即使你已经处理的这些数据完全正确，

仍然存在：

> Freshness Problem.

---

## 二、Freshness 和 Accuracy 不一样

假设：

```text
block 20,999,500
```

之前的所有数据：

```text
100% correct
```

但是最新链已经到：

```text
21,000,000
```

那么：

```text
Accuracy = PASS
Completeness within processed range = PASS
Freshness = FAIL
```

这说明：

> Correct data can still be stale data.

---

## 三、Freshness 和 Completeness 也不一样

Completeness 问：

> 应该存在的数据有没有缺失？

Freshness 问：

> 数据有没有及时到达？

例如：

```text
现在是 12:00
11:55 的 block 还没进入你的 Fact
```

可能有两种情况。

第一种：

```text
pipeline still processing
```

这是：

> Freshness Lag.

第二种：

```text
pipeline already passed that range
but data never arrived
```

这是：

> Completeness Failure.

所以：

```text
not yet arrived
```

和：

```text
missing forever / unexpectedly absent
```

不是一个概念。

---

## 四、[Time 视角] Event Time 和 Processing Time

实时数据系统里，一个事实至少有两个重要时间。

### Event Time

事实在源头真正发生的时间。

例如：

```text
block timestamp = 12:00:03
```

### Processing Time

你的系统处理这条数据的时间。

例如：

```text
processed_at = 12:00:20
```

那么：

```text
processing delay = 17 seconds
```

Freshness 经常就是在看：

> Event Time 到数据可用时间之间的差距。

---

## 五、Availability Time 更贴近数据产品

还有第三个时间：

### Availability Time

数据真正可以被下游查询或使用的时间。

例如：

```text
block created        = 12:00:03
indexer decoded      = 12:00:08
Kafka delivered      = 12:00:10
Postgres committed   = 12:00:13
DWS updated          = 12:00:25
Dashboard refreshed  = 12:01:00
```

如果用户看 Dashboard，

真正关心的是：

```text
12:00:03
→
12:01:00
```

也就是接近 57 秒延迟。

---

## 六、Freshness 是 End-to-End 的

不能只看某一层。

例如：

```text
Indexer lag = 2 sec
Kafka lag = 1 sec
Consumer lag = 3 sec
DWS refresh lag = 20 sec
Dashboard cache lag = 60 sec
```

用户最终体验：

```text
~86 sec stale
```

所以：

> End-to-end Freshness is determined by the slowest accumulated path, not by one component alone.

---

## 七、Block Lag 是 Blockchain Pipeline 很常见的 Freshness Metric

定义：

```text
chain_head_block
-
pipeline_processed_block
```

例如：

```text
chain head = 21,000,000
checkpoint = 20,999,950
```

那么：

```text
block lag = 50
```

这是非常直观的指标。

---

## 八、Block Lag 的优点

优点：

```text
simple
chain-native
easy to monitor
independent of local clock
```

而且特别适合：

```text
Indexer
Raw ingestion
Fact pipeline
```

这种按 block 推进的数据链路。

---

## 九、Block Lag 也有局限

不同链：

```text
block time
```

不同。

即使同一链，block interval 也不完全固定。

所以：

```text
100 blocks lag
```

到底代表：

```text
20 sec
2 min
20 min
```

要看链。

因此 Block Lag 通常还会配：

> Time Lag.

---

## 十、Time Lag 怎么算？

一种简单方式：

```text
now
-
latest_processed_block_timestamp
```

例如：

```text
current time = 12:10:00
latest processed block timestamp = 12:08:30
```

那么：

```text
time lag = 90 seconds
```

这个指标对产品更容易理解。

---

## 十一、[Data Engineer 视角] Checkpoint 本身就是 Freshness Signal

你之前一直在学习 Checkpoint。

现在可以再看一个新视角。

Checkpoint 不只表示：

> processed up to here.

它还可以帮助判断：

> how far behind are we?

例如：

```text
chain head = 21,000,000
checkpoint = 20,998,000
```

说明：

```text
pipeline is 2,000 blocks behind
```

所以 Checkpoint 同时具有：

```text
processing state
+
freshness observability
```

---

## 十二、Consumer Lag 和 Data Freshness 不完全一样

Kafka 里：

```text
consumer lag
```

通常表示：

```text
latest topic offset
-
consumer committed offset
```

这是消息系统视角。

但它不一定直接等于：

```text
business data freshness
```

因为一个 offset：

```text
可能是一条 event
可能是一批 event
```

而且上游 Indexer 自己也可能已经 lag。

所以：

```text
Kafka Consumer Lag
```

只是 Freshness Chain 的一个组成部分。

---

## 十三、一个完整 Lag Chain

可以拆成：

```text
Chain Head Lag
↓
Indexer Lag
↓
Kafka Producer Delay
↓
Consumer Lag
↓
Sink Commit Lag
↓
DWS Refresh Lag
↓
Dashboard Refresh Lag
```

每一层都可能贡献：

```text
latency
```

---

## 十四、为什么要区分 Lag Location？

假设用户说：

> Dashboard 慢了 10 分钟。

可能原因完全不同。

例如：

```text
Indexer behind chain head
```

或者：

```text
Kafka consumer backlog
```

或者：

```text
Postgres write slow
```

或者：

```text
DWS scheduler late
```

或者：

```text
Dashboard cache not refreshed
```

所以不能只记录：

```text
total lag
```

还要知道：

> where the lag is introduced.

---

## 十五、Freshness SLA / SLO 是什么？

你可以把它理解为：

> 数据最多允许“旧”到什么程度。

例如：

```text
wallet activity API
Freshness SLO:
95% of events available within 10 seconds
```

或者：

```text
daily analytics dashboard
Freshness SLO:
data ready by 08:00 next day
```

不同产品完全不同。

---

## 十六、[Product 视角] Freshness 没有统一标准

同一份链上数据：

对于钱包通知：

```text
30 sec
```

可能已经太慢。

对于：

```text
monthly protocol report
```

延迟 30 分钟可能完全无所谓。

所以：

> Freshness is workload-dependent.

不能说：

```text
5 minutes lag = bad
```

必须问：

> Bad for which product?

---

## 十七、Realtime 不等于 Zero Lag

所谓 realtime 通常不是：

```text
0 ms
```

而是：

> sufficiently low latency for the use case.

例如：

```text
2 sec
5 sec
10 sec
30 sec
```

都可能被称为 realtime / near-realtime。

重点是：

```text
meets product requirement
```

---

## 十八、Batch 系统也有 Freshness

Freshness 不只是 Streaming 问题。

例如银行数据仓库：

```text
T+1
```

如果规则是：

```text
每天 07:00 前生成昨日数据
```

那么：

```text
06:30 完成
→ Freshness PASS

08:30 完成
→ Freshness FAIL
```

哪怕数据最终完全正确。

---

## 十九、Late Data 和 Freshness 的关系

某些数据晚到：

```text
Late-arriving Data
```

会造成：

```text
temporary incompleteness
+
freshness degradation
```

例如：

```text
99% data arrives within 10 sec
1% arrives after 5 min
```

那么你必须决定：

```text
when is data considered usable?
```

这涉及：

```text
watermark
lateness tolerance
```

但本课只需要理解概念，不深入流处理窗口机制。

---

## 二十、Fresh but Mutable vs Stable but Delayed

这是 Blockchain 特别重要的 trade-off。

实时链路：

```text
very fresh
but may still be affected by reorg
```

稳定链路：

```text
wait N confirmations
less fresh
but more canonical confidence
```

所以常见架构是：

```text
Realtime / Fresh Path
+
Canonical / Stable Path
```

你之前已经学过：

> fresh but mutable vs stable canonical fact.

现在从 Freshness 维度看会更清晰。

---

## 二十一、[Blockchain 视角] Finality 直接影响 Freshness

如果要求：

```text
only finalized blocks
```

那么天然就会引入：

```text
finality delay
```

这不是性能问题，

而是业务规则主动选择的延迟。

所以：

> Not all lag is bad lag.

有些 lag 是为了：

```text
correctness
stability
canonical confidence
```

而主动引入的。

---

## 二十二、Freshness 和 Correctness 有 trade-off

极端方案 A：

```text
chain head immediately
```

优点：

```text
fresh
```

缺点：

```text
mutable
reorg risk
```

极端方案 B：

```text
wait many confirmations
```

优点：

```text
stable
```

缺点：

```text
stale
```

真实系统需要：

> choose the right point for the product.

---

## 二十三、为什么同一家公司会有多个 Freshness Tier？

例如：

```text
Tier 1
Realtime wallet activity
~seconds

Tier 2
Canonical analytics
~minutes

Tier 3
Daily reconciled reporting
~hours
```

它们服务的需求不同。

并不是：

```text
越快越好
```

因为更低延迟通常意味着：

```text
more cost
more complexity
more reorg handling
more operational pressure
```

---

## 二十四、多 Sink Freshness 不一致不一定是错误

假设：

```text
Postgres checkpoint = 21,000,000
ClickHouse checkpoint = 20,999,700
Parquet = 20,999,000
```

这说明：

```text
different freshness
```

但不一定说明：

```text
data corruption
```

只要各自：

```text
progressing normally
within SLA
```

就是合理状态。

---

## 二十五、什么时候 Lag 才真正是 Quality Failure？

核心标准不是：

```text
lag > 0
```

而是：

```text
lag > allowed freshness boundary
```

例如 SLA：

```text
<= 100 blocks
```

当前：

```text
lag = 40
```

Freshness PASS。

当前：

```text
lag = 800
```

Freshness FAIL。

---

## 二十六、Temporary Lag vs Systematic Lag

Temporary Lag：

```text
short spike
then recover
```

例如：

```text
traffic burst
provider throttling
brief DB slowdown
```

Systematic Lag：

```text
lag keeps increasing
```

例如：

```text
producer 10k/s
consumer 8k/s
```

那么：

```text
lag grows 2k/s
```

这说明：

> processing capacity < incoming rate.

---

## 二十七、Increasing Lag 是危险信号

假设：

```text
10:00 lag = 100
10:05 lag = 500
10:10 lag = 1,200
10:15 lag = 2,500
```

说明：

```text
pipeline is falling behind
```

即使系统仍在处理。

这和：

```text
lag stable at 100
```

完全不同。

所以要观察：

> lag level and lag trend.

---

## 二十八、Stalled Pipeline 更严重

例如：

```text
chain head increasing
checkpoint unchanged
```

那么：

```text
lag continuously increases
```

且：

```text
processing throughput = 0
```

这是：

> stalled pipeline.

这通常已经不仅是 Freshness Degradation，

还是：

```text
availability / operational failure
```

---

## 二十九、Lag Recovery 需要看 Throughput

如果：

```text
incoming rate = 1,000 events/s
processing rate = 1,500 events/s
```

那么 backlog 可以逐渐恢复。

如果：

```text
incoming rate = 1,000
processing rate = 900
```

永远追不上。

所以判断能不能恢复，需要看：

```text
processing_rate - incoming_rate
```

---

## 三十、[Capacity 视角] Back Pressure 和 Freshness 是直接相关的

Back Pressure 的外部表现之一就是：

```text
lag growth
```

这和你 Module 10 学过的内容连接起来：

```text
Producer faster than Consumer
↓
Backlog
↓
Lag grows
↓
Freshness degrades
```

所以：

> Lag is one of the clearest observable symptoms of back pressure.

---

## 三十一、Freshness Monitoring 应记录什么？

常见指标：

```text
chain_head_block
processed_block
block_lag
latest_event_time
latest_available_time
time_lag
consumer_lag
processing_rate
incoming_rate
lag_growth_rate
```

不一定全部都要有，

但至少要能回答：

```text
How far behind?
Since when?
Where?
Recovering or worsening?
```

---

## 三十二、Freshness Failure 是否应该阻止 Checkpoint？

这里需要非常小心。

如果：

```text
当前 block 已正确处理并验证
```

只是：

```text
整体 pipeline behind chain head
```

那么已经完成的 block：

```text
checkpoint 可以推进
```

因为 checkpoint 表示：

> validated completion up to this processed boundary.

不能因为：

```text
still behind head
```

就不让 checkpoint 前进。

---

## 三十三、这和前几课不同

前几课：

```text
Validation FAIL
Reconciliation FAIL
```

意味着：

```text
current processing range not trustworthy
```

所以：

```text
do not advance checkpoint
```

但 Freshness FAIL 可能只是：

```text
processing too slowly
```

当前处理的数据仍然：

```text
correct
validated
```

所以：

```text
checkpoint may still advance
```

---

## 三十四、[关键视角] Checkpoint 和 Freshness SLA 是两个不同状态

可以出现：

```text
checkpoint advancing normally
Freshness SLA failing
```

例如：

```text
每分钟处理 100 blocks
链每分钟产生 120 blocks
```

Checkpoint 一直前进，

但：

```text
lag keeps growing
```

所以：

```text
Processing State = progressing
Freshness Quality = degrading
```

这两个必须分开监控。

---

## 三十五、什么时候 Freshness 问题会间接阻止 Pipeline？

如果 Lag 原因实际上是：

```text
downstream unavailable
```

导致当前 batch：

```text
无法完成 write / validation
```

那就不是单纯 Freshness 问题了。

而是：

```text
processing failure
```

此时自然不能推进 checkpoint。

所以要分清：

```text
slow but successful
vs
failed
```

---

## 三十六、Freshness Incident 怎么处理？

典型流程：

```text
Detect SLA breach
↓
Locate lag layer
↓
Measure backlog
↓
Compare input vs processing rate
↓
Find bottleneck
↓
Recover / scale / optimize
↓
Catch up
↓
Confirm freshness restored
```

---

## 三十七、Catch-up 模式

系统从故障恢复后，经常进入：

```text
catch-up mode
```

例如：

```text
chain head = 21,000,000
checkpoint = 20,900,000
```

然后提高处理速度：

```text
processing faster than incoming
```

直到：

```text
lag returns below SLA
```

---

## 三十八、Freshness 不应该只看平均值

假设：

```text
average lag = 5 sec
```

听起来很好。

但实际上：

```text
99% = 1 sec
1% = 10 min
```

对于某些用户，这 1% 可能非常严重。

所以成熟系统可能看：

```text
p50
p95
p99
max
```

尤其是：

```text
event-to-availability latency
```

---

## 三十九、Data Freshness 也需要 Product Contract

例如可以写成：

```text
Wallet activity
95% available within 10 sec
99% within 30 sec

Canonical analytics
available within 2 min after finalization

Daily report
ready before 07:00 UTC
```

这样 Freshness 才真正可测。

---

## 四十、本课核心心智模型

可以压缩为：

```text
Source Reality
↓
Event occurs
↓
Pipeline processes
↓
Data becomes available
↓
Measure delay
↓
Compare with product SLA
↓
Freshness PASS / FAIL
```

以及：

```text
Correct
≠
Fresh
```

---

## 本课核心结论

> Freshness asks how up-to-date the data is.

> Accuracy can PASS while Freshness FAILS.

> Event Time, Processing Time and Availability Time represent different stages of latency.

> End-to-end Freshness is the accumulated result of Indexer, Kafka, Consumer, Sink, DWS and serving delays.

> Block Lag is chain-native and simple; Time Lag is often easier for product interpretation.

> Consumer Lag is only one component of business-data freshness.

> Freshness thresholds must come from workload and product requirements.

> Realtime does not mean zero latency; it means sufficiently low latency for the use case.

> Multi-sink systems may have different freshness without being incorrect.

> Lag itself is not automatically a failure; exceeding the allowed freshness boundary is.

> Increasing Lag usually indicates the system is falling behind.

> Checkpoint advancement and Freshness SLA are separate concerns: a pipeline can advance checkpoint while still failing freshness.

> Current data can be correct and validated but operationally too stale to be useful.

---

## 理解检查

### 问题一

当前 Ethereum Chain Head：

```text
21,000,000
```

你的 Indexer Checkpoint：

```text
20,999,800
```

而 20,999,800 之前的数据都已经正确处理并通过 Validation。

请回答：

1. 当前主要是哪一种 Data Quality 问题？
2. Accuracy 是否一定有问题？
3. Checkpoint 是否可以继续推进？为什么？

### 问题二

某实时 Pipeline：

```text
10:00 lag = 100 blocks
10:05 lag = 500 blocks
10:10 lag = 1,200 blocks
10:15 lag = 2,500 blocks
```

请回答：

1. 这和“lag 一直稳定在 100 blocks”有什么本质区别？
2. 更可能说明什么系统问题？
3. 你接下来最应该比较哪两个 rate？

### 问题三

Postgres：

```text
checkpoint = 21,000,000
```

ClickHouse：

```text
checkpoint = 20,999,500
```

两边已经处理完成的数据都正确。

请回答：

1. 能不能直接说 ClickHouse 数据错误？
2. 这里主要体现的是哪一种差异？
3. 做 Postgres vs ClickHouse Reconciliation 时，为什么还要先对齐共同 processing boundary？

## 用户回答（问题一）

问题一：

1.freshness 可能由问题

2. Accuracy 不一定有问题
3. Checkpoint可以推进，因为当前并没有accuracy 等其他数据问题

## 老师判断 / 校准（问题一）

通过。

1. **当前主要是哪一种 Data Quality 问题**：你的方向正确。更准确地说，当前主要是 **Freshness / Lag 问题**。Chain Head 已经到 21,000,000，而 Indexer Checkpoint 还在 20,999,800，说明 Pipeline 落后于当前链头。
2. **Accuracy 是否一定有问题**：正确，不一定。题目已经明确说明 20,999,800 之前的数据都正确处理并通过 Validation，因此可以出现：

```text
Accuracy = PASS
Freshness = FAIL
```

3. **Checkpoint 是否可以继续推进**：正确，可以。因为 Checkpoint 表示：

> validated completion up to this processed boundary.

只要后续 block 被正确处理并通过 Validation，Checkpoint 就应该继续前进。不能因为 Pipeline 还落后于 Chain Head，就阻止已经完成的数据推进 Checkpoint。

这里最重要的区分是：

```text
Checkpoint
→ processing / validated completion state

Freshness
→ distance from current source reality
```

因此可以同时出现：

```text
Checkpoint advancing
Freshness SLA failing
```

核心结论：

> Being behind the chain head does not automatically mean the processed data is inaccurate.

> Freshness Failure alone does not necessarily block checkpoint advancement.

## 用户回答（问题二）

问题二：

1. lag一直在增长，说明Pipeline 消费速度比生产速度慢，这个情况是有问题的，会导致数据延迟越来越高。
2. 说明系统在 消费处理上有问题

## 老师判断 / 校准（问题二）

前两小问正确，第 3 小问尚未回答，因此本题暂不判定完成。

1. **Increasing Lag 和稳定 Lag 的区别**：正确。Lag 持续增长说明 Pipeline 正在持续落后，Freshness 会不断恶化；而 Lag 稳定在 100 blocks，说明系统虽然存在固定延迟，但处理能力大体能跟上输入速度。
2. **更可能说明什么系统问题**：正确。更可能说明 Processing Capacity 不足，常见表现是 Consumer / Processor 的处理速度低于上游产生速度，形成 Backlog。
3. **下一步最应该比较哪两个 rate**：尚未回答。

这里应比较：

```text
incoming_rate
vs
processing_rate
```

例如：

```text
incoming_rate = 1000 events/s
processing_rate = 800 events/s
```

那么：

```text
backlog grows by 200 events/s
```

说明系统会越来越落后。

核心结论：

> Increasing Lag usually means processing capacity is below incoming rate.

> To judge whether the pipeline can recover, compare incoming rate and processing rate.

## 用户回答（问题三）

问题三：

1.不能

2.Postgres和ClickHouse是不同的数据库，他们有各自的checkpoint 。

3. 虽然Postgres和ClickHouse是不同的数据库， 但是它们都是对fact数据的不同处理，可以说它们的来源是相同的，所以在相同的processing boundary之内是可以做  Reconciliation&#x20;

## 老师判断 / 校准（问题三）

通过。

1. **不能直接说 ClickHouse 数据错误**：正确。两边已经处理完成的数据都正确，因此 ClickHouse 只是处理进度更落后。
2. **主要体现的差异**：你的“各自有独立 checkpoint”是正确基础。更准确地说，这里体现的是 **Freshness / Processing Progress Difference**，不是 Accuracy Failure。
3. **为什么 Reconciliation 前要对齐共同 Processing Boundary**：正确。Postgres 和 ClickHouse 虽然是不同 Sink，但来源于同一套业务事实。只有在相同 processing boundary 下比较，才是在比较“同一范围的同一业务事实”。

例如：

```text
Postgres checkpoint   = 21,000,000
ClickHouse checkpoint = 20,999,500
```

如果直接比较各自最新全量结果：

```text
Postgres
包含 20,999,501 → 21,000,000

ClickHouse
还没有这些数据
```

此时 mismatch 只是 Freshness Difference，不代表数据错误。

所以更合理的是对齐：

```text
common boundary = 20,999,500
```

然后比较：

```text
Postgres up to 20,999,500
vs
ClickHouse up to 20,999,500
```

核心结论：

> Multi-sink reconciliation must align the same processing boundary.

> Different checkpoints can indicate different freshness without indicating incorrect data.

## 用户补充回答（问题二第 3 小问）

比较
incoming_rate
vs
processing_rate

## 老师判断 / 校准（问题二第 3 小问）

正确，问题二正式通过。

应该比较：

```text
incoming_rate
vs
processing_rate
```

三种典型状态：

```text
processing_rate < incoming_rate
→ backlog grows
→ lag increases
→ freshness keeps degrading
```

```text
processing_rate ≈ incoming_rate
→ backlog roughly stable
→ lag remains roughly stable
```

```text
processing_rate > incoming_rate
→ backlog shrinks
→ catch-up possible
→ freshness can recover
```

因此，判断一个 Lagging Pipeline 是否有能力恢复，不能只看当前 Lag 数值，还必须观察 Input / Processing Throughput。

核心结论：

> Lag level tells us how far behind the pipeline is.

> Incoming rate vs processing rate tells us whether the situation is improving or worsening.

## 结课判定

Module 12 第 6 课理解检查全部通过，正式完成。

已经能够：
- 区分 Freshness、Accuracy 与 Completeness。
- 理解 Correct Data 仍然可能是 Stale Data。
- 区分 Event Time、Processing Time 与 Availability Time。
- 理解 Freshness 是 End-to-End Quality，而不是只看 Kafka Consumer Lag。
- 使用 Block Lag / Time Lag / Consumer Lag 描述实时数据链路状态。
- 理解 Freshness SLA / SLO 必须根据具体 Product / Workload 定义。
- 区分 Stable Lag、Increasing Lag 与 Stalled Pipeline。
- 使用 incoming_rate vs processing_rate 判断 Backlog 趋势和 Catch-up 能力。
- 理解 Back Pressure 会直接表现为 Lag Growth 和 Freshness Degradation。
- 理解不同 Sink 可以拥有独立 Checkpoint 和不同 Freshness，而不代表 Accuracy Failure。
- 理解 Multi-sink Reconciliation 必须先对齐共同 Processing Boundary。
- 理解 Checkpoint Advancement 与 Freshness SLA 是两个不同状态。
- 理解 Freshness Failure 本身不一定阻止 Checkpoint；只有当前处理范围无法完成正确写入 / Validation 时才应阻止推进。
- 理解 Blockchain 中 Freshness 与 Canonical Stability / Finality 之间存在工程 Trade-off。