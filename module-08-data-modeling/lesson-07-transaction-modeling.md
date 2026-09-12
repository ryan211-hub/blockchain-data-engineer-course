# 第7课｜Transaction 建模：Transaction Fact 与 Transfer / Swap 的 Grain 边界

## Lesson Contract

【Blockchain Data Engineer 视角】

所属 Module：Module 8 — 数据建模。

本课核心问题：

> Transaction 本身应该怎样建模？它和 Transfer、Swap 都发生在同一笔链上交易里，为什么仍然必须分成不同 Fact？

学完本课，你应该能够：

1. 给 `fact_transactions` 定义正确 Grain。
2. 设计最小 Transaction Fact 字段与 Key。
3. 区分 Transaction-level、Transfer-level、Swap-level 三种 Grain。
4. 判断哪些字段属于 Transaction，而不应该复制到下游事件 Fact。
5. 理解“一笔 Transaction 包含很多业务事件”为什么不等于“这些业务事件就是 Transaction”。
6. 根据 Query Semantics 决定从 Transaction、Transfer 还是 Swap Fact 出发。

本课明确不展开：

- EIP-1559 的完整 Gas 定价细节；
- Trace / Internal Call 的完整建模；
- Account Abstraction；
- MEV；
- DEX Router 的复杂路径识别；
- ETL / Streaming 实现。

---

## 一、先纠正一个最常见的直觉

很多人看到链上数据，会自然地认为：

```text
Transaction
= 一次完整业务交易
```

但这个理解只对了一半。

【协议 / Ethereum 视角】

Transaction 是：

> 用户签名并提交给 Ethereum 的一次状态转换请求。

例如 Alice 发起：

```text
USDC → WETH → DAI
```

从用户视角看，这可能只是：

```text
一次 Swap
```

但从 Ethereum 执行结果看，这一笔 Transaction 里可能发生：

```text
1 Transaction

2 Pool Swaps

4 Token Transfers

若干 Logs

若干 Internal Calls
```

所以我们必须先建立：

```text
Transaction
≠
Transfer
≠
Swap
```

它们可以属于同一次链上执行，但不是同一个 Business Object。

---

## 二、Transaction 自己也是一个 Fact

上一课我们有：

```text
fact_token_transfers
fact_pool_swaps
```

现在补上：

```text
fact_transactions
```

它的 Grain：

> 一行 = 某条链上的一笔 canonical Transaction。

因此最自然的 Key：

```text
(chain_id, tx_hash)
```

注意这里和 Transfer 不一样。

Transfer 的 Grain 是：

```text
one Transfer Event
```

所以需要：

```text
(chain_id, tx_hash, log_index)
```

而 Transaction 本身：

```text
一个 tx_hash
→ 就代表一笔 Transaction
```

在多链场景再加：

```text
chain_id
```

作为 Namespace。

---

## 三、最小 Transaction Fact 可以放什么？

可以先设计：

```text
fact_transactions
────────────────────
chain_id

block_number
block_time

tx_hash
tx_index

from_address
to_address

nonce
value

gas_limit
gas_used

status
```

这里先不追求完整。

关键是先看字段语义。

---

## 四、哪些字段明显属于 Transaction Grain？

例如：

```text
tx_hash
from_address
to_address
nonce
value
gas_limit
gas_used
status
```

这些都描述：

> 这一笔 Transaction 本身。

例如：

```text
gas_used
```

是 Transaction 执行最终消耗的 Gas。

它不是某一条 Transfer 的 Gas。

也不是某一个 Pool Swap 的 Gas。

这是非常重要的 Grain 边界。

---

## 五、为什么 gas_used 不能直接复制进 Transfer Fact？

假设：

```text
tx_hash = 0xABC
gas_used = 150000
```

这笔 Transaction 产生：

```text
Transfer #1
Transfer #2
Transfer #3
```

如果你把：

```text
gas_used = 150000
```

复制到三行：

```text
fact_token_transfers
```

就得到：

```text
Transfer #1 → gas_used = 150000
Transfer #2 → gas_used = 150000
Transfer #3 → gas_used = 150000
```

然后分析师执行：

```sql
SUM(gas_used)
```

得到：

```text
450000
```

但真实 Transaction Gas：

```text
150000
```

这就是典型的：

```text
Mixed Grain / Metric Duplication
```

所以：

> Transaction-level Measure 不应该无条件复制到 Event-level Fact。

---

## 六、Transaction 和 Transfer 是一对多关系

关系很自然：

```text
fact_transactions
      1
      │
      │
      N
fact_token_transfers
```

关联字段：

```text
chain_id
tx_hash
```

例如：

```text
fact_transactions
────────────────
Ethereum
0xABC
Alice
Router
gas_used = 150000
```

对应：

```text
fact_token_transfers
────────────────────
0xABC / log_index 3
0xABC / log_index 5
0xABC / log_index 8
```

所以：

```text
Transaction
= Parent Execution Context

Transfer
= Child Event
```

这里说 Parent / Child 是建模关系，不是数据库必须真的做外键约束。

---

## 七、Transaction 和 Swap 同样是一对多

例如：

```text
Alice
USDC → WETH → DAI
```

一笔 Transaction：

```text
tx_hash = 0xXYZ
```

内部经过：

```text
Pool A
USDC → WETH

Pool B
WETH → DAI
```

则：

```text
fact_transactions
1 row
```

而：

```text
fact_pool_swaps
2 rows
```

所以：

```text
Transaction Grain
= one transaction

Pool Swap Grain
= one pool swap execution
```

这两者都正确。

---

## 八、Transaction.value 特别容易被误解

这是 Ethereum 数据分析中一个典型坑。

假设 Alice 用 USDC Swap：

```text
1000 USDC → WETH
```

那么：

```text
Transaction.value
```

很可能是：

```text
0
```

为什么？

【EVM / Protocol 视角】

`Transaction.value` 表示：

> 这笔 Transaction 随调用发送了多少原生 ETH。

它不是：

```text
这笔交易的“经济价值”
```

也不是：

```text
ERC-20 Token amount
```

所以：

```text
Transaction.value = 0
```

不代表：

```text
这笔 Transaction 没有发生资产转移
```

因为 ERC-20 Transfer 是：

```text
Contract State Change + Event Log
```

不是 `Transaction.value`。

---

## 九、所以 Transaction Fact 不能代替 Transfer Fact

假设你只有：

```text
fact_transactions
```

字段：

```text
tx_hash
from_address
to_address
value
```

你想问：

> Alice 今天转出了多少 USDC？

光靠 Transaction Fact 很可能回答不了。

因为：

```text
Transaction.to
```

可能只是：

```text
Uniswap Router
```

而真正发生的 Token Transfer：

```text
Alice → Pool
1000 USDC
```

记录在：

```text
ERC-20 Transfer Event
```

所以：

```text
Transaction Fact
```

回答的是：

> 谁发起了哪一笔链上执行请求？

而：

```text
Transfer Fact
```

回答的是：

> 哪个 Token 从哪里移动到了哪里？

---

## 十、Transaction.to 也不等于业务对手方

这同样很关键。

例如 Alice：

```text
Alice
  ↓
Uniswap Router
  ↓
USDC/WETH Pool
```

Transaction：

```text
from = Alice
to   = Router
```

但业务分析可能真正关心：

```text
Pool
```

甚至：

```text
最终 Token Receiver
```

所以：

```text
Transaction.to
```

只是：

> Transaction 的直接调用目标。

它不一定是：

```text
最终资金接收方
```

也不一定是：

```text
业务 Counterparty
```

因此不能用：

```text
Transaction.from / to
```

直接替代所有业务参与者字段。

---

## 十一、status 属于谁？

假设：

```text
status = 0
```

表示 Transaction Revert。

这属于：

```text
Transaction Fact
```

因为它描述：

> 整笔 Transaction 最终执行是否成功。

如果 Transaction Revert：

```text
state changes
```

不会最终保留。

正常情况下：

```text
successful canonical Transfer / Swap facts
```

也不应该把 reverted execution 当作成功业务事实保存。

但这是：

```text
Indexer / Data Quality
```

层面的处理规则。

本课只需要理解：

> status 是 Transaction-level execution result。

---

## 十二、Transaction 与 Receipt 的关系

【RPC / Indexer 视角】

Ethereum 里常见两个来源：

```text
eth_getBlockByNumber
→ Transaction

eth_getTransactionReceipt
→ Receipt
```

Transaction 提供：

```text
from
to
nonce
value
gas
input
```

Receipt 提供：

```text
status
gasUsed
contractAddress
logs
```

但到了：

```text
Normalized Fact
```

我们不一定要机械复制 RPC 对象。

可以把：

```text
Transaction + Receipt
```

合并成一个稳定的：

```text
fact_transactions
```

例如：

```text
tx_hash
from_address
to_address
value
gas_limit
gas_used
status
```

这再次说明：

> RPC Schema ≠ Analytical Data Model。

---

## 十三、Transaction Fact 的 Key 为什么不是 block_number + tx_index？

理论上：

```text
(chain_id, block_number, tx_index)
```

也可以定位 canonical block 中的一笔 Transaction。

但更常见的 Source Identity 是：

```text
(chain_id, tx_hash)
```

原因是：

```text
tx_hash
```

是 Transaction 自身稳定的链上标识。

而：

```text
block_number
tx_index
```

更像：

> 它被包含在哪个 Block、处于 Block 中什么位置。

所以建模时通常会保留：

```text
block_number
tx_index
```

但 Unique Key 更自然地选择：

```text
(chain_id, tx_hash)
```

---

## 十四、Reorg 会不会推翻这个判断？

不会改变 Grain，但会影响 canonical 状态。

例如某笔交易：

```text
tx_hash = 0xABC
```

先进入 Block 100：

```text
block_hash = old_branch
```

后来 Reorg 后：

```text
old block orphaned
```

交易可能：

```text
重新进入新 Block
```

或者：

```text
暂时不再 canonical
```

这属于 Module 12 的 Data Quality / Reorg 管理。

在模型语义上仍然：

```text
Transaction Grain
= one Transaction
```

只是你还需要维护：

```text
canonical / orphaned
```

等状态。

---

## 十五、不要把所有执行结果塞进 fact_transactions

一个常见错误是：

```text
fact_transactions
```

不断增加：

```text
token_in
token_out
transfer_amount
swap_amount
nft_id
bridge_protocol
lending_action
...
```

最后变成：

```text
万能链上交易表
```

问题是：

一笔 Transaction 可以同时：

```text
Transfer
Swap
Mint
Burn
Stake
Bridge
```

这些事件：

```text
Business Object 不同
Grain 不同
字段语义不同
```

所以应该：

```text
fact_transactions
fact_token_transfers
fact_pool_swaps
...
```

分别建模。

---

## 十六、什么时候从 Transaction Fact 出发查询？

需求：

> Alice 今天发起了多少笔 Transaction？

从：

```text
fact_transactions
```

过滤：

```text
from_address = Alice
```

---

需求：

> Alice 今天支付了多少 Gas？

主要从：

```text
fact_transactions
```

统计：

```text
gas_used
gas_price / effective_gas_price
```

---

需求：

> Alice 今天收到了多少 USDC？

应该从：

```text
fact_token_transfers
```

而不是 Transaction。

---

需求：

> Alice 今天做了多少次 Uniswap Pool Swap？

应该从：

```text
fact_pool_swaps
```

---

所以：

```text
Query Semantics
→ 选择 Business Object
→ 选择对应 Grain
→ 选择对应 Fact
```

---

## 十七、三张 Fact 放在一起看

```text
fact_transactions
Grain:
one canonical transaction

Key:
(chain_id, tx_hash)
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

关系：

```text
             fact_transactions
                    1
              ┌─────┴─────┐
              │           │
              N           N
              │           │
fact_token_transfers   fact_pool_swaps
```

重点不是：

> 哪张表更底层？

而是：

> 它们分别描述什么事实？

---

## 十八、为什么 Transfer 和 Swap 都可能有同样的 Key 结构？

例如：

```text
(chain_id, tx_hash, log_index)
```

既可能用于：

```text
fact_token_transfers
```

也可能用于：

```text
fact_pool_swaps
```

但不能因此说：

```text
Transfer = Swap
```

Key 只解决：

> 如何唯一定位一行？

Grain 解决：

> 这一行语义上是什么？

所以：

```text
Key
≠
Business Meaning
```

这个区分非常重要。

---

## 十九、Transaction Fact 里的 from_address 是什么身份？

【Protocol / Source Identity 视角】

Transaction：

```text
from_address
```

表示：

> 这笔 Transaction 的签名发送者。

但它不一定等于：

```text
最终资产 Owner
```

也不一定等于：

```text
业务 Entity
```

例如：

```text
Smart Wallet
Relayer
Multisig
Account Abstraction
```

都会让：

```text
transaction sender
```

和：

```text
business user
```

产生区别。

本课不展开这些复杂情况。

只需要保留原则：

> Transaction.from 是 Source-level Execution Identity，不要直接把它等同于更高层 Business Identity。

---

## 二十、本课核心心智模型

把 Transaction 加入上一课模型：

```text
dim_wallets
dim_tokens

fact_transactions
fact_token_transfers
fact_pool_swaps
```

然后记住：

```text
Transaction
= Execution Request / Execution Result Grain

Transfer
= Asset Movement Grain

Swap
= Exchange Execution Grain
```

一笔 Transaction：

```text
1 Transaction
→ 0..N Transfers
→ 0..N Swaps
```

因此：

> **Transaction 是执行上下文，不是所有业务事件的替代模型。**

---

# 理解检查

## 问题 1

一笔 Transaction：

```text
tx_hash = 0xABC
gas_used = 150000
```

内部产生 3 条 Token Transfer。

请回答：

1. `fact_transactions` 保存几行？
2. `fact_token_transfers` 保存几行？
3. 为什么不能把 `gas_used = 150000` 复制到每一条 Transfer 后再直接 `SUM(gas_used)`？

---

## 问题 2

Alice 发起：

```text
1000 USDC → WETH
```

Transaction：

```text
from = Alice
to = Uniswap Router
value = 0
```

请回答：

1. 为什么 `value = 0` 并不代表这笔交易没有经济价值？
2. 为什么不能用 `Transaction.to` 直接判断最终业务对手方？
3. 查询 Alice 实际转出了多少 USDC，应该主要查哪张 Fact？

---

## 问题 3

请分别给出下面三张表的 Grain 和 Key：

```text
fact_transactions
fact_token_transfers
fact_pool_swaps
```

然后说明：

> 为什么 `fact_token_transfers` 和 `fact_pool_swaps` 即使可能使用相同结构的 Unique Key，也仍然必须是两张不同的 Fact？