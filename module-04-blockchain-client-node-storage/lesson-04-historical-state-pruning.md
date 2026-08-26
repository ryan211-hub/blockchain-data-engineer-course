# 第4课｜Historical State、Pruning 与 Archive Node：有了 stateRoot，为什么还不能随便查询过去的 State？

## 本课目标

理解 Historical State 与 Blockchain History 的区别，理解旧 State 的物理保存方式、Pruning 的工程意义，以及 Archive Node 为什么能够提供历史状态查询能力。

## 一、三个必须区分的问题

- What happened? → History → Transaction / Receipt / Log / Trace
- What is true now? → Current State
- What was true at Block N? → Historical State

## 二、stateRoot 不是 Historical State 本身

`stateRoot` 是某一时刻整个 State 的密码学承诺。Block Header 可以长期保存历史 `stateRoot`，但仅凭 `stateRoot` 无法恢复账户余额、nonce 或 Contract Storage 等真实状态数据。

## 三、旧 State 数据保存在哪里

从节点实现视角看，旧 State 通常不是以“每个 Block 一份完整 State 快照”的形式独立保存，而是体现为节点数据库中仍然存在的旧 Trie 节点、状态版本或历史差异数据。

逻辑上可以理解为：

```text
stateRoot@BlockN
      ↓
Historical State Structure
      ↓
Node Database
      ↓
Storage Engine
      ↓
SSD
```

具体物理表示由 Client 实现决定。

## 四、为什么不同历史 State 可以共享数据

如果一个新区块只改变 Alice 的状态，State Trie 只需要为变化路径生成新的节点，未变化分支可以复用。因此 Historical State 并不等价于“完整复制一遍世界状态”。

## 五、Pruning

Pruning 的核心是删除不再为当前执行、近期 Reorg 或节点运行所必需的旧状态数据。

```text
产生新 State
   ↓
旧状态版本暂时存在
   ↓
确认不再需要
   ↓
Pruning / GC
```

## 六、Pruning 删除的不是 Blockchain History

一个节点仍然可能保存旧 Block、Transaction、Receipt、Log 和 Block Header，但对应历史 State 所需的数据已经被删除。

因此：

```text
有 Historical Block
≠
有 Historical State
```

## 七、为什么 Full Node 要 Pruning

验证新区块主要依赖当前 State 和少量近期状态，而绝大多数旧 State 对正常跟链没有持续价值。如果永久保留所有历史状态，节点磁盘空间和数据库维护成本会持续增长。

Pruning 本质上是在：

> 历史状态查询能力 与 节点运行成本之间做取舍。

## 八、Archive Node

Archive Node 为了支持 Historical State Query，会额外保留或构建可访问历史状态的数据结构。

例如：

```text
eth_getBalance(address, historical_block)
```

这种问题需要的是 Historical State，而不是单纯的 Blockchain History。

## 九、实现视角的重要修正

Historical State 是逻辑能力，不代表所有 Client 都必须“保存每个历史 Block 的完整 Trie”。现代 Client 可以采用 snapshot、diff、index、versioned state 等不同方式提供历史状态访问能力。

## 十、理解检查结果

1. 仅有旧 `stateRoot`，如果对应状态节点已被 pruning，不能直接得到历史账户状态。回答正确。
2. “Block N 中 Alice 是否收到 USDC”属于 History；“Block N 执行后 Alice 一共有多少 USDC”属于 Historical State。回答正确，经术语校准后掌握。
3. Pruning 用于控制历史状态带来的存储与数据库维护成本，使 Full Node 不必承担 Archive Node 的长期存储负担。回答正确。

## 本课状态

✅ 已完成。
