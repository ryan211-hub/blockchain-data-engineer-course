# 第7课｜RPC Provider 到底卖什么：成本、计费与为什么公司愿意付费

## Lesson Contract
- 所属 Module：Module 5 — RPC
- 本课核心问题：既然 Ethereum JSON-RPC 是公开标准、Node 也可以自建，为什么 Alchemy、QuickNode、Helius 这样的 RPC Provider 仍然能收费并形成商业模式？
- 学完后应能回答：Provider 的主要成本来自哪里；为什么不同 RPC Method 可以有不同计费权重；企业购买 Provider 实际购买的是什么能力；什么时候自建 Node，什么时候购买 Provider 更合理。
- 本课明确不展开：Archive Node 的完整成本模型（Module 6）；Indexer Pipeline 与数据落库（Module 7）；具体厂商最新套餐价格与专有实现。

## 一、RPC 标准免费，为什么还要付钱
Ethereum JSON-RPC 规范不是 Provider 发明的。自己买服务器、安装 Geth、同步 Ethereum、开放 8545，就可以直接调用 `eth_getBalance`、`eth_getLogs` 等 Method。
真正的问题不是“能不能调用 RPC”，而是：谁来负责让这些 RPC 7×24 小时稳定、快速、可扩展地工作。

## 二、自建生产 Node 真正需要承担什么
生产环境除了 Server、SSD、Network，还要承担：
```text
Node Sync
Client Upgrade
Monitoring
Backup / Recovery
Health Check
Failover
Load Balancing
Rate Limit
Security
Capacity Planning
On-call
```
因此 Provider 运营的不是一个简单转发器，而是一套 Blockchain Data Access Infrastructure。

## 三、第一类成本：Compute
不同 RPC 消耗不同。`eth_getBalance(address, latest)` 通常更接近 Current State Lookup；`eth_call` 可能涉及 State、Contract Code、EVM Execution、Storage Read；大范围 `eth_getLogs` 可能消耗 CPU、Disk I/O、Filtering 和大量执行时间。
核心结论：
```text
1 RPC Request
≠
统一资源成本
```

## 四、第二类成本：Storage
Node 需要保存 Blocks、Transactions、Receipts、State、Indexes 等数据，承担 SSD 容量、性能、故障替换、I/O、Compaction 等真实成本。Module 4 中的 LSM Tree、Compaction、Write Amplification、SSD I/O，在这里体现为基础设施账单。

## 五、第三类成本：Network
Wallet、dApp、Bot、Indexer 等大量客户端会产生出站 Bandwidth / Egress Cost。完整 Block、大量 Logs、WebSocket 长连接都会消耗网络、连接数、Memory 与 Socket Resources。

## 六、第四类成本：Redundancy
商业 Provider 不能只运行一台 Node。为了高可用，通常需要多 Node、多 Region 和额外容量。平时看似闲置的资源其实是 Failover Capacity 与 Headroom，用于节点故障和流量尖峰。

## 七、第五类成本：Operations
Node Sync Lag、Disk Full、Client Upgrade、RPC Latency、Node Crash 等都需要 DevOps / SRE、Monitoring、Alerting、On-call、Incident Response。
因此公司真正比较的不是“一台服务器多少钱”，而是：
```text
TCO = Total Cost of Ownership
```

## 八、为什么不能只按 Request Count 收费
`1,000 × eth_getBalance(latest)` 与 `1,000 × 大范围 eth_getLogs` 请求数相同，但 Compute、Disk I/O 和执行时间可能差很多。因此 Provider 常用 Compute Units、Credits、Request Weight 等抽象单位。

## 九、Weight 的含义
教学示意：
```text
eth_blockNumber       weight = 1
eth_getBalance        weight = 2
eth_call              weight = 10
large eth_getLogs     weight = 100
```
这些数字只是示意，不代表任何厂商真实价格。Weight 的本质是把不同 RPC 的资源消耗映射成可计费单位。

## 十、Provider 的计费本质
```text
Usage
→ Resource Consumption
→ Price
```
Provider 把 CPU、I/O、Storage、Bandwidth、Connections、Historical Data 等复杂基础设施成本，抽象成客户容易理解的计费体系。

## 十一、企业为什么愿意付钱：Engineering / Operations
购买 Provider 可以把 Node Deployment、Sync、Upgrade、Monitoring、Failover、Capacity Planning、Security、Incident Response 等责任外包，让工程团队集中在产品本身。

## 十二、企业为什么愿意付钱：Time-to-Market
如果创业公司把工程师投入自建 RPC Infrastructure，就会减少 Wallet、Trading、Analytics 等核心产品开发资源。直接使用 Provider Endpoint 能更快上线和验证业务。
因此企业是在用钱换：
```text
工程时间
运维复杂度
上线速度
```

## 十三、Managed Database 类比
SQL 本身免费，但公司仍然购买 RDS、Cloud SQL、Aurora，因为买的是 Managed Database 的高可用、监控、备份、升级、恢复、扩容。
同理：JSON-RPC 免费，但 Managed RPC Infrastructure 可以收费。

## 十四、Provider 到底卖什么
不是卖 RPC Method，而是在卖：
```text
Availability
Performance
Scalability
Freshness
Operational Simplicity
Global Access
```
即稳定、性能、扩展能力、数据新鲜度和运维托管。

## 十五、为什么大公司仍可能自建 Node
随着调用规模增长，Provider Bill 可能高于自建成本；同时企业可能需要更强控制权、特殊 Node 配置、特殊 RPC、更低延迟、数据隐私、专用容量、降低 Vendor Lock-in。

## 十六、Buy vs Build
### Buy Provider
优势：启动快、运维少、高可用现成、容易扩容。
代价：持续费用、Provider 限制、Vendor Dependency。

### Self-host Node
优势：控制权高、可定制、大规模时单位成本可能更低。
代价：基础设施复杂、运维成本高、需要专业团队、自己承担故障。

## 十七、Hybrid Architecture
成熟公司常采用：
```text
            Application
                 ↓
        RPC Access Layer
          /           \
Own Nodes            Provider
```
例如核心高频数据走 Own Nodes，流量高峰或备用走 Provider，小众链或临时需求走 Provider，特殊配置或高控制需求走 Own Nodes。
核心是平衡 Cost、Control、Operational Complexity。

## 十八、Blockchain Data Engineer 视角
选 RPC 数据入口时，不应只问“哪家便宜”，而要先问：查询的是 Current State、Historical Logs、Historical State、Trace 还是 Realtime Subscription；QPS、数据量、延迟要求、历史范围、失败容忍度分别是多少。之后再决定 Provider、Self-host 或 Hybrid。

## 十九、Module 5 的完整链条
```text
第1课 RPC / Node / Provider
↓
第2课 JSON-RPC Request / Response
↓
第3课 不同 RPC 的成本模型
↓
第4课 Rate Limit / Cache / Load Balancer
↓
第5课 Health Check / Failover
↓
第6课 HTTP / WebSocket / Subscription
↓
第7课 成本 → 计费 → 商业模式
```
最终结论：
> RPC Provider 把复杂、昂贵、不稳定的 Node 基础设施包装成稳定、可扩展、按需付费的 Blockchain Data Access Service。

## 理解检查
### 问题 1
既然 Ethereum JSON-RPC 是公开标准，自己安装 Geth 也能开放 RPC，那么一家钱包公司为什么仍然可能愿意付钱购买 RPC Provider？请从 Engineering / Operations 与 Time-to-Market 两个角度解释。

### 用户回答
“问题一：
1. 从项目工程角度：
   付费购买 RPC Provider 可以降低项目难度，减少自己运维 Node 的精力花费。
2. 从 Time to Market 角度：
   购买使用 RPC Provider 会提升项目的进度，可以更快地完成项目目标”

### 老师判断与校准
✅ 通过。从 Engineering / Operations 角度，购买 Provider 是把 Node Deployment、Sync、Monitoring、Upgrade、Failover、Capacity Planning、Security、Incident Response 等基础设施责任外包；从 Time-to-Market 角度，直接使用 Provider Endpoint 可以缩短基础设施准备时间，把工程资源集中到核心产品。企业本质上是在用钱换工程时间、运维复杂度和上线速度。

### 问题 2
为什么 `eth_getBalance(address, latest)` 与大范围 `eth_getLogs` 即使都只算“一次 RPC Request”，Provider 也可能给后者更高的计费 Weight？

### 用户回答
“问题二：A、B 这两个请求虽然都是一次 RPC request，但是对节点 Node 来说，它们所做的工作是完全不同的。
1. 获取余额：会比较简单方便，直接从当前 state 就可以查询得到。
2. get logs：涉及到历史数据的查询（很有可能是历史大范围数据的查询），这需要消耗非常高的服务器资源。
所以对 RPC provider 来说，很可能给 B 更高的计费 weight”

### 老师判断
✅ 通过。`eth_getBalance(address, latest)` 通常接近 Current State Lookup；大范围 `eth_getLogs` 更接近 Historical Range Scan + Filtering + Disk I/O + CPU。Request Count 相同并不代表 Resource Cost 相同，Weight 是将真实基础设施消耗映射到计费单位。

### 问题 3
为什么大型 Blockchain Data Company 可能采用 Own Nodes + RPC Provider 的 Hybrid Architecture？

### 用户回答
“问题 3：采用哪种架构？
主要是从经济成本上去考虑，Hybrid Architecture 这个方式应该比较符合实际情况：
1. 如果采用 100% Provider，有可能某些权重高的查询花费会比自己运营节点来得高。在这种情况下，更经济的选择是自己运营一些 Node。
2. 如果全部 100% 自己 Self-host，对公司来说运营成本可能过高。在某些情况下，选择使用 RPC Provider 会更加快捷有效、方便便捷。
所以一般都会采用 Hybrid Architecture 这种混合模式”

### 老师判断与校准
✅ 通过。Hybrid Architecture 不只是经济成本折中，而是在 Cost、Control、Operational Complexity 三个变量之间做优化。核心高频工作负载可能放在 Own Nodes；流量峰值、备用、小众链或临时需求使用 Provider；特殊配置和高控制需求则更适合 Self-host。

## 重点总结
1. JSON-RPC 是公开标准，但稳定运行生产级 RPC 基础设施并不免费。
2. Provider 的成本主要来自 Compute、Storage、Network、Redundancy 和 Operations。
3. Request Count 不能准确表达资源成本，因此 Provider 会使用 Weight / Credit / Compute Unit 一类抽象。
4. 企业购买的是 Availability、Performance、Scalability、Freshness 与 Operational Simplicity，而不是 RPC Method 本身。
5. Buy vs Build 的关键是 TCO、Time-to-Market、Control 和规模。
6. 大型数据公司常采用 Hybrid Architecture，在 Own Nodes 与 Provider 之间按工作负载分工。
7. 至此已经达到 Module 5 的结束标准：能够解释 RPC 请求路径、RPC Interface 与 Provider 的区别、不同查询成本，以及为什么生产环境愿意购买 Provider 能力。