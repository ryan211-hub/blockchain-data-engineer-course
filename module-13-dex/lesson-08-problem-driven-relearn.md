# Module 13 第 8 课（问题驱动重学版）
## DEX Data Modeling：从一个真实查询需求推导数据模型

### Lesson Contract

本课只解决一个工程问题：

> 一家 Web3 数据公司希望每天查询 Uniswap Pool 的交易量和日末储备量。作为 Data Engineer，我们应该保存哪些数据，怎样设计表，才能保证历史查询正确？

我们将以 Ethereum 上的 Uniswap v2 USDC/WETH Pool 为例。

学完后，你需要能够自己推导：为什么要分开保存 Pool 身份、Swap 事件、Pool State 历史和 Daily Snapshot，以及怎样从这些数据得到可信指标。

本课暂不深入 Uniswap v3 Tick/Position 建模、复杂 Aggregator 路由归因和数据库性能调优。这些内容不会妨碍我们先建立正确的基本模型。

---

## 第一阶段：产品经理提出需求，我们手里有什么？

假设你加入了一家 Web3 数据公司。

公司已经运行了一个 Ethereum Indexer，能够采集：

```text
Block
Transaction
Receipt
Log
```

也可以根据 Uniswap v2 的 ABI（Application Binary Interface，应用程序二进制接口）解析出 `Swap` 和 `Sync` Event。

现在产品经理提出一个需求：

**请为 Pool A 生成每日数据：**

- 这个 Pool 交易的是哪两种 Token？
- 2026-10-09（UTC）发生了多少次 Swap？
- 当天 Pool 的 Trading Volume 是多少？
- 2026-10-09 结束时，Pool 里有多少 USDC 和 WETH？

我们的目标是生成类似下面的结果：

| 日期（UTC） | Pool | Swap Count | Trading Volume | 日末 Reserve |
|---|---|---:|---:|---|
| 2026-10-09 | Pool A | 3 | 待计算 | 待计算 |

其中 Swap Count 为 3 是下面教学数据的结果，其余指标需要继续推导。

先不要设计五六张表。

我们只问一个问题：

**现有的链上记录能回答哪些需求，还有哪些回答不了？**

### 先观察数据

假设 Pool A 的资产是：

```text
token0 = USDC
token1 = WETH
```

在 Block #1000 开始前，Pool A 的 USDC Reserve 是：

```text
100,000 USDC
```

Block #1000 内依次发生三次 Swap：

```text
Block #1000

Tx 10:
    Swap 1
    USDC Reserve after = 103,000

Tx 35:
    Swap 2
    USDC Reserve after = 105,000

Tx 120:
    Swap 3
    USDC Reserve after = 102,000
```

这是一组简化的教学数据。这里三次 Swap 属于不同 Transaction，但都修改了同一个 Pool。

我们先从最简单的需求入手。

---

## 第二阶段：先回答“这个 Pool 今天发生了几次 Swap？”

### 1. 为什么需要 Pool 身份？

假设我们直接读取全部 Uniswap Swap Event。

每条 Event 都包含发出事件的合约地址。

但产品经理说的是：

> 我要查看 USDC/WETH Pool 的数据。

因此我们需要知道：

```text
Pool Address
    ↓
Which Protocol?
Which Token0?
Which Token1?
Which Fee Tier?
```

这里有一个容易犯的错误：直接使用 Token Symbol 识别 Pool。

比如：

```text
USDC + WETH
```

但同一 Token Pair 可以在不同协议版本、不同 Fee Tier 下拥有多个 Pool。

因此，我们需要给 Pool 建立一个稳定身份。

这就自然引出了第一张表：

**`dim_dex_pool` — Pool Dimension**

```text
Grain:
One Pool Contract on One Chain

Unique Key:
chain_id + pool_address

Attributes:
protocol
protocol_version
token0_address
token1_address
fee_tier
```

Grain 就是一行数据究竟代表什么对象。

对于这张表：

```text
One Row = One Pool
```

Pool 是交易市场的身份对象。Token Symbol 是展示属性，不是可靠的链上资产身份；Token Identity 应使用 `chain_id + token_address`。

### 2. 有了 Pool 表，就能统计 Swap Count 吗？

不能。

`dim_dex_pool` 只负责告诉我们 Pool A 是什么，并不记录它执行了多少次 Swap。

我们需要第二张表：

**`fact_dex_pool_swap` — Pool Swap Fact**

```text
Grain:
One successful Pool Swap Event

Unique Key:
chain_id + tx_hash + log_index
```

假设我们解析出三条 Swap Event：

| tx_index | log_index | pool_address | USDC 方向及数量 |
|---:|---:|---|---|
| 10 | 15 | Pool A | In 3,000 |
| 35 | 48 | Pool A | In 2,000 |
| 120 | 171 | Pool A | Out 3,000 |

其中 `log_index` 是在该 Block 的所有 Logs 中的索引，不是 Swap 序号。这里的索引均为教学示例。

于是我们能够回答：

```text
Pool A
2026-10-09
Pool Swap Count = 3
```

两张表的职责开始清晰：

```text
dim_dex_pool
    │
    │ one pool → many swap events
    ▼
fact_dex_pool_swap
```

这里先得到一个设计原则：

> A fact table records business events. A dimension identifies and describes the business object.

但问题来了。

**这两张表能告诉我们 2026-10-09 结束时 Pool A 的 USDC Reserve 吗？**

不一定。

Swap Fact 告诉我们发生了什么兑换，但不等于已经保存了每个需要查询的状态时间点。

于是进入下一个问题。

---

## 第三阶段：为什么还需要保存 Pool State？

现在产品经理要求：

> 查询 Pool A 在 2026-10-09 结束时的 Reserve。

假设我们的系统只有一张 Current State 表：

```text
current_dex_pool_state

Pool A:
USDC Reserve = 102,000
```

看起来已经满足需求。

但是到了 2026-10-10，Pool A 又发生 Swap，Reserve 变成：

```text
104,000 USDC
```

程序执行：

```sql
UPDATE current_dex_pool_state
SET reserve0 = 104000
WHERE pool_address = 'Pool A';
```

此时旧的：

```text
102,000 USDC
```

已经被覆盖。

### 1. 第一个设计问题：Current State 能否回答 History Query？

不能独立回答。

它只能直接给出系统当前维护的有效状态。

这与你熟悉的银行账户余额很相似：

```text
10:00    Balance = 10,000
11:00    Balance = 15,000
12:00    Balance = 12,000
```

如果银行数据库不断覆盖账户余额，最终只剩下：

```text
Current Balance = 12,000
```

但如果客户问：

> 11:30 我的账户余额是多少？

Current Balance 已经不够。

要回答过去的状态，就需要保存历史变化，或者拥有能够准确重建历史状态的数据。

DEX Pool 也是同样的问题。

所以，我们引入：

**`pool_state_change_history` — Pool State Change History**

```text
Pool A
    │
    ├── After Swap 1: 103,000 USDC
    ├── After Swap 2: 105,000 USDC
    └── After Swap 3: 102,000 USDC
```

在 Uniswap v2 中，可以利用 `Sync` Event 获取 Reserve 更新后的数值，并按链上实际执行顺序保存。

这里出现了第三种 Grain：

```text
One Pool
+
One Relevant State Change
```

不是：

```text
One Pool
```

也不是：

```text
One Pool Swap Event
```

虽然 Swap 与 State Change 相关，它们仍然是不同的数据对象。

### 2. 第二个设计问题：能否只保存 Block-end Snapshot？

为了减少存储成本，有人提出：

> 既然产品主要查询历史状态，我们在每个 Block 执行结束时保存一次 Pool Reserve，不就够了吗？

看看 Block #1000：

```text
Before Block #1000:
100,000

After Swap 1:
103,000

After Swap 2:
105,000

After Swap 3:
102,000

Block #1000 End:
102,000
```

如果只保存 Block-end Snapshot，数据库中可能只有：

```text
pool_address = Pool A
block_number = 1000
reserve0 = 102,000
```

那么系统能够回答：

> Block #1000 执行完毕后 Reserve 是多少？

答案是 102,000 USDC。

但如果用户进一步问：

> Swap 2 执行之前，Pool A 的 Reserve 是多少？

正确答案是：

```text
103,000 USDC
```

仅凭当前这个 Block-end Snapshot，无法直接还原。

这说明：

**Snapshot 的采样粒度决定它能直接回答多细的历史查询。**

这是数据模型的设计约束，而不只是存储策略的区别。

### 3. 第三个设计问题：到底应该保存哪一种历史？

这时我们不应该立即决定“所有数据都保存”。

先看业务需要：

| 查询需求 | 适合的数据 |
|---|---|
| 当前最新 Reserve | Current State |
| 某个 Block 结束时的 Reserve | Block-end Snapshot，或可还原该边界的完整历史 |
| 某次 Swap 执行前的 Reserve | 细粒度 State Change History 或等价重放数据 |
| 过去 90 天每天结束时的 Reserve | Daily Snapshot，或能够按日末边界查询的历史状态 |

注意，这些不一定都要成为独立的物理表。

例如，你可以保留完整 State Change History，再派生 Current State 与 Daily Snapshot。

真正的决策标准是：

> Storage granularity should be determined by query and correctness requirements.

**根据查询需要的精度来设计历史粒度，而不是先决定保存频率，再要求业务适应。**

### 4. 第四个设计问题：Ethereum 没有“日末”，怎样获得 Daily Snapshot？

产品经理要求：

> 2026-10-09 的 End-of-Day Reserve。

但 Ethereum 的自然边界是 Block，不是银行营业日。

我们先定义业务日采用 UTC。

那么：

```text
2026-10-09 End of Day
=
2026-10-10 00:00:00 UTC 之前
最后一个 Canonical Block 执行完成后的状态
```

假设：

```text
23:59:35 → Block #1000
23:59:47 → Block #1001
23:59:59 → Block #1002
00:00:11 → Block #1003
```

那么 2026-10-09 的截止区块是：

```text
Block #1002
```

但还有一个重要细节：

**Block #1002 不一定包含 Pool A 的 Swap。**

如果 Pool A 最后一次 Reserve 更新发生在 Block #1000，而之后没有变化，那么 #1002 结束时 Pool A 仍然沿用 #1000 更新后的 Reserve。

因此：

```text
Find last Canonical Block before UTC boundary
             ↓
Find Pool State valid at that block end
             ↓
Carry forward last known state if unchanged
             ↓
Daily Pool Snapshot
```

最终形成：

**`daily_pool_snapshot`**

```text
Grain:
One UTC Date
+
One Chain
+
One Pool

Example:
2026-10-09
Ethereum
Pool A
reserve0 = 102,000 USDC
```

这正是你之前在课堂补充备注中已经推导过的设计。

现在它不再是突然出现的一种数据表，而是**为解决“如何准确查询历史日末 Reserve”而产生的结果**。

---

## 第四阶段：现在能回答每日 Trading Volume 吗？

回到产品经理最初的四个需求。

我们已经有：

```text
dim_dex_pool
fact_dex_pool_swap
pool_state_change_history
daily_pool_snapshot
```

现在产品经理问：

> Pool A 今天的 Trading Volume 是多少美元？

从 Swap Fact 中，我们有三次执行：

```text
Swap 1:
3,000 USDC in

Swap 2:
2,000 USDC in

Swap 3:
3,000 USDC out
```

如果对每次 Swap 选择 USDC 一侧作为成交金额，并且本教学示例假设这三次 USDC 均按 1 USDC = 1 USD 估值：

```text
Pool Gross Volume
= 3,000 + 2,000 + 3,000
= 8,000 USD
```

请注意：

```text
USDC in / out
```

方向不同，但它们都代表 Pool 实际执行过的一次成交。

不能把：

```text
+3,000 +2,000 -3,000
```

得到的净储备变化 `+2,000 USDC` 当作 Trading Volume。

这里存在两个不同指标：

```text
Reserve Net Change
≠
Trading Volume
```

前者描述库存状态如何变化，后者描述执行了多少交易活动。

### 为什么 Token Metadata 开始变得必要？

现实链上 Swap Event 保存的是整数形式的 Raw Amount。

例如：

```text
USDC amount_raw = 3000000000
```

为了得到：

```text
3,000 USDC
```

需要知道：

```text
USDC decimals = 6
```

所以我们需要 Token Metadata，并可以建立：

```text
dim_token
```

Grain：

```text
One Token Contract on One Chain
```

Unique Key：

```text
chain_id + token_address
```

它不是因为“数据仓库标准配置必须有 Token Dimension”才被引入。

而是因为业务查询需要正确解释 Token Amount。

### 为什么还需要 Price Data？

本例中的 USDC 价格是明确说明的教学假设。

但对于：

```text
WETH → UNI
```

仅凭 Token Amount，并不能直接确定 USD Volume。

我们还需要与交易执行时间对应的可信 USD Price，以及估值方法。

因此：

```text
Raw Amount
    +
Token Decimals
    +
Valuation Price
    ↓
Derived USD Volume
```

`volume_usd` 是派生指标，不是 Swap Event 直接输出的链上原始事实。

到这里，我们已经可以为每条 Pool Swap 提供标准化金额及 USD Valuation，再生成 Daily Pool Volume。

---

## 第五阶段：把整个数据模型连接起来

现在回顾，我们并没有一开始就决定建立所有这些表。

每张表的出现，都有具体原因：

| 业务问题 | 不足之处 | 引入的模型 |
|---|---|---|
| Pool A 是哪个交易市场？ | Swap Event 地址缺乏可直接使用的 Pool 业务信息 | `dim_dex_pool` |
| Pool A 今天执行了几次交易？ | Pool Dimension 不记录执行事件 | `fact_dex_pool_swap` |
| 昨天的 Reserve 是多少？ | Current State 被覆盖 | `pool_state_change_history` |
| 怎样高效查询每日结束状态？ | 每次从全部历史重建成本较高 | `daily_pool_snapshot` |
| Raw Amount 怎样变成 Token Amount？ | 缺少 Decimals 和 Token Identity | `dim_token` |
| 怎样得到 USD Trading Volume？ | Token Amount 不等于 USD 金额 | Price Attribution / Enriched Swap |

最后得到：

```text
Raw Blocks / Receipts / Logs
            │
            ▼
    Protocol-specific Decode
            │
            ├─────────────────────────────┐
            ▼                             ▼
    Pool Swap Events              Pool State Changes
            │                             │
            ▼                             ├── Current State
    fact_dex_pool_swap                    │
            │                             └── Daily Snapshot
            ├── dim_dex_pool
            ├── dim_token
            └── valuation prices
            │
            ▼
    Daily Pool Metrics
```

于是产品经理最初的需求可以得到回答：

```text
Pool:
Pool A (USDC / WETH)

UTC Date:
2026-10-09

Pool Swap Count:
3

Pool Gross Volume:
8,000 USD
(under the stated example valuation assumption)

End-of-Day USDC Reserve:
102,000 USDC
```

WETH 日末 Reserve 也需要保存或重建对应的 `reserve1`；本例没有提供其具体数值，所以不能凭空填写。

### 最后一个边界：这是否代表用户级 Trading Volume？

不一定。

我们计算的是：

```text
Pool Gross Volume
```

如果这三次 Pool Swap 其实属于一笔 Multi-hop 或某个 Aggregator 的路由，用户级经济交易量就可能不同。

只有当产品经理进一步要求：

> 我要统计用户实际发起的兑换金额，而不是 Pool 内部成交量。

我们才需要引入：

```text
fact_dex_routed_swap
```

这是上一课已经建立的概念。

我们不在这里重新展开复杂的 Route Reconstruction，只记住：

```text
Pool Execution Grain
≠
User Intent Grain
```

另外，所有历史事实与 Daily Snapshot 都依赖 Canonical Chain。发生 Reorg 时，受影响的事实和派生快照需要失效、重放或重算。

---

## 第六阶段：本课真正需要掌握的工程方法

当你收到一个数据建模需求时，不要立即列出所有可能用到的表。

先依次问：

1. **Business Question**：用户到底想查询什么？
2. **Grain**：答案的一行数据代表什么？
3. **Available Evidence**：链上已经提供哪些直接事实？
4. **Missing Information**：这些事实不足以回答什么？
5. **Storage Decision**：需要增加哪类事实、维度或历史状态？
6. **Validation**：怎样证明查询结果符合业务口径及链上执行语义？

这六个问题，比记住 `dim_dex_pool`、`fact_dex_pool_swap` 的字段名称更重要。

---

# 理解检查：沿用同一个 Pool A 场景

这次不考零散定义，而是判断你会怎样完成需求。

### 问题一：选择正确的 Grain

现在手里有 Pool A 的三条 Swap Event。产品经理要求：

> 查询 2026-10-09 的 Swap Count，同时能查看每条交易明细。

请回答：

1. 只用 `dim_dex_pool` 能否完成？为什么？
2. 为什么 `fact_dex_pool_swap` 的 Grain 应该是一条 Pool Swap Event，而不是一个 Pool 或一个 Transaction？
3. 当我们进一步统计 Daily Swap Count 时，汇总结果的一行应该代表什么？

### 问题二：设计正确的 State History

产品经理追加需求：

> 不仅要查询昨天日末的 Reserve，还要查询 Block #1000 中第二次 Swap 之前的 Reserve。

请回答：

1. 只保存 Current State 能否满足？
2. 只保存 Block-end Snapshot 能否同时回答这两个问题？
3. 如果保留 State Change History，为什么可以支持更细粒度的历史查询？
4. 对于一个 Pool 当天没有发生 Swap 的情况，Daily Snapshot 应怎样处理？

### 问题三：验证业务指标

Pool A 在当天发生三次 Swap：

```text
3,000 USDC in
2,000 USDC in
3,000 USDC out
```

假设这三次 USDC 均按 1 USD 估值。

请回答：

1. 当日 USDC Reserve 的净变化是多少？
2. Pool Gross Volume 是多少？
3. 为什么这两个数字不一样？
4. 如果产品经理以后要求 User-level Routed Volume，我们是否一定能够直接使用当前的 Pool Swap Fact 得到正确答案？

---

这次先停在这里。你可以继续采用逐题回答的方式；我们会围绕同一个场景逐步检查，而不是切换到另一套无关的问题。


## 用户回答（问题驱动重学版｜理解检查问题一）

问题一：

1. 不能，因为 `dim_dex_pool`   是定义了 各种pool 业务对象，表里并没有swap数据。
2. `fact_dex_pool_swap`   保存的是每一次的swap event。
3. one day one pool Swap Count

## 老师判断 / 校准（问题一）

**判定：通过。三个回答都抓住了核心，第 3 点补充一个工程细节。**

**1. 为什么不能只用 `dim_dex_pool`？**

正确。`dim_dex_pool` 主要描述 Pool 的身份和业务属性，例如 Token Pair、Protocol Version、Fee Tier。它不记录每次 Swap，因此无法独立回答交易明细和交易次数。

**2. 为什么 `fact_dex_pool_swap` 的 Grain 是一条 Swap Event？**

正确。每条 Swap Event 代表一次实际的 Pool-level Swap Execution。

即使同一 Transaction 包含多次 Swap，也应分别记录。对应 Unique Key：

```text
chain_id + tx_hash + log_index
```

**3. Daily Swap Count 的汇总 Grain 是什么？**

你回答的 `one day one pool Swap Count` 是正确方向。

需要补充：在多链数据仓库中，还必须包含 `chain_id`，并明确日期使用 UTC。

```text
Grain:
One UTC Date
+
One Chain
+
One Pool
```

例如：

| trade_date_utc | chain_id | pool_address | pool_swap_count |
|---|---|---|---:|
| 2026-10-09 | 1 | Pool A | 3 |

这里要注意：

- `trade_date_utc + chain_id + pool_address` 是汇总表的 Grain / Unique Key。
- `pool_swap_count = 3` 是该 Grain 下计算出来的 Metric（指标）。

这两者不能混淆。

你已经建立了本题最重要的认识：**Pool Dimension 描述对象，Swap Fact 记录事件，Daily Summary 汇总事件。**


## 用户回答（问题驱动重学版｜理解检查问题二）

问题二：

1. 不能， Current State 查询的是当前最新的 pool state.
2. 只保存 Block-end Snapshot 可以查询到昨天日末的 Reserve ，但查不到 另一个问题
3. State Change History 已经保存了 State Change 的所有数据。
4. Daily Snapshot 按正常的逻辑处理， 只是state 没有变化， Reserve与前一天一样。

## 老师判断 / 校准（问题二）

**判定：第 1、2、3 题正确；第 4 题需要补充一个重要的边界条件。**

### 1. Current State 能否满足历史查询？

**正确。**

`current_dex_pool_state` 保存最新有效的 Pool State，不能单独回答昨天日末或者某次历史 Swap 之前的 Reserve。

### 2. Block-end Snapshot 能否满足两个需求？

**正确。**

- 查询昨天日末 Reserve：可以，前提是快照或其历史记录足以还原日末对应区块的状态。
- 查询 Block #1000 中第二次 Swap 之前的 Reserve：仅凭 Block-end Snapshot 不行，因为它没有记录区块内的中间状态。

你已经明确区分了 **Block-level History 与 Event-level History**。

### 3. State Change History 为什么支持细粒度查询？

**正确，但有一个前提。**

你说它保存了所有 State Change，这是核心原因。

更精确地讲，需要保存相关状态变化后的值及其执行顺序，才能确定任意一次 Swap 之前的状态。

例如：

```text
Before Swap 1: 100,000
After  Swap 1: 103,000

Before Swap 2: 103,000
After  Swap 2: 105,000
```

所以第二次 Swap 前的 Reserve 是 103,000 USDC。

### 4. 没发生 Swap，Daily Snapshot 是否直接沿用前一天？

**有条件地正确。**

如果 Pool 当天确实没有发生任何改变 Reserve 的操作，那么你说得对：Daily Snapshot 仍需要生成，Reserve 沿用前一天的状态。

但要特别注意：

**没有 Swap，不代表 Pool Reserve 一定没有变化。**

例如 Uniswap v2 中，Liquidity Provider（流动性提供者）可以增加或移除流动性，触发 Reserve 更新，即使当天没有发生 Swap。

因此应采用：

```text
当天没有 Swap
       ↓
检查是否还有其他 Reserve State Change
       │
       ├── 有变化 → 使用当天日末有效的新 Reserve
       │
       └── 无变化 → Carry Forward 前一天 Reserve
```

决定 Daily Snapshot 数值的依据是 **State 是否变化**，不是 **Swap Count 是否为零**。

### 一个小确认

假设 Pool A 在某一天：

```text
Swap Count = 0

Liquidity Provider 添加流动性
导致 USDC Reserve:
100,000 → 120,000
```

那么当天的 Daily Snapshot 应保存：

A. 100,000 USDC，因为当天没有 Swap。

B. 120,000 USDC，因为日末 Pool State 已经改变。

你回答 A 或 B 即可。通过后，问题二就正式完成。


## 用户回答（问题驱动重学版｜问题二补充确认）

b

## 老师判断 / 校准（问题二最终确认）

**正确，选择 B。第 8 课（重学版）问题二正式通过。**

即使当天没有 Swap，Liquidity Provider 添加流动性仍然会改变 Pool Reserve。

因此，当天的 Daily Snapshot 应记录：

```text
Swap Count = 0
End-of-Day USDC Reserve = 120,000
```

关键原则是：

**Daily Snapshot 反映指定日期结束时的实际 Pool State，而不是当天是否发生过 Swap。**

只有确认当天没有相关 State Change，才能沿用前一天的 Reserve。

目前问题一、问题二均已通过，接下来是问题三：验证业务指标。


## 用户回答（问题驱动重学版｜理解检查问题三）

问题三：

1. 净变化 2,000 USDC&#x20;
2. Pool Gross Volume 是 8000 USDC
3. 这两个业务对象是不一样的， 它们的值不一样是正常的。
4. 不一定，从 Pool Swap Fact 反推不出 user-level 的情况，还需要其他一些信息

## 老师判断 / 校准（问题三）

**判定：通过。四个回答的核心判断全部正确。**

### 1. Reserve 净变化：正确

```text
+3,000 USDC
+2,000 USDC
-3,000 USDC
--------------
+2,000 USDC
```

Pool A 的 USDC Reserve 净增加 2,000 USDC。

### 2. Pool Gross Volume：正确，注意计价单位

三次成交的 USDC 一侧金额合计为：

```text
3,000 + 2,000 + 3,000 = 8,000 USDC
```

本题假设每次交易均按 `1 USDC = 1 USD` 估值，因此：

```text
pool_gross_volume_usd = $8,000
```

你的数值正确。只是要区分 **8,000 USDC 的 Token Amount** 与 **8,000 USD 的估值金额**。

### 3. 为什么两个数字不同？

你的判断方向正确，但术语可以更准确。

这里不是两个完全不同的业务对象，而是对同一组 Pool 交易事实使用了两种不同的指标定义：

- **Reserve Net Change**：统计储备余额的净增减，考虑正负方向。
- **Pool Gross Volume**：统计执行过的成交活动规模，不用买卖方向相互抵消。

因此，同一天完全可能出现较大的 Trading Volume，但很小的 Reserve Net Change。

### 4. 能否直接得到 User-level Routed Volume？

正确，不能保证。

Pool Swap Fact 记录每个 Pool 的执行事件，不一定能完整说明用户级 Swap Intent。

需要进一步结合 Router Calldata、Execution Trace、Token Transfer 和协议特定的路由证据，才能可靠重建用户级兑换。

你已经掌握本课核心逻辑：

**先确定 Business Question，再确定 Grain、数据来源和计算口径。**

至此，**第 8 课重学版三组理解检查全部通过，达到正式结课标准。**

## 结课判定

Module 13 第 8 课《DEX Data Modeling：从一个真实查询需求推导数据模型》（问题驱动重学版）正式通过并结课。

已能从 Pool A 的真实查询需求逐步推导 Pool Dimension、Pool Swap Fact、State Change History、Daily Snapshot、Token Metadata 与 USD Valuation 的职责及 Grain；能够区分 Reserve Net Change 与 Pool Gross Volume，解释池内执行与用户级路由之间的口径边界。原第 8 课 Canonical Base 及课堂补充备注保留，不覆盖。
