# 第4课｜什么时候真正需要 Archive Node：Historical State、数仓派生状态与需求决策

## Lesson Contract

【Blockchain Data Engineer 视角】

- 所属 Module：Module 6 — Archive Node
- 本课核心问题：一个业务需求涉及“历史数据”，是不是就意味着一定需要 Archive Node？
- 学完后应能回答 / 设计：业务需求 → 识别数据类型 → History / Log / Derived State / Historical State → 判断已有数仓是否能回答 → 决定普通 Node / Indexer / Warehouse / Archive RPC / 自建 Archive。
- 本课明确不展开：Indexer Pipeline 具体实现；Checkpoint / Backfill 机制；如何设计 Balance Snapshot 表；如何解析 Aave / Uniswap 协议状态。这些进入 Module 7 以后。

## 一、先解决最重要的误区

很多团队会说：

> “我们要查历史数据，所以需要 Archive Node。”

这个判断太粗。

因为“历史数据”至少可能指四种不同东西：

```text
1. Historical Block / Transaction
2. Historical Log / Event
3. Derived Historical Data
4. Historical State
```

只有第 4 类真正直接指向 Archive State 能力。

所以第一原则是：

> **Historical Data ≠ Historical State**

这句话其实是 Module 6 前几课的最终工程化版本。

## 二、第一类：Historical Block / Transaction

例如：

```text
查询 Block 15,000,000

查询某个旧 Transaction

查询某个旧 Receipt
```

这类需求主要是在问：

> 过去发生了什么？

它属于：

```text
History
```

而不是：

```text
Historical State
```

因此一个保留相应历史数据的普通 Full Node、RPC Provider 或自己的 Indexer 就可能回答。

不应该因为：

```text
block_number 很旧
```

就自动得出：

```text
需要 Archive Node
```

## 三、第二类：Historical Logs / Events

例如：

```text
2024 年所有 USDC Transfer

2023 年所有 Uniswap Swap

某合约过去两年的 Deposit Event
```

它们主要依赖：

```text
Logs
Events
Receipts
```

这同样属于 History。

典型数据路径是：

```text
Historical Block
↓
Receipt
↓
Log
↓
Decode
↓
Fact Table
```

例如：

```text
ERC20 Transfer Log
↓
token_transfers
```

或者：

```text
Uniswap Swap Event
↓
dex_swaps
```

所以即使查询跨越几年，也未必需要 Archive State。

这点非常重要。

因为：

> 查询范围大 ≠ 查询类型是 Historical State。

范围大意味着：

```text
expensive query
```

但不意味着：

```text
archive state query
```

## 四、第三类：Derived Historical Data

这是本课的新重点。

【数据仓库视角】

假设业务问：

> Alice 在 2025-01-01 的 ETH Balance 是多少？

从链上原始数据语义上讲：

```text
Balance
=
State
```

所以直接向 Node 查询：

```text
eth_getBalance(
    Alice,
    old_block
)
```

确实需要 Historical State。

但是假设你的数据平台已经自己维护了：

```text
wallet_balance_snapshot
```

例如：

| block_number | address | balance |
|---|---|---:|
| 20,000,000 | Alice | 10 ETH |
| 20,100,000 | Alice | 8 ETH |
| 20,200,000 | Alice | 12 ETH |

那么业务问历史余额时：

```text
SELECT balance
FROM wallet_balance_snapshot
WHERE address = 'Alice'
AND block_number = ...
```

就能回答。

这时候：

> 业务问题本身是 Historical State 问题。

但：

> **当前查询不一定还需要 Archive Node。**

因为 State 已经提前被你派生并保存进 Warehouse。

这是 Blockchain Data Engineer 非常重要的思维。

## 五、Archive Node 和 Data Warehouse 是替代关系吗？

不是完全替代。

更准确地说：

```text
Archive Node
=
通用 Historical State Source
```

而：

```text
Warehouse
=
针对特定业务问题预计算 / 派生后的数据
```

例如 Archive Node 可以回答很多任意状态查询：

```text
Alice ETH balance at block N

USDC balanceOf(Alice) at block N

Contract slot X at block N

Aave getUserAccountData at block N
```

Warehouse 则可能只维护：

```text
wallet_balance_snapshot

token_balance_snapshot

aave_user_position_snapshot
```

优点是：

```text
查询快
成本可控
适合分析
```

缺点是：

```text
必须提前知道要保存什么
```

如果没保存：

```text
某历史状态
```

你可能就只能重新构建或回 Archive State Source 查。

## 六、一个银行系统类比

假设银行核心系统有每天的账户状态。

方案 A：

```text
保留完整历史核心账务状态
```

类似 Archive Node。

你可以随时问：

> 2021-07-13 14:31:20 Alice 的账户状态是什么？

方案 B：

数据仓库每天只保存：

```text
daily_account_balance
```

那么：

```text
2021-07-13 日终余额
```

很好查。

但如果突然问：

```text
2021-07-13
14:31:20
冻结金额是多少？
```

如果数仓当时没保存：

```text
frozen_amount snapshot
```

就未必回答得出来。

因此：

```text
Archive
=
通用历史状态能力

Warehouse Snapshot
=
针对已知分析需求保存的历史状态派生结果
```

这是两者真正的区别。

## 七、于是出现第一个决策问题

假设业务说：

> 我们要支持历史余额查询。

不要马上部署 Archive Node。

先问：

```text
需要查询哪些资产？
```

可能只是：

```text
ETH
USDC
USDT
```

再问：

```text
查询精度是什么？
```

可能只是：

```text
每日余额
```

而不是每个 Block。

那么完全可能设计成：

```text
Indexer
↓
Balance Calculation
↓
Daily Snapshot
↓
Warehouse
```

这样长期运行成本可能远低于：

```text
Archive Node
```

因此真正的问题不是：

> Archive Node 能不能做？

当然能。

而是：

> **Archive Node 是不是满足这个需求成本最低、最合适的方法？**

## 八、什么时候 Warehouse 更合适？

【数据产品视角】

如果需求具有以下特点：

```text
字段固定
+
业务稳定
+
查询频繁
+
需要聚合分析
```

例如：

```text
每日钱包余额
每日 TVL
每日 Lending Position
每日 Token Holder Count
```

更适合：

```text
Indexer
+
Derived Table / Snapshot
+
Warehouse
```

原因很简单：

Archive Node 本质上还是 Node Database。

它不是分析数据库。

你不应该让 BI Dashboard 每次刷新都：

```text
for wallet in 10,000,000 wallets:
    eth_getBalance(wallet, old_block)
```

这会非常昂贵。

更合理的是：

```text
Archive / Node / Indexer
        ↓
ETL
        ↓
Warehouse
        ↓
Dashboard
```

## 九、什么时候 Archive State 特别有价值？

有三类典型场景。

### 场景 1：Ad-hoc Historical Query

即：

> 事先不知道未来会查什么。

比如研究人员突然问：

```text
Block 16,432,123
某 Contract slot 的值是什么？
```

Warehouse 不可能提前保存所有：

```text
contract
×
storage slot
×
block
```

这时 Archive 很有价值。

### 场景 2：Historical eth_call

例如：

```text
eth_call(
    Aave.getUserAccountData(Alice),
    block=18,000,000
)
```

这里不是简单读一个字段。

它需要：

```text
Historical State
+
Contract Code
+
Storage
+
EVM Execution
```

如果你没有提前维护对应派生状态：

```text
Archive State
```

通常是最直接的来源。

### 场景 3：Reconstruction / Validation

假设你的 Indexer 出 Bug 了。

你怀疑：

```text
2024-06-01
Aave position 数据算错了
```

你可能需要一个独立 Historical State Source 去：

```text
重新查询
重新执行
验证结果
```

这时 Archive Node 可以作为：

```text
Source of Truth / Validation Source
```

这里它的价值不是给日常 Dashboard 查询。

而是：

> **作为通用、可回溯的历史状态底座。**

## 十、一个非常重要的架构模式

很多成熟系统不会设计成：

```text
业务系统
↓
所有请求
↓
Archive Node
```

而更可能：

```text
                  Blockchain
                       │
             ┌─────────┴─────────┐
             │                   │
        Full / RPC           Archive RPC
             │                   │
             │                   │
             └─────────┬─────────┘
                       ↓
                    Indexer
                       ↓
                   Warehouse
                       ↓
              API / Dashboard
```

其中：

```text
普通 History / Logs
→ Full Node / RPC

特殊 Historical State
→ Archive

频繁分析查询
→ Warehouse
```

这叫：

> **按查询类型选择数据源。**

不要让一个系统解决所有问题。

## 十一、第二个决策：Archive RPC 还是自建 Archive？

上一课我们学过 Build vs Buy。

现在把它变成真正的决策树。

假设已经确定：

```text
我们确实需要 Historical State
```

接下来问：

### 第一问：调用量大吗？

```text
低
→ Archive RPC 更合理

高
→ 继续评估自建
```

### 第二问：是否大量 Backfill？

如果：

```text
几亿 / 几十亿 Historical State Requests
```

外部 RPC 可能出现：

```text
费用
Rate Limit
吞吐限制
```

自建动力增加。

### 第三问：是否要求基础设施自主可控？

例如：

```text
关键生产系统
不能依赖单一 Provider
```

那么可能采用：

```text
Self-hosted
+
Provider fallback
```

### 第四问：团队有没有 Node Operations 能力？

如果没有：

```text
Archive Node
```

本身可能成为巨大工程负担。

所以 Build vs Buy 不是：

```text
硬件成本
vs
API Price
```

而是：

```text
Total Cost of Ownership
```

即：

```text
TCO
=
Hardware
+
Cloud
+
Engineer
+
Operations
+
Recovery
+
Risk
```

## 十二、一个完整的需求决策框架

以后遇到任何类似问题，可以走下面这条路径：

```text
业务需求
↓
Step 1
需要 History 还是 State？
↓
History
→ Block / Tx / Log / Event
→ 普通 RPC / Indexer

State
↓
Step 2
Current 还是 Historical？
↓
Current
→ Full Node / Normal RPC

Historical
↓
Step 3
Warehouse 是否已经保存派生历史状态？
↓
YES
→ Warehouse

NO
↓
Step 4
是否真的需要通用 Historical State Queryability？
↓
YES
→ Archive Capability

↓
Step 5
请求规模 / SLA / 成本 / 团队能力
↓
Archive RPC
or
Self-hosted Archive
or
Hybrid
```

这是本课最重要的一张图。

## 十三、三个实际案例

### Case A：DEX Dashboard

需求：

```text
过去两年的 Uniswap
Daily Swap Volume
```

判断：

```text
Swap Event
↓
Historical Logs
↓
Indexer
↓
dex_swaps
↓
daily_volume
```

结论：

```text
Archive State
不是必要条件
```

### Case B：Wallet Historical Balance API

需求：

```text
用户输入：
address + date

返回当日 ETH Balance
```

如果你已经有：

```text
daily_wallet_balance
```

结论：

```text
Warehouse
```

如果没有任何历史余额数据：

```text
Archive RPC
```

可能用于 Backfill。

注意这里甚至可能是：

```text
Archive RPC
↓
一次性 Backfill
↓
Warehouse
```

之后生产查询就不再打 Archive RPC。

这是非常常见的工程思路。

### Case C：链上研究工具

需求：

研究员可以输入：

```text
contract
function
block_number
parameters
```

系统执行：

```text
historical eth_call
```

这种需求很难提前派生。

因为查询空间：

```text
几乎无限
```

所以：

```text
Archive State Queryability
```

就是核心产品能力。

## 十四、Blockchain Data Engineer 的思维变化

学习到这里，你应该逐渐避免这种判断：

```text
需求涉及历史
→ Archive Node
```

而改成：

```text
需求涉及历史
↓
历史的什么？
```

具体问：

```text
Block？
Transaction？
Log？
Event？
State？
Derived State？
```

然后再决定：

```text
Node
RPC
Indexer
Warehouse
Archive
```

这才是数据基础设施工程师的判断方式。

## 十五、Module 6 到现在已经形成完整链条

第 1 课：

```text
History
≠
Historical State
```

第 2 课：

```text
Pruning
→ Historical State 不再可直接查询
```

第 3 课：

```text
Historical State Retention
→ Archive Cost
```

第 4 课：

```text
不是所有 Historical Data
都需要 Archive

而是根据：
Query Semantics
+
Existing Derived Data
+
Cost
+
Scale

做 Infrastructure Decision
```

所以 Archive Node 的核心问题其实已经从：

> Archive Node 是什么？

变成了：

> **我到底什么时候应该为 Historical State Queryability 付钱？**

这才是这一 Module 对 Blockchain Data Engineer 最重要的能力。

## 理解检查

### 问题 1

业务提出：

> “我们需要查询每个钱包过去一年的每日 ETH Balance，因此必须部署 Archive Node。”

你是否同意？

请分别讨论：

```text
A. 数仓目前什么历史 Balance 数据都没有

B. 数仓已经每天保存 wallet_balance_snapshot
```

两种情况下的判断。

### 用户回答

问题 1：

情况 A：数仓目前什么历史 balance 数据都没有，是否同意部署 archive node？
不同意。这种情况下，该需求属于可预测类型数据，调用 Archive Node RPC 成本是可控的。过去一年每日 ETH balance 可以认为是一次性需求，因为我们获取到数据后，完全可以保存在自建的数据库里，后续如果再需要查询，直接查数据库即可，不需要再去查 archive node。

情况 B：数仓每天已经保存了 wallet balance snapshot。
这种情况下更不需要部署 archive node，因为数据在这张表里就已经保存了，直接查询表是最简便的方式

### 老师判断与校准

✅ 通过。A 的判断尤其关键：需求本身属于 Historical State，但这不等于需要长期自建 Archive Node。更合理的做法是使用 Archive RPC 作为初始 Historical State Source 做一次性或批量 Backfill，将结果写入自建 Warehouse / Snapshot 表；之后日常查询直接读取派生数据。B 中既然 `wallet_balance_snapshot` 已经存在，则生产查询直接使用 Warehouse，Archive 更没有必要成为主查询路径。

### 问题 2

下面两个需求：

```text
A.
Dashboard 每天统计
Uniswap Volume

B.
研究员可以任意输入
contract + function + block_number
执行 historical eth_call
```

哪个更适合 Warehouse，哪个更真正依赖 Archive State？为什么？

### 用户回答

问题二：A需求更适合warehouse，B需求更真正依赖archive state。
因为 A 需求是做每日统计的，完全可以将每日数据保存到 warehouse 里面，这是最经济简便的方式。

而 B 需求是对任意 block 的调用，属于研究型的需求，事前不可预测到底要查询哪个 block、哪个余额、哪个账户或哪个 contract，所以它更依赖 archive state 来 backfill 得到 historic state

### 老师判断与校准

✅ 通过。A 是稳定、重复、可预计算的分析型需求，更适合 `Historical Logs → Indexer → Warehouse`；B 的查询空间不可提前穷举，且本质上是对任意旧高度执行 Historical `eth_call`，因此真正依赖通用 Historical State Queryability。补充一点：这里 Archive State 的主要作用不是把结果“backfill 成固定表”，而是作为研究型查询的直接 Historical State Source。

### 问题 3

一家公司已经确认确实需要大量 Historical State Query，正在考虑：

```text
方案 A：购买 Archive RPC
方案 B：自己运行 Archive Node
```

除了“API 费用和硬件价格”，你认为至少还应该比较哪三个因素？

### 用户回答

问题 3：还需要考虑请求规模、SLA、成本、团队能力等因素

### 老师判断与校准

✅ 通过。需要比较的至少包括：请求规模与 Backfill 强度、SLA / 延迟与可用性要求、团队的 Node Operations 能力、Provider Rate Limit 与吞吐限制、外部依赖 / Vendor Risk、故障恢复成本以及长期 Total Cost of Ownership。也就是说 Build vs Buy 应比较完整 TCO，而不是只比较 API 单价与机器价格。

## 重点总结

1. Historical Data 不等于 Historical State；旧 Block、Transaction、Log、Event 查询不因为“旧”就自动需要 Archive State。
2. Historical State 需求也不等于长期自建 Archive Node；如果需求稳定、可预计算，可以用 Archive RPC 做 Backfill，再把结果沉淀成 Warehouse Snapshot。
3. Archive Node / Archive RPC 更适合作为通用 Historical State Source；Warehouse 更适合固定、高频、分析型的派生历史数据查询。
4. Ad-hoc Historical Query、任意 Historical `eth_call`、历史状态重建与独立验证，是 Archive State 最有价值的典型场景。
5. 数据源选择应从 Query Semantics 出发：History / Log → 普通 RPC 或 Indexer；已有 Derived State → Warehouse；真正通用 Historical State → Archive Capability。
6. 确认需要 Archive Capability 后，再按请求规模、SLA、Backfill、团队能力、外部依赖和 TCO 决定 Archive RPC、Self-hosted Archive 或 Hybrid。
