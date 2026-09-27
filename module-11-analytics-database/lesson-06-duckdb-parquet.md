# Module 11 · 第 6 课
## DuckDB + Parquet：本地分析与低成本历史查询

### Lesson Contract

所属 Module：Module 11 — 分析数据库

本课核心问题：

> 如果我只是想分析一批历史链上数据，是否一定要先部署一套 ClickHouse 或 Postgres？有没有一种更轻量的方式，直接对文件做 SQL Analytics？

学完以后，你应该能够解释：

- DuckDB 是什么，它和 Postgres / ClickHouse 的角色有什么不同。
- Parquet 为什么适合保存历史分析数据。
- 为什么 DuckDB 可以直接查询 Parquet，而不需要先把数据导入传统数据库。
- DuckDB + Parquet 适合哪些 Blockchain Data Engineering 场景。
- 为什么它很适合 Ad-hoc Analytics、Backfill Validation、Local Analysis。
- 为什么它通常不替代 Postgres Serving 或 ClickHouse Online Analytics。
- 如何理解“文件就是数据层，DuckDB 是查询引擎”。

本课不深入 DuckDB Vectorized Execution、Parquet Encoding 内部算法、Object Storage 底层实现。

---

## 一、先从一个实际问题开始

假设你有过去 3 年的 Ethereum Transfer 数据。

总量：

```text
2 TB
```

你现在只想做一次分析：

```text
统计过去 3 年每个月 USDC Transfer Volume
```

一个传统思路是：

```text
Parquet / CSV files
        ↓
Load
        ↓
Postgres / ClickHouse
        ↓
SQL Query
```

但这里有一个问题：

> 为了一次分析，有没有必要先部署数据库、建表、导入数据？

不一定。

另一种模式是：

```text
Parquet Files
      ↓
DuckDB
      ↓
SQL
```

直接查询。

---

## 二、DuckDB 是什么？

[Data Engineer 视角]

DuckDB 可以先理解为：

> An embedded analytical SQL database.

这里有两个关键词：

```text
embedded
analytical
```

Analytical：

它更偏 OLAP。

Embedded：

它不一定要求你先运行一个独立数据库服务器。

例如 Postgres 通常是：

```text
Application
    ↓
Network
    ↓
Postgres Server
```

而 DuckDB 可以更像：

```text
Python Process
      ↓
DuckDB Engine
      ↓
Parquet Files
```

也就是说：

> DuckDB can run inside your application or local process.

---

## 三、一个你很熟悉的类比：SQLite

如果你知道 SQLite，可以先这样理解：

```text
SQLite
→ embedded database
→ OLTP-oriented / application data

DuckDB
→ embedded database
→ OLAP-oriented / analytical data
```

这不是完全等价，但作为第一层理解非常有用。

可以先记：

> DuckDB is often described as “SQLite for analytics”.

重点不是品牌比较，而是：

```text
无需维护独立数据库服务
+
直接在本地进程运行
+
支持 SQL Analytics
```

---

## 四、那 Parquet 是什么？

Parquet 不是数据库。

它是一种：

> Columnar file format.

也就是列式文件格式。

例如逻辑上有：

```text
block_time
token_address
wallet_address
amount
```

Parquet 会以更适合分析的列式结构保存数据。

这和我们前面学的 Column Store 思想是一致的：

> many rows + few columns

分析 Query 只需要：

```text
block_time
token_address
amount
```

那么就不一定需要读取所有字段。

---

## 五、为什么 Parquet 适合历史数据？

Blockchain Historical Data 往往具有：

```text
large
append-heavy
historical
rarely updated
analytical
```

这类数据非常适合放成文件。

例如：

```text
s3://blockchain-data/transfers/
    year=2026/
        month=01/
        month=02/
        month=03/
```

或者本地：

```text
data/
  transfers/
    2026-01.parquet
    2026-02.parquet
    2026-03.parquet
```

这种模式下，历史数据不一定要一直放在昂贵的在线数据库中。

---

## 六、Parquet 和我们上一课的 Partition 思想是连起来的

上一课我们讲：

```text
Partition
→ coarse-grained pruning
```

文件世界里也一样。

例如：

```text
transfers/
  year=2026/
    month=01/
    month=02/
    month=03/
```

Query：

```sql
WHERE block_time >= '2026-03-01'
  AND block_time < '2026-04-01'
```

理想情况下，只需要访问：

```text
month=03
```

所以你会发现：

> Partitioning is not only a database concept.

它也是 Data Lake / Parquet Dataset 的重要设计思想。

---

## 七、DuckDB 为什么和 Parquet 很搭？

因为 DuckDB 可以直接查询 Parquet。

例如：

```sql
SELECT
    token_address,
    SUM(amount) AS volume
FROM read_parquet('transfers/*.parquet')
GROUP BY token_address;
```

不需要先做：

```text
CREATE TABLE
INSERT 2 TB
WAIT
QUERY
```

而是：

```text
Parquet
→ SQL directly
```

这就是 DuckDB + Parquet 非常重要的工程价值。

---

## 八、核心区别：Database-centric vs File-centric

传统数据库模式：

```text
Data
→ Load into Database
→ Query Database
```

DuckDB + Parquet 模式：

```text
Data stays in files
→ Query engine reads files directly
```

可以理解成：

```text
Parquet
= Storage

DuckDB
= Compute / Query Engine
```

这是一种非常重要的解耦：

> Storage and Compute can be separated.

---

## 九、为什么这对 Blockchain Data Engineer 很有用？

假设你的 Blockchain Data Platform：

```text
RPC
 ↓
Indexer
 ↓
Kafka
 ↓
ClickHouse
```

ClickHouse 保存在线分析数据。

但是你可能还有：

```text
5 years historical raw facts
```

这些历史数据：

- 很少查询；
- 主要用于 Backfill；
- Audit；
- Validation；
- Research；
- Bug Repair。

如果全部长期保存在高性能在线数据库里：

```text
cost ↑
```

一种常见思路就是：

```text
Hot Data
→ ClickHouse

Cold / Historical Data
→ Parquet
```

需要分析历史时：

```text
DuckDB + Parquet
```

---

## 十、Hot Data 和 Cold Data

可以建立这个概念：

```text
Hot Data
→ queried frequently
→ low latency required
→ online DB

Cold Data
→ queried occasionally
→ latency requirement lower
→ cheap file storage
```

Blockchain 数据天然会不断增长。

例如：

```text
2024
2025
2026
2027
...
```

不是所有年份都需要一直处于：

```text
high-performance online database
```

因此可能设计：

```text
Recent 6 months
→ ClickHouse

Older history
→ Parquet / Object Storage
```

---

## 十一、一个银行系统类比

银行也可能有：

```text
近几个月交易
→ Online Warehouse / Data Mart
```

而更久以前的历史：

```text
Archive Files
→ Low-cost Storage
```

平时不查询。

但如果：

```text
监管审计
历史追溯
专项分析
```

再读取历史数据。

Blockchain 也是类似：

```text
Recent Analytics
→ ClickHouse

Long-term Historical Archive
→ Parquet

Temporary Analysis
→ DuckDB
```

---

## 十二、DuckDB 很适合 Ad-hoc Analytics

Ad-hoc 的意思是：

> 临时的、非固定生产查询。

例如今天你突然想验证：

> 2024 年某个 DeFi Protocol 是否出现异常 Transfer Spike？

数据已经在 Parquet。

你可以直接：

```sql
SELECT
    toDate(block_time) AS day,
    COUNT(*) AS transfer_count
FROM read_parquet('2024/*.parquet')
WHERE protocol = 'ExampleProtocol'
GROUP BY day
ORDER BY day;
```

不需要先设计正式 Pipeline。

---

## 十三、这和 Dashboard 查询不一样

Dashboard：

```text
24/7
many users
repeatable queries
low latency
high concurrency
```

更适合：

```text
ClickHouse
```

而 Ad-hoc Analysis：

```text
one analyst
one laptop / server
temporary query
historical files
```

更适合：

```text
DuckDB + Parquet
```

所以仍然是同一句：

> Different workloads, different engines.

---

## 十四、DuckDB 很适合 Backfill Validation

这是 Blockchain Data Engineer 很实际的一个场景。

假设你修复 Decoder Bug。

你重新 Backfill：

```text
block 18,000,000
→
block 19,000,000
```

生成：

```text
new_transfer.parquet
```

旧版本：

```text
old_transfer.parquet
```

你想比较：

```text
row count
aggregate amount
token distribution
wallet distribution
```

完全可以用 DuckDB：

```sql
SELECT COUNT(*)
FROM read_parquet('new_transfer.parquet');
```

或者：

```sql
SELECT
    token_address,
    SUM(amount)
FROM read_parquet('new_transfer.parquet')
GROUP BY token_address;
```

用于：

> Reconciliation / Validation.

---

## 十五、DuckDB 也很适合本地 Debug

比如 Indexer 解析出了：

```text
10 GB Transfer data
```

你不想：

```text
启动 Postgres
建表
Insert
建 Index
```

只是想看看：

```text
哪些 token 出现最多？
有没有 amount 异常？
某个 tx 是否解析正确？
```

DuckDB 就非常方便。

例如：

```sql
SELECT
    token_address,
    COUNT(*) AS cnt
FROM read_parquet('transfer.parquet')
GROUP BY token_address
ORDER BY cnt DESC
LIMIT 20;
```

---

## 十六、为什么 Parquet 比 CSV 更适合这个场景？

CSV 很简单。

但是对于大规模 Analytics：

```text
CSV
→ row-oriented text
→ weak typing
→ poor compression
→ usually reads more data
```

Parquet：

```text
columnar
typed
compressed
metadata-rich
analytics-friendly
```

所以对于：

```text
hundreds of GB
TB-scale historical data
```

Parquet 通常更合适。

---

## 十七、一个重要思想：Parquet 可以成为数据交换层

假设：

```text
Indexer
↓
Normalized Data
```

你可以输出：

```text
Parquet
```

然后不同工具都可以读取：

```text
DuckDB
Spark
Python
ClickHouse
Data Warehouse
```

所以 Parquet 很适合作为：

> Interoperable analytical storage format.

这意味着：

> 数据并不一定被某一个数据库产品锁死。

---

## 十八、这和 Raw / Normalized / DWS 分层怎么结合？

例如：

```text
Raw
→ JSON / raw block files

Normalized
→ Parquet facts

DWS
→ ClickHouse aggregated tables

ADS
→ ClickHouse / Postgres serving tables
```

也可以：

```text
Normalized Historical Fact
→ Parquet

Recent Analytical Fact
→ ClickHouse
```

没有唯一答案。

关键还是：

> workload + cost + retention.

---

## 十九、DuckDB 是否等于“小型 ClickHouse”？

不建议这么理解。

虽然两者都偏 OLAP，但角色不同。

ClickHouse 更偏：

```text
server database
multi-user
online analytics
high concurrency
continuous ingestion
dashboard serving
```

DuckDB 更偏：

```text
embedded
local
single-process
ad-hoc
file analytics
developer / analyst workflow
```

所以：

> Same OLAP family, different deployment role.

---

## 二十、Postgres / ClickHouse / DuckDB 三者终于可以放在一起了

现在可以建立完整地图：

```text
Postgres
→ Serving / Operational

ClickHouse
→ Online Historical Analytics

DuckDB
→ Local / Ad-hoc / File Analytics
```

再加 Parquet：

```text
Parquet
→ Analytical File Storage
```

所以：

```text
Postgres
ClickHouse
DuckDB
Parquet
```

并不是四个互相竞争的东西。

它们处在不同层次。

---

## 二十一、一个完整架构例子

```text
Ethereum RPC
      ↓
Indexer
      ↓
Kafka
      ↓
Normalized Pipeline
      ↓
 ┌──────────────┬──────────────┐
 ↓              ↓              ↓
Postgres     ClickHouse      Parquet
 ↓              ↓              ↓
API          Dashboard      Archive
Current      Analytics      Backfill
State                       Research
                              ↓
                            DuckDB
```

这里：

```text
Postgres
= operational serving

ClickHouse
= online analytical serving

Parquet
= cheap historical storage

DuckDB
= on-demand analytical engine
```

---

## 二十二、成本为什么是 DuckDB + Parquet 的重要优势？

假设你有：

```text
50 TB historical blockchain data
```

如果全部长期放在高性能数据库中：

```text
compute
memory
storage
replication
operations
```

都会产生持续成本。

但如果大量冷数据只是：

```text
偶尔分析
```

那么：

```text
cheap object storage
+
Parquet
+
compute only when needed
```

可能更经济。

这就是：

> Low-cost historical analytics.

---

## 二十三、但低成本不是免费

也必须看到 trade-off。

DuckDB + Parquet 通常不适合直接承担：

```text
thousands of concurrent dashboard users
millisecond API serving
high-frequency random updates
transaction processing
always-on realtime serving
```

也就是说：

```text
cheap + simple
```

换来的通常是：

```text
less online serving capability
```

所以仍然不能把它当万能方案。

---

## 二十四、一个常见工程场景

假设发现 Decoder Bug：

```text
Bug affected:
2025-01 → 2025-06
```

你要：

```text
Backfill
↓
Generate corrected Parquet
↓
DuckDB validation
↓
Compare aggregates
↓
Load corrected result into ClickHouse
```

这里 DuckDB 的角色非常自然：

> temporary analytical validation engine.

它不需要成为 Production Serving Database。

---

## 二十五、为什么这对你尤其重要？

你之前已经学过：

```text
Backfill
Reconciliation
Historical Repair
Replay
Validation
```

现在可以把这些概念接起来。

过去我们说：

```text
Backfill Pipeline
→ generate historical data
```

现在进一步：

```text
Backfill Output
→ Parquet
→ DuckDB Validation
→ Production Sink
```

形成一个非常实用的 Data Engineering workflow。

---

## 二十六、从 Data Engineer 视角做选择

以后看到一个需求，可以这样判断：

### 场景 A

```text
Wallet Balance API
P99 < 100ms
```

优先想到：

```text
Postgres
```

### 场景 B

```text
Dashboard
10 billion Transfer rows
many concurrent users
```

优先想到：

```text
ClickHouse
```

### 场景 C

```text
分析 3 年历史 Transfer
每月跑一次
数据已经是 Parquet
```

优先想到：

```text
DuckDB + Parquet
```

这就是：

> Workload → Storage → Engine.

---

## 本课核心结论

> DuckDB is an embedded analytical SQL engine.

> Parquet is a columnar analytical file format, not a database.

> DuckDB can query Parquet directly without first loading the data into a database server.

> DuckDB + Parquet is especially useful for local analytics, ad-hoc queries, historical research, backfill validation, and low-cost cold-data analysis.

> ClickHouse is better suited to online analytical serving and high-concurrency dashboards, while DuckDB is better suited to on-demand analytical workloads.

> Postgres, ClickHouse, DuckDB, and Parquet are not direct substitutes; they occupy different roles in a data platform.

---

## 理解检查

### 问题一

假设你有 3 TB 的历史 Transfer 数据，已经按月份保存成 Parquet。

现在只是临时分析一次：

> 过去 3 年每个月 USDC 的 Transfer Volume。

为什么这里可以考虑 DuckDB + Parquet，而不一定需要先把 3 TB 数据导入 ClickHouse？

### 问题二

请解释下面四者的角色区别：

```text
Postgres
ClickHouse
DuckDB
Parquet
```

其中哪个是文件格式，另外三个分别更偏向什么 workload？

### 问题三

假设你修复了一个 Decoder Bug，并重新 Backfill 了 6 个月的数据。

新的结果写成：

```text
corrected_transfer.parquet
```

为什么 DuckDB 很适合放在：

```text
Backfill
→ Validation
→ Production Load
```

中间的 Validation 阶段？

## 用户回答

问题一：

最主要的原因是，这是个临时性的需求，只需临时性分析一次。并且它的数据量比较大（有 3 TB），而且已经保存成 Parquet 文件格式。

所以这里可以考虑使用 DuckDB 加 Parquet，而不用导入 ClickHouse 这种在线分析数据库

问题 2：

1. Postgres：OLTP型数据库，偏向轻量化的 workload，适用于对latency要求比较高
2. ClickHouse：是OLAP型的数据库，主要面向统计分析型的业务需求（像 Dashboard 这种），能支持并发查询吗？
3. DuckDB：是嵌入式分析数据库，适合处理 Ad-hoc Analytics  ，可以直接查询 Parquet 格式文件
4. Parquet：是文件格式，是一种列式存储的、列式保存数据的文件格式

问题三：因为这类需求是 Ad-hoc 式的，它并不是常态化的，而是临时性、突发性的需求，而且是对大量的 cold data 做统计分析，这种情况比较适合搭 DuckDB

## 老师判断与校准

三题总体通过。

- 问题一：判断正确。关键不只是数据量大，而是 workload 是一次性的 / ad-hoc historical analysis，而且数据已经是 Parquet。DuckDB 可以直接查询这些文件，避免为了单次分析先部署数据库、建表并把 3 TB 数据完整导入 ClickHouse。
- 问题二：整体正确，但对 Postgres 的表述需要校准。Postgres 不应简单理解成“轻量 workload”，更准确是 general-purpose relational + transactional + serving database，适合 Current State、Operational State、Metadata、point lookup、低延迟 API 等 workload。ClickHouse 是面向 OLAP / online analytical serving 的列式分析数据库，能够支持多用户和并发分析查询，典型场景包括 Dashboard、Historical Analytics、Large Scan、Heavy Aggregation。DuckDB 是 embedded analytical database，更适合 local / ad-hoc / file analytics。Parquet 是 columnar analytical file format，不是数据库。
- 问题三：方向正确。这里除了 ad-hoc / cold data 之外，还有一个更关键的工程点：Backfill 已经产出 corrected_transfer.parquet，DuckDB 可以直接对这些 Parquet 执行 SQL，用于 Row Count、Aggregate Reconciliation、Token / Wallet Distribution 等校验，而不必先加载到 Production Sink。这使它非常适合临时 Validation 阶段。

关于你的追问：**ClickHouse 能支持并发查询。**它本身就是面向 Online Analytical Processing 的 server database，适合多人 / 多 Dashboard 并发执行分析型查询；这正是它与更偏 single-process / local / ad-hoc 的 DuckDB 的一个重要角色差异。

## 结课判定

Module 11 第 6 课理解检查通过，正式完成。已经能够区分 Postgres、ClickHouse、DuckDB 与 Parquet 的不同角色，并能根据 online serving、online analytics、ad-hoc analysis、historical file storage 等 workload 选择合适的 Storage / Engine 组合。
