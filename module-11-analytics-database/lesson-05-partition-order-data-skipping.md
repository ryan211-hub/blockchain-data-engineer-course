# Module 11 · 第 5 课
## Partition、Order Key、Data Skipping 与大表查询

### Lesson Contract

所属 Module：Module 11 — 分析数据库

本课核心问题：

> 当一张分析表已经有几十亿、几百亿行时，数据库为什么还能把很多查询控制在可接受范围内？关键不是“把整张表扫得更快”，而是尽量少读不相关的数据。

学完以后，你应该能够解释：

- Partition 的核心作用是什么。
- Order Key 为什么会影响大表查询效率。
- Data Skipping 是什么，以及它和 Index 的思路有什么相似与不同。
- 为什么 Partition Key 和 Order Key 不能随便选。
- 为什么 Blockchain 数据常见的时间范围、Token、Wallet、Pool 查询会影响物理设计。
- 为什么一个好的查询条件，只有和物理布局匹配时，才能真正减少扫描量。
- 如何为 Transfer / Swap 这类大 Fact 设计基础的 Partition + Order Key。

本课不深入 ClickHouse MergeTree 内部数据结构、Sparse Index 实现细节、Granule 内部算法；只建立 Data Engineer 需要掌握的物理设计心智模型。

---

## 一、先明确一个问题：列式存储还不够

上一课我们知道：

> ClickHouse 适合 many rows + few columns。

但这里还有一个问题。

假设：

```text
fact_swap = 100 billion rows
```

即使 Query 只读取：

```text
block_time
pool_address
amount_usd
```

如果每次都要扫描 1000 亿行，依然很贵。

所以还需要进一步解决：

> Can we avoid reading large portions of irrelevant rows?

这就是本课的核心。

分析数据库的性能，不只是：

```text
Column Store
```

还来自：

```text
Partitioning
+
Ordering
+
Data Skipping
```

---

## 二、Partition：先把超大表切成大块

假设 `fact_swap` 按月份 Partition：

```text
2026-01
2026-02
2026-03
2026-04
...
```

如果 Query 是：

```sql
SELECT
    pool_address,
    SUM(amount_usd)
FROM fact_swap
WHERE block_time >= '2026-03-01'
  AND block_time <  '2026-04-01'
GROUP BY pool_address;
```

数据库就可能直接跳过：

```text
2026-01
2026-02
2026-04
...
```

只访问：

```text
2026-03
```

这就是：

> Partition Pruning.

Partition 的核心作用可以记成：

> Eliminate entire large chunks of data before scanning.

---

## 三、Partition 不是“越细越好”

这是很容易犯的错误。

如果按天 Partition：

```text
2026-03-01
2026-03-02
2026-03-03
...
```

可能还合理。

但如果你按：

```text
wallet_address
```

做 Partition：

可能产生海量 Partition。

例如：

```text
millions of wallets
→ millions of partitions
```

这会带来大量管理成本。

所以 Partition Key 通常应该：

- cardinality 不要过高；
- 能明显对应常见过滤条件；
- 能形成相对大的数据块。

因此在 Blockchain Analytics 中，常见思路是：

> Time-based Partitioning.

例如：

```text
month
day
```

而不是随便用高基数字段。

---

## 四、为什么 Blockchain 数据特别适合时间 Partition？

链上数据天然具有时间顺序：

```text
block_number
block_time
```

而很多查询也天然是时间范围：

```sql
WHERE block_time >= now() - INTERVAL 7 DAY
```

```sql
WHERE block_time >= now() - INTERVAL 30 DAY
```

```sql
WHERE block_time BETWEEN ...
```

所以：

```text
time-based partition
```

通常与：

```text
time-range query
```

高度匹配。

例如：

```text
PARTITION BY month(block_time)
```

可以帮助数据库先排除大量历史月份。

---

## 五、但 Partition 只能解决第一层问题

假设：

```text
March Partition = 2 billion rows
```

你的 Query 只查：

```text
USDC
```

即使已经通过 Partition Pruning，只剩 3 月数据：

```text
2 billion rows
```

还是很多。

所以还需要进一步在 Partition 内减少读取。

这里就进入：

> Order Key.

---

## 六、Order Key：让相关数据尽量靠在一起

假设同一个 Partition 内的数据有两种排列方式。

方案 A：

```text
random order
```

例如：

```text
USDC
DAI
USDT
USDC
WETH
DAI
USDC
...
```

方案 B：

按 `token_address` 排序：

```text
DAI
DAI
DAI
...
USDC
USDC
USDC
...
USDT
USDT
...
WETH
WETH
...
```

如果 Query 是：

```sql
WHERE token_address = 'USDC'
```

方案 B 中，USDC 数据会更集中。

这意味着数据库有机会：

> skip large regions that cannot contain USDC.

所以：

> Ordering creates physical locality.

---

## 七、Order Key 和传统 Index 不是同一回事

这里要特别区分。

[Postgres / OLTP 视角]

传统 B-tree Index 思路更像：

```text
value
→ pointer
→ locate row
```

例如：

```sql
WHERE tx_hash = '0xabc'
```

可以快速定位少量行。

而分析数据库里的 Ordering + Data Skipping 更像：

```text
这一大块数据的 min/max 范围
→ 判断 Query 是否可能命中
→ 如果不可能，整块跳过
```

重点不是：

> 精确定位一行。

而是：

> Skip large irrelevant ranges.

所以它更适合：

```text
large scan
```

而不是 point lookup。

---

## 八、Data Skipping 是什么？

假设一个数据块里：

```text
block_time:
2026-03-01 ~ 2026-03-03
```

另一个数据块：

```text
block_time:
2026-03-04 ~ 2026-03-06
```

Query：

```sql
WHERE block_time >= '2026-03-05'
```

那么：

```text
2026-03-01 ~ 2026-03-03
```

这一块完全不可能命中。

数据库可以直接跳过。

这就是：

> Data Skipping.

核心思想：

> If metadata proves a data block cannot match the predicate, don't read it.

---

## 九、Order Key 为什么会增强 Data Skipping？

如果数据完全随机：

```text
Block A:
USDC / DAI / WETH / USDT

Block B:
USDC / DAI / WETH / USDT

Block C:
USDC / DAI / WETH / USDT
```

那么 Query：

```sql
WHERE token_address = 'USDC'
```

几乎每个 Block 都可能有 USDC。

数据库就很难 skip。

但如果数据按 `token_address` 排序：

```text
Block A:
DAI only

Block B:
USDC only

Block C:
USDT only
```

Query：

```sql
WHERE token_address = 'USDC'
```

那么：

```text
Block A → skip
Block B → read
Block C → skip
```

所以：

> Good ordering increases the effectiveness of data skipping.

---

## 十、Partition 和 Order Key 的职责不同

可以用一句话区分：

> Partition eliminates large coarse-grained regions.

> Order Key improves locality within the remaining data.

例如：

```text
Partition:
2026-03

Inside Partition ordered by:
token_address
```

Query：

```sql
WHERE block_time >= '2026-03-01'
  AND block_time <  '2026-04-01'
  AND token_address = 'USDC'
```

执行思路可以抽象成：

```text
Step 1
Partition Pruning
→ only March

Step 2
Data Skipping using Order
→ mostly USDC ranges

Step 3
Column Scan
→ only required columns

Step 4
Aggregation
```

这就是多层减少工作量。

---

## 十一、为什么物理设计必须从 Query Pattern 反推？

假设你把表排序成：

```text
ORDER BY tx_hash
```

但 90% 的 Dashboard 都是：

```sql
WHERE block_time >= ...
  AND token_address = ...
GROUP BY token_address;
```

那 `tx_hash` 的排序对这些查询帮助可能很小。

所以不能问：

> 什么字段“看起来重要”？

而应该问：

> What predicates appear most often in analytical queries?

也就是：

```text
WHERE 用什么？
GROUP BY 用什么？
时间范围怎么查？
常见过滤维度是什么？
```

再反推：

```text
Partition Key
Order Key
```

---

## 十二、Blockchain Transfer 的一个典型设计

假设表：

```text
fact_token_transfer
```

常见查询：

```sql
WHERE block_time >= ...
```

以及：

```sql
WHERE token_address = ...
```

再按：

```text
wallet
token
day
```

做聚合。

那么一个高层设计思路可能是：

```text
Partition:
month(block_time)

Order Key:
(token_address, block_time)
```

这里的逻辑是：

```text
month(block_time)
→ prune irrelevant months

token_address
→ cluster same-token rows

block_time
→ preserve time locality within token
```

这不是唯一答案。

但它体现了：

> Physical design follows query pattern.

---

## 十三、如果查询更偏 Wallet 呢？

假设产品是 Wallet Analytics。

最常见 Query：

```sql
WHERE wallet_address = '0xabc'
  AND block_time >= ...
```

那么你可能更希望数据按：

```text
wallet_address
block_time
```

组织。

因为你希望：

```text
same wallet
→ physically near each other
```

从而帮助：

```text
wallet range query
```

所以：

> There is no universally best Order Key.

它依赖 workload。

---

## 十四、一个关键 Trade-off：一个排序不可能同时完美服务所有 Query

假设两个主要 workload：

A：

```sql
WHERE token_address = ...
```

B：

```sql
WHERE wallet_address = ...
```

如果 Order Key 主要围绕：

```text
token_address
```

那么 A 会很好。

但 B 不一定同样好。

反过来也一样。

所以分析数据库设计经常是在做：

> workload prioritization.

即：

```text
Which queries matter most?
```

而不是：

```text
Can one physical layout optimize everything?
```

答案通常是不能。

---

## 十五、这和传统 Oracle / Postgres 有什么联系？

其实思路你并不陌生。

传统数据库里会问：

```text
Should I create an index on this column?
```

本质是在问：

> Which access pattern should I optimize?

分析数据库里只是把这个问题进一步放大到：

```text
Partition
Ordering
Data Skipping
Compression
```

所以核心仍然是：

> Physical design follows access pattern.

---

## 十六、Data Skipping 不是“跳过结果”，而是“跳过读取”

这个区别很重要。

假设总表：

```text
100 billion rows
```

最后结果只返回：

```text
100 rows
```

并不意味着 Query 便宜。

如果为了得到 100 行结果，仍然扫描了 1000 亿行：

还是很贵。

真正重要的是：

> How much data did the engine actually read?

所以 Data Skipping 关注的是：

```text
bytes read
rows scanned
data blocks touched
```

而不是：

```text
result row count
```

这和你第 2 课里纠正过的概念是一致的。

---

## 十七、一个完整的查询成本缩减链条

假设原始表：

```text
100 billion rows
```

Query 只查：

```text
March
USDC
3 columns
```

理想情况下：

```text
100 billion rows
        ↓ Partition Pruning
March only
        ↓ Data Skipping
Mostly USDC ranges
        ↓ Column Pruning
Only 3 columns
        ↓ Compression
Read fewer bytes
        ↓ Aggregation
Produce result
```

你会发现：

ClickHouse 快，不是来自一个魔法功能。

而是多层减少工作量。

---

## 十八、常见错误一：Partition Key 选高基数字段

例如：

```text
wallet_address
tx_hash
```

如果用作非常细的 Partition：

可能产生过多 Partition。

这通常不是好设计。

Partition 更适合：

```text
coarse-grained boundary
```

例如：

```text
month
day
chain_id + month
```

具体仍然要看 workload 和数据规模。

---

## 十九、常见错误二：Order Key 只考虑唯一性

传统关系数据库思维容易把：

```text
Primary Key
```

直接理解成：

> 唯一标识。

但在分析数据库里，Order Key 更重要的一个问题是：

> How should data be physically clustered for common queries?

所以不能只问：

```text
哪几个字段能唯一确定一行？
```

还要问：

```text
哪几个字段最常出现在过滤条件里？
```

这是 OLTP 和 OLAP 物理设计思维差异之一。

---

## 二十、Blockchain 场景中的典型 Query Pattern

Transfer：

```text
time range
token
wallet
chain
```

Swap：

```text
time range
pool
protocol
token pair
```

NFT：

```text
time range
collection
wallet
marketplace
```

所以不同 Fact Table 的物理设计可能不同。

不能简单统一成：

```text
ORDER BY block_time
```

然后认为所有查询都解决了。

---

## 二十一、从 Data Engineer 视角设计表时要先问什么？

在定 Partition / Order Key 前，先列出：

```text
Top Queries
```

例如：

```text
Q1:
Daily token volume by token

Q2:
Wallet transfer history

Q3:
Protocol daily active users

Q4:
Pool volume over time
```

然后拆解：

```text
Filter columns
Time range
Grouping columns
Expected scan volume
Concurrency
```

最后才设计：

```text
Partition Key
Order Key
Pre-aggregation
```

这是更成熟的数据建模过程。

---

## 二十二、一个 Swap 表例子

假设：

```text
fact_swap
```

最重要 Dashboard：

```sql
SELECT
    toDate(block_time) AS day,
    protocol,
    pool_address,
    SUM(amount_usd)
FROM fact_swap
WHERE block_time >= now() - INTERVAL 90 DAY
  AND protocol = 'Uniswap'
GROUP BY
    day,
    protocol,
    pool_address;
```

一个可能的高层设计：

```text
Partition:
month(block_time)

Order Key:
(protocol, pool_address, block_time)
```

思路：

```text
month
→ prune old months

protocol
→ cluster protocol data

pool_address
→ cluster same pool

block_time
→ maintain time locality
```

注意：

这只是基于该 workload 的合理候选设计，不是通用标准答案。

---

## 二十三、Partition、Order Key、Data Skipping 的关系

可以压缩成：

```text
Partition
→ Which big chunks can I ignore?

Order Key
→ Which rows stay physically close?

Data Skipping
→ Which smaller regions can I prove are irrelevant?
```

三者共同目标：

> Reduce data read before expensive aggregation starts.

---

## 二十四、和上一课连接起来

上一课我们说：

> ClickHouse is good at scanning large historical datasets.

这一课补充：

> But a good analytical design tries not to scan more than necessary.

所以完整逻辑变成：

```text
Column Store
→ read fewer columns

Partition
→ read fewer large regions

Order Key
→ improve locality

Data Skipping
→ read fewer data blocks

Compression
→ read fewer bytes

Aggregation Engine
→ process remaining data efficiently
```

这才是完整的分析查询性能模型。

---

## 本课核心结论

> Partition is a coarse-grained data elimination mechanism.

> Order Key creates physical locality for common query dimensions.

> Data Skipping avoids reading data blocks that cannot match the query predicate.

> Partition Key and Order Key must be derived from real Query Patterns, not chosen from “important-looking” columns.

> One physical layout cannot optimize every query equally well; analytical design requires workload prioritization.

> Query performance depends much more on data actually read than on final result size.

---

## 理解检查

### 问题一

假设一张 `fact_transfer` 有 1000 亿行，并按月份 Partition。

一个 Query 只查 2026 年 3 月。

Partition 能帮助它解决什么问题？

如果 3 月本身仍然有 30 亿行，为什么 Partition 还不够？

### 问题二

假设 3 月 Partition 内，数据按：

```text
(token_address, block_time)
```

排序。

Query：

```sql
WHERE token_address = 'USDC'
  AND block_time >= '2026-03-01'
  AND block_time <  '2026-04-01'
```

为什么这种 Order Key 有助于 Data Skipping？

### 问题三

下面两种产品：

A. Token Analytics：最常见查询是按 `token_address + time range`

B. Wallet Analytics：最常见查询是按 `wallet_address + time range`

为什么它们可能不应该使用完全相同的 Order Key？

请从 Query Pattern / Physical Locality 的角度解释。

## 问题一｜用户回答

Partition 能帮他解决数据分区的问题。如果我把 Partition 设置为按月份来分区，那么在查询这个 factor_transfer 表的时候，就可以忽略其他的月份，直接查 3 月份的数据。

如果 3 月本身仍然有 30 亿行，那 Partition 这个数据量还是比较大，仅靠 Partition 的话还不够，我们可以通过 Order Key 的方式再进行优化

## 问题一｜老师判断与校准

回答正确。

- 第一部分判断准确：按月份 Partition 后，查询 2026 年 3 月时，可以通过 Partition Pruning 排除其他月份，只读取 3 月对应的大块数据。
- 第二部分也正确：如果 3 月 Partition 本身仍有 30 亿行，Partition 只能完成 coarse-grained elimination，剩余数据量仍然很大。
- 下一步需要依赖 Order Key 提高 Physical Locality，并让 Data Skipping 有机会进一步跳过不可能命中的数据块。

可以把这一题压缩成：

> Partition 解决“先排除哪些大块”；Order Key + Data Skipping 解决“在剩下的大块里还能少读多少”。

## 问题二｜用户回答

这里我将 token address 来做 order key 处理，那么相同的 token 就会被分在一起。如果不对数据做 OrderKey 处理，那么 Token 就会随机分布在各个 block 数据块里面。这样一来，查询的时候就会把范围内的所有 block 都搜索查询一遍。在 order key 处理之后，USDC 这个 token 数据已经被聚集到一块了，在相对集中的几个 block 里面，这样就可以忽略其他的无关的block，它们会被 data skipping，从而减少查询范围。

## 问题二｜老师判断与校准

回答正确。

- 核心因果链已经完整：Order Key 让相同 token 的数据形成更强的 Physical Locality；USDC 不再随机散布在大量 data blocks / granules 中。
- 如果数据随机分布，很多数据块都可能包含 USDC，因此查询引擎很难证明这些块与条件无关，也就难以跳过。
- 按 token_address 排序后，USDC 会集中在较少的连续区域；其他区域更容易根据 metadata 判断“不可能命中”，从而执行 Data Skipping，减少实际读取的数据块数量。

更精确地说，不是“必须搜索每一个 block”，而是“随机分布会让更多 data blocks 保持 potentially relevant，降低 skipping effectiveness”。

## 问题三｜用户回答

问题三：A、B 两种产品，它们的 query pattern 不完全相同，但在 time range 这个条件上，它们是相同的，那我们可以根据 time 来做 partition 分区。但是另一个查询条件，一个是 token address，另外一个是 wallet address。

对产品 A 来说，它要做 token analytics，需要根据 token address 来做 order key。但是对于产品 B（Wallet Analytics）来说，它要根据 wallet address 来做 order key。

所以，它们使用的是不相同的 order key

## 问题三｜老师判断与校准

回答正确。

- 你已经正确区分了可以共享的物理设计维度和不能共享的物理设计维度：两种产品都常见 time range，因此都可以考虑 time-based Partition。
- 但主要 Query Pattern 不同：Token Analytics 主要围绕 token_address，Wallet Analytics 主要围绕 wallet_address。
- 因此，为了形成更好的 Physical Locality，A 更适合围绕 token_address 设计 Order Key，B 更适合围绕 wallet_address 设计 Order Key。
- 核心原则不是“某个字段重要就拿来排序”，而是 Physical Design follows Query Pattern。

可以压缩成：

> Same partitioning dimension does not imply the same ordering dimension. Partition answers “which large time chunks can I prune?”, while Order Key answers “which query dimension should be physically clustered inside those chunks?”
