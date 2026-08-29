# 第10课｜UTXO Set：Bitcoin 如何维护可花费状态，与 Ethereum Account State 有什么不同？

## Lesson Contract

- 所属 Module：Module 4 — Blockchain Client & Node Storage
- 本课核心问题：Bitcoin 节点如何通过 UTXO Set 表示“当前可花费状态”，以及这种状态模型与 Ethereum Account State 的核心差异是什么。
- 学完后应能回答：为什么 Bitcoin 不需要账户余额表、UTXO Set 为什么属于 Current State、UTXO 模型与 Account 模型各自如何影响节点存储和数据工程分析。
- 本课明确不展开：Bitcoin Script 深入实现、Mempool 策略、共识源码、UTXO Database 内核优化。

## 一、UTXO 是什么

UTXO 是 Unspent Transaction Output，即“未花费交易输出”。它不是“账户余额”本身，而是一个尚未被其他交易作为 Input 消费的 Transaction Output 对象。

Bitcoin 并不存在协议意义上的：

```text
Alice.balance = 1 BTC
```

更接近：

```text
Alice controls:
UTXO A = 0.3 BTC
UTXO B = 0.7 BTC
```

地址余额是对这些 UTXO 的派生汇总：

```text
balance = SUM(unspent outputs)
```

## 二、Bitcoin 转账如何更新 State

如果 Alice 有 0.3 BTC 和 0.7 BTC 两个 UTXO，并向 Bob 支付 0.8 BTC，则交易会消费旧 UTXO，并创建新的 Outputs，例如 Bob 的 0.8 BTC 与 Alice 的找零。

因此 Bitcoin 的 State Transition 更接近：

```text
consume old outputs
        ↓
create new outputs
```

而 Ethereum Account Model 更接近：

```text
Alice.balance -= amount
Bob.balance   += amount
```

## 三、为什么 UTXO Set 是 Current State

完整 Blockchain Transaction History 描述“发生过什么”；UTXO Set 描述“哪些 Output 当前仍然可以被消费”。

```text
History
   ↓
按照协议规则执行
   ↓
Current State
```

在 Bitcoin 中，Current State 的核心就是 UTXO Set。节点不需要每次从 Genesis 扫描所有历史来判断某个 Output 是否已经被消费。

## 四、UTXO 与双花

双花不是 UTXO 模型独有要解决的问题，Ethereum Account Model 同样必须防止双花。

Bitcoin 中，两个交易如果尝试消费同一个 UTXO，则只有 canonical ordering 中先被有效消费的一笔可以成功；之后该 UTXO 已从 UTXO Set 中消失。

Ethereum 中，双花通过 Account State、nonce、余额检查、确定性执行与共识排序共同防止。

因此更准确的结论是：

> UTXO 与 Account Model 是两种不同的状态表示方式；双花是所有数字货币系统都必须解决的一致性问题。

## 五、Bitcoin 与 Ethereum 的核心差异

| 维度 | Bitcoin | Ethereum |
| --- | --- | --- |
| State 模型 | UTXO | Account |
| 基本状态对象 | Unspent Output | Account |
| 余额怎么得到 | 汇总 UTXO | Account 中直接记录 |
| 转账 | 消费旧 UTXO，创建新 UTXO | 修改账户状态 |
| Current State | UTXO Set | Account State |

## 六、Blockchain Data Engineer 视角

多链数据平台底层应该保留各链原生事实模型：

```text
Bitcoin → UTXO / Input / Output
Ethereum → Account / State / Log
        ↓
Normalized Layer
        ↓
balance / asset / transfer / position
```

统一业务语义不等于抹掉底层协议差异。Bitcoin 的 `address_balance` 属于派生业务视图，而不是协议原生状态对象。

## 理解检查与回答

### 问题 1
Alice 收到 0.2、0.3、0.5 BTC 三个 Output，0.3 BTC 对应 UTXO 已被消费。当前余额是多少？为什么不能理解为节点保存 `Alice.balance`？

**回答：** 0.7 BTC。Bitcoin 的设计里没有账户余额这个协议原生概念，使用 UTXO Set 表示当前可花费状态，因此不能把 Ethereum 式 `balance` 状态直接套到 Bitcoin。

**判断：** 正确。

### 问题 2
为什么 UTXO Set 是 Current State，而完整 Transaction History 不是？

**回答：** Transaction History 包含全部历史交易，而 Current State 表示区块链在当前时点的有效状态，两者不能等同。

**校准：** Current State 更严格的表达是“由历史交易按照协议规则执行后推导出的当前有效状态”，不是简单的“历史当前时点的样子”。

### 问题 3
多链数据平台应强制统一为 address_balance，还是底层保留原生事实、上层再统一？

**回答：** 选择后者。Raw / 底层事实层应尽量保留原系统事实，上层再通过 Normalized 处理形成统一业务语义与主题数据。

**判断：** 正确。

## 本课重点总结

1. UTXO 是“未花费交易输出对象”，不应简单翻译成“未花费余额”。
2. Bitcoin 的 Current State 是 UTXO Set；地址余额是派生结果。
3. Ethereum 的 Current State 是 Account State。
4. UTXO 与 Account Model 都可以防止双花，关键依赖 State Validation + Transaction Ordering + Consensus。
5. 多链数据平台底层应尊重原生事实模型，上层再统一业务语义。

## 本课状态

✅ 已完成。
