# 第5课｜Raw / Normalized Fact / DWS / ADS：数据职责边界

## Lesson Contract

【Blockchain Data Engineer 视角】

所属 Module：Module 8 — 数据建模。

本课核心问题：

> 同一份链上数据，为什么不能从 RPC 原始数据一路“加工覆盖”到最终 Dashboard？为什么要同时保留 Raw、Normalized Fact、DWS、ADS 这些不同层级？

学完本课，你应该能够：

1. 区分 Raw、Normalized Fact、DWS、ADS 四层各自解决的问题。
2. 判断一张表应该放在哪一层，而不是只根据表名判断。
3. 理解为什么 Raw 的核心价值是“保真与可重放”，Normalized Fact 的核心价值是“稳定明细语义”。
4. 理解 DWS 为什么是按明确 Grain 聚合后的可复用指标层。
5. 理解 ADS 为什么更接近具体产品、Dashboard、API，而不应该反过来成为底层事实源。
6. 能从一个链上分析需求反推：应该查询哪一层。

本课不展开：

- ETL Framework 的具体调度与实现，这属于 Module 9；
- Kafka / Streaming，这属于 Module 10；
- ClickHouse / Postgres 等数据库选型；
- 系统性的 Data Quality / Reorg Repair；
- DEX 协议业务细节。

---

## 一、先看一个很现实的问题

假设我们的 Indexer 已经从 Ethereum 得到一条 ERC-20 Transfer Log：

```text
address = 0xUSDC

topics[0] = Transfer(...)
topics[1] = Alice
topics[2] = Bob

data = 0x0000...
```

如果你把 RPC 返回的原始 Log 保存下来，我们可以称它为：

```text
Raw Log
```

Decoder 解析之后得到：

```text
token_address = 0xUSDC
from_address  = Alice
to_address    = Bob
amount_raw    = 100000000
```

再结合：

```text
decimals = 6
```

得到：

```text
amount = 100
```

然后业务要求：

> 统计 USDC 每日 Transfer Volume。

你聚合得到：

```text
date
token
transfer_volume
transfer_count
```

最后 Dashboard 只需要：

```text
2026-09-12
USDC
$2.1B
1,250,000 transfers
```

注意：

这四种数据虽然都来自同一批链上事实，但它们不是同一个层次。

大致可以看成：

```text
Blockchain / RPC
      ↓
Raw
      ↓
Normalized Fact
      ↓
DWS
      ↓
ADS
```

---

## 二、Raw：回答“数据源当时到底说了什么？”

【Source / Indexer 视角】

Raw 层最重要的原则不是“方便分析”。

而是：

> **尽可能忠实地保存数据源提供的原始事实。**

例如：

```text
raw_logs
```

可能保存：

```text
chain_id
block_number
block_hash
tx_hash
log_index
address
topics
data
removed
```

这里：

```text
topics
data
```

仍然保持 RPC / Node 提供的原始结构。

Raw 层回答：

> 节点当时到底返回了什么？

而不是：

> 这条数据业务上是什么意思？

---

## 三、Raw 为什么重要？因为 Decoder 可能写错

假设第一版 Decoder 写错了：

真实：

```text
amount_raw = 100000000
```

但你的代码错误解析成：

```text
amount_raw = 10000000
```

如果系统只保存了解析后的：

```text
fact_token_transfers
```

而没有 Raw，

那么以后你修复 Decoder 时，就可能需要：

```text
重新请求 RPC
重新拉历史区块
重新获取 Log
```

但如果 Raw 还在：

```text
raw_logs
      ↓
new decoder
      ↓
重新生成 fact_token_transfers
```

因此 Raw 一个非常重要的工程价值就是：

```text
Replayability
```

也就是：

> **可以从源事实重新生成下游模型。**

---

## 四、所以 Raw 与 Normalized Fact 的职责不同

Raw 追求：

```text
Source Fidelity
```

Normalized Fact 追求：

```text
Stable Analytical Semantics
```

例如 Raw：

```text
raw_logs
────────────────
address
topics
data
```

对业务分析人员并不好用。

于是 Decoder / Normalizer 把它转成：

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

这时：

> 一行 = 一个 canonical Token Transfer Event。

你已经获得一个明确的：

```text
Business Object
+
Grain
+
Identity
+
Measure
```

这就是 Normalized Fact 的价值。

---

## 五、Normalized 的“Normalized”到底是什么意思？

这里不要把它和数据库范式里的：

```text
1NF / 2NF / 3NF
```

混为一谈。

在我们当前课程语境里：

```text
Normalized Fact
```

主要强调的是：

> 把不同协议、不同原始结构，转换成稳定、统一、适合查询的事实语义。

例如不同 ERC-20 Token：

```text
USDC Transfer Log
USDT Transfer Log
DAI Transfer Log
```

原始 Contract Address 不同，

但它们都可以转换成：

```text
fact_token_transfers
```

统一字段：

```text
token_address
from_address
to_address
amount_raw
amount
```

所以这里的 Normalized 更接近：

```text
Unified / Standardized Analytical Fact
```

---

## 六、到这里为什么还需要 DWS？

现在假设：

```text
fact_token_transfers
```

有 20 亿行。

业务每天都在问：

> 每个 Token 每天的 Transfer Volume 是多少？

如果每一次 Dashboard 查询都执行：

```sql
SELECT
    date(block_time),
    token_address,
    SUM(amount)
FROM fact_token_transfers
GROUP BY
    date(block_time),
    token_address;
```

那么每次都要扫描大量明细事实。

数据虽然“正确”，但查询成本很高。

于是我们可以提前建立：

```text
dws_token_daily_transfer
```

它的 Grain 是：

> 一行 = 一个 Chain 上，一个 Token 在某一天的 Transfer 汇总。

例如：

```text
chain_id
date
token_address

transfer_count
transfer_amount
active_sender_count
active_receiver_count
```

这就是 DWS。

---

## 七、DWS 的本质不是“汇总表”三个字

DWS 最关键的仍然是：

```text
Grain
```

例如：

```text
dws_token_daily_transfer
```

Grain：

```text
chain + date + token
```

而：

```text
dws_wallet_daily_activity
```

Grain：

```text
chain + date + wallet
```

两个都是 DWS，

但它们回答完全不同的问题。

所以不要理解成：

```text
DWS = GROUP BY 一下
```

更准确的是：

> **DWS 是围绕稳定业务分析主题，在明确 Grain 上形成的可复用聚合模型。**

---

## 八、为什么 DWS 还不是 ADS？

假设你已经有：

```text
dws_token_daily_transfer
```

产品团队做一个 Dashboard：

> Stablecoin Market Overview

它只展示：

```text
USDC
USDT
DAI
```

并且需要：

```text
7d volume
30d volume
growth_rate
market_share
rank
```

这些字段已经高度针对这个产品。

于是可以设计：

```text
ads_stablecoin_market_dashboard
```

例如：

```text
date
asset_id
volume_1d
volume_7d
volume_30d
growth_7d
market_share
rank
```

它的目的不是：

> 给所有分析师提供通用底层数据。

而是：

> 直接服务某个 Dashboard / API / 产品。

这就是 ADS。

---

## 九、四层最核心的职责可以这样理解

```text
Raw
↓
What did the source say?
数据源到底给了什么？

Normalized Fact
↓
What actually happened?
以稳定统一的分析语义，发生了什么？

DWS
↓
What reusable metric do we need?
在稳定 Grain 上，哪些指标值得反复复用？

ADS
↓
What does this application need?
这个 Dashboard / API / 产品最终需要什么？
```

如果换成银行数据仓库语言，大概可以类比：

```text
Raw / ODS
→ DWD
→ DWS
→ ADS
```

但只是职责上的类比，不要求不同公司的命名完全一致。

---

## 十、为什么不能只保留 ADS？

这是非常重要的问题。

假设 Dashboard 今天只需要：

```text
USDC daily volume
```

所以你只保存：

```text
date
asset
volume
```

三个月之后业务问：

> 哪些 Wallet 导致了这一天 USDC Volume 暴增？

如果你只有 ADS：

```text
2026-09-12
USDC
2.1B
```

你已经无法回答。

因为：

```text
Wallet
tx_hash
log_index
from_address
to_address
```

这些明细已经消失。

所以：

> **上层模型可以丢失细节，前提是底层事实仍然存在。**

这其实和上一课 Source Identity 的思想非常相似：

```text
底层保真
上层抽象
```

---

## 十一、为什么也不能让所有人都直接查 Raw？

反过来也不行。

假设分析师需要：

> 查询 Alice 收到的所有 USDC Transfer。

如果只提供：

```text
raw_logs
```

那么分析师自己需要：

1. 找出 ERC-20 Transfer Event Signature；
2. 解析 `topics[1]`；
3. 解析 `topics[2]`；
4. 解码 `data`；
5. 查询 Token Metadata；
6. 根据 decimals 换算 amount；
7. 处理链和 Contract Identity。

这意味着：

> 每个分析师都在重复实现 Decoder。

最终很容易出现：

```text
团队 A 的 USDC volume
≠
团队 B 的 USDC volume
```

所以 Normalized Fact 本质上是在建立：

```text
Shared Semantic Contract
```

也就是整个数据平台共享的一套事实定义。

---

## 十二、谁最适合查询哪一层？

这不是绝对规则，但可以建立一个基本判断。

如果你是：

### Indexer / Data Platform Engineer

经常需要：

```text
Raw
+
Normalized Fact
```

因为你需要：

```text
debug
replay
decoder repair
data lineage
```

### Data Analyst

最常使用：

```text
Normalized Fact
+
DWS
```

因为既需要自由分析，也需要性能。

### Dashboard / API

更适合：

```text
DWS
+
ADS
```

因为需要：

```text
稳定
快速
固定口径
低查询成本
```

所以并不存在：

> 越上层越高级。

真正的关系是：

> **不同层解决不同问题。**

---

## 十三、一个完整的数据流

把我们 Module 7 和 Module 8 连起来：

```text
Ethereum Node / RPC
        ↓
raw_blocks
raw_transactions
raw_receipts
raw_logs
        ↓
Parser / Decoder
        ↓
Normalized Fact
────────────────────
fact_transactions
fact_token_transfers
fact_pool_swaps
        ↓
Dimension Enrichment
────────────────────
dim_tokens
dim_wallets
dim_business_assets
        ↓
DWS
────────────────────
dws_token_daily_transfer
dws_wallet_daily_activity
        ↓
ADS
────────────────────
ads_stablecoin_dashboard
ads_wallet_profile
        ↓
Dashboard / API / Product
```

【Blockchain Data Engineer 视角】

你的工作不是只负责其中某一张表。

真正要理解的是：

> **每层为什么存在、它接受什么输入、输出什么语义、下游为什么依赖它。**

---

## 十四、再看一个 Swap 例子

假设 Alice：

```text
1000 USDC
→ WETH
→ 3000 DAI
```

链上实际经过两个 Pool Swap。

### Raw

可能是：

```text
raw_logs
```

保存两个 Pool 的原始 Swap Logs。

### Normalized Fact

```text
fact_pool_swaps
```

Grain：

> 一行 = 一次 Pool Swap execution。

所以：

```text
2 rows
```

如果我们还建立：

```text
fact_user_swaps
```

Grain：

> 一行 = 一次 User Swap Intent。

那么：

```text
1 row
```

### DWS

例如：

```text
dws_token_pair_daily_volume
```

Grain：

```text
chain + date + token_pair
```

### ADS

例如：

```text
ads_dex_top_pairs
```

直接服务：

> DEX Dashboard 的 Top Trading Pairs 排行榜。

所以从 Raw 到 ADS：

> 不是同一张表不断增加字段，而是在不断改变数据的**语义用途与 Grain**。

---

## 十五、一个很重要的设计错误：Layer Leakage

假设你在：

```text
fact_token_transfers
```

里面直接加入：

```text
7d_volume
30d_rank
market_share
dashboard_category
```

会发生什么？

这些字段根本不属于：

```text
Transfer Event Grain
```

它们属于更高层的聚合或应用语义。

这叫一种典型的：

```text
Layer Leakage
```

也就是：

> 上层职责泄漏到底层模型。

反过来也一样。

如果 ADS 还要求前端自己解析：

```text
topics
data
```

那么说明：

> Raw 层职责泄漏到了应用层。

好的模型希望：

```text
每层有明确职责
```

而不是所有逻辑都堆在一起。

---

## 十六、判断一张表应该在哪一层，不要看名字

假设一张表叫：

```text
token_transfer_summary
```

这个名字不能告诉你它一定是 DWS。

你仍然必须问：

1. 一行代表什么？
2. 数据从哪里来？
3. 是否保留源结构？
4. 是否已经形成稳定业务事实？
5. 是否进行了聚合？
6. 聚合是否可跨多个应用复用？
7. 是否只针对某个具体 Dashboard / API？

还是我们之前的原则：

> **先看语义和 Grain，再看表名。**

---

## 十七、本课核心心智模型

把 Module 8 前五课连起来：

```text
业务问题
   ↓
Business Object
   ↓
Grain
“一行是什么？”
   ↓
Identity
“它是谁？”
   ↓
Fact / Dimension
“事件和对象属性怎么分？”
   ↓
Layer
“这个语义应该处在哪一层？”
```

然后：

```text
Raw
= 保真，可重放

Normalized Fact
= 稳定、统一的明细事实

DWS
= 稳定 Grain 上的可复用聚合

ADS
= 面向具体应用的最终模型
```

本课最需要记住一句：

> **Raw 为可追溯与重放负责，Normalized Fact 为统一事实语义负责，DWS 为可复用指标负责，ADS 为具体应用负责。**

---

# 理解检查

## 问题 1

假设 ERC-20 Transfer 的 RPC 原始 Log 是：

```text
address
topics
data
tx_hash
log_index
```

Decoder 解析后得到：

```text
token_address
from_address
to_address
amount_raw
amount
```

请回答：

1. 哪一组更接近 Raw？
2. 哪一组更接近 Normalized Fact？
3. 为什么即使已经有 Normalized Fact，仍然值得保存 Raw？

---

## 问题 2

业务每天都查询：

> 每个 Token 每天的 Transfer Volume 和 Transfer Count。

现在有两种设计：

A：

每次都扫描 `fact_token_transfers` 实时 `GROUP BY`。

B：

建立：

```text
dws_token_daily_transfer
```

Grain：

```text
chain + date + token
```

保存：

```text
transfer_volume
transfer_count
```

请回答：

1. B 为什么更像 DWS？
2. 它相比 Normalized Fact 改变了什么？
3. 为什么不能因此删除 `fact_token_transfers`？

---

## 问题 3

现在 Dashboard 要展示：

```text
Stablecoin Top 10
7d_volume
30d_volume
market_share
rank
```

你准备建立：

```text
ads_stablecoin_dashboard
```

请回答：

1. 为什么这张表更像 ADS，而不是 DWS？
2. 如果未来另一个 API 不需要 `rank`，但需要 Wallet Breakdown，它是否应该被迫直接依赖这张 ADS？
3. 从这个例子解释：为什么“越上层”不等于“越通用”？

## 问题 1｜用户回答
> RPC 原始 log 更接近 raw，解析之后的更接近 normalized fact。为什么即使已经有 normalized fact，仍然值得保存 raw？因为 raw 层最重要的能力是 replayability。
>
> 如果 raw 层不保留的话，那么假设 decoder 有 bug，在 decoder 修复之后，它需要 raw 层的数据重新 decode。如果这时候 raw 层的数据已经没有保存，就只能重新从 RPC 获取。

## 问题 1｜老师判断与校准
**判断：通过。**

你的判断和理由都正确。RPC 原始 Log 更接近 Raw，Decoder 解析后的 `token_address / from_address / to_address / amount_raw / amount` 更接近 Normalized Fact。

你把 Raw 的核心价值归结为 **Replayability** 是准确的：如果 Decoder 有 Bug，只要 Raw 仍然保留，就可以在修复 Decoder 后直接从 Raw 重新生成下游 Normalized Fact，而不必重新向 RPC 拉取完整历史数据。

这里再补一个更完整的工程表述：Raw 不只是为了“备份原始数据”，而是为了同时保留 **Source Fidelity + Replayability + Auditability**。其中本题最核心的是 Replayability：

```text
raw_logs
   ↓
fixed decoder
   ↓
rebuild normalized fact
```

因此可以固定一句：
> **Raw 层负责保留可重新解释的源事实；Normalized Fact 负责提供当前版本的稳定分析语义。**

问题 1 已通过；问题 2、问题 3 待回答。