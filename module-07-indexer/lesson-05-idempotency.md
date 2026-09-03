# 第5课｜Idempotency：Indexer 重复处理同一个 Block 为什么不会重复写数据

## Lesson Contract
【Blockchain Data Engineer 视角】
- 所属 Module：Module 7 — Indexer
- 本课核心问题：Indexer 因为 Crash、Retry、Checkpoint 未更新等原因，可能重复处理同一个 Block。怎样保证重复处理不会把数据库写坏？
- 学完以后应能解释：Idempotency 是什么；为什么 Indexer 不天然具备幂等性；Unique Key 为什么是幂等写的基础；INSERT / ON CONFLICT DO NOTHING / UPSERT 各自解决什么问题；为什么 At-least-once processing 常常要配合 Idempotent Write；为什么“重复执行”不一定等于“重复结果”；Block / Transaction / Log 如何设计幂等键。
- 本课暂不展开：Reorg 回滚、严格 Exactly Once、Kafka offset、完整事务隔离、大规模 Dedup Pipeline。

## 一、从上一课的 Crash 场景开始
上一课的安全顺序是：
```text
Checkpoint = N
↓
Process Block N+1
↓
Write Data
↓
Checkpoint = N+1
```
如果 Data 已写成功但 Checkpoint 还没更新就 Crash，重启后会再次处理 Block N+1。重复处理在生产 Indexer 里不是异常，而是恢复机制的一部分。于是问题变成：同一个 Block 重跑后，数据库会不会多出一份相同数据？

## 二、Idempotency 是什么
最简单定义：同一个操作执行一次和执行多次，最终结果相同。工程上可以理解为：process Block 101 执行一次后数据库状态为 S，再执行第二次，数据库最终仍为 S，而不是 S + duplicate rows。

## 三、Indexer 不天然幂等
如果代码只是：
```text
for log in logs:
    db.insert(log)
```
重复跑两次就可能重复插入两份数据。所以 Idempotency 不是 Blockchain 自动提供的性质，而是一种必须通过数据库约束与写入逻辑设计出来的 Data Engineering Design Property。

## 四、第一根支柱：Stable Identity
系统首先必须知道“这是不是同一条数据”。不能仅凭 Alice → Bob 100 USDC 这类业务字段判断重复，因为两笔不同链上事实可能业务字段相同。要使用链上稳定身份键。

### Log
```text
(chain_id, tx_hash, log_index)
```
### Transaction
```text
(chain_id, tx_hash)
```
### Receipt
```text
(chain_id, tx_hash)
```
### Block
【链身份视角】
```text
(chain_id, block_hash)
```
block_number 更像高度，不是不可变身份；Reorg 下同一高度可能对应不同 block_hash。

## 五、Unique Key 是幂等写基础
例如 logs 表：
```sql
UNIQUE(chain_id, tx_hash, log_index)
```
第一次写入成功，第二次写入相同 key 时数据库能识别这是同一条 Log。重要的不是“有一个主键”，而是这个唯一约束必须表达链上事实的真实稳定身份。即使表有自增 id 主键，也无法防止同一条 Log 被重复插入。

## 六、三种常见写入策略
### 1. Plain INSERT
配 Unique Constraint 时，重复会抛 duplicate key error。能防重复，但 Retry 体验差。

### 2. INSERT ... ON CONFLICT DO NOTHING
```sql
INSERT INTO logs (...)
VALUES (...)
ON CONFLICT (chain_id, tx_hash, log_index)
DO NOTHING;
```
第一次 INSERT；第二次相同 key 被忽略。最终仍只有一行。这是典型 Idempotent Insert。

### 3. UPSERT
```text
不存在 → INSERT
已存在 → UPDATE
```
适合需要用重跑结果修正已有数据的场景。例如旧 Decoder 有 Bug，修复后重新处理同一 Raw Log，DO NOTHING 会保留旧错误值，而 UPSERT 可以把这一 logical fact 更新成当前正确值。

## 七、DO NOTHING 不是万能答案
当源数据 immutable 且同一 identity 的内容理论上应完全一致时，DO NOTHING 很合理；但如果需要修正 decoder_version、amount、canonical 等派生字段，可能需要 UPSERT。

## 八、为什么 UPSERT 也可以幂等
如果每次都把 amount 设置成 100：
```text
UPSERT 100
UPSERT 100
UPSERT 100
```
最终仍是 100。幂等关注的是最终效果是否随执行次数变化。

## 九、不是所有 UPDATE 都幂等
幂等：
```sql
SET amount = 100
```
重复执行后仍为 100。

非幂等：
```sql
SET total_volume = total_volume + 100
```
重复执行会持续累加。一般来说“赋值”比“累加”更容易保持幂等。

## 十、与数仓分层的关系
明细事实表如 dex_swaps、token_transfers 通常有稳定事件身份，容易通过 Unique Key + UPSERT / DO NOTHING 保证幂等；而直接维护 daily_volume、swap_count 这类累计值，重跑时容易重复累加。因此常见思路是：
```text
先保证明细 Fact 幂等
↓
再从 Fact 重算聚合
```
这和之前学过的 Fact → DWS → ADS 是连贯的。

## 十一、银行数据平台类比
银行流水可能用 source_system + transaction_id 作为唯一业务键。ETL 重跑时，同一流水不能再次插入成第二行。链上系统只是把这个身份换成 chain_id + tx_hash + log_index，本质相同。

## 十二、第二根支柱：Deterministic Processing
只有唯一键还不够。同样 Raw Log 如果第一次 Decode amount=100、第二次 amount=120，说明转换逻辑本身不稳定。一个可靠重放系统还希望：
```text
Same Input
↓
Same Transformation
↓
Same Output
```
Raw Layer + 可追踪 Decoder Version + Stable Key 是非常重要的组合。

## 十三、At-least-once Processing
At-least-once 表示一条数据至少处理一次，可能处理 1 次，也可能因 Retry 处理 2、3 次，但不能 0 次。为了避免漏数据，系统在不确定上一次是否完全成功时会选择再处理一次，因此天然带来 Duplicate Processing Risk。

所以生产级 Indexer 的经典组合是：
```text
Checkpoint
+
At-least-once Processing
+
Idempotent Write
```
操作可能执行多次，但最终数据库效果像只执行一次，也就是 Effectively Once。

## 十四、一个简单的幂等表结构
```sql
CREATE TABLE logs (
    chain_id BIGINT,
    tx_hash TEXT,
    log_index INT,
    block_number BIGINT,
    block_hash TEXT,
    address TEXT,
    topics JSONB,
    data TEXT,
    PRIMARY KEY (chain_id, tx_hash, log_index)
);
```
写入使用：
```sql
INSERT ...
ON CONFLICT (chain_id, tx_hash, log_index)
DO NOTHING;
```
Transactions / Receipts 可使用 (chain_id, tx_hash)，Blocks 可使用 (chain_id, block_hash)。

## 十五、为什么主键设计是 Indexer Correctness 的核心
Retry、Replay、Backfill、Rebuild 都会再次碰到相同链上事实。如果没有 Stable Identity，每次重跑都可能制造重复。所以 Unique Key 不是单纯数据库细节，而是 Indexer Correctness 的核心。

## 十六、Idempotency 与 Deduplication 的区别
Idempotency 强调“重复操作不会改变最终结果”，是事前写入设计；Deduplication 强调“已经有重复数据后再找出并清理”，属于事后修复。生产系统更希望从写入阶段就幂等，而不是长期依赖事后去重。

## 十七、Reorg 与 Idempotency 要区分
如果 block_number=100 原来对应 block_hash=A，后来 Reorg 后对应 block_hash=B，这不是“同一数据被重复处理”，而是链事实变化。那是 canonical / orphaned、rollback / replay 问题，后面单独讲。本课只处理同一链事实的重复处理。

## 十八、本课核心模型
```text
Same Blockchain Fact
↓
Stable Identity
↓
Unique Key
↓
INSERT ON CONFLICT / UPSERT
↓
Retry N times
↓
One Logical Database Row
```
与上一课连接：
```text
Checkpoint
↓
At-least-once Processing
↓
Possible Retry
↓
Idempotent Write
↓
No Duplicate Data
```

## 理解检查与校准
### 问题 1
假设 logs 表没有任何能够表达链上事实身份的 Unique Key。Block 101 第一次写入 500 条 Logs；数据已写成功但 Checkpoint 没更新，程序 Crash；重启后再次 INSERT 500 条。会发生什么？为什么 At-least-once Processing 必须配合 Idempotent Write？

**用户回答：**
> 问题一：这是数据库里面 block 101 的 log 数据记录，会有 1000 条。因为 logs 表没有主键约束，所以它可以随便插入记录，重复的数据也可以写入。At-least-once processing 必须配合 idempotent write（幂等写入）一起使用。否则的话，重复处理多次，它的数据记录就会重复插入多次，造成数据的大量重复。

**老师判断与校准：✅ 通过。**
核心判断正确。术语上进一步收紧：问题不只是“没有主键”，更准确是没有能够表达链上事实身份的 Unique Key / Unique Constraint。即使表有自增 id 主键，也挡不住同一 Log 重复插入。真正需要类似 UNIQUE(chain_id, tx_hash, log_index) 的稳定身份约束。

### 问题 2
假设 ERC-20 Transfer Fact 的唯一键是 (chain_id, tx_hash, log_index)，写入方式是 INSERT ... ON CONFLICT DO NOTHING。为什么同一个 Block 重跑很多次以后 token_transfers 仍然不会出现重复行？

**用户回答：**
> 问题二：
> 因为这个 fact 表有唯一键约束，并且这个唯一键是由 chain ID、TX Hash 和 Log Index 三个字段组成的。
> 写入方式是 （insert on conflict do nothing）：
> 1. 初次解析这个 block 的时候，会直接 insert。
> 2. 重复跑、第二次跑的时候会遇到主键约束，因为 chain ID、TX Hash 和 Log Index 的数据已经存在一份了。写入方式里有判断，当 conflict 的时候是 do nothing，不做任何操作。
> 所以，即使同一个 block 跑了很多次以后，token transfer 数据依然没有变化

**老师判断与校准：✅ 通过。**
逻辑完整。更准确地说，这三列不只是普通组合键，而是在表达“同一条链上、同一笔 Transaction、同一个 Log”的 Stable Identity。整体链路是 Stable Identity → Unique Constraint → Retry-safe Write → One Logical Fact = One Database Row。

### 问题 3
A: SET amount = 100；B: SET total_volume = total_volume + 100。Retry 执行两次，哪个更容易保持幂等？为什么？

**用户回答：**
> 问题 3：
> A 方式更容易保持幂等。
> 因为当执行两次的时候，set amount 等于 100，第二次执行时 amount 还是 100。但是 B 的话，因为第一次 total volume 已经是一百，所以第二次 total volume 会变成 101

**老师判断与校准：✅ 通过，但有一个数值笔误。**
A 的判断正确。B 第二次执行后不是 101，而是 200：初始 0，第一次 0+100=100，第二次 100+100=200。A 重复执行最终仍为 100，因此更容易保持幂等；B 每次都会继续累加，因此不是幂等操作。

## 本课重点总结
1. Indexer 不天然幂等，幂等性必须设计出来。
2. Stable Identity + Unique Constraint 是幂等写基础。
3. 同一 logical fact 应始终对应同一数据库行。
4. DO NOTHING 适合 immutable 且不需修正的事实；UPSERT 适合需要重跑修正的派生结果。
5. SET value = X 一般比 value = value + X 更容易幂等。
6. At-least-once Processing 必须和 Idempotent Write 配合，才能实现 Effectively Once。
7. Reorg 不是普通 Retry，而是链事实变化，后续单独处理。