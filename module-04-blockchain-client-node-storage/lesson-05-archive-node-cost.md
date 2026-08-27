# 第5课｜Archive Node 为什么贵：历史状态查询、存储成本与数据公司的基础设施选择

## 本课目标

从基础设施与数据工程视角理解 Archive Node 的成本来源，以及它与 Warehouse / Snapshot 的分工。

## 一、Current State 与 Historical State 的成本差异

`Alice 现在有多少 ETH？` 属于 Current State 查询；`Alice 在 Block 10,000,000 时有多少 ETH？` 属于 Historical State 查询。后者要求节点具备历史状态访问能力。

## 二、Archive Node 为什么贵

Archive Node 不只是“Full Node + 大硬盘”。它需要承担更多 Historical State 数据带来的 Storage、随机 I/O、数据库维护、缓存、同步恢复和运维成本。

## 三、Archive Node 与 Analytics Warehouse 的访问模式不同

Archive Node 更适合按地址、区块读取链上历史状态；Dashboard 往往需要批量扫描、聚合、排序、趋势分析。后者更适合在 Warehouse 中使用结构化 Snapshot / Balance 表。

## 四、为什么不能让 Dashboard 每次直接查询 Archive Node

如果大量地址、多个日期都实时调用历史 `balanceOf`，会形成海量点查。更合理的方式是持续 Ingest / Index，维护 `balance` 与 `daily_snapshot`，让查询复用已加工的数据资产。

## 五、Archive Node 在数据公司的典型角色

- Historical State Ground Truth
- Backfill 数据源
- Pipeline 异常校验
- 数据修复依据

它通常不是最终 Dashboard 的直接分析数据库。

## 六、Backfill 与校验

Backfill 是把过去的数据一次性补入自己的数据平台；校验则是将 Warehouse 结果与历史链上 State 对比，发现漏采、Reorg、Decode 或 Pipeline 问题。

## 七、History 与 Archive State

完整 Blockchain History 理论上可以通过 Replay 重建任意历史 State，但计算成本高。Archive State 用更高的存储成本换取 Historical State 的直接可访问性。

## 八、核心心智模型

```text
Blockchain History → 过去发生了什么
Archive State      → 过去某一时刻世界是什么样
Warehouse/Snapshot → 让这些数据适合大规模业务查询和分析
```

## 九、理解检查结果

1. 能区分 Current State 与 Historical State 查询的基础设施成本。
2. 能解释为什么 Balance Dashboard 更适合由 Indexer + Warehouse + Snapshot 提供，而不是逐次调用 Archive Node。
3. 能解释 Archive Node 用于 Backfill / 校验与 Warehouse Serving Layer 并不冲突。
4. 能从“可重建性”和“计算成本”角度解释为什么有完整 History 仍需要 Archive State。

## 本课状态

✅ 已完成。
