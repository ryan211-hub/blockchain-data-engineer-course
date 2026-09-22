## Lesson Contract

所属 Module：Module 11 — 分析数据库

本课核心问题：

> Postgres 到底什么时候“够用”，什么时候开始因为 workload 变化而变得不经济？问题是 Postgres 不够强，还是它承担了不适合自己的角色？

学完以后，你应该能够解释：

- Postgres 在 Blockchain Data Platform 中适合承担哪些角色。
- 为什么 Postgres 可以做一定程度的分析，但不是所有 OLAP workload 的最佳选择。
- 为什么“数据量大”本身并不是立刻放弃 Postgres 的理由。
- 什么样的 Query Pattern、Data Volume、Concurrency、Latency Requirement 会让 Postgres 开始吃力。
- Index、Partitioning、Materialized View 等手段能解决什么、不能解决什么。
- 为什么 Serving Workload 与 Large-scale Analytics 经常需要分离。
- 什么时候继续优化 Postgres，什么时候引入 ClickHouse 这类分析数据库。

本课不深入 PostgreSQL MVCC、WAL、Buffer Manager、Planner 内核实现；只讨论 Blockchain Data Engineer 在架构选型中需要掌握的边界。

---

## 一、先纠正一个常见误解

很多人在接触 ClickHouse 之后会形成一个过度简单的判断：

> 数据量大 → Postgres 不行 → 换 ClickHouse。

这个判断太粗糙。

更准确的问题是：

> What workload is making Postgres expensive?

因为 Postgres 并不是“只能处理小数据”。

它完全可以处理：

- millions of rows
- tens of millions of rows
- even much larger datasets

真正决定它是否合适的，不只是 table size。

还包括：

- Query Pattern
- Selectivity
- Scan Range
- Concurrency
- Update Pattern
- Index Strategy
- Latency Requirement
- Hardware / Cost

所以：

> Big table does not automatically mean wrong database.

---

## 二、Postgres 在 Blockchain Data Platform 中的位置

从我们前两课的框架看：

Postgres 更偏：

> General-purpose relational + transactional + serving database.

在 Blockchain Data Platform 里，它很适合：

### 1. Serving State

例如：

wallet_balance
wallet_profile
token_metadata
protocol_metadata

特点：

- point lookup
- small result set
- low latency
- may update frequently

### 2. Operational State

例如：

indexer_checkpoint
backfill_job
pipeline_status
consumer_state

这些本来就非常适合关系型数据库。

### 3. Small / Medium Fact Tables

例如：

recent_transfers
recent_swaps
alerts

如果数据量和 Query Pattern 仍然可控，Postgres 完全可以承担。

### 4. API Backend

例如：

GET /wallet/0xabc/balance

底层：

SELECT ...
WHERE wallet_address = ...

这是典型 Serving Workload。

---

## 三、为什么 Postgres 也能做 Analytics？

Postgres 并不是不能：

GROUP BY
SUM
COUNT
JOIN
Window Function
CTE
Materialized View

事实上，它有非常强的 SQL 能力。

小到中等规模分析中：

Postgres can be perfectly fine.

例如：

统计最近 7 天 USDC Transfer Volume。

如果数据只有几百万行，或者查询范围很小：

完全没有必要为了“听起来更专业”就引入 ClickHouse。

这涉及一个很重要的架构原则：

> Avoid unnecessary system complexity.

每多一种数据库，就多一套：

deployment
monitoring
backup
schema sync
ETL / CDC
permission
cost
on-call burden

所以：

> If Postgres meets the SLA at acceptable cost, it may already be the right choice.

---

## 四、什么时候 Postgres 开始吃力？

不是某个固定行数。

没有一个宇宙常数：

100 million rows → must use ClickHouse.

真正要看 workload。

我们从几个维度来看。

---

## 五、信号一：Query 开始频繁做 Large Scan

假设：

fact_token_transfer = 5 billion rows

Dashboard 每分钟执行：

“过去一年每天每个 Token 的 Transfer Volume。”

Query 需要扫描：

大量历史数据
+
少量列
+
GROUP BY
+
SUM

这已经是典型：

many rows + few columns.

而 Postgres 是 Row Store。

即使 SQL 没问题，Physical Layout 也意味着它在这种场景下不是最自然的选择。

这时问题不是：

SQL 写得差。

而是：

> workload and storage layout are mismatched.

---

## 六、信号二：Index 已经帮不了太多

Index 最擅长：

> locate a small subset of rows.

例如：

WHERE tx_hash = '0xabc'

或者：

WHERE wallet_address = 'Alice'
AND block_time > now() - interval '1 day'

如果 Selectivity 很高：

Index 非常有效。

但假设：

WHERE block_time >= one_year_ago

结果命中全表 60% 的数据。

这时候即使有 Index：

你还是要处理海量行。

所以：

> Index helps you avoid scanning irrelevant rows.
> It does not make relevant rows disappear.

如果业务问题天然要求分析几亿、几十亿行：

Index 不能根本改变 workload。

---

## 七、Selectivity 是一个很重要的词

Selectivity 可以先简单理解为：

> 查询条件最终筛掉多少数据。

例如：

10 亿行里，只查 10 行。

高 Selectivity。

10 亿行里，查 6 亿行。

低 Selectivity。

对于：

WHERE tx_hash = X

通常：

very selective.

对于：

WHERE block_time >= 2026-01-01

可能：

not selective enough.

所以：

Index 对前者很强。

对后者帮助可能有限。

这个词以后你看 Query Plan 会经常遇到。

---

## 八、信号三：Analytics Query 开始影响 Serving Query

这点非常关键。

假设同一个 Postgres 同时承担：

Wallet Balance API
+
Historical Analytics

API：

SELECT balance
FROM wallet_balance
WHERE wallet_address = 'Alice';

要求：

P99 < 100 ms

与此同时 Dashboard：

扫描几亿 Transfer 做 GROUP BY。

Analytics 会大量消耗：

CPU
Memory
Disk I/O
Buffer Cache
Worker

于是：

Dashboard workload
↓
resource contention
↓
API latency becomes unstable

这时问题已经不是：

“这个 SQL 能不能跑完？”

而是：

> Can this mixed workload meet both SLAs at the same time?

这就是 Workload Isolation。

---

## 九、银行系统类比

你很熟悉这种思路。

银行核心交易数据库不会希望：

白天高峰期有人直接在核心库上跑：

过去三年所有交易
GROUP BY branch
GROUP BY customer
GROUP BY day

即使 SQL 能跑。

原因不是：

Oracle 不支持 GROUP BY。

而是：

> analytical workload should not interfere with operational workload.

所以才会有：

Core DB
↓
ETL
↓
Warehouse

Blockchain 平台也是同样逻辑。

---

## 十、信号四：Concurrency 上来了

单个分析 Query 跑 5 秒，可能还能接受。

但如果：

100 个 Dashboard
50 个 Analyst
20 个 API service

同时查询同一套大 Fact：

问题会迅速放大。

分析型 workload 的一个特点是：

single query may be heavy.

当并发增加：

CPU
Memory
I/O

都会出现竞争。

所以数据库选型不能只 benchmark：

one query.

还要考虑：

> concurrent workload.

---

## 十一、信号五：Storage Cost 开始不经济

Row Store 对 Wide Fact：

可能存很多重复字段。

Blockchain 数据又往往：

append-heavy
high volume
long retention

比如：

50 billion Transfer rows

如果每行都包含：

chain_id
block_number
block_time
token_address
addresses
amount
metadata

Row Store 的 storage footprint 可能很大。

而 Column Store 往往：

better compression
→ lower storage
→ less I/O

所以有时迁移到分析数据库，不只是为了“更快”。

也是：

> better cost efficiency for historical analytical data.

---

## 十二、Postgres Partitioning 能不能解决？

可以解决一部分问题。

例如按月份 Partition：

transfers_2026_01
transfers_2026_02
transfers_2026_03

如果 Query：

WHERE block_time BETWEEN '2026-03-01' AND '2026-03-31'

数据库可以只访问 March Partition。

这叫：

Partition Pruning.

它非常有价值。

但是注意：

如果 Query 就是：

查过去两年所有数据。

那还是需要访问很多 Partition。

所以 Partitioning 解决的是：

> reduce irrelevant data access.

它不能改变：

> this query genuinely needs to process a huge amount of relevant data.

---

## 十三、Materialized View 能不能解决？

也可以解决一部分问题。

例如原始 Transfer：

10 billion rows

Dashboard 常查：

daily_token_volume

你可以提前聚合：

dws_token_daily_volume

这样 Dashboard 不再每次扫 10 billion rows。

这是我们 Module 8 / 9 已经学过的思想：

Precompute
→ reduce query cost

这时候即使底层是 Postgres：

也可能跑得很好。

所以不要把问题变成：

Postgres vs ClickHouse 二选一。

实际上还有：

Data Modeling
Aggregation Layer
Materialized View
Partitioning
Index
Caching

这些优化手段。

---

## 十四、那什么时候应该继续用 Postgres？

可以用一个工程化判断。

如果满足：

- dataset still manageable
- queries are selective
- dashboard mostly hits DWS / ADS
- SLA is satisfied
- concurrency is moderate
- storage cost acceptable
- team wants operational simplicity

那么：

> Stay with Postgres.

因为：

one database
→ simpler operations
→ fewer moving parts

架构不是越复杂越高级。

---

## 十五、什么时候考虑引入 ClickHouse？

如果逐渐出现：

- billions / tens of billions rows
- frequent historical scans
- many rows + few columns
- heavy GROUP BY / aggregation
- high dashboard concurrency
- analytical queries interfere with serving
- storage / compression cost becomes significant
- Postgres tuning gets increasingly expensive

这时：

> introduce an analytical engine.

注意措辞：

不是：

replace Postgres.

而往往是：

add ClickHouse.

例如：

Postgres
→ Serving / Operational State

ClickHouse
→ Historical Analytics

这就是角色分工。

---

## 十六、不要用“迁移”思维，要用“分层”思维

常见误区：

“We moved from Postgres to ClickHouse.”

现实更常见的是：

“We kept Postgres for serving and introduced ClickHouse for analytics.”

架构变成：

Indexer / ETL
↓
Normalized Facts
↓
├── Postgres
│   └── current state / API / metadata
│
└── ClickHouse
    └── historical facts / dashboards / analytics

这和上一课的：

one source, multiple serving engines

完全一致。

---

## 十七、一个非常实际的 Blockchain 例子

假设你做 Wallet Analytics。

需求 1：

查询 Alice 当前 USDC Balance。

典型：

SELECT balance
FROM wallet_token_balance
WHERE chain_id = 1
AND wallet_address = 'Alice'
AND token_address = 'USDC';

特点：

few rows
low latency
current state

Postgres 很适合。

需求 2：

查询 Alice 过去 5 年所有 Token Flow，并按月、Token、Protocol 聚合。

可能扫描：

hundreds of millions of activities

特点：

historical
large scan
aggregation
few columns

这就更适合 ClickHouse。

---

## 十八、Reorg 对两种数据库角色也不同

[Blockchain Data Engineer 视角]

Postgres 里如果保存：

current_wallet_balance

Reorg Correction 可能需要：

UPDATE current state.

而 ClickHouse 里保存：

historical_transfer_fact

可能更偏：

append corrected version
mark canonical state
rebuild aggregate

两者面对的是不同 Data Semantics。

所以：

Serving State
和
Historical Fact

不仅 Query Pattern 不同。

Correction Pattern 也可能不同。

---

## 十九、Postgres 的“吃力”不是失败，而是信号

这句话很重要：

> Postgres becoming expensive is often a workload signal, not a database failure.

也就是说：

不是 Postgres “垃圾”。

而是你的系统已经从：

operational workload

成长到了：

large-scale analytical workload.

这其实是架构演进。

---

## 二十、一个判断框架

以后你可以用这 6 个问题判断：

1. How many rows does the query touch?
2. How many columns does it need?
3. How selective are the filters?
4. How often does the data update?
5. What latency / concurrency SLA do we need?
6. Is analytics interfering with serving?

如果答案越来越像：

many rows
few columns
low selectivity
append-heavy
high concurrency
heavy aggregation

就越来越偏向：

Column Store / Analytical DB.

---

## 二十一、把前三课连起来

第 1 课：

OLTP vs OLAP
→ Workload.

第 2 课：

Row Store vs Column Store
→ Physical Layout.

第 3 课：

Postgres 什么时候够用
→ Architecture Boundary.

三课的逻辑：

Workload
↓
Storage Pattern
↓
Database Role

---

## 本课核心结论

> Postgres is not “slow”; it is optimized for a different workload profile.

> Large table size alone does not mean Postgres is the wrong choice.

> The real warning signs are large scans, low-selectivity analytical queries, high concurrency, serving interference, and rising cost.

> Indexes and Partitioning reduce irrelevant access, but cannot eliminate genuinely relevant large-scale scans.

> Materialized Views / DWS / ADS can extend how far Postgres remains practical.

> When analytical workload becomes dominant, the usual move is not “replace Postgres”, but “separate roles”.

> Postgres often remains the Serving / Operational database, while ClickHouse becomes the Historical Analytical database.

---

## 理解检查

### 问题一

一张 Transfer 表已经有 5 亿行。

能不能仅凭“5 亿行”就判断必须从 Postgres 迁移到 ClickHouse？

为什么？

### 问题二

假设 Postgres 上已经有 block_time Index。

一个 Query 要查询过去两年 70% 的 Transfer，然后 GROUP BY token_address。

为什么这个 Index 仍然不能根本解决 OLAP 性能问题？

### 问题三

一个 Wallet 产品同时有：

1. Current Balance API，要求 P99 < 100ms。
2. Historical Dashboard，经常扫描几十亿条 Transfer。

如果两个 workload 都压在同一个 Postgres 上，最大的架构风险是什么？

你会更倾向：

A. 完全替换 Postgres。

还是：

B. 让 Postgres 和 ClickHouse 分别承担不同角色。

为什么？