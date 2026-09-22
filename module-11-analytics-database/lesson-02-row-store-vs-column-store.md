## Lesson Contract

所属 Module：Module 11 — 分析数据库

本课核心问题：

> 为什么同样是保存一张表，按“行”组织数据和按“列”组织数据，会让 OLTP 与 OLAP 的性能特征产生这么大差异？

学完以后，你应该能够解释：

- Row Store 与 Column Store 的基本存储方式。
- 为什么 Row Store 适合 point lookup / transactional workload。
- 为什么 Column Store 适合 large scan / aggregation。
- 为什么列式存储通常有更好的 compression。
- 为什么分析查询经常只读取少数列。
- 为什么“列式数据库适合分析”不是因为 SQL 写法不同，而是因为 physical layout 不同。
- 在 Blockchain Transfer / Swap 场景下，什么时候 Row Store 更合适，什么时候 Column Store 更合适。

本课不深入 ClickHouse MergeTree、Vectorized Execution、SIMD、编码算法细节；只建立 Blockchain Data Engineer 必须掌握的 storage layout 心智模型。

---

## 一、先看同一张表

假设有一张 ERC-20 Transfer 表：

fact_token_transfer

字段：

chain_id
block_number
block_time
tx_hash
log_index
token_address
from_address
to_address
amount

假设现在只有三行：

Row 1:
chain_id = 1
block_number = 100
token = USDC
from = Alice
to = Bob
amount = 100

Row 2:
chain_id = 1
block_number = 101
token = USDT
from = Carol
to = Dave
amount = 200

Row 3:
chain_id = 1
block_number = 102
token = USDC
from = Eve
to = Frank
amount = 300

逻辑上，这三行没有变化。

区别只在：

> How are these values physically laid out?

这就是 Row Store vs Column Store 的核心。

---

## 二、Row Store：一行的数据放在一起

Row Store 可以先抽象理解成：

Row 1:
[1, 100, USDC, Alice, Bob, 100]

Row 2:
[1, 101, USDT, Carol, Dave, 200]

Row 3:
[1, 102, USDC, Eve, Frank, 300]

也就是：

> values from the same row are stored together.

Postgres 典型上就是 row-oriented database。

注意，这里讲的是一个高层物理布局模型，不展开 page / tuple / MVCC 等内部实现。

---

## 三、为什么 Row Store 适合 OLTP？

假设查询：

SELECT *
FROM fact_token_transfer
WHERE tx_hash = '0xabc';

找到目标行以后，应用通常需要这条记录的多个字段：

tx_hash
token_address
from_address
to_address
amount
block_time

Row Store 的优势是：

> once you locate the row, most fields you need are physically close together.

也就是说，对“找到一行，然后读取这一整行”的 workload 很自然。

OLTP 经常就是这种访问模式：

- point lookup
- single row update
- small range
- read most columns of a row

所以：

> Row-oriented layout matches row-oriented access.

---

## 四、再看一个 OLAP Query

现在执行：

SELECT
    token_address,
    SUM(amount)
FROM fact_token_transfer
WHERE block_time >= ...
GROUP BY token_address;

假设表有 50 个字段，但这个 Query 只需要：

block_time
token_address
amount

真正需要的是 3 列。

其他 47 列：

tx_hash
from_address
to_address
log_index
...
都没有参与计算。

如果数据按行存储，扫描大量行时，底层经常会把这些不需要的列也一起读进来。

这会造成：

> unnecessary I/O.

---

## 五、Column Store：同一列的数据放在一起

同样三行数据，Column Store 可以抽象成：

chain_id:
[1, 1, 1]

block_number:
[100, 101, 102]

token_address:
[USDC, USDT, USDC]

from_address:
[Alice, Carol, Eve]

to_address:
[Bob, Dave, Frank]

amount:
[100, 200, 300]

也就是：

> values from the same column are stored together.

如果 Query 只需要：

token_address
amount

那么引擎可以主要读取：

token_address column
amount column

而不必把 from_address、to_address 等大量无关字段全部扫描。

这就是列式分析的第一个巨大优势：

> Read only the columns you need.

---

## 六、为什么这对 Blockchain Analytics 特别重要？

Blockchain Fact Table 往往会很宽。

例如 Transfer / Swap 表可能有：

chain_id
block_number
block_hash
block_time
tx_hash
tx_index
log_index
contract_address
token_address
from_address
to_address
amount_raw
amount
decimals
symbol
protocol
pool_address
is_canonical
...

可能几十个字段。

但很多分析 Query 只会用其中几列。

例如：

“每天 USDC Transfer Volume”

只需要：

block_time
token_address
amount

“某 Token 每日活跃钱包数”

可能只需要：

block_time
token_address
from_address
to_address

因此在 billions of rows 上：

> avoiding unnecessary column reads matters a lot.

---

## 七、一个简化的 I/O 对比

假设：

10 亿行
每行 100 bytes

整表大约：

100 GB

现在 Query 只需要其中 3 个字段，占总数据的 15%。

Row-oriented scan 在简化模型下可能需要接近扫描：

100 GB

Column-oriented scan 可能只需要读取：

15 GB

实际系统还会受到索引、压缩、数据跳过、缓存等影响。

但核心思想是：

> analytical queries often touch many rows but few columns.

这正是 Column Store 的优势模式。

---

## 八、Row Store 和 Column Store 的关键差异不是“行 vs 列的 SQL”

这里很容易产生一个误解。

SQL 还是一样的：

SELECT token_address, SUM(amount)
FROM transfers
GROUP BY token_address;

区别不在 SQL Syntax。

区别在：

> Physical Storage Layout.

也就是：

数据在磁盘 / 存储块上怎样组织。

所以这是一个 Physical Design 问题，而不是 Logical Schema 问题。

---

## 九、为什么 Column Store 更容易压缩？

再看 token_address：

USDC
USDC
USDC
USDC
USDT
USDT
USDT

同一列的数据通常：

- 类型相同。
- 值分布相似。
- 重复度可能高。
- 数值范围可能有规律。

因此压缩算法更容易利用这些规律。

例如：

token_address column

可能大量重复同一个 token。

block_number column

可能：

100
101
102
103
104

具有强顺序性。

这类数据比一整行混合：

address + timestamp + number + hash + string

更容易有效压缩。

所以 Column Store 的第二个重要优势是：

> better compression due to homogeneous column values.

---

## 十、Compression 为什么会让 Query 更快？

Compression 不只是省磁盘。

假设原数据：

100 GB

压缩后：

20 GB

查询从磁盘读取时，只需要传输 20 GB。

虽然 CPU 要做 decompression，但现代 CPU 往往非常快。

很多分析系统里：

> less disk I/O outweighs decompression cost.

因此：

Compression
→ less data read from disk
→ less I/O
→ often faster scans

所以列式数据库里：

> Compression is also a performance feature.

---

## 十一、为什么 Row Store 更适合 UPDATE 一行？

假设 Wallet Balance：

wallet = Alice
USDC = 1000
USDT = 500
ETH = 2

现在需要更新 Alice 的一条状态记录。

Row-oriented layout 对：

find one row
→ read row
→ update row

很自然。

而 Column Store 的设计目标通常不是：

> frequent random single-row updates.

而更倾向：

> append lots of rows, then scan many rows.

这和 Blockchain Historical Fact 非常吻合。

因为 Transfer / Swap 通常：

发生后追加进去。

很少需要像银行 Account Balance 那样持续随机 UPDATE 同一行。

---

## 十二、Blockchain 数据为什么和 Column Store 很匹配？

链上 Historical Facts 往往有几个特征：

### 1. Append-heavy

新区块不断产生：

Block N
Block N+1
Block N+2

对应 Fact 持续追加。

### 2. Immutable-ish

已经形成的 Transfer / Swap 大多数时候不会修改。

Blockchain 特殊点是 Reorg 会造成 correction，但这仍不同于普通 OLTP 中大量随机 UPDATE。

### 3. Huge volume

长期积累可达到：

millions
billions
tens of billions of rows

### 4. Analytical Query

大量：

GROUP BY
SUM
COUNT
range scan

因此：

> Blockchain historical facts are naturally column-store-friendly.

---

## 十三、但 Column Store 不是所有场景都更好

回到上一课的原则：

> workload first.

如果需求是：

“API 根据 tx_hash 查完整 Transaction。”

或者：

“查询 Alice 当前余额。”

这种请求：

- 返回行数很少。
- 希望低延迟。
- 经常需要整条记录。
- 可能涉及频繁 UPDATE。

Row Store 依然非常合适。

所以不能说：

> Column Store replaces Row Store.

更准确的是：

> Column Store complements Row Store for analytical workloads.

---

## 十四、一个很重要的访问模式对照

Row Store 更像：

> Give me this row.

Column Store 更像：

> Give me these columns across millions of rows.

这两句话非常值得记。

例如：

Wallet API：

Give me Alice's current wallet state.

→ Row Store friendly.

Analytics：

Give me amount and token_address across 5 billion transfers.

→ Column Store friendly.

---

## 十五、和你熟悉的 Oracle 联系起来

传统 Oracle / Postgres OLTP 思维经常是：

Table
→ Index
→ locate row
→ fetch row

优化重点经常围绕：

How can I avoid full table scan?

但是在 OLAP 里，有时候：

> scanning a huge amount of data is exactly what the query needs to do.

问题变成：

> How can I scan a huge amount of data as efficiently as possible?

这就是一个很重要的思维切换。

OLTP：

Avoid scans when possible.

OLAP：

Make scans cheap.

---

## 十六、为什么 Index 不能完全解决 OLAP 问题？

可能会想到：

“那我在 Postgres 上加 Index 不就可以了吗？”

Index 对：

WHERE tx_hash = X

非常有效。

因为它把：

全表扫描

变成：

快速定位少量行。

但对于：

SELECT token_address, SUM(amount)
FROM transfers
WHERE block_time BETWEEN one_year_ago AND now()
GROUP BY token_address;

如果这个 Query 本身就需要处理几亿行：

Index 不能让这些几亿行 magically disappear。

你仍然需要处理大量数据。

这时关键不是：

> locate a few rows.

而是：

> process many rows cheaply.

这是分析数据库存在的重要原因。

---

## 十七、一个工程上的简单判断方法

面对一个 Query，可以先问两个问题。

### 问题 1

How many rows will this query touch?

少量：

偏 Row Store / OLTP。

大量：

可能偏 Column Store / OLAP。

### 问题 2

How many columns will this query need?

多数列：

Row Store 没有明显劣势。

少数列：

Column Store 优势可能很大。

所以一个非常典型的 Column Store workload 是：

> many rows + few columns.

---

## 十八、将两课连接起来

第 1 课：

OLTP vs OLAP

讲的是：

> Workload Difference.

第 2 课：

Row Store vs Column Store

讲的是：

> Physical Storage Difference.

两者的关系是：

OLTP workload
→ often matches Row Store

OLAP workload
→ often matches Column Store

注意这里是：

often

不是绝对绑定。

工程系统永远要看具体 workload。

---

## 十九、从 Blockchain Data Engineer 视角总结

假设：

fact_token_transfer = 50 billion rows

Dashboard Query：

SELECT
    toDate(block_time),
    token_address,
    SUM(amount)
FROM fact_token_transfer
WHERE block_time >= ...
GROUP BY
    toDate(block_time),
    token_address;

你应该开始形成这样的判断：

1. Query touches huge number of rows.
2. Query only needs a few columns.
3. Fact is append-heavy.
4. Data types within each column are homogeneous.
5. Compression can be effective.

因此：

> Column-oriented analytical storage is a natural fit.

这比简单说：

“ClickHouse 很快”

要专业得多。

---

## 本课核心结论

> Row Store keeps values from the same row together.

> Column Store keeps values from the same column together.

> Row Store fits workloads that access a small number of rows and often need many fields from each row.

> Column Store fits workloads that scan many rows but often need only a subset of columns.

> Column Store usually enables better compression because values within a column are homogeneous.

> OLTP tries to avoid large scans; OLAP tries to make large scans cheap.

> Blockchain historical facts are often append-heavy, huge, and analytical, which makes them a natural fit for column-oriented storage.

---

## 理解检查

### 问题一

假设 Transfer 表有 50 个字段、10 亿行。

一个 Query 只需要：

block_time
token_address
amount

然后扫描过去一年所有记录做 SUM / GROUP BY。

为什么 Column Store 通常比 Row Store 更适合这个 Query？

### 问题二

为什么 Column Store 的 Compression 通常会比 Row Store 更有效？

Compression 除了节省磁盘，还有什么性能价值？

### 问题三

下面两个场景分别更偏向 Row Store 还是 Column Store？为什么？

A. 根据 wallet_address 查询一个 Wallet 当前 Balance，并返回完整状态。

B. 扫描 50 亿条 Swap，只读取 block_time、pool_address、amount_usd，然后统计每个 Pool 每天的 Volume。

## 用户回答

问题一：因为这个场景下需要扫描大量的历史数据，而返回呢只需要很小的一个数据集，所以 column store 通常比 row store 更适合这个 query

问题二：

因为 Column Store 的设计下，相同的 column 数据会放在一块，并且同一个 column 字段的数据格式都是相同的，非常固定。

所以，它很适合做 compression。做了 compression 之后，不仅可以节省磁盘，还可以提高查询效率

问题 3：A 偏向于 row store，B 偏向于 column store。

原因是：

1. A 是查询一个随机的 wallet_address，然后返回它的余额，这很适合 row store 的情况
2. B 需要扫描 50 亿 swap 数据，这个数据量非常大，然后返回一个小的数据集，这种情况下适合 column store

## 老师判断与校准

三题全部通过。

- 问题一：方向正确。需要进一步固定的关键表达是：Column Store 的优势并不主要来自“最终返回结果少”，而是这个 Query 会 touch many rows but read only a few columns（block_time、token_address、amount）。列式布局可以避免读取大量无关列，从而减少 unnecessary I/O。
- 问题二：判断正确。同一 Column 内数据类型一致、分布相似、重复或顺序规律更容易被 Compression 利用。Compression 除了节省磁盘，还会减少查询需要从存储层读取的数据量：less data read → less disk I/O → often faster scans。
- 问题三：判断正确。A 是少量行 / point lookup，并需要返回当前 Wallet 的完整状态，更偏 Row Store；B 需要扫描 50 亿行但只读取 block_time、pool_address、amount_usd 三列，并进行 aggregation，是典型 many rows + few columns，更偏 Column Store。

## 结课判定

Module 11 第 2 课理解检查全部通过，正式完成。已经能够从 Physical Storage Layout 解释 Row Store 与 Column Store 的差异，并建立核心判断：Row Store 更适合 small number of rows + many fields；Column Store 更适合 many rows + few columns，同时利用更高 Compression 降低大规模 Scan 的 I/O 成本。
