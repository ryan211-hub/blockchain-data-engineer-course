# Module 11 · 第 4 课
## ClickHouse：为什么适合 Blockchain Analytics

### Lesson Contract

所属 Module：Module 11 — 分析数据库

本课核心问题：

> ClickHouse 为什么特别适合 Blockchain Analytics？它的优势到底来自“品牌快”，还是来自它与链上分析 workload 的天然匹配？

学完以后，你应该能够解释：

- ClickHouse 在 Blockchain Data Platform 中承担什么角色。
- 为什么 Column-oriented Storage 对链上历史分析有优势。
- 为什么 ClickHouse 适合 append-heavy historical facts。
- 为什么 Compression、Scan Throughput、Aggregation Performance 对链上分析重要。
- 为什么 ClickHouse 常用于 Dashboard / Historical Analytics，而不是 Current State Serving。
- ClickHouse 与 Postgres 的角色为什么互补。
- 为什么“适合 ClickHouse”应该从 workload 推导，而不是从“数据量大”直接得出结论。

本课只讲 ClickHouse 的高层工程模型，不深入 MergeTree 内部合并算法、Vectorized Execution、SIMD、分布式一致性等实现细节。

---

## 一、先从上一课继续

上一课我们得出的结论是：

> Postgres becoming expensive is often a workload signal, not a database failure.

也就是说，当系统出现：

```text
many rows
few columns
historical scan
heavy aggregation
high concurrency
append-heavy data
```

我们开始考虑：

> 是否需要一个 Analytical Database？

ClickHouse 就是在这种问题背景下出现的。

所以这一课不要先问：

> ClickHouse 有哪些功能？

而应该先问：

> What kind of workload is ClickHouse designed for?

---

## 二、ClickHouse 的定位

[Data Engineer 视角]

可以先把 ClickHouse 理解成：

> A column-oriented analytical database designed for high-throughput analytical queries.

三个关键词：

```text
column-oriented
analytical
high-throughput
```

这三个词基本已经解释了它为什么适合 Blockchain Analytics。

因为典型链上分析 workload 恰好也是：

```text
huge historical facts
+
few-column scans
+
GROUP BY / SUM / COUNT
+
time-range analytics
```

所以不是因为：

> “ClickHouse 是 Web3 专用数据库。”

它不是。

而是因为：

> Blockchain Analytics workload 与 ClickHouse 的设计目标高度匹配。

---

## 三、Blockchain Historical Fact 的典型形态

假设我们有：

```text
fact_token_transfer
```

字段可能有：

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
amount
protocol
is_canonical
...
```

长期运行以后：

```text
10 billion rows
50 billion rows
100 billion rows
```

都可能出现。

但典型 Dashboard Query 可能只是：

```sql
SELECT
    toDate(block_time) AS day,
    token_address,
    SUM(amount) AS volume
FROM fact_token_transfer
WHERE block_time >= now() - INTERVAL 180 DAY
GROUP BY
    day,
    token_address;
```

这个 Query 的特点：

```text
touch many rows
read few columns
aggregate heavily
```

这就是 ClickHouse 的典型舒适区。

---

## 四、优势一：Column-oriented Storage

我们第 2 课已经学过：

> Column Store keeps values from the same column together.

所以如果 Query 只需要：

```text
block_time
token_address
amount
```

就主要读取这几列。

而不需要把：

```text
tx_hash
from_address
to_address
log_index
protocol
...
```

全部一起读进来。

这对宽表尤其重要。

链上事实表往往：

> many columns + huge rows

而分析查询经常：

> many rows + few columns

两者刚好匹配。

---

## 五、优势二：Compression

同一列的数据天然更容易压缩。

例如：

```text
token_address
```

可能大量重复：

```text
USDC
USDC
USDC
USDT
USDT
USDC
...
```

而：

```text
block_number
```

通常具有明显顺序性：

```text
100
101
102
103
104
...
```

因此 Column Store 可以更有效地利用：

```text
same type
similar distribution
repetition
ordering
```

得到更高 Compression Ratio。

Compression 的价值不只是省磁盘。

链条是：

```text
better compression
→ less data stored
→ less data read
→ less disk I/O
→ faster analytical scan
```

所以：

> Compression is both a storage feature and a query performance feature.

---

## 六、优势三：Scan Throughput

OLTP 的典型目标是：

> Avoid scanning too much.

OLAP 的目标则常常是：

> If scanning is unavoidable, make scanning cheap.

例如：

```sql
SELECT
    pool_address,
    SUM(amount_usd)
FROM fact_swap
WHERE block_time >= now() - INTERVAL 365 DAY
GROUP BY pool_address;
```

如果这本来就需要处理 20 亿行，那么重点不是：

> “怎么完全不扫描？”

而是：

> “怎么更高效地扫描和聚合这 20 亿行？”

这就是 ClickHouse 的核心价值之一。

---

## 七、优势四：Aggregation-heavy Workload

Blockchain Analytics 中最常见的 SQL 之一就是：

```sql
GROUP BY
SUM
COUNT
AVG
uniq
```

例如：

```sql
SELECT
    toDate(block_time) AS day,
    pool_address,
    SUM(amount_usd) AS volume_usd,
    COUNT(*) AS swap_count
FROM fact_swap
WHERE block_time >= now() - INTERVAL 30 DAY
GROUP BY
    day,
    pool_address;
```

这类 Query 不是查“某一行是什么”。

而是在问：

> What pattern exists across a huge number of rows?

这正是 OLAP。

ClickHouse 就是为这种高吞吐聚合设计的。

---

## 八、为什么 Blockchain 数据特别适合 Append-heavy 模式

链上事实有一个很特殊的特点：

```text
Block N
Block N+1
Block N+2
...
```

新的事实持续产生。

例如：

```text
new transfer
new swap
new mint
new burn
```

大多数时候是：

> append new facts

而不是：

> random update existing rows all day long

所以从写入模型看：

```text
append-heavy
```

与分析数据库天然比较契合。

这也是为什么：

```text
Historical Transfer
Historical Swap
Historical Event
```

很适合放到 ClickHouse 一类系统中。

---

## 九、但 Blockchain 并不是完全 Immutable

这里要加一个视角前提。

[Blockchain Data Engineer 视角]

链上事实通常是：

> immutable-ish

不是绝对 immutable。

因为还有：

```text
Reorg
Correction
Canonical / Orphan
Late-arriving enrichment
```

所以不能简单理解成：

> “写进去以后永远不变。”

更准确是：

> 大多数历史事实以 append 为主，但系统仍需要支持 correction semantics。

这也是后续设计：

```text
is_canonical
version
replay
rebuild aggregate
```

时要考虑的问题。

不过整体上，它仍然比典型 OLTP 更偏 append-heavy。

---

## 十、ClickHouse 更适合哪一层？

把我们已经学过的数据分层放进来。

```text
Raw
→ Normalized
→ DWS
→ ADS
```

ClickHouse 通常比较适合：

```text
Normalized Historical Facts
DWS Aggregation
ADS Analytical Serving
```

例如：

```text
fact_token_transfer
fact_swap
fact_nft_transfer
dws_token_daily_volume
dws_wallet_daily_flow
ads_protocol_dashboard
```

因为这些表通常：

```text
large
historical
append-heavy
aggregation-oriented
```

---

## 十一、Postgres 和 ClickHouse 的角色对照

可以先建立这个简单地图：

```text
Postgres
→ current state
→ operational state
→ metadata
→ low-latency point lookup
→ API serving

ClickHouse
→ historical facts
→ large scans
→ aggregation
→ dashboard
→ analytical serving
```

所以更准确的关系不是：

```text
Postgres VS ClickHouse
```

而是：

```text
Postgres + ClickHouse
```

即：

> Different engines for different workloads.

---

## 十二、一个典型 Blockchain Data Platform

```text
Ethereum / RPC
      ↓
Indexer
      ↓
Kafka / ETL
      ↓
Normalized Data
      ↓
 ┌───────────────┬────────────────┐
 ↓               ↓
Postgres       ClickHouse
 ↓               ↓
Current State  Historical Facts
API            Dashboard
Metadata       Analytics
Job State      Aggregation
```

注意：

> one source, multiple serving engines.

不是说每份数据一定完全重复。

而是：

> 同一个业务事实可以根据不同 workload，被组织到不同 serving engine 中。

---

## 十三、为什么 ClickHouse 不一定适合 Current Balance API？

假设 API：

```sql
SELECT balance
FROM wallet_token_balance
WHERE chain_id = 1
  AND wallet_address = '0xabc'
  AND token_address = 'USDC';
```

特点：

```text
few rows
current state
low latency
frequent updates
```

这并不是 ClickHouse 最典型的目标 workload。

Postgres 这种 Row Store + Index 的模式通常更自然。

因此：

> ClickHouse is not automatically the best database just because it is fast at analytics.

仍然必须看 workload。

---

## 十四、为什么 ClickHouse 很适合 Dashboard？

Dashboard 常见 Query：

```sql
SELECT
    toStartOfMonth(block_time) AS month,
    token_address,
    SUM(amount_usd) AS volume
FROM fact_token_transfer
WHERE block_time >= now() - INTERVAL 2 YEAR
GROUP BY
    month,
    token_address;
```

它的特点：

```text
historical
large scan
few columns
aggregation
repeatable analytical pattern
```

这几乎就是 ClickHouse 的目标场景。

所以链上：

```text
DEX Dashboard
Wallet Analytics
Token Analytics
Protocol Metrics
Active Address Metrics
Volume / TVL / Fee Analytics
```

都很容易出现 ClickHouse 的使用空间。

---

## 十五、为什么“ClickHouse 快”这个说法不够专业？

因为：

> Fast for what?

如果是：

```text
SELECT one row by primary key
```

Postgres 可能已经非常快。

如果是：

```text
scan 10 billion rows
read 3 columns
group by 2 dimensions
```

ClickHouse 可能更自然。

所以专业表达应该是：

> ClickHouse is efficient for large-scale analytical workloads because its column-oriented storage, compression, and execution model reduce the cost of scanning and aggregating large historical datasets.

而不是：

> ClickHouse 比 Postgres 快。

---

## 十六、一个银行类比

银行体系里：

```text
Core Banking DB
→ current account state
→ transaction processing
```

和：

```text
Data Warehouse
→ historical transactions
→ reporting
→ aggregation
```

本来就是不同角色。

Blockchain 世界里：

```text
Postgres
→ Wallet State / API / Metadata
```

对应：

```text
Operational / Serving
```

而：

```text
ClickHouse
→ Transfer / Swap / Protocol Analytics
```

更接近：

```text
Analytical Warehouse / Serving
```

你会发现，本质上并不是新问题。

只是数据量、实时性和 append-heavy 特征更突出。

---

## 十七、什么时候 ClickHouse 的优势会变明显？

通常当这些条件越来越集中出现：

```text
billions of rows
few-column scans
high compression potential
historical analysis
heavy GROUP BY
high dashboard concurrency
append-heavy data
long retention
```

ClickHouse 的价值就会越来越明显。

注意：

不是必须所有条件都满足。

而是：

> workload 越接近这个模式，ClickHouse 越自然。

---

## 十八、什么时候没必要急着上 ClickHouse？

如果：

```text
dataset still small
queries are selective
dashboard already uses DWS / ADS
Postgres meets SLA
concurrency is low
team is small
```

那完全可以继续 Postgres。

因为引入 ClickHouse 会增加：

```text
new database
new deployment
new monitoring
new sync path
new data consistency problem
new operational cost
```

所以架构决策永远是：

> benefit vs complexity

---

## 十九、ClickHouse 的真正价值不是“替代数据库”

它真正的价值是：

> 为大规模分析 workload 提供一个更合适的物理执行环境。

因此你以后设计架构时，可以形成这样的思维：

```text
Question 1:
What is the workload?

Question 2:
What storage layout matches it?

Question 3:
What engine fits that workload?

Question 4:
Is the added system complexity worth it?
```

这比直接问：

> “要不要用 ClickHouse？”

更接近 Data Engineer 的思考方式。

---

## 二十、从 Blockchain Data Engineer 视角总结

假设：

```text
fact_swap = 30 billion rows
```

核心 Query：

```sql
SELECT
    toDate(block_time) AS day,
    pool_address,
    SUM(amount_usd) AS volume_usd
FROM fact_swap
WHERE block_time >= now() - INTERVAL 365 DAY
GROUP BY
    day,
    pool_address;
```

你应该这样分析：

```text
1. huge historical dataset
2. append-heavy
3. query touches many rows
4. only a few columns are needed
5. aggregation is heavy
6. compression potential is high
7. dashboard concurrency may be high
```

因此：

> This workload is a natural fit for ClickHouse-style analytical storage.

---

## 本课核心结论

> ClickHouse is not a “Web3 database”; it is an analytical database whose design matches many Blockchain Analytics workloads.

> Its main advantage comes from workload fit: column-oriented storage, compression, high scan throughput, and efficient aggregation.

> Blockchain historical facts are often append-heavy, huge, long-retained, and queried through time-range scans and aggregation.

> Postgres and ClickHouse usually play complementary roles rather than replacing each other.

> Database choice should still start from workload, not from product reputation.

---

## 理解检查

### 问题一

为什么不能简单地说：

> “Blockchain 数据量很大，所以应该使用 ClickHouse。”

请从 workload 的角度解释。

### 问题二

假设有一张 200 亿行的 `fact_swap`：

Dashboard 每天都执行：

```sql
SELECT
    toDate(block_time) AS day,
    pool_address,
    SUM(amount_usd)
FROM fact_swap
WHERE block_time >= now() - INTERVAL 180 DAY
GROUP BY
    day,
    pool_address;
```

请说明至少三个原因，为什么这个 workload 比较适合 ClickHouse。

### 问题三

为什么一个 Blockchain Data Platform 可能同时保留：

```text
Postgres
+
ClickHouse
```

而不是只选择其中一个？

请分别说明它们更适合承担什么角色。