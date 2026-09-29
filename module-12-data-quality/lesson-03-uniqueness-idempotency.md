# Module 12 第 3 课｜Uniqueness & Idempotency：Duplicate Delivery 与重复事实

## Lesson Contract

所属 Module：Module 12 — 数据质量

本课核心问题：

> How do we ensure that the same blockchain fact is not stored more than once?

学完以后，你应该能够解释并设计：

- 什么是 Uniqueness。
- 为什么 Duplicate Delivery 不一定是系统异常。
- Kafka / Retry / Replay 为什么天然可能产生重复消息。
- 为什么 At-least-once delivery 必须配合 Idempotent Sink。
- 如何区分 Message Duplicate 和 Business Fact Duplicate。
- 为什么 Unique Key 必须从数据 Grain 推导，而不能随便选字段。
- 为什么 `tx_hash` 往往不能单独唯一标识一条 Event / Log。
- 如何为 Blockchain Fact Table 设计幂等键。
- 为什么 Backfill / Replay / Reorg Correction 都必须复用同一套幂等约束。
- 如何检测历史 Duplicate Facts，并安全修复。

本课暂不展开：
- Decoder Accuracy；
- Aggregate Reconciliation；
- Reorg 的完整 canonical rollback/replay；
- Exactly-once transaction protocol 的内部实现。

---

## 一、Uniqueness 到底在问什么？

上一课的 Completeness 问：

> Is anything missing?

这一课的 Uniqueness 问：

> Is the same fact stored more than once?

假设真实链上只发生一次：

```text
Alice
↓
Transfer 100 USDC
↓
Bob
```

但数据库里出现两行：

```text
Transfer A
Transfer A
```

这就是：

> Duplicate Fact

也就是 Uniqueness Failure。

---

## 二、[Data Quality 视角] 重复不是“多了一行”这么简单

假设：

```text
fact_token_transfer
```

本来应该有：

```text
1 row
```

结果出现：

```text
2 rows
```

如果下游做：

```sql
SUM(amount)
```

真实金额：

```text
100 USDC
```

最终统计：

```text
200 USDC
```

于是：

```text
Duplicate Fact
↓
Wrong Aggregate
↓
Wrong DWS
↓
Wrong Dashboard
```

所以重复问题最终可能表现成 Accuracy Problem。

但它的直接质量维度是：

> Uniqueness.

这和上一课类似：

```text
Symptom
≠
Root quality dimension
```

---

## 三、为什么 Blockchain Pipeline 特别容易产生 Duplicate？

因为真实数据系统通常不是：

```text
Read once
↓
Write once
↓
Never retry
```

而是：

```text
Read
↓
Process
↓
Possible failure
↓
Retry / Replay
```

特别是 Streaming 系统中，重复是非常正常的工程现象。

例如 Kafka 常见语义：

```text
At-least-once delivery
```

意思不是：

> 每条消息只会到一次。

而是：

> 每条消息至少会到一次，但可能重复到达。

---

## 四、最典型的 Duplicate Delivery 场景

假设 Consumer 处理 Kafka Message：

```text
offset = 105
```

正常流程：

```text
Read message
↓
Transform
↓
INSERT database
↓
Commit offset
```

但如果发生：

```text
INSERT database = success
↓
Consumer crash
↓
Offset 105 not committed
```

Consumer 重启以后：

```text
Kafka
↓
read offset 105 again
```

于是：

```text
same message
↓
processed again
```

如果数据库直接：

```sql
INSERT
```

那么最终：

```text
same business fact
stored twice
```

这不是 Kafka Bug。

这是 At-least-once 的正常结果。

---

## 五、[Kafka 视角] Offset 和业务事实不是一回事

这里有一个很重要的视角区分。

Kafka 关心：

```text
topic
partition
offset
```

例如：

```text
topic = wallet_activity
partition = 3
offset = 105
```

这唯一标识的是：

> Kafka 中的一条 Message Position.

但 Blockchain Data Engineer 最终关心的是：

> 这条消息代表哪个链上事实？

例如：

```text
chain_id = 1
tx_hash = 0xabc...
log_index = 7
```

这才是在链上数据模型中更接近业务事实身份的东西。

所以：

> Message Identity ≠ Business Fact Identity.

---

## 六、为什么不能直接拿 Kafka Offset 当数据库 Unique Key？

假设同一个链上事实因为 Replay 被重新生产。

第一次：

```text
partition = 3
offset = 105
```

Replay 后可能变成：

```text
partition = 3
offset = 900105
```

Kafka Message Identity 不一样。

但业务事实仍然是：

```text
same chain
same tx
same log
```

如果数据库 Unique Key 是：

```text
partition + offset
```

那么两条消息都会被认为不同。

最终：

```text
same blockchain fact
↓
stored twice
```

所以 Sink 的幂等键必须尽量基于：

> Stable Business / Source Identity.

而不是传输层身份。

---

## 七、从 Grain 推导 Unique Key

前面 Module 8 已经学过：

> Unique Key 应该从 Grain 推导。

例如：

```text
blocks
grain = one canonical block record
```

一种基础 key 可以是：

```text
chain_id + block_hash
```

或者在只保存 canonical snapshot 的特定模型中：

```text
chain_id + block_number
```

但注意 Reorg 场景下：

```text
block_number
```

并不是永久稳定的链上事实身份。

因此是否能作为唯一键，要看你的数据模型。

---

## 八、Transaction 的 Unique Key

Ethereum Transaction 本身通常可用：

```text
chain_id + tx_hash
```

表示。

因为：

```text
tx_hash
```

在同一条链上用于标识该 Transaction。

所以：

```text
grain = one transaction
```

可以设计：

```text
UNIQUE(chain_id, tx_hash)
```

---

## 九、Log 为什么不能只用 tx_hash？

一笔 Ethereum Transaction 可以产生多个 Logs。

例如：

```text
Transaction 0xabc
├─ Log 0
├─ Log 1
├─ Log 2
└─ Log 3
```

所以：

```text
chain_id + tx_hash
```

只能确定：

> 哪一笔 Transaction

不能确定：

> 这笔 Transaction 中哪一个 Log

因此 Log 的常见 Source Identity 是：

```text
chain_id
+
tx_hash
+
log_index
```

也就是：

> one log within one transaction.

---

## 十、ERC-20 Transfer Fact 的 Unique Key

假设：

```text
fact_token_transfer
grain = one decoded ERC-20 Transfer event
```

如果它直接一对一来源于一个 Log，那么可以用：

```text
chain_id
tx_hash
log_index
```

作为基础唯一键。

例如：

```sql
CREATE TABLE fact_token_transfer (
    chain_id          BIGINT NOT NULL,
    tx_hash           TEXT NOT NULL,
    log_index         BIGINT NOT NULL,
    contract_address  TEXT NOT NULL,
    from_address      TEXT NOT NULL,
    to_address        TEXT NOT NULL,
    amount_raw        NUMERIC NOT NULL,
    PRIMARY KEY (chain_id, tx_hash, log_index)
);
```

这样，同一个 Log 再来一次：

```text
same chain_id
same tx_hash
same log_index
```

数据库不会把它当成第二个事实。

---

## 十一、为什么 from_address + to_address + amount 不适合作唯一键？

假设同一笔 Transaction 里真的发生两次完全相同的 Transfer：

```text
Alice → Bob 100 USDC
Alice → Bob 100 USDC
```

这在技术上是可能存在的。

如果你设计：

```text
from_address
+
to_address
+
amount
```

作为 Unique Key，

那么两条真实不同的 Event 会被错误地合并。

所以：

> Business values are not automatically source identity.

唯一键首先应该回答：

> What makes this row this specific blockchain fact?

---

## 十二、Idempotency 是什么？

Idempotency 的核心意思是：

> 同一个操作执行一次和执行多次，最终结果相同。

例如：

第一次处理：

```text
Transfer A
↓
database has Transfer A
```

第二次重复处理：

```text
Transfer A
↓
database still has only one Transfer A
```

那么 Sink 就具有幂等性。

可以简单记成：

```text
process(A)
process(A)
process(A)

final state
=
process(A) once
```

---

## 十三、为什么 Idempotency 比“防止重复消息”更现实？

你当然可以尝试让上游：

```text
never send duplicate
```

但真实分布式系统里很难保证。

因为存在：

```text
Retry
Consumer restart
Network timeout
Producer retry
Backfill
Replay
Reorg correction
```

所以更成熟的设计不是要求：

> Duplicate must never happen.

而是：

> Duplicate may happen, but repeated processing must not corrupt the sink.

也就是：

> Design for duplicate tolerance.

---

## 十四、银行系统类比

假设银行付款接口：

```text
payment_request
request_id = PAY123
amount = 1000
```

客户端发送请求后网络超时。

客户端不知道银行是否已经处理成功，于是 Retry：

```text
PAY123
```

如果银行系统每收到一次就扣款一次：

```text
第一次 -1000
第二次 -1000
```

这显然不行。

所以系统会用：

```text
request_id
```

做 Idempotency Key。

如果：

```text
PAY123 already processed
```

Retry 不会再次扣款。

Blockchain Sink 完全类似。

---

## 十五、Insert-only 为什么容易出问题？

如果 Consumer 逻辑是：

```sql
INSERT INTO fact_token_transfer (...)
VALUES (...);
```

而数据库没有任何 Unique Constraint，

那么重复消息：

```text
A
A
A
```

就会变成：

```text
3 rows
```

因此：

> Application logic alone is not enough.

更可靠的方式是：

```text
Application Idempotency
+
Database Constraint
```

双层保证。

---

## 十六、数据库层的幂等设计

例如 PostgreSQL：

```sql
INSERT INTO fact_token_transfer (
    chain_id,
    tx_hash,
    log_index,
    contract_address,
    from_address,
    to_address,
    amount_raw
)
VALUES (
    1,
    '0xabc',
    7,
    '0xusdc',
    '0xalice',
    '0xbob',
    100000000
)
ON CONFLICT (chain_id, tx_hash, log_index)
DO NOTHING;
```

如果同一条事实再次到达：

```text
same unique key
```

数据库不会再插入第二行。

这是一种非常典型的：

> Idempotent Write.

---

## 十七、DO NOTHING 不是所有场景都够

假设第一次写入：

```text
amount_raw = 100000000
```

后来 Decoder Bug 修复以后 Replay，

正确结果变成：

```text
amount_raw = 1000000
```

如果你一直：

```sql
ON CONFLICT DO NOTHING
```

旧错误数据不会被修正。

所以 Replay / Repair 场景可能需要：

```text
UPSERT
```

例如：

```sql
INSERT INTO ...
ON CONFLICT (...)
DO UPDATE
SET amount_raw = EXCLUDED.amount_raw;
```

这里要区分两个目标：

```text
Duplicate protection
```

和：

```text
Correction / Repair
```

二者相关，但不完全一样。

---

## 十八、[视角前提] Idempotency 不等于 Immutable

这是容易混淆的一点。

Idempotency 是：

> 同一输入重复执行，不应该重复制造事实。

Immutable 是：

> 数据写入以后不修改。

Blockchain 数据平台里，有些层：

```text
Raw immutable archive
```

可以尽量 append-only。

但有些层面对：

```text
Reorg
Decoder correction
Enrichment update
Canonical correction
```

可能需要更新。

所以：

> Idempotent does not necessarily mean insert-only.

真正重点是：

> repeated processing converges to the correct final state.

---

## 十九、Duplicate Delivery 和 Duplicate Fact 的区别

假设 Kafka 里出现：

```text
Message A
Message A
```

这是：

> Duplicate Delivery.

但如果 Sink 幂等：

```text
database:
Fact A
```

只有一行。

那么：

```text
Delivery duplicate
```

存在，

但：

```text
Data Quality Uniqueness Failure
```

没有发生。

这非常重要。

因为：

> Duplicate message is not automatically duplicate data.

真正的数据质量问题发生在：

```text
same fact stored multiple times
```

---

## 二十、反过来也成立：没有 Duplicate Delivery，也可能产生 Duplicate Fact

假设 Indexer Bug 把同一个 Log decode 了两次：

```text
Log 7
↓
Transfer A
Transfer A
```

Kafka 中是两条不同消息：

```text
message 1
message 2
```

从 Kafka 看：

```text
not redelivery
```

但业务事实却是重复的。

所以：

> Transport-level deduplication is not enough.

最终必须基于：

> Business / Source Identity

判断重复。

---

## 二十一、Replay 为什么必须依赖 Idempotency？

你之前已经学过：

```text
Replay
```

用于：

```text
Decoder bug repair
Historical correction
Logic bug repair
```

如果 Replay 一个范围：

```text
block 10,000 → 20,000
```

而原数据没有删除，又没有 Idempotent Sink：

```text
old rows
+
replayed rows
```

会导致大量 Duplicate。

所以 Replay Safety 的前提之一是：

> Idempotent processing.

可以理解为：

```text
Replay-safe
=
Deterministic logic
+
Stable identity
+
Idempotent sink
```

---

## 二十二、Backfill 也是一样

Realtime 已经处理：

```text
block 100 → 1000
```

你做一个 Backfill：

```text
block 500 → 700
```

如果这个区间原来部分数据已经存在，

那么 Backfill 再跑时必须允许：

```text
existing facts
+
same facts arriving again
```

而不产生重复。

因此：

> Backfill must be idempotent.

---

## 二十三、Reorg Correction 为什么更复杂？

Reorg 会产生：

```text
Old canonical branch
↓
becomes orphan

New branch
↓
becomes canonical
```

这时问题不只是：

```text
duplicate
```

还包括：

```text
old facts must be invalidated / removed / marked non-canonical
new facts must be inserted
```

因此 Reorg 场景中的幂等性是：

```text
Replay same correction multiple times
↓
final canonical state remains correct
```

我们会在第 7 课完整展开。

---

## 二十四、如何检测 Duplicate Facts？

假设：

```text
fact_token_transfer
unique grain:
chain_id + tx_hash + log_index
```

那么可以检查：

```sql
SELECT
    chain_id,
    tx_hash,
    log_index,
    COUNT(*) AS cnt
FROM fact_token_transfer
GROUP BY
    chain_id,
    tx_hash,
    log_index
HAVING COUNT(*) > 1;
```

如果返回：

```text
chain_id = 1
tx_hash = 0xabc
log_index = 7
cnt = 3
```

说明：

```text
same source fact
stored three times
```

---

## 二十五、为什么 Duplicate Check 也必须基于 Grain？

假设你写：

```sql
GROUP BY tx_hash
HAVING COUNT(*) > 1
```

对于 Log 表来说，

一笔 Transaction 本来就可能有：

```text
10 logs
```

于是：

```text
COUNT(*) = 10
```

你会错误地把正常数据判断成 Duplicate。

所以 Duplicate Detection 的前提仍然是：

> Define the row grain first.

这和上一课 Completeness 是同一条基础原则。

---

## 二十六、Unique Constraint 是 Data Quality Gate

如果数据库定义：

```sql
UNIQUE (
    chain_id,
    tx_hash,
    log_index
)
```

那么重复写入时数据库会直接阻止。

这意味着：

```text
Data Quality Rule
↓
enforced by storage layer
```

这种规则可以理解为：

> Preventive Quality Control.

而运行 SQL 去找重复：

```text
GROUP BY ... HAVING COUNT(*) > 1
```

更接近：

> Detective Quality Control.

成熟系统通常两者都需要。

---

## 二十七、Preventive vs Detective

可以这样区分：

```text
Preventive
→ prevent bad data from entering

Detective
→ detect bad data already present
```

例如：

```text
UNIQUE constraint
→ Preventive
```

```text
Duplicate audit SQL
→ Detective
```

再比如：

```text
Idempotent consumer
→ Preventive
```

```text
Daily duplicate scan
→ Detective
```

---

## 二十八、为什么还需要 Detective Check？

你可能会问：

> 既然有 Unique Constraint，为什么还要查重复？

因为现实中：

```text
some tables may not have constraints
historical data may predate constraints
ClickHouse may use different dedup semantics
Parquet is file-based
bugs may change grain definition
multi-sink systems may diverge
```

所以：

> Constraint does not eliminate the need for validation.

---

## 二十九、多 Sink 系统中的 Uniqueness

假设：

```text
Kafka
├─ Postgres Consumer
└─ ClickHouse Consumer
```

Postgres 有：

```text
UNIQUE(chain_id, tx_hash, log_index)
```

ClickHouse 配置不当，重复消息直接写入。

最终：

```text
Postgres = 1 row
ClickHouse = 2 rows
```

那么：

```text
Postgres Uniqueness = PASS
ClickHouse Uniqueness = FAIL
Cross-sink Consistency = FAIL
```

一个事故可以同时跨多个质量维度。

---

## 三十、Set-based 思维 vs Add-based 思维

你在 Module 9 已经接触过这个思想。

危险逻辑：

```text
existing_amount
+
new_amount
```

重复执行一次：

```text
+100
```

再执行一次：

```text
+100
```

最终：

```text
+200
```

这是：

> Add-based.

更适合幂等的思路往往是：

```text
for this key
the correct value should be X
```

然后：

```text
replace / upsert to X
```

这是：

> Set-based.

也就是：

```text
Set state to desired result
```

而不是：

```text
Apply delta blindly
```

---

## 三十一、余额表为什么尤其需要注意？

假设 Consumer 处理：

```text
Transfer +100
```

如果直接做：

```sql
UPDATE wallet_balance
SET balance = balance + 100;
```

重复消费一次：

```text
balance + 100
balance + 100
```

结果错误。

所以余额这种 Derived State 通常不能只依赖：

```text
blind additive update
```

需要：

```text
dedupe before apply
```

或者：

```text
event ledger + deterministic rebuild
```

或者其他可重放设计。

---

## 三十二、[Data Engineer 视角] 先保存 Fact，再构建 State

一个常见稳健思路是：

```text
Raw Event
↓
Idempotent Fact Table
↓
Derived State / Aggregate
```

而不是：

```text
Raw Event
↓
直接修改余额
```

因为 Fact Table 如果有稳定 Unique Key：

```text
same event
↓
cannot duplicate
```

那么下游余额、DWS、ADS 的可重建性更强。

这也是为什么 Fact Layer 对数据平台非常重要。

---

## 三十三、一个完整 Duplicate Incident

假设：

```text
Kafka Consumer
↓
Postgres
↓
DWS Daily Transfer
```

某次：

```text
DB INSERT success
↓
Consumer crashes before offset commit
```

重启后：

```text
same offset replayed
```

如果没有 Unique Constraint：

```text
fact_transfer = duplicate
```

然后 DWS：

```sql
SUM(amount)
```

于是：

```text
daily transfer volume over-counted
```

如果 Dashboard 发现问题，

完整的 Root Cause Chain 是：

```text
Consumer crash
↓
Offset not committed
↓
Message redelivery
↓
Sink not idempotent
↓
Duplicate Fact
↓
Uniqueness Failure
↓
Aggregate over-count
```

这里真正的架构缺陷不是：

> Kafka 重复发消息。

而是：

> Sink 无法安全处理重复。

---

## 三十四、怎么修复已经存在的 Duplicate？

第一步：

```text
Identify duplicate key
```

第二步：

```text
Determine canonical row
```

第三步：

```text
Delete / replace duplicates
```

第四步：

```text
Add preventive constraint
```

第五步：

```text
Rebuild downstream aggregates
```

第六步：

```text
Re-validate
```

不能只：

```text
DELETE duplicate
```

然后结束。

因为 DWS / ADS 可能已经把重复数据聚合进去了。

---

## 三十五、Repair Blast Radius

如果 Duplicate Fact 已经进入：

```text
Fact
↓
DWS
↓
ADS
↓
Dashboard
```

那么修复范围不仅是：

```text
Fact row
```

而是：

```text
all downstream products derived from affected range
```

所以数据质量事故必须问：

> What is the blast radius?

这也是后面 Historical Repair 课会继续展开的内容。

---

## 三十六、Idempotency Key 应该满足什么条件？

一个好的 Idempotency / Unique Key 通常应该满足：

```text
Stable
Deterministic
Grain-aligned
Replay-stable
Source-derived when possible
```

也就是：

```text
稳定
可确定
和 grain 对齐
Replay 后不变
尽量来自源事实身份
```

对于 Log：

```text
chain_id + tx_hash + log_index
```

就很符合这些特点。

---

## 三十七、不要把“字段组合很多”误认为 Key 更安全

例如有人可能设计：

```text
chain_id
tx_hash
log_index
contract_address
from_address
to_address
amount_raw
block_number
block_time
```

看起来非常严格。

但 Unique Key 的目的不是：

> 把所有字段都塞进去。

而是：

> 用最小稳定字段集合唯一标识这个 Grain。

字段越多，如果某些字段是：

```text
derived
mutable
enriched
correctable
```

反而会破坏幂等性。

例如：

```text
token_symbol
amount_usd
```

显然不应该参与 Raw Transfer 的 Source Identity。

---

## 三十八、Canonical Identity 和 Derived Attributes 分开

建议形成这个心智模型：

```text
Identity Columns
→ 谁是这条事实

Attributes
→ 这条事实有哪些属性
```

例如：

```text
Identity:
chain_id
tx_hash
log_index
```

而：

```text
Attributes:
contract_address
from_address
to_address
amount_raw
token_symbol
amount_usd
```

其中有些 Attributes 后续可能重算。

但 Identity 不应该随之改变。

---

## 三十九、本课核心心智模型

把这一课压缩成：

```text
Duplicate may happen upstream
↓
Stable Fact Identity
↓
Idempotent Processing
↓
Unique Constraint / Upsert
↓
No Duplicate Fact
↓
Safe Retry / Replay / Backfill
```

真正目标不是：

```text
No duplicate message ever
```

而是：

```text
Duplicate-safe system
```

---

## 本课核心结论

> At-least-once delivery means duplicate delivery is expected behavior, not necessarily a system bug.

> Message Identity and Business Fact Identity are different. Kafka partition/offset should not normally be used as the blockchain fact’s unique identity.

> Unique Key must be derived from Grain. For an Ethereum Log-derived fact, `chain_id + tx_hash + log_index` is a typical stable source identity.

> Idempotency means repeated processing of the same fact converges to the same final state.

> Duplicate Delivery does not become a Data Quality failure if the Sink is idempotent.

> Replay, Backfill and Reorg Correction all require stable identity and idempotent processing.

> Preventive controls such as Unique Constraints and Idempotent Writes should be combined with Detective duplicate checks.

> Repairing duplicate facts must include downstream rebuild and validation, not only deleting extra rows.

---

## 理解检查

### 问题一

Kafka Consumer 处理一条 Transfer：

```text
partition = 2
offset = 500
```

执行顺序：

```text
INSERT Postgres success
↓
Consumer crashes
↓
offset 500 未提交
```

重启以后再次消费：

```text
offset = 500
```

请回答：

1. Kafka 再次发送这条消息，是否一定意味着 Kafka 出错？
2. 如果数据库最终出现两条完全相同的 Transfer Fact，主要属于哪个 Data Quality Dimension？
3. 应该主要在哪一层解决这个问题？

### 问题二

一个 ERC-20 Transaction：

```text
tx_hash = 0xabc
```

产生了：

```text
Log 0
Log 1
Log 2
```

其中 Log 1 和 Log 2 都是 Transfer。

为什么：

```text
chain_id + tx_hash
```

不能作为 `fact_token_transfer` 的唯一键？

你认为更合适的唯一键是什么？

### 问题三

现在已有一张 `fact_token_transfer`：

```text
PRIMARY KEY (
    chain_id,
    tx_hash,
    log_index
)
```

后来发现 Decoder Bug，需要 Replay：

```text
block 10,000,000 → 10,500,000
```

请回答：

1. 为什么这张表比“没有任何 Unique Constraint 的表”更适合 Replay？
2. 如果 Replay 的目的不仅是防止重复，而是修正旧的错误字段，为什么 `ON CONFLICT DO NOTHING` 可能还不够？
3. 这个场景最终更可能需要 `DO NOTHING` 还是 `DO UPDATE / Replace`？为什么？

## 用户回答（问题一）

问题一：
1.Kafka 再次发送这条消息，不一定意味着 Kafka 出错，对 Kafka 来说，offset 保存 consumer 的进度信息。Consumer 按照正常的 offset 进度来消费 message，是否重复消费并不是他需要关注的。
2.如果数据库最终出现了两条完全相同的 transfer fact，这应该属于 uniqueness data quality 问题。
3.应该在 fact 层解决这个问题。出现这个问题，说明在 fact 层 preventive 的功能没有做好

## 老师判断 / 校准（问题一）

通过。

- 第一问判断正确：Kafka 再次投递这条消息并不意味着 Kafka 出错。在 At-least-once 语义下，如果 Consumer 在数据库写入成功后、提交 offset 前崩溃，那么重启后再次消费同一 offset 是正常行为。这里 Kafka 维护的是消费位置（topic / partition / offset），而不是业务事实是否已经成功落库。
- 第二问正确：如果同一个 Transfer Fact 最终在数据库中实际存储了两次，直接的数据质量维度就是 **Uniqueness Failure**。
- 第三问方向正确。更精确地说，应主要在 **Sink / Fact storage layer** 做 Preventive Control，例如稳定的 Business / Source Identity、Unique Constraint、Idempotent Write、UPSERT 等。你说“Fact 层 preventive 没做好”抓到了核心。

这一题最重要的结论是：

> Duplicate Delivery is acceptable; Duplicate Fact is not.

以及：

> At-least-once delivery requires an idempotent sink.

## 用户回答（问题二）

问题二，更适合的唯一键是 chain ID、tx hash、log index

## 老师判断 / 校准（问题二）

通过。

对于 `fact_token_transfer`，如果 Grain 是 one decoded Transfer event，并且每条 Transfer 一对一来源于一个 Ethereum Log，那么：

```text
chain_id
+
tx_hash
+
log_index
```

可以稳定标识这一条链上事实。

这里的关键原因是：

- `chain_id + tx_hash` 只能定位到 Transaction；
- 一笔 Transaction 可以产生多个 Logs；
- `log_index` 用来区分同一 Transaction 内的不同 Log；
- 因此这个组合和该 Fact 的 Grain 对齐。

核心原则仍然是：

> Unique Key must be derived from Grain.

## 用户回答（问题三）

问题三：
1.如果表没有任何的 unique constraint，那么所有 replay 的数据都会进入到这张表里，而先前已经进入了这张表里的数据也都存在在那里，那么这个表里会有 uniqueness 的数据问题
2.On conflict do nothing 可能还不够，是因为如果 replay 修正了旧的错误字段，那么修正后的数据，因为表的 unique constraint，会导致无法修正表里的错误数据。
3.这个场景需要的是 do update replay。因为这样的话，会根据 unique key 将表里的数据 update，既保证了表的 uniqueness，又修复了数据

## 老师判断 / 校准（问题三）

通过。

- 第一问正确：如果没有 Unique Constraint 或其他 Idempotent Sink 机制，Replay 会把已经存在的历史事实再次写入，从而形成 Duplicate Fact，导致 Uniqueness Failure。
- 第二问正确：`ON CONFLICT DO NOTHING` 只能做到“遇到同一个 Unique Key 时不再插入”，但不能修正该 Key 对应的旧错误属性。如果 Replay 的目标包含 Decoder Bug Correction，那么仅防重复是不够的。
- 第三问正确：这里更合适的是 `DO UPDATE / Replace` 一类语义。这样可以基于稳定 Unique Key 收敛到正确最终状态，同时避免重复事实。

这个场景可以概括为：

```text
Stable Unique Key
+
Replay
+
UPSERT / Replace
=
Duplicate-safe correction
```

最重要的区分是：

> Duplicate Protection and Data Correction are related, but they are not the same operation.

以及：

> Idempotency means repeated processing converges to the correct final state, not merely that duplicate INSERTs are rejected.

## 结课判定

Module 12 第 3 课理解检查全部通过，正式完成。

已经能够：
- 区分 Duplicate Delivery 与 Duplicate Fact。
- 理解 At-least-once delivery 为什么天然要求 Idempotent Sink。
- 区分 Kafka Message Identity 与 Blockchain Business / Source Identity。
- 从 Grain 推导 Stable Unique Key。
- 为 Ethereum Log-derived Fact 使用 `chain_id + tx_hash + log_index` 作为典型 Source Identity。
- 理解 Unique Constraint、Idempotent Write、UPSERT 的不同作用。
- 区分 `DO NOTHING` 的防重复语义与 `DO UPDATE / Replace` 的修复语义。
- 理解 Replay / Backfill 安全执行依赖 Stable Identity + Idempotent Processing。
- 理解修复 Duplicate Fact 时不仅要修 Fact，还要考虑下游 DWS / ADS 的 Blast Radius。