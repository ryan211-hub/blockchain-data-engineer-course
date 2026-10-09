# Module 13 第 8 课｜DEX Data Modeling：Swap Fact、Pool Dimension 与核心指标

## Lesson Contract

所属阶段：第四阶段 Protocol  
所属 Module：Module 13 — DEX（Decentralized Exchange，去中心化交易所）

**本课核心问题：**

> How do we turn protocol-level swap events into reliable analytical data models?

前面七课主要回答“DEX 如何工作”。从这一课开始，我们要把协议知识转化为实际的数据仓库设计能力。

学完后应能：

1. 设计可信的 Pool-level Swap Fact。
2. 解释 Pool Dimension 的 Grain、主键及维度属性。
3. 区分 Event Fact、State Snapshot 与 Derived Business Fact。
4. 为 Uniswap v2 / v3 设计统一的 Normalized Swap Schema。
5. 确定 Volume、Swap Count、Unique Trader 等指标的正确口径。
6. 解释为什么 Routed Swap 与 Pool Swap 不能混为一谈。
7. 识别 Join Fanout、Double Counting、Price Attribution 等数据质量风险。

本课不展开复杂的 v3 LP 收益核算、Aggregator 完整归因算法、ClickHouse 物理优化或 Dashboard 开发。下一课将进一步学习 DEX Analytics。

---

## 一、为什么 Protocol Event 不能直接成为 Dashboard 数据？

假设你现在已经拥有完整的 Ethereum Logs：

```text
blocks
transactions
receipts
logs
```

并成功解析 Uniswap v2、v3 的 Swap Event。

产品经理提出四个需求：

1. 今天 Uniswap 的 Trading Volume 是多少？
2. 哪些 Pool 的交易最活跃？
3. 今天有多少 Unique Traders？
4. USDC/WETH 在不同 Fee Tier 的交易量分别是多少？

看起来似乎只要对 Swap Event 做 SQL 聚合就可以。

但实际存在三个问题。

**第一，v2 和 v3 的 Event Schema 不同。**

v2：

```text
amount0In
amount1In
amount0Out
amount1Out
```

v3：

```text
amount0
amount1
sqrtPriceX96
tick
liquidity
```

**第二，Raw Event 不等于完整业务语义。**

例如 v3 的 `amount0` 为正，代表 token0 净流入 Pool，但你仍然需要映射 token0 的真实 Token Address、Decimals 和交易方向。

**第三，同一个指标可能存在不同 Grain。**

```text
1 user request
→ 2 pool swap events
```

如果不明确统计对象，最终 Dashboard 数字可能看似正确，业务含义却完全错误。

所以本课的主要工作不是“写一条 GROUP BY”，而是建立可信的数据模型。

---

## 二、[Data Engineer 视角] 从 Raw 到 Analytics 的分层

建议使用以下逻辑链路：

```text
Raw Layer
  blocks / transactions / logs / traces
          │
          ▼
Protocol Decode Layer
  uniswap_v2_swap_events
  uniswap_v3_swap_events
          │
          ▼
Normalized Fact Layer
  fact_dex_pool_swap
          │
          ├── dim_dex_pool
          ├── dim_token
          └── token_price_history
          │
          ▼
Business Attribution Layer
  fact_dex_routed_swap
          │
          ▼
DWS / ADS
  daily_pool_volume
  daily_protocol_volume
  trader_activity
```

其中：

- **Raw**：保存链上原始证据。
- **Decode**：根据协议 ABI（Application Binary Interface，应用程序二进制接口）解析事件。
- **Normalized**：形成统一、具有明确 Grain 的业务事实。
- **Business Attribution**：在证据充分时还原用户级交易意图。
- **DWS（Data Warehouse Summary，数据仓库汇总层）**：对事实进行可复用汇总。
- **ADS（Application Data Service，应用数据服务层）**：面向具体产品提供指标。

本课重点是 Normalized Fact、Dimension 和指标口径。

---

## 三、第一张核心表：dim_dex_pool

先确定 Grain：

> One Pool Contract on One Chain.

建议主键：

```text
(chain_id, pool_address)
```

概念 Schema：

```sql
CREATE TABLE dim_dex_pool (
    chain_id           BIGINT NOT NULL,
    pool_address       VARCHAR(42) NOT NULL,
    protocol           VARCHAR(40) NOT NULL,
    protocol_version   VARCHAR(20) NOT NULL,
    factory_address    VARCHAR(42),
    token0_address     VARCHAR(42) NOT NULL,
    token1_address     VARCHAR(42) NOT NULL,
    fee_tier           INTEGER,
    created_block      BIGINT,
    PRIMARY KEY (chain_id, pool_address)
);
```

以上是教学用 PostgreSQL 风格 DDL。实际生产需要统一地址大小写、链标识、类型约束以及合约验证规则。

为什么 Pool 表需要保存 `fee_tier`？

因为在 Uniswap v3 中：

```text
USDC / WETH / 0.05%
USDC / WETH / 0.30%
```

可能是两个独立 Pool。

不能只按：

```text
token0_address + token1_address
```

把它们合并。

需要特别区分：

```text
Pool Identity:
chain_id + pool_address

Pool Business Attributes:
protocol_version
token0
token1
fee_tier
```

`fee_tier` 属于业务维度的重要属性，但不能代替 `pool_address` 主键。

---

## 四、为什么 Pool Dimension 不能直接保存当前 Reserve 当作历史事实？

假设某个 Pool 今天有：

```text
reserve0 = 1,000,000 USDC
reserve1 = 320 WETH
```

明天变成：

```text
reserve0 = 1,200,000 USDC
reserve1 = 370 WETH
```

如果直接覆盖 `dim_dex_pool.reserve0`，就无法使用这张表还原昨天的历史 Reserve。

因此应区分：

```text
dim_dex_pool
→ relatively stable identity and attributes

fact_dex_pool_state
→ state at a defined block / event / time
```

这里涉及你已经学过的：

**Event ≠ State。**

v2 Pool Reserve、v3 Tick、Active Liquidity、Current Price 等动态数据，都需要根据实际产品需求设计独立的 Current State 或 Historical Snapshot 模型。

不能把不断变化的 State 当作 Pool 的静态身份属性。

---

## 五、第二张核心表：fact_dex_pool_swap

这张表是本课中心。

先定义 Grain：

> One successful Pool-level Swap Event.

因此建议自然唯一键：

```text
chain_id + tx_hash + log_index
```

教学用 Schema：

```sql
CREATE TABLE fact_dex_pool_swap (
    chain_id           BIGINT NOT NULL,
    tx_hash            VARCHAR(66) NOT NULL,
    log_index          INTEGER NOT NULL,
    block_number       BIGINT NOT NULL,
    block_timestamp    TIMESTAMP NOT NULL,
    transaction_index  INTEGER NOT NULL,

    protocol_version   VARCHAR(20) NOT NULL,
    pool_address       VARCHAR(42) NOT NULL,

    token_in_address   VARCHAR(42) NOT NULL,
    token_out_address  VARCHAR(42) NOT NULL,

    amount_in_raw      NUMERIC(78, 0) NOT NULL,
    amount_out_raw     NUMERIC(78, 0) NOT NULL,

    amount_in          NUMERIC(38, 18),
    amount_out         NUMERIC(38, 18),

    PRIMARY KEY (chain_id, tx_hash, log_index)
);
```

这是一个精简的教学模型，不代表所有 Token 精度及极大整数都适合 `NUMERIC(38,18)`；生产系统要依据资产精度与计算引擎选择数值类型。

为什么不能只使用：

```text
chain_id + tx_hash
```

作为 Unique Key？

因为：

```text
Transaction 0xABC
  ├── Swap Event log_index = 12
  └── Swap Event log_index = 19
```

它们是两条独立的 Pool Swap Fact。

即使它们来自同一个 Pool，也不能合并。

---

## 六、统一 Uniswap v2 与 v3 的交易方向

我们已经知道两种协议 Event Schema 不同。

对于 v2：

```text
amount0In  = 3,000 USDC
amount1Out = 0.98 WETH
```

Normalized 结果：

```text
token_in  = USDC
amount_in = 3000

token_out  = WETH
amount_out = 0.98
```

对于 v3，典型 Swap Event：

```text
amount0 = +3000000000
amount1 = -980000000000000000
```

假设：

```text
token0 = USDC, decimals = 6
token1 = WETH, decimals = 18
```

那么：

```text
token0 net inflow  = +3,000 USDC
token1 net outflow = -0.98 WETH
```

Normalized 结果仍然是：

```text
token_in  = USDC
amount_in = 3000

token_out  = WETH
amount_out = 0.98
```

统一后的模型对下游 SQL 非常有价值。

但需要注意：协议特有的原始字段、符号、事件地址与 ABI 版本都应在 Decode/Raw 层保留，以支持审计和重放。

---

## 七、第三张重要维度：dim_token

为什么 Swap Fact 需要 Token Dimension？

因为 Raw Amount 并不是 Human-readable Amount。

例如：

```text
USDC amount_raw = 3000000000
USDC decimals   = 6
```

可以推导：

```text
amount = 3000
```

Token 维度可以包含：

```text
chain_id
token_address
decimals
symbol
name
```

主键：

```text
(chain_id, token_address)
```

这里有两个容易出错的地方。

第一，Token Symbol 不能作为唯一身份。

```text
symbol = USDC
```

不足以确认某个链上的合约资产，更不能跨链唯一识别。

第二，Token Metadata 可能不完整、变化或具有不可信内容。

因此真实数据系统需要保留来源、验证和更新策略。

---

## 八、为什么 amount_usd 不是链上直接事实？

假设一条 Swap：

```text
3,000 USDC → 0.98 WETH
```

我们希望计算：

```text
volume_usd
```

但 USD Valuation 是 Derived Metric。

它需要：

```text
amount
×
valuation_price
```

其中价格必须具有明确的：

```text
price_source
price_timestamp / block
price_currency
valuation_method
```

为什么？

因为：

```text
WETH/USD = 3000
```

与：

```text
WETH/USD = 3100
```

会得出不同结果。

因此：

```text
Raw Swap Amount
≠
USD Trading Volume
```

`amount_in_raw` 是协议事件提供的原始量；`volume_usd` 则依赖额外估值口径。

---

## 九、[Metrics 视角] Volume 的统计口径

假设：

```text
USDC → WETH → UNI
```

两个 Pool Swap 的名义成交额各为：

```text
$1,000
```

那么：

```text
Pool Gross Volume = $2,000
```

但经过验证、确认为一次逻辑兑换的用户级 Routed Volume 可能是：

```text
User Routed Volume = $1,000
```

这不是 SQL 计算精度问题。

它是业务 Grain 不同。

所以产品指标应显式命名为：

```text
pool_gross_volume_usd

routed_volume_usd
```

而不是统一使用一个含糊的：

```text
volume_usd
```

**Metric definitions must specify the counting grain.**

---

## 十、Swap Count 到底统计什么？

至少存在三种不同的统计对象：

| 指标 | 统计对象 |
|---|---|
| `pool_swap_count` | Pool Swap Event 数 |
| `swap_tx_count` | 包含 Pool Swap 的不同 Transaction 数 |
| `routed_swap_count` | 已可靠还原的逻辑兑换请求数 |

对于：

```text
1 tx
2 pool swaps
1 confirmed route
```

三个结果分别为：

```text
pool_swap_count = 2
swap_tx_count = 1
routed_swap_count = 1
```

但这只是当前案例。

不能据此定义：

```text
1 Transaction = 1 Routed Swap
```

---

## 十一、Unique Trader 为什么比 Swap Count 更难？

假设：

```text
tx.from = Alice

Swap.sender = Router

Swap.recipient = Next Pool
```

谁才是 Trader？

这取决于你的业务定义与证据。

可能的指标包括：

```text
unique_tx_senders
unique_attributed_traders
unique_swap_recipients
```

三者不能混为一谈。

尤其在智能钱包、Relayer、Aggregator 或合约代理调用场景中，`tx.from` 不一定等于实际经济交易者。

因此构建：

```text
daily_unique_traders
```

时，必须明确 Trader Attribution 规则。

如果没有可靠归因，就不应把交易发送者统计直接命名为“真实用户数”。

---

## 十二、为什么要把 Routed Swap 单独建模？

上一课已经说明：

```text
Pool Swap Fact
= directly decoded execution event

Routed Swap Fact
= reconstructed logical business intent
```

两者的数据来源与可信度不同。

一个简单结构是：

```text
fact_dex_routed_swap
    route_id
    chain_id
    tx_hash
    input_token
    output_token
    input_amount
    output_amount
    attribution_method
    validation_status
```

以及连接表：

```text
bridge_route_pool_swap
    route_id
    pool_swap_id
    hop_index
    branch_index
```

这能表达：

```text
1 logical route
→ multiple pool executions
```

并支持复杂 Split Routing。

但是 `route_id` 必须来自可靠的协议解析或工程生成规则，不能简单假定：

```text
route_id = tx_hash
```

---

## 十三、SQL Join Fanout：DEX 数据仓库中的常见错误

假设：

```text
Pool A
  ├── Swap 1
  ├── Swap 2
  └── Swap 3
```

以及：

```text
Pool A
  ├── Tick 100
  └── Tick 200
```

如果直接按 `pool_address` Join Swap Fact 与 Tick State：

```text
3 Swap rows × 2 Tick rows = 6 rows
```

那么你再：

```sql
SELECT
    SUM(s.amount_in)
FROM fact_dex_pool_swap s
JOIN fact_dex_tick_state t
  ON s.chain_id = t.chain_id
 AND s.pool_address = t.pool_address;
```

可能会错误放大成交量。

因为 Tick State 是一对多且随时间变化的数据。

这叫 **Join Fanout**。

正确处理需要事先明确：

- 是否真的需要 Tick State；
- 需要哪个时间点的 Tick；
- 每条 Swap 应关联哪个状态版本；
- Join 后是否仍保持原 Swap Grain。

不能因为主键字段有交集，就随意关联两张事实表。

---

## 十四、历史价格与 Pool State 关联必须考虑执行顺序

假设同一 Block 中：

```text
Swap A
  ↓
Pool State changes
  ↓
Swap B
```

若要计算：

```text
pool_price_before_swap
```

就必须恢复对应 Swap 执行之前的状态。

不能简单用：

```text
Block End Price
```

给整个 Block 的所有 Swap 赋值。

你在第 5 课已经理解这一问题。

这是一种：

```text
Point-in-time correctness
```

也就是历史时间点正确性。

工程上还应结合：

```text
block_number
transaction_index
log_index
```

以及特定协议的状态更新语义，避免错误关联。

---

## 十五、如何设计 DWS 汇总表？

假设产品需要每日 Pool Volume。

可以设计：

```text
dws_dex_pool_daily
```

Grain：

```text
one chain
+
one pool
+
one UTC date
```

核心字段：

```text
trade_date
chain_id
pool_address

pool_swap_count
pool_gross_volume_usd
```

其来源：

```text
fact_dex_pool_swap
+
validated price attribution
```

示意 SQL：

```sql
SELECT
    CAST(block_timestamp AS DATE) AS trade_date,
    chain_id,
    pool_address,
    COUNT(*) AS pool_swap_count,
    SUM(volume_usd) AS pool_gross_volume_usd
FROM fact_dex_pool_swap_enriched
WHERE is_canonical = TRUE
GROUP BY
    CAST(block_timestamp AS DATE),
    chain_id,
    pool_address;
```

这里 `fact_dex_pool_swap_enriched` 是示意性衍生视图，假设已提供验证过的 `volume_usd` 和 `is_canonical` 字段。

生产环境还需要明确 UTC 日期、USD 价格缺失处理、去重及更新窗口。

---

## 十六、为什么不能直接从 Pool Daily DWS 得到用户成交量？

如果：

```text
USDC → WETH → UNI
```

两个 Pool 都记录约 $1,000 的交易：

```text
Pool A Daily Volume = $1,000
Pool B Daily Volume = $1,000
```

对 Pool Activity 来说，合计 $2,000 合理。

但如果 Dashboard 的问题是：

> 今天用户通过 DEX 总共兑换了多少经济价值？

那么不应直接：

```text
SUM(all pool volumes)
```

并将其命名为：

```text
user_routed_volume_usd
```

正确数据源应是：

```text
validated routed swap facts
```

这就是为什么 DWS 设计必须从 Business Definition 出发，而不是从最容易聚合的表出发。

---

## 十七、银行数据中台类比

假设银行客户提出：

```text
换汇请求：CNY → EUR
```

系统内部执行：

```text
Execution 1:
CNY → USD

Execution 2:
USD → EUR
```

业务系统至少有两种事实：

```text
Customer FX Request
```

以及：

```text
FX Execution Leg
```

如果统计交易台成交量，可以统计 Execution Leg。

如果统计客户换汇请求金额，应统计 Customer Request。

与 DEX 对应：

```text
Customer FX Request
≈ Routed Swap Intent

FX Execution Leg
≈ Pool Swap Fact
```

你过去使用 Oracle / PL/SQL 进行业务数据抽取时，应该熟悉类似情况：底层多条业务流水不一定对应多个客户业务申请。

区块链数据工程只是让这个问题在 Transaction、Trace、Log 与 Protocol Event 之间表现得更加明显。

---

## 十八、一个可靠的 DEX Analytics Data Contract

建议至少规定：

**Pool Dimension**

```text
Grain:
(chain_id, pool_address)

Contains:
protocol
version
token0
token1
fee_tier
```

**Pool Swap Fact**

```text
Grain:
one Pool Swap Event

Unique Key:
(chain_id, tx_hash, log_index)

Contains:
pool
token_in/out
amount_in/out_raw
execution ordering
```

**Pool State History**

```text
Grain:
one pool at a defined state boundary

Contains:
reserves or price/tick/liquidity
```

**Routed Swap Fact**

```text
Grain:
one verified logical swap intent

Contains:
input/output
route identity
attribution evidence
```

**DWS Pool Daily**

```text
Grain:
one date + chain + pool

Metrics:
pool_swap_count
pool_gross_volume_usd
```

五个对象分离后，指标血缘和异常定位才能足够清楚。

---

## 十九、Data Quality 应该验证什么？

本课至少关注六类验证。

1. **Identity**：Pool Contract 是否属于已验证的 DEX Factory 或可信注册来源。
2. **Uniqueness**：`chain_id + tx_hash + log_index` 是否唯一。
3. **Token Mapping**：`token0/token1` 与 `token_in/token_out` 是否对应正确。
4. **Amount Semantics**：Raw Amount、Decimals 和方向是否正确，是否存在无效或异常量。
5. **Temporal Consistency**：价格和状态是否对应正确的 Block / Execution Point。
6. **Canonical Consistency**：Reorg 后是否撤销失效分支的 Facts 和下游 Aggregates。

此外，用户级 Routed Swap 应进行专门的 Route Attribution Validation。

特别要记住：

```text
Decoded successfully
≠
Business fact validated
```

---

## 二十、本课核心结论

> A trustworthy DEX warehouse must model each business object at its natural grain.

> Pool Dimensions describe trading market identity; Pool Swap Facts describe execution events; Pool State History describes changing state.

> Uniswap v2 and v3 require protocol-specific decoding but can share a normalized Swap Fact model.

> USD trading volume is a derived metric requiring an explicit valuation method and price source.

> Pool-level volume and user-level routed volume are valid but different metrics.

> Transaction senders, Swap senders and economically attributed traders are not always the same.

> Joins across different grains must preserve row cardinality and point-in-time correctness.

> Canonical status, uniqueness and data lineage are fundamental to reliable DEX analytics.

---

# 理解检查

## 问题一：Data Modeling

你准备建立：

```text
dim_dex_pool

fact_dex_pool_swap

fact_dex_pool_state
```

请回答：

1. 三张表各自的 Grain 是什么？
2. 为什么不能把 Pool Current State 直接当作静态维度属性，覆盖所有历史状态？
3. `fact_dex_pool_swap` 应该采用什么 Unique Key？
4. 为什么 `token_symbol` 不应该作为 Token Identity？

## 问题二：Metrics 与 Join Fanout

某个 Transaction 执行：

```text
USDC → WETH → UNI
```

两次 Pool Swap 的 USD Volume 各约为 $1,000，已验证属于同一个用户级兑换请求。

请回答：

1. `pool_swap_count` 是多少？`routed_swap_count` 是多少？
2. Pool Gross Volume 和 User Routed Volume 分别是多少？
3. 如果一张 Swap Fact 有 3 行，另一张 Tick State 有 4 行，同一个 Pool 关联后得到 12 行，可能是什么问题？
4. `amount_raw` 和 `volume_usd` 哪一个更接近链上原始事实？为什么？

## 问题三：综合架构判断

某团队提出：

> 我们已经有全部 Uniswap Swap Events，直接用 SQL 聚合就可以建立完整 DEX Dashboard，不需要 Pool Dimension、Token Dimension 或 Routed Swap Fact。

请回答：

1. 这个设计至少有哪些问题？
2. 如果产品要看 Unique Traders，为什么不能直接对 `Swap.sender` 做 `COUNT(DISTINCT ...)` 并认为是真实 Trader 数？
3. 如果发生 Reorg，`fact_dex_pool_swap` 和每日 DWS 分别需要怎样处理？

本课先进入理解检查阶段。可以逐题回答，校准通过后再正式结课。
