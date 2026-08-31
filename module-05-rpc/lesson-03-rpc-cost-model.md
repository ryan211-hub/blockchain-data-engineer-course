# 第3课｜为什么不同 RPC 成本差这么多：State、History、Logs 与 EVM Execution

## Lesson Contract
- 所属 Module：Module 5 — RPC
- 本课核心问题：为什么 `eth_getBalance`、`eth_getBlockByNumber`、`eth_getLogs`、`eth_call` 看起来都是 RPC Method，但节点端成本差异很大？
- 学完后应能回答：不同 RPC Method 实际访问的是 State、History、Logs 还是 EVM Execution；为什么查询范围、磁盘访问和执行复杂度决定成本；为什么 Provider 会对某些 Method 设置更严格限制或更高计费权重。
- 本课明确不展开：Archive Node 的完整历史状态机制（Module 6）；Indexer 持续抓取与 Checkpoint（Module 7）；Provider 的具体商业定价模型。

## 一、核心分类
【Blockchain Client 视角】常见 RPC 可以粗略分为四类：

```text
State Query
History Query
Log / Range Query
Execution Query
```

对应：

```text
eth_getBalance        → State
eth_getBlockByNumber  → History
eth_getLogs           → Logs / Range Scan
eth_call              → EVM Execution
```

关键结论：一次 RPC 的成本不是固定值，而取决于它实际访问什么数据、扫描多大范围、是否执行 EVM、是否依赖历史状态。

## 二、State Query
`eth_getBalance(address, latest)` 主要查询最新 Current State，可近似理解为 Point Lookup / Key Lookup，通常较轻。

但 `eth_getBalance(address, old_block)` 是 Historical State Query，节点必须具备对应旧状态数据，因此成本可能明显更高，普通节点甚至可能无法直接提供。

## 三、History Query
`eth_getBlockByNumber`、`eth_getTransactionByHash`、`eth_getTransactionReceipt` 主要读取 Block / Transaction / Receipt 等 History。若节点已有合适索引，例如 `tx_hash → transaction`，常可近似理解为 Indexed Lookup。

## 四、为什么 eth_getLogs 常常很重
例如：

```text
eth_getLogs
fromBlock = 10,000,000
toBlock   = 20,000,000
```

这是一个大范围 Range Scan / Filter。查询范围越大，需要检查的区块、Bloom、Receipt / Log 候选越多，磁盘 I/O、CPU 与过滤成本通常越高。

Logs Bloom 可以帮助快速排除“一定不包含目标 Log”的区块，但 Bloom 只能降低成本，不能把大范围查询变成零成本。

## 五、eth_call 的本质
【EVM 执行视角】`eth_call` 不是简单读取一个值，而是在某个 State 上模拟执行一次 EVM Call。典型过程：

```text
加载 State
↓
加载 Contract Code
↓
EVM Execute
↓
读取 Contract Storage / 调用其他 Contract
↓
返回 Result
```

用户不支付链上 Gas，不代表节点执行没有成本。Provider 仍然承担 CPU、Memory、Disk I/O 与 EVM Execution Time。

## 六、Provider 为什么会区别计费
如果把所有 RPC 都简单计成“1 request”，就无法反映 `eth_getBalance(latest)` 与超大范围 `eth_getLogs` 的资源差异。因此 Provider 往往会采用 Compute Unit、Credit、Request Weight 等类似 Query Cost 的概念，而不仅按 Request Count 计费。

## 七、Blockchain Data Engineer 的判断模型
看到 RPC Method 时，不要只问“怎么调用”，而要先问“这个 Method 后面访问什么”。

| RPC | 主要工作 |
|---|---|
| `eth_getBalance` | State Lookup |
| `eth_getTransactionByHash` | History / Indexed Lookup |
| `eth_getBlockByNumber` | Block History |
| `eth_getLogs` | Log Range Scan / Filter |
| `eth_call` | State + EVM Execution |

可记为：

```text
RPC Cost
≈ Data Access Cost
+ Scan Range
+ Execution Cost
+ Historical Data Requirement
```

## 理解检查与用户回答
### 问题 1
为什么 `eth_getLogs(fromBlock=10,000,000, toBlock=20,000,000)` 通常比 `eth_getBalance(Alice, latest)` 昂贵很多？

### 用户回答
“A 请求只获取 Alice 账户的最新状态，所以这个 RPC 只要从节点获取最新的 state 就可以了。B 请求 get logs，他的请求是从 block 1000 万到 block 2000 万的这个区间范围。所以他查询的 block 数据量非常大，范围很广，相应的会昂贵很多。”

### 老师判断
✅ 通过。核心区分正确：A 更接近 Point Lookup；B 是大范围 Range Scan / Filter。

### 问题 2
为什么 `eth_call` 即使用户不支付 Gas，也仍然会给 RPC Provider 带来实际成本？

### 用户回答
“Eth_call 即使用户不支付 gas，也仍然会给 RPC provider 带来实际成本。因为 call 会涉及到 EVM execution，需要执行智能合约之后才能处理这个请求，因此它确实会带来实际的成本，而这个成本是由 RPC provider 来承担的。”

### 老师判断
✅ 通过。不支付链上 Gas ≠ 节点执行无成本。

### 问题 3
比较：
1. `eth_getBalance(address, latest)`
2. `eth_getBalance(address, block=12,000,000)`
3. `eth_getLogs(fromBlock=10,000,000, toBlock=20,000,000)`

### 用户回答
“1. 第一个请求最容易处理：它是 get 指定 address 的 balance（最新 balance），所以它只要从最新状态获取数据就可以了。
2. 第二个请求需要查询这个地址在 block 一亿两千万高度时候的余额：这涉及到历史数据的查询，消耗的资源会比第一个大，也就是说第二个查询比第一个查询更昂贵。
3. 第三个 get logs：它属于block 的范围查询，这个查询应该是最昂贵的。”

### 老师校准
✅ 核心理解通过。数字口误：题目是 Block 12,000,000（一千二百万），不是“一亿两千万”。另外不应把真实成本记成绝对固定排行榜。更准确的是：1 通常最容易；2 与 3 都可能昂贵；3 在大范围查询条件下通常很重，但具体成本还受 Provider 架构、索引、缓存和历史状态保存方式影响。

## 重点总结
- RPC Cost ≠ Request Count。
- `latest` State Point Lookup 通常较轻。
- Historical State 查询可能需要更强的节点能力。
- 大范围 `eth_getLogs` 常受 Range Scan / Filter 成本影响。
- `eth_call` 会真实执行 EVM，因此即使不上链也消耗节点资源。
- Provider 的计费与限流需要反映真实 Query Cost。

## 当前状态
✅ 已完成。讲解、理解检查和必要校准均已完成。下一步进入 Module 5 第 4 课。