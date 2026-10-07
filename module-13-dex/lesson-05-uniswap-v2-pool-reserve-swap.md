# Module 13 第 5 课｜Uniswap v2：Pool / Reserve / Swap 的完整数据语义

## Lesson Contract

所属 Module：Module 13 — DEX（Decentralized Exchange，去中心化交易所）

**本课核心问题：**

> Given raw Uniswap v2 events, how can a Blockchain Data Engineer reconstruct pool state and interpret a swap correctly?

学完本课，你应当能够解释：

1. Factory、Pair、Router 分别是什么，Pool 到底对应哪个链上对象。
2. `token0`、`token1` 与 `reserve0`、`reserve1` 的对应关系。
3. `Swap`、`Sync`、`Mint`、`Burn` Event 分别表达什么。
4. 为什么一笔 Swap 不能仅通过 ERC-20 `Transfer` 判断。
5. 如何识别一笔交易的输入 Token、输出 Token、实际数量与执行价格。
6. 为什么 Pool State 与 Swap Fact 应当分开建模。
7. 为什么同一个 Transaction 内多个 Log 的执行顺序很重要。

本课不展开 Uniswap v2 合约源码逐行分析、复杂 Router 路径以及 Uniswap v3 的 Tick / Position 模型。下一阶段才会系统讨论 Multi-hop Swap。

---

## 一、从抽象 AMM 进入真实 Protocol

前四课我们学习了：

```text
Liquidity Provider
        ↓
Liquidity Pool
        ↑
      Trader

Pool Reserves
        ↓
    x * y = k
        ↓
   Swap Execution
```

但如果你真的开发一个 Ethereum DEX Indexer，会发现链上并没有直接提供一个名为：

```text
dex_swap_business_fact
```

的标准业务对象。

你拿到的是：

```text
Block
  └── Transaction
        └── Transaction Receipt
              └── Logs
```

这些 Logs 由不同 Smart Contract 发出。

因此，本课真正要解决的问题是：

**如何把 Protocol Evidence 转换为可信的 DEX Business Fact？**

这正是 Blockchain Data Engineer 的工作。

---

## 二、[Protocol 视角] Uniswap v2 的三个核心合约角色

Uniswap v2 的典型交易架构包含三个角色。

### 1. Factory

Factory 负责创建和登记交易对。

例如：

```text
UniswapV2Factory
    │
    ├── ETH / USDC Pair
    ├── WBTC / USDC Pair
    └── UNI / WETH Pair
```

需要注意：以太坊原生 ETH 并非 ERC-20 Token。在 Uniswap v2 的 Pair 中，通常实际使用 **WETH（Wrapped Ether，封装的以太币）**。

因此，我们口头常说的 ETH / USDC Pool，在合约层可能实际是：

```text
WETH / USDC Pair
```

Factory 发出 `PairCreated` Event，表示新的 Pair 已被创建。

### 2. Pair

Pair 是本课最重要的对象。

**A Pair contract is the actual liquidity pool.**

它负责维护：

```text
token0
token1

reserve0
reserve1

LP token supply
```

并执行：

```text
swap()
mint()
burn()
sync()
```

这里的 `mint()` 与 `burn()` 是 LP 份额相关操作，不是 ERC-20 交易 Token 的普通 Mint / Burn。

### 3. Router

Router 提供方便用户调用的交易入口，例如完成 Token 转移、路径处理以及最小输出检查。

但要区分：

```text
Router
= user-facing execution coordination

Pair
= pool state and swap execution
```

**一笔交易由 Router 发起，并不意味着 Swap Event 由 Router 发出。**

Uniswap v2 的核心 `Swap` Event 由 Pair 合约发出。

这也是后续建立数据模型时识别 `pool_address` 的关键。

---

## 三、[Data Engineer 视角] Pool 的身份是什么？

假设存在一个 WETH / USDC Pair。

它有：

```text
chain_id
factory_address
pair_address
token0_address
token1_address
```

对于 Uniswap v2，常用的 Pool Identifier 是：

```text
chain_id + pair_address
```

原因是：

```text
Pair Contract Address
```

标识实际执行交易和维护 Reserve 的合约。

不过，生产环境中应该保留 `factory_address`、协议版本等信息，并验证 Pair 来自可信 Factory，不能只根据“合约里恰好有两个 Token”来认定它属于 Uniswap v2。

一个 Pool 维度表可以设计为：

```sql
CREATE TABLE dim_dex_pool (
    chain_id         BIGINT,
    pool_address     VARCHAR(42),
    factory_address  VARCHAR(42),
    protocol_name    VARCHAR(50),
    protocol_version VARCHAR(20),
    token0_address   VARCHAR(42),
    token1_address   VARCHAR(42),

    PRIMARY KEY (chain_id, pool_address)
);
```

这张表的 Grain 是：

> One pool per chain.

注意，这里是 Pool 的身份与相对稳定属性，不是实时 Reserve Snapshot。

---

## 四、token0 和 token1 谁来决定？

这是实际解析 Uniswap v2 Event 时必须掌握的概念。

Pair 有两个 Token：

```text
token0
token1
```

它们不是按以下规则排列：

```text
token0 = 用户输入的 Token
token1 = 用户输出的 Token
```

也不是：

```text
token0 = ETH
token1 = USDC
```

在 Uniswap v2 中，两个 Token 按 **Token Contract Address 的排序规则**确定 `token0` 与 `token1`。

因此：

```text
token0
token1
```

是 **Pair-level fixed token ordering**。

而：

```text
token_in
token_out
```

是 **Swap-level trade direction**。

二者不是同一个概念。

例如同一个 Pair：

```text
token0 = USDC
token1 = WETH
```

Alice 可以：

```text
USDC → WETH
```

Bob 也可以：

```text
WETH → USDC
```

Pair 的 `token0`、`token1` 顺序始终不变，但交易方向可以改变。

---

## 五、Reserve 为什么必须带 Token 语义？

Uniswap v2 Pair 内存在：

```text
reserve0
reserve1
```

它们对应：

```text
reserve0 → token0
reserve1 → token1
```

如果你只存：

```text
reserve0 = 300000
reserve1 = 100
```

而没有记录 Token Mapping，那么无法正确解释其经济意义。

比如我们假设：

```text
token0 = USDC
token1 = WETH

reserve0 = 300,000 USDC
reserve1 = 100 WETH
```

那么：

```text
WETH Price
= reserve0 / reserve1
= 3,000 USDC per WETH
```

但是，如果 Token 顺序反过来，这个公式也必须相应调整。

还有一个工程细节：

**Raw Token Amount 不等于 Human-readable Token Amount。**

真实链上 Reserve 采用 Token 最小单位，需要结合各 Token 的 `decimals` 转换。

例如：

```text
USDC decimals = 6
WETH decimals = 18
```

不要直接用两个原始整数相除作为报价。

---

## 六、Uniswap v2 中四个重要的 Event

本课重点认识：

```text
Swap
Sync
Mint
Burn
```

它们都可以由 Pair Contract 发出，但业务含义完全不同。

| Event | 核心含义 | 数据工程用途 |
|---|---|---|
| `Swap` | Pair 执行了兑换 | 构建 Swap Fact |
| `Sync` | Pair 更新了记录的 Reserves | 维护 Reserve State |
| `Mint` | 新增流动性、铸造 LP 份额 | 识别 Add Liquidity |
| `Burn` | 移除流动性、销毁 LP 份额 | 识别 Remove Liquidity |

另外，Factory 的：

```text
PairCreated
```

用于发现新 Pool。

注意 Event 粒度与业务粒度不同：一次用户交易可以涉及多个 Contract Event，不能假设：

```text
1 Transaction = 1 Event
```

---

## 七、[Protocol 视角] Swap Event 有哪些字段？

Uniswap v2 的 Swap Event 结构可以简化为：

```text
Swap(
    sender,
    amount0In,
    amount1In,
    amount0Out,
    amount1Out,
    to
)
```

这六个字段非常重要。

其中：

```text
amount0In
amount1In
```

描述两个 Token 向 Pair 的输入数量。

而：

```text
amount0Out
amount1Out
```

描述两个 Token 从 Pair 输出的数量。

注意是 **相对 Pair 的输入与输出**，不是相对用户钱包。

举例：

```text
token0 = USDC
token1 = WETH
```

Alice 用 USDC 买 WETH。

Swap Event：

```text
amount0In  = 3,000 USDC
amount1In  = 0

amount0Out = 0
amount1Out = 0.987 WETH
```

可以解释成：

```text
Input  Token = token0 = USDC
Output Token = token1 = WETH
```

于是：

```text
Execution Price
= 3,000 / 0.987
≈ 3,039.51 USDC / WETH
```

这个例子中的 `0.987 WETH` 作为示意数值使用，实际输出由当时 Reserve、Fee 和交易条件决定。

---

## 八、Swap Event 里的 sender 是最终 Trader 吗？

未必。

这是很容易产生错误 Wallet Analytics 的地方。

假设：

```text
Alice
   ↓
Router
   ↓
Pair
```

Pair 发出：

```text
Swap.sender = Router address
```

这种情况下，`sender` 是执行 Pair Swap 调用的地址。

它未必是：

```text
end-user wallet
```

同理：

```text
Swap.to
```

表示这一次 Pair 发送输出 Token 的接收地址，也未必是最终用户。

所以不要直接认为：

```text
Swap.sender = trader_wallet
```

在 Data Modeling 中，至少区分：

```text
swap_sender
swap_recipient
transaction_from
trader_wallet
```

其中 `trader_wallet` 往往属于需要进一步归因的业务字段。

---

## 九、Sync Event 是做什么的？

`Sync` 的结构非常简单：

```text
Sync(
    reserve0,
    reserve1
)
```

它表达的是：

> Pair updated its recorded reserve state to these values.

例如：

```text
Sync(
    reserve0 = 303000 USDC,
    reserve1 = 99.013 WETH
)
```

表示更新后 Pool 记录的 Reserves。

它不是：

```text
Swap Event
```

也不是：

```text
Token Transfer Event
```

而是：

```text
Reserve State Update
```

在 Uniswap v2 的通常 Swap 执行路径中，Pair 会先更新储备并发出 `Sync`，然后发出 `Swap`。

因此，同一笔 Transaction 里有可能看到：

```text
Transfer
Transfer
Sync
Swap
```

这里是示意，真实 Transfer 数量和顺序可能随执行路径变化。

**不能因为 Sync 与 Swap 相邻，就把它们合并成一条 Event。**

它们是同一次业务执行过程里的不同证据。

---

## 十、为什么不能把 Sync 直接当成 Swap？

因为：

```text
Sync
```

还可能与非 Swap 操作有关。

例如：

```text
Add Liquidity
Remove Liquidity
```

也会涉及 Reserve 更新。

此外，合约还具有显式 `sync()` 操作，用来使记录的 Reserve 与当前 Token Balance 对齐。

因此：

```text
Sync happened
```

不等于：

```text
A trade happened
```

判断是否真正发生 Swap，最直接的协议证据仍然是 Pair 的有效 `Swap` Event，并结合成功交易及可信合约来源。

---

## 十一、Mint / Burn 为什么不是 Token Transfer 的同义词？

上一课我们已经学过：

```text
Transfer
≠ Add Liquidity
```

现在可以精确到 Uniswap v2 的 Event。

假设 Alice 增加流动性：

```text
10 WETH
30,000 USDC
```

可能涉及：

```text
WETH Transfer → Pair
USDC Transfer → Pair
LP Token Mint
Pair emits Mint
Pair emits Sync
```

其中：

```text
Mint Event
```

表示添加流动性的协议操作。

它不应和 ERC-20 `Transfer` 的事件签名及语义混为一谈。

同理，当 Alice 移除流动性：

```text
LP Token Burn
Pair emits Burn
Underlying Token Transfer
Pair emits Sync
```

这些信息组合起来，才能完整还原 LP 生命周期。

---

## 十二、[Data Engineer 视角] 处理一笔 Swap 的完整链路

现在把此前学习的 Indexer / Decoder / ETL 联系起来。

```text
Ethereum Block
      ↓
Transaction Receipt
      ↓
Logs
      ↓
Identify Pair Contract
      ↓
Decode Swap Event
      ↓
Resolve token0 / token1
      ↓
Normalize raw amounts
      ↓
Determine trade direction
      ↓
Build DEX Swap Fact
```

其中每一步都有独立责任。

例如：

```text
Decoder
```

负责把 Event ABI（Application Binary Interface，应用程序二进制接口）解析成结构化字段。

而：

```text
Semantic Transformer
```

负责回答：

```text
What token was actually sold?
What token was actually bought?
Which pool executed this swap?
What price did this execution achieve?
```

这就是 **Event Decoding 与 Business Interpretation 的区别**。

---

## 十三、Swap Fact 的 Grain 应当是什么？

本课先采用 **Uniswap v2 Pair-level Swap** 作为事实粒度。

```text
One successful Pair Swap Event
= One Pair-level Swap Fact
```

对应唯一键可以是：

```text
chain_id
+
tx_hash
+
log_index
```

更明确地说：

```sql
CREATE TABLE fact_dex_swap_v2 (
    chain_id         BIGINT,
    tx_hash          VARCHAR(66),
    log_index        INTEGER,

    pool_address     VARCHAR(42),
    block_number     BIGINT,

    token0_address   VARCHAR(42),
    token1_address   VARCHAR(42),

    amount0_in_raw   NUMERIC,
    amount1_in_raw   NUMERIC,
    amount0_out_raw  NUMERIC,
    amount1_out_raw  NUMERIC,

    swap_sender      VARCHAR(42),
    swap_recipient   VARCHAR(42),

    PRIMARY KEY (chain_id, tx_hash, log_index)
);
```

这是教学用的基础 Schema，而不是完整生产模型。

生产环境还需考虑：

```text
block_hash
transaction_index
canonical status
reorg handling
token decimal metadata
decoded source provenance
```

这些已经属于你此前学过的 Indexer、ETL 与数据质量知识。

---

## 十四、为什么不能只用 tx_hash 作为 Swap 唯一键？

因为同一笔 Ethereum Transaction 可以产生多个 Swap Event。

例如：

```text
Transaction X
   ├── Pair A emits Swap
   └── Pair B emits Swap
```

那么：

```text
tx_hash = same
```

但：

```text
log_index = different
```

所以：

```text
chain_id + tx_hash
```

不足以唯一确定 Pair-level Swap。

这与你之前学习 ERC-20 Transfer Fact 时的原则是一致的：

> Fact Unique Key must match the Fact Grain.

---

## 十五、[State 视角] Swap Fact 与 Pool State 的关系

假设某 Pool 初始：

```text
reserve0 = 300,000 USDC
reserve1 = 100 WETH
```

后来发生 Swap：

```text
amount0In  = 3,000 USDC
amount1Out = 0.987 WETH
```

那么本次交易后，Pool 记录的 Reserves 可能变成：

```text
reserve0 = 303,000 USDC
reserve1 = 99.013 WETH
```

注意：

```text
Swap Fact
```

表达发生了什么交易。

而：

```text
Pool Reserve State
```

表达某一执行时点的 Pool 资产库存状态。

二者属于不同类型的数据：

```text
Swap = Event / Fact

Reserve = State / Snapshot
```

如果只有最新 Reserve：

```text
reserve0 = 303000
reserve1 = 99.013
```

你无法仅靠它知道过去每笔 Swap 的金额和用户。

反过来，只有一条 Swap Event，也不能直接推断任意历史时点的完整 Pool State。

---

## 十六、Reserve Snapshot 应当如何建模？

可以单独设计：

```sql
CREATE TABLE fact_dex_pool_reserve_state (
    chain_id        BIGINT,
    pool_address    VARCHAR(42),
    tx_hash         VARCHAR(66),
    log_index       INTEGER,
    block_number    BIGINT,

    reserve0_raw    NUMERIC,
    reserve1_raw    NUMERIC,

    PRIMARY KEY (chain_id, tx_hash, log_index)
);
```

这里选择的 Grain 是：

> One Pair Sync Event per chain.

所以虽然名字中使用了 `state`，它实际记录的是 **Reserve State Change 的历史快照**。

对于日终或区块末尾状态，也可以派生另一张表：

```text
Grain:
one pool per block
```

但两者的语义必须分开。

例如，同一 Block 里 Pool 可能发生多次 Swap，因此：

```text
Pool State after Swap A
```

不等于：

```text
Pool State at end of Block
```

---

## 十七、为什么 log_index 在这里特别重要？

假设：

```text
Block 1000
    Transaction A
        log_index = 10  → Sync
        log_index = 11  → Swap

    Transaction B
        log_index = 20  → Sync
        log_index = 21  → Swap
```

对于同一个 Pool：

```text
State after Transaction A
```

和：

```text
State after Transaction B
```

是不一样的。

要正确计算：

```text
pool_price_before
pool_price_after
```

不能只按照 `block_number` 排序。

至少需要保留链上执行顺序，例如：

```text
block_number
transaction_index
log_index
```

尤其不要直接把一个 Block 末尾的 Reserve 值套到 Block 内所有 Swap 上。

这会导致 **Historical State Attribution Error**。

---

## 十八、一个重要的边界：Swap Event 和 Sync Event 的对应关系

对于标准 Uniswap v2 Pair 的普通成功 Swap，常见执行顺序是：

```text
Pair updates reserves
↓
Pair emits Sync
↓
Pair emits Swap
```

因此，分析同一次 Swap 的执行后 Reserve 时，可以利用对应的 `Sync`。

但不能使用下面这种过度简化的逻辑：

```text
Every Sync must belong to a Swap
```

原因是 Mint、Burn 以及显式 Sync 等操作同样可能改变 Reserve State。

正确的数据工程原则是：

> Use protocol execution semantics and ordered logs, not merely event proximity, to attribute reserve changes.

---

## 十九、[银行数据中台类比] 流水与账户余额

把这个区别放到你熟悉的银行数据系统中。

银行转账流水：

```text
transaction_id
from_account
to_account
amount
timestamp
```

表示业务操作。

账户余额快照：

```text
account_id
balance
snapshot_time
```

表示状态。

那么：

```text
Bank Transfer Fact
≈ DEX Swap Fact
```

而：

```text
Account Balance Snapshot
≈ Pool Reserve State
```

当然，银行账户余额与 AMM Reserve 的经济机制不同，但对数据建模来说：

```text
Business Event
vs
State Snapshot
```

是高度相似的。

如果你把一条转账流水和账户最新余额混为一谈，就会出现数据时间口径错误。

DEX 也是如此。

---

## 二十、实际 Indexer 的三个数据对象

经过本课，可以把 Uniswap v2 数据初步整理为：

```text
dim_dex_pool
    │
    ├── fact_dex_swap_v2
    │       Grain:
    │       one Pair Swap Event
    │
    └── fact_dex_pool_reserve_state
            Grain:
            one Pair Sync Event
```

加上 LP 相关：

```text
fact_dex_liquidity_event
```

用于表示：

```text
Add Liquidity
Remove Liquidity
```

这里先不深入设计 LP Position Snapshot，因为那是上一课已经介绍的独立 Grain。

我们只需建立三个清晰的语义边界：

```text
Pool Identity
Swap Business Fact
Reserve State History
```

---

## 二十一、如何验证还原出来的 Swap Fact？

假设 Decoder 输出：

```text
pool_address

token0 = USDC
token1 = WETH

amount0In  = 3,000
amount1In  = 0

amount0Out = 0
amount1Out = 0.987
```

我们可以做几层 Validation。

**第一层：Contract Identity。**

确认 Event 的发出地址是可信 Uniswap v2 Pair。

**第二层：Token Mapping。**

确认 `token0` 和 `token1` 来自对应的 Pair，而不是根据用户输入方向猜测。

**第三层：Amount Interpretation。**

依据 Token Decimals 将 Raw Amount 转换为正确的 Token 数量，并保留 Raw Value。

**第四层：Execution Semantics。**

将输入、输出金额映射到 Token 地址，计算本次 Swap 的 Execution Price。

**第五层：State Reconciliation。**

在合适的执行顺序与状态边界上，验证 Swap 与 Reserve 更新之间的关系，注意 Swap Fee、Token Balance 与内部 Reserve State 的差异。

这一套校验思路，直接继承之前的 Data Quality 与 Reconciliation 模块。

---

## 二十二、本课需要记住的工程原则

**第一，Protocol Event 不是已经完成的数据模型。**

```text
Decoded Event
≠
Trusted Business Fact
```

**第二，Token Ordering 与 Trade Direction 是两个不同维度。**

```text
token0 / token1
≠
token_in / token_out
```

**第三，Event Fact 与 State Snapshot 必须有明确 Grain。**

```text
Swap Event
≠
Sync Event
```

**第四，Transaction Hash 不一定是合适的业务唯一键。**

```text
One Transaction
→ Multiple Logs
→ Potentially Multiple Swaps
```

**第五，链上执行顺序决定历史状态解释的正确性。**

```text
Block Number alone
≠
Complete execution ordering
```

---

# 本课核心结论

> Uniswap v2 uses Factory contracts to create Pair contracts, while the Pair is the actual liquidity pool.

> `token0` and `token1` are fixed by Pair token ordering; they are not determined by the user's swap direction.

> `Swap` records Pair-level trading activity, whereas `Sync` records updated Pool Reserves.

> `Mint` and `Burn` represent liquidity operations; ERC-20 Transfers alone are insufficient to establish their full business semantics.

> One Ethereum transaction can produce multiple Swap Events, so `tx_hash` alone is not necessarily a valid Swap Fact unique key.

> A trusted DEX Indexer must preserve Contract Identity, Token Mapping, Raw Amounts, Event Ordering and Business Semantics.

> Pool Identity, Swap Fact and Reserve State History belong to different data grains.

---

# 理解检查

## 问题一：Protocol Semantics

假设某个 Uniswap v2 Pair：

```text
token0 = USDC
token1 = WETH
```

发生一个 Swap：

```text
amount0In  = 3,000 USDC
amount1In  = 0

amount0Out = 0
amount1Out = 0.98 WETH
```

请回答：

1. 这笔 Swap 中，Trader 输入与输出的 Token 分别是什么？
2. 如果下一笔 Trader 用 WETH 换 USDC，`token0` 和 `token1` 的身份会变化吗？
3. 为什么不能简单把 `token0` 当成 Input Token？

## 问题二：Event 与 State

同一笔 Transaction 中出现：

```text
log_index = 10 → Sync
log_index = 11 → Swap
```

请回答：

1. `Sync` 和 `Swap` 分别表达什么？
2. 为什么不能把所有 `Sync` Event 都认定为 Swap？
3. 如果一个 Block 内同一个 Pool 发生两笔 Swap，为什么不能直接用 Block End Reserve 来计算两笔交易各自的 `pool_price_before`？

## 问题三：Data Modeling

现在你准备构建：

```text
fact_dex_swap_v2
```

请回答：

1. 这张表应该采用什么 Grain？
2. 为什么不能只用 `chain_id + tx_hash` 作为 Unique Key？
3. `Swap.sender` 是否一定等于最终用户钱包？
4. `Pool Reserve State` 应不应该直接与 Swap Fact 混为同一条业务事实？为什么？

本课进入理解检查阶段。你可以分题回答，也可以一次性回答三题。

## 用户回答（理解检查｜问题一）

1. Trader 输入 Token 是 USDC ， 输出 Token 是 WETH
2. 不会变化
3. 因为它们 token 0 和 token 1 有固定的顺序规则，和 input、output 没有关系，他们是按 token contract address 的排序规则来确定的

## 老师判断 / 校准（问题一）

问题一通过，三点都正确。

1. 输入 Token = USDC，输出 Token = WETH：正确。
2. 下一笔即使交易方向变成 WETH → USDC，`token0` 与 `token1` 的身份也不会变化：正确。
3. 核心原因表述准确：`token0 / token1` 是 Pair-level fixed ordering，由 Token Contract Address 的排序规则决定；`token_in / token_out` 则是 Swap-level trade direction。

```text
token0 / token1
= pool identity / fixed ordering

token_in / token_out
= swap direction / per-trade semantics
```

本题判定：通过。
