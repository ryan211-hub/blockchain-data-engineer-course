# 第8课｜Logs Bloom、索引与缓存：为什么 eth_getLogs 仍然可能很慢？

## 本课目标

理解 Ethereum 节点如何减少大范围 Log 查询成本，区分 Bloom Filter、索引与缓存的作用，并理解为什么 `eth_getLogs` 即使有优化机制，仍然可能是昂贵查询。

## 一、最朴素的 Log 查询为什么昂贵

如果没有任何辅助结构，查询 USDC `Transfer` Event 需要在目标 Block Range 内读取大量 Receipts / Logs 并逐条过滤。其本质是大范围历史扫描。

## 二、Bloom Filter 的核心能力

Bloom Filter 是概率型数据结构，适合回答：

- 一定不存在
- 可能存在

它不能证明“一定存在”。因此节点可以安全跳过 Bloom 判定为不可能匹配的 Block；对“可能匹配”的 Block 仍需读取真实 Log 做精确验证。

## 三、Ethereum Logs Bloom

Block Header 中包含 `logsBloom`。Log Bloom 的可检索元素主要与 Contract Address、Topics 有关，因此 `address = USDC`、`topic0 = Transfer` 这类过滤条件可以利用 Bloom 先排除大量无关 Block。

`logsBloom` 不是完整 Log，也不是精确索引，而是帮助快速排除不可能匹配区块的紧凑摘要。

## 四、Bloom Filter、Index 与 Cache 的区别

- Bloom Filter：快速排除“不可能匹配”的数据块。
- Index：更快定位目标数据。
- Cache：避免重复访问底层数据。

三者都用于性能优化，但解决的问题不同。

## 五、为什么有 Bloom 仍然可能很慢

Bloom 只是减少无意义扫描，并没有把 Range Query 变成 Key Lookup。

查询成本通常随以下因素增加：

- Block Range 变大
- 候选 Block 变多
- Filter Selectivity 变低
- Receipt / Log IO 增加

因此 `fromBlock = 1, toBlock = latest` 仍然可能非常昂贵。

## 六、Filter Selectivity

Selectivity 表示查询条件筛选数据的能力。

例如：

- `USDC + Transfer + Alice`：Selectivity 较高。
- 所有 Contract 的所有 Logs：Selectivity 很低。

Block Range 和 Filter Selectivity 共同决定 `eth_getLogs` 的基础设施压力。

## 七、为什么 RPC Provider 会限制大范围 eth_getLogs

超大范围 History Scan 会带来 Disk IO、CPU、Cache Pollution、Timeout 等资源压力。因此常见 RPC Provider 会限制最大 Block Range、返回条数、请求速率等。

## 八、Checkpoint 的基础设施意义

生产级 Indexer 不应每次从 Genesis 重新扫描，而应：

```text
Backfill
   ↓
保存 checkpoint
   ↓
Incremental Processing
```

这不仅避免数据工程上的重复计算，也减少 Node / RPC 的重复历史扫描压力。

## 九、Cache 的作用

对于 finalized Historical Block 等稳定数据，Cache 可以避免重复访问 Node DB。对于 `latest` State 等高频变化数据，缓存策略更复杂。

## 十、与传统数据工程的类比

- Bloom Filter ≈ 快速排除不需要扫描的数据块 / 类似 Partition Pruning 的思想
- Index ≈ 定位目标数据
- Cache ≈ Redis 等重复查询缓存
- Checkpoint ≈ 增量 ETL / CDC 的消费位置

## 十一、理解检查结果

### 问题1

为什么 Bloom Filter 说“不包含 USDC address”可以跳过，而“可能包含”仍需继续验证？

**回答正确。** Bloom Filter 可以确定数据不存在，但不能确定数据一定存在，因此对可能存在的候选仍需进一步检查。

### 问题2

A 与 B 两个 `eth_getLogs` 查询哪个压力更大？

**回答正确：B。**

- B 从 Genesis 扫到 Latest，Block Range 远大于 A。
- B 没有 address/topic 等高选择性过滤条件，Filter Selectivity 更低。
- 因此 B 对 Node 的 History Scan 压力显著更大。

### 问题3

为什么生产级 Indexer 应保存 checkpoint？

**回答正确。**

- 数据工程视角：已有数据 + 增量数据即可维持结果更新，避免重复处理历史数据。
- Node Infrastructure 视角：避免每次从 Genesis 调用 `eth_getLogs`，显著降低节点和 RPC 压力。

## 本课状态

✅ 已完成：理解检查通过。
