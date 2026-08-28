# 第9课｜LevelDB / RocksDB 为什么适合节点：LSM Tree、Compaction 与写放大

## 本课目标

理解 Ethereum Client 底层 Key-Value Storage Engine 的核心写入模型，建立 LSM Tree、MemTable、SSTable、Compaction、Write Amplification 与节点 SSD I/O 之间的关系。

## 一、为什么节点适合 LSM Tree

Ethereum 节点持续执行新区块并产生大量碎片化 State 写入，属于典型 write-heavy workload。LSM Tree 不倾向每次都随机修改磁盘中的旧数据，而是优先把写入放入内存中的 MemTable，并配合 WAL 保证可恢复性；达到阈值后再批量 Flush 成不可变 SSTable。这样可以把大量小型 Random Write 转换为更适合磁盘的顺序 / 批量写入。

## 二、MemTable、WAL 与 SSTable

```text
Write
 ├→ WAL
 └→ MemTable
       ↓
      Flush
       ↓
    SSTable
```

MemTable 负责高效吸收新写入，WAL 用于崩溃恢复，SSTable 是磁盘上的不可变有序数据文件。

## 三、Compaction

同一个 Key 可能在不同 SSTable 中出现多个版本。Storage Engine 会通过 Compaction 对多个 SSTable 进行 Merge / Sort，并清理不再需要的旧版本，以控制查询成本和文件数量。

## 四、Write Amplification

一次应用层写入可能经历 WAL、Flush 和一级或多级 Compaction，因此同一份数据可能被多次写入 SSD。

```text
Application Write = 100 GB
Physical SSD Write = 600 GB
```

这种现象称为 Write Amplification。

## 五、其他 Amplification

- Read Amplification：一次逻辑查询可能需要检查多个 MemTable / SSTable / Level。
- Space Amplification：旧版本、Compaction 中间文件、Metadata 等会导致实际磁盘占用高于当前有效数据量。

Bloom Filter、Index、Cache 等机制可以降低部分 Read Amplification。

## 六、节点 SSD 为什么重要

节点磁盘压力不仅来自“区块链数据量大”，还来自 State 持续写入、WAL / Flush、SSTable Compaction、Random Read 和 Write Amplification。因此除了容量，还需要关注 IOPS、Latency 与 SSD Write Endurance。

## 七、与 Archive Node 的关系

Archive Node 保存更多历史状态能力，会进一步增加 Storage Capacity、I/O、Compaction、Write Amplification、Recovery 和 Operational Cost。因此 Archive Node 成本不能只按磁盘 TB 数理解。

## 八、Oracle 类比

Oracle 的 Buffer Cache + Redo Log + DB Writer 与 LSM Tree 的 WAL + MemTable + Flush 在“内存缓冲、日志保证可恢复、延迟落盘”这一工程思想上相似，但具体数据结构不同。LSM Tree 更强调顺序写入与后台 Merge。

## 九、理解检查

1. Random Write 容易形成磁盘瓶颈，因此 LSM Tree 倾向使用 Memory + Sequential / Batched Write。回答正确。
2. 逻辑写入 100 GB、SSD 实际写入 600 GB 可能属于 Write Amplification；Compaction / Merge 会造成重复写盘。回答正确。
3. RPC QPS 没明显上涨但 SSD utilization 接近 100% 时，不能直接归因于 RPC；Compaction backlog、Write Amplification、Cache miss、State DB growth 或其他磁盘瓶颈也可能导致该现象。回答正确。

## 本课状态

✅ 已完成。
