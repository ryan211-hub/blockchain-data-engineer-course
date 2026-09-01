# 第6课｜HTTP 还是 WebSocket：Polling、Subscription 与实时 RPC

## Lesson Contract
- 所属 Module：Module 5 — RPC
- 本课核心问题：为什么同样是 RPC，应用有时使用 HTTP Polling，有时使用 WebSocket Subscription？两者在交互模型和资源占用上有什么差异？
- 学完后应能回答：HTTP Request/Response 与 WebSocket 长连接的区别；Polling 与 Subscription 的区别；为什么实时场景常用 WebSocket，但历史批量查询仍常用 HTTP。
- 本课明确不展开：Indexer Pipeline、Checkpoint、消费位点与断线重放（Module 7）；Provider 的专有 WebSocket 实现细节。

## 一、HTTP RPC：问一次，答一次
前几课常见调用都是：
```text
Python
  ↓
HTTP Request
  ↓
RPC Provider
  ↓
Ethereum Node
  ↓
HTTP Response
```
例如 `eth_getBalance`、`eth_getBlockByNumber`、`eth_getLogs`。
【网络交互视角】HTTP Request/Response 的基本模型是：客户端问一次，服务器回答一次。如果之后还想知道最新状态，需要再发新的 Request。

## 二、Polling：客户端反复问有没有变化
如果钱包希望尽快知道新区块出现，而使用普通 HTTP，可以不断调用：
```text
eth_blockNumber
```
例如每 2 秒轮询一次：
```text
12:00:00 → 有新区块吗？
12:00:02 → 有新区块吗？
12:00:04 → 有新区块吗？
```
这就是 Polling。
如果短时间没有新区块，会产生大量重复请求。Polling Frequency 越高，实时性越好，但 RPC Request 数量和成本也越高，因此存在 Latency 与 Request Cost 的权衡。

## 三、WebSocket：长期保持连接
WebSocket 的连接模型不同：
```text
Client
   ↕
长期保持连接
   ↕
Server
```
连接建立后，双方可以持续发送消息，因此 Server 可以在未来事件发生时主动把数据推给 Client。

## 四、Subscription：有变化时告诉我
客户端可以通过 WebSocket 建立 Subscription，例如订阅新区块 `newHeads`：
```text
Client
  ↓
订阅 newHeads
  ↓
Provider / Node

—— 保持连接 ——

新区块出现
  ↓
Server 主动 Push
  ↓
Client
```
因此：
```text
Polling = Pull
Subscription = Push
```
Polling 是客户端不断问；Subscription 是客户端先订阅，之后由 Server 在事件发生时主动推送。

## 五、为什么不全部用 WebSocket
WebSocket 适合等待未来发生的事件，但历史数据查询通常目标明确、范围已知，例如：
```text
Block 10,000,000 → Block 11,000,000
```
这类 Historical Catch-up 更适合 HTTP / Batch RPC，因为客户端知道要查询哪些 Block 或 Logs。
判断标准可以简化为：
```text
Known Historical Data
→ HTTP / Pull

Future Events
→ WebSocket / Push
```

## 六、Ethereum 中典型 Subscription
Ethereum JSON-RPC 可以通过 `eth_subscribe` 订阅诸如：
```text
newHeads
logs
newPendingTransactions
```
例如 `eth_subscribe("newHeads")` 表示新区块 Header 出现时推送；订阅 `logs` 则可用于监听未来符合条件的新 Log。

## 七、Blockchain Data Engineer 为什么会同时使用两者
典型数据系统启动时可能落后链头很多 Block：
```text
Historical Catch-up
↓
HTTP / Batch RPC
```
追到链头后：
```text
Real-time Follow
↓
WebSocket Subscription
```
这已经接近 Indexer Pipeline，但断线重放、Checkpoint、补数据等机制留到 Module 7。

## 八、WebSocket 不是可靠消息队列
WebSocket 主要解决实时推送，不天然提供可靠持久化消息系统常见的能力，例如：
```text
持久化
ACK
Offset
Replay
Exactly-once delivery
```
如果连接断开 30 秒，期间产生 Block 101、102、103，重新连接后不能仅凭 WebSocket 假设 Provider 一定会把这些数据全部补回来。
因此：
```text
WebSocket
≠
Reliable Message Queue
```

## 九、HTTP、JSON-RPC 与 WebSocket 的关系
第二课已经建立：
```text
HTTP / WebSocket
= Transport

JSON-RPC
= RPC Message Protocol
```
因此 Ethereum JSON-RPC 可以通过 HTTP 承载，也可以通过 WebSocket 承载。`eth_getBalance` 仍然是 JSON-RPC Method；HTTP / WebSocket 主要解决消息怎么传。

## 十、场景选择
| 场景 | 更自然的方式 |
|---|---|
| 查询一个历史 Block | HTTP |
| 查询 Transaction Receipt | HTTP |
| 扫历史 Logs | HTTP |
| 查询 latest Balance | HTTP |
| 每隔几秒检查新区块 | HTTP Polling 可行 |
| 实时监听 New Block | WebSocket Subscription |
| 实时监听新 Logs | WebSocket Subscription |

## 十一、本课核心模型
```text
HTTP Request/Response
= 问一次，答一次

Polling
= 客户端反复问有没有变化

WebSocket
= 建立长期双向连接

Subscription
= 有变化时 Server 主动推送
```
以及：
```text
Historical Catch-up
→ HTTP / Batch RPC

Real-time Follow
→ WebSocket / Subscription

WebSocket
≠ Reliable Message Queue
```

## 理解检查
### 问题 1
一个钱包想尽快发现新区块。方案 A：每 1 秒调用一次 `eth_blockNumber`；方案 B：通过 WebSocket 订阅 `newHeads`。请分别说明 A、B 属于什么模式，以及为什么 A 会产生很多重复请求。

### 用户回答
“问题一：

A 方案属于 pull 模式，B 属于 push 模式。

1. 方案 A：每秒调用一次 block number，每秒都发送一次请求。如果短时间内没有新的区块，它会一直重复发送请求。
2. 方案 B：不需要客户端主动发送请求，而是服务端会将新区块的消息主动通知到客户端”

### 老师判断与校准
✅ 通过。A 是 Polling / Pull，B 是 Subscription / Push。A 会在没有新区块时仍然持续产生重复请求。补充术语校准：B 并不是客户端“完全不发送请求”，而是客户端先建立连接并发送一次订阅请求，之后的新事件主要由服务端主动推送。

### 问题 2
现在要获取 Block 10,000,000 到 Block 11,000,000 之间的历史数据。为什么通常优先使用 HTTP / Batch RPC，而不是 WebSocket Subscription？

### 用户回答
“问题二：这个情况下应该优先使用 HTTP 的方式，而不是 WebSocket Subscription，因为它查询的是历史 block 数据，而这些数据都是稳定不变的，这里涉及的不是未知的新数据，所以不适合用 WebSocket 的方式”

### 老师判断
✅ 通过。这里不仅因为历史数据已经稳定，更关键的是查询目标已经明确存在且范围已知，属于 Historical Catch-up / Batch Query；WebSocket Subscription 更适合未来出现的新 Block / 新 Log。

### 问题 3
为什么不能认为“用了 WebSocket，就不会丢数据，所以 Indexer 不需要其他补偿机制”？

### 用户回答
“问题三：

WebSocket 并不能保证传输过程中的数据不丢失。如果想要保证这一点，需要采用其他的方案（比如消息队列之类）。

而 WebSocket 解决的是：对尚未发生的新的数据，由服务器主动通知到客户端，而不用采用轮询的方式，由客户端一直轮询查询、发送请求”

### 老师判断与校准
✅ 通过。WebSocket 解决实时推送，但不天然提供持久化、ACK、Offset、Replay、Exactly-once 等可靠消息保障。对 Blockchain Indexer 来说，防止漏块还会涉及 Checkpoint、Block Height Comparison、Backfill、Retry、Reconciliation 等机制，这些放到 Module 7。

## 重点总结
1. HTTP Request/Response 是一次请求对应一次响应；Polling 是周期性 Pull。
2. WebSocket 提供长连接；Subscription 用于 Server 对未来事件主动 Push。
3. 已知历史范围通常适合 HTTP / Batch RPC，未来实时事件适合 WebSocket Subscription。
4. Ethereum JSON-RPC 是消息协议，HTTP / WebSocket 是 Transport。
5. WebSocket 提高实时性，但不等于可靠消息队列，也不能单独保证 Indexer 不漏数据。
6. Blockchain Data Engineer 常见组合是 Historical Catch-up 用 HTTP，Real-time Follow 用 WebSocket。