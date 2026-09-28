# Module 12 第 2 课｜Completeness：Missing Block、Missing Log 与 Provider Gap

## Lesson Contract

所属 Module：Module 12 — 数据质量

本课核心问题：

> How do we know that the blockchain data we ingested is complete?

学完以后，你应该能够解释并设计：

- 什么是 Blockchain Data Completeness。
- 为什么“Block 没缺”不代表数据完整。
- Missing Block、Missing Transaction、Missing Receipt、Missing Log 分别意味着什么。
- 为什么 Completeness 必须和 Data Grain 一起判断。
- Provider Gap 是什么，以及它为什么比程序报错更危险。
- 为什么单一 Provider 成功返回数据，仍然不能证明数据完整。
- 如何检测连续 Block Range 中的 Missing Block。
- 如何从 Block / Receipt / Log 等不同层次交叉验证 Completeness。
- 为什么补历史数据通常应使用 Backfill / Replay，而不是直接修改实时 Checkpoint。
- 如何设计一个最小 Completeness Detection & Repair Flow。

本课暂不展开：
- Duplicate Detection；
- Decoder Accuracy；
- Aggregate Reconciliation 的完整方法；
- Reorg 的完整修复流程。

这些分别在后续课程展开。

---

## 一、Completeness 到底在问什么？

上一课我们把 Completeness 简化成：

> Is anything missing?

但工程上这个问题还不够精确。

因为首先要问：

> What exactly is supposed to exist?

假设一个 Pipeline 处理：

```text
Block 100
Block 101
Block 102
Block 103
```

数据库里只有：

```text
100
101
103
```

这当然是 Missing Block。

但另一种情况：

```text
Block 102 exists
```

Block 也成功写进数据库。

可是 Block 102 中原本有：

```text
10 transactions
```

你的数据库只有：

```text
9 transactions
```

那么 Block Level 看起来完整，但 Transaction Level 已经不完整。

所以第一条原则是：

> Completeness is always grain-dependent.

也就是：

> 完整性必须绑定到具体的数据粒度来判断。

---

## 二、[Data Engineer 视角] 先明确你的 Grain

前面 Module 8 我们已经反复讲过 Grain。

现在 Data Quality 会再次依赖它。

例如：

```text
blocks
grain = one row per block
```

```text
transactions
grain = one row per transaction
```

```text
receipts
grain = one row per transaction receipt
```

```text
logs
grain = one row per log
```

```text
token_transfers
grain = one row per decoded transfer event
```

于是你不能只问：

> 数据完整吗？

而应该问：

```text
Block complete?
Transaction complete?
Receipt complete?
Log complete?
Decoded Event complete?
```

这些问题的答案可能完全不同。

---

## 三、Missing Block：最容易发现的 Completeness Failure

假设你的处理范围是：

```text
start_block = 20,000,000
end_block   = 20,000,999
```

理论上：

```text
expected blocks = 1000
```

数据库里只有：

```text
999 blocks
```

如果 canonical block number 应连续，就可以进一步检查：

```text
20,000,000
20,000,001
20,000,002
...
20,000,526
20,000,528
...
```

这时：

```text
20,000,527 missing
```

这是典型的：

> Missing Range / Missing Block Detection.

最简单的检查思路是：

```text
Expected Block Range
        ↓
Compare
        ↓
Actual Block Numbers
        ↓
Find Gaps
```

这类错误通常比较容易发现，因为 Block Number 天然提供了连续序列。

---

## 四、一个基础 Missing Block SQL

假设 PostgreSQL 中有：

```sql
CREATE TABLE blocks (
    chain_id      BIGINT,
    block_number  BIGINT,
    block_hash    TEXT,
    block_time    TIMESTAMP,
    PRIMARY KEY (chain_id, block_number)
);
```

如果数据库支持 `generate_series`，可以做：

```sql
WITH expected_blocks AS (
    SELECT generate_series(20000000, 20000999) AS block_number
)
SELECT e.block_number
FROM expected_blocks e
LEFT JOIN blocks b
    ON b.chain_id = 1
   AND b.block_number = e.block_number
WHERE b.block_number IS NULL;
```

如果返回：

```text
20000527
```

说明这一高度缺失。

这里的逻辑很直接：

```text
Expected Set
-
Actual Set
=
Missing Set
```

---

## 五、但 Row Count 仍然不够

假设：

```text
expected blocks = 1000
actual blocks   = 1000
```

是不是一定完整？

不一定。

例如错误数据是：

```text
20000000
...
20000526
20000528
...
20000999
20000999
```

也就是：

```text
一个 Block 缺失
+
另一个 Block 重复
```

最终：

```text
row count = 1000
```

Row Count 完全一致。

但数据仍然错。

因此：

> Row Count can be necessary, but it is often not sufficient.

这也是为什么后面第 5 课我们会专门学习 Reconciliation。

---

## 六、Missing Transaction：Block 在，不代表里面的数据都在

Ethereum Block 中包含 Transaction 列表。

从逻辑结构看：

```text
Block
└── Transactions
```

假设 RPC 返回：

```text
Block 100
transaction_count = 120
```

Indexer 最终写入：

```text
transactions table
block_number = 100
count = 119
```

那么：

```text
Block exists
```

但是：

```text
Transaction Completeness failed
```

这里就可以做一个局部验证：

```text
source tx count
vs
sink tx count
```

例如：

```text
RPC Block tx count = 120
Database tx count  = 119
```

那么问题非常明确：

```text
1 transaction missing
```

---

## 七、Missing Receipt：Transaction 在，也不代表 Receipt 完整

Ethereum 数据链路里：

```text
Transaction
↓
Execution
↓
Receipt
↓
Logs
```

Receipt 非常重要，因为它包含：

```text
status
gasUsed
logs
contractAddress
...
```

假设你成功保存：

```text
1,000 transactions
```

但只保存：

```text
998 receipts
```

这时 Transaction Completeness 没问题，但 Receipt Completeness 已经失败。

一个很自然的 Invariant 是：

```text
For finalized / fully processed transactions:

Transaction Count
=
Receipt Count
```

更准确地说，应该基于同样的：

```text
chain
block range
canonical boundary
processing status
```

来比较。

---

## 八、Missing Log：最值得警惕的一类

Missing Log 比 Missing Block 难发现很多。

因为：

```text
Block exists
Transaction exists
Receipt exists
```

都可能正常。

但是 Receipt 中本来有：

```text
5 logs
```

你最终只保存：

```text
4 logs
```

那么：

```text
Block-level check = PASS
Transaction-level check = PASS
Receipt-level check = PASS
Log-level check = FAIL
```

这说明：

> High-level completeness does not imply lower-level completeness.

换句话说：

```text
Block complete
≠
Logs complete
```

---

## 九、为什么 Missing Log 对 Blockchain Data Engineer 特别危险？

因为很多业务事实不是直接来自 Transaction，而是来自 Log。

例如：

```text
ERC-20 Transfer
DEX Swap
Liquidity Add / Remove
NFT Transfer
Lending events
```

很多都是：

```text
Raw Log
↓
Decoder
↓
Business Fact
```

如果漏掉一个 Log：

```text
Missing Raw Log
↓
Missing Decoded Event
↓
Missing Fact
↓
Wrong DWS
↓
Wrong Dashboard
```

于是最上层可能表现为：

```text
daily volume 少了 2%
```

但真正根因是：

```text
Provider / ingestion 漏了部分 Logs
```

所以 Completeness Error 会沿整个数据血缘传播。

---

## 十、Provider Gap 是什么？

所谓 Provider Gap，可以简单理解为：

> Provider 在某个时间段、Block Range 或某类请求中，没有返回理论上应存在的完整数据。

例如：

```text
eth_getLogs
fromBlock = 15000000
toBlock   = 15000999
```

Provider 返回：

```text
HTTP 200
JSON-RPC success
result = [...]
```

程序看起来完全正常。

但其中：

```text
block 15000321 的两个 Logs 没返回
```

这就是最危险的一类问题之一：

```text
Request succeeded
but
Result incomplete
```

所以：

> Successful RPC response does not prove complete RPC data.

---

## 十一、[RPC 视角] 为什么 Provider 可能出现 Gap？

本课不深入 Provider 内部实现，但工程上要知道可能来源很多。

例如：

```text
Backend node temporary inconsistency
Index lag
Cache inconsistency
Range query issue
Internal timeout / partial result
Provider bug
Backend failover inconsistency
Historical indexing gap
```

注意：

这不意味着任何一次缺失都一定是 Provider 的责任。

也可能是：

```text
Indexer bug
pagination bug
retry logic bug
range split bug
filter bug
```

所以真正工程化的说法应该是：

> We detected an ingestion completeness gap.

然后再 Diagnose：

```text
Source really missing?
Provider incomplete?
Indexer dropped it?
Consumer lost it?
Sink rejected it?
```

先描述事实，再定位根因。

---

## 十二、为什么不能只相信一个 Provider？

假设 Provider A 返回：

```text
Block 15,003
Logs = 18
```

你怎么知道 18 就一定完整？

如果没有其他参考，你其实不知道。

因此一些关键场景会使用：

```text
Provider A
vs
Provider B
```

或者：

```text
Provider
vs
Self-hosted Node
```

做交叉验证。

例如：

```text
Provider A: 18 logs
Provider B: 21 logs
```

这时至少可以判断：

```text
There is a consistency discrepancy.
```

然后继续检查：

```text
same block hash?
same canonical chain?
same filter?
same query range?
same finality point?
```

如果这些条件相同，而结果仍然不同，就有很强的证据表明某一侧数据不完整。

---

## 十三、[视角前提] Completeness 与 Consistency 不完全一样

这两个概念容易混。

Completeness 问：

> 应有的数据是否全部存在？

Consistency 问：

> 不同表示之间是否一致？

例如：

```text
Provider A = 20 logs
Provider B = 18 logs
```

你首先发现的是：

```text
Consistency Problem
```

但它可能帮助你发现：

```text
Provider B Completeness Problem
```

所以：

```text
Consistency check
→ can be evidence
→ for detecting Completeness failure
```

但二者不是同一个概念。

---

## 十四、完整性检查必须沿数据层级做

一个比较实用的思路是：

```text
Block
↓
Transaction
↓
Receipt
↓
Log
↓
Decoded Event
↓
Fact
```

每一层都可以建立自己的 completeness check。

例如：

```text
Block Layer
→ block range continuity

Transaction Layer
→ tx count per block

Receipt Layer
→ one receipt per processed tx

Log Layer
→ receipt log count vs stored log count

Decoded Event Layer
→ expected decoder coverage / known protocol logs

Fact Layer
→ source events vs derived facts
```

这就是：

> Layered Completeness Validation.

---

## 十五、银行系统类比

假设银行日终数据链路：

```text
核心交易系统
↓
交易接口文件
↓
ODS
↓
Fact
↓
日报
```

文件每天都到了：

```text
file received = YES
```

不代表：

```text
all transactions received
```

再比如：

```text
总文件 100 个
全部收到
```

但其中一个文件内部原本应该有：

```text
50,000 rows
```

实际只有：

```text
48,000 rows
```

那么：

```text
File-level Completeness = PASS
Record-level Completeness = FAIL
```

Blockchain 里完全一样：

```text
Block-level PASS
Log-level FAIL
```

这就是为什么 Grain 非常重要。

---

## 十六、Detection 之后不要立刻“手工补一行”

假设检测出：

```text
Block 20,000,527 missing
```

最差的修复思路之一是：

```text
手工 INSERT 一条 block
```

因为真实链路通常不只有 blocks 表。

还可能有：

```text
transactions
receipts
logs
token_transfers
swaps
DWS
ADS
```

因此正确修复通常应该重新执行同一处理逻辑：

```text
Missing Range
↓
Backfill / Replay
↓
process_block(...)
↓
Rebuild downstream facts
↓
Validate
```

这和你前面 Module 7 / Module 9 的设计原则一致：

> Reuse the same processing logic for realtime, backfill and replay whenever possible.

---

## 十七、为什么不能直接回退 Realtime Checkpoint？

假设：

```text
Realtime Checkpoint = 20,100,000
```

后来发现：

```text
20,000,527 missing
```

如果你直接把实时 Checkpoint 改回：

```text
20,000,526
```

那么实时 Pipeline 会重新处理：

```text
20,000,527
→
20,100,000
```

这会造成：

```text
大量无必要 Replay
可能冲击实时任务
可能产生重复写
可能扩大故障范围
```

更合理的方式通常是：

```text
Realtime Pipeline
checkpoint = 20,100,000
继续处理最新数据

Backfill Pipeline
range = [20,000,527, 20,000,527]
独立修复历史缺口
```

如果缺口更大：

```text
range = [20,000,527, 20,000,650]
```

也是同样逻辑。

---

## 十八、Detection 和 Repair 的状态必须分开

检测出缺口以后，可以维护类似：

```text
quality_issue_id
chain_id
issue_type
start_block
end_block
detected_at
status
repair_job_id
verified_at
```

例如：

```text
issue_type = MISSING_BLOCK
start_block = 20000527
end_block = 20000527
status = DETECTED
```

然后：

```text
DETECTED
↓
REPAIRING
↓
REPAIRED
↓
VERIFIED
```

为什么不能直接：

```text
DETECTED
↓
DONE
```

因为：

> Repair execution succeeded ≠ Repair correctness verified.

这和上一课：

> Pipeline Success ≠ Data Correctness

是同一条思想。

---

## 十九、一个最小 Completeness Pipeline

可以先建立这样的结构：

```text
Blockchain / RPC
↓
Indexer
↓
Raw Tables
↓
Completeness Checks
├─ Block Range Check
├─ Tx Count Check
├─ Receipt Coverage Check
└─ Log Coverage Check
↓
PASS?
├─ Yes → mark range validated
└─ No
    ↓
    Record Quality Issue
    ↓
    Backfill / Replay
    ↓
    Re-run Checks
    ↓
    Verified
```

这就是最基础的：

> Completeness Detection & Repair Loop.

---

## 二十、Checkpoint 和 Quality Boundary

这里需要把上一课再往前推进一步。

假设实时 Pipeline 已处理：

```text
1000 → 1099
```

但发现：

```text
block 1057 missing
```

如果这个 Missing Block 是当前批次的 Validation Failure，那么：

```text
checkpoint ≠ 1099
```

因为这一范围不能声明完成。

但是如果：

```text
checkpoint 已经在几天前推进到 1099
```

今天通过独立质量扫描才发现 1057 当时漏了，那么你面对的是：

> Historical Data Quality Incident.

此时一般不应该简单把实时 Checkpoint 回退。

而应该：

```text
Create repair range
↓
Backfill / Replay
↓
Reconcile
↓
Verify
```

所以要区分：

```text
Pre-checkpoint validation failure
```

和：

```text
Post-checkpoint historical issue
```

这两个场景的修复策略不同。

---

## 二十一、Completeness 的证据应该是什么？

一个范围要被声明为完整，最好有可以审计的证据。

例如：

```text
Block range:
20,000,000 → 20,000,999

Expected blocks:
1000

Actual canonical blocks:
1000

Missing block count:
0

Transaction source count:
185,420

Stored transaction count:
185,420

Receipt coverage:
100%

Missing log issues:
0

Validation status:
PASS
```

于是你可以更严谨地说：

> Completeness checks passed for block range 20,000,000–20,000,999 under the defined validation rules.

注意：

不是说：

> 数据绝对永远不会有任何问题。

而是说：

> 在当前定义的检查规则下，这个范围通过了 Completeness Validation。

工程语言应该尽量精确。

---

## 二十二、CTO / Architecture 视角

真正的数据平台不能依赖：

```text
用户发现 Dashboard 数字不对
↓
工程师查日志
↓
发现三个月前漏数据
```

更成熟的架构应该主动检测：

```text
Unexpected block gap
Unexpected tx count mismatch
Receipt coverage drop
Log count anomaly
Provider discrepancy
```

并且知道：

```text
Which range is affected?
Which tables are affected?
Which downstream products are affected?
Can we replay safely?
Has repair been verified?
```

所以 Completeness 不只是：

```text
COUNT(*)
```

而是：

> Detectable + Traceable + Repairable Completeness.

---

## 二十三、本课核心心智模型

先记住这条链：

```text
Completeness
↓
Define expected data
↓
Define grain
↓
Compare expected vs actual
↓
Detect gaps
↓
Locate affected range
↓
Backfill / Replay
↓
Re-validate
```

最重要的是前两步：

```text
What should exist?
At what grain?
```

如果这两个问题没有定义清楚，后面的“数据有没有缺”其实无从判断。

---

## 本课核心结论

> Completeness is grain-dependent. A complete block table does not prove complete transactions, receipts, logs, or decoded facts.

> Missing Block is easy to detect because block numbers form a natural sequence; Missing Log can be much harder because higher-level records may still look complete.

> A successful RPC response does not prove that the returned dataset is complete.

> Provider Gap is an ingestion completeness problem that may only become visible through range checks, cross-layer validation, or cross-provider comparison.

> Row Count alone is insufficient because missing and duplicate records can cancel each other out.

> Historical completeness issues should normally be repaired with targeted Backfill / Replay rather than by rewinding the realtime checkpoint.

> Repair is not complete until the repaired range has been validated again.

---

## 理解检查

### 问题一

你有下面的数据：

```text
Block 18,000,000
RPC 显示：
transaction_count = 125
```

数据库里：

```text
blocks 表：有这个 block
transactions 表：只有 124 条
```

请回答：

1. Block Completeness 是否通过？
2. Transaction Completeness 是否通过？
3. 为什么不能只检查 blocks 表？

### 问题二

某个 Provider 执行：

```text
eth_getLogs
block 19,000,000 → 19,000,999
```

请求：

```text
HTTP 200
JSON-RPC success
```

但后来使用另一个 Provider 检查，发现 block `19,000,527` 少了 3 个 Logs。

请回答：

1. 这是不是 Provider / ingestion 层的 Completeness 问题？
2. 为什么 `RPC success` 不能证明 Completeness？
3. 第二个 Provider 在这里主要起什么作用？

### 问题三

Realtime Pipeline 当前：

```text
checkpoint = 21,000,000
```

质量扫描发现历史：

```text
block 20,500,123
```

整块缺失。

你会选择：

A.

```text
把 realtime checkpoint 改回 20,500,122
重新跑到 21,000,000
```

还是 B.

```text
保留 realtime checkpoint
建立独立 backfill range
20,500,123 → 20,500,123
重新执行 process_block
然后重新验证
```

请选择并说明原因。

## 用户回答（问题一）

问题 1：
1.Block completeness 是否通过？
通过。
2：Transaction completeness 是否通过？
不通过。
3：为什么不能只检查 Blocks 表？
因为不同的层级需要分别检查。Block 作为最高层级的 grain，它检查通过了，不能保证更细粒度的 grain 检查也能通过，所以不能只检查 Block 表。

## 老师判断 / 校准（问题一）

通过。

- Block Completeness 判断正确：blocks 表中存在 Block 18,000,000，因此在 block grain 上这一项通过。
- Transaction Completeness 判断正确：RPC 显示该 Block 有 125 个 transactions，而数据库只有 124 条，因此 transaction grain 上存在缺失。
- 第三点解释正确，而且抓到了本课最关键原则：**Completeness is grain-dependent.** 更高层级的数据完整，只能证明该层级本身通过，不能推出更细粒度的 Transaction / Receipt / Log / Decoded Event 也完整。

这一题可以概括为：

> High-level completeness does not imply lower-level completeness.

## 用户回答（问题二）

问题二：
1，这是不是 provide ingestion 层的 completeness 问题？是的，因为通过对比两个不同 provider 提供的数据，它们并不一致，说明其中一个 provider 的 completeness 有问题。
2，为什么 RPC success 不能证明 completeness？

因为 RPC success 只能保证这个请求访问的过程是成功的，但所请求的数据是否完整，这个不能保证。
3. 第二个 provider 在这里起对比、验证、校验的作用

## 老师判断 / 校准（问题二）

通过。

- 第一问方向正确：这是 Provider / ingestion 层需要关注的 Completeness 问题。但要更精确地区分“发现”与“归因”：两个 Provider 结果不一致时，首先能直接证明的是 **Consistency discrepancy**。只有进一步确认双方使用的是同一 block hash、同一 canonical chain、同一 filter、同一 query range、同一 finality point 等条件后，才能更有把握地判断某一侧存在 Completeness Gap。
- 第二问正确：`RPC success` 只能说明请求/响应流程在协议和执行层成功完成，并不能证明返回结果在数据语义上是完整的。也就是：

> Request success ≠ Result completeness.

- 第三问正确：第二个 Provider 主要作为独立参考源，用于 cross-provider comparison、validation / verification，帮助发现单一 Provider 可能存在的 silent gap。

本题最重要的校准是：

> Consistency discrepancy is evidence; after aligning comparison conditions, it can help identify a Completeness failure.

## 用户回答（问题三）

问题三，我会选择方案 B。

因为在这个情况下，有问题批次的数据已经处理完成，Checkpoint 已经推进到更新的高度，所以现在发现历史 Block 有缺失，选用方案 B 更好。方案 B 是保留 real-time checkpoint，建立独立的 backfill range

## 老师判断 / 校准（问题三）

通过。

选择方案 B 正确。

这里的关键判断是：这个 Missing Block 不是“当前批次在推进 Checkpoint 前刚刚发现”的 Validation Failure，而是 **Checkpoint 已经推进以后才发现的 Historical Data Quality Incident**。

因此不应该为了修复一个历史缺口，把 Realtime Checkpoint 从 21,000,000 回退到 20,500,122。否则会让实时 Pipeline 重新处理一个很大的已完成区间，带来不必要的 Replay、重复写风险、资源压力，以及对实时数据链路的干扰。

更合理的修复方式是：

```text
Keep realtime checkpoint
↓
Create isolated backfill range
20,500,123 → 20,500,123
↓
Reuse process_block(...)
↓
Repair downstream data
↓
Re-run completeness validation
↓
Mark issue VERIFIED
```

这和前面 Module 7 / Module 9 的原则一致：

> Realtime progress and historical repair should use independent processing state.

同时要保留最后一步：

> Repair execution succeeded ≠ Repair correctness verified.

Backfill 跑完以后仍然必须重新执行 Completeness Validation，确认这个历史缺口已经真正修复。

## 结课判定

Module 12 第 2 课理解检查全部通过，正式完成。

已经能够：
- 理解 Completeness 是 grain-dependent 的，高层粒度完整不代表更细粒度完整。
- 区分 Block / Transaction / Receipt / Log 等不同层级的 Completeness。
- 理解 RPC success 只证明请求成功，不证明返回数据完整。
- 使用 Cross-provider Comparison 发现 Consistency discrepancy，并将其作为进一步定位 Completeness Gap 的证据。
- 区分 Pre-checkpoint Validation Failure 与 Post-checkpoint Historical Data Quality Incident。
- 对历史缺口使用独立 Backfill / Replay，而不是直接回退 Realtime Checkpoint。
- 理解历史修复完成后必须重新 Validation，才能把问题状态推进到 VERIFIED。