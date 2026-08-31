# 第2课｜一次 JSON-RPC 请求到底发生了什么：Method、Params、Response 与 Error

## Lesson Contract
- 所属 Module：Module 5 — RPC
- 本课核心问题：一条 Ethereum JSON-RPC 请求从客户端发出以后，协议层到底包含哪些信息，节点又如何把请求和响应对应起来？
- 学完后应能回答：`method`、`params`、`id`、`result`、`error` 分别解决什么问题；HTTP 与 JSON-RPC 是什么关系；为什么 RPC Error 不等于 HTTP Error。
- 本课明确不展开：Provider 的负载均衡、缓存、限流、多节点路由；Archive Node；Indexer Pipeline。

## 一、先建立一个最重要的分层
上一课我们已经建立：
```text
Application
  ↓
RPC Provider / RPC Endpoint
  ↓
RPC Interface
  ↓
Blockchain Client
```

这一课再往前一步：真正通过网络发送的，并不是“一个 Python 函数”，而是一段符合 JSON-RPC 规范的消息。

【协议视角】可以先把它理解成两层：
```text
HTTP / WebSocket
      ↓ 负责把消息送过去
JSON-RPC
      ↓ 负责说明“要调用什么、参数是什么、结果是什么”
Ethereum RPC Method
```

因此 HTTP 和 JSON-RPC 不是同一个东西。HTTP 更像运输通道，JSON-RPC 更像运输的业务消息格式。

## 二、一条最典型的 JSON-RPC Request
例如查询最新 ETH Balance：
```json
{
  "jsonrpc": "2.0",
  "method": "eth_getBalance",
  "params": [
    "0x1234...",
    "latest"
  ],
  "id": 1
}
```

这个 Request 里最重要的是四个部分。

### 1. `jsonrpc`
```json
"jsonrpc": "2.0"
```
表示这条消息遵循 JSON-RPC 2.0 协议。

### 2. `method`
```json
"method": "eth_getBalance"
```
告诉远端：我要调用哪个 RPC Method。

可以把它近似类比成调用：
```text
getBalance(...)
```

只是这里函数不是在本地执行，而是在远端 Ethereum Client 的 RPC 层处理。

### 3. `params`
```json
"params": ["0x1234...", "latest"]
```
这是传给 Method 的参数。

对于 `eth_getBalance`：
```text
第一个参数：address
第二个参数：block tag / block number
```

所以：
```text
eth_getBalance(address, latest)
```
意思是：查询这个 address 在最新状态下的 ETH Balance。

### 4. `id`
```json
"id": 1
```
`id` 不是 Ethereum 的 block number、transaction id，也不是数据库主键。

它的作用只是：
> 让客户端把 Response 和原来的 Request 对应起来。

例如程序连续发两个请求：
```text
Request id=101 → eth_getBalance(Alice)
Request id=102 → eth_getBalance(Bob)
```

远端返回时：
```text
Response id=102
Response id=101
```

客户端仍然可以通过 `id` 知道每个结果属于哪个请求。

## 三、正常 Response 长什么样
成功时通常类似：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": "0xde0b6b3a7640000"
}
```

这里最重要的是：
```json
"result": "0xde0b6b3a7640000"
```

Ethereum JSON-RPC 很多数值使用十六进制 Quantity 返回，因此 SDK 通常会帮你转成更容易使用的整数。

例如 Web3.py 最终可能让你看到：
```python
1000000000000000000
```
也就是 1 ETH = 10^18 Wei。

【数据工程师视角】因此 Raw RPC Response 和我们最终落入分析表的数据往往不是完全同一种表达：
```text
RPC Raw Hex
   ↓ decode / normalize
Integer / Decimal
   ↓ semantic transform
ETH / USD 等业务字段
```

## 四、失败时不是 `result`，而是 `error`
如果 RPC 请求无法正常执行，可能得到：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid params"
  }
}
```

也就是说 Response 通常是二选一：
```text
result
或
error
```

`error` 里面常见：
```text
code
message
```
有时还会有额外 `data`。

## 五、最容易混淆：HTTP Error 和 RPC Error 不是一回事
【网络传输视角】HTTP 负责的是“这次 HTTP 请求有没有被服务器正确接收和处理到应用层”。

例如：
```text
HTTP 200
```
只说明 HTTP 层请求成功返回了一个响应。

但响应体里面完全可能是：
```json
{
  "error": {
    "code": -32602,
    "message": "Invalid params"
  }
}
```

所以可能出现：
```text
HTTP：成功
JSON-RPC：失败
```

反过来，如果 Endpoint 本身不存在、Provider Gateway 故障、认证失败或发生网络错误，则可能在 HTTP / 网络层就失败，甚至根本到不了 Ethereum RPC Method。

这对写 Indexer 很重要，因为错误处理至少要区分：
```text
网络 / HTTP 错误
        vs
JSON-RPC 业务错误
```

不能简单地写成：
```text
只要 status_code == 200 就算成功
```

## 六、一次请求真正经过哪些层
现在可以把第一课的路径再展开一点：
```text
Python / Web3.py
      ↓
构造 JSON-RPC Request
      ↓
HTTP / WebSocket
      ↓
RPC Provider Endpoint（如果使用 Provider）
      ↓
Blockchain Client RPC Handler
      ↓
根据 method 执行对应逻辑
      ↓
访问 State / History / EVM 等
      ↓
形成 result 或 error
      ↓
JSON-RPC Response
      ↓
HTTP / WebSocket 返回
      ↓
Python / Web3.py
```

这里有一个重要结论：
> RPC Method 名字虽然看起来像函数名，但不同 Method 后面访问的东西并不一样。

例如：
```text
eth_getBalance → State
eth_getBlockByNumber → Block / History
eth_getLogs → Historical Logs 扫描 / 过滤
eth_call → EVM 模拟执行
```

这正是 Module 5 后面要继续讲“为什么不同 RPC 成本差异很大”的基础。

## 七、银行系统类比
可以把 JSON-RPC 想成一套统一的远程调用报文。

例如银行接口请求：
```json
{
  "method": "queryAccountBalance",
  "params": ["6222..."],
  "id": 1001
}
```

系统返回：
```json
{
  "id": 1001,
  "result": 5000
}
```

如果账号格式不对：
```json
{
  "id": 1001,
  "error": {
    "code": "INVALID_ACCOUNT",
    "message": "account format invalid"
  }
}
```

所以 JSON-RPC 最核心解决的并不是区块链问题，而是：
```text
我要调用什么？
参数是什么？
这是哪一个请求？
执行结果是什么？
如果失败，失败在哪里？
```

## 八、本课核心模型
先记住：
```text
HTTP / WebSocket
= Transport

JSON-RPC
= Remote Call Message Format / Protocol

eth_getBalance
= Ethereum RPC Method
```

一条完整 Request / Response 最重要字段：
```text
Request:
jsonrpc
method
params
id

Response:
jsonrpc
id
result / error
```

## 理解检查与校准
### 问题 1
用户回答：HTTP 是超文本传输协议；GET、POST 属于 HTTP Method；JSON-RPC 承载 Ethereum RPC 调用；`eth_getBalance` 不是 HTTP 接口，而是 RPC 接口。

老师校准：核心理解正确。更精确地说，`GET` / `POST` 是 HTTP Method；`eth_getBalance` 不是一个完整 Request 实例，而是一个 Ethereum JSON-RPC Method。完整 Request 还包含 `jsonrpc`、`method`、`params`、`id` 等字段。

### 问题 2
用户回答：`id` 用于定位具体 Request；多个请求并发发送时，通过 `id` 可以确定哪个 Response 对应哪个 Request。

老师判断：✅ 通过。补充：JSON-RPC 的 `id` 与 Ethereum transaction hash 无关，只是请求/响应关联标识。

### 问题 3
用户回答：RPC 调用失败。HTTP 200 只表示 HTTP 层成功返回响应，但 JSON-RPC Body 中返回了 `error`，且错误为 `Invalid params`。

老师判断：✅ 通过。核心掌握 `HTTP success ≠ RPC success`。

## 当前状态
✅ 已完成。讲解、理解检查和必要校准均已结束。下一步进入 Module 5 第 3 课。