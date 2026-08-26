# 第2课｜节点的磁盘里到底保存了什么：History、State 与 Key-Value Store

## 本课目标

理解 Node Database 为什么通常采用 Key-Value Storage，区分 Node Database 与 Analytics Database，并建立 Historical Block 与 Historical State 的区别。

## 一、Node Database 不是传统关系型业务库

传统 Oracle / PostgreSQL 使用者容易自然想象：

```text
BLOCK 表
TRANSACTION 表
ACCOUNT 表
```

但 Ethereum Client 内部更接近：

```text
Storage Engine

Key                     Value
-----------------------------------------
某种 Block Key      →   编码后的 Block 数据
某种 Header Key     →   编码后的 Header
某种 Receipt Key    →   编码后的 Receipts
某种 State Key      →   编码后的 State 数据
```

核心模型是：

```text
Key → Value
```

## 二、为什么节点更偏好 Key-Value Storage

节点的核心职责不是给分析师做灵活 SQL 查询，而是：

- 高速同步区块
- 校验数据
- 执行 Transaction
- 高频更新 State
- 根据已知 Key 快速读取对象

因此其工作负载大量表现为：

```text
已知 Key
   ↓
读取 / 写入 Value
```

而不是：

```text
JOIN
GROUP BY
ORDER BY
复杂聚合分析
```

### 【视角提醒】

不是 PostgreSQL 无法保存区块链数据，而是节点工作负载通常与嵌入式 KV Storage Engine 更匹配。这是工程优化选择，而不是能力限制。

## 三、C++ map 类比

Key-Value Store 可以近似类比为 C++：

```cpp
map<Key, Value>
```

这个类比适合理解最基础的数据访问模式，但 LevelDB / RocksDB 还负责磁盘持久化、排序、压缩、写放大控制等存储引擎问题。

## 四、Node Database 与 Analytics Database

### 【节点视角】

Node Database 服务 Blockchain Client：

```text
同步
验证
执行
维护 State
```

重点关注：

- Key lookup latency
- 高频写入
- SSD IO
- 状态更新效率

### 【分析系统视角】

ClickHouse / PostgreSQL 等 Analytics Database 服务：

- Data Engineer
- Analyst
- API
- Dashboard

重点关注：

```text
SELECT
WHERE
GROUP BY
JOIN
SUM
COUNT
```

因此真正的区别不只是“KV vs 二维表”，而是：

> 两类系统针对不同 Workload 做优化。

## 五、为什么需要 Indexer

Node Database 面向协议执行，而不是分析模型。

Indexer 承担关键转换：

```text
Protocol-oriented Data
        ↓
Indexer
        ↓
Analytics-oriented Data
```

整体数据链路可以理解为：

```text
Ethereum Node
      ↓
Key-Value Storage
      ↓
RPC
      ↓
Indexer
      ↓
Postgres / ClickHouse
      ↓
SQL / Dashboard / API
```

## 六、History DB / State DB 的“逻辑”和“物理”区别

### 【逻辑视角】

可以区分 History 数据与 State 数据。

### 【物理存储视角】

不同 Client 的物理布局并不相同，不能简单假设磁盘上一定存在：

```text
history.db
state.db
```

以后看到 History DB、State DB、Chain DB、Ancient DB、Freezer 等术语时，必须先明确：

> 这是逻辑数据类型，还是某个具体 Client 的物理实现？

## 七、Historical Block 与 Historical State

查询：

```text
Block 20,000,000
```

属于 Historical Block / History 查询。

查询：

```text
Alice 最新 ETH balance
```

属于 Current State 查询。

查询：

```text
Alice 在 Block 10,000,000 时的 ETH balance
```

则属于 Historical State 查询。

关键区别：

> Historical Block ≠ Historical State。

Block 是历史账本的一部分，并不会直接保存一份可随机查询的完整 Historical State。

## 八、stateRoot 提供了什么

历史 Block Header 中包含：

```text
stateRoot
```

它表示该 Block 执行完成后整个 Ethereum State 的密码学承诺。

因此 Historical Block “知道”那个时刻对应的 State Root，但：

```text
stateRoot ≠ 完整 Historical State
```

它更像当时整个世界状态的密码学指纹，而不是所有账户余额和 Contract Storage 的完整副本。

## 九、如何重建 Historical State

理论上可以：

```text
Genesis State
   ↓
Replay Blocks
   ↓
Target Historical State
```

但工程上不一定每次都从 Genesis 开始。如果节点保留了某个更近的可用 State，也可以从该点继续 Replay。

因此真正的问题变成：

> 节点到底保存多少历史 State？

这会直接引出后面的 Pruning 与 Archive Node。

## 十、理解检查结果

### 问题1

为什么 Ethereum Client 更倾向 LevelDB / RocksDB，而不是直接使用 PostgreSQL？

**回答正确：选择 B。** 节点主要面对按已知 Key 高频读写、状态维护和同步执行，不需要复杂 SQL 分析能力。

### 问题2

Node Database 与 ClickHouse Analytics Database 最大区别是什么？

**回答正确。** 用户用 C++ `map` 类比 Node Database，并指出 Node 的主要任务是同步、验证与状态维护；ClickHouse 更适合二维表模型和分析查询。

**补充：** 更根本的区别是两者优化的 Workload 不同，而非某类数据库“能不能保存”某种数据。

### 问题3

“只要保存了从 Genesis 到今天所有 Block，就一定可以方便查询任意历史时刻余额”是否正确？

**回答正确：不对。** Historical Block 是历史账本，不等于保存了可直接读取的 Historical State。若历史 State 未保留，则需要从某个已保存 State 重放后续区块；极端情况下可以从 Genesis 开始重建，资源消耗很大。

## 本课状态

✅ 已完成
