## Lesson Contract

所属 Module：Module 11 — 分析数据库

本课核心问题：

> 为什么一个已经有 Postgres 的 Blockchain Data Platform，还需要 ClickHouse、DuckDB 这类分析数据库？本质上到底是“数据库品牌不同”，还是 Workload 不同？

学完以后，你应该能够解释：

- OLTP 与 OLAP 的核心差异。
- 为什么“查一条钱包记录”和“扫描 10 亿条 Transfer 做聚合”是两种完全不同的 workload。
- 为什么 Postgres 并不是“不好”，而是它的设计目标和大规模分析 workload 不完全相同。
- 为什么 Blockchain Analytics 天然更偏 OLAP。
- 为什么 Data Engineer 做数据库选型时应该先看 Query Pattern，而不是先看产品名。

本课不深入 ClickHouse 内部实现、列式存储细节、MergeTree、DuckDB Vectorized Execution；这些留到后续课程。

---

## 一、先从一个你非常熟悉的 SQL 场景开始

假设我们有一张 Ethereum ERC-20 Transfer 表：

fact_token_transfer

字段包括：

chain_id
block_number
block_time
tx_hash
log_index
token_address
from_address
to_address
amount

现在有两个查询。

查询 A：

“查 tx_hash = 0xabc 的这一笔 Transfer。”

查询 B：

“统计过去 180 天，USDC 每天的 Transfer Volume，并按 Wallet 分组，扫描 20 亿条记录。”

两条都是 SQL。

但它们不是同一种问题。

Query A 的核心是：

> Find a few rows quickly.

Query B 的核心是：

> Scan and aggregate a huge amount of data efficiently.

这就是 OLTP / OLAP 分界的入口。

---

## 二、OLTP：面向小范围、低延迟、事务型访问

OLTP = Online Transaction Processing。

典型特点：

- 单条 / 少量记录查询。
- 高频 INSERT / UPDATE / DELETE。
- 依赖 Primary Key / Index 快速定位。
- 强调事务一致性。
- 单次查询涉及的数据量通常较小。
- 响应延迟要求低。

例如银行：

SELECT *
FROM account
WHERE account_id = 'A123';

或者：

UPDATE account
SET balance = ...
WHERE account_id = 'A123';

这是典型 OLTP。

从数据访问角度看：

> I know roughly what row I want.

---

## 三、Postgres 为什么很适合这类场景？

Postgres 是非常成熟的通用关系型数据库。

对于：

- wallet profile
- user account
- job state
- checkpoint
- small-to-medium fact table
- API serving
- metadata
- transactional update

它非常适合。

例如：

SELECT *
FROM wallet_balance
WHERE chain_id = 1
AND wallet_address = '0x...';

这种查询通过 Index 很容易快速定位。

所以我们不能形成一个错误印象：

> Postgres slow, ClickHouse fast.

这句话没有意义。

正确的问题是：

> Fast for what workload?

---

## 四、OLAP：面向大范围扫描和聚合

OLAP = Online Analytical Processing。

典型特点：

- 扫描大量历史数据。
- GROUP BY。
- SUM / COUNT / AVG。
- 时间范围查询。
- 多维聚合。
- 很少逐行 UPDATE。
- 更关注 scan throughput 和 aggregation performance。

例如：

SELECT
    date(block_time),
    token_address,
    SUM(amount)
FROM fact_token_transfer
WHERE block_time >= now() - interval '180 days'
GROUP BY
    date(block_time),
    token_address;

这不是“找几行”。

这是：

> scan millions / billions of rows and aggregate them.

这就是 OLAP。

---

## 五、Blockchain Analytics 为什么天然偏 OLAP？

想一下链上分析常见问题：

“过去一年 USDC 转账量是多少？”

“过去 30 天 Uniswap 每个 Pool 的交易量是多少？”

“某个 Wallet 过去两年的所有 Token Flow 是什么？”

“每天有多少 Active Wallet？”

“哪个 Token 的 Transfer Count 增长最快？”

这些查询都有共同特征：

> Historical Scan + Filter + Group By + Aggregate.

因此 Blockchain Analytics 的核心 workload 往往是：

> append-heavy + historical + analytical.

这和典型银行核心交易系统非常不同。

---

## 六、银行系统类比

你可以把它类比成：

银行核心系统：

customer
account
transaction status

需要：

> 快速定位一条记录、事务更新。

这是 OLTP。

数据仓库：

fact_transaction
fact_transfer
dws_customer_daily_flow

需要：

> 扫描大量历史、统计、聚合、报表分析。

这是 OLAP。

所以你以前接触的：

Oracle OLTP
→ ETL
→ Data Warehouse

和 Blockchain 世界：

Postgres / Operational Store
→ ETL / Stream
→ ClickHouse / Analytical Store

背后的思想是一样的。

---

## 七、为什么不能只用一套数据库？

技术上当然可以。

小规模系统里：

> Postgres can do both.

问题是随着规模增长，两类 workload 会互相干扰。

例如：

API 正在执行：

SELECT balance
FROM wallet_balance
WHERE wallet_address = 'Alice';

同时 Analyst 执行：

SELECT token_address, SUM(amount)
FROM transfer
WHERE block_time BETWEEN ...
GROUP BY token_address;

第二个 Query 可能扫描几亿行，占用大量：

CPU
Memory
Disk I/O

结果可能是：

Analytics workload
↓
consumes resources
↓
Operational query latency increases

所以问题不只是“分析 Query 慢”。

还可能：

> analytical workloads interfere with serving workloads.

这就是 workload isolation 的价值。

---

## 八、从 Data Engineer 视角理解数据库选型

不要先问：

> Which database is best?

应该先问：

> What workload are we serving?

至少要看：

Query Pattern
Data Volume
Write Pattern
Update Frequency
Latency Requirement
Concurrency
Cost

例如：

Wallet Balance API：

- point lookup
- latest state
- low latency
- frequent update

更接近 OLTP / Serving。

Historical Transfer Analytics：

- huge scan
- aggregation
- append-heavy
- historical query

更接近 OLAP。

---

## 九、Postgres、ClickHouse、DuckDB 先建立位置感

这一课暂时不深入实现，只建立地图。

Postgres：

> General-purpose relational / transactional / serving database.

适合在线业务、元数据、状态表、规模可控的数据集和复杂关系模型。

ClickHouse：

> Analytical database designed for high-throughput columnar analytics.

适合大规模历史 Fact、时间序列式 append、聚合和 Dashboard。

DuckDB：

> Embedded analytical database.

适合本地分析、Parquet、Ad-hoc Query、开发调试、小型分析任务。

所以三者不是：

Gold / Silver / Bronze。

而是：

> different tools for different workloads.

---

## 十、一个典型 Blockchain Data Platform

可以先形成这样的心智模型：

Ethereum / Provider
↓
Indexer
↓
Kafka / ETL
↓
Normalized Facts
↓
├── Postgres
│   └── API / Wallet State / Metadata
│
├── ClickHouse
│   └── Dashboard / Large-scale Analytics
│
└── Parquet + DuckDB
    └── Local Research / Ad-hoc Analysis

注意：

这不是固定架构。

而是在表达：

> one source, multiple serving engines.

---

## 十一、为什么 Query Pattern 比数据库品牌重要？

假设你有 10 亿行 Transfer。

问题 1：

“查 tx_hash = X。”

问题 2：

“统计每个月所有 Token 的 Transfer Volume。”

即使数据完全相同：

同一张逻辑 Fact。

最佳物理访问方式也可能完全不同。

所以：

> Logical Data Model can be shared, while Physical Storage can differ.

这也是 Data Modeling 和 Storage Design 的区别。

Module 8 更关注：

> What does this row mean?

Module 11 更关注：

> How should these rows be physically stored and queried efficiently?

---

## 十二、不要把 Data Warehouse 和 Database Product 混为一谈

Data Warehouse 是一种数据组织 / 分析体系。

ClickHouse 是一种数据库产品。

Postgres 也可以承载小型 Warehouse。

Parquet + DuckDB 也能完成很多 analytical workload。

所以：

> Warehouse is an architectural role; database is an implementation choice.

这点很重要。

---

## 十三、OLTP vs OLAP 的核心对照

OLTP：

- few rows
- point lookup
- frequent updates
- transactions
- low latency
- serving application

OLAP：

- many rows
- large scans
- aggregation
- append-heavy
- historical analysis
- dashboards / analytics

可以压缩成一句：

> OLTP asks “what is this record now?”
> OLAP asks “what patterns exist across a lot of records?”

---

## 十四、回到 Blockchain Data Engineer

以后看到一个数据库选型问题，不要直接说：

“链上数据量大，所以 ClickHouse。”

这还不够。

你应该能够说：

> 我们的 Transfer Fact 是 append-heavy；核心 Query 是按时间范围扫描、按 token / wallet / protocol 聚合；数据规模可能达到 billions of rows，因此 workload 更偏 OLAP。为了获得更高 scan throughput、压缩率和聚合性能，可以把 Historical Analytics 放到 ClickHouse，而把 Wallet State / Job State / API Serving 放在 Postgres。

这才是完整的数据工程判断。

---

## 本课核心结论

> Database choice should follow workload, not product popularity.

> OLTP focuses on small, transactional, low-latency access.

> OLAP focuses on large scans, aggregation, and historical analytics.

> Blockchain analytics is naturally OLAP-heavy because chain data is append-heavy and historical queries often scan large ranges.

> Postgres, ClickHouse, and DuckDB are not simple substitutes; they occupy different positions in a data architecture.

> Logical data modeling and physical storage design are related but different problems.

---

## 理解检查

### 问题一

有两个查询：

A. 根据 tx_hash 查询一笔 Transfer。

B. 扫描过去一年 10 亿条 Transfer，统计每个 Token 每天的 Transfer Volume。

哪一个更偏 OLTP，哪一个更偏 OLAP？为什么？

### 问题二

为什么不能简单地说：

“ClickHouse 比 Postgres 快，所以应该全部换成 ClickHouse”？

请从 Workload 的角度回答。

### 问题三

假设一个 Wallet Analytics 产品同时需要：

1. API 实时返回某个 Wallet 当前 Token Balance。
2. Dashboard 查询过去一年每个月的 Wallet Transfer Volume。

你会倾向让这两个 workload 使用完全相同的数据库角色吗？为什么？

## 用户回答

问题一：A 偏向于 OLTP，B 偏向于 OLAP

问题二：ClickHouse 和 Postgres 它们是面向不同 workload 的数据库产品。PostgreSQL 是面向小范围、低延迟的事务型访问，而 ClickHouse 是面向大范围扫描和聚合的数据库产品

问题三：这两个 workload 应该是用不相同的数据库角色。

1. API 实时返回某个 wallet 当前 token balance：
   这个是小范围查询，即时性要求高。针对这种小范围、低延迟查询，可以使用 PostgreSQL，属于 OLTP 型数据库。
2. Dashboard 查询过去一年每个月的 wallet transfer volume：
   这个属于 OLAP 型数据库。

## 老师判断与校准

三题全部通过。

- 问题一：判断正确。根据 tx_hash 查询单条 / 少量记录属于 point lookup，更偏 OLTP；扫描一年、10 亿条 Transfer 并按 Token / 日期聚合属于 large scan + aggregation，更偏 OLAP。
- 问题二：判断正确。ClickHouse 与 Postgres 面向的是不同 workload。数据库选型的核心不是“谁更快”，而是具体 Query Pattern、Data Volume、Latency Requirement 和 Write / Update Pattern。
- 问题三：判断正确。实时 Wallet Balance 更偏 Serving / OLTP，适合低延迟点查和频繁状态更新；历史 Wallet Transfer Volume 更偏 OLAP，适合大范围扫描和聚合。两者可以共享逻辑数据模型，但物理存储和查询引擎角色可以不同。

## 结课判定

Module 11 第 1 课理解检查全部通过，正式完成。已建立本 Module 的核心判断框架：Database choice should follow workload。能够区分 OLTP 与 OLAP，并能从 Query Pattern 推导 Postgres / ClickHouse 等数据库在架构中的不同角色。
