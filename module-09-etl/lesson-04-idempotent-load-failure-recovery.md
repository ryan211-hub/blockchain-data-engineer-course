# 第4课｜Idempotent Load 与失败恢复：为什么 ETL 必须可安全重跑

【Lesson Contract】
所属 Module：Module 9 — ETL。
本课核心问题：
> 一个 ETL Job 失败后重新执行，为什么不能简单地“再 INSERT 一遍”？怎样设计 Load，才能让同一个 Input Range 重跑多次而不会把 Target 搞坏？
学完本课，你应该能够：
- 准确定义 ETL 场景中的 Idempotency；
- 理解为什么失败恢复天然要求“允许重跑”；
- 区分 Append、Upsert、Delete + Insert / Replace Partition 等常见 Load 策略；
- 根据 Target Grain / Unique Key 设计幂等 Load；
- 判断什么情况下“唯一键去重”足够，什么情况下需要整段重算；
- 理解 Checkpoint 和 Idempotent Load 为什么必须配合使用。
本课暂时不展开：
- 数据库具体 Merge Engine 实现；
- ClickHouse / Postgres 选型；
- 分布式 Exactly Once；
- Kafka 语义；
- 系统性 Data Repair Framework。
这些属于后续 Module。

---

## 一、先从上一课的失败场景继续
上一课有这个场景：
```text
checkpoint = 09-14
```
开始处理：
```text
09-15
```
执行：
```text
Extract
✓

Transform
✓

Load
写了一半
↓

Crash
```
因为 Load 没有完整成功，所以：
```text
checkpoint
仍然是
09-14
```
重启以后：
```text
重新处理 09-15
```
到这里没有问题。
但真正麻烦的问题是：
> 第一次 09-15 已经写进去一部分数据了。
假设一天应该产生：
```text
100,000 rows
```
第一次执行已经写了：
```text
60,000 rows
```
然后程序挂了。
第二次重跑，又生成完整：
```text
100,000 rows
```
如果 Load 只是：
```sql
INSERT INTO target
SELECT ...
```
会发生：
```text
原来的 60,000
+
重新写入的 100,000
=
160,000
```
其中有：
```text
60,000
```
条重复。
所以：
> 只会“从正确位置重跑”还不够。
还必须保证：
> 重跑本身是安全的。
这就是 Idempotency。

---

## 二、Idempotency 到底是什么意思？
Idempotency 中文一般翻译成：
```text
幂等
```
它的工程含义是：
> 同一个操作执行一次和执行多次，最终系统状态应保持一致。
写成：
```text
f(x)

f(f(x))

f(f(f(x)))
```
最终结果应该一样。
在 ETL 场景里，就是：
```text
run_job(09-15)
```
执行一次：
```text
Target = 正确的 09-15 数据
```
执行十次：
```text
Target
仍然 =
正确的 09-15 数据
```
而不能变成：
```text
10 份重复数据
```

---

## 三、这里要区分“程序执行次数”和“业务结果”
【Data Engineer 视角】
Idempotency 并不是说：
```text
第二次什么都不做
```
它完全可能重新：
```text
Extract
Transform
Load
```
甚至消耗同样的计算资源。
但最终：
```text
Target State
```
必须相同。
例如：
```text
第一次运行
100 rows
```
第二次运行：
```text
还是 100 rows
```
第三次：
```text
还是 100 rows
```
所以真正关心的是：
> 最终数据状态，而不是程序有没有重复执行。

---

## 四、为什么 ETL 必须假设“任务一定会重跑”？
生产环境里失败是正常现象。
可能出现：
```text
RPC timeout
database timeout
network disconnect
process crash
machine restart
OOM
disk full
SQL error
schema change
dependency failure
```
所以一个生产 ETL 如果设计成：
> “这个 Job 只能执行一次，第二次就会产生重复”
这种设计本身就不可靠。
更合理的工程假设应该是：
> 任何 Processing Unit 都可能被重跑。
例如：
```text
09-15
```
应该允许：
```text
run_job(09-15)

run_job(09-15)

run_job(09-15)
```
而最终数据完全一样。

---

## 五、第一种最简单的 Load：纯 INSERT
假设 Target：
```text
fact_token_transfers
```
Load：
```sql
INSERT INTO fact_token_transfers
SELECT ...
```
如果 Input 重放两次：
```text
Transfer A
Transfer B
Transfer C
```
第一次：
```text
A
B
C
```
第二次又：
```text
A
B
C
```
结果：
```text
A
B
C
A
B
C
```
显然不幂等。
所以：
```text
INSERT
```
本身并不能保证 Idempotency。

---

## 六、Unique Key 是幂等性的核心基础之一
你在 Module 7 和 Module 8 已经反复碰到：
```text
chain_id
tx_hash
log_index
```
对于 ERC-20 Transfer，这可以作为稳定身份。
所以：
```text
Unique Key
=
chain_id
+ tx_hash
+ log_index
```
这时候相同 Transfer 重跑：
```text
同一个 chain_id
同一个 tx_hash
同一个 log_index
```
系统应该识别：
> 这不是一条新的 Transfer，这是同一条事实。
所以 Idempotent Load 的第一层基础就是：
> 你必须知道“一条 Target Row 到底是谁”。
也就是：
```text
Row Identity
```

---

## 七、为什么 Grain 和 Unique Key 在这里又回来了？
Module 8 学 Grain 时，我们强调：
> 一张表的一行代表什么？
现在你会发现，这不是纯建模理论。
因为只有知道：
```text
一行代表什么
```
才能知道：
```text
哪两行是同一条数据
```
进一步才能判断：
```text
重跑时应该 INSERT
还是 UPDATE
还是替换
```
例如：
```text
fact_token_transfers
```
Grain：
```text
one token transfer event
```
Unique Key：
```text
chain_id
+ tx_hash
+ log_index
```
而：
```text
dws_wallet_token_daily_flow
```
Grain：
```text
one wallet
+ one token
+ one day
+ one chain
```
Unique Key：
```text
chain_id
+ date
+ wallet_address
+ token_address
```
两个 Target 的 Grain 不同，因此幂等 Load 策略也可能不同。

---

## 八、策略一：Insert If Not Exists
一种思路：
```text
先检查是否存在
不存在 → INSERT
存在 → Skip
```
例如：
```sql
INSERT ...
WHERE NOT EXISTS (...)
```
或者依赖 Unique Constraint。
这样重复 Transfer：
```text
A
```
第二次来：
```text
A
```
就不会再写一行。
这种方案适合：
> 已经写入的数据永远不需要修改。
例如某些 immutable facts。

---

## 九、但 Insert If Not Exists 有一个问题
假设第一次计算错了。
例如：
```text
amount_usd = 95
```
后来修复价格逻辑，正确应该是：
```text
amount_usd = 100
```
现在 Backfill 重跑。
如果你的逻辑是：
```text
存在就跳过
```
结果：
```text
95
```
永远不会被修正。
所以：
> “防止重复”不等于“完整的幂等设计”。
这是非常重要的区别。

---

## 十、幂等不是单纯 Deduplication
Deduplication：
```text
不要产生两条相同记录
```
Idempotency：
```text
无论执行多少次
最终 Target 都应该达到正确状态
```
两者有关，但不完全一样。
例如：
```text
旧值 = 95
新正确值 = 100
```
如果你只去重：
```text
仍然保留 95
```
没有重复，但结果还是错的。
所以：
> Idempotent Load 更关注“最终正确状态”。

---

## 十一、策略二：Upsert
Upsert 可以理解为：
```text
存在 → UPDATE
不存在 → INSERT
```
例如：
```text
Unique Key
=
chain_id
+ date
+ wallet
+ token
```
第一次计算：
```text
received_amount = 100
sent_amount = 30
```
后来重跑：
```text
received_amount = 120
sent_amount = 30
```
Upsert：
```text
同一个 Unique Key
```
把原来那一行更新成：
```text
120 / 30
```
最终 Target 中仍然只有：
```text
1 row
```
而且值是新的正确结果。
这就是一个典型 Idempotent Load。

---

## 十二、Upsert 的核心不是 SQL 语法
不同数据库可能叫：
```text
MERGE
UPSERT
INSERT ... ON CONFLICT
INSERT ... ON DUPLICATE KEY UPDATE
```
具体语法不是现在重点。
真正重点是：
> Target 必须存在稳定 Identity，系统才能知道“更新哪一行”。
所以顺序应该是：
```text
Define Grain
↓
Define Unique Key
↓
Choose Load Strategy
```
不是反过来。

---

## 十三、策略三：Delete + Insert
对于 Daily Aggregate，另外一种很常见的方法其实更简单。
例如 Target：
```text
dws_wallet_token_daily_flow
```
你要重新处理：
```text
2026-09-15
```
可以先：
```sql
DELETE
FROM dws_wallet_token_daily_flow
WHERE date = '2026-09-15';
```
然后重新：
```sql
INSERT
完整的 09-15 结果
```
也就是：
```text
Delete Range
↓
Recompute
↓
Insert Complete Range
```
这样无论跑多少次：
```text
09-15
```
最后都只有一份完整结果。

---

## 十四、为什么 Delete + Insert 很适合聚合表？
假设：
```text
09-15
wallet A
USDC
```
第一次算：
```text
received = 100
sent = 30
```
后来 Source 增加了漏掉的数据。
现在正确应该：
```text
received = 120
sent = 30
```
如果只 Upsert 某些增量结果，很容易遇到：
```text
是把 20 加进去？
还是重新写成 120？
```
逻辑容易复杂。
而 Delete + Insert：
```text
DELETE 09-15
↓
从 Source 完整重算 09-15
↓
INSERT
```
思维非常直接。
所以很多 Batch ETL 会把一个 Partition 当成：
> 可整体重建的 Processing Unit。

---

## 十五、这就是为什么 Processing Unit 很重要
上一课我们说：
```text
Processing Unit
=
one day
```
那么 Idempotent Load 就可以设计成：
```text
重建 one day
```
例如：
```text
09-15
```
整个生命周期：
```text
Extract 09-15
↓
Transform 09-15
↓
Replace Target 09-15
↓
Checkpoint 09-15
```
如果失败：
```text
重新执行整个 09-15
```
不需要记：
```text
当天已经处理到第几百万行
```
这大幅简化恢复逻辑。

---

## 十六、Replace Partition 是 Delete + Insert 的进一步抽象
如果底层数据库支持 Partition：
```text
date = 2026-09-15
```
你可以把：
```text
DELETE + INSERT
```
理解成：
```text
Replace Partition
```
也就是：
```text
旧的 09-15 partition
↓
被新的完整 09-15 partition 替换
```
这是一种非常自然的 Batch ETL 思路。
现在不用关注具体数据库实现。
你只需要记住：
> 对于可以按 Range / Partition 完整重算的数据，整体替换通常比逐行修补更简单。

---

## 十七、事实表和聚合表的 Load 思维可能不同
【Data Modeling + ETL 视角】
以两个表为例。
事实表：
```text
fact_token_transfers
```
每一行是：
```text
one event
```
更常见：
```text
Unique Key
+
Insert / Upsert
```
因为每条 Event 有稳定 Source Identity。
而聚合表：
```text
dws_wallet_token_daily_flow
```
每一行是：
```text
one wallet-token-day
```
更适合：
```text
Recompute Partition
+
Replace
```
因为它是 Derivative / Aggregate。
所以：
> Idempotency 不是“一种固定 SQL 写法”。
它必须结合：
```text
Target Grain
Target Identity
Processing Unit
Data mutability
```
一起设计。

---

## 十八、拿银行数仓做类比
假设银行有：
```text
dws_customer_daily_transaction
```
每天生成：
```text
2026-09-15
```
如果 09-15 Job 跑失败。
一种常见做法：
```text
DELETE date = 09-15
↓
重新聚合交易流水
↓
INSERT date = 09-15
```
而不是：
```text
记住上次插到第 843,291 行
然后从第 843,292 行继续
```
为什么？
因为后者复杂很多。
如果一天数据可以在合理成本内重算：
> 重算一个完整 Processing Unit 往往比精细恢复更可靠。

---

## 十九、为什么“继续从失败的那一行开始”不一定是好设计？
假设一天：
```text
10,000,000 rows
```
失败在：
```text
row 6,382,417
```
看起来最节省资源的方式：
```text
从 6,382,418 继续
```
但你需要维护：
```text
row-level cursor
partial output state
transaction boundary
exact ordering
retry semantics
```
系统复杂度非常高。
而另一种方式：
```text
重跑 09-15
```
虽然多计算一些数据，但逻辑简单很多。
这就是经典的数据工程权衡：
```text
Compute Cost
vs
System Complexity
```
很多时候：
> 宁愿多算一点，也不要把恢复逻辑设计得过于脆弱。

---

## 二十、Checkpoint 为什么一定要在 Load 成功后更新？
现在可以把上一课讲得更完整。
正确流程：
```text
Input Range
↓
Extract
↓
Transform
↓
Idempotent Load
↓
Validate
↓
Advance Checkpoint
```
注意：
```text
Idempotent Load
```
在 Checkpoint 之前。
因为如果：
```text
Checkpoint 先推进
```
然后 Load 失败：
```text
系统认为完成
但 Target 没完成
```
会产生数据缺失。
所以：
> Checkpoint 是“完成声明”，不能是“开始声明”。

---

## 二十一、如果 Load 成功了，但 Checkpoint 更新失败呢？
这是一个非常经典的问题。
场景：
```text
09-15 Load
✓
```
Target 已经完整写入。
然后：
```text
update checkpoint
```
失败。
所以状态：
```text
Target:
09-15 已存在

Checkpoint:
仍然 09-14
```
系统重启后：
```text
重新执行 09-15
```
这时候为什么不会出问题？
答案：
```text
Idempotent Load
```
因为第二次执行：
```text
仍然得到同样 Target State
```
然后：
```text
checkpoint → 09-15
```
所以你现在可以看到：
> Idempotency 甚至可以解决“数据已经成功，但进度状态没来得及更新”的失败窗口。

---

## 二十二、这是 Checkpoint + Idempotency 最重要的组合
可以记成：
```text
Checkpoint
=
允许保守地重跑
```
Idempotency：
```text
让重跑变得安全
```
两者组合：
```text
失败时宁可重复处理
也不要跳过数据
```
这是一个非常重要的数据工程原则。
也就是说：
> At-least-once processing + idempotent write，通常比“绝不重复执行”更现实。
这里先理解思想，不展开 Streaming 的 Exactly Once。

---

## 二十三、为什么宁愿重复，也不能漏？
假设两个错误：
A：
```text
09-15 重跑了一次
```
但因为幂等：
```text
结果还是正确
```
B：
```text
09-15 被错误跳过
```
那么：
```text
数据永久缺失
```
所以在数据管道设计中，通常更容易接受：
```text
可能重复执行
```
前提是：
```text
重复执行安全
```
而不能接受：
```text
某段数据完全没处理
```

---

## 二十四、Blockchain 场景尤其适合这种思维
区块链历史数据天然适合 Replay。
例如：
```text
block 20,000,000
~
20,001,000
```
如果某段处理失败：
```text
重新 Fetch
重新 Decode
重新 Normalize
重新 Write
```
只要：
```text
Stable Identity
+
Idempotent Write
```
存在，就可以安全 Replay。
这和你 Module 7 Mini Indexer 完全一致。
Module 7：
```text
(chain_id, tx_hash, log_index)
```
保证 Event Identity。
Module 9：
```text
Target Grain + Unique Key
```
保证 ETL Target Identity。
两个其实是一条逻辑链。

---

## 二十五、但 Blockchain 还有一个特殊问题：Reorg
这里只点到为止。
即使一个 Load 是 Idempotent：
```text
同一 Event 重跑不会重复
```
也不代表：
```text
旧链数据会自动消失
```
Reorg 会产生：
```text
old canonical branch
→ invalid
```
这不是简单的：
```text
重复写入
```
而是：
```text
已有数据后来失效
```
所以：
> Idempotency 解决 Retry / Replay 安全性，但不能单独解决 Reorg Correctness。
Reorg 会在 Module 12 数据质量 / Repair 中系统展开。
本课不往下钻。

---

## 二十六、一个实际设计：fact_token_transfers
假设：
```text
fact_token_transfers
```
Grain：
```text
one Transfer Event
```
Unique Key：
```text
chain_id
tx_hash
log_index
```
一个合理 Load：
```text
Extract block range
↓
Decode Transfer
↓
Normalize
↓
Upsert / Insert with Unique Key
```
重跑同一个 block range：
```text
不会增加重复 Transfer
```
这就是：
```text
event-level idempotency
```

---

## 二十七、再设计 dws_wallet_token_daily_flow
Target：
```text
dws_wallet_token_daily_flow
```
Grain：
```text
chain_id
+ date
+ wallet_address
+ token_address
```
Processing Unit：
```text
one day
```
一个更简单的 Load：
```text
DELETE target
WHERE date = :date

↓

INSERT
完整重新计算的 daily result
```
或者抽象成：
```text
replace partition(date)
```
于是：
```text
run_job(09-15)
```
执行 1 次和执行 10 次：
```text
最终 Target 相同
```
这就是：
```text
partition-level idempotency
```

---

## 二十八、一个容易犯的错误：增量加法
假设每天统计：
```text
daily_volume
```
你写：
```sql
UPDATE target
SET volume = volume + :new_volume;
```
第一次：
```text
volume = 100
```
重跑以后：
```text
volume = 200
```
再次：
```text
300
```
这显然不是 Idempotent。
因为：
```text
+=
```
天然依赖：
```text
“这个操作只执行一次”
```
所以非常危险。

---

## 二十九、什么时候“加法”才安全？
如果系统能明确保证：
```text
每个输入事件只消费一次
```
理论上可以。
但生产系统里：
```text
Retry
Replay
Backfill
Failure Recovery
```
都会导致同一输入再次到达。
因此 Batch ETL 更常见、更稳健的思路是：
> 从明确 Source Range 重新计算目标状态。
例如：
```text
SUM(all source rows for 09-15)
```
得到：
```text
100
```
无论重跑几次：
```text
还是 100
```
而不是：
```text
previous target + delta
```

---

## 三十、这里出现两个不同思维
第一种：
```text
State Mutation
```
例如：
```text
current_value += 20
```
第二种：
```text
State Reconstruction
```
例如：
```text
current_value
=
SUM(source range)
```
【Blockchain Data Engineer 视角】
对于可 Replay 的历史数据：
> State Reconstruction 往往比依赖一次性 Mutation 更容易保证正确性。
这和 Blockchain 本身其实也有一点精神上的相似：
```text
History
→ derive state
```
当然这里只是抽象类比。

---

## 三十一、判断一个 Load 是否幂等，可以问一个简单问题
假设同一个：
```text
Input Range
```
执行两次。
问：
> 第二次运行以后 Target 会不会变化？
如果：
```text
第一次 → 100 rows
第二次 → 200 rows
```
不是幂等。
如果：
```text
第一次 → 100 rows
第二次 → 100 rows
```
而且内容完全一致：
```text
Idempotent
```
所以测试也很简单：
```text
Run once
↓
Snapshot / checksum / row count

Run again
↓
Compare
```
Module 7 的：
```text
Idempotent Replay Test
```
本质就是这个。

---

## 三十二、失败恢复的完整模型
到现在我们已经可以画出一个完整 Batch ETL：
```text
Read Checkpoint
      ↓
Determine Input Range
      ↓
Extract
      ↓
Transform
      ↓
Idempotent Load
      ↓
Validate
      ↓
Advance Checkpoint
```
失败发生在任何位置：
```text
↓
Restart
↓
Read old checkpoint
↓
Replay same Processing Unit
```
由于：
```text
Load is idempotent
```
所以：
```text
Replay is safe
```
这就是一个基本可靠的数据管道。

---

## 三十三、本课最重要的工程结论
可以压缩成一句话：
> Checkpoint 负责“不漏”，Idempotency 负责“不怕重复”。
更完整一点：
```text
Checkpoint
→ 保守记录最后成功位置

Idempotent Load
→ 同一 Input Range 可以安全重复执行
```
两者组合：
```text
Failure
↓
Replay
↓
Safe Recovery
```

---

## 三十四、和前 3 课串起来
现在 Module 9 已经开始形成一条完整链路：
```text
Lesson 1
ETL = 可持续 Pipeline

↓

Lesson 2
每次到底处理哪段数据？
Full / Incremental / Backfill

↓

Lesson 3
怎么记住处理到哪里？
Watermark / Cursor / Checkpoint

↓

Lesson 4
失败以后怎么安全重跑？
Idempotent Load
```
这四课其实在回答同一个生产问题：
> 如何让一个 ETL 不只是“能跑”，而是“长期可靠地跑”。

---

# 第 4 课理解检查
问题一：
有一个 Daily ETL：
```text
checkpoint = 09-14
```
09-15 第一次执行时：
```text
100,000 rows
```
只写入了前：
```text
60,000 rows
```
程序就崩溃。
重启后又完整重新计算 09-15。
如果 Load 只是普通 INSERT，会发生什么？
你认为正确设计至少应该满足什么条件？

问题二：
Target：
```text
dws_wallet_token_daily_flow
```
Grain：
```text
chain_id
+ date
+ wallet_address
+ token_address
```
现在要重跑：
```text
2026-09-15
```
两种方案：
A：
```text
把重新计算出来的 sent_amount / received_amount
继续加到原来 Target 上
```
B：
```text
先删除 / 替换 09-15 这个 Processing Unit，
再写入重新计算后的完整 09-15 结果
```
你选哪个？为什么？

问题三：
发生这个情况：
```text
09-15 Target Load
成功

↓

Checkpoint 更新失败
```
因此系统现在：
```text
Target 已经有完整 09-15

Checkpoint 仍然是 09-14
```
服务重启后会再次处理 09-15。
为什么一个正确设计的 ETL 不应该害怕这种情况？
请结合：
```text
Checkpoint
+
Idempotent Load
```
一起回答。

---

## 用户回答
问题一：如果 load 只是普通 insert 会发生什么呢？

只是普通 insert 的话，那么 0519 的数据会有部分重复。正确的处理设计应该满足 idempotent（幂等）这个条件，就算重复执行当天的 ETL，结果也会保持一致
问题 2 选用方案 B。因为这个处理方式更简洁、简单明了；方案 A 的话，它是重复累加，会导致数据错误
问题三，因为这个设计Checkpoint 记录已经处理过的数据，然后 idempotent 负责保持记录结果的一致性。

所以在这个设计下，不用担心数据重跑会有什么问题

## 老师判断与校准
三题都答对了。第 1 题里的“0519”按上下文应是“09-15”，不影响你的判断；核心结论正确：普通 INSERT 会让已写入部分在重跑时重复，而幂等 Load 要保证同一日任务重复执行后 Target 状态仍一致。

第 3 题再精确一点：Checkpoint 不是“记录已经处理过的数据本身”，而是记录“最后安全完成的位置”；Idempotent Load 则保证从这个位置之后重跑同一个 Processing Unit 时不会把 Target 搞坏。你已经抓住了两者的配合关系。

## 结课判定
Module 9 第 4 课理解检查通过，必要校准完成，本课正式结束。
