# Module 13 第 7 课｜Router、Multi-hop Swap 与 Aggregator：一笔交易为什么有多个 Swap

## Lesson Contract

所属 Module：Module 13 — DEX（Decentralized Exchange，去中心化交易所）。

**本课核心问题：**

> Why can one user swap be executed through multiple pools, producing multiple Swap Events within a single transaction?

完成本课后，你应该能够：

1. 区分 Router、Pool/Pair 与 Aggregator 的职责。
2. 理解 Single-hop、Multi-hop 和 Split Routing。
3. 区分用户级 Swap Intent 与 Pool-level Swap Execution。
4. 从多个 Swap Event 识别 Token 兑换路径。
5. 理解为什么 `tx_hash` 不能唯一标识一条 Pool-level Swap Fact。
6. 区分 `tx.from`、`Swap.sender`、`Swap.to` 和最终 Trader。
7. 解释如何避免在交易量（Volume）统计中重复计算。
8. 为后续 DEX Data Modeling 建立正确的 Grain 和 Data Lineage。

**本课不展开：** Router 合约源码、复杂最优路径算法、MEV（Maximal Extractable Value，最大可提取价值）策略、跨链桥内部执行、完整 Aggregator 交易重建算法。

---

## 一、为什么用户的一笔交易会产生多个 Swap？

前两课我们学习的模型是：

```text
Trader
   ↓
Pool
   ↓
Swap Event
```

这适合理解单个 Pool 的交易行为。

但实际 DEX 产品面对的问题是：

假设 Alice 持有 USDC，希望换成 UNI。

市场上可能没有一条能够提供最佳报价的 USDC/UNI Pool，但存在：

```text
USDC / WETH Pool

WETH / UNI Pool
```

于是可以这样执行：

```text
Alice
  │
  │ 1,000 USDC
  ▼
USDC / WETH Pool
  │
  │ 0.32 WETH
  ▼
WETH / UNI Pool
  │
  │ 160 UNI
  ▼
Alice
```

以上金额是教学示例，并非实际市场报价。

从 Alice 的角度：

```text
1,000 USDC → 160 UNI
```

这是一次用户级兑换。

但从 Protocol 角度：

```text
Swap 1: USDC → WETH
Swap 2: WETH → UNI
```

这是两次 Pool-level Swap。

**One user swap intent can generate multiple pool swap executions.**

这是本课最重要的起点。

---

## 二、[Protocol 视角] Router 是什么？

Router 可以理解为一个协调执行交易的智能合约。

在典型 Uniswap v2 路径中：

```text
User
  │
  ▼
Router
  │
  ├── Determine specified path
  │
  ├── Transfer input tokens
  │
  ├── Execute Pool A swap
  │
  ├── Execute Pool B swap
  │
  └── Enforce transaction conditions
```

它负责根据用户提交的路径和交易参数协调执行过程。

请区分：

```text
Router
= Execution Coordinator

Pool / Pair
= Liquidity Provider and Swap Executor
```

Router 通常不是提供流动性的主体。

真正用于兑换的资产库存存在 Pool 中；Swap 由对应 Pool 执行，相关 Swap Event 也由 Pool 合约发出。

银行类比：Router 有点像支付系统里的交易编排服务（Transaction Orchestrator），负责协调执行步骤；Pool 则更接近真正持有资产库存并执行兑换的业务单元。

但 Router 与传统 Orchestrator 有一个关键区别：在同一笔成功的以太坊 Transaction 中，多步执行具有原子性（Atomicity）。如果其中一个必要步骤失败并使交易 Revert，那么本次交易的链上状态变更整体回滚。

---

## 三、Single-hop 与 Multi-hop

### Single-hop Swap

Single-hop 是通过一个 Pool 完成兑换。

```text
Alice
  │
  │ USDC
  ▼
USDC / WETH Pool
  │
  │ WETH
  ▼
Alice
```

对应：

```text
1 user intent
1 pool execution
1 Swap Event
```

这是最简单的正常情况。

### Multi-hop Swap

Multi-hop 是经过多个 Pool 完成兑换。

```text
Alice
  │ USDC
  ▼
Pool A: USDC / WETH
  │ WETH
  ▼
Pool B: WETH / UNI
  │ UNI
  ▼
Alice
```

对应：

```text
1 user intent
2 pool executions
2 Swap Events
```

这里的 WETH 称为 Intermediate Token（中间代币）。

它是连接两段流动性路径的资产，并不一定由 Alice 最终持有。

所以不要把第二段 WETH → UNI 自动解释为 Alice 独立发起的另一笔交易意图。

---

## 四、为什么需要经过中间 Token？

最常见的原因是 Liquidity 和 Execution Quality。

假设有三条可能路径：

```text
Path A:
USDC → UNI

Path B:
USDC → WETH → UNI

Path C:
USDC → USDT → UNI
```

即使 Path A 更短，也不一定产生最佳实际输出。

因为交易执行结果还受以下因素影响：

- 每个 Pool 的 Liquidity Depth；
- Price Impact；
- 各 Pool 的 Trading Fee；
- 中间资产的兑换价格；
- Gas Cost；
- 执行时的市场状态。

因此：

**Shortest Path ≠ Best Execution Path.**

不过，多经过一个 Pool 通常意味着多一次 Pool 级兑换和对应 Fee。增加 Hop 不一定提高最终收益。

---

## 五、[Event 视角] Multi-hop 会产生什么 Logs？

仍然使用：

```text
USDC → WETH → UNI
```

假设同一笔 Ethereum Transaction 内：

```text
Transaction Hash = 0xABC
```

我们观察到两个 Pool 的 Swap Event：

```text
log_index = 18
Pool A
Swap:
  1,000 USDC in
  0.32 WETH out

log_index = 25
Pool B
Swap:
  0.32 WETH in
  160 UNI out
```

对链上 Indexer 来说，存在两条可识别的 Pool-level Swap Fact。

```text
(chain_id, 0xABC, 18)

(chain_id, 0xABC, 25)
```

两条记录的 `tx_hash` 相同，但 `log_index` 不同，Pool 也不同。

因此，上一课建立的 Unique Key：

```text
chain_id
+
tx_hash
+
log_index
```

仍然适用于此处的 Event-level Grain。

注意：实际 Ethereum Receipt 中还可能穿插多个 `Transfer`、`Sync` 等其他 Event，Swap 的两个 `log_index` 不一定相邻。

---

## 六、[Data Engineer 视角] 为什么“同一 tx_hash”不代表“同一 Swap”？

因为这里存在两个不同的分析层级：

```text
Transaction level
        │
        ├── Swap Event A
        │
        └── Swap Event B
```

Transaction 是链上执行容器。

Swap Event 则记录其中某个 Pool 的一次具体交易执行。

所以：

```text
1 Transaction
≠
1 Pool Swap
```

同时还需要特别注意：

```text
1 Transaction
≠
necessarily 1 User Swap Intent
```

一次 Transaction 也可能包含：

```text
Swap A
Swap B
Liquidity operation
Transfer
Other contract calls
```

甚至可能通过某个合约执行多次彼此独立的兑换。

因此，仅使用 `tx_hash` 把所有 Swap 强行合并成一个用户级 Swap，也不一定正确。

**Transaction Boundary is not always Business Intent Boundary.**

---

## 七、Router Path 到底表示什么？

以 Uniswap v2 常见的 Token Path 为例：

```text
[USDC, WETH, UNI]
```

它表达预期的兑换顺序。

```text
USDC → WETH → UNI
```

对于这个例子：

```text
Token Path Length = 3

Hop Count = 2
```

一般来说，简单线性路径中：

```text
Hop Count = Token Path Length - 1
```

但这个公式只适用于此处的线性路径模型，不应直接套到复杂 Split Routing 或任意合约调用。

从数据工程角度，Path 是业务解释的关键信息，但它通常不是简单地存在于每个 Pool 的 Swap Event 里。

你必须通过 Router Call Data、Execution Trace、Token Flow 和 Event 顺序等证据，才能可靠重建更高层的 Route。

---

## 八、为什么单纯看 Swap Event 还不够？

假设 Indexer 得到：

```text
Pool A:
USDC → WETH

Pool B:
WETH → UNI
```

仅凭这两条记录，你可以确认：

```text
Two pool executions happened
```

但还不能无条件确认：

```text
Alice intentionally requested
USDC → UNI
```

为什么？

因为：

1. 两个 Swap 可能属于一次线性 Multi-hop。
2. 也可能来自两个独立交易意图。
3. 也可能是 Aggregator 构造的复杂 Route 的一部分。
4. 可能还存在其他 Pool Event 和 Token Flow。

所以必须区分：

```text
Directly observed:
Pool-level Swap Events

Derived / attributed:
User-level Swap Route
```

这与你过去学过的：

```text
Raw Facts
→ Business Interpretation
```

完全一致。

---

## 九、Router.sender 和 Trader 是同一回事吗？

不一定。

在 v2 中：

```text
Swap.sender
```

通常表示直接调用 Pair `swap()` 的地址。

典型 Router 路径下：

```text
Swap.sender = Router
```

而：

```text
tx.from = Alice
```

但这也不能推导出：

```text
tx.from always equals economic trader
```

例如，智能钱包、Relayer、Account Abstraction 或其他代理合约，都可能改变直接调用者与经济受益人的对应关系。

至少要区分：

```text
transaction_from
swap_sender
swap_recipient
user_intent_owner
```

其中 `user_intent_owner` 是业务归因结果，不应从一个 Event 字段直接复制而来。

---

## 十、什么是 Aggregator？

Aggregator（聚合器）是一类专门寻找和执行兑换路径的交易服务或协议。

它可能利用多个 DEX 或多个 Pool，尝试改善用户实际收到的资产数量。

例如：

```text
Alice:
Swap 10,000 USDC to ETH
```

某条路径在交易量较大时，可能产生明显 Price Impact。

Aggregator 可以尝试：

```text
Route 1:
USDC → WETH
via Pool A

Route 2:
USDC → WETH
via Pool B
```

然后分别执行部分订单。

这就是 Split Routing（拆分路由）的基本思想。

但 Aggregator 不保证每次一定给出最佳成交结果，实际效果受到报价时效、Gas、Fees、流动性和执行条件约束。

---

## 十一、Multi-hop 与 Split Routing 有什么不同？

这两个概念很容易混淆。

**Multi-hop** 是串行兑换：

```text
USDC
  ↓
WETH
  ↓
UNI
```

前一段输出通常构成后一段输入。

**Split Routing** 是把一个输入金额分配给多条执行路径：

```text
            ┌── 600 USDC → Pool A → ETH
1,000 USDC ┤
            └── 400 USDC → Pool B → ETH
```

因此：

```text
Multi-hop
= Sequential / chained execution

Split Routing
= Parallel logical routes / order allocation
```

注意“Parallel logical routes”表示交易计划中多条分支，并不意味着 Ethereum 在同一个 Transaction 中并行执行这些合约调用。EVM 执行仍然具有确定的顺序。

现实中还可能出现两者结合：

```text
                 ┌── USDC → WETH
Input USDC ──────┤
                 └── USDC → USDT → WETH
```

这让 Route Reconstruction 更复杂。

---

## 十二、一个具体的 Aggregator 数据例子

假设 Alice 请求：

```text
Swap 10,000 USDC → WETH
```

Aggregator 把订单分为两段：

```text
Route A:
6,000 USDC → 1.90 WETH

Route B:
4,000 USDC → 1.25 WETH
```

最终用户结果：

```text
Input:
10,000 USDC

Output:
3.15 WETH
```

从 Pool-level 数据看：

```text
Swap Fact A
Swap Fact B
```

从用户级别看：

```text
1 logical user swap request
2 underlying pool executions
```

注意这里的“1 logical user swap request”是已经知道 Aggregator 执行计划的前提。不能仅因两条 Swap 的 `tx_hash` 相同，就自动认定它们属于一个用户请求。

---

## 十三、Volume 为什么容易重复计算？

这是本课重要的数据工程问题。

回到：

```text
USDC → WETH → UNI
```

两段 Pool Swap：

```text
Hop 1:
1,000 USDC → 0.32 WETH

Hop 2:
0.32 WETH → 160 UNI
```

如果每一 Hop 的 USD Trading Volume 近似为 1,000 USD，那么：

```text
Pool-level gross traded volume
≈ 2,000 USD
```

但 Alice 用户级别的一次兑换：

```text
User-level input notional
≈ 1,000 USD
```

注意两者都可能是合法指标，只是**口径不同**。

不能直接说其中之一是错误数据。

真正的问题在于：

> Which business grain does this volume metric represent?

---

## 十四、两种 Volume 口径

**Pool-level Volume：**

用于回答：

```text
How much trading activity did each pool execute?
```

Multi-hop 每一段 Swap 都有自己的 Pool-level Volume。

因此，汇总各 Pool 时，多段成交金额都可能进入统计。

**User-level Routed Volume：**

用于回答：

```text
How much value did users intend to exchange?
```

通常关注用户最初投入或最终获得资产的一侧经济金额，并按预先定义的计价方式估值。

对于线性两跳交易，如果直接将两个 Pool 的 Volume 相加作为用户交易量，容易造成 Double Counting。

因此你在 Data Warehouse 中必须明确：

```text
pool_gross_volume_usd
```

与：

```text
user_routed_volume_usd
```

是不同指标。

---

## 十五、银行数据中台类比

假设银行客户发起一笔跨币种汇款：

```text
Customer:
CNY → EUR
```

实际系统可能执行：

```text
CNY → USD
USD → EUR
```

于是有两个内部 FX Trade Records。

但客户只发起了一次：

```text
Currency Conversion Request
```

如果你要统计银行内部外汇交易 Desk 的成交活动，可以统计两段交易。

如果统计客户主动发起的换汇需求金额，就不能把两段简单相加。

DEX 完全存在类似的问题：

```text
Customer Request
≠
Underlying Execution Legs
```

这个类比能够帮助你理解为什么要设计两种不同事实 Grain。

---

## 十六、[Data Modeling 视角] 至少区分两个 Fact Grain

在本课阶段，建议先区分：

```text
fact_dex_pool_swap
```

以及：

```text
fact_dex_routed_swap
```

第一张：

```text
Grain:
one successful Pool Swap Event
```

典型 Key：

```text
chain_id + tx_hash + log_index
```

第二张：

```text
Grain:
one reliably reconstructed logical swap intent
```

其 Identity 不能一律使用：

```text
chain_id + tx_hash
```

因为一个 Transaction 可能包含多个逻辑意图。

它需要结合具体协议和路由机制，形成可靠的：

```text
route_id / intent_id
```

以及相应的 Transaction 上下文。

这正是后面正式做 DEX Data Modeling 时要进一步处理的内容。

---

## 十七、两种 Fact 应该怎样建立关系？

假设 Alice 一次 Multi-hop：

```text
USDC → WETH → UNI
```

可以建模为：

```text
Routed Swap
route_id = R001
     │
     ├── Pool Swap S001
     │     USDC → WETH
     │
     └── Pool Swap S002
           WETH → UNI
```

这代表：

```text
One routed swap
→ Many pool swaps
```

但对于复杂交易，现实中可能需要独立关联表：

```text
bridge_route_pool_swap
```

用来保存：

```text
route_id
pool_swap_id
hop_index
branch_index
```

这种设计可支持线性路径，也可以扩展到拆单、多分支路由。

这里的 `route_id` 是数据模型中的可靠业务标识，不意味着链上天然存在同名字段。

---

## 十八、Trace 为什么在这里变得重要？

前面课程已经学过：

```text
Transaction
Trace
Log
Transfer
```

它们视角不同。

Log 告诉你：

```text
Which Pool emitted which Swap Event?
```

Trace 帮助你理解：

```text
Which contract called which other contract?
```

而 Transfer 告诉你：

```text
Where did the tokens move?
```

所以复杂 Route Reconstruction 可能需要：

```text
Transaction Calldata
+
Traces
+
Swap Events
+
Token Transfers
```

但要注意，Trace 并不总是直接提供一个清楚的“用户最终交易意图”。它仍然是执行证据，需要业务解释。

---

## 十九、为什么不能只靠 Token Transfer 串起来？

假设两个 Pool 之间存在：

```text
WETH Transfer
```

你可能推断：

```text
Pool A output WETH
↓
Pool B input WETH
```

这个推断可能合理，但还需要检查：

```text
Token address
Amount consistency
Execution order
Contract identity
Transaction context
```

并防止误把同一个 Transaction 中不相关的 Transfer 串联起来。

尤其在 Aggregator 或复杂 Router 场景，Token 可能经过中间合约、余额净额结算或其他操作。

所以：

**Token Flow is supporting evidence, not a universally sufficient definition of a route.**

---

## 二十、如何处理跨 DEX Aggregator？

假设：

```text
Aggregator
   │
   ├── Uniswap v2 Pool
   ├── Uniswap v3 Pool
   └── Another DEX Pool
```

不同协议 Swap Event 格式可能不同。

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

所以 Raw Decode 必须遵守协议特有的 ABI（Application Binary Interface，应用程序二进制接口）。

但在 Normalized Layer 可以统一：

```text
chain_id
tx_hash
log_index

protocol
pool_address

token_in
token_out
amount_in
amount_out
```

然后才能进行跨协议 Route Reconstruction。

---

## 二十一、Data Lineage 应该如何表达？

建议建立下列链路：

```text
Raw Transaction / Receipt / Logs / Traces
                    │
                    ▼
        Protocol-specific Decode
                    │
                    ▼
        Normalized Pool Swap Fact
                    │
                    ▼
      Route Reconstruction / Attribution
                    │
                    ▼
           Routed Swap Fact
                    │
                    ▼
              Analytics
```

这里需要特别注意：

```text
Normalized Pool Swap
```

属于对已有 Protocol Evidence 的直接结构化和标准化。

而：

```text
Routed Swap Fact
```

包含更强的业务归因。

因此应保留 Reconstruction Method、Evidence 和 Confidence / Validation Status，避免把尚未可靠归因的业务假设伪装成链上直接事实。

---

## 二十二、一个 Data Quality 问题

假设系统还原了：

```text
USDC → WETH → UNI
```

两段输出输入是：

```text
Hop 1:
0.320 WETH out

Hop 2:
0.300 WETH in
```

能否直接判定 Pipeline 错误？

**不能。**

这可能是：

- 部分 WETH 被用于其他业务步骤；
- 部分输出未进入第二个 Pool；
- 路由涉及手续费或复杂资金流；
- 两个 Swap 根本不属于同一条线性 Route；
- 或者确实存在解析、关联错误。

因此要继续检查 Token Transfer、Trace 和 Router Execution。

对于简单的标准线性 Multi-hop，金额衔接通常应能被验证；但复杂路由中不能机械地要求每两个事件数量永远完全相等。

---

## 二十三、Reorg 与幂等在 Routed Swap 中有什么影响？

你已经完成 Indexer、ETL 和 Data Quality 模块，所以这里只做应用。

如果：

```text
Pool Swap S001
Pool Swap S002
```

来自某条后来被 Reorg 移除的区块分支，那么基于它们派生的：

```text
Routed Swap R001
```

也不能继续作为 Canonical 事实保留。

因此修复链路应是：

```text
Invalidate / replay pool swap facts
            ↓
Rebuild affected route attribution
            ↓
Correct routed swap analytics
```

这再次说明：

```text
Derived Data
depends on
Underlying Canonical Evidence
```

---

## 二十四、本课的核心工程设计判断

假设产品经理提出：

> 我想看用户交易次数、DEX 总交易量、热门兑换路径，以及每个 Pool 的实际交易次数。

如果只有一张：

```text
fact_dex_pool_swap
```

我们可以较好回答：

```text
Pool execution counts
Pool-level volume
```

但未必能准确回答：

```text
User intent counts
Popular routed paths
```

因为后两者需要额外的 Route Reconstruction。

如果直接：

```sql
SELECT COUNT(DISTINCT tx_hash)
FROM fact_dex_pool_swap;
```

并把它命名为：

```text
User Swap Count
```

就可能产生错误口径。

这条 SQL 统计的是包含已记录 Pool Swap 的不同 Transaction 数量，不必然等于用户逻辑兑换次数。

---

## 二十五、本课核心结论

> Router coordinates swap execution, while Pool contracts perform the actual swaps and emit pool-level events.

> Multi-hop means sequential execution through multiple pools; Split Routing means allocating one logical order across multiple paths.

> One Transaction can contain multiple Pool Swap Events, and one Transaction does not necessarily represent exactly one user intent.

> Pool-level Swap Fact and Routed Swap Fact have different grains.

> Summing pool-level volume across hops may be appropriate for pool activity but can double-count user-level routed notional.

> Route Reconstruction requires protocol-aware evidence, such as calldata, traces, Swap Events and Token Transfers.

> Reliable data modeling must distinguish directly observed protocol facts from reconstructed business intentions.

---

# 理解检查

## 问题一：Multi-hop Semantics

Alice 发起：

```text
1,000 USDC → UNI
```

实际执行：

```text
Pool A:
1,000 USDC → 0.32 WETH

Pool B:
0.32 WETH → 160 UNI
```

请回答：

1. 从 Alice 的业务视角，这是几次兑换请求？
2. 从 Pool-level 视角，应记录几条 Swap Fact？
3. WETH 在这个场景中扮演什么角色？
4. 如果两个 Swap 的 `tx_hash` 相同，能否只保存一条 Pool Swap Fact？为什么？

## 问题二：Volume 与 Aggregator

假设两段 Multi-hop Swap，各自的 USD Volume 都约为 1,000 美元。

请回答：

1. Pool-level Gross Volume 合计约是多少？
2. 用户级 Routed Volume 约是多少？
3. 为什么这两个数字不相同，却未必有一个是错误数据？
4. Multi-hop 与 Split Routing 的核心差别是什么？

## 问题三：Data Modeling 与 Data Lineage

某 Ethereum Transaction 同时出现：

```text
Swap A: USDC → WETH
Swap B: WETH → UNI
```

请回答：

1. 能否只凭相同的 `tx_hash`，断定它们构成一次完整的用户级 Multi-hop？
2. 还需要哪些链上证据来验证这个 Route？
3. 为什么 `Swap.sender` 与 `tx.from` 不一定代表同一个业务角色？
4. `fact_dex_pool_swap` 与 `fact_dex_routed_swap` 分别应该采用什么 Grain？
5. 如果一次 Reorg 删除了原来的 Pool Swap Events，对上层 Routed Swap 应该做什么？

本课先停在理解检查。你可以逐题回答，我会按课程规则逐轮校准并同步。

## 用户回答（理解检查｜问题一）

1. 从 Alice 的业务视角，这是一次兑换请求
2. 从 Pool Level 视角，应该记录两条 Swap Fact
3. 这里的 WETH 称为 Intermediate Token（中间代币）。
4. 不能，因为从 Pool 的角度来看，这是有两个 Pool 发生的事件，一条 Swap Fact 体现不了这个情况。

## 老师判断 / 校准（问题一）

**判定：通过，4 个小题全部正确。**

1. **User-level Swap Intent：1 次。** Alice 只发起了一次 `USDC → UNI` 的兑换请求。
2. **Pool-level Swap Fact：2 条。** Pool A 执行 `USDC → WETH`，Pool B 执行 `WETH → UNI`。
3. **WETH 是 Intermediate Token（中间代币）。** 它连接两段兑换路径，并非 Alice 最终希望获得的资产。
4. **不能合并成一条 Pool Swap Fact。** 两个 Pool 分别执行了兑换，应当保留两条独立的 Swap Event 记录。

### 补充一个工程细节

你在第 4 点强调“两个 Pool 发生了事件”，这个判断正确。但还要进一步理解：

**即使同一个 Pool 在一笔 Transaction 中连续执行两次 Swap，也必须记录两条 Pool-level Swap Fact。**

因为本课的 Grain 是：

```text
One Pool Swap Event
= One Pool-level Swap Fact
```

对应 Unique Key：

```text
chain_id + tx_hash + log_index
```

因此，决定事实表记录数量的是 **Swap Event 的数量**，而不是 Pool 的数量，也不是 Transaction 的数量。

这也是后续区分 `Pool-level Volume` 与 `User-level Routed Volume` 的基础。
