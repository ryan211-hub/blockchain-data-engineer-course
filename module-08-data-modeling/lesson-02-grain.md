# 第2课｜Grain：为什么“一行代表什么”是数据建模的第一原则

## Lesson Contract
【Blockchain Data Engineer 视角】

所属 Module：Module 8 — 数据建模。

本课核心问题：**为什么设计任何事实表之前，第一件事不是列字段，而是先定义“一行到底代表什么”？**

学完本课应能够：
- 用一句话定义一张表的 Grain。
- 区分 Transaction Grain、Log / Transfer Grain、Pool Swap Grain、User Swap Grain、Wallet Daily Balance Grain。
- 解释为什么同一张表混入两个 Grain 会造成重复、歧义和错误聚合。
- 理解 Grain 与 Primary / Unique Key 的关系：先定义 Grain，再寻找能够唯一标识该 Grain 的 Key。
- 面对业务需求时，先确定分析对象和 Grain，再决定字段。

本课明确不展开：Fact / Dimension 的完整建模方法、复杂 Star Schema、SCD、ETL 实现、DEX 协议细节。

---
## 一、先忘掉字段，先问“一行是什么？”
上一课我们已经建立三个问题：
```text
1. 一行代表什么？ → Grain
2. 对象怎么唯一识别？ → Identity / Key
3. 哪些是事实，哪些是补充语义？ → Fact / Dimension / Derived Data
```
今天只处理第一个。

**Grain（粒度）就是：一张表中，一行数据所代表的最基本业务事实是什么。**

例如：
```text
transactions
一行 = 一笔链上 Transaction
```
```text
token_transfers
一行 = 一次 Token Transfer Event
```
```text
wallet_daily_balances
一行 = 某个 Wallet 在某一天、某个 Token 的余额快照
```

注意 Grain 不是“字段有多细”，而是**一行的语义单位**。

---
## 二、为什么 Grain 必须在字段之前确定？
假设产品需求是：
> 统计 Alice 最近 30 天的 USDC 转账。

一种错误的建模方式是直接开始列字段：
```text
wallet
transaction_hash
token
amount
block_time
...
```
这些字段本身都没有错，但表仍然没有定义清楚，因为我们不知道：
```text
一行 = 一笔 Transaction？
一行 = 一个 Transfer Event？
一行 = 一个 Wallet 的一日汇总？
```

如果 Grain 不明确，同样的字段组合可以表达完全不同的数据集。

所以正确顺序是：
```text
业务问题
   ↓
业务对象
   ↓
Grain
   ↓
Key
   ↓
Fields
```
而不是：
```text
先列 Fields
   ↓
最后再猜这张表是什么
```

---
## 三、Transaction Grain 与 Transfer Grain 不是一回事
【Ethereum 执行 / 数据工程视角】

假设 Alice 发起一笔 Transaction，这笔交易内部发生了 3 个 ERC-20 Transfer Event：
```text
Transaction 0xabc

Transfer #1
A → B  100 USDC

Transfer #2
B → C   50 USDC

Transfer #3
C → D   20 USDC
```

如果表是：
```text
transactions
一行 = 一笔 Transaction
```
那么这笔链上交易只应该有：
```text
1 row
```

如果表是：
```text
token_transfers
一行 = 一个 Token Transfer Event
```
那么同一笔 Transaction 应该有：
```text
3 rows
```

两张表都正确，因为 Grain 不同。

这也解释了为什么：
```text
(chain_id, tx_hash)
```
可以标识 Transaction Grain，但通常不能唯一标识 Transfer Grain。

Transfer Grain 还需要进一步区分同一 Transaction 中的多个 Event，例如：
```text
(chain_id, tx_hash, log_index)
```

因此有一个非常重要的顺序：
> **不是先看到一个 Unique Key，然后决定 Grain；而是先定义 Grain，再设计能够唯一标识这个 Grain 的 Key。**

---
## 四、同一 Transaction 中，Grain 可以继续变化
我们再用上一课的 Router Swap：
```text
Alice
1000 USDC
   ↓
Pool A
USDC → WETH
   ↓
Pool B
WETH → DAI
   ↓
3000 DAI
```

可以存在至少三种合理 Grain：

```text
transactions
一行 = 一笔 Transaction
→ 1 row
```

```text
pool_swaps
一行 = 一个 Pool 实际执行的 Swap
→ 2 rows
```

```text
user_swaps
一行 = 一个用户完整 Swap 意图
→ 1 row
```

这说明一个关键问题：
> **链上并不存在唯一正确的“最标准 Grain”。Grain 取决于你正在描述哪个业务对象。**

【协议执行视角】Pool Swap 是一个真实执行事实。

【用户业务视角】User Swap 是一个更高层的业务事实。

两者不能互相替代。

---
## 五、Grain 错误最危险的情况：Mixed Grain
假设有人设计一张表：
```text
wallet_activity

chain_id
tx_hash
wallet_address
token_address
transfer_amount
tx_gas_used
```
看起来字段都很合理。

但如果“一行 = 一个 Token Transfer”，这里会出现问题。

假设一笔 Transaction 有 3 个 Transfer：
```text
Transaction 0xabc
gas_used = 90,000

Transfer 1
Transfer 2
Transfer 3
```
如果把 Transaction-level 的 `gas_used = 90,000` 填到每一个 Transfer 行：
```text
Transfer 1 → gas_used = 90,000
Transfer 2 → gas_used = 90,000
Transfer 3 → gas_used = 90,000
```
然后分析师执行：
```sql
SUM(gas_used)
```
得到：
```text
270,000
```
但真实 Transaction Gas 只有：
```text
90,000
```

这不是 SQL 写错，也不是链上数据错。

真正的问题是：
> **把 Transaction Grain 的指标复制到了 Transfer Grain 的表里，然后按 Transfer Grain 聚合。**

这就是 Mixed Grain 带来的典型错误。

---
## 六、判断一个字段能否进入某张表，要问它属于哪个 Grain
如果：
```text
token_transfers
一行 = 一个 Transfer Event
```
那么天然属于这一 Grain 的字段有：
```text
tx_hash
log_index
token_address
from_address
to_address
amount_raw
```

而：
```text
tx_gas_used
```
属于：
```text
Transaction Grain
```
它不是不能出现在 Transfer 表，而是如果为了查询便利进行冗余，就必须明确：
- 它是重复的 Transaction-level 属性；
- 不能直接在 Transfer Grain 上无脑 SUM；
- 模型消费者必须知道这种语义。

【Blockchain Data Engineer 视角】因此建模并不是“字段能不能 Join 进来”，而是：
> **Join 进来以后，这个字段在当前 Grain 下还有没有正确的聚合语义？**

---
## 七、再看一个完全不同的 Grain：Wallet Daily Balance
需求：
> 查询每个 Wallet 每天结束时持有多少 USDC。

我们可以设计：
```text
wallet_daily_balances
```
Grain：
```text
一行 =
一个 chain
+ 一个 date
+ 一个 wallet
+ 一个 token
的日末余额
```
例如：
```text
chain_id      = 1
date          = 2026-09-10
wallet        = 0xAlice
token         = USDC
balance       = 1250.50
```

这里最重要的是：这已经不是 Event Grain。

底层可能有：
```text
上午收到 100 USDC
下午转出 50 USDC
晚上收到 200 USDC
```
3 个 Transfer Event，最终只形成：
```text
1 个 Wallet Daily Balance row
```

所以：
```text
Transfer = 流水事实
Balance  = 状态 / 快照事实
```
它们的 Grain 完全不同。

---
## 八、Grain 往往可以写成一句“唯一性声明”
实际工作中，我建议你在建表前强迫自己先写一句：
> **One row represents ...**

例如：
```text
fact_transactions
One row represents one canonical transaction on one chain.
```

```text
fact_token_transfers
One row represents one canonical token transfer event on one chain.
```

```text
fact_pool_swaps
One row represents one swap execution emitted by one liquidity pool.
```

```text
wallet_daily_balances
One row represents the end-of-day balance of one wallet-token pair on one chain and one date.
```

如果你无法清楚写出这句话，通常说明表还没有设计好。

---
## 九、Grain 与 Key 的关系
先定义：
```text
Grain
```
再问：
```text
什么字段组合能够唯一标识这一行？
```

例如：
```text
Transaction Grain
→ (chain_id, tx_hash)
```

```text
EVM Log / Transfer Event Grain
→ (chain_id, tx_hash, log_index)
```

```text
Wallet Daily Token Balance Grain
→ (chain_id, date, wallet_address, token_address)
```

这里先不讨论 surrogate key 等经典数仓设计，只建立一个原则：
> **Unique Key 是 Grain 的数据库表达，而 Grain 是业务语义。**

不能只看到 Unique Constraint 就以为自己理解了模型。

---
## 十、一个与你银行数据经验很接近的例子
银行中可能同时有：
```text
account_transactions
一行 = 一笔账户流水
```
和：
```text
account_daily_balance
一行 = 一个账户一天的日末余额
```

如果今天账户发生 20 笔交易：
```text
account_transactions → 20 rows
account_daily_balance → 1 row
```

没人会认为其中一张表“不完整”。

因为它们回答不同问题：
```text
流水表：今天发生了什么？
余额表：今天结束时状态是什么？
```
链上也是完全相同的建模逻辑。

---
## 十一、本课核心心智模型
```text
业务需求
   ↓
Business Object
   ↓
定义 Grain
“一行代表什么？”
   ↓
确定唯一身份
Key / Unique Key
   ↓
判断字段是否属于该 Grain
   ↓
设计 Table
```

数据建模中最危险的做法之一，是先堆字段再讨论含义。

本课只要求记住一句：
> **Before designing columns, declare the grain.**

---
## 理解检查
### 问题 1
假设一笔 Ethereum Transaction 里面产生了 5 个 ERC-20 Transfer Event。

请分别回答：
- `transactions` 表如果 Grain 是“一行一笔 Transaction”，应该写几行？
- `token_transfers` 表如果 Grain 是“一行一个 Transfer Event”，应该写几行？
- 为什么不能只用 `(chain_id, tx_hash)` 作为 `token_transfers` 的 Unique Key？

### 问题 2
有一张 `token_transfers` 表：
```text
一行 = 一个 Transfer Event
```
有人把 `tx_gas_used` 冗余到每一行，然后执行：
```sql
SELECT SUM(tx_gas_used)
FROM token_transfers;
```
为什么这个结果很可能被放大？问题根源是什么？

### 问题 3
设计一张：
```text
wallet_daily_token_balances
```
需求是保存每个 Wallet 每天每个 Token 的日末余额。

请你用一句话定义它的 Grain，并尝试给出一个合理的 Unique Key。

本课当前状态：等待理解检查回答，尚未正式结束。

### 问题 1｜用户回答
> 问题一：
>
> 1. transactions 应该产生一行，而 token transfers 应该产生五行。
> 2. token transfer 不能只使用 chain ID 和 tx hash 作为 unique key，因为这两个字段不能完全确定一条 token transfer。需要增加一个字段 log index，通过这三个字段的组合，才能作为 token transfer 的 unique key

### 问题 1｜老师判断与校准
**判断：通过。**

你的回答已经完整抓住了两个点：

1. `transactions` 的 Grain 是“一行一笔 Transaction”，所以这一笔交易只产生 1 行；`token_transfers` 的 Grain 是“一行一个 Transfer Event”，同一笔 Transaction 中有 5 个 Transfer Event，因此产生 5 行。
2. `(chain_id, tx_hash)` 只能唯一标识这笔 Transaction，不能区分同一 Transaction 内的多个 Transfer Event；需要再加入 `log_index`，形成 `(chain_id, tx_hash, log_index)`，才能唯一标识 EVM Log / Transfer Event Grain。

这里可以进一步固定一句本课原则：
> **先定义 Grain，再设计能够唯一标识该 Grain 的 Unique Key。**

问题 1 已通过；问题 2、问题 3 待回答。
