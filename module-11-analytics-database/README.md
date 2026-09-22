# Module 11 — 分析数据库

## Module Contract

### Module 目标

把前面已经掌握的 Blockchain ETL / Streaming / Modeling 数据，推进到“如何高效存储、查询和服务分析负载”。

本 Module 重点不是背数据库产品，而是建立一个 Data Engineer 的核心判断：

> Different workloads need different storage and query engines.

完成后，应能根据数据规模、写入模式、查询模式、延迟要求和成本，在 Postgres、ClickHouse、DuckDB 等工具之间做合理选择，并解释为什么。

### 必须掌握

- OLTP vs OLAP 的基本区别。
- Row-oriented vs Column-oriented 的核心差异。
- 为什么 Postgres 擅长事务型 / serving workload，但大规模扫描分析会变贵。
- 为什么 ClickHouse 适合大规模 append-heavy analytics。
- 为什么 DuckDB 适合本地 / 嵌入式 / ad-hoc analytics。
- Partitioning、Ordering / Primary Key、Data Skipping、Compression 的基本工程意义。
- Blockchain workload 的典型特点：大规模 append、时间范围扫描、地址 / token / protocol 聚合、历史分析。
- 如何把 Raw / Normalized / DWS / ADS 映射到不同存储层和查询引擎。
- 能设计一套基础 Blockchain Analytics Storage Architecture。

### 可以了解

- ClickHouse MergeTree 家族的高层概念。
- Materialized View、Projection、TTL 等高层用途。
- DuckDB + Parquet 的本地分析模式。
- Postgres Index / Partitioning 与分析型数据库的边界。

### 本 Module 不展开

- ClickHouse 底层源码、MergeTree 内部合并算法。
- PostgreSQL MVCC / WAL 内核实现。
- DuckDB Vectorized Execution 内核细节。
- 分布式数据库一致性协议。
- 云厂商具体产品采购与价格对比。

### 结束标准

完成本 Module 后，应能回答：

1. 为什么一个 Blockchain Data Platform 不能只用一种数据库解决所有问题？
2. Postgres、ClickHouse、DuckDB 各自适合什么 workload？
3. 为什么列式存储适合分析型查询？
4. 为什么 Partition / Order Key 会影响大表查询性能？
5. 如何为 Transfer / Swap / Wallet Analytics 设计分析存储？
6. Realtime Serving 与 Historical Analytics 为什么可能需要不同 Sink？
7. 如何在性能、成本、复杂度之间做取舍？

## 课程结构

1. 第 1 课｜为什么需要分析数据库：OLTP、OLAP 与不同 Workload
2. 第 2 课｜Row Store vs Column Store：为什么列式数据库适合分析
3. 第 3 课｜Postgres 的位置：什么时候够用，什么时候开始吃力
4. 第 4 课｜ClickHouse：为什么适合 Blockchain Analytics
5. 第 5 课｜Partition、Order Key、Data Skipping 与大表查询
6. 第 6 课｜DuckDB + Parquet：本地分析与低成本历史查询
7. 第 7 课｜Serving DB + Analytics DB：混合存储架构
8. 结束标准综合检查｜设计 Blockchain Analytics Storage Architecture

## 当前学习进度

- 当前阶段：第三阶段 Data Engineering
- 当前 Module：Module 11 — 分析数据库
- 已完成 Lesson：第 1 课｜为什么需要分析数据库：OLTP、OLAP 与不同 Workload；第 2 课｜Row Store vs Column Store：为什么列式数据库适合分析；第 3 课｜Postgres 的位置：什么时候够用，什么时候开始吃力
- 当前 Lesson：第 4 课｜ClickHouse：为什么适合 Blockchain Analytics
- 当前状态：第 3 课理解检查通过并已完成同步；准备进入第 4 课
- 下一步准确入口：开始 Module 11 第 4 课｜ClickHouse：为什么适合 Blockchain Analytics。

## 课程路径

- [第 1 课｜为什么需要分析数据库：OLTP、OLAP 与不同 Workload](lesson-01-oltp-olap-workloads.md)
- [第 2 课｜Row Store vs Column Store：为什么列式数据库适合分析](lesson-02-row-store-vs-column-store.md)
- [第 3 课｜Postgres 的位置：什么时候够用，什么时候开始吃力](lesson-03-postgres-boundary.md)
