# 第8课｜从分析需求反推数据模型：Query Semantics 驱动建模

【Blockchain Data Engineer 视角】

这一课是 Module 8 前面内容的一次真正整合。

前 7 课我们实际上一直在拆零件：

```text
Grain
Fact / Dimension
Source Identity
Business Identity
Raw / Normalized Fact / DWS / ADS
Wallet
Token
Transfer
Swap
Transaction
```

但是现实工作中，产品经理通常不会对你说：

> 请设计一个 Grain 为 `(chain_id, tx_hash, log_index)` 的 Token Transfer Fact。

他更可能只给你一句：

> 我想看一个钱包最近 30 天都收到了哪些 Token。

或者：

> 我想统计 Ethereum 上 USDC 每天的转账金额和活跃钱包数。

这时候真正考验数据工程师的不是“会不会建表”，而是：

> **你能不能把业务问题翻译成正确的数据模型。**

---

## Lesson Contract

所属 Module：Module 8 — 数据建模。

本课核心问题：

> 面对一句业务分析需求，如何反推出正确的 Business Object、Grain、Fact / Dimension、Key 和数据层级？

学完本课，你应该能够：

1. 从 Query Semantics 判断真正分析的对象是什么。
2. 不被字段名或 RPC 对象误导。
3. 从 Business Object 推导 Grain。
4. 从 Grain 推导 Unique Key。
5. 判断应该使用 Transaction、Transfer 还是 Swap Fact。
6. 判断哪些属性来自 Dimension。
7. 判断需求应直接查询 Normalized Fact，还是建立 DWS / ADS。
8. 识别 Mixed Grain 和“一张万能宽表”的问题。

本课明确不展开：

```text
ETL 调度与实现        → Module 9
Kafka / Streaming    → Module 10
ClickHouse 等选型    → Module 11
系统性 Data Quality  → Module 12
DEX 业务深入         → Module 13
```

所以这一课仍然只讨论：

> **数据模型怎么设计。**

---

## 一、建模真正的起点不是 Table，而是 Query

先看一个需求：

> 查询 Alice 最近 30 天收到的所有 Token，以及每种 Token 收到的总金额。

如果直接开始想：

```text
表名叫什么？
字段放什么？
```

其实顺序已经错了。

第一步应该问：

> 这句话在业务上到底是什么意思？

把它拆开：

```text
Alice
→ Wallet

收到
→ Receive

Token
→ Asset

最近 30 天
→ Time Range

总金额
→ Aggregate Measure
```

再进一步：

```text
“收到 Token”
```

这个行为对应什么链上事实？

答案不是：

```text
Transaction
```

而是：

```text
Token Transfer
```

所以真正的 Business Object 是：

```text
Token Transfer
```

这一步可以叫：

```text
Query Semantics
→ Business Object
```

---

## 二、为什么不能看到“钱包”就从 Transaction 开始？

假设 Bob 给 Alice 转 100 USDC。

【EVM / Protocol 视角】

用户发起的 Transaction 可能是：

```text
from = Bob
to   = USDC Contract
value = 0
```

但 ERC-20 Event 是：

```text
Transfer
from = Bob
to   = Alice
value = 100 USDC
```

所以，如果业务问：

> Alice 收到了哪些 Token？

你去查：

```text
fact_transactions.to_address = Alice
```

可能根本找不到。

因为这里：

```text
Transaction.to_address
```

表示：

> Transaction 直接调用哪个 Account / Contract。

而：

```text
Transfer.to_address
```

表示：

> Token 实际转给谁。

这正好连接上一课我们学的：

```text
字段语义
=
Business Object
+
Grain
+
Role
```

所以第一条非常重要的原则是：

> **不要根据字段名字选择表，要根据业务语义选择 Fact。**

---

## 三、找到 Business Object 后，再确定 Grain

现在我们已经确定：

```text
Business Object
=
Token Transfer
```

那么问：

> 一行应该代表什么？

答案：

```text
一行
=
一次 canonical Token Transfer Event
```

因此：

```text
fact_token_transfers
```

的 Grain 是：

```text
one canonical token transfer event
```

然后才能继续问：

> 怎么唯一定位这一行？

对于 Ethereum ERC-20 Transfer：

```text
(chain_id, tx_hash, log_index)
```

就是一个自然的 Unique Key。

所以推导过程是：

```text
Query Semantics
↓
Business Object = Token Transfer
↓
Grain = one Transfer Event
↓
Unique Key = (chain_id, tx_hash, log_index)
```

注意顺序。

不是：

```text
看到 tx_hash
↓
觉得它像主键
↓
然后开始建表
```

而是：

```text
先定义一行是什么
↓
再决定什么字段能唯一定位这一行
```

---

## 四、为什么 `(chain_id, tx_hash)` 不够？

因为：

```text
1 Transaction
```

可能产生：

```text
Transfer #1
Transfer #2
Transfer #3
```

例如：

```text
Alice
↓
Uniswap Router
↓
Pool
```

内部可能产生：

```text
USDC Alice → Pool
WETH Pool → Alice
Fee Token ...
```

这些 Transfer：

```text
chain_id 相同
tx_hash 相同
```

但：

```text
log_index 不同
```

所以：

```text
(chain_id, tx_hash)
```

定位的是：

> 一笔 Transaction。

而：

```text
(chain_id, tx_hash, log_index)
```

才能定位：

> 其中某一个 Event。

因此：

> **Unique Key 必须服从 Grain。**

---

## 五、然后才判断哪些是 Fact，哪些是 Dimension

现在需求是：

> Alice 最近 30 天收到哪些 Token？

我们的 Transfer Fact 可以保存：

```text
fact_token_transfers
────────────────────
chain_id
block_number
block_time
tx_hash
log_index

token_address

from_address
to_address

amount_raw
amount
```

但是业务最终可能还希望显示：

```text
USDC
USD Coin
6 decimals
```

这些是谁的属性？

不是 Transfer Event 的属性。

而是：

```text
Token
```

的属性。

所以应该来自：

```text
dim_tokens
```

例如：

```text
dim_tokens
────────────────
chain_id
token_address
symbol
name
decimals
token_type
```

关系：

```text
fact_token_transfers
        │
        │ chain_id + token_address
        ↓
    dim_tokens
```

这里的逻辑就是：

```text
Fact
→ 发生了什么？

Dimension
→ 参与这个事实的对象是什么？
```

---

## 六、Alice 又属于什么？

Alice 是：

```text
Wallet
```

在最小模型中，可以有：

```text
dim_wallets
```

Grain：

```text
one wallet address on one chain
```

Key：

```text
(chain_id, wallet_address)
```

但是 Transfer 中有两个 Wallet Role：

```text
from_address
to_address
```

所以同一个：

```text
dim_wallets
```

逻辑上可能被引用两次：

```text
             dim_wallets
             ↑         ↑
             │         │
          sender     receiver
             │         │
             └────┬────┘
                  │
        fact_token_transfers
```

这说明一个 Dimension 可以在一个 Fact 中扮演不同 Role。

不用急着记传统数仓里的术语。

你只需要理解：

```text
Wallet 本身是谁
```

和：

```text
Wallet 在这次事件中扮演什么角色
```

是两回事。

---

## 七、现在业务需求发生变化

最初需求：

> 查看 Alice 最近 30 天收到的所有 Transfer 明细。

这时候：

```text
fact_token_transfers
```

就足够。

但是产品又说：

> 我不需要所有明细，我想看 Alice 每天每种 Token 收到了多少、转出了多少。

注意，这个需求的 Grain 已经变了。

不再是：

```text
one Transfer Event
```

而是：

```text
one day
+
one wallet
+
one token
```

所以可以建立：

```text
dws_wallet_token_daily_flow
```

Grain：

```text
(chain_id, date, wallet_address, token_address)
```

字段例如：

```text
received_amount
sent_amount

received_transfer_count
sent_transfer_count
```

---

## 八、DWS 不是“给 Fact 加几个聚合字段”

这一点很重要。

底层：

```text
fact_token_transfers
```

Grain：

```text
one Transfer Event
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
```

这是两个不同 Grain。

所以：

```text
Normalized Fact
↓
DWS
```

本质上是：

> 从底层事实产生一个新的分析 Grain。

不是：

> 把 SUM(amount) 塞回 Transfer 表。

这也是为什么层级职责必须分开。

---

## 九、什么时候不需要 DWS？

这里容易走向另一个极端：

> 既然聚合查询更快，那是不是所有查询都建 DWS？

不是。

如果需求只是偶尔分析：

> Ethereum 上某个 USDC 地址昨天有多少 Transfer？

完全可以：

```sql
SELECT ...
FROM fact_token_transfers
...
```

直接查询 Normalized Fact。

DWS 更适合：

```text
查询非常高频
+
聚合口径稳定
+
Grain 稳定
+
多个下游重复使用
```

例如：

```text
wallet_daily_activity
token_daily_transfer
pool_daily_volume
```

所以：

【Data Warehouse 视角】

DWS 的核心不是：

> 更快。

而是：

> **把稳定、可复用的分析 Grain 固化下来。**

性能提升通常是结果之一。

---

## 十、再看一个 Swap 需求

业务要求：

> 查询过去 7 天每个 Uniswap Pool 的交易量。

第一步依然不是建表。

先确定：

```text
Query Semantics
```

它关注的是：

```text
Pool-level Swap Execution
```

Business Object：

```text
Pool Swap
```

所以主要 Fact 应该是：

```text
fact_pool_swaps
```

而不是：

```text
fact_transactions
```

也不是：

```text
fact_token_transfers
```

它的 Grain：

```text
one pool swap execution
```

如果 Swap Event 可以唯一映射到一条 Log：

```text
Unique Key
=
(chain_id, tx_hash, log_index)
```

如果业务每天都查：

```text
每个 Pool 每天的 Swap Volume
```

那么进一步可以形成：

```text
dws_pool_daily_swap
```

Grain：

```text
chain
+ date
+ pool
```

推导过程仍然完全一样：

```text
业务问题
↓
Pool Swap
↓
Pool Swap Grain
↓
Pool Swap Fact
↓
Daily Aggregate Grain
↓
DWS
```

---

## 十一、为什么不能直接拿 Transfer 算所有 Swap？

因为：

```text
Transfer
```

回答的是：

> Token 从哪里移动到了哪里？

而：

```text
Swap
```

回答的是：

> 哪个 Pool 执行了一次资产交换？

一次 Swap 通常产生多个 Transfer。

例如：

```text
Alice → Pool
1000 USDC

Pool → Alice
0.4 WETH
```

你不能因为两个 Transfer 和一个 Swap 出现在同一 Transaction 里，就认为：

```text
Transfer = Swap
```

所以还是回到：

```text
Business Object
```

不同。

这一点比字段本身更加重要。

---

## 十二、现在看一个 Transaction 需求

业务要求：

> 统计过去一个月每个 Wallet 发起 Transaction 的平均 Gas Used。

关键词：

```text
Wallet 发起 Transaction
平均 Gas Used
```

Business Object：

```text
Transaction
```

所以应该从：

```text
fact_transactions
```

出发。

因为：

```text
gas_used
```

属于：

```text
Transaction Grain
```

而不是：

```text
Transfer Grain
```

如果你：

```text
fact_transactions
JOIN
fact_token_transfers
```

然后直接：

```sql
AVG(gas_used)
```

就可能出错。

因为：

```text
Transaction A
→ 1 Transfer

Transaction B
→ 5 Transfers
```

Join 后：

```text
A 出现 1 次
B 出现 5 次
```

于是 B 会得到 5 倍权重。

这不是我们真正想要的：

```text
平均 Transaction Gas
```

---

## 十三、这就是 Grain 在 SQL 中真正的意义

很多人学习 Grain 时觉得：

> 它只是一个建模定义。

其实不是。

它直接决定：

```text
SUM
COUNT
AVG
JOIN
```

是否正确。

例如：

```text
Transaction Grain
1 row = 1 transaction
```

Transfer Grain：

```text
1 row = 1 transfer event
```

如果做：

```text
Transaction
1:N
Transfer
```

Join 后：

```text
Transaction-level Measure
```

可能被重复。

所以做任何 Join 前都应该先问一句：

> Join 前后 Grain 发生了什么变化？

这是非常实用的数据工程习惯。

---

## 十四、最危险的一种设计：万能宽表

假设设计这样一张表：

```text
tx_hash
tx_from
tx_to
gas_used

token_address
transfer_from
transfer_to
transfer_amount

pool_address
token_in
token_out
swap_amount

token_symbol
token_decimals

wallet_label

daily_volume
weekly_volume
...
```

看起来非常方便。

但问一句：

> 一行代表什么？

就开始答不出来了。

因为这里混入：

```text
Transaction Grain
Transfer Grain
Swap Grain
Token Dimension
Wallet Dimension
Daily Aggregate Grain
```

这种问题可以看作：

```text
Mixed Grain
+
Layer Leakage
```

最大的危险不是“表太宽”。

而是：

> **表已经失去了稳定的业务语义。**

---

## 十五、宽表并不是永远错误

这里也要避免另一个误区。

真实分析系统可能为了性能建立宽表。

例如：

```text
transfer_with_token_metadata
```

里面可能冗余：

```text
symbol
decimals
```

这不一定错。

因为它仍然可以保持：

```text
Grain
=
one Transfer Event
```

只是在这个 Grain 上 Denormalize 一些 Dimension Attribute。

所以判断宽表是否有问题，不应该问：

> 字段多不多？

而应该问：

> **有没有混入不同 Grain？**

这是两个完全不同的问题。

---

## 十六、以后拿到需求，可以固定用这个设计顺序

这是本课最重要的部分。

建议固定成：

```text
Step 1
Query Semantics
业务到底要回答什么？

↓

Step 2
Business Object
Transaction？
Transfer？
Swap？
Wallet？
Token？

↓

Step 3
Grain
一行代表什么？

↓

Step 4
Source Identity
链上如何识别这个对象？

↓

Step 5
Unique Key
什么字段唯一确定这一行？

↓

Step 6
Fact / Dimension
哪些是事件？
哪些是对象属性？

↓

Step 7
Relation
Fact 之间以及 Fact / Dimension 如何关联？

↓

Step 8
Layer
Raw？
Normalized Fact？
DWS？
ADS？
```

注意：

字段列表其实是在这些步骤之后才逐渐确定出来的。

---

## 十七、把 Module 8 到现在全部串起来

你现在已经有：

```text
Raw Blockchain Data
        ↓
Indexer
        ↓
Normalized Fact
```

例如：

```text
fact_transactions
fact_token_transfers
fact_pool_swaps
```

再配合：

```text
dim_wallets
dim_tokens
```

形成基础分析模型。

然后：

```text
Normalized Facts + Dimensions
        ↓
DWS
```

例如：

```text
dws_wallet_token_daily_flow
dws_token_daily_transfer
dws_pool_daily_swap
```

最后特定应用可以形成：

```text
ADS
```

例如：

```text
ads_wallet_profile
ads_token_dashboard
ads_dex_dashboard
```

所以完整结构是：

```text
Node / RPC
    ↓
Raw
    ↓
Indexer / Decoder / Normalizer
    ↓
Normalized Facts + Dimensions
    ↓
DWS
    ↓
ADS
    ↓
Dashboard / API / Analytics
```

注意这里我们只是在讨论数据模型职责。

真正“怎样调度、怎样跑 ETL”，属于 Module 9。

---

## 十八、本课最终心智模型

以后看到一句业务需求，不要首先问：

> 我要建什么表？

先问：

```text
这个业务问题到底在问什么？
```

然后：

```text
Query Semantics
↓
Business Object
↓
Grain
↓
Identity
↓
Unique Key
↓
Fact / Dimension
↓
Relation
↓
Layer
```

本课最需要记住的一句话：

> **数据模型不是从字段开始设计的，而是从业务问题开始设计的。**

这也是 Module 8 Contract 里“面向查询场景设计模型”的真正含义。

---

## 理解检查 1

业务提出：

> 查询 Alice 最近 30 天收到的所有 USDC Transfer 明细。

请你从建模角度回答：

1. 这个需求主要分析的 Business Object 是 `Transaction` 还是 `Token Transfer`？
2. 应该主要从 `fact_transactions` 还是 `fact_token_transfers` 出发？
3. 这个 Fact 的 Grain 是什么？
4. 为什么 `(chain_id, tx_hash)` 不能作为它的 Unique Key？

### 用户回答

这个需求主要分析的 Business Object 是 Token Transfer，应该主要从 fact_token_transfers 出发这个 fact 的定义是一行代表一条 transfer。

为什么 chain ID 和 tx hash 不能作为它的 unique key？

因为 chain ID 和 tx hash 只能确定唯一的 transaction。如果需要确定 transfer 的话，需要再增加 log index

### 老师判断与校准

通过。

你的四点判断都正确：

1. Business Object 是 `Token Transfer`。
2. 应该从 `fact_token_transfers` 出发。
3. Grain 是“一行代表一条 Token Transfer Event”。
4. `(chain_id, tx_hash)` 只能唯一定位一笔 Transaction；同一笔 Transaction 可能产生多条 Transfer，所以还需要 `log_index` 来区分具体的 Event。

这里再把最后一点固定得更严谨一些：

```text
Transaction Grain
Unique Key
= (chain_id, tx_hash)

Transfer Event Grain
Unique Key
= (chain_id, tx_hash, log_index)
```

【Blockchain Data Engineer 视角】

关键不是“因为 Transfer 多，所以多加一个字段”，而是：

> Unique Key 必须与 Grain 对齐。

你先确定：

```text
一行 = one Token Transfer Event
```

然后才反推：

```text
什么字段能够唯一识别这一行？
```

所以你的回答已经抓住了本题核心。

---

## 理解检查 2

业务需求变成：

> 查询每个 Wallet 每天收到和转出的每种 Token 的总金额，以及收发 Transfer 次数。

请回答：

1. 如果直接从 `fact_token_transfers` 查询，能不能完成这个需求？
2. 如果这个查询每天被大量重复使用，是否值得建立 DWS？
3. 如果建立 DWS，你认为它的 Grain 应该是什么？
4. 为什么这个 DWS 不能继续使用 `(chain_id, tx_hash, log_index)` 作为 Unique Key？

### 用户回答

1 可以完成，fact数据从transfers表中获取，Token的dim属性，需要关联dim表
2 建立DWS是推荐的，每天重复使用，且口径明确固定，建立DWS提高查询效率
3 grain 有 date，wallet address，Token
4 DWS是汇总的数据，由business object 来确定grain ，显然transfers表的unique key是不适用的

### 老师判断与校准

通过，而且这题已经不只是“答对”，而是开始用完整的建模链条在判断。

第 1 点正确。`fact_token_transfers` 本身已经包含 Transfer 明细事实，所以从它出发可以完成统计；如果需要 `symbol`、`decimals` 这类 Token 属性，再关联 `dim_tokens`。这里你已经把“事实”和“维度属性”区分开了。

第 2 点也正确。这个需求具有两个典型特征：一是每天高频重复使用，二是统计口径稳定，因此很适合沉淀成 DWS。这里再补一个边界：建立 DWS 不只是为了“快”，更重要的是把稳定、可复用的分析 Grain 固化下来。

第 3 点基本正确，但需要补全一个关键字段。你说的 Grain 是：

```text
date + wallet_address + token
```

在多链环境下，更完整应该是：

```text
(chain_id, date, wallet_address, token_address)
```

因为同一个 `wallet_address` 或 `token_address` 在不同链上不能默认视为同一个 Source Identity。

第 4 点正确。`(chain_id, tx_hash, log_index)` 描述的是：

```text
one Transfer Event
```

而 DWS 的一行已经变成：

```text
one wallet + one token + one day
```

所以它的 Unique Key 必须跟新的 Grain 对齐，不能继续沿用底层 Transfer Fact 的 Key。

把这一题固定成一句话：

> 从 Normalized Fact 到 DWS 时，Grain 发生了变化，因此 Unique Key 也必须重新定义。

---

## 理解检查 3

业务提出两个需求：

```text
A. 查询某个 Uniswap Pool 每天的 Swap Volume
B. 查询某个用户发起的完整 Swap 意图，例如 USDC → DAI，即使底层经过多个 Pool
```

请回答：

1. A 和 B 的 Business Object 是否相同？
2. A 更适合使用 `fact_pool_swaps` 还是 `fact_user_swaps`？
3. B 更适合使用哪一个？
4. 为什么不能因为 `fact_user_swaps` 的 Grain 更粗，就认为它不如 `fact_pool_swaps` 正确？

### 用户回答

1. A 和 B 的 Business Object 不相同。
2. A 更适合使用 fact_pool_swaps。
3. B 更适合使用 fact_user_swaps。
4. 这两个 Grain 粗细程度不一样，是因为他们服务的目的不同，不存在一个正确、另外一个不正确。

### 老师判断与校准

通过。

你的第 4 点尤其关键：Grain 的“粗/细”不是正确性标准，正确性取决于它是否匹配对应的 Business Object 和查询语义。

- A 关注的是具体 Pool 的执行事实，因此适合 `fact_pool_swaps`。
- B 关注的是用户层面的完整 Swap 意图，因此适合 `fact_user_swaps`。
- Grain 更细或更粗，不代表模型更正确或更错误；关键在于它是否匹配对应的业务语义和查询目的。

可以把这一点固定为：

> Grain 没有脱离业务语义的“好坏”，只有是否适合当前 Business Object。

---

## 结课判定

Module 8 第 8 课三道理解检查全部通过。

已掌握本课核心建模链条：

```text
Query Semantics
↓
Business Object
↓
Grain
↓
Source Identity
↓
Unique Key
↓
Fact / Dimension
↓
Relation
↓
Layer
```

能够从查询需求反推适合的 Transaction / Transfer / Pool Swap / User Swap Fact；能够判断何时直接使用 Normalized Fact、何时形成 DWS；能够识别 Mixed Grain、Join 后 Measure 重复以及万能宽表的风险。

**Module 8 第 8 课正式完成。**