# Module 5 — RPC

## 模块目标

从 Blockchain Data Engineer 视角理解 RPC 在 dApp / Wallet / Indexer 与 Blockchain Client 之间的位置，区分 Node、RPC Interface 与 RPC Provider，并逐步理解 Provider 的基础设施能力、限制、性能成本与商业模式。目标不是只会调用 API，而是知道一次 RPC 请求后面发生了什么，以及这些能力为什么值得付费。

## Module Contract

### 必须掌握

- Node、RPC、RPC Provider 三个层级的区别
- JSON-RPC 的基本调用模型
- 常见 Ethereum RPC 方法访问的对象与基本成本差异
- RPC Provider 为什么需要节点集群、限流、缓存和可用性治理
- 为什么 eth_getLogs、历史状态等请求通常更昂贵
- Blockchain Data Engineer 如何选择和使用 RPC 数据入口

### 可以了解

- HTTP / WebSocket 两种接入方式
- Provider 的多区域、负载均衡、缓存等基础设施思路
- Alchemy、QuickNode、Helius 等服务的产品差异

### 本 Module 不展开

- Archive Node 的完整历史状态基础设施与成本模型 → Module 6
- Indexer Pipeline、Checkpoint、数据落库 → Module 7
- Client 内部 Storage Engine 实现 → Module 4 已结束

## 当前学习进度

- 当前阶段：第二阶段 Infrastructure
- 当前 Module：Module 5 — RPC
- 已完成 Lesson：第 1 课、第 2 课、第 3 课、第 4 课、第 5 课、第 6 课
- 当前 Lesson：第 7 课｜RPC Provider 到底卖什么：成本、计费与为什么公司愿意付费
- 当前状态：🟡 学习中
- 最近完成：第 6 课理解检查与校准已完成；HTTP / WebSocket、Polling / Subscription 的职责区别已掌握。
- 尚未完成：第 7 课正式讲解与理解检查
- 下一步准确入口：开始第 7 课，理解 RPC Provider 的成本来源、计费权重、商业价值，以及企业为什么愿意购买 Provider 而不是全部自建 Node。

> 本区块是 Notion「课程总目录」主进度记录的同步镜像；若发生冲突，以课程总目录为准。

## 课程索引

1. [第1课｜RPC 到底是什么：从 dApp 请求到 Node Client 的接口层](lesson-01-rpc-basics.md)
2. [第2课｜一次 JSON-RPC 请求到底发生了什么：Method、Params、Response 与 Error](lesson-02-json-rpc-request-response.md)
3. [第3课｜为什么不同 RPC 成本差这么多：State、History、Logs 与 EVM Execution](lesson-03-rpc-cost-model.md)
4. [第4课｜RPC Provider 为什么需要限流、缓存与多节点：从单节点到生产级服务](lesson-04-rpc-provider-rate-limit-cache-load-balancer.md)
5. [第5课｜Node 活着为什么还不能接请求：Health Check、Sync Lag 与 Failover](lesson-05-health-check-sync-lag-failover.md)
6. [第6课｜HTTP 还是 WebSocket：Polling、Subscription 与实时 RPC](lesson-06-http-websocket-polling-subscription.md)
7. 第7课｜RPC Provider 到底卖什么：成本、计费与为什么公司愿意付费（当前学习中）