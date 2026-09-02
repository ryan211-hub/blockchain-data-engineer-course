# 第2课｜Indexer 内部数据流：Raw Block → Parser → Decoder → Normalized Fact → Database

## Lesson Contract

【Blockchain Data Engineer 视角】

- 所属 Module：Module 7 — Indexer
- 本课核心问题：一个 Indexer 从 RPC 拿到原始链上数据以后，内部到底经过哪些处理步骤，才能最终变成可以查询的结构化数据？
- 学完后应能回答 / 设计：解释 Raw Block、Parser、Decoder、Normalized Fact、Database 分别做什么；区分 Parser ≠ Decoder；解释 Block、Transaction、Receipt、Log、Event 分别在 Indexer Pipeline 中处于什么位置。
- 本课明确不展开：Checkpoint / Cursor、Idempotency、Reorg 回滚、Backfill 调度、数据库选型、ABI Registry 的完整设计。

## 一、先看完整流水线

上一课停在：

```text
Blockchain
↓
Node
↓
RPC
↓
Indexer
↓
Database
```

今天打开 Indexer 这一层：

```text
RPC
↓
Raw Data
↓
Parser
↓
Decoder
↓
Normalized Fact
↓
Database
```

更具体一点：

```text
Raw Block
    ↓
Parser
    ↓
Block / Transaction / Receipt / Log Objects
    ↓
Decoder
    ↓
Transfer / Swap / Mint / Burn ...
    ↓
Normalized Fact
    ↓
Database
```

本课最重要的是把每层职责拆开。

## 二、第一层：Raw Data

【RPC / 原始数据视角】

Indexer 一开始拿到的不是 Alice transferred 100 USDC to Bob，而是 RPC 返回的原始结构，例如 Block、Transaction、Log 的 JSON。

这些数据特点是：协议原生 + 编码化 + 业务语义弱，所以这一层通常叫 Raw Layer。

## 三、为什么 Raw Data 很重要？

【数据工程视角】

Raw Data 是最接近链上原始事实的副本。假设以后发现 Decoder 写错了，或者某个 ABI 配错了，如果只保留最终 token_transfers，就可能必须重新调用 RPC；如果保存 raw_logs、raw_transactions、raw_receipts，就可以重新解析。

因此生产系统经常保留：

```text
Raw
↓
Decoded / Normalized
```

这和数仓里的 ODS → DWD 思维很接近。

## 四、第二层：Parser

Parser 负责理解“链上数据结构”，主要解决：这是什么对象？里面有哪些字段？对象之间是什么关系？

例如 Parser 会把原始 RPC JSON 转成程序内部更明确的 Block Object、Transaction Object、Receipt Object、Log Object。

这时候 Parser 还不一定知道某个 Log 是 ERC20 Transfer；它只知道这是一个 Ethereum Log，它有 address、topics、data、log_index。

## 五、Parser 解决的是“结构”

```text
Raw Bytes / JSON
↓
Parser
↓
Blockchain Data Structure
```

它回答的是：这个数据在 Ethereum 数据模型里是什么？而不是这个数据在业务上代表什么。

## 六、第三层：Decoder

【协议语义 / ABI 视角】

Decoder 解决：这些编码字段到底表示什么业务动作？

例如一条 Log 中，Parser 已经知道它是 Log；Decoder 则根据 topics[0]、ABI、topics 和 data 进一步识别 ERC-20 Transfer，并还原 from、to、value。

## 七、Parser 和 Decoder 的区别

```text
Parser = Structure
Decoder = Meaning
```

或者：Parser 回答“它是什么对象？”；Decoder 回答“这个对象表达了什么？”

## 八、银行系统类比

银行收到固定长度报文后，Parser 先按报文格式拆出 transaction_type、account_no、amount、currency、timestamp；业务 Decoder 再解释 transaction_type = 0200 代表 ATM Withdrawal。于是 Parser = 报文结构解析，Decoder = 业务语义解释。

## 九、Block / Transaction / Receipt / Log 分别在哪？

Block 包含 Transactions；Transaction 执行后产生 Receipt，Receipt 中包含 Logs。

Indexer 通常会把这些全部作为原始或基础对象处理，例如 raw_blocks、raw_transactions、raw_receipts、raw_logs。

## 十、一个非常重要的视角前提

【协议执行视角】

```text
Transaction
↓ EVM Execution
Receipt + Logs + State Change
```

Receipt / Logs 是执行结果。

【Indexer 数据处理视角】

```text
Block
↓
Transactions

Transaction Hash
↓
Receipt

Receipt
↓
Logs
```

这是数据获取和关联路径。前者回答“谁产生谁”，后者回答“Indexer 怎么把数据拉回来并关联”。

## 十一、Transaction 需要 Decoder 吗？

需要。Transaction 的 input 例如以 0xa9059cbb 开头，Parser 先得到 tx_hash、from、to、input；Decoder 再识别 0xa9059cbb 为 transfer(address,uint256)，并解析 to 与 amount。

这里要区分 Transaction.to = USDC Contract，而 Decoded transfer.to = Bob。

## 十二、Log Decoder 又是什么？

Log Decoder 处理 topics + data。topics[0] 是 Event Signature，indexed 参数来自 topics，non-indexed 参数来自 data。ERC20 Transfer 中 topics[1] → from，topics[2] → to，data → value。

## 十三、Decoder 的输出还不是最终数据模型

不同协议可能产生 Transfer、Swap、Mint、Burn、Deposit、Withdraw、Borrow、Repay；甚至不同 DEX 的 Swap 字段名称完全不同。如果把 Decoder 输出原样交给业务，上层数据仍然很乱，因此还需要 Normalized Fact。

## 十四、第四层：Normalized Fact

Normalized 的意思是：把不同来源的数据统一成稳定的数据模型。

例如 USDC、USDT、DAI 的 Transfer 都可以统一进入 token_transfers：

```text
chain_id
block_number
block_time
tx_hash
log_index
token_address
from_address
to_address
amount_raw
```

## 十五、为什么叫 Fact？

因为它表达链上发生的一件可观察事实，例如 Alice 在 Block N 通过 Tx X 把 100 USDC 转给 Bob。一行代表 One Token Transfer Event。这和 Module 3 的 One Row = One Pool Swap 是同一个思想。

## 十六、Decoder 和 Normalizer 的区别

Decoder = 把协议编码解释出来；Normalizer = 把解释后的结果映射到统一数据模型。

```text
Raw Log
↓
Parser
↓
Ethereum Log Object
↓
Decoder
↓
ERC20 Transfer Event
↓
Normalizer
↓
token_transfers row
```

## 十七、为什么不能跳过 Normalized 层？

如果没有 Normalized 层，数据库和上层会被迫直接理解 usdc_transfer_events、usdt_transfer_events、uniswap_v2_swap、curve_token_exchange 等协议特有结构；统一成 token_transfers、dex_swaps 后，上层只需要理解稳定模型，不需要知道底层来自哪个协议。

## 十八、第五层：Database

到这里，数据已经经过 Raw → Parsed → Decoded → Normalized，最后进入 blocks、transactions、logs、token_transfers、dex_swaps、nft_transfers 等数据库表。

生产系统可以同时保留 Raw Tables + Normalized Tables：Raw 用于可追溯、可重放、可重新 Decode；Normalized 用于查询、API 和 Analytics。

## 十九、完整 ERC-20 Transfer Pipeline

```text
Ethereum Block
↓
Raw Block JSON
↓
Parser
↓
Block / Transaction / Receipt / Log
↓
Decoder
↓
Transfer(from, to, value)
↓
Normalizer
↓
token_transfer row
↓
Database
```

## 二十、为什么 Parser 和 Decoder 要分层？

小 Demo 可以一个函数全做完，但生产系统通常分层，因为 Parser 更通用、链级别；Decoder 是协议相关的。

```text
Generic Ethereum Parser
          ↓
   ┌──────┼──────┐
   ↓      ↓      ↓
 ERC20  Uniswap  Aave
Decoder Decoder Decoder
```

扩展新协议时不需要重写 Blockchain Parser，只需要增加 Decoder。这是典型的 Separation of Concerns。

## 二十一、一个很实用的工程模型

```text
Chain Structure
↓
Protocol Meaning
↓
Business Data Model
```

对应：

```text
Parser
↓
Decoder
↓
Normalizer
```

第一类逻辑是 Chain-aware，负责 Block / Tx / Receipt / Log；第二类是 Protocol-aware，负责 ERC20 / Uniswap / Aave / NFT；第三类是 Data-model-aware，负责 token_transfers / dex_swaps / lending_positions。

## 二十二、为什么 Raw Layer 不能被忽略？

如果 Uniswap Decoder 有 Bug，且 amount_in / amount_out 写反了，如果只有 dex_swaps，修复时可能需要重新请求 RPC、重新拉历史 Logs、重新 Decode；如果保留 raw_logs，就可以直接 raw_logs → new decoder → rebuild dex_swaps。

核心结论：Raw Data 是重建能力的一部分。

## 二十三、这和数仓分层有什么对应关系？

【类比视角】

可以粗略类比：Blockchain / RPC Raw ≈ Source，Raw Tables ≈ ODS，Normalized Facts ≈ DWD，Aggregates ≈ DWS / ADS。但这只是理解类比，不表示 Blockchain Indexer 必须采用传统数仓命名。

## 二十四、本课核心模型

```text
Raw Blockchain Data
        ↓
      Parser
        ↓
Blockchain Structure
        ↓
      Decoder
        ↓
Protocol Meaning
        ↓
    Normalizer
        ↓
Normalized Fact
        ↓
     Database
```

三个问题：Parser：这是什么链上对象？Decoder：这个对象表达什么协议语义？Normalizer：如何把它统一成我们的数据模型？

## 理解检查

### 问题 1

拿到一条 Ethereum Log：address、topics、data、log_index。Parser 和 Decoder 分别做什么？请明确区分两者。

**用户回答：**

这里拿到的 Ethereum log 已经是 parse 的结果了，接下去是 decode 的工作。它从这个 log 的内容中解析出业务语义，比如这是一个 transfer 或者 swap，还是 mint，还是 burn 等业务操作。

**老师判断与校准：**

✅ 通过。你抓到了一个关键边界：在题目已经给出 address / topics / data / log_index 这种标准 Log 结构时，可以把它看成 Parser 的主要工作已经完成。更精确地说，Raw RPC JSON / Raw Response → Parser → Ethereum Log Object → Decoder → Transfer / Swap / Mint / Burn。Parser 负责“结构是什么”，Decoder 负责“这些字段表达什么协议语义”。

### 问题 2

假设 Uniswap V2 和另一个 DEX 都能产生 Swap，但它们 Event 字段完全不同。为什么不能直接把两个协议的 Decoder 输出原样交给 Dashboard？Normalized Fact 层解决了什么问题？

**用户回答：**

正因为两个协议的 event 字段完全不同，所以不能把两个 decoder 的输出源直接交给 dashboard，而是要通过 Normalized factor 在这里统一业务语义，把两个 decoder 的结果映射到一个统一的数据模型上面，这样子的话再提交给 dashboard。

否则，这个标准化的工作只能在 dashboard 的内部负责整合统一了

**老师判断与校准：**

✅ 通过。Normalized Fact 的核心作用，就是把不同协议 Decoder 输出的“协议特有结构”映射成统一的数据模型。这样 Dashboard 只需要理解 dex_swaps 等统一模型，不需要理解每个 DEX 的 Event 差异。你最后一句尤其关键：如果没有 Normalized Fact，统一语义的责任就会被推到 Dashboard / API / Analyst，各上层消费者会重复做协议适配。复杂性应该尽量在底层一次性处理，再由多个上层复用。

### 问题 3

假设 dex_swaps 已经保存了两年数据，后来发现 Decoder 有 Bug，部分 amount_in 和 amount_out 解析反了。如果系统保留了 raw_logs，修复会有什么优势？如果完全没有 Raw Layer，会发生什么？

**用户回答：**

如果系统保留了 Raw logs，那么就不用再去从 RPC Provider 获取原始数据了，可以直接根据 Raw logs 重新走一遍 indexer 流程。

如果完全没有 raw layer，那么首先需要获取原始数据。获取原始数据的话，需要从 RPC Provider 访问得到，这样会带来另外的经济损失。

**老师判断与校准：**

✅ 通过。保留 raw_logs 后，可以直接从本地 Raw Layer 使用修正后的 Decoder / Normalizer 重放并重建 dex_swaps，不必重新扫描链上历史。没有 Raw Layer，就必须再次依赖 RPC Provider 获取原始数据，增加调用成本、时间成本和外部依赖风险；如果历史范围很大，还会受限于 RPC Provider 的 rate limit、range limit 或历史数据能力。

## 本课总结

1. Parser 负责 Chain Structure；Decoder 负责 Protocol Meaning；Normalizer 负责统一 Business Data Model。
2. Block / Transaction / Receipt / Log 属于链级基础对象；Transfer / Swap / Mint / Burn 属于协议语义；token_transfers / dex_swaps 属于统一事实模型。
3. Raw Layer 不只是“备份”，还是错误修复、重放和重建能力的重要基础。
4. Normalized Fact 把协议差异封装在底层，避免 Dashboard / API / Analyst 重复适配协议。
5. Indexer 的核心数据流可以压缩为 Raw Blockchain Data → Parser → Decoder → Normalizer → Normalized Fact → Database。