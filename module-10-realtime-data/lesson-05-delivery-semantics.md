## Lesson Contract

所属 Module：Module 10 — 实时数据

本课核心问题：

> Consumer 读取一条 Kafka Event 后，业务处理和 Offset Commit 的先后顺序不同，会产生什么 failure semantics？

学完以后，你应该能够解释：

- At-most-once、At-least-once、Exactly-once 的工程含义。
- 为什么“先 Commit 再处理”容易 data loss。
- 为什么“先处理再 Commit”容易 duplicate processing。
- 为什么 At-least-once 通常需要 idempotent sink。
- 为什么 Exactly-once 不是一句简单的 Kafka 配置，而是 end-to-end consistency 问题。
- Kafka Offset、业务数据库写入、外部 side effect 三者之间为什么很难天然原子化。

本课不深入 Kafka Transaction Protocol、Producer Idempotence、Flink Checkpoint Barrier 等内部实现，只建立 Data Engineer 必须掌握的 delivery semantics 心智模型。

---

## 一、先从一个最简单的 Event 开始

假设 Kafka 里有一条记录：

Topic = wallet_transfers
Partition = P0
Offset = 100

Event:
Alice +100 USDC

Consumer 读取它以后，需要做两件事：

1. Process the event
2. Commit the offset

例如业务处理可能是更新 wallet_balance，随后提交 P0 的消费进度。

问题来了：

> Which one should happen first?

这就是 Delivery Semantics 的核心。

---

## 二、先理解两个动作不是一回事

【Kafka 视角】

Offset Commit 表示：

> “这个 Consumer Group 认为自己已经处理到这里。”

【业务系统视角】

Database Update 表示：

> “业务结果已经真正产生。”

这两个系统通常是分开的，例如 Kafka + Postgres，所以它们天然不是一个原子操作。

这意味着两类故障都可能发生：

- DB write succeeds but offset commit fails
- offset commit succeeds but DB write fails

这就是问题的根源。

---

## 三、方案 A：先 Commit，再处理

流程：

Read Event 100 → Commit Offset 100 → Update Database

如果在 Commit 后、Database Update 前 crash：

Read Event 100 → Commit Offset 100 → CRASH → Database not updated

Consumer 重启以后会认为 Offset 100 已经完成，于是从后续位置继续。Event 100 不会再次处理。

这就是：Data Loss。

这种语义更接近：

> At-most-once.

意思不是“保证处理一次”，而是：

> An event is processed zero or one time.

可能一次，也可能一次都没有。

---

## 四、At-most-once

At-most-once 的核心特征：

> No duplicates, but data may be lost.

典型思想是：Commit first, Process later。

如果中间 crash，就会出现：Offset says done, Business says not done。

它适合某些 non-critical metrics、best-effort telemetry、temporary monitoring signals 等“少一条问题不大”的场景。

但 Blockchain Financial Data 的核心事实处理通常不适合优先采用这种语义。

---

## 五、方案 B：先处理，再 Commit

另一种流程：

Read Event 100 → Update Database → Commit Offset 100

假设 Database update succeeds，但 Consumer 在 Commit Offset 前 crash。

Consumer 重启后看到 Committed Offset 仍然停留在旧位置，于是会重新读取 Event 100。

结果是同一个 Event 可能再次执行 Database Update。

这就产生：

> Duplicate Processing.

这就是：

> At-least-once.

---

## 六、At-least-once

At-least-once 的含义：

> An event will be processed one or more times.

重点是 one or more，不是“刚好一次”。

它优先保证：

> Do not lose the event.

代价是：

> Duplicates are possible.

这在 Data Engineering 中非常常见，因为很多系统宁可接受 duplicate，也不愿接受 silent data loss。

---

## 七、这为什么和 Idempotency 直接连接？

你在 Module 9 已经学过：

> Retry + At-least-once usually requires idempotency.

假设 Transfer Event：

chain_id = 1
tx_hash = 0xabc
log_index = 7
amount = 100

第一次处理插入 Transfer，Consumer crash，重启后 Event 被重新消费。

如果仍然直接 INSERT，就可能插入两行。

所以可以使用：

UNIQUE(chain_id, tx_hash, log_index)

或者：

UPSERT / ON CONFLICT DO NOTHING

这样同一 Event 即使处理两次，Business result remains the same。

这就是 idempotent processing。

---

## 八、Set-style vs Add-style

假设余额更新写成：

balance = balance + 100

同一 Event 重放两次就会变成 +200。

这是 Add-style update，它天然不幂等。

如果业务结果可以通过唯一 Event Identity 去重后再计算，或者写成 set-style recomputation，就更容易实现幂等。

所以：

> At-least-once delivery shifts correctness responsibility to the consumer and sink.

---

## 九、At-least-once 为什么常见？

因为它的 failure model 更保守。

遇到不确定情况时，它倾向：

> Replay the event.

而不是：

> Assume it was done.

对于 Blockchain Data Pipeline，这通常更合理。

如果不知道一条 Transfer / Swap / Balance Update 是否真正处理成功，更安全的策略通常是 process again + deduplicate / idempotent write，而不是直接跳过。

---

## 十、Exactly-once 听起来最完美

直觉上我们希望：

> Every event is processed exactly once.

也就是 No loss + No duplicate effect。

但工程上的问题是：

> What does "exactly once" mean across multiple systems?

例如：Kafka → Consumer → Postgres → Send Alert API。

一个 Event 可能产生多个 side effects：DB updated、Offset committed、Alert sent。

要做到 all succeed once or all fail together，非常困难。

---

## 十一、Exactly-once 本质是 Atomicity 问题

现实里 Kafka、Postgres、External API 通常是三个独立系统。

例如：DB success → Alert API success → Offset commit fails。

重启后 Event Replay，那么 DB 也许可以幂等，但 Alert 可能发送两次。

所以：

> End-to-end exactly-once is much harder than exactly-once inside one subsystem.

---

## 十二、Exactly-once 不等于“Kafka 配置打开”

有些 Kafka 功能或 Stream Processing Framework 会提供 exactly-once semantics，但必须追问：

> Exactly once within what boundary?

例如，它可能只保证 Kafka Topic A → Transform → Kafka Topic B 这条链路。

如果最终还写 Postgres、Email、Webhook 或其他 External API，就需要额外机制。

所以 Data Engineer 不应该只记“Kafka supports exactly-once”，而要问：

> Which state transitions are atomic?

---

## 十三、三个语义的核心比较

At-most-once:
- Typical order: Commit → Process
- Main risk: Data loss

At-least-once:
- Typical order: Process → Commit
- Main risk: Duplicate processing

Exactly-once:
- Requires an atomic or coordinated processing boundary
- Main cost: Higher complexity

可以压缩成：

> At-most-once = no duplicates, possible loss.

> At-least-once = no intentional loss, possible duplicates.

> Exactly-once = no duplicate effect and no loss within a defined boundary.

---

## 十四、这里要区分 Delivery 和 Effect

Kafka 可能把同一个 Event delivery 多次。

但如果 Consumer Sink 是幂等的，例如使用 UNIQUE KEY + ON CONFLICT DO NOTHING，最终数据库里仍然只有一条。

所以：

> Duplicate delivery does not necessarily mean duplicate business effect.

很多系统真正追求的是 effectively-once business outcome。

可以形成一个工程近似：

At-least-once delivery + Idempotent processing ≈ Exactly-once effect

注意：这不表示 Kafka 协议层本身已经变成 Exactly-once。

---

## 十五、Blockchain Example：Transfer Fact

假设 Consumer 负责写入 fact_token_transfer。

唯一键：

(chain_id, tx_hash, log_index)

处理逻辑使用 INSERT ... ON CONFLICT DO NOTHING。

Kafka 使用 At-least-once。

如果 DB write success → Consumer crash → Offset not committed → Event replay，第二次写入会被唯一键挡住。

最终结果仍然是：1 Event → 1 Row。

这就是典型的：

> At-least-once delivery + idempotent sink.

---

## 十六、Blockchain Example：Wallet Balance

Wallet Balance 更麻烦。

如果直接执行 balance = balance + amount，那么重复消费会重复累加。

所以通常需要设计：

- event deduplication
- idempotency table
- processed-event marker
- set-style recomputation
- state versioning

核心目标都是：

> make replay safe.

因为 Streaming 系统里 Replay 不是异常，而是正常能力。

---

## 十七、银行系统类比

假设银行交易流水有 transaction_id = T123。

第一次 INSERT transaction T123，Consumer crash 后重放，第二次又尝试 INSERT T123。

如果 transaction_id 有唯一约束，第二次就会被识别为重复。

这和 Blockchain 的 (chain_id, tx_hash, log_index) 完全是同一类设计思想。

所以可以把 Blockchain Event Identity 理解成：

> replay-safe business key.

---

## 十八、什么时候 Commit Offset？

一个 conservative pattern 是：

Read → Process → Persist successfully → Commit Offset

也就是：

> Commit after successful processing.

这样更接近 At-least-once，因为 crash 时最多导致 replay。

但这不等于“完全安全”。如果业务写入不是幂等，Replay 仍会产生 duplicate side effect。

---

## 十九、Consumer Offset 不是 Business Transaction

Offset Commit 只表示 Kafka 消费进度。

它不代表：

- Database transaction committed
- External API succeeded
- Downstream state correct

因此：

> Offset is a processing progress marker, not a proof of business correctness.

这和 Module 9 的 Checkpoint 很像：Checkpoint 也不是 Business correctness itself，它只是记录 pipeline believes this boundary is complete。

---

## 二十、Exactly-once 的常见工程思路

本课不深入底层，只认识几个方向：

1. Idempotent Sink：At-least-once + Unique Key / UPSERT。
2. Transactional Outbox：业务数据和待发送 Event 放入同一个 DB Transaction。
3. Kafka Transactions：在 Kafka 内部 consume-transform-produce 场景协调 records 和 offsets。
4. Stream Processor State + Checkpoint：例如 Flink 通过 state snapshot / checkpoint 协调处理状态。

这些方案本质上都是：

> define and enforce an atomic boundary.

不是魔法。

---

## 二十一、Blockchain Reorg 又增加一层复杂度

即使你实现了 exactly-once processing，Blockchain 还可能发生：

processed fact → later becomes orphaned

所以 Blockchain Data Engineer 还要同时处理：

delivery correctness + canonical-chain correctness

这是两个不同问题。

Exactly-once 解决：同一条输入 Event 被处理几次？

Reorg 解决：这条 Event 后来还算不算 canonical fact？

这一点会在第 7 课正式结合。

---

## 二十二、从系统设计角度怎么选？

通常不是：Exactly-once is always best。

而是：

> Choose the simplest semantics that preserves business correctness.

对于很多 Data Pipeline：

At-least-once + Idempotent Sink

已经是非常成熟、常见、可恢复的设计，因为它 avoids silent loss、supports replay、works well with retries，并且实现相对可理解。

代价是必须认真设计幂等。

---

## 二十三、把 Module 9 和 Module 10 接起来

Module 9：Retry + Backfill + Idempotency

Module 10：Replay + At-least-once + Idempotency

实际上是同一个工程哲学：

> If data may be processed again, make repeated processing safe.

所以 Idempotency 不是 Batch ETL 的局部技巧，而是 Data Engineering 中非常核心的 reliability pattern。

---

## 本课核心结论

第一：
> At-most-once prioritizes avoiding duplicates, but may lose data.

第二：
> At-least-once prioritizes avoiding loss, but may process duplicates.

第三：
> At-least-once usually requires idempotent processing or an idempotent sink.

第四：
> Exactly-once only makes sense within a clearly defined atomic boundary.

第五：
> Offset Commit records consumer progress; it does not prove that all business side effects are correct.

第六：
> In practice, At-least-once + Idempotency is often a pragmatic design for reliable data pipelines.

---

## 理解检查

### 问题一

流程：Read Event 100 → Commit Offset 100 → CRASH → Database write did not happen。

这更接近哪一种 Delivery Semantics？最终最主要的风险是什么？

### 问题二

流程：Read Event 100 → Database write succeeds → CRASH → Offset 100 was not committed。

Consumer 重启后会发生什么？为什么这种模式通常需要 Idempotency？

### 问题三

假设 Transfer Consumer 使用 UNIQUE(chain_id, tx_hash, log_index) + INSERT ... ON CONFLICT DO NOTHING，Kafka 采用 At-least-once。

请解释：为什么 Event 即使被重复 Delivery，最终数据库仍然可以只有一条 Transfer Fact？

以及，这是否意味着 Kafka 本身已经变成 Exactly-once？