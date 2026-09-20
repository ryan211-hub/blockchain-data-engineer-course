## Lesson Contract

所属 Module：Module 10 — 实时数据

本课核心问题：

> 当实时 Consumer 处理失败、积压扩大、或者需要重新处理历史数据时，系统如何 Retry、Replay、控制 Backpressure，并恢复到健康状态？

学完以后，你应该能够解释：

- Retry 和 Replay 的区别。
- 为什么 Retry 不能无限立即重试。
- Backoff、Dead Letter Queue / Dead Letter Topic 的作用。
- 什么是 Consumer Lag，以及它为什么是实时系统最关键的健康指标之一。
- 什么是 Backpressure，以及它和 Producer / Consumer speed mismatch 的关系。
- Consumer 处理能力不足时有哪些扩容和降压手段。
- 为什么 Replay 必须建立在 Retention、Offset 和 Idempotency 之上。
- 为什么实时故障恢复不只是“重启 Consumer”。

本课不深入 Kafka Broker 内部 flow control、Flink backpressure algorithm 或复杂调度算法，只建立 Blockchain Data Engineer 必须掌握的 failure recovery 设计模型。

---

## 一、先区分 Retry 和 Replay

这两个词经常混在一起，但工程含义不同。

> Retry = retry a failed operation / event.

> Replay = reprocess a range of previously stored events.

例如：

Retry：某一条 Event 写数据库超时，于是稍后再处理这条 Event。

Replay：发现 Decoder 有 Bug，需要重新处理 Offset 10000~20000 的历史数据。

所以可以先记：

Retry is local recovery.

Replay is historical reprocessing.

---

## 二、Retry 解决的是“这一次没成功”

假设 Consumer 处理 Event 100：

Read Event 100 → Write Postgres → timeout

这时最自然的动作是 Retry。

但 Retry 并不等于：

while true: retry immediately

如果 Postgres 本身正在故障，立刻无限重试只会让系统更糟。

这会产生：

- retry storm
- resource exhaustion
- connection saturation
- cascading failure

所以 Retry 必须有策略。

---

## 三、Retry Strategy

一个常见模式是：

Retry 1 → wait 1s
Retry 2 → wait 2s
Retry 3 → wait 4s
Retry 4 → wait 8s

这叫：

> Exponential Backoff.

通常还会加 jitter，避免大量 Consumer 同时重试造成同步冲击。

核心思想是：

> If downstream is unhealthy, do not attack it harder.

---

## 四、不是所有错误都应该 Retry

需要区分：

Transient Error
vs
Permanent Error

Transient Error 例如：

- temporary network timeout
- database connection reset
- RPC rate limit
- temporary downstream overload

这类错误通常值得 Retry。

Permanent Error 例如：

- malformed payload
- unsupported schema
- impossible business state
- decoder bug for a specific record

这类错误如果无限 Retry，可能永远无法成功。

所以要有：

> bounded retries.

也就是有限次数重试。

---

## 五、Retry 失败以后怎么办？

常见做法是把 Event 放到：

> Dead Letter Queue / Dead Letter Topic.

例如：

wallet_transfers
→ consumer fails 5 times
→ wallet_transfers_dlq

DLQ 的目的不是“丢掉错误数据”。

而是：

> isolate bad events so healthy traffic can continue.

以后可以人工检查、修复逻辑，再单独处理这些 Event。

---

## 六、Retry 和 Ordering 有冲突

这里有一个重要 trade-off。

假设 P0：

Offset 100 → Event A
Offset 101 → Event B
Offset 102 → Event C

Event A 失败。

如果你要求 strict partition ordering，那么 B、C 往往也不能越过 A 继续处理。

于是：

one bad event
can block one partition.

这叫 head-of-line blocking 的一种表现。

如果允许 B、C 继续，则吞吐更高，但业务顺序可能被破坏。

所以：

> Retry policy must respect ordering requirements.

---

## 七、Replay 解决的是“我要重新处理过去”

Replay 和 Retry 不同。

Replay 可能不是因为单条 Event 失败，而是因为：

- decoder bug
- business logic changed
- historical repair
- reconciliation mismatch
- reprocessing after incident

例如：

Consumer 当前已到 Offset 50000。

后来发现 40000~42000 的 Transform 逻辑错误。

如果 Kafka Retention 里数据还在，就可以把消费位置重置到过去，重新处理。

这就是：

> Replay from an earlier offset.

---

## 八、Replay 的三个前提

Replay 能工作，需要至少三个基础：

### 1. Durable Retention

过去的 Event 还在。

### 2. Position

知道从哪个 Topic / Partition / Offset 开始。

### 3. Idempotency

重复处理不会破坏最终数据。

所以可以压缩成：

> Replay = Retained Data + Position + Idempotent Processing.

这和你之前的 Backfill 非常像。

---

## 九、Replay 和 Backfill 的关系

Module 9 里：

Backfill = reprocess a historical input range.

Module 10 里：

Replay = reprocess historical events from the stream.

两者工程哲学一致。

区别只是 input abstraction 不同。

Batch 更常按 block range / date range。

Streaming 更常按 topic-partition-offset。

---

## 十、Backpressure 是什么？

假设：

Producer = 10,000 msg/s
Consumer = 3,000 msg/s

那么每秒积压：

7,000 msg

这时：

Incoming Rate > Processing Rate

系统就产生持续积压。

这就是 Backpressure 场景。

更准确地说：

> Backpressure is pressure created when downstream cannot keep up with upstream.

---

## 十一、Kafka 为什么适合吸收短期 Backpressure？

因为 Kafka 本身是 Durable Log。

Producer 可以继续写，Consumer 暂时落后。

例如：

Log End Offset = 1,000,000
Consumer Offset = 920,000

那么：

Consumer Lag = 80,000

这代表 Consumer 还落后 80,000 条记录。

所以 Kafka 可以吸收：

> temporary speed mismatch.

但不能无限吸收。

---

## 十二、Consumer Lag 是什么？

可以近似理解：

> Lag = Log End Offset - Consumer Position.

例如：

End Offset = 1000
Consumer Offset = 850

则：

Lag = 150

Consumer Lag 是非常关键的 operational metric。

因为它直接回答：

> How far behind is the consumer?

---

## 十三、Lag 大不一定立刻等于故障

要看趋势。

例如：

短时间流量突增：

Lag 0 → 10k → 30k → 5k → 0

说明 Consumer 能追上。

但如果：

Lag 10k → 50k → 100k → 300k

持续增长，说明：

> processing capacity < incoming rate.

这才是结构性问题。

所以比单点 Lag 更重要的是：

> Lag trend.

---

## 十四、如果 Consumer 跟不上，怎么办？

通常有几类手段。

### 1. Scale Consumers

如果 Partition 足够，可以增加 Consumer。

例如：

4 partitions + 2 consumers
→ scale to 4 consumers

提高 parallelism。

但如果已经：

4 partitions + 4 consumers

再加 Consumer 通常不会增加有效并行度。

这时需要考虑 Partition Count。

---

### 2. Optimize Processing

例如：

- batch writes
- reduce per-event DB round trips
- cache reference data
- async I/O
- avoid unnecessary RPC calls

本质是提高：

> events processed per second.

---

### 3. Reduce Downstream Pressure

例如：

- rate limit
- buffer writes
- temporarily pause low-priority consumers
- degrade non-critical enrichment

这是系统层面的 graceful degradation。

---

## 十五、Pause 并不等于丢数据

Kafka 的一个优势就是：

Consumer 可以暂停一段时间。

只要 Retention 足够，Event 仍然保存在 Log 中。

恢复后继续消费即可。

所以：

> Pausing consumption can be safer than failing fast under overload.

前提是：

Retention window > recovery time.

---

## 十六、Retention 是实时恢复能力的一部分

如果 Kafka 只保留 1 小时数据，Consumer 故障 3 小时，恢复后早期 Event 可能已经被删除。

这时就无法简单 Replay。

所以 Retention 不只是存储成本参数。

它实际上定义了：

> How far back can the system recover by replay?

这就是 recovery window。

---

## 十七、Blockchain Streaming 的实际故障场景

假设：

Ethereum Provider
→ Indexer
→ Kafka
→ Transfer Consumer
→ Postgres

现在 Postgres 故障 20 分钟。

合理行为通常不是让 Indexer 停掉整个链上采集。

而是：

Indexer continues producing
Kafka buffers
Consumer retries / pauses
Lag grows
Postgres recovers
Consumer catches up

这就是 Kafka 作为 buffer + durable log 的价值。

---

## 十八、但 Buffer 不是无限的

如果 Postgres 故障 3 天，而 Kafka 只保留 24 小时数据，那么后面的历史可能无法靠 Kafka 完整恢复。

这时要依赖：

- Backfill from blockchain source
- Archive RPC / historical source
- reconciliation job

所以实时系统仍然需要 Batch repair path。

这是 Module 10 一个非常重要的结论。

---

## 十九、Failure Recovery 不等于 Restart

很多人会把恢复理解成：

Consumer crash → restart consumer

但真正的恢复至少要回答：

- 从哪个 Offset 继续？
- 哪些 Event 已经成功写入？
- 哪些 Event 可能重复？
- Sink 是否幂等？
- 是否有 poison message？
- Lag 是否在继续增长？
- Retention 是否还覆盖故障窗口？

所以：

> Recovery is a state problem, not merely a process restart problem.

---

## 二十、一个完整 Recovery Mental Model

可以把实时恢复压缩成：

Detect
→ Isolate
→ Retry
→ Resume
→ Replay if needed
→ Reconcile

其中：

Detect = 发现错误、Lag、下游故障。

Isolate = DLQ / pause / partition-level isolation。

Retry = 处理 transient failure。

Resume = 从 committed position 继续。

Replay = 重处理历史范围。

Reconcile = 检查最终数据是否正确。

---

## 二十一、和前几课连起来

Lesson 3：Offset tells us where we are.

Lesson 4：Consumer Group decides who owns which partition.

Lesson 5：Delivery Semantics decides what happens around commit and failure.

Lesson 6：Retry / Replay / Backpressure decides how the system survives ongoing failure and overload.

这四节实际上构成了 Kafka Consumer Reliability 的完整基础。

---

## 二十二、Blockchain Data Engineer 视角

对于链上实时 Pipeline，你真正需要设计的是：

Happy Path + Failure Path + Repair Path

Happy Path：

Event → Transform → Sink → Commit

Failure Path：

Retry / Backoff / Pause / DLQ

Repair Path：

Replay / Backfill / Reconciliation

只设计 Happy Path 的实时系统，不算完整的生产系统。

---

## 本课核心结论

> Retry handles local transient failures; Replay reprocesses historical events.

> Retry should be bounded and usually use backoff.

> Backpressure appears when downstream processing capacity cannot keep up with upstream arrival rate.

> Consumer Lag is the key signal of how far behind a consumer is.

> Kafka can absorb temporary lag, but Retention defines the recovery window.

> Replay requires retained data, a known position, and idempotent processing.

> Production-grade recovery needs Retry, Replay, Backpressure control, and a repair path—not just restart.

---

## 理解检查

### 问题一

假设 Postgres 暂时不可用 30 秒，某个 Transfer Event 写入失败。

这里更适合 Retry 还是 Replay？

为什么不能无限立即 Retry？

### 问题二

假设：

Producer Rate = 10,000 msg/s
Consumer Rate = 6,000 msg/s

如果持续 60 秒，理论上大约会新增多少条 Lag？

这说明系统处于什么状态？

### 问题三

假设某个 Decoder Bug 导致过去 50,000 条 Event 的结果错误，而 Kafka Retention 仍然保留这些 Event。

为什么这里应该使用 Replay，而不是普通 Retry？

要让 Replay 安全成立，至少需要哪几个前提？

## 用户回答

问题一，这里更适合 Retry，但是不能无限立即 Retry。
无限立即 Retry 的话，会导致系统资源被 Retry 一直占用，使得 Postgres 的问题更不好解决。
问题二：如果持续 60 秒，理论上大约会新增 24 万条 lag，说明系统 producer 的速度远大于 consumer 的速度，会导致 lag 越积越多
问题 3：

这里使用 replay 而不是普通的 retry，因为这是 decoder bug 导致的数据问题，而不是系统的临时故障，所以需要 replay 重跑，重新 consume 历史的数据。

而正好这些历史数据还在 Kafka retention 之中仍然保留着，所以这个时候 replay 是可行的。

## 老师判断与校准

- 问题一：通过。这里属于 transient downstream failure，更适合 Retry。不能无限立即 Retry，因为会形成 retry storm，持续占用连接、线程和数据库资源，反而加剧下游故障。更合理的是 bounded retry + backoff。
- 问题二：通过。Producer 与 Consumer 的速率差是 4,000 msg/s，持续 60 秒会新增约 240,000 条 Lag。说明 incoming rate 持续高于 processing rate；如果这个趋势持续，就是结构性 Backpressure，Lag 会不断扩大。
- 问题三：判断方向正确，但尚未完整通过。Decoder Bug 导致的是一段历史数据已经被错误处理，不是某一条 Event 的临时失败，因此应做 Replay / historical reprocessing，而不是普通 Retry。你也正确指出 Kafka Retention 中数据仍然存在，因此具备 Replay 的第一个前提。还需要补充另外两个关键前提：需要知道从哪个 Topic / Partition / Offset 范围重新开始（Position），并且重复处理必须是安全的（Idempotent Processing）。
