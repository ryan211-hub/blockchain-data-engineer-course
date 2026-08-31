# 第5课｜Node 活着为什么还不能接请求：Health Check、Sync Lag 与 Failover

## Lesson Contract
- 所属 Module：Module 5 — RPC
- 本课核心问题：一个 Ethereum Node 进程还在运行，为什么 RPC Provider 仍可能判定它“不健康”并停止向它路由请求？
- 学完后应能回答：Liveness、Readiness、Sync Lag 的区别；Provider 为什么需要 Health Check；Failover 如何与 Load Balancer 协作保证 RPC 可用性。
- 本课明确不展开：Archive Node 的历史状态架构（Module 6）；Indexer Pipeline 的重试与 Checkpoint（Module 7）；具体 Provider 的专有监控实现。

## 一、“服务器没死”不等于“这个 Node 能正常服务”
假设 Node A 的 Geth 进程还存在，CPU 也还能响应。从操作系统角度看，Node A 是 Alive。

但它可能已经落后主链：
```text
Ethereum Network Head = 25,000,000
Node A = 24,999,500
```
此时用户请求：
```text
eth_getBalance(Alice, latest)
```
Node A 所理解的 `latest` 实际上还是 24,999,500，因此用户可能拿到旧 State。

核心结论：
```text
Process Alive
≠
RPC Service Healthy
```

## 二、Liveness：它是不是还“活着”
【基础设施视角】Liveness 问的是：这个服务进程是不是还活着？

典型检查包括：
```text
Geth process exists?
RPC port responds?
Process crashed?
Server unreachable?
```
如果 Geth 已经 Crash，那么：
```text
Liveness = false
```
Liveness 解决的是服务是否已经彻底失效。

## 三、Readiness：它活着，但现在适不适合接请求
如果 Geth 进程仍在运行、RPC Port 也可以连接，但节点刚重启、正在恢复或尚未同步到最新 Block，则：
```text
Alive = Yes
Ready = No
```
可以这样记：
```text
Liveness：你还活着吗？
Readiness：你现在能工作吗？
```
这两个概念不是区块链专有的，在 Kubernetes、微服务等生产系统中也很常见。

银行系统也类似：Oracle Instance 已启动，并不意味着正在 Recovery 或 Standby Lag 很大的实例应该立即承接生产查询流量。

## 四、Blockchain Node 的特殊健康指标：Sync Status
普通 Web Server 判断健康通常看 HTTP、CPU、Memory；Blockchain Node 还必须看它跟主链同步到哪里。

例如：
```text
Ethereum Network Head = 25,000,000
Node A = 25,000,000
Node B = 24,999,999
Node C = 24,995,000
```
可以定义：
```text
Sync Lag = Network Head - Node Head
```
于是：
```text
Node A lag = 0
Node B lag = 1
Node C lag = 5,000
```
Provider 可以根据业务要求设定可接受的 Freshness / Sync Threshold，超出阈值后将节点降级或移出流量池。

## 五、为什么 Sync Lag 对 RPC 特别重要
很多 RPC 都依赖 Node 自己认为的 `latest`：
```text
eth_blockNumber
eth_getBalance(address, latest)
eth_call(..., latest)
eth_getTransactionCount(address, latest)
```
如果 Node 落后：
```text
Node Local Latest
≠
Ethereum Network Latest
```
那么 RPC 本身可能执行成功，但返回的数据却已经过时。

【RPC / 数据正确性视角】
```text
Transport Success
≠
RPC Execution Success
≠
Data Freshness / Correctness
```

## 六、Health Check 实际检查什么
一个 Provider 的 Health Check 可以粗略理解成：
```text
Node 能否连接？
        ↓
RPC 能否响应？
        ↓
Block Height 是否正常？
        ↓
Sync Lag 是否在允许范围？
        ↓
Latency 是否异常？
        ↓
Error Rate 是否异常？
```
例如：

| Node | Process | RPC | Block Lag | Latency | 判断 |
|---|---|---|---:|---:|---|
| A | OK | OK | 0 | 80ms | Healthy |
| B | OK | OK | 1 | 100ms | Healthy |
| C | OK | OK | 5000 | 90ms | Unhealthy |
| D | OK | OK | 0 | 8s | Unhealthy / Degraded |
| E | Down | Fail | — | — | Dead |

Node C 虽然进程、RPC、延迟都正常，但 Block Lag = 5000，仍然不适合作为正常 latest RPC Node。

## 七、为什么“延迟很高”也可能算不健康
Node D 已同步到最新高度，但 `eth_getBalance` 每次要 8 秒，而其他节点只要约 100ms。可能原因包括：
```text
SSD I/O 饱和
Compaction
CPU 满载
大量 eth_getLogs
Memory Pressure
```
虽然它最终还能返回 Response，但商业 Provider 仍可能把它判定为 Degraded，并暂时不给它新请求。

## 八、Health Check 与 Load Balancer 的关系
Health Check 回答：
> 哪些 Node 可以使用？

Load Balancer 回答：
> 在可以使用的 Node 里面，这个请求发给谁？

简化关系：
```text
Monitoring / Health Check
        ↓
Healthy Node Pool
        ↓
Load Balancer
        ↓
Node A / Node B / Node C
```

## 九、Failover：当前后端故障时怎么办
如果 Node A 突然出现 SSD 故障：
```text
Request
  ↓
Node A
  ↓
Timeout / Error
```
Provider 会通过 Health Check 将 Node A 标记为 Unhealthy，并从 Backend Pool 中移除；之后由其他 Healthy Node 承接新请求。

Failover 可以理解成：
```text
Current Backend
      ↓
发生故障
      ↓
切换到其他 Healthy Backend
```
用户看到的 Provider Endpoint 通常保持不变，不需要知道内部已经从 Node A 切换到 Node B。

## 十、Provider Endpoint 是逻辑服务地址，不是物理 Node 身份
第一课中，RPC 抽象屏蔽了 Node Storage 内部实现；到了这里，Provider 又增加一层抽象：
```text
Application
      ↓
Stable Provider Endpoint
      ↓
Provider Infrastructure
      ↓
Healthy Node Pool
      ↓
Blockchain Clients
```
Provider 帮上层应用屏蔽：谁挂了、谁正在升级、谁同步落后、当前请求具体由哪台 Node 处理。

## 十一、“落后 1 个 Block 就必须踢掉”吗
不一定。我们当前建立的是架构模型，不是某家 Provider 的具体规则。

是否 Healthy 取决于：
```text
业务要求
Provider 策略
请求类型
链的出块速度
一致性要求
```
正确理解是：Provider 会定义一个可接受的 Freshness / Sync Threshold。

## 十二、同一台 Node 对不同能力可能有不同 Readiness
例如某 Node 的 Current State 正常，但 Historical State 不完整，那么：
```text
eth_getBalance(address, latest)
```
可能可以执行，而：
```text
eth_getBalance(address, very_old_block)
```
可能无法执行。

因此 “Healthy” 有时不是简单的 True / False，也可能是面向某种服务能力，例如 Latest State Pool、Archive Pool、Trace Pool、Log-heavy Pool。Archive Node 的完整机制留到 Module 6。

## 十三、Blockchain Data Engineer 视角
Indexer 使用同一个 Provider Endpoint 时，底层处理请求的 Node 可能一直变化：
```text
Request A → Node A
Request B → Node B
Request C → Node C
```
因此不能假设：
```text
同一个 Endpoint = 永远同一台 Node
```
Provider 可能不断进行 Health Check、Failover、Load Balance、Node Upgrade、Node Replacement。

核心认识：
> Endpoint 是逻辑服务地址，不是物理 Node 身份。

## 十四、把第 4、5 课连接起来
第 4 课解决：请求很多怎么办？
```text
Rate Limit
Cache
Load Balancer
```
第 5 课解决：Node 不可靠怎么办？
```text
Health Check
Failover
```
完整简化模型：
```text
Application
    ↓
Provider Endpoint
    ↓
Authentication
    ↓
Rate Limit
    ↓
Cache
    ↓
Load Balancer
    ↓
Healthy Node Pool
    ├── Node A
    ├── Node B
    └── Node C

Health Check
    ↑    ↑    ↑
    └────┴────┘
```
如果某个 Node 异常：
```text
Health Check
↓
Remove Node
↓
Failover
↓
其他 Healthy Node 接管请求
```

## 十五、本课核心概念
```text
Liveness
= 这个 Node / Process 还活着吗？

Readiness
= 它现在适合接生产请求吗？

Failover
= 当前 Node 不适合服务时，把流量切到其他 Healthy Node。
```
对 Blockchain Node，Readiness 中一个特别重要的指标是 Sync Lag。

核心两句话：
```text
Alive ≠ Ready
RPC Success ≠ Data Freshness
```

## 理解检查
### 问题 1
Ethereum 当前最新高度是 25,000,000。Node A 的 Geth 正常运行、RPC 正常返回，但 Block Height = 24,995,000。Liveness 和 Readiness 分别怎么判断？为什么？

### 用户回答
“问题 1：Liveness 应该是 True，Readiness 应该是 False，因为 Geth正常运行，RPC 正常返回，所以 Liveness 是 true。但是 block higher height 和当前的最新高度相差了 5000 block，所以 Readiness is false”

### 老师判断
✅ 通过。Geth 进程仍在运行且 RPC 能响应，因此 `Liveness = True`；但节点比链头落后 5,000 个 Block，`latest` 数据明显过时，因此不适合作为正常生产 RPC 后端，`Readiness = False`。核心是：Alive 不等于 Ready。

### 问题 2
Node B 已同步到 latest，`eth_getBalance` 也能返回，但每次需要 8 秒，而其他 Node 只需 100ms。为什么 Provider 仍可能暂时把 Node B 从 Healthy Pool 中移除？

### 用户回答
“問題二：因为 Node B 的 latency 有 8s，相对于正常的延迟是 100 毫秒来说，现在的延迟太高了，所以 Node B 可能正在处理资源占用高的任务，导致它的延迟比较高。

所以如果现在把 Node B 列为正常 RPC Node 的话，它可能不能及时响应 request 请求。因此，可以暂时把 Node B 从 Load Balancer 的 Healthy Pool 中移除”

### 老师判断
✅ 通过。即使节点已同步到最新高度，只要服务质量明显退化，例如 Latency 从 100ms 上升到 8s，Provider 也可能将其标为 Degraded。原因可能包括 CPU、SSD I/O、Memory Pressure、后台 Compaction 或大量重 RPC。Health Check 不只判断“死没死”，也判断服务质量是否达到可接受标准。

### 问题 3
Health Check、Load Balancer、Failover 三者分别负责什么？

### 用户回答
“问题 3：谁判断 node 能不能用？Health check 判断 node 能不能用。

谁决定请求发给哪台？Load balancer 决定请求发给哪台。

当前 node 出问题以后，谁负责切换？Failover 负责切换。”

### 老师判断与校准
✅ 通过。
```text
Health Check
→ 判断哪些 Node 当前可用

Load Balancer
→ 在可用 Node 中决定请求发给谁

Failover
→ 当前后端失效时，把流量切到其他健康 Node
```
三者形成连续机制：Health Check 维护 Healthy Node Pool；Load Balancer 从其中选择后端；节点故障后触发 Failover，将流量转移给其他 Healthy Node。

## 重点总结
1. Process Alive 不等于 Service Ready。
2. Blockchain Node 的 Readiness 要特别关注 Sync Lag，因为 RPC 成功也可能返回 stale data。
3. Health Check 不只检查进程和端口，还会观察同步高度、Latency、Error Rate 等服务质量指标。
4. Load Balancer 只应从 Healthy Node Pool 中选择后端。
5. Failover 在当前后端失效时把流量切到其他健康节点。
6. 对 Blockchain Data Engineer 来说，同一个 Provider Endpoint 是逻辑服务地址，不能假设请求始终由同一台物理 Node 处理。