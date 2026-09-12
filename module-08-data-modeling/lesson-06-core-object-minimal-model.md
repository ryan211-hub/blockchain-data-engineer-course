# 第6课｜核心对象最小模型：Wallet / Token / Transfer / Swap

## Lesson Contract

【Blockchain Data Engineer 视角】

所属 Module：Module 8 — 数据建模。

本课核心问题：

> 如果现在让你真正设计一个最小链上分析模型，Wallet、Token、Transfer、Swap 应该分别是什么表？它们的 Grain、Key、Identity 和关系怎么确定？

学完本课，你应该能够：

1. 为 Wallet、Token、Transfer、Swap 定义清晰 Grain。
2. 区分哪些应该是 Dimension，哪些应该是 Fact。
3. 为每张表设计最小可用 Key。
4. 理解 Wallet / Token 为什么更像“对象”，Transfer / Swap 为什么更像“事件”。
5. 把 Source Identity、Business Identity、Fact / Dimension、Layer 连接成一套最小模型。
6. 根据查询需求判断应该从哪张表出发。

本课明确不展开：

- Wallet Label 算法；
- Token Registry 治理；
- DEX 协议内部 AMM 数学；
- User Swap Intent 的复杂路由识别；
- ETL 调度；
- ClickHouse / Postgres 选型。

---

## 一、先把前五课真正合起来

到目前为止，我们已经学了：

```text
Lesson 1
为什么不能直接拿 RPC 当分析模型

Lesson 2
Grain：一行代表什么

Lesson 3
Fact / Dimension：事件与对象属性怎么分

Lesson 4
Source Identity / Business Identity：它到底是谁

Lesson 5
Raw / Normalized / DWS / ADS：它应该放在哪一层
```

现在第一次把它们用于一个完整设计。

假设产品提出三个需求：

```text
1. 查询一个 Wallet 最近收到哪些 Token

2. 查询 USDC 在 Ethereum 上所有 Transfer

3. 查询 Uniswap 某个 Pool 最近发生的 Swap
```

我们需要一个最小数据模型。

---

## 二、先不要设计表，先找 Business Object

这是非常重要的顺序。

不要一上来写：

```sql
CREATE TABLE ...
```

先问：

> 业务世界里有哪些对象？

这里至少有：

```text
Wallet
Token
Transfer
Swap
```

但是这四个对象性质并不相同。

### Wallet

更像一个：

```text
Entity / Object
```

它存在于链上，并参与很多事件。

### Token

也是：

```text
Entity / Object
```

它有地址、symbol、decimals 等属性。

### Transfer

不是一个长期存在的对象。

它表示：

> 某个时间发生了一次资产转移。

所以它是：

```text
Event / Fact
```

### Swap

同样表示：

> 某个时间发生了一次交换事件。

因此：

```text
Wallet → Dimension-like Object
Token  → Dimension-like Object

Transfer → Fact
Swap     → Fact
```

这里先用 “Dimension-like” 是因为工程里 Wallet 不一定必须真的落成传统维表，但从建模语义来说，它首先是“对象”，不是“事件”。

---

## 三、先设计 Token：最清晰的 Dimension

【Source Identity 视角】

Token 在 EVM 中最典型的 Source Identity 是：

```text
(chain_id, token_address)
```

所以：

```text
dim_tokens
```

可以设计成：

```text
chain_id
token_address

symbol
name
decimals
token_type

business_asset_id
```

Grain：

> 一行 = 一个 Chain 上的一个 Token Contract。

Key：

```text
(chain_id, token_address)
```

这里：

```text
symbol
name
decimals
token_type
```

是 Dimension Attributes。

而：

```text
business_asset_id
```

是上一课讲过的：

```text
Source Identity
→ Business Mapping
→ Business Identity
```

例如：

```text
Ethereum USDC ─┐
Base USDC     ─┼→ business_asset_id = USDC
Arbitrum USDC ─┘
```

---

## 四、为什么 decimals 放 dim_tokens，而不是 fact_token_transfers？

这是个非常典型的问题。

一笔 USDC Transfer：

```text
amount_raw = 100000000
```

USDC：

```text
decimals = 6
```

于是：

```text
amount = 100
```

其中：

```text
amount_raw
```

属于 Transfer Event 的事实。

而：

```text
decimals
```

属于 Token 这个对象的属性。

所以：

```text
fact_token_transfers.amount_raw
```

合理。

```text
dim_tokens.decimals
```

也合理。

但如果每一条 Transfer 都重复保存：

```text
symbol
name
decimals
```

那你其实把 Token Dimension 的职责泄漏到了 Fact。

当然，真实工程里可能为了查询性能做 Denormalization，但那是性能设计，不改变语义归属。

---

## 五、设计 Wallet：先保持简单

Wallet 最小模型：

```text
dim_wallets
────────────────
chain_id
wallet_address

first_seen_block
first_seen_time
```

Grain：

> 一行 = 一个 Chain 上的一个 Wallet Address。

Source Identity：

```text
(chain_id, wallet_address)
```

Key：

```text
(chain_id, wallet_address)
```

现在注意：

```text
entity_name = Binance
```

不应该成为 Wallet Source Identity。

如果未来有：

```text
0xA → Binance
0xB → Binance
0xC → Binance
```

这属于更高层的 Business Mapping。

最小模型里可以以后扩展：

```text
entity_id
```

但不能覆盖：

```text
wallet_address
```

---

## 六、Wallet 有一个特殊点：它不像 Token 那么“稳定有元数据”

Token 通常可以查询：

```text
symbol
name
decimals
```

Wallet 本身链上往往没有：

```text
owner_name
company_name
category
```

例如地址：

```text
0xABC...
```

链上不会直接告诉你：

```text
这是 Binance Hot Wallet
```

这种字段往往来自：

```text
Label Provider
Manual Research
Heuristics
External Dataset
```

所以：

```text
Wallet Source Identity
```

和：

```text
Wallet Business Metadata
```

之间的边界尤其重要。

---

## 七、接下来是最重要的 Fact：Token Transfer

我们已经多次用过：

```text
fact_token_transfers
```

现在正式设计一次。

字段可以是：

```text
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

Grain：

> 一行 = 一个 canonical Token Transfer Event。

在 Ethereum ERC-20 Event 模型下，一个非常自然的 Unique Key 是：

```text
(chain_id, tx_hash, log_index)
```

为什么？

因为：

```text
tx_hash
```

只能定位 Transaction。

一个 Transaction 可以产生很多 Logs。

所以还需要：

```text
log_index
```

来定位该 Transaction 中具体哪条 Log。

再加：

```text
chain_id
```

作为多链 Namespace。

---

## 八、Transfer 表的关系

`fact_token_transfers` 中：

```text
token_address
```

可以关联：

```text
dim_tokens
```

通过：

```text
(chain_id, token_address)
```

而：

```text
from_address
to_address
```

逻辑上都可以关联：

```text
dim_wallets
```

通过：

```text
(chain_id, wallet_address)
```

所以关系大概是：

```text
                 dim_tokens
                     ↑
                     │ token
                     │
dim_wallets ← fact_token_transfers → dim_wallets
   from                                 to
```

这已经开始有经典 Star Schema 的味道。

但本 Module 不要求你现在深入 Star Schema 理论。

只要理解：

> Fact 记录事件，Dimension 描述参与事件的对象。

---

## 九、为什么 Transfer 不应该和 Transaction 合成一张表？

例如 Transaction：

```text
tx_hash = 0x123
from = Alice
to = Router
gas_used = ...
```

但内部可能产生：

```text
Transfer #1
USDC Alice → Pool

Transfer #2
WETH Pool → Alice

Transfer #3
Fee ...
```

所以：

```text
Transaction Grain
≠
Transfer Grain
```

Transaction：

> 一行 = 一笔 Transaction。

Transfer：

> 一行 = 一次 Transfer Event。

如果强行合并：

```text
tx_hash
gas_used
token
from
to
amount
```

那么：

```text
gas_used
```

会在多个 Transfer Row 上重复。

之后：

```sql
SUM(gas_used)
```

就可能被重复计算。

这正是我们 Lesson 2 讲的：

```text
Mixed Grain
```

问题。

---

## 十、再设计 Swap

先从最稳定的链上语义开始：

```text
fact_pool_swaps
```

字段示例：

```text
chain_id

block_number
block_time

tx_hash
log_index

protocol
pool_address

token_in_address
token_out_address

amount_in
amount_out

sender_address
recipient_address
```

Grain：

> 一行 = 一次 Pool Swap Execution。

Key：

如果 Swap 是由唯一 Event Log 表示：

```text
(chain_id, tx_hash, log_index)
```

仍然是一个很自然的 Unique Key。

---

## 十一、为什么先建 Pool Swap，而不是直接建 User Swap？

因为：

```text
Pool Swap
```

更接近链上可直接观察的事实。

例如 Alice：

```text
1000 USDC
→ WETH
→ 3000 DAI
```

可能经过：

```text
Pool 1
USDC / WETH

Pool 2
WETH / DAI
```

于是链上观察到：

```text
2 个 Pool Swap
```

但用户业务意图可能是：

```text
USDC → DAI
```

即：

```text
1 个 User Swap
```

所以：

```text
Pool Swap
= Source-level / Execution-level Fact

User Swap
= Higher-level Business Fact
```

在最小模型中，我们优先保留：

```text
fact_pool_swaps
```

因为它更容易建立稳定、可追溯的事实定义。

---

## 十二、Transfer 与 Swap 是什么关系？

这点很容易混淆。

一次 Swap 通常会导致若干 Token Transfer。

例如：

```text
Alice → Pool
1000 USDC

Pool → Alice
0.4 WETH
```

所以同一 Transaction 中可能有：

```text
1 Swap Event
+
2 Transfer Events
```

但：

```text
Swap ≠ Transfer
```

Transfer 只回答：

> Token 从哪里移动到了哪里？

Swap 回答：

> 哪个 Pool 完成了一次资产交换？

因此：

```text
fact_token_transfers
```

和：

```text
fact_pool_swaps
```

应该是两个不同的 Fact。

不要试图用一张“万能事实表”装所有链上业务事件。

---

## 十三、一个最小模型已经出现了

现在我们有：

```text
dim_wallets
dim_tokens

fact_token_transfers
fact_pool_swaps
```

关系：

```text
                 dim_tokens
                    ↑   ↑
                    │   │
          ┌─────────┘   └──────────┐
          │                        │
fact_token_transfers        fact_pool_swaps
          │                        │
          └─────────┐   ┌──────────┘
                    ↓   ↓
                 dim_wallets
```

当然真实模型中 Token 会以：

```text
token_in
token_out
token
```

等不同 Role 关联。

Wallet 也会以：

```text
from
to
sender
recipient
```

等不同 Role 关联。

这种现象可以理解为：

> 同一个 Dimension，在不同 Fact 中扮演不同角色。

---

## 十四、最小模型为什么不需要一开始就加入所有字段？

例如 `dim_tokens`，你可以想加：

```text
logo_url
website
twitter
coingecko_id
market_cap
circulating_supply
issuer
risk_score
```

这些字段不是都“错”。

问题是：

> 它们是不是当前最小模型所必需？

同样 `dim_wallets` 也可以加入：

```text
label
entity
category
risk_score
ens_name
```

但如果本课目标只是：

```text
支持 Wallet / Token / Transfer / Swap 查询
```

那我们应该优先保证：

```text
Grain 正确
Identity 正确
Key 正确
Relation 正确
```

而不是字段越多越好。

这是工程上很重要的：

```text
Minimum Viable Model
```

思维。

---

## 十五、现在加入 Layer 视角

上一课我们学：

```text
Raw
Normalized Fact
DWS
ADS
```

本课这些表主要位于：

```text
Normalized / Analytical Base Layer
```

例如：

```text
dim_tokens
dim_wallets
fact_token_transfers
fact_pool_swaps
```

然后可以在它们上面建立：

```text
dws_wallet_daily_activity
dws_token_daily_transfer
dws_pool_daily_volume
```

再上面：

```text
ads_wallet_profile
ads_token_dashboard
ads_dex_leaderboard
```

所以：

```text
Raw Logs
   ↓
Normalized Facts + Dimensions
   ↓
DWS
   ↓
ADS
```

整个模型开始连起来了。

---

## 十六、查询场景决定从哪里出发

需求：

> Alice 最近收到哪些 Token？

主要从：

```text
fact_token_transfers
```

过滤：

```text
to_address = Alice
```

再 join：

```text
dim_tokens
```

获取：

```text
symbol
decimals
```

---

需求：

> USDC 最近一天有多少 Transfer？

如果只做临时分析，可以：

```text
fact_token_transfers
```

如果这是高频查询，更可能：

```text
dws_token_daily_transfer
```

---

需求：

> 某个 Pool 最近发生了哪些 Swap？

从：

```text
fact_pool_swaps
```

查询：

```text
pool_address = ...
```

---

需求：

> Binance 所有 Wallet 最近一天总 Transfer Volume？

这时就需要：

```text
Business Entity Mapping
+
fact_token_transfers
```

甚至进一步建立：

```text
dws_entity_daily_transfer
```

所以设计永远回到：

```text
Query Semantics
```

---

## 十七、最重要的设计顺序

以后面试官给你一句：

> “设计一个链上 Token Transfer 数据模型。”

不要立刻报字段。

应该先说：

```text
1. 先确定 Business Object
2. 再确定 Grain
3. 再确定 Source Identity
4. 再确定 Unique Key
5. 再区分 Fact / Dimension
6. 再设计 Relation
7. 再决定 Layer
8. 最后根据 Query Pattern 做必要的聚合 / Denormalization
```

这是比“我会建几张表”更成熟的数据工程回答。

---

## 十八、本课最终最小模型

```text
dim_tokens
Grain:
one token contract on one chain

Key:
(chain_id, token_address)
```

```text
dim_wallets
Grain:
one wallet address on one chain

Key:
(chain_id, wallet_address)
```

```text
fact_token_transfers
Grain:
one canonical token transfer event

Key:
(chain_id, tx_hash, log_index)
```

```text
fact_pool_swaps
Grain:
one pool swap execution

Key:
(chain_id, tx_hash, log_index)
```

注意：

这两个 Fact 都可能使用：

```text
(chain_id, tx_hash, log_index)
```

作为各自表内唯一键。

但这并不意味着：

> Transfer 和 Swap 是同一种事实。

Key 只是定位方式。

真正决定表语义的是：

```text
Business Object + Grain
```

---

## 十九、本课核心心智模型

```text
Wallet / Token
= Objects
= Dimension-like

Transfer / Swap
= Events
= Facts
```

再加：

```text
Grain
↓
Identity
↓
Key
↓
Fact / Dimension
↓
Relation
↓
Layer
```

最终形成：

```text
Raw
 ↓
Normalized Facts + Dimensions
 ↓
DWS
 ↓
ADS
```

本课最需要记住一句：

> **先定义“一行是什么”，再定义“它是谁”，最后才定义“表里放什么字段”。**

---

# 理解检查

## 问题 1

你要设计：

```text
fact_token_transfers
```

请回答：

1. 它的 Grain 是什么？
2. 为什么 `(chain_id, tx_hash)` 不能作为唯一键？
3. 为什么 `(chain_id, tx_hash, log_index)` 更合适？

---

## 问题 2

现在有：

```text
dim_tokens
```

和：

```text
fact_token_transfers
```

字段：

```text
symbol
decimals
amount_raw
amount
```

请判断这些字段更适合分别放在哪里，并说明原因。

---

## 问题 3

Alice 发起一次：

```text
USDC → WETH → DAI
```

实际经过两个 Pool。

请回答：

1. `fact_pool_swaps` 应该保存几行？
2. 如果建立 `fact_user_swaps`，可能保存几行？
3. 为什么两张表都可以是正确的 Fact，而不能说其中一张“粒度更粗所以不正确”？