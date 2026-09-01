# 第3课｜Archive Node 为什么昂贵：Historical State Retention 的成本模型

## Lesson Contract

【课程边界视角】

- 所属 Module：Module 6 — Archive Node
- 本课核心问题：Archive Node 只是“少做了 Pruning”，为什么成本却会高这么多？
- 学完后应能回答：Archive Node Cost = Storage + Write / I/O + Sync + Database Maintenance + Operations；并能够判断为什么很多数据公司宁愿购买 Archive RPC，也不一定自己维护 Archive Node。
- 本课明确不展开：Geth / Erigon / Reth 具体磁盘格式；Trie GC / Snapshot 内核算法；RAID、NVMe 参数调优；云厂商具体价格。我们关心的是成本模型。

## 一、先纠正一个直觉

上一课我们建立了：

```text
Pruned Node
→ 删除部分旧 State

Archive Node
→ 保留 Historical State
```

很容易因此产生一个直觉：

> 那 Archive Node 不就是多占一点硬盘吗？

问题就在这个“一点”。

Archive Node 的成本并不是：

```text
Full Node
+
多几百 GB
```

更合理的理解是：

```text
更多数据被长期保留
        ↓
数据库越来越大
        ↓
写入、读取、维护成本一起上升
        ↓
同步、备份、迁移、恢复也一起变重
```

所以 Storage 只是最表面的一层。

## 二、第一个成本：Storage

【物理存储视角】

先看普通节点。

假设状态不断演进：

```text
State 100
↓
State 101
↓
State 102
↓
...
↓
State 100000
```

Pruned Node 可以逐渐清理：

```text
old State
old State
old State
    ↓
  Prune
```

因此它主要维持：

```text
Current State
+
Recent State
+
必要 History
```

Archive Node 则不能这么激进。

它需要维持：

```text
Current State

+

Historical State Information
```

所以随着链运行：

```text
时间 →
Historical State Data
████
████████
████████████
████████████████
```

它天然具有一个特点：

> **数据集长期单向增长。**

普通业务数据库经常可以：

```text
归档
删除
TTL
冷热分层
```

但 Archive Node 的核心卖点恰恰是：

```text
过去的数据还能查
```

你不能为了节省空间，把最核心的能力删掉。

## 三、但真正麻烦的不只是“磁盘容量”

这里开始进入工程上更重要的一层。

假设：

```text
Node A DB = 500 GB
Node B DB = 8 TB
```

并不是说：

> B 只需要买一块更大的硬盘。

因为数据库越大，很多操作也会变重。

例如：

```text
查找
写入
Compaction / Maintenance
Backup
Restore
Migration
Integrity Check
```

都会受到影响。

所以：

```text
Storage Size
```

不是孤立变量。

它会放大：

```text
I/O Cost
+
Operational Cost
```

## 四、第二个成本：Write / I/O

【节点执行视角】

Ethereum 每产生一个 Block：

```text
Block N
↓
EVM Execution
↓
State Changes
```

例如：

```text
Alice balance:
100 → 90

USDC balance:
5000 → 6000

Uniswap pool:
reserve changes

Aave:
position changes
```

节点必须把这些变化写入数据库。

对于 Pruned Node：

```text
新数据写入

+

旧 State 可以逐渐删除
```

而 Archive Node：

```text
新数据写入

+

历史数据继续保留
```

因此数据库持续膨胀。

这里就出现一个非常现实的问题：

> 数据不是“写进去一次就结束了”。

在底层 KV / LSM Storage Engine 中，数据还可能经历：

```text
Write
↓
Flush
↓
Compaction
↓
Rewrite
```

你在 Module 4 已经接触过：

```text
Write Amplification
```

所以应用层：

```text
写入 1 GB 数据
```

并不意味着 SSD 只发生：

```text
1 GB Physical Write
```

底层可能产生更多实际 I/O。

这也是为什么：

> **Archive Node 的磁盘不仅要“大”，通常还要“快”。**

容量很大的廉价 HDD，并不自动适合高负载 Execution Client。

## 五、第三个成本：Read I/O

这部分容易被忽略。

Archive Node 存在的目的不是：

```text
把历史 State 放在那里收藏
```

而是：

```text
Historical State Queryability
```

例如用户请求：

```text
eth_getBalance(
  Alice,
  block=15,000,000
)
```

或者：

```text
eth_call(
  someContract,
  block=15,000,000
)
```

节点需要从一个可能非常大的数据库中定位相应历史状态。

所以：

```text
Archive Node

不是只有
Write-heavy

还可能是
Read-heavy
```

尤其 RPC Provider 面对大量用户时：

```text
User 1 → block 12M
User 2 → block 18M
User 3 → block 9M
User 4 → block 20M
...
```

访问模式可能非常随机。

这意味着：

```text
Random Read
+
Large Working Set
```

会持续给：

```text
SSD
Cache
Memory
Database
```

造成压力。

## 六、第四个成本：Sync

现在假设你想新建一台 Archive Node。

一个误解是：

> 下载完 Blockchain 就行。

不是。

【节点执行视角】

我们之前已经学过：

```text
Block
≠
State
```

Archive Node 不仅要得到历史 Block，还必须构建它需要保存的 Historical State 信息。

最直观的逻辑模型是：

```text
Genesis
↓
Block 1
↓
Execute
↓
State 1
↓
Block 2
↓
Execute
↓
State 2
↓
...
↓
Current Block
```

所以 Archive 能力的构建，本身就是非常重的任务。

即使现代 Client 会采用各种优化方式，我们在这一课不用研究具体实现。

你只需要理解：

```text
Archive Node Sync
```

通常意味着：

> 不只是“下载大量历史数据”，还涉及“执行、构建并保存大量状态历史信息”。

因此：

```text
CPU
+
Disk Write
+
I/O
+
Time
```

全部成为成本。

## 七、这里出现一个重要的生产环境问题

假设你公司只有：

```text
1 台 Archive Node
```

有一天机器故障了。

如果只是普通应用服务器：

```text
重新部署
→ 几分钟
```

但 Archive Node：

```text
硬盘损坏
↓
重新建节点
↓
重新获取大量数据
↓
恢复 Historical State
↓
追上 Current Chain
```

恢复成本可能非常高。

所以生产环境不会只考虑：

```text
运行成本
```

还要考虑：

```text
Recovery Cost
```

这就把问题引向下一层。

## 八、第五个成本：Operations

【基础设施运维视角】

假设你拥有一个巨大的 Archive Node 数据库。

你需要处理：

```text
磁盘容量监控
SSD 健康度
DB corruption
Client Upgrade
Backup
Restore
Failover
Monitoring
Performance
RPC Load
```

而且区块链有一个特殊性质：

```text
它不会停
```

Ethereum：

```text
Block
Block
Block
Block
...
```

持续产生。

因此系统面对的是：

> 一个持续增长、持续写入、持续服务查询的数据库。

这和一个：

```text
写完后基本不动的 8 TB 数据仓库
```

不是同一种运维问题。

## 九、一个很适合数据工程师的类比

可以把 Archive Node 和一个银行系统比较。

假设银行规定：

普通系统：

```text
交易流水：永久保存
当前余额：保存
过去每日余额：只保存最近 90 天
```

Archive 系统：

```text
交易流水：永久保存
当前余额：保存
过去每一个时间点的余额查询能力：
长期保存
```

第二种系统带来的不只是：

```text
多一张历史余额表
```

而是：

```text
历史数据持续增长
↓
索引持续增长
↓
查询 Working Set 变大
↓
备份变大
↓
恢复时间变长
↓
灾备机器也要更大
↓
测试环境也可能需要更多资源
```

所以真实基础设施成本是连锁反应。

## 十、一个非常重要的成本公式

现在把它压缩成一个模型：

```text
Archive Node Cost
≈

Historical State Retention
        ↓
Storage Growth
        ↓
More Write / Read I/O
        ↓
Longer / Heavier Sync
        ↓
Harder Backup & Recovery
        ↓
Higher Operational Cost
```

也就是说：

> Archive Node 贵，并不是因为某一个单独技术特别昂贵。

而是：

> **“不丢历史 State”这个产品需求，让整个基础设施成本曲线发生了变化。**

## 十一、为什么 RPC Provider 可以靠这个赚钱？

这里和 Module 5 开始接起来了。

假设一家 Web3 公司需要：

```text
eth_getBalance(address, old_block)
eth_getStorageAt(..., old_block)
eth_call(..., old_block)
```

它有两个方案。

方案 A：

```text
自己跑 Archive Node
```

那么公司承担：

```text
Server
SSD
Bandwidth
CPU
Memory
Sync
Monitoring
Upgrade
Recovery
Engineer Time
```

方案 B：

```text
Alchemy / QuickNode / 其他 Provider
```

公司只承担：

```text
API Bill
```

因此 Provider 实际卖的并不只是：

```text
HTTP Endpoint
```

而是在替客户承担：

```text
Blockchain Infrastructure Complexity
```

这也是为什么我们之前说：

> RPC Provider 卖的是基础设施能力，而不仅仅是一个 URL。

## 十二、那为什么大型数据公司又可能自己跑 Archive Node？

因为随着规模扩大，另一条成本曲线开始出现。

假设你每天只请求：

```text
1000 次 Historical State
```

购买 RPC 很合理。

但如果你是：

```text
Nansen
Dune
大型钱包
交易基础设施
链上风控平台
```

每天需要：

```text
几亿次内部数据访问
+
大规模 Backfill
+
自定义 Historical Query
```

那么：

```text
Provider Cost
```

可能越来越高。

同时还存在：

```text
Rate Limit
External Dependency
Latency
Data Availability
Vendor Risk
```

于是可能出现：

```text
请求规模较小
→ Buy

请求规模巨大
→ Build
```

但真实公司通常不是绝对二选一。

更常见的是：

```text
Own Node
+
External RPC Provider
```

例如：

```text
Primary
→ 自建

Fallback
→ Provider
```

或者：

```text
普通查询
→ 自建 Full Node

特殊 Historical State
→ Archive Provider
```

这叫：

```text
Build vs Buy
```

是典型基础设施决策。

## 十三、Blockchain Data Engineer 需要什么程度？

这也是课程边界。

你不需要现在学会：

```text
如何把 Geth Archive Node
调到最高 IOPS
```

但面试中如果有人问：

> 为什么 Archive Node 贵？

你不能只回答：

> 因为硬盘比较大。

更完整的回答应该是：

```text
Archive Node 为了保留 Historical State Queryability，
不能像普通 Pruned Node 一样大量清理旧 State。

这导致：

历史 State 数据持续增长
→ 更高 Storage

数据库规模和持续状态写入
→ 更高 Read / Write I/O

构建这些 Historical State
→ 更重的 Sync

数据集巨大
→ Backup / Recovery / Upgrade 更困难

最终
→ Hardware + Operations + Engineering Cost 都更高
```

这已经是比较标准的 Blockchain Data Infrastructure 回答。

## 十四、再进一步：数据工程师真正应该问什么？

当业务说：

> 我们需要 Archive Node。

你不应该马上回答：

```text
那就部署 Archive Node。
```

而应该先问：

> **到底哪个数据需求要求 Historical State？**

例如：

### 需求 A

```text
过去三年的 Uniswap Swap
```

主要来自：

```text
Historical Logs
```

未必需要 Archive Node。

### 需求 B

```text
每个 Block 的 Aave 用户健康因子
```

这就复杂得多。

如果没有自己维护协议状态，你可能需要：

```text
Historical State
+
Historical eth_call
```

这时 Archive 能力价值就明显提高。

所以：

```text
Archive Node
```

应该是一个：

```text
Requirement-driven infrastructure decision
```

而不是：

```text
区块链数据公司标配
```

## 十五、本课核心模型

最后把第 1～3 课串起来。

第 1 课：

```text
History
≠
Historical State
```

第 2 课：

```text
Pruning
→ 放弃旧 State

Archive
→ 保留 Historical State Queryability
```

第 3 课：

```text
Historical State Retention
        ↓
Storage Growth
        ↓
I/O
        ↓
Sync
        ↓
Operations
        ↓
Cost
```

所以：

```text
Archive Node 的高成本
不是因为“节点更会验证”

而是因为

它承担了
Historical State Retention
这个额外的数据基础设施责任
```

## 理解检查

### 问题 1

有人说：

> “Archive Node 贵，主要就是因为比 Full Node 多买几块大硬盘。”

这个说法为什么不完整？

请至少从 **Storage、I/O、Sync、Operations** 中解释三个方面。

### 用户回答

问题一，这个说法不完整。

Archive Node 不只是需要多买几块大硬盘，更重要的是它因为数据量巨大而带来的多方面难度的提升：

1. 数据量大，需要更大的 storage（这个是最直观的）
2. 数据量大，read 和 write 等 I/O 操作的 I/O 要求、I/O 能力也需要提升
3. 数据量大会对 Sync 也会带来影响
4. 数据量大，Operations（像 backup、recovery、upgrade 等操作）都会带来难度的提升

### 老师判断与校准

✅ 通过。回答完整覆盖了 Storage、I/O、Sync、Operations 四个维度。最重要的认识是：Archive Node 的成本不是“容量线性增加”这么简单，而是 Historical State Retention 让数据库规模持续增长，并进一步放大读写 I/O、同步、备份、恢复和升级等一整套基础设施成本。

### 问题 2

假设：

```text
Company A
每天只有 500 次历史 State 查询

Company B
每天需要进行数亿次历史 State 查询和大规模 Backfill
```

从 Build vs Buy 的角度看，哪家公司更可能直接购买 Archive RPC，哪家公司更有动力自建 Archive Infrastructure？为什么？

### 用户回答

Company A 更可能直接购买 Archive RPC，因为 Company A 的需求比较固定，成本消耗比较直观可控。

Company B 更有动力自建 Archive Infrastructure，因为 Company B 主要是查询历史 state 的次数多和大规模 Backfill，会导致查询支付的 RPC 费用非常高，很可能超过自建的成本

### 老师判断与校准

✅ 通过。Company A 的查询量很低，直接购买 Archive RPC 通常能把固定运维成本转换成更可控的 API 成本；Company B 的调用量和 Backfill 规模很大，Provider 费用、Rate Limit、外部依赖和吞吐限制会越来越明显，因此更有动力自建 Archive Infrastructure。生产环境里也常采用“自建 + Provider fallback”的混合模式，而不是绝对二选一。

### 问题 3

下面哪个需求更能证明“我们真的需要 Archive State”？

```text
A.
统计 2024 年 Uniswap Swap Volume

B.
在 Block 18,000,000
执行某 Lending Protocol 的
getUserAccountData(Alice)
```

为什么？

### 用户回答

需求B更能证明我们真的需要 archive state，

A 需求是统计 2024 年 Uniswap 的 swap volume。这个只要查询 2024 年这个时间段内的历史记录就可以了，它不涉及到 historical state。

但是需求 B 的需求是要获取 Alice 用户的一个 historical state，它需要查询 archive state。

### 老师判断与校准

✅ 通过。A 主要可以由 Historical Logs / Events 构建 Swap Fact 后统计，不因为查询的是 2024 年就自动需要 Archive State。B 则是在指定旧 Block 上执行 Lending Protocol 的读取逻辑，本质是 Historical `eth_call`，需要对应高度的 Contract Code、Storage 和账户状态，因此真正依赖 Historical State Queryability。

## 重点总结

1. Archive Node 的主要成本不是简单的“多买硬盘”，而是 Historical State Retention 引发的 Storage、Read / Write I/O、Sync、Backup / Recovery 与 Operations 的连锁放大。
2. Archive Node 的磁盘既要有容量，也通常需要较强随机读写能力；数据库越大，维护与恢复成本越重。
3. Build vs Buy 取决于查询规模、Backfill 强度、Provider 费用、Rate Limit、延迟、外部依赖与工程团队能力。
4. 历史 Block / Log 查询不等于 Historical State Query；只有真正依赖过去 State 的业务需求才构成 Archive 能力的直接理由。
5. 对 Blockchain Data Engineer 而言，Archive Node 是 Requirement-driven infrastructure decision，而不是数据公司默认必须部署的标配。