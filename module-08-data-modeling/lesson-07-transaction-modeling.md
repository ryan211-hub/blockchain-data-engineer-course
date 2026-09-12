# 第7课｜Transaction 建模：Transaction Fact 与 Transfer / Swap 的 Grain 边界

## Lesson Contract

【Blockchain Data Engineer 视角】

所属 Module：Module 8 — 数据建模。

本课核心问题：

> Transaction 本身应该如何建模？为什么 Transaction、Transfer、Swap 虽然都来自同一笔链上执行，却必须保持不同 Grain？

学完本课，你应该能够：

1. 为 `fact_transactions` 定义明确 Grain。
2. 设计 Transaction Fact 的最小字段与 Key。
3. 理解 Transaction Request 与 Receipt / Execution Result 为什么可以在同一 Transaction Grain 下合并。
4. 区分 `transaction.value`、Token Transfer、Swap 三种不同事实语义。
5. 解释为什么一笔 Transaction 可以关联多条 Transfer / Swap Fact。
6. 避免把 Transaction-level 字段错误复制到 Event-level Fact 中造成 Mixed Grain。

本课明确不展开：

- EIP-1559 的完整费率机制；
- Trace / Internal Call 的系统建模；
- Account Abstraction；
- MEV；
- DEX 路由识别算法；
- Reorg / Data Quality 修复。

---

## 一、先看一个很典型的误区

假设 Alice 发起一笔 Uniswap Swap：

```text
Alice
↓
Transaction
↓
Uniswap Router
↓
Pool A
↓
Pool B
```

最终：

```text
1000 USDC
→ WETH
→ 3000 DAI
```

这时链上可能同时出现：

```text
1 Transaction
2 Pool Swap Events
若干 Token Transfer Events
```

如果只看：

```text
tx_hash = 0x123...
```

它们都属于同一笔 Transaction。

但建模时不能说：

> 既然 tx_hash 都一样，那干脆放一张表。

因为它们的 Grain 不一样。

---

## 二、Transaction 本身是什么 Business Object？

【Protocol / User Request 视角】

Transaction 首先代表：

> 用户提交给链的一次执行请求。

比如：

```text
from = Alice
to = UniswapRouter
input = swap(...)
value = 0
nonce = 105
```

因此在分析模型中：

```text
fact_transactions
```

最自然的 Grain 是：

> 一行 = 一条 Chain 上的一笔 Transaction。

即：

```text
one transaction on one chain
```

这和上一课的：

```text
one transfer event
one pool swap execution
```

是不同 Grain。

---

## 三、Transaction 的 Source Identity 和 Key

Transaction 在 EVM 链上最自然的 Source Identity 是：

```text
(chain_id, tx_hash)
```

因此：

```text
fact_transactions
```

Unique Key：

```text
(chain_id, tx_hash)
```

这里为什么不需要 `log_index`？

因为：

```text
tx_hash
```

本身已经定位到这笔 Transaction。

而：

```text
log_index
```

用于区分这笔 Transaction 执行后产生的多条 Log。

所以：

```text
Transaction Grain
→ (chain_id, tx_hash)

Log / Transfer / Pool Swap Grain
→ (chain_id, tx_hash, log_index)
```

---

## 四、一个最小的 fact_transactions

可以先设计：

```text
fact_transactions
────────────────────
chain_id

block_number
block_time
transaction_index

tx_hash

from_address
to_address

nonce
value
input

gas_limit

status
gas_used
effective_gas_price
transaction_fee
```

这里不要先纠结所有字段。

最重要的是：

```text
Grain = one transaction
```

以及：

```text
Key = (chain_id, tx_hash)
```

---

## 五、这里出现一个很重要的问题：Transaction 和 Receipt 来自两个 RPC 对象

【RPC / Source 视角】

Ethereum RPC 中，Transaction 对象会给你：

```text
hash
from
to
nonce
value
input
gas
maxFeePerGas
...
```

Receipt 会给你：

```text
status
gasUsed
effectiveGasPrice
logs
contractAddress
...
```

从 Raw 层看：

```text
raw_transactions
```

和：

```text
raw_receipts
```

当然应该分开保留。

因为它们是不同的 Source Objects。

但到了 Normalized Fact 层，我们需要换一个视角。

---

## 六、为什么 Transaction + Receipt 可以合并到 fact_transactions？

【Data Modeling 视角】

关键仍然看 Grain。

Transaction Request：

> 一笔 Transaction。

Receipt Result：

> 还是这同一笔 Transaction 的执行结果。

它们的共同 Grain 是：

```text
one transaction
```

所以：

```text
from_address
to_address
nonce
value
```

和：

```text
status
gas_used
effective_gas_price
```

完全可以出现在：

```text
fact_transactions
```

同一行。

因为不存在 Grain 冲突。

可以理解为：

```text
Transaction Request
        +
Execution Result
        ↓
fact_transactions
```

---

## 七、这和把 Transfer 塞进 Transaction 表有什么不同？

这就是本课最重要的边界。

Transaction：

```text
1 row
```

可能产生：

```text
Transfer #1
Transfer #2
Transfer #3
```

因此：

```text
Transaction
1
:
N
Transfer
```

如果把 Transfer 字段：

```text
token_address
from_address
to_address
amount
```

硬塞进 Transaction 表，会遇到问题：

```text
一笔 Transaction 有多个 token_address
一笔 Transaction 有多个 transfer amount
```

一行根本表达不了。

如果你复制 Transaction 行：

```text
tx_hash    gas_used    transfer
0x123      100000      #1
0x123      100000      #2
0x123      100000      #3
```

那么：

```text
gas_used
```

被重复三次。

这就是：

```text
Mixed Grain
```

---

## 八、Transaction Fact 与 Transfer Fact 的关系

可以这样理解：

```text
fact_transactions
Grain:
one transaction

         1
         │
         │ tx_hash
         │
         N

fact_token_transfers
Grain:
one token transfer event
```

关联字段：

```text
chain_id
tx_hash
```

但是注意：

在 `fact_token_transfers` 中：

```text
(chain_id, tx_hash)
```

只是：

```text
Foreign / Reference Identity
```

并不是 Transfer 自己的 Unique Key。

Transfer 自己仍然需要：

```text
(chain_id, tx_hash, log_index)
```

---

## 九、Transaction Fact 与 Swap Fact 也是同样关系

```text
fact_transactions
         1
         │
         N
fact_pool_swaps
```

一笔 Transaction：

```text
Alice
USDC → DAI
```

可能路由为：

```text
Pool A
USDC → WETH

Pool B
WETH → DAI
```

于是：

```text
fact_transactions
1 row
```

```text
fact_pool_swaps
2 rows
```

这不是重复数据。

而是：

> 同一链上执行，从不同 Business Object / Grain 被建模。

---

## 十、Transaction.value 到底是什么？

这个字段特别容易误解。

【EVM / Protocol 视角】

Transaction 的：

```text
value
```

表示：

> 随这笔 Transaction 直接发送的原生 ETH 数量。

例如：

```text
Alice → Bob
1 ETH
```

Transaction：

```text
to = Bob
value = 1 ETH
```

这时：

```text
transaction.value
```

很重要。

但是：

```text
Alice swap 1000 USDC → DAI
```

通常：

```text
transaction.value = 0
```

完全正常。

因为 USDC 不是 ETH。

USDC 的移动来自：

```text
ERC-20 Transfer
```

不是：

```text
transaction.value
```

---

## 十一、所以 transaction.value ≠ Token Transfer Amount

这一点要明确固定。

```text
transaction.value
```

表示：

```text
Native Asset Transfer
```

而：

```text
fact_token_transfers.amount
```

表示：

```text
Token Contract Transfer Event
```

例如：

```text
Transaction:
Alice → USDC Contract
value = 0
```

内部执行：

```text
USDC Transfer
Alice → Bob
100 USDC
```

因此：

```text
transaction.value = 0
```

并不能推出：

> 这笔 Transaction 没有资产转移。

---

## 十二、Contract Call 里的 to_address 也容易误解

假设 Alice Swap：

```text
Alice
→ Uniswap Router
```

Transaction：

```text
from_address = Alice
to_address = Router
```

但是 Token Transfer 可能是：

```text
Alice → Pool
Pool → Alice
```

所以：

```text
fact_transactions.to_address
```

回答的是：

> 用户把 Transaction 发给哪个账户 / Contract？

而：

```text
fact_token_transfers.to_address
```

回答的是：

> 这次 Token Transfer 的接收方是谁？

字段名字可能一样：

```text
to_address
```

但语义由：

```text
Table + Grain
```

决定。

---

## 十三、这就是为什么字段名本身不能决定语义

例如都有：

```text
from_address
to_address
```

但：

```text
fact_transactions.from_address
```

表示：

```text
Transaction Sender
```

而：

```text
fact_token_transfers.from_address
```

表示：

```text
Token Transfer Sender
```

而 Swap 中：

```text
sender_address
recipient_address
```

又属于 Swap Execution 的角色。

所以不要形成：

> “字段名一样 = 同一种业务含义”

的错误认识。

更准确的是：

```text
字段语义
=
Business Object
+
Grain
+
Role
```

---

## 十四、status 应该放哪里？

`status` 来自 Receipt：

```text
1 = success
0 = failed
```

它描述的是：

> 整笔 Transaction 的执行结果。

所以它属于：

```text
fact_transactions
```

Transaction Grain。

如果一笔 Transaction：

```text
status = 0
```

意味着最终执行失败。

正常情况下，失败 Transaction 不会留下成功提交后的 canonical Event Logs 作为最终链上事件结果。

因此：

```text
status
```

是 Transaction-level Fact。

---

## 十五、gas_used 为什么也应该只放 Transaction Fact？

`gas_used` 描述：

> 整笔 Transaction 最终消耗多少 Gas。

例如：

```text
tx_hash = 0x123
gas_used = 180000
```

即使里面有：

```text
3 Transfers
2 Swaps
```

也只有：

```text
one transaction gas_used = 180000
```

如果复制到每个 Transfer：

```text
Transfer 1 → 180000
Transfer 2 → 180000
Transfer 3 → 180000
```

然后：

```sql
SUM(gas_used)
```

就得到：

```text
540000
```

这明显错误。

因此一个很实用的建模检查是：

> 这个 Measure 是属于 Transaction Grain，还是 Event Grain？

---

## 十六、transaction_fee 也是 Transaction-level Measure

简化来看：

```text
transaction_fee
≈ gas_used × effective_gas_price
```

它描述的是：

> 这笔 Transaction 最终支付的执行费用。

所以：

```text
transaction_fee
```

也属于：

```text
fact_transactions
```

而不是：

```text
fact_token_transfers
```

或：

```text
fact_pool_swaps
```

如果业务未来想分析：

> 每次 Swap 平均花了多少 Gas？

那应该根据：

```text
Swap
→ tx_hash
→ Transaction
```

关联后分析。

而不是把 Transaction Fee 复制到每个 Swap Row。

---

## 十七、那么 Transaction 是不是“最底层 Fact”？

不是。

这里不要建立层级高低误解。

Transaction、Transfer、Swap 都可以是：

```text
Normalized Fact
```

只是 Grain 不同。

例如：

```text
fact_transactions
one transaction

fact_token_transfers
one token transfer event

fact_pool_swaps
one pool swap execution
```

它们可以处于同一个建模层：

```text
Normalized Fact Layer
```

但描述不同 Business Object。

所以：

> Grain 的粗细，不等于 Layer 的高低。

这是非常重要的。

---

## 十八、一个 Transaction 可能没有 Transfer 或 Swap

比如 Alice：

```text
Alice → Bob
1 ETH
```

可能只有：

```text
fact_transactions
1 row
```

而：

```text
fact_token_transfers
0 rows
fact_pool_swaps
0 rows
```

也可能是：

```text
Contract Function Call
```

但没有 ERC-20 Transfer。

所以：

```text
Transaction
```

不是 Transfer / Swap 的容器表。

它是独立的 Fact。

---

## 十九、反过来，Transfer / Swap 必须能追溯回 Transaction

虽然它们是不同 Fact，但仍然应该保存：

```text
chain_id
tx_hash
```

因为这提供：

```text
Lineage
```

例如：

```text
fact_pool_swaps
↓ tx_hash
fact_transactions
```

你可以进一步获取：

```text
transaction sender
gas_used
status
transaction_fee
block_time
```

因此模型不是把事实拆散。

而是：

> 在正确 Grain 下拆分，再通过稳定 Identity 建立关系。

---

## 二十、本课最小模型

现在 Module 8 的核心模型进一步变成：

```text
dim_wallets
dim_tokens

fact_transactions
fact_token_transfers
fact_pool_swaps
```

关系：

```text
                    fact_transactions
                           1
                    ┌──────┴──────┐
                    │             │
                    N             N
                    │             │
        fact_token_transfers   fact_pool_swaps
              │                      │
              └───────┬──────────────┘
                      │
              dim_wallets / dim_tokens
```

核心 Grain：

```text
fact_transactions
= one transaction

fact_token_transfers
= one token transfer event

fact_pool_swaps
= one pool swap execution
```

---

## 二十一、本课核心心智模型

以后看到同一笔链上交易，不要问：

> “这些数据是不是都属于同一个 tx_hash？”

而要问：

> “我现在建模的 Business Object 是什么？”

然后：

```text
Transaction
→ one user-submitted execution request

Transfer
→ one token movement event

Pool Swap
→ one pool-level swap execution
```

它们通过：

```text
(chain_id, tx_hash)
```

建立关系。

但分别保留自己的 Grain 和 Unique Key。

本课最需要记住一句：

> **同一个 tx_hash 可以连接多个事实模型，但不能让多个 Grain 混进同一张事实表。**

---

# 理解检查

## 问题 1

一笔 Transaction：

```text
tx_hash = 0x123
```

执行后产生：

```text
3 Token Transfers
2 Pool Swaps
```

请回答：

1. `fact_transactions` 应保存几行？
2. `fact_token_transfers` 应保存几行？
3. `fact_pool_swaps` 应保存几行？
4. 为什么这三张表都不是重复数据？

---

## 问题 2

Alice 调用 USDC Contract：

```text
transfer(Bob, 100 USDC)
```

Transaction：

```text
from = Alice
to = USDC Contract
value = 0
```

请回答：

1. 为什么 `transaction.value = 0`？
2. 这是否意味着没有资产发生转移？
3. `fact_transactions.to_address` 和 `fact_token_transfers.to_address` 分别可能是谁？

---

## 问题 3

假设一笔 Transaction：

```text
gas_used = 150000
```

内部产生 3 条 Token Transfer。

有人建议把：

```text
gas_used = 150000
```

复制到三条 `fact_token_transfers` 中。

请回答：

1. 这样做有什么问题？
2. `gas_used` 更适合放在哪张表？
3. 如果以后要分析“发生过 Token Transfer 的 Transaction 平均 gas_used”，应该怎么做？