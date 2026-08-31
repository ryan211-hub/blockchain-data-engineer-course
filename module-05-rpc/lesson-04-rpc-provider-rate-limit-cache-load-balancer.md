# 第4课｜RPC Provider 为什么需要限流、缓存与多节点：从单节点到生产级服务

## Lesson Contract
- 所属 Module：Module 5 — RPC
- 本课核心问题：为什么商业 RPC Provider 不能只把所有请求直接转给一台裸 Node？限流、缓存和多节点分别解决什么问题？
- 学完后应能回答：单节点 RPC 的主要瓶颈；Rate Limit、Cache、Load Balancing / Multi-node 的职责区别；为什么这些能力构成 Provider 的核心商业价值。
- 本课明确不展开：Archive Node 的完整历史状态架构（Module 6）；Indexer Pipeline 与 Checkpoint（Module 7）；具体 Provider 专有实现和定价细节。

## 一、从单节点开始：为什么商业 Provider 不能只跑一个 Geth
假设自己运行一个 Node：
```text
Python
  ↓
http://my-node:8545
  ↓
Geth RPC
  ↓
State / History / EVM
```
如果只有少量自用请求，这完全可行。但商业 Provider 要面对大量用户和不同成本的 RPC Workload，而 Node 的 CPU、Memory、SSD I/O、Network Bandwidth 都是有限资源。

例如同时到来：
```text
User A → eth_getBalance
User B → eth_getBalance
User C → eth_getLogs（大范围）
User D → eth_call（复杂合约）
User E → eth_getLogs
```
上一课已经知道，不同 RPC 的成本差异很大。如果什么都不治理，一个重请求用户就可能影响整个共享 Node。

## 二、Rate Limit：控制入口流量，保护共享资源
某个用户如果程序写错，持续高频调用 `eth_getLogs`，可能快速吃满 Node 的 CPU、Disk I/O 或并发处理能力。

```text
用户
 ↓
大量重请求
 ↓
Node
 ↓
CPU / Disk I/O 饱和
 ↓
其他用户延迟或超时
```

因此 Provider 需要 Rate Limit：限制单个用户在一定时间内可以消耗多少服务能力。

【Provider 运维视角】Rate Limit 首先是资源保护与多租户隔离，不只是商业计费手段。

## 三、Cache：相同问题不要反复问 Node
很多 RPC 请求高度重复。例如大量钱包几乎同时查询同一个历史 Block：
```text
eth_getBlockByNumber(10,000,000)
```
已经稳定的历史 Block 可以近似看作 immutable，因此很适合缓存。

第一次：
```text
Cache MISS
  ↓
Node
  ↓
Block Data
  ↓
写入 Cache
```
后续：
```text
Cache HIT
  ↓
直接返回
```
这样大量用户请求不一定转化成同数量的 Node 查询。

但 `eth_getBalance(Alice, latest)` 不同，因为 `latest` 对应 Current State，会持续变化。如果缓存过久，就可能出现缓存余额与真实余额不一致。

核心权衡：
```text
Freshness
vs
Performance
```

## 四、Multi-node：一台 Node 会坏，也会过载
商业服务不能把可用性建立在单台 Node 上。Node 可能因为 SSD 故障、Client Crash、同步落后、网络异常、升级重启而不可用。

因此需要多个后端 Node：
```text
                ┌→ Node A
User → Provider ├→ Node B
                └→ Node C
```
Multi-node 提供容量扩展和故障冗余。

## 五、Load Balancer：请求应该发给哪一台 Node
有多台 Node 后，需要决定每个 Request 路由给谁。

```text
Users
  ↓
Load Balancer
  ↓
 ├── Node A
 ├── Node B
 └── Node C
```

最简单可以 Round Robin；生产环境还可能根据当前连接数、响应时间、健康状态、同步高度、请求类型等信号路由。

【重要视角】
```text
Ethereum 协议视角：
A/B/C 都只是 Ethereum Nodes

Provider 基础设施视角：
A/B/C 是自己管理的 Server / Node Pool
```

所以 Provider 能知道 Node 的 CPU、延迟、同步状态等，因为这些 Node 是其自己运营或管理的基础设施。

## 六、Rate Limit、Cache、Load Balancer 的职责边界
```text
Rate Limit
→ 不让请求进得太猛

Cache
→ 相同问题不要反复问 Node

Load Balancer
→ 确实要访问 Node 时，把请求合理分散到健康后端
```

一个简化架构：
```text
                  Internet Users
                       ↓
                  Rate Limit
                       ↓
                     Cache
                 ↙ HIT   MISS ↘
             Direct Return       Load Balancer
                                      ↓
                          ┌───────────┼───────────┐
                        Node A      Node B      Node C
```

## 七、为什么 Provider 不是简单的 RPC 转发器
真正的 Provider 更接近：
```text
                    Provider
                       │
        ┌──────────────┼───────────────┐
        │              │               │
 Authentication    Rate Limit        Cache
        │              │               │
        └──────────────┼───────────────┘
                       ↓
                 Load Balancer
                       ↓
        ┌──────────────┼──────────────┐
        │              │              │
      Node A         Node B         Node C
```
后续还可能包含 Monitoring、Failover、Multi-region、Archive Node、Tracing Node、Logs Infrastructure 等。

Provider 真正出售的是稳定、可扩展的 Blockchain Data Access Infrastructure，而不是 JSON-RPC 标准本身。

## 八、Blockchain Data Engineer 视角
Indexer 使用 RPC Provider 时，Endpoint 不是无限资源。实际会面对：
```text
Rate Limit
Request Weight
Timeout
Max Block Range
Concurrency Limit
Provider Failure
```
这些约束会影响后续 Indexer 的请求速率、重试、并发和失败处理设计，但具体 Pipeline 放到 Module 7。

## 九、本课核心模型
```text
Application
     ↓
JSON-RPC Request
     ↓
RPC Provider
     ↓
Rate Limit
     ↓
Cache
     ↓
Load Balancer
     ↓
Node Pool
     ↓
Blockchain Client
     ↓
State / History / EVM
```

Provider 的核心价值可以概括为：
> 把“一台 Node 的能力”变成“可供大量用户稳定共享的 RPC Service”。

## 理解检查
### 问题 1
假设一个 Provider 只有一台 Ethereum Node，并且完全没有 Rate Limit。某个用户突然连续发送大量 `eth_getLogs`，为什么这可能影响其他正常用户？

### 用户回答
“因为 get logs 是个 range scan（范围查询），会涉及到查询大量历史的 block，这是非常消耗资源的操作。所以，这会将这台 node 的资源耗尽，从而影响到其他正常用户。”

### 老师判断
✅ 通过。`eth_getLogs` 往往是 Range Scan，需要跨较大区块范围检查历史日志；如果没有限流，一个用户持续发这类重请求，可能把 Node 的 CPU、Disk I/O、内存或并发处理能力压满，其他用户的请求延迟会上升，甚至超时。Rate Limit 的本质是保护共享后端资源，避免单个租户拖垮整体服务。

### 问题 2
100,000 个用户几乎同时查询 `eth_getBlockByNumber(10,000,000)`，为什么 Cache 特别适合这个场景？而 `eth_getBalance(Alice, latest)` 为什么缓存需要更谨慎？

### 用户回答
“10 万个用户几乎同时查询 getBlockByNumber，为什么 cache 特别适合这个场景？因为 getBlockByNumber 查询的是历史的 block 数据，历史的 block 数据是 immutable（不可改变的）。如果将历史的 block 数据放到 cache 里面，那么这 10 万个用户查询时命中缓存，效率就会特别快。

getBalanceLatest 为什么缓存需要更谨慎？因为获取最新的余额查询的是当前的 state，而当前的 state 是非常容易发生变化的。如果把当前的状态数据直接放到 cache 里面，很有可能会导致命中缓存时和实际真实的余额数据不一致：真实的余额已经发生变化，而缓存中的余额还是当时缓存时的状况。所以缓存当前状态时需要更谨慎。”

### 老师判断
✅ 通过。稳定历史 Block 很适合缓存，因为重复请求可以直接命中 Cache，避免重复访问 Node。`latest` State 会持续变化，所以需要在 Freshness 与 Performance 之间权衡缓存策略。

### 问题 3
分别用一句话解释 Rate Limit、Cache、Load Balancer 三者解决什么问题。

### 用户回答
“1. Rate Limit：解决了防止一个用户占用过多资源，从而影响其他正常用户的问题。
2. Cache：解决的是相同、同样的查询不用每次都去访问 Node，从而提高查询效率。
3. Load Balancer：可以解决、防止单一节点接收了过多的 Request，而其他的空闲节点却资源闲置，造成 Request 请求分配不均衡的情况。”

### 老师判断与校准
✅ 通过。可以压缩成生产环境表达：
```text
Rate Limit
→ 控制入口流量，防止单个用户过度消耗共享资源

Cache
→ 减少重复计算 / 重复访问 Node

Load Balancer
→ 把请求合理分散到多个健康后端 Node
```
三者分别处在不同阶段：请求进来之前由 Rate Limit 控制；请求进入后先判断 Cache 能否直接复用；确实需要访问 Node 时，再由 Load Balancer 选择后端。

## 重点总结
1. 单 Node 的计算、存储和网络资源有限，商业 Provider 必须做资源治理。
2. Rate Limit 解决多租户资源保护；Cache 解决重复查询；Load Balancer 解决多节点流量分配。
3. 历史不可变数据通常更适合缓存，`latest` State 的缓存要重点处理 Freshness。
4. Multi-node 提供容量与冗余，Load Balancer 把请求分配给合适的健康节点。
5. Provider 的价值不是“转发 JSON-RPC”，而是把 Node 能力包装为稳定、可扩展的基础设施服务。
6. 对 Blockchain Data Engineer 来说，Provider 的限流、查询权重、超时和区块范围限制都会成为后续 Indexer 设计约束。