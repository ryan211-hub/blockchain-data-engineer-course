# 第5课｜Late-arriving Data 与 Reorg：为什么已经处理的数据后来还会变化

【Lesson Contract】
所属 Module：Module 9 — ETL。
本课核心问题：
> 我们明明已经把某一天、某个 Block Range 处理完成并推进了 Checkpoint，为什么后来还可能发现：历史数据缺了一条、增加了一条，甚至原来的一条链上数据已经不成立？
学完本课，你应该能够：
- 区分 Late-arriving Data 和 Reorg；
- 理解 Event Time、Arrival Time、Processing Time 的区别；
- 理解为什么 Watermark / Checkpoint 并不意味着历史数据永远不会变化；
- 理解 Lookback Window / Reprocessing Window 的基本作用；
- 判断 Late-arriving 与 Reorg 分别需要怎样的 ETL 思维；
- 理解 Blockchain ETL 为什么不能简单假设“历史数据一旦写入就永远固定”。
本课明确不展开：
- Ethereum 共识与 Finality 算法细节；
- Reorg common ancestor 的完整工程实现；
- 数据质量监控体系；
- 自动 Repair Framework；
- Kafka Event-time Watermark。
这些后面再讲。

---

## 一、先挑战我们前四课建立的模型
前面几课我们已经建立了一个相当完整的 Batch ETL：
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
例如：
```text
checkpoint = 2026-09-14
```
处理：
```text
2026-09-15
```
Load 成功以后：
```text
checkpoint = 2026-09-15
```
看起来很完整。
但这里其实隐含了一个很强的假设：
> 当 09-15 处理完成以后，属于 09-15 的 Source Data 不会再发生变化。
传统数据工程里，这个假设已经不一定成立。
Blockchain 里更不一定成立。
今天就是要拆掉这个假设。

---

## 二、第一类问题：Late-arriving Data
先不谈 Blockchain。
假设银行交易表里有一笔交易：
```text
transaction_time
=
2026-09-15 23:58
```
按照业务时间：
> 它属于 09-15。
但是上游系统因为网络问题，直到：
```text
2026-09-16 00:20
```
才把这条交易同步到数据仓库。
于是出现两个时间。
业务发生时间：
```text
Event Time
=
09-15 23:58
```
数据到达时间：
```text
Arrival Time
=
09-16 00:20
```
这就是：
> Late-arriving Data。

---

## 三、Late-arriving 的关键不是“时间晚”
Late-arriving 的严格含义不是：
```text
今天的数据晚上才到
```
而是：
> 数据到达 Pipeline 的时间，晚于我们按照其业务时间本来应该处理它的时间窗口。
假设 Daily ETL：
```text
00:05
```
开始处理前一天：
```text
09-15
```
但那条：
```text
23:58
```
的交易：
```text
00:20
```
才到。
于是：
```text
00:05 ETL
```
扫描：
```text
09-15 00:00
~
09-16 00:00
```
时，这条数据根本还不存在。
ETL 正常成功：
```text
checkpoint = 09-15
```
但是 Target：
```text
09-15
```
已经少了一条。
注意这里：
> Pipeline 没有报错。
这是 Late-arriving 最麻烦的地方之一。

---

## 四、三个“时间”必须分开
【Data Engineering 视角】
以后看到数据 Pipeline，最好区分三个时间。
第一：
```text
Event Time
```
事件实际上什么时候发生。
例如：
```text
block_time
transaction_time
```
第二：
```text
Arrival Time
```
你的数据平台什么时候看到这条数据。
第三：
```text
Processing Time
```
你的 ETL 什么时候处理它。
可能出现：
```text
Event Time
09-15 23:58

Arrival Time
09-16 00:20

Processing Time
09-16 01:00
```
这三个时间不是一个概念。

---

## 五、Blockchain 为什么也会 Late-arriving？
你可能会产生一个疑问：
> 区块链不是已经把交易写进 Block 了吗？怎么还会 Late-arriving？
这里必须加视角。
【Protocol 视角】
某笔 Transaction 已经进入 Block。
链知道它。
但是：
【Blockchain Data Engineer 视角】
你的数据平台并不等于 Blockchain。
你的数据流可能是：
```text
Ethereum
↓
Node / Provider
↓
Indexer
↓
fact_token_transfers
↓
ETL
↓
DWS
```
其中任意一层都可能延迟。
比如：
```text
Provider 延迟
Indexer Lag
RPC timeout
某段 Block 拉取失败
Decoder 暂时失败
Writer 堵塞
上游 Pipeline 暂停
```
于是：
```text
Block 里的 Event
```
已经存在，
但是：
```text
fact_token_transfers
```
晚一点才出现。
对于下游 Daily ETL 来说：
> 它就是 Late-arriving Data。

---

## 六、一个具体 Blockchain 例子
假设 Ethereum 上：
```text
block_time
=
2026-09-15 23:59:50
```
有一笔 USDC Transfer。
但你的 Indexer 当时：
```text
RPC timeout
```
这一小段 Block 没成功写入。
Daily ETL 在：
```text
09-16 00:10
```
开始：
```sql
WHERE block_time >= '2026-09-15 00:00'
  AND block_time <  '2026-09-16 00:00'
```
结果：
```text
这笔 Transfer 不存在
```
ETL 正常完成。
然后 Indexer：
```text
09-16 00:30
```
补回这段 Block。
现在：
```text
fact_token_transfers
```
里突然多了一条：
```text
block_time = 09-15 23:59:50
```
但：
```text
Daily ETL checkpoint
```
已经越过：
```text
09-15
```
这就是经典问题。

---

## 七、Checkpoint 为什么没有救我们？
因为 Checkpoint 表示：
> Pipeline 已经成功处理了当时可见的 Input Range。
它不代表：
> 这个 Input Range 未来永远不会再出现新数据。
这个区别很重要。
以前我们说：
```text
Checkpoint
=
durable recovery point
```
今天要补上一句话：
> Checkpoint 是 Processing State，不是对 Source Completeness 的数学证明。
所以：
```text
checkpoint = 09-15
```
只说明：
```text
09-15 Job
成功执行
```
不自动说明：
```text
所有属于 09-15 的数据
已经百分百到齐
```

---

## 八、这就是 Watermark 更微妙的地方
上一课之前我们讲过 Watermark：
```text
处理边界
```
如果简单定义：
```text
watermark = 09-16 00:00
```
你可能理解成：
> 09-16 以前全部结束。
但现实里可能只能说：
> 我们当前准备把 09-16 作为处理边界。
如果 Source 存在延迟：
```text
Late-arriving
```
这个边界就不能太激进。
所以工程里常见一个思想：
```text
Safe Watermark
```
而不是：
```text
Latest Possible Watermark
```

---

## 九、第一种简单办法：等待
例如：
```text
09-15
```
的数据不在：
```text
09-16 00:01
```
马上计算。
而等到：
```text
09-16 02:00
```
甚至：
```text
09-16 06:00
```
再算。
为什么？
为了给上游留一个：
```text
lateness buffer
```
也就是：
```text
等待迟到数据
```
这是一种非常常见的办法。
但是问题是：
> 你永远无法仅靠等待解决所有 Late-arriving。
因为数据可能晚：
```text
5 分钟
2 小时
1 天
```
甚至更久。

---

## 十、第二种重要办法：Lookback Window
这是今天要掌握的一个核心概念。
假设：
```text
checkpoint = 09-15
```
今天理论上只需要处理：
```text
09-16
```
但实际 ETL 不只读取：
```text
09-16
```
而是重新读取：
```text
09-14
09-15
09-16
```
也就是说：
```text
Lookback Window
=
最近 3 天
```
每次 ETL 都重新计算最近几天。

---

## 十一、为什么 Lookback 可以解决 Late-arriving？
假设：
```text
09-15
```
有一条迟到数据在：
```text
09-16
```
才进入 Source。
当天 ETL：
```text
reprocess
09-14 ~ 09-16
```
那么重新扫描 09-15 时：
```text
发现迟到记录
```
重新计算：
```text
09-15 DWS
```
于是历史结果得到修正。
所以：
```text
Lookback Window
```
本质上是在说：
> 不要认为最近的历史已经完全封闭。

---

## 十二、Lookback 和 Backfill 不完全一样
两者都会重跑历史。
但语义不完全相同。
Backfill：
```text
明确指定某段历史
重新处理
```
例如：
```text
2026-06-01 ~ 06-30
```
通常是人为或任务触发。
Lookback：
```text
正常 Incremental Job
每次自动顺便重算最近一小段历史
```
例如每天：
```text
today - 3 days
~
today
```
可以理解：
```text
Incremental
+
small rolling reprocessing window
```

---

## 十三、为什么上一课的 Idempotency 在这里突然非常重要？
假设每天都重算最近 3 天：
```text
09-14
09-15
09-16
```
那意味着：
```text
09-14
09-15
```
会被重复执行很多次。
如果 Load 不是 Idempotent：
```text
重复 INSERT
```
数据会不断膨胀。
而如果：
```text
Replace Partition
```
那么：
```text
重新计算 09-15
↓
替换 09-15
```
非常自然。
所以你现在可以看到课程设计为什么是：
```text
Lesson 4
Idempotency

↓

Lesson 5
Late-arriving
```
因为：
> 没有安全重跑能力，就很难可靠处理 Late-arriving Data。

---

## 十四、现在进入 Blockchain 特有的第二类问题：Reorg
Late-arriving 是：
> 数据本来存在，只是你的 Pipeline 晚看见。
Reorg 完全不同。
Reorg 是：
> 你之前看到的数据，当时看起来有效；后来 Blockchain 告诉你，它不再属于 canonical chain。
这不是“迟到”。
这是：
```text
历史事实被修正
```
至少从你的数据平台的 canonical 数据视角来看如此。

---

## 十五、一个最小 Reorg 例子
【Protocol 视角】
假设你最开始看到：
```text
Block 100
↓
Block 101A
↓
Block 102A
```
Indexer 已经处理：
```text
101A
102A
```
里面有：
```text
Transfer X
Transfer Y
```
然后链发生重组。
最终 canonical chain 变成：
```text
Block 100
↓
Block 101B
↓
Block 102B
```
现在：
```text
101A
102A
```
成为：
```text
orphan / non-canonical
```
所以：
```text
Transfer X
Transfer Y
```
可能已经不应该算进正式数据。

---

## 十六、Reorg 和 Late-arriving 的根本区别
这个区分必须非常清楚。
Late-arriving：
```text
昨天本来有 100 条

你第一次只看到 99 条

后来第 100 条来了
```
Source Truth：
```text
没有改变
```
只是：
```text
你的视野不完整
```
Reorg：
```text
第一次看到 100 条
```
后来发现：
```text
其中 3 条来自 non-canonical branch
```
甚至新 canonical branch 又来了另外：
```text
2 条
```
Source Truth：
```text
本身发生了变化
```
这是两类完全不同的问题。

---

## 十七、用银行系统类比一下
Late-arriving 很容易类比。
例如：
```text
某分行 09-15 的流水
```
因为批量文件迟到，
09-16 才传上来。
那么：
```text
09-15 报表
```
需要重算。
但 Reorg 在传统银行系统里就不那么常见。
它有点像：
> 昨天总行已经告诉你一批交易是正式有效的，今天又告诉你其中一批实际上被撤销，并用另一组记录替换。
所以 Blockchain 数据工程有一个特殊挑战：
> Source 本身存在短期不稳定性。

---

## 十八、为什么“Blockchain 是 immutable”仍然会有 Reorg？
这是一个特别容易混淆的问题。
我们通常说：
```text
Blockchain is immutable
```
这更多是：
> 已经达到足够共识稳定性的 canonical history 很难被修改。
但在链头附近：
```text
latest blocks
```
可能还存在短期分叉和重新选择 canonical branch。
所以必须加视角。
【链上数据平台视角】
不是所有你刚看到的 Block 都具有同样的稳定程度。
越靠近：
```text
chain tip
```
越需要考虑：
```text
Reorg risk
```
越往历史深处：
```text
稳定性通常越高
```
本课不展开 Ethereum 共识细节，只需要建立这个工程认识。

---

## 十九、因此“最新数据”和“稳定数据”不是完全同义
假设：
```text
latest block = 20,000,100
```
你的 Pipeline 可以立即处理：
```text
20,000,100
```
这样：
```text
latency 很低
```
但：
```text
reorg risk 较高
```
也可以只处理到：
```text
20,000,088
```
故意落后：
```text
12 blocks
```
这样：
```text
latency 增加
```
但：
```text
stability 提高
```
这里出现一个工程权衡：
```text
Freshness
vs
Stability
```

---

## 二十、这和 Watermark 又连起来了
假设 Chain Tip：
```text
20,000,100
```
但你的 Safe Watermark：
```text
20,000,088
```
意味着：
```text
20,000,089 ~ 20,000,100
```
虽然已经存在，
但暂时不进入稳定 ETL 范围。
所以：
```text
Source High Watermark
=
20,000,100
```
而：
```text
Safe Processing Watermark
=
20,000,088
```
这正是上一课为什么说：
> Watermark 不一定就是 Source 的最大值。

---

## 二十一、两种 Blockchain Pipeline 策略
简单来说可以有两种思路。
方案 A：
```text
Wait Before Processing
```
只处理相对稳定的 Block。
优点：
```text
Reorg correction 少
```
缺点：
```text
数据延迟更高
```
方案 B：
```text
Process Early
+
Correct Later
```
最新 Block 立即处理。
如果 Reorg：
```text
rollback / invalidate
+
replay
```
优点：
```text
低延迟
```
缺点：
```text
工程复杂
```

---

## 二十二、Indexer 和 ETL 对 Reorg 的责任不同
这是一个重要视角。
【Indexer 视角】
Indexer 离 Blockchain Source 最近。
它负责识别：
```text
Block Hash
Parent Hash
Canonical Chain
Reorg
```
发生 Reorg 时：
```text
Invalidate old branch
Replay new branch
```
这些你在 Module 7 已经实践过。
但：
【ETL 视角】
ETL 的问题是：
> 上游 Fact Table 被修正以后，我的 DWS / ADS 怎么跟着修正？
例如：
```text
fact_token_transfers
```
09-15 的数据发生变化。
那么：
```text
dws_wallet_token_daily_flow
```
的：
```text
09-15 partition
```
也已经过期。
所以需要：
```text
重新计算 09-15
```

---

## 二十三、这说明 Reorg 会向下游传播
完整数据流：
```text
Blockchain
↓
Indexer
↓
Normalized Fact
↓
DWS
↓
ADS
↓
Dashboard
```
Reorg 发生在最上游。
但影响可能一直传播：
```text
Block changed
↓
Transfer Fact changed
↓
Daily Flow changed
↓
Wallet Metric changed
↓
Dashboard changed
```
所以：
> Reorg 不是只有 Indexer 关心。
Blockchain Data Engineer 必须理解这种：
```text
downstream invalidation
```

---

## 二十四、但本课不要把 ETL 做成 Reorg Engine
这里要控制课程边界。
Module 9 你只需要掌握：
> ETL 必须接受“已经完成的历史 Processing Unit 可能需要重新计算”。
例如：
```text
09-15
```
已经完成。
后来上游通知：
```text
09-15 source changed
```
那么：
```text
recompute 09-15
```
即可。
至于：
```text
如何发现 Reorg
如何找 common ancestor
如何保存 canonical/orphan
如何自动传播 repair
```
这些不在本课展开。

---

## 二十五、Late-arriving 和 Reorg 最后都可能表现为“历史重算”
虽然原因不同：
```text
Late-arriving
=
之前少看到数据
```
Reorg：
```text
之前看到的数据后来失效 / 被替换
```
但对于一个 Daily Aggregate ETL，最终动作可能很相似：
```text
Recompute affected partition
```
例如：
```text
09-15 changed
↓
run_job(09-15, 09-16)
↓
Replace 09-15 partition
```
这就是为什么：
```text
Backfill
Idempotency
Processing Unit
```
前面这些概念全都开始汇合起来。

---

## 二十六、为什么不能简单“把 Checkpoint 倒退”？
假设正常 ETL：
```text
checkpoint = 09-16
```
发现：
```text
09-13
```
有迟到数据。
是不是直接：
```text
checkpoint = 09-13
```
然后重新往后跑？
通常没必要。
和之前 Backfill 一样：
```text
Normal Incremental State
```
应该保持：
```text
09-16
```
另起一个：
```text
repair / backfill range
=
09-13
```
重新处理受影响历史范围。
所以仍然遵守：
> Historical Repair 不污染正常 Incremental Checkpoint。

---

## 二十七、一个比较完整的运行模型
正常 Daily ETL：
```text
incremental checkpoint
=
09-15
```
今天：
```text
process 09-16
```
同时保留一个：
```text
lookback = 2 days
```
所以实际：
```text
09-14
09-15
09-16
```
都重新计算。
如果后来发现更早的：
```text
09-01
```
受 Reorg Repair / Decoder Fix / Late Data 影响，
则单独执行：
```text
backfill / repair
09-01
```
而不是：
```text
rollback incremental checkpoint
```

---

## 二十八、现在把五课连起来
到这里 Module 9 已经不是零散概念了。
```text
Lesson 1
ETL 是长期 Pipeline

↓

Lesson 2
Input Range
Full / Incremental / Backfill

↓

Lesson 3
Processing State
Watermark / Cursor / Checkpoint

↓

Lesson 4
Safe Replay
Idempotent Load

↓

Lesson 5
为什么历史需要重新打开？
Late-arriving / Reorg
```
你现在应该逐渐形成一个生产级思维：
> 数据管道不是“处理一次就结束”，而是在不断接收新数据、重跑旧数据、修正历史，同时保持最终结果一致。

---

## 二十九、本课最核心的区别
这两个概念最好直接记成：
```text
Late-arriving Data
=
Truth 没变
但 Pipeline 晚看到
```
```text
Reorg
=
Pipeline 当时看到的 Truth
后来不再是 canonical Truth
```
再进一步：
```text
Late-arriving
→ 补进来

Reorg
→ 删除 / 失效旧数据
  + 接受新 canonical 数据
```
这是本课最关键的概念区分。

---

## 三十、从 Blockchain Data Engineer 视角总结
一个可靠 Blockchain ETL 不能简单认为：
```text
checkpoint 过去了
=
历史永远关闭
```
更准确的模型是：
```text
Old History
大体稳定

Recent History
可能迟到
可能变化

Chain Tip
最不稳定
```
所以系统需要同时具备：
```text
Incremental Processing
+
Replay
+
Idempotency
+
Historical Reprocessing
```
这四个能力组合起来，才是长期运行的数据管道。

---

## 第 5 课理解检查
### 问题一
Daily ETL 在 09-16 00:10 已经成功处理完 09-15 并推进 Checkpoint。但是 09-16 00:30 Indexer 补写了一笔 block_time = 09-15 23:59:50 的 Transfer。
1. 这是 Late-arriving Data 还是 Reorg？
2. 为什么 Checkpoint 已经成功推进，仍然可能发生这种情况？
3. 如果 Daily ETL 有 3 天 Lookback Window，它如何修复这个问题？
### 问题二
Indexer 原来处理 Block 101A、102A，后来发现 canonical chain 实际变成 Block 101B、102B。为什么这不是 Late-arriving，而是 Reorg？重点从 Source Truth 有没有变化来回答。
### 问题三
正常 Daily ETL checkpoint = 09-16，后来发现 09-10 的数据因为 Reorg / Late-arriving 需要重新计算。会不会把正常 Incremental Checkpoint 改回 09-10？如果不会，会怎么处理？

---

## 用户回答
问题一，1：这是 latest arriving date

为什么 Checkpoint 已经成功推进，仍然可能发生这种情况？

2：因为这是上游数据延迟，和 Checkpoint 其实没有什么关系。上游数据发生延迟的情况很多。

3. Daily ETL 有 3 天 Lookback Window。那么这个数据延迟在一天范围之内，这也在这个 Lookback Window 的处理范围之内，那么它会按正常流程修复这个问题

二，因为这是链上的事实发生了变化，而不是数据缺失

问题3：在 Index 里面遇到 Reorg 的情况，是把 Checkpoint 修改，然后 ETL 这里同样修改 Checkpoint 也是可行的吧

---

## 教师判断与校准
问题一和问题二正确。“latest arriving date”应为 **Late-arriving Data**，不影响概念判断。
问题三需要校准：技术上倒退 ETL Checkpoint 后重跑可能得到正确结果，但生产 ETL 中通常不这样设计。
【Indexer 视角】Reorg 会让 Indexer 原来的 canonical progression 本身失效，因此需要 rollback checkpoint 到 common ancestor 附近，再 invalidate old branch、replay new branch，并重新推进 checkpoint。
【ETL 视角】如果 Daily ETL 已经正常运行到 09-16，后来只是 09-10 的上游 Fact 被修正，那么“09-16 已经完成”这个 processing state 仍然成立；失效的是 09-10 的 Target Partition，而不是正常增量进度。因此通常保持 incremental checkpoint = 09-16，另外执行 repair/backfill 09-10。
核心区别：Indexer Reorg 修改了“走过的 canonical chain path”；ETL Historical Repair 修改的是“过去算出的某个 Target 结果”。

---

## 用户追问
我对修改的数据范围有点疑问：发生 reorg 的时候，Indexer 回退找到共同的父节点，然后把之后的 blocks 全部重新解析。

按我的理解，把 block 的修改范围转换成时间的话，应该是从某个时间节点之后的所有数据都发生了变化？

按照这个想法，我觉得 ETL 处理这块，它的 checkpoint 也需要从修改的时间节点开始，把之后每天的数据全部重新跑一遍 ETL

---

## 进一步校准：Source Changed Range ≠ Downstream Affected Range
这个推理有一半正确。Indexer 从 common ancestor 之后到当前 chain tip 的 canonical block sequence 确实需要重新解析，但“Indexer 需要重放的 Block Range”不等于“所有下游 ETL 从那个时间点之后都必须全部重跑”。
真正判断标准是：**下游数据模型对时间是否存在传递依赖。**
如果 Reorg 改变的 Block Range 映射到 09-15，而 Target 是 `dws_wallet_token_daily_flow` 这种 Independent Partition：
```text
09-15 = SUM(source where date=09-15)
09-16 = SUM(source where date=09-16)
```
那么 09-15 改变只需要 recompute 09-15，09-16 以后并不依赖 09-15 的计算结果。
如果 Target 是 `daily_wallet_balance`、running total、cumulative volume、position snapshot 等 Cumulative / State-dependent 模型，例如：
```text
09-16 balance
=
09-15 ending_balance
+ 09-16 net_flow
```
那么 09-15 改变会向后传播，Repair Range 可能确实需要从 09-15 一直重算到 current。
因此应按以下链路确定修复范围：
```text
Reorg Changed Block Range
        ↓
Map to affected source rows
        ↓
Map to affected target partitions
        ↓
Follow downstream dependencies
        ↓
Determine repair range
```
即使 Repair Range 是 09-15 ~ current，也仍然不等于必须 rollback 正常 incremental checkpoint。更清晰的状态设计仍是：
```text
normal checkpoint = current
repair range = 09-15 ~ current
```
“需要重跑后续所有日期”与“必须把正常 checkpoint 改回去”是两个不同问题。

---

## 结课判定
Module 9 第 5 课理解检查通过，必要校准完成，本课正式结束。
已掌握：Late-arriving Data 与 Reorg 的本质区别；Checkpoint 不等于 Source Completeness；Lookback Window 用于自动重新打开近期历史；Historical Repair 与正常 Incremental State 应隔离；Reorg 的下游修复范围应由 Source Changed Range、Target Partition 与 Dependency / Lineage 共同决定，而不是机械地从某时间点后全部重跑。