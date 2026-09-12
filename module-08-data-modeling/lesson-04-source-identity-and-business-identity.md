# 第4课｜Source Identity 与 Business Identity

## Lesson Contract

【Blockchain Data Engineer 视角】

所属 Module：Module 8 — 数据建模。

本课核心问题：

> 链上看到的是 `address / tx_hash / contract_address`，业务分析看到的却可能是 `USDC / Binance / Uniswap / Stablecoin`。这两套“身份”为什么不能混为一谈？

学完本课，你应该能够：

1. 区分 Source Identity 与 Business Identity。
2. 理解为什么多链环境下通常不能只用 `address` 作为唯一身份。
3. 解释为什么 Ethereum USDC、Base USDC、USDC.e 可以是不同 Source Identity，却可能映射到同一个更高层 Business Identity。
4. 理解 Wallet Address 与 Entity（如 Binance）之间通常不是一一对应。
5. 设计“底层保留真实身份、上层做业务映射”的基本模型。

本课明确不展开：

- Wallet Label 算法与聚类方法；
- 跨链桥协议细节；
- Token Registry 的完整治理机制；
- SCD；
- 实体识别机器学习；
- DEX / Lending 协议内部业务。

---

## 一、先从一个看起来很简单的问题开始

假设你有两个 Token：

```text
Ethereum
0xAAAA...
symbol = USDC
```

```text
Base
0xBBBB...
symbol = USDC
```

业务人员问：

> 这两个是不是“同一个 USDC”？

这个问题如果不先说明视角，是没有唯一答案的。

【链上 / Source 视角】

不是同一个对象。

因为：

```text
(chain_id = Ethereum, token_address = 0xAAAA...)
```

和：

```text
(chain_id = Base, token_address = 0xBBBB...)
```

是两个不同的链上 Contract Identity。

【业务分析视角】

它们又可能都属于：

```text
USDC
```

这个统一资产概念。

于是出现了本课的核心：

```text
Source Identity
≠
Business Identity
```

---

## 二、什么是 Source Identity？

先用一个工程化定义：

> **Source Identity：数据源本身能够直接、稳定区分一个对象的身份。**

在 Ethereum / EVM 中，Token Contract 常见的 Source Identity 是：

```text
(chain_id, token_address)
```

例如：

```text
(1, 0xA0b8...)
```

表示：

```text
Ethereum Mainnet 上的某个 Token Contract
```

为什么不是只用：

```text
token_address
```

因为同一个 20-byte address：

```text
0xABC...
```

完全可能同时存在于：

```text
Ethereum
Base
Arbitrum
Optimism
```

它们不是同一个链上对象。

所以：

```text
address
```

只有放进 Chain Namespace 后才真正完整。

这就是我们之前反复提到：

```text
chain_id + address
```

的原因。

---

## 三、Source Identity 不是“业务上它叫什么”

假设：

```text
token_address = 0xA0b8...
```

你可以从 Token Metadata 得到：

```text
symbol = USDC
name = USD Coin
```

但：

```text
USDC
```

不是一个足以唯一识别链上 Contract 的 Source Identity。

为什么？

因为：

```text
symbol
```

不是全局唯一的。

理论上完全可以存在：

```text
Token A → symbol = USDC
Token B → symbol = USDC
Token C → symbol = USDC
```

甚至恶意 Token 也可以把自己命名成：

```text
USDC
USDT
ETH
```

因此从数据工程角度：

> **不要用 symbol 当底层身份。**

底层身份必须尽量依赖数据源本身稳定提供的唯一定位信息。

---

## 四、那什么是 Business Identity？

> **Business Identity：为了业务分析，把一个或多个 Source Identity 映射成的业务对象身份。**

例如我们可以建立：

```text
asset_id = USDC
```

然后映射：

```text
Ethereum USDC ─┐
Base USDC     ─┼──→ asset_id = USDC
Arbitrum USDC ─┘
```

这样：

```text
(chain_id, token_address)
```

回答的是：

> 链上具体是哪一个 Contract？

而：

```text
asset_id = USDC
```

回答的是：

> 从业务分析角度，它属于哪个资产？

这就是两个完全不同的问题。

---

## 五、一个非常重要的模型原则

不要为了方便业务分析，直接丢掉 Source Identity。

错误做法：

```text
dim_tokens

symbol
name
...
```

然后：

```text
symbol = USDC
```

就认为所有 USDC 都是一条记录。

这样会丢失链上真实世界的差异。

更稳妥的设计是：

```text
dim_tokens
────────────────────
chain_id
token_address
symbol
name
decimals
business_asset_id
```

Grain：

> 一行 = 一个 Chain 上的一个 Token Contract。

于是：

```text
(chain_id, token_address)
```

仍然保留 Source Identity。

同时：

```text
business_asset_id
```

提供业务映射。

例如：

```text
Ethereum USDC → business_asset_id = USDC
Base USDC     → business_asset_id = USDC
```

所以：

> **底层保留真实差异，上层建立业务统一。**

这是本课最重要的一条原则。

---

## 六、为什么不能一开始就把它们合并成一条？

因为业务上的“相同”通常只是某个分析维度上的相同。

假设：

```text
Ethereum USDC
Base USDC
```

都属于 USDC。

但你仍然可能需要分析：

```text
Ethereum 上的 USDC Transfer Volume
Base 上的 USDC Transfer Volume
```

或者：

```text
哪条链上的 USDC 用户更多？
```

如果底层一开始就只保存：

```text
asset = USDC
```

而丢掉：

```text
chain_id
token_address
```

你就无法再还原这些差异。

因此：

```text
Source Identity
    ↓ 保留
Business Identity
    ↓ 映射
Business Aggregation
```

而不是：

```text
Source Identity
    ↓ 覆盖掉
Business Identity
```

---

## 七、USDC 与 USDC.e：这时问题更有意思

假设某条链上存在：

```text
Native USDC
```

以及：

```text
USDC.e
```

业务上，它们都和 USDC 有关系。

但链上身份不同：

```text
Token Contract A
Token Contract B
```

而且经济含义可能也不完全相同：

```text
Native USDC
→ Circle 原生发行
```

```text
USDC.e
→ 从其他链桥接而来的资产表示
```

因此你可能需要两个层次：

```text
Source Token Identity
```

和：

```text
Asset Family / Business Asset
```

例如：

```text
Native USDC ─┐
             ├──→ USDC Family
USDC.e      ─┘
```

但是在某些风险、流动性或资产负债分析里：

```text
Native USDC
```

和：

```text
USDC.e
```

又不能直接当成完全等价。

这说明：

> **Business Identity 本身也取决于业务问题。**

---

## 八、所以 Business Identity 不是“绝对真理”

这是这一课需要特别注意的地方。

Source Identity 通常比较客观：

```text
chain_id = 1
token_address = 0x...
```

它是链上可观察事实。

但：

```text
这个 Token 是否属于 USDC Family？
```

属于：

```text
Business Mapping
```

它可能由：

```text
Token Registry
Asset Mapping Rule
人工维护
协议规则
业务口径
```

决定。

因此：

```text
Source Identity
= 数据源事实
```

而：

```text
Business Identity
= 建模语义
```

后者天然比前者多了一层人为定义。

---

## 九、Wallet 的例子更明显

假设链上有三个地址：

```text
0xA
0xB
0xC
```

【链上视角】

这是三个不同 Wallet Address。

Source Identity 可以写成：

```text
(chain_id, wallet_address)
```

但是某数据公司经过标签分析发现：

```text
0xA → Binance
0xB → Binance
0xC → Binance
```

于是业务模型可能有：

```text
entity_id = BINANCE
```

形成：

```text
0xA ─┐
0xB ─┼──→ Binance
0xC ─┘
```

所以：

```text
Wallet Address
≠
Business Entity
```

一家公司可以控制很多地址。

同一个地址也可能因为证据变化，业务标签被重新判断。

这就是为什么 Wallet Label 不能直接覆盖链上地址身份。

---

## 十、银行系统里其实也有完全相同的问题

用你更熟悉的银行场景。

假设：

```text
account_no = 6222...
```

这是 Source System 中一个账户的身份。

但客户：

```text
customer_id = C001
```

可能有：

```text
储蓄账户
信用卡账户
理财账户
外币账户
```

于是：

```text
account_no
```

和：

```text
customer_id
```

不是一个层次。

可以理解成：

```text
多个 Account
      ↓
一个 Customer
```

类似链上：

```text
多个 Wallet Address
      ↓
一个 Binance Entity
```

或者：

```text
多个 Token Contract
      ↓
一个 USDC Business Asset
```

---

## 十一、从数据库建模来看，通常是 Many-to-One

最常见结构：

```text
Source Identity A ─┐
Source Identity B ─┼──→ Business Identity X
Source Identity C ─┘
```

例如：

```text
Ethereum USDC ─┐
Base USDC     ─┼──→ USDC
Arbitrum USDC ─┘
```

或者：

```text
Wallet A ─┐
Wallet B ─┼──→ Binance
Wallet C ─┘
```

但不要死记成一定 Many-to-One。

有些业务关系可能更复杂。

比如某 Wallet 的标签可能同时包含：

```text
entity = Binance
category = Exchange
role = Hot Wallet
```

所以实际工程里可能拆成多个映射维度。

本课只要求建立最基本的：

```text
Source Identity
→ Business Mapping
→ Business Identity
```

---

## 十二、Source Identity 最重要的价值：可追溯

假设分析结果说：

```text
USDC Transfer Volume = $5B
```

Blockchain Data Engineer 必须能够继续往下追：

```text
USDC
↓
哪些链？
↓
哪些 token contracts？
↓
哪些 transfer rows？
↓
哪些 tx_hash / log_index？
```

这叫：

```text
Traceability
```

如果你只保存了 Business Identity：

```text
USDC
```

而没有保留源身份，

你就失去了：

```text
业务结果
→ 原始事实
```

的追溯链。

对于数据平台来说，这是严重问题。

---

## 十三、这与前两课刚好连起来

第 2 课我们学：

```text
Grain
```

第 3 课学：

```text
Fact / Dimension
```

现在加上：

```text
Identity
```

例如：

```text
dim_tokens
```

Grain：

```text
一行 = 一个 Chain 上的一个 Token Contract
```

Source Identity：

```text
(chain_id, token_address)
```

Dimension Attributes：

```text
symbol
name
decimals
token_type
```

Business Mapping：

```text
business_asset_id
```

于是模型已经开始完整起来：

```text
Grain
+
Source Identity
+
Dimension Attributes
+
Business Identity
```

---

## 十四、一个最小模型

可以设计：

```text
dim_tokens
────────────────────────
chain_id
token_address

symbol
name
decimals
token_type

business_asset_id
```

Source Grain：

> 一行代表一个 Chain 上的一个 Token Contract。

Source Unique Key：

```text
(chain_id, token_address)
```

然后：

```text
dim_business_assets
────────────────────────
business_asset_id
asset_symbol
asset_name
asset_category
```

例如：

```text
USDC
USD Coin
Stablecoin
```

关联关系：

```text
dim_tokens
    │
    │ business_asset_id
    ↓
dim_business_assets
```

这就把两个层次明确拆开了。

---

## 十五、什么时候应该按 Source Identity 查询，什么时候按 Business Identity？

如果问题是：

> Base 上 USDC Contract 的 Transfer Volume 是多少？

使用：

```text
Source Identity
(chain_id, token_address)
```

如果问题是：

> 全部链上的 USDC 总 Transfer Volume 是多少？

使用：

```text
Business Identity
business_asset_id = USDC
```

所以选择哪一个，不是技术偏好，而是：

> **Query Semantics。**

---

## 十六、本课核心心智模型

```text
Blockchain / Source
        ↓
Source Identity
(chain_id, address)
        ↓
保留真实链上对象
        ↓
Business Mapping
        ↓
Business Identity
(USDC / Binance / ...)
        ↓
Business Analysis
```

请固定一句：

> **Source Identity 负责“它在源数据里到底是谁”，Business Identity 负责“业务上我们把它当成谁”。**

两者不能互相替代。

---

## 理解检查

### 问题 1

Ethereum 上有：

```text
token_address = 0xAAA
symbol = USDC
```

Base 上有：

```text
token_address = 0xBBB
symbol = USDC
```

请回答：

1. 从 Source Identity 角度，它们是不是同一个对象？为什么？
2. 从 Business Identity 角度，是否可以把它们映射成同一个 `USDC`？
3. 为什么底层 `dim_tokens` 不能只保留一条 `USDC`？

### 问题 2

假设：

```text
0xA
0xB
0xC
```

三个地址都被标记为 Binance。

请回答：

1. Source Identity 有几条？
2. Business Identity 可以有几条？
3. 为什么不能因为它们都属于 Binance，就把三个地址在底层直接合并掉？

### 问题 3

现在有两类需求：

A：

> 查询 Base 链某个 USDC Contract 的 Transfer Volume。

B：

> 查询 Ethereum + Base + Arbitrum 全部 USDC 的总 Transfer Volume。

请判断：

- A 更应该使用 Source Identity 还是 Business Identity？
- B 更应该使用 Source Identity 还是 Business Identity？
- 为什么？

### 问题 1｜用户回答
> 从 source identity 角度，它们不是同一个对象。它们是在不同的链上的两个不同的token：一个是在 Ethereum 上，另外一个是在 Base 链上。
> 从 business identity 角度，它们可以映射为同一个 USDC
> 为什么底层 dim_tokens 不能只保留一条 USDC？
>
> 因为为了数据可以追溯，我们必须保留原始数据，而不能在 USDC 业务层面将数据整合后只保留成一条。这样对数据分析来说是丢失了很多信息，无法再追踪还原 token 在链上的真实情况。

### 问题 1｜老师判断与校准
**判断：通过。**

你的答案已经抓住了本课的核心：
- Source Identity 负责保留链上真实对象，因此 Ethereum USDC 与 Base USDC 是两个不同的 Source Identity；
- Business Identity 可以把多个 Source Identity 映射到同一个 `USDC` 业务资产；
- 底层 `dim_tokens` 不能只保留一条 `USDC`，因为那会丢失链、合约地址等源身份信息，破坏可追溯性，也会使后续按链、按 Contract 的分析无法还原。

这里再固定一个更精确的表达：
> **底层模型负责保真，业务层负责统一。**

也就是：
```text
Source Identity
(chain_id, token_address)
        ↓ 保留
Business Mapping
        ↓
Business Identity
(USDC)
```

问题 1 已通过；问题 2、问题 3 待回答。