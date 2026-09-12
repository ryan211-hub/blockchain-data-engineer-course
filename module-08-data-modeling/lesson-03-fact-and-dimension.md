# 第3课｜Fact 与 Dimension：哪些数据是事实，哪些数据是维度

## Lesson Contract

【Blockchain Data Engineer 视角】

所属 Module：Module 8 — 数据建模。

本课核心问题：

> 当 Grain 已经确定以后，一张分析模型里的字段应该如何分工？哪些字段描述“发生了什么”，哪些字段描述“这件事涉及的对象是谁、是什么”？

学完本课，你应该能够：

1. 用直观方式区分 Fact 与 Dimension。
2. 判断 `amount_raw / amount / token_symbol / decimals / wallet_label / block_time` 等字段更接近哪一类。
3. 理解为什么 Fact 与 Dimension 分开后，模型更稳定、更容易复用。
4. 理解“链上有这个字段”与“它属于事实表”不是同一回事。
5. 初步设计 `fact_token_transfers + dim_tokens` 这样的模型。

本课明确不展开：

- Star Schema / Snowflake Schema 的系统理论；
- SCD；
- surrogate key 的深入设计；
- ETL 实现；
- DEX / Lending 等复杂协议建模。

---

## 一、上一课解决了“一行是什么”，这一课解决“这一行里放什么”

上一课我们已经确定：

```text
Business Object
      ↓
Grain
      ↓
Unique Key
```

例如：

```text
fact_token_transfers

一行 = 一个 Token Transfer Event
```

以及：

```text
Unique Key
= (chain_id, tx_hash, log_index)
```

但接下来还有一个问题：

```text
这一行到底应该放哪些字段？
```

比如我们可能想放：

```text
chain_id
tx_hash
log_index
block_time

token_address
token_symbol
token_name
decimals

from_address
from_label
to_address
to_label

amount_raw
amount
amount_usd
```

看起来都“有用”。

但如果全部塞进同一张表，就会产生新的问题：

- 哪些字段属于这个 Transfer 本身？
- 哪些字段其实只是 Token 的属性？
- 哪些字段只是 Wallet 的标签？
- 哪些字段是计算出来的？
- 哪些字段会变化？
- 哪些字段会被大量重复？

这就是 Fact / Dimension 要解决的问题。

---

## 二、先建立最简单的心智模型

可以先暂时不用教科书定义。

先记：

> **Fact：发生了什么。**

> **Dimension：这件事涉及的对象是什么。**

例如：

```text
Alice 给 Bob 转了 100 USDC
```

拆开来看：

### Fact

```text
Transfer
Alice → Bob
amount = 100
time = 10:00
```

它描述的是：

> 一件事情发生了。

### Dimension

```text
USDC
symbol = USDC
name = USD Coin
decimals = 6
token_type = ERC20
```

它描述的是：

> 这个 Token 是什么。

再比如：

```text
Alice
label = Binance Hot Wallet
entity_type = Exchange
```

它描述的是：

> 这个 Wallet 是谁。

所以最基本的结构就是：

```text
Fact
= Event / Activity / Measurement

Dimension
= Entity / Attribute / Context
```

---

## 三、用 ERC-20 Transfer 来看

假设链上发生：

```text
Transfer(
    Alice,
    Bob,
    100000000
)
```

【链上事实视角】

这条 Event 直接告诉我们的核心内容是：

```text
token_contract
from
to
value_raw
```

Indexer 再附带：

```text
chain_id
block_number
block_time
tx_hash
log_index
```

于是我们可以形成：

```text
fact_token_transfers
```

例如：

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
```

这里描述的是：

> 某条链上，在某个时间，发生了一次 Token Transfer。

这是 Fact。

---

## 四、那 token_symbol 和 decimals 为什么不一定放 Fact？

假设：

```text
token_address = 0xA0b8...
```

这是 USDC。

它对应：

```text
symbol   = USDC
name     = USD Coin
decimals = 6
```

注意：

每发生一次 USDC Transfer，这三个值通常都是一样的。

如果一天有：

```text
10,000,000
```

条 USDC Transfer，

你把：

```text
symbol = USDC
name = USD Coin
decimals = 6
```

重复写：

```text
10,000,000 次
```

当然技术上可以。

但从模型设计看：

> 这三个字段描述的不是“这一笔 Transfer 发生了什么”，而是在描述“USDC 这个 Token 是什么”。

所以更自然的做法是：

```text
dim_tokens
```

保存：

```text
chain_id
token_address
symbol
name
decimals
token_type
```

然后 Fact 只保留：

```text
token_address
```

或者后续可能保留：

```text
token_id
```

作为关联。

于是变成：

```text
fact_token_transfers
        │
        │ token identity
        ↓
dim_tokens
```

---

## 五、这里有一个非常重要的判断方式

判断字段是不是 Dimension，不要只问：

> 这个字段是不是来自链上？

而要问：

> **这个字段是在描述一次事件，还是在描述一个实体？**

例如：

```text
decimals
```

它确实可以来自链上 Token Contract。

但它描述的是：

> Token 的精度属性。

它不是在描述：

> 这一次 Transfer 做了什么。

所以：

```text
“来自链上”
≠
“一定属于 Fact”
```

这是一个很重要的边界。

---

## 六、amount_raw、amount、amount_usd 怎么看？

这三个字段非常适合做层次区分。

### amount_raw

```text
100000000
```

这是 Transfer Event 直接记录的数量。

所以它非常接近：

```text
原始事实
Raw / Source Fact
```

### amount

如果：

```text
decimals = 6
```

那么：

```text
amount
= amount_raw / 10^6
= 100
```

它已经是标准化结果。

它仍然属于“这次 Transfer 的数量”，所以从分析模型角度通常仍然放在 Fact 中。

也就是说：

```text
amount_raw
amount
```

都可以属于 Fact。

但性质不同：

```text
amount_raw
= source fact

amount
= normalized fact
```

### amount_usd

如果：

```text
100 USDC
×
$1
=
$100
```

那么：

```text
amount_usd
```

是进一步衍生出来的业务度量。

它也可以存在于 Fact 或更高层分析模型，但你要清楚：

```text
amount_usd
不是链上原始事实
```

而是：

```text
Derived Measure
```

所以这里可以形成一个层次：

```text
amount_raw
    ↓
amount
    ↓
amount_usd

Raw Fact
    ↓
Normalized Measure
    ↓
Derived Business Measure
```

---

## 七、Dimension 最核心的价值：把“对象属性”集中管理

假设你不做 `dim_tokens`。

那每张表都自己存：

```text
token_symbol
token_name
decimals
token_type
```

那么可能出现：

```text
fact_token_transfers
fact_swaps
fact_balances
fact_prices
fact_liquidity
```

每张表都重复一份 Token 信息。

以后如果：

```text
symbol
token category
stablecoin classification
canonical token mapping
```

发生补充或修正，

你可能需要同时改很多表。

而有了：

```text
dim_tokens
```

之后：

```text
fact_token_transfers ─┐
fact_swaps            ├──→ dim_tokens
fact_balances         ┤
fact_prices           ┘
```

Token 的公共属性集中在一个地方。

这就是 Dimension 的复用价值。

---

## 八、银行数据中其实也是完全一样的

假设有一张：

```text
fact_transactions
```

保存：

```text
transaction_id
account_no
amount
txn_time
branch_code
```

然后：

```text
dim_branch
```

保存：

```text
branch_code
branch_name
city
province
region
branch_level
```

交易事实回答：

> 发生了多少金额的交易？

Branch Dimension 回答：

> 这个机构是什么机构？属于哪里？

你不会希望每一笔交易都重复存：

```text
上海分行
上海市
华东地区
一级分行
```

区块链数据模型也是同一个逻辑。

---

## 九、Wallet Dimension 要更谨慎

Token Dimension 相对容易，因为：

```text
symbol
name
decimals
```

比较像对象属性。

Wallet 就复杂一点。

例如：

```text
0xABC...
```

我们可能想知道：

```text
label = Binance
entity_type = Exchange
category = CEX
risk_level = ...
```

这些都不是链原生事实。

而是：

```text
Enrichment / Business Semantics
```

于是可以有：

```text
dim_wallets
```

或者更专业一些：

```text
dim_wallet_labels
```

保存：

```text
chain_id
wallet_address
label
entity
entity_type
source
confidence
```

【Blockchain Data Engineer 视角】

这里一定要注意：

```text
wallet_address
```

是链上身份。

但：

```text
Binance Hot Wallet
```

是链下语义标签。

所以：

```text
Address Identity
≠
Business Identity
```

这会在后面的 Source Identity / Business Identity 课程继续展开。

---

## 十、block_time 是 Fact 还是 Dimension？

这是一个很好的边界问题。

现实数据仓库里，经常会有：

```text
dim_date
dim_time
```

但在链上事实表里：

```text
block_time
```

通常仍然会直接存在。

例如：

```text
fact_token_transfers

chain_id
block_time
tx_hash
log_index
...
```

为什么？

因为：

> 发生时间是这个事实本身非常核心的属性。

同时为了分析：

```text
按日
按周
按月
工作日 / 周末
季度
```

也可以再关联：

```text
dim_date
```

所以不要把 Fact / Dimension 理解成一种机械规则：

```text
时间一定 Dimension
字符串一定 Dimension
数字一定 Fact
```

这是错误的。

真正判断依据还是：

> **这个字段在模型里扮演什么语义角色？**

---

## 十一、Fact 并不等于“数字字段”

这是初学数仓时最容易犯的错误之一。

例如：

```text
tx_hash
log_index
from_address
to_address
```

这些都不是数字度量。

但它们仍然属于 Fact 表。

因为它们是在描述：

> 这一条 Transfer Fact 的身份和参与者。

所以 Fact 表里通常既有：

```text
Measures
```

也有：

```text
Keys / IDs / References
```

例如：

```text
fact_token_transfers

chain_id        -- identity/context
tx_hash         -- identity
log_index       -- identity

token_address   -- dimension reference
from_address    -- participant reference
to_address      -- participant reference

amount_raw      -- measure
amount          -- normalized measure

block_time      -- event time
```

因此：

> Fact ≠ 只有数字。

---

## 十二、Dimension 也不等于“字符串字段”

例如：

```text
decimals = 6
```

它是数字。

但明显属于：

```text
Token Attribute
```

因此更接近 Dimension。

又比如：

```text
token_type = 20
```

即使你用数字编码，它依旧是维度属性。

所以：

```text
数据类型
≠
建模角色
```

Fact / Dimension 是语义划分，不是 SQL datatype 划分。

---

## 十三、一个最小链上分析模型

现在把前面组合起来。

### Fact

```text
fact_token_transfers
────────────────────────
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

> 一行 = 一个 Chain 上，一个 canonical Token Transfer Event。

Unique Key：

```text
(chain_id, tx_hash, log_index)
```

### Token Dimension

```text
dim_tokens
────────────────────────
chain_id
token_address

symbol
name
decimals
token_type
```

Grain：

> 一行 = 一个 Chain 上的一个 Token Contract。

Unique Key：

```text
(chain_id, token_address)
```

于是查询：

```text
Transfer Fact
   ↓ join
Token Dimension
```

可以得到：

```text
100000000
↓ decimals = 6
100
↓ symbol = USDC
100 USDC
```

---

## 十四、这一课真正想让你形成的判断顺序

以后看到某个字段，不要立刻问：

> 放哪张表？

先问三个问题：

```text
1. 这个字段描述的是一次事件，还是一个对象？
2. 它属于当前 Fact 的 Grain 吗？
3. 它是直接事实、标准化结果，还是补充语义？
```

例如：

```text
amount_raw
```

回答：

```text
描述事件
属于 Transfer Grain
直接事实
```

再比如：

```text
decimals
```

回答：

```text
描述 Token
不属于单次 Transfer Grain
Token 属性
```

再比如：

```text
amount_usd
```

回答：

```text
描述事件的价值
属于 Transfer 的分析度量
但不是链上直接事实
而是衍生结果
```

---

## 十五、本课核心心智模型

把前三课串起来：

```text
Business Question
       ↓
Business Object
       ↓
Grain
“一行是什么？”
       ↓
Unique Key
“怎么唯一识别？”
       ↓
Fact / Dimension
“哪些是事件，哪些是对象属性？”
       ↓
Analytical Model
```

第 1 课解决：

```text
为什么要建模？
```

第 2 课解决：

```text
一行是什么？
```

第 3 课解决：

```text
这一行应该放什么？
哪些属性应该拆出去？
```

---

## 理解检查

### 问题 1

假设有：

```text
fact_token_transfers
```

字段如下：

```text
chain_id
tx_hash
log_index
token_address
from_address
to_address
amount_raw
token_symbol
decimals
```

请你判断：

哪些字段更应该属于 Transfer Fact？

哪些字段更应该属于 Token Dimension？

重点解释：

> 为什么 `decimals` 虽然可以来自链上，但仍更适合作为 Dimension 属性？

### 问题 2

现在有：

```text
amount_raw = 100000000
decimals = 6
token_symbol = USDC
token_price_usd = 1
```

请你把：

```text
amount_raw
decimals
token_symbol
amount
amount_usd
```

按照下面四类来划分：

```text
链上直接事实
对象属性 / Dimension
标准化结果
业务衍生结果
```

### 问题 3

为什么把：

```text
token_symbol
token_name
decimals
token_type
```

全部重复存进：

```text
fact_token_transfers
fact_swaps
fact_balances
```

通常不是一个很好的基础模型设计？

请从：

1. 重复存储；
2. 维护；
3. 复用；

三个角度回答。

### 问题 1｜用户回答
> 问题一：
>
> 1. chain_id：属于 transfer_fact，同时也属于 token_dimension，它是一个来自链上的事实
> 2. tx_hash：属于 transfer_fact
> 3. log_index：属于 transfer_fact
> 4. token_address：属于来自链上的事实，但是 transfer_fact 和 token_dimension 都需要它
> 5. from_address：属于 transfer_fact
> 6. to_address：属于 transfer_fact
> 7. amount_raw：属于 transfer_fact
> 8. token_symbol：属于 token_dimension
> 9. decimals：属于 token_dimension
>
> 为什么 decimals 虽然可以来自链上，但仍更适合作为 dimension 属性？
> 因为 decimals 描述的是 token，它不属于单次 transfer 事件，是 token 的一个属性

### 问题 1｜老师判断与校准
**判断：通过。**

整体分类正确，尤其是对 `decimals` 的解释已经抓住了核心：Fact / Dimension 按语义角色划分，不按“是否来自链上”划分。

需要把两个角色再区分得更精确：
- `chain_id` 可以同时存在于 Fact 与 Dimension，但更准确的角色是 **Identity Namespace / Context**，不是像 `amount_raw` 那样的事实度量。
- `token_address` 在 Fact 中是 Dimension Reference，在 `dim_tokens` 中则是 Token Identity 的组成部分。

因此可以整理为：
```text
Transfer Fact

Identity / Context
- chain_id
- tx_hash
- log_index

Participant / Reference
- token_address
- from_address
- to_address

Measure
- amount_raw
```

而：
```text
Token Dimension

Identity
- chain_id
- token_address

Attributes
- token_symbol
- decimals
```

这里还要固定一个原则：
> **Fact 表不只是 Measure。Fact 表还会包含 Identity、参与者、Dimension Reference 和 Event Time。**

`decimals` 虽然可以从链上 Token Contract 获取，但它回答的是“这个 Token 是什么样的 Token”，而不是“这一笔 Transfer 发生了什么”，因此更适合作为 Token Dimension Attribute。

### 问题 2｜用户回答
> 问题二：
>
> 1. amount_raw 属于链上直接事实
> 2. decimals 属于对象属性
> 3. token_symbol 属于对象属性
> 4. amount 属于标准化结果
> 5. amount_usd 属于业务衍生结果

### 问题 2｜老师判断与校准
**判断：通过。**

分类完全正确：
- `amount_raw`：链上直接事实 / Source Fact；
- `decimals`：Token 对象属性 / Dimension；
- `token_symbol`：Token 对象属性 / Dimension；
- `amount`：由 `amount_raw` 与 `decimals` 计算得到的标准化结果；
- `amount_usd`：进一步结合价格得到的业务衍生结果。

这里需要固定的层次是：
```text
amount_raw
    ↓ decimals
amount
    ↓ price
amount_usd

Source Fact
    ↓
Normalized Measure
    ↓
Derived Business Measure
```

### 问题 3｜用户回答
> 问题三，如果把 token 的属性全部存到事实表里的话，主要有以下几个问题：
>
> 1. 导致重复存储，占用的数据容量会更大
> 2. 不便维护：如果 token 的属性有改动，就需要修改所有事实表中的历史数据
> 3. 关于复用：复用性这一块目前还不太清楚

### 问题 3｜老师判断与校准
**判断：通过，复用部分已补充校准。**

前两点判断正确：
1. **重复存储**：同一个 Token 的 `symbol / name / decimals / token_type` 会随着大量 Transfer、Swap、Balance 记录反复出现，增加冗余。
2. **维护成本**：如果 Token 属性、分类或映射需要补充或修正，分散在多张 Fact 中会造成多处更新与口径不一致风险。

“复用”的核心是：
> **多个 Fact 共享同一个 Dimension，从而复用同一套对象语义和业务口径。**

例如：
```text
fact_token_transfers ─┐
fact_swaps            ├──→ dim_tokens
fact_balances         ┘
```

三张 Fact 都通过 `(chain_id, token_address)` 关联同一 `dim_tokens`。这样 `symbol / decimals / token_type / canonical mapping` 只需要在统一维度层定义一次，所有下游模型得到相同解释。

因此 Dimension 的复用价值不仅是“少存几列”，更重要的是：
> **集中维护、统一语义、统一口径，让不同 Fact 对同一个业务对象使用同一份定义。**

---
## 本课结论
三道理解检查已经完成并通过必要校准，本课正式结束。

本课需要固定的核心判断：
1. **Fact 描述发生了什么；Dimension 描述参与对象是什么。**
2. **“来自链上”不等于“一定属于 Fact”**；判断依据是字段在模型中的语义角色。
3. Fact 中既可以有 Measure，也可以有 Identity、Participant、Reference 与 Event Time。
4. `amount_raw → amount → amount_usd` 分别对应链上直接事实、标准化结果与业务衍生结果。
5. Dimension 的关键价值是集中维护对象属性，并让多个 Fact 复用统一的业务语义和口径。