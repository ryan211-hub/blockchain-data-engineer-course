# 第2课｜Full / Pruned Node vs Archive Node：Pruning 到底删掉了什么？

## Lesson Contract

【课程边界视角】

- 所属 Module：Module 6 — Archive Node
- 本课核心问题：一个普通 Full Node 明明同步、验证过全部历史 State，为什么过了一段时间以后，却不能再查询这些 State？
- 学完后应能回答：Full Node、Pruned Node、Archive Node 三者真正的区别在哪里，以及为什么 Pruning 是理解 Archive Node 的关键。
- 本课明确不展开：Geth / Erigon / Reth 各自具体的 pruning 数据结构；Trie Node 在磁盘上的具体 GC 算法；LevelDB / RocksDB Compaction；Indexer 如何保存 Snapshot。这些分别属于 Client 内核或后面的 Module 7。

## 一、先解决一个很容易产生的误解

上一课我们说：

```text
Archive Node
→ 保留 Historical State

普通 Node
→ 通常不保留完整 Historical State
```

这里马上会出现一个问题：

> 普通 Full Node 当初不是也执行过 Block 12,000,000 吗？

当然执行过。

假设节点从 Genesis 一路同步：

```text
Block 1
↓
执行
↓
State 1

Block 2
↓
执行
↓
State 2

...

Block 12,000,000
↓
执行
↓
State 12,000,000
```

所以在同步到 12,000,000 的那个时刻，它当然拥有：

```text
State 12,000,000
```

否则它根本无法验证 Block 12,000,001。

问题不是：

> 它有没有产生过这个 State？

而是：

> **后来它有没有继续把这个旧 State 留下来？**

这就是 Pruning 要解决的问题。

## 二、Pruning 是什么？

【节点存储视角】

Pruning 可以先理解成：

> **节点确认一些旧状态数据以后不再需要用于当前正常运行，于是把这些旧状态数据清理掉。**

非常粗略地画：

一开始：

```text
State 100
State 101
State 102
State 103
State 104
State 105
```

链继续运行到很后面：

```text
Current State = State 1000
```

普通节点真正高频需要的是：

```text
State 1000
```

以及为了安全运行、近期重组等原因需要的一部分近期状态。

它通常没有必要永远保留：

```text
State 100
State 101
State 102
...
```

于是旧状态相关数据逐渐被 Prune。

所以逻辑上变成：

```text
History
────────────────────────
Block 1
Block 2
Block 3
...
Block 1000
        ↑
很多仍然可以保留


State
────────────────────────
× State 1
× State 2
× State 3
...
recent states
Current State 1000
```

这里最重要的是：

> **Pruning 的重点不是“把整个旧 Block 删掉”，而是减少不再需要的旧 State 数据。**

这正好解释了上一课那个现象：

```text
eth_getBlockByNumber(12,000,000)
        ↓
可能成功

eth_getBalance(Alice, 12,000,000)
        ↓
可能失败
```

因为：

```text
Block History 还在

但

Historical State 已被 Pruned
```

## 三、为什么 Node 要这么做？

这里你可以从数据库工程的直觉出发。

假设 Ethereum 每一个 Block 都产生一些 State Change：

```text
Block N

Alice:
100 ETH → 90 ETH

Contract A:
slot 5: 100 → 120

Contract B:
slot 17: 0 → 1
```

然后：

```text
Block N+1
Block N+2
Block N+3
...
```

不断发生变化。

如果要求节点：

> 每一个历史高度都永久具有完整可查询能力。

那么这些历史变化相关的数据就不能随便删除。

随着链不断运行：

```text
Historical State Data
↑
↑
↑
持续增长
```

这就带来：

```text
更多磁盘
+
更多 I/O
+
更多数据库维护成本
+
更长同步 / 运维过程
```

而普通 Ethereum 用户运行 Node 的主要目标通常不是帮数据分析师查询 5 年前 Alice 在某一个 Block 的余额，它首先需要：

```text
验证新区块
维护当前 State
参与网络
回答常规 RPC
```

于是就产生了一个很合理的工程选择：

```text
我已经验证过历史

≠

我必须永久保留所有历史 State
```

这实际上就是一种：

> **Storage Cost vs Historical Queryability 的 trade-off。**

## 四、这里一定要区分 Full Node 和 Archive Node

“Full Node”这个词特别容易造成误解。

很多人第一次看到 Full，会自然理解成：

> Full = 什么历史数据都有。

实际上不应该这么理解。

【协议验证视角】

Full Node 的“Full”更接近：

> 它独立执行和验证区块，并维护足够的数据来验证当前链状态。

而不是：

> 它是一个永久保存所有 Historical State 的数据仓库。

因此逻辑上可以这样理解：

```text
Full / Pruned Node

完整验证链
        ✅

维护 Current State
        ✅

保存历史 Block / Tx
        通常可以

任意历史 State 查询
        ❌ 不保证
```

而：

```text
Archive Node

完整验证链
        ✅

维护 Current State
        ✅

保存 History
        ✅

保留 Historical State Queryability
        ✅
```

所以：

> **Archive Node 不是“比 Full Node 更完整地验证区块”。**

这一点很重要。

Archive Node 和普通 Full Node 的核心差异，不是：

```text
谁验证得更认真
```

而是：

```text
谁保留了更多历史状态数据
```

## 五、一个银行系统类比

假设银行系统每天有 Account Balance，每天结束之后生成一次余额。

第 1 天：

```text
Alice = 10,000
```

第 2 天：

```text
Alice = 8,000
```

第 3 天：

```text
Alice = 12,000
```

到了第 1000 天。

一种系统设计是：

```text
交易流水：永久保存

当前余额：
Alice = 23,000

历史余额 Snapshot：
只保留最近 30 天
```

那么它完全可以正常处理 Alice 当前余额，也能查询某天做过哪些交易。但是你突然问：

```text
2022-03-17
Alice 当天结束时余额是多少？
```

它可能不能直接回答。

因为历史余额 Snapshot 已经删除了。

这并不表示银行从来不知道 Alice 当时余额是多少，而是当时知道，但后来为了控制存储成本，没有永久保留那个 Snapshot。

这和 Pruning 的心智模型非常接近。

## 六、但 Archive Node 不是每天保存一整份 State Copy

看到：

```text
State 1
State 2
State 3
...
State 25,000,000
```

可能会理解成 Archive Node 磁盘上真的存在：

```text
Ethereum State 完整副本 × 25,000,000
```

不是。这会大得离谱。

【逻辑视角 vs 物理实现视角】

课程中画：

```text
State at Block 100
State at Block 101
State at Block 102
```

表达的是：这些历史高度具有可恢复 / 可查询的 State 信息。

它不是在描述底层磁盘物理布局。实际 Client 会利用状态之间高度重复的特性，通过不同的数据结构和历史状态机制来保存。这个实现细节现在不下钻。

作为 Blockchain Data Engineer，这一课只需要建立：

```text
Archive Node
不是
每个 Block 保存一个完整数据库 Copy

而是
保留足够历史信息
使 Historical State 可查询
```

## 七、Pruning 到底导致什么 RPC 能力变化？

假设：

```text
Current Block = 25,000,000

目标 Block = 12,000,000
```

请求 A：

```text
eth_getBlockByNumber(12,000,000)
```

需要 Historical Block。如果历史 Block 仍然保留，可以成功。

请求 B：

```text
eth_getTransactionReceipt(tx)
```

主要需要历史 Receipt，是否可查取决于节点历史数据配置。

请求 C：

```text
eth_getBalance(
    Alice,
    12,000,000
)
```

需要 Historical State。如果对应 State 已 Pruned，则无法直接查询。

请求 D：

```text
eth_getStorageAt(
    Contract,
    slot,
    12,000,000
)
```

同样需要 Historical State → Contract Storage。

请求 E：

```text
eth_call(
    USDC.balanceOf(Alice),
    block=12,000,000
)
```

需要：

```text
Historical State
↓
Contract Code
+
Contract Storage
↓
EVM Execution
```

因此也需要相应的 Historical State 能力。

## 八、所以 Pruning 本质上改变的是什么？

不是：

```text
节点知道不知道过去发生了什么
```

而主要是：

```text
节点还能不能直接进入过去某个时刻的世界状态
```

这是本课最重要的一句话。

可以把 Blockchain 想成一部电影。

History 是：

```text
完整电影录像
```

Current State 是：

```text
电影当前这一帧
```

Historical State 是：

```text
可以直接拿到电影第 38 分钟第 17 秒对应的完整世界状态
```

Pruned Node 更像：

```text
录像还在
+
当前帧还在

但过去每一帧对应的“完整可查询状态”
没有全部保留
```

Archive Node 则专门提供更强的过去状态访问能力。

## 九、Blockchain Data Engineer 为什么必须理解这个区别？

【Blockchain Data Engineer 视角】

以后做数据需求时，如果产品经理说：

> 给我查 Alice 在 2022 年 5 月 1 日的 ETH Balance。

不能立刻想到查 transactions，因为：

```text
Balance
=
State
```

应该先判断：

```text
我们自己是否已经维护历史余额 Snapshot？
```

如果有：

```text
Warehouse
↓
wallet_balance_snapshot
```

直接查。

如果没有，再考虑：

```text
Archive Node / Archive RPC
```

但如果产品经理说：

> 给我统计 2022 年 5 月 Uniswap 的 Swap Volume。

那很可能：

```text
Historical Logs
↓
Indexer
↓
swap fact
```

就够了，不一定需要 Historical State。

所以一个 Blockchain Data Engineer 的判断路径应该逐渐变成：

```text
业务问题
↓
这是 History 还是 State？
↓
如果是 State：
Current 还是 Historical？
↓
Historical State 是否已经在数仓中派生保存？
↓
没有的话，是否需要 Archive Node？
```

这才是我们学 Archive Node 的真正目的，不是为了研究 Client GC。

## 十、本课核心模型

把 Module 6 前两课合起来，现在形成：

```text
                    Ethereum Node
                         │
            ┌────────────┴────────────┐
            │                         │
         History                    State
            │                         │
     Block / Tx / Log          Current State
                                      │
                              Historical State
                                      │
                         ┌────────────┴───────────┐
                         │                        │
                      Pruned                  Retained
                         │                        │
                  Full/Pruned Node          Archive Node
```

最终记住四句话：

```text
Full Node
≠
All Historical State Node
```

```text
Pruning
=
放弃部分旧 State 数据
以降低节点长期存储成本
```

```text
Pruned Historical State
≠
Historical Block 被删除
```

以及最关键的：

```text
Archive Node 的核心差异
=
Historical State Retention
+
Historical State Queryability
```

## 理解检查

### 问题 1

一个 Full Node 在同步 Block 12,000,000 时，明明拥有过 `State 12,000,000`。为什么今天链已经到 Block 25,000,000 后，它却可能无法再执行 `eth_getBalance(Alice, 12,000,000)`？请用 Pruning 来解释。

### 用户回答

“问题一：

Full node 确实曾经拥有过 1200 万区块高度时候的 state，但是当前的 block 已经到了 2000 多万的高度，那之前的 state 很有可能已经被 pruning（修剪）了。

所以，它很可能没有办法直接获取到历史高度时的 state。”

### 老师判断与校准

✅ 通过。核心逻辑完全正确：Full Node 在同步到 Block 12,000,000 时确实生成并拥有过对应 State，但随着链继续推进，旧 State 数据可能被 Pruning 清理。今天还能访问历史 Block，并不意味着仍然保留那个高度的 Historical State。因此它可能无法直接执行 `eth_getBalance(Alice, 12,000,000)`。这里要保留的关键区分是：“曾经拥有 / 曾经验证过” ≠ “今天仍然保留并可查询”。

### 问题 2

下面两个 Node 都完整验证了 Ethereum：

```text
Node A
完整验证历史区块
保存历史 Block / Transaction
只维护 Current / Recent State

Node B
完整验证历史区块
保存历史 Block / Transaction
并保留 Historical State Queryability
```

A 和 B 在“区块验证正确性”上是否一定有高低之分？它们真正的核心区别是什么？

### 用户回答

“A 和 B 在区块验证正确性上没有差异，没有高低之分。但是，他们的 Node 功能目标是不一样的：

1. Node A：它是一个 Full Node 节点，目标是维护 current 或者是 recent state
2. Node B：作为一个 Archive Node，它可以保留 historical state”

### 老师判断与校准

✅ 通过。A 与 B 都可以完整执行并验证 Ethereum 区块，因此不能把 Archive Node 理解成“验证得更完整”或“共识正确性更高”的 Node。真正的核心差异在数据保留策略和查询能力：A 主要维护 Current / Recent State，并允许旧 State 被 Pruned；B 保留足够的 Historical State，使过去高度仍具有 Historical State Queryability。

### 问题 3

数据平台收到两个需求：

```text
A.
统计 2023 年所有 USDC Transfer 的总金额

B.
查询 Alice 在
Block 17,000,000
时的 ETH Balance
```

假设数仓目前还没有维护历史 Balance Snapshot。哪个需求更可能真正要求使用 Archive State 能力？为什么？

### 用户回答

“B 需求更可能真正要求使用 archiveState 能力。因为 B 需求是想要查询在历史 block 时的相应 state；而 A 需求是要统计在 2023 年所有 transfer 的金额，这个需求和 state 无关，和历史 log 有关”

### 老师判断与校准

✅ 通过。B 查询的是指定历史 Block 下的 ETH Balance，本质是 Historical State Query；在数仓没有历史 Balance Snapshot 的前提下，更可能需要 Archive State / Archive RPC。A 统计 USDC Transfer 总金额，主要可以从 ERC-20 `Transfer` Historical Logs / Events 构建事实数据完成，并不因为范围很旧就自动变成 Historical State Query。

## 重点总结

1. Full Node 曾经拥有过某个历史 State，不代表今天仍然保留它。
2. Pruning 主要清理不再需要的旧 State 数据，以降低长期存储与维护成本；它不等于删除全部历史 Block。
3. Full / Pruned Node 与 Archive Node 在区块验证正确性上没有天然高低之分；核心区别是 Historical State Retention 与 Queryability。
4. `eth_getBlockByNumber(old)` 与 `eth_getBalance(address, old)` 属于不同类别的问题：前者主要是 History，后者是 Historical State。
5. 数据工程师判断是否需要 Archive Node 时，先判断业务需求到底是在问 History、Current State 还是 Historical State，再看数仓是否已经维护了相应派生状态。