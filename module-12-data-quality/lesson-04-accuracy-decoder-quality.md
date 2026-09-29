# Module 12 第 4 课｜Accuracy & Decoder Quality：解析正确不等于业务语义正确

## Lesson Contract

所属 Module：Module 12 — 数据质量

本课核心问题：

> How do we know that decoded blockchain data means what we think it means?

学完以后，你应该能够解释并设计：

- 什么是 Accuracy。
- 为什么“成功解析”不等于“解析正确”。
- Raw Log、Decoded Event、Business Fact 之间的关系。
- Decoder Bug 如何产生错误事实。
- ABI / Event Signature / Indexed Parameters / Data Parsing 为什么会影响 Accuracy。
- 为什么 `amount_raw` 正确，不代表 `amount` 或 `amount_usd` 正确。
- Token Decimals、Symbol、Price、Protocol Semantics 为什么属于不同层次的 Accuracy。
- 如何区分 Parser Error、Decoder Error、Enrichment Error、Business Semantic Error。
- 如何使用 Golden Samples / Reference Dataset / Invariant Check 检查 Decoder Quality。
- 为什么 Decoder 修复通常需要 Replay + Update / Replace，而不是只修代码。

本课暂不展开：

- 完整 Reconciliation Framework；
- Reorg Repair；
- 高级 ABI 自动推断；
- Protocol-specific DEX / Lending 业务细节；
- 复杂 ML 异常检测。

---

## 一、Accuracy 到底在问什么？

上一课的 Uniqueness 问：

> Is the same fact stored more than once?

这一课的 Accuracy 问：

> Is the stored fact correct?

例如链上真实发生：

```text
Alice transferred 100 USDC to Bob
```

你的数据库也只有一行，没有重复，也没有缺失。

但是你存成：

```text
Alice transferred 10,000 USDC to Bob
```

那么：

```text
Completeness = PASS
Uniqueness   = PASS
Accuracy     = FAIL
```

所以一个数据集可以：

```text
not missing
not duplicated
but still wrong
```

---

## 二、[Data Engineer 视角] “解析成功”只是 Processing Success

假设 Indexer 收到一个 Ethereum Log：

```text
topics
data
address
tx_hash
log_index
```

Decoder 成功运行，没有异常：

```text
decode_status = SUCCESS
```

这只能说明：

> Decoder executed successfully.

不能说明：

> Decoder produced the correct business meaning.

这和 Module 12 第 1 课的思想完全一致：

> Pipeline Success ≠ Data Correctness.

在这里进一步变成：

> Decode Success ≠ Decode Accuracy.

---

## 三、Raw Log 到 Business Fact 中间发生了什么？

典型链路：

```text
Raw Log
↓
Event Identification
↓
ABI Decode
↓
Decoded Event
↓
Business Mapping
↓
Enrichment
↓
Business Fact
```

例如 ERC-20 Transfer：

```text
Raw Log
↓
Transfer(address,address,uint256)
↓
from
to
value_raw
↓
token decimals
↓
amount
↓
price
↓
amount_usd
```

这个链条里，每一层都可能出错。

---

## 四、Raw Data Correct，不代表 Derived Data Correct

假设节点返回的原始 Log 完全正确：

```text
topics = correct
data   = correct
```

但 Decoder 把 `data` 按错误类型解析。

结果：

```text
Source Quality = PASS
Derived Data Quality = FAIL
```

所以 Accuracy 必须区分：

```text
Source Accuracy
vs
Derived Accuracy
```

很多 Blockchain Data Quality 问题其实发生在：

```text
Transform / Decode / Enrich
```

而不是 Provider。

---

## 五、Parser 和 Decoder 不完全一样

这里需要做一个视角区分。

### Parser

Parser 更偏向：

> 把原始结构拆出来。

例如：

```text
block
transaction
receipt
log
```

或者：

```text
topics[0]
topics[1]
topics[2]
data
```

### Decoder

Decoder 更偏向：

> 把原始字段解释成业务语义。

例如：

```text
topics[1]
↓
from_address
```

```text
topics[2]
↓
to_address
```

```text
data
↓
amount_raw
```

所以：

```text
Parser correct
```

并不意味着：

```text
Decoder correct
```

---

## 六、ERC-20 Transfer 为什么是一个好例子？

Transfer Event：

```text
Transfer(address indexed from, address indexed to, uint256 value)
```

对应常见 Log 结构：

```text
topics[0] = event signature
topics[1] = from
topics[2] = to
data      = value
```

如果 Decoder 把：

```text
topics[1]
```

当成 `to`，

把：

```text
topics[2]
```

当成 `from`，

程序仍然可以正常运行。

但业务事实完全反了。

这就是：

> Syntactically valid, semantically wrong.

---

## 七、Event Signature 判断错误

假设：

```text
topics[0]
```

被错误映射到另一个 Event。

那么可能发生：

```text
Raw Log A
↓
wrong event type
↓
wrong decoder
↓
wrong business fact
```

最危险的是：

```text
no exception
```

因为代码可以“顺利地”产出一条错误记录。

这种错误比程序直接报错更难发现。

---

## 八、Indexed Parameters 判断错误

Ethereum Event 参数分：

```text
indexed
non-indexed
```

Indexed 参数通常进入：

```text
topics
```

Non-indexed 参数通常进入：

```text
data
```

如果 ABI 定义错了，比如把本来 non-indexed 的字段当 indexed，

Decoder 可能从错误位置取值。

结果同样可能是：

```text
decode success
but wrong values
```

---

## 九、Data Type 错误也会造成 Accuracy Failure

假设真实字段：

```text
uint256
```

但 Decoder 按错误类型解释。

或者 offset / tuple layout 理解错误。

那么：

```text
raw bytes = correct
decoded value = wrong
```

这仍然属于：

> Derived Accuracy Failure.

---

## 十、amount_raw 和 amount 不一样

Blockchain 数据平台经常保存：

```text
amount_raw
amount
```

例如 USDC：

```text
amount_raw = 100000000
decimals   = 6
```

那么：

```text
amount = 100
```

公式：

```text
amount = amount_raw / 10^decimals
```

如果：

```text
amount_raw = correct
decimals   = wrong
```

那么：

```text
amount_raw = accurate
amount     = inaccurate
```

这说明 Accuracy 也有层次。

---

## 十一、Decimals Bug 是很典型的 Accuracy Bug

假设 USDC 实际：

```text
decimals = 6
```

系统却配置：

```text
decimals = 18
```

那么：

```text
amount_raw = 100000000
```

会被计算成：

```text
0.0000000001
```

而不是：

```text
100
```

Raw Log 完全没问题。

问题发生在：

```text
Token Metadata / Enrichment
```

所以根因不是：

```text
Provider Error
```

而是：

> Derived Accuracy Failure caused by incorrect token metadata.

---

## 十二、Symbol 正确与否是另一类 Accuracy

假设某 Token Contract：

```text
0xabc...
```

真实 Symbol：

```text
USDC
```

你数据库错误标成：

```text
USDT
```

即使：

```text
amount_raw
from
to
tx_hash
log_index
```

全部正确，

业务展示仍然错误。

所以：

```text
Source Identity Accuracy
≠
Metadata Accuracy
```

---

## 十三、amount_usd 又是下一层

假设：

```text
amount = 100 USDC
```

完全正确。

但 Price Feed 错误：

```text
USDC price = 0.75
```

于是：

```text
amount_usd = 75
```

这时：

```text
Transfer Decoder Accuracy = PASS
Token Amount Accuracy     = PASS
Price Enrichment Accuracy = FAIL
```

最终 Dashboard 仍然错。

---

## 十四、Accuracy 是分层的

可以这样看：

```text
Raw Accuracy
↓
Decode Accuracy
↓
Metadata Accuracy
↓
Business Semantic Accuracy
↓
Enrichment Accuracy
↓
Aggregate Accuracy
```

越往下游：

```text
dependencies increase
```

也就是说：

> Downstream accuracy depends on upstream correctness plus transformation correctness.

---

## 十五、Business Semantic Error 比字段解析更危险

假设你在做 DEX Swap Fact。

Raw Event 成功 decode 出：

```text
amount0
amount1
```

字段值也完全正确。

但你把：

```text
amount0 > 0
```

错误解释成：

```text
token0 bought
```

而协议实际语义是：

```text
token0 sent to pool
```

那么：

```text
raw decode = correct
business semantic mapping = wrong
```

这种错误通常不会被基础 Schema Validation 发现。

---

## 十六、[视角前提] Decoder Correctness 和 Business Correctness 不完全一样

可以区分：

```text
Binary / ABI Correctness
```

问：

> bytes 是否正确解析成字段？

而：

```text
Business Semantic Correctness
```

问：

> 字段是否被正确解释成业务含义？

例如：

```text
amount0 = -10
amount1 = +20000
```

值可能完全 decode 正确。

但你如何解释：

```text
who paid?
who received?
which token is input?
which token is output?
```

属于 Business Semantic Layer。

---

## 十七、为什么区块链特别容易出现这种问题？

因为链上数据天然更接近：

```text
protocol-level data
```

而数据产品需要的是：

```text
business-level facts
```

中间存在巨大语义转换：

```text
bytes
↓
ABI fields
↓
protocol event
↓
business event
↓
analytics fact
```

传统银行数据库往往已经有：

```text
transaction_type
account_id
amount
currency
```

但链上很多时候你拿到的是：

```text
topics
data
contract address
```

这就是 Blockchain Data Engineer 的核心价值之一：

> Convert protocol data into trustworthy business facts.

---

## 十八、银行系统类比

假设核心银行系统传出一条交易：

```text
txn_code = 3102
amount = 1000
```

字段解析完全成功。

但数据平台把：

```text
3102
```

错误映射成：

```text
Deposit
```

其实它代表：

```text
Withdrawal
```

那么：

```text
Parser = PASS
Field parsing = PASS
Business mapping = FAIL
```

这和链上 Decoder 完全一样。

---

## 十九、Accuracy Failure 最危险的地方：它可能“看起来合理”

Missing Data 通常容易暴露：

```text
NULL
missing row
missing block
```

Duplicate 也可能通过：

```text
COUNT > 1
```

发现。

但 Accuracy Error 可能长这样：

```text
amount = 100
```

这个数值看起来完全合理。

真实值却应该是：

```text
1000
```

所以 Accuracy 检测通常比 Completeness / Uniqueness 更困难。

---

## 二十、Schema Validation 为什么不够？

假设字段定义：

```text
amount NUMERIC
```

错误数据：

```text
100
```

正确数据：

```text
1000
```

两者都满足：

```text
NUMERIC
NOT NULL
amount > 0
```

所以：

```text
Schema Validity = PASS
Accuracy        = FAIL
```

Validity 和 Accuracy 不是一回事。

---

## 二十一、那怎么检查 Decoder Accuracy？

不能只依赖：

```text
no exception
row count
schema check
```

通常需要多种证据。

例如：

```text
Golden Samples
Reference Data
Cross-source Comparison
Invariant Checks
Known Transaction Verification
Protocol-specific rules
```

---

## 二十二、Golden Sample 是什么？

Golden Sample 可以理解为：

> 一组人工确认过、结果已知的代表性链上样本。

例如选 100 笔 ERC-20 Transfer：

```text
tx_hash
log_index
expected_from
expected_to
expected_amount_raw
```

然后 Decoder 每次改动以后跑：

```text
actual
vs
expected
```

如果不一致：

```text
test fail
```

这就是最基础的：

> Decoder Regression Test.

---

## 二十三、为什么 Golden Sample 很重要？

Decoder Bug 常见于：

```text
ABI change
code refactor
protocol upgrade
new event variant
edge case
```

如果只有正常单元测试：

```text
synthetic test data
```

有可能遗漏真实链上的复杂情况。

Golden Sample 直接来自：

```text
real blockchain transactions
```

因此对链上 Decoder 很有价值。

---

## 二十四、Reference Dataset 是什么？

Reference Dataset 比单个 Golden Sample 更系统。

例如：

```text
block range 20,000,000–20,001,000
```

由：

```text
trusted implementation
manual validation
known-good provider
historical verified output
```

生成一份：

```text
reference transfer dataset
```

然后新 Decoder 输出：

```text
candidate dataset
```

做比较：

```text
Reference
vs
Candidate
```

---

## 二十五、Reference Comparison 看什么？

至少可以比较：

```text
row count
unique keys
from/to
amount_raw
contract_address
event type
```

更进一步：

```text
aggregate amount
per-token count
per-block count
```

这就开始进入后面 Reconciliation 的内容。

本课只需要记住：

> Known-good reference is one of the strongest tools for Decoder Accuracy validation.

---

## 二十六、Invariant Check 可以发现什么？

Invariant 是：

> 按业务或协议语义，本来应该一直成立的规则。

例如 ERC-20 Transfer：

```text
from_address != NULL
to_address   != NULL
amount_raw   >= 0
```

但这些还比较弱。

更强的 Invariant 可能是：

```text
decoded event count
=
matching raw Transfer logs count
```

或者某协议中：

```text
token0 + token1 relationship
```

符合特定规则。

---

## 二十七、Invariant 不一定证明“完全正确”

即使所有 Invariant 都通过：

```text
PASS
```

也不能绝对证明 Decoder 100% 正确。

因为：

```text
wrong data may still satisfy invariants
```

所以工程上更准确地说：

> Invariants provide evidence of correctness, not mathematical proof of full semantic accuracy.

---

## 二十八、Cross-source Comparison 也能帮助 Accuracy

例如同一笔 Transfer：

```text
your decoder
vs
Etherscan-style explorer
vs
another analytics provider
```

如果：

```text
from/to/amount
```

不一致，

说明有问题值得排查。

但和上一课类似：

> Difference is evidence, not automatic proof which side is correct.

仍然要检查：

```text
same transaction
same log
same token metadata
same decimal assumptions
same interpretation
```

---

## 二十九、Decoder Version 很重要

假设你修复 Decoder Bug。

旧数据：

```text
decoder_version = v1
```

新逻辑：

```text
decoder_version = v2
```

如果系统完全不记录版本，

你很难回答：

```text
哪些历史数据由旧 Decoder 产生？
哪些数据可能受影响？
应该 Replay 哪个范围？
```

因此实践中常会保留类似：

```text
decoder_name
decoder_version
decoded_at
```

这些字段。

---

## 三十、Bug Blast Radius 怎么定位？

发现 Decoder Bug 后，第一个问题不是：

> 改代码了吗？

而是：

> Which historical facts were produced by the buggy decoder?

可能范围：

```text
contract addresses
event signatures
block range
decoder version
protocol version
```

例如：

```text
decoder = erc20_transfer_v1
bug start = block 18,000,000
bug fixed = block 19,500,000
```

那么：

```text
affected range
=
18,000,000 → 19,500,000
```

---

## 三十一、修 Decoder 代码不等于修数据

这是本课最重要的工程结论之一。

假设：

```text
Bug existed for 3 months
```

今天代码修好了。

那么：

```text
future data
```

会正确。

但：

```text
past 3 months data
```

仍然错误。

所以完整修复是：

```text
Fix decoder
↓
Identify blast radius
↓
Replay affected range
↓
Update / Replace wrong facts
↓
Rebuild downstream
↓
Re-validate
```

---

## 三十二、为什么这里通常不能用 DO NOTHING？

这和上一课直接连接。

旧错误数据：

```text
same unique key
wrong amount
```

Replay 后：

```text
same unique key
correct amount
```

如果：

```sql
ON CONFLICT DO NOTHING
```

那么：

```text
wrong row remains
```

所以 Accuracy Repair 常需要：

```text
UPDATE
UPSERT
Replace partition
Delete + Insert
```

取决于数据模型。

---

## 三十三、Raw Layer 为什么最好保留？

假设你保留：

```text
Raw Receipt
Raw Log
```

那么 Decoder Bug 修复后：

```text
无需再次依赖 Provider
```

可以直接：

```text
Raw Data
↓
new decoder
↓
rebuild fact
```

这就是 Raw Layer 的重要价值之一：

> Re-decodability.

---

## 三十四、如果只保存 Decoded Fact 会怎样？

如果你只有：

```text
fact_token_transfer
```

没有保存 Raw Log，

后来发现 Decoder 错了，

你可能必须重新：

```text
call RPC
```

获取历史原始数据。

这带来：

```text
provider cost
rate limit
historical availability
provider gap risk
network dependency
```

所以成熟数据平台通常会重视：

```text
Raw / Bronze layer retention
```

---

## 三十五、[Architecture 视角] Raw 是“可重新解释的证据”

Raw 数据的意义不只是：

```text
备份
```

更重要的是：

> 它保存了可以被未来新逻辑重新解释的 source evidence.

例如：

```text
Raw Log
```

今天用：

```text
Decoder v1
```

明天可以用：

```text
Decoder v2
```

重新生成 Derived Fact。

---

## 三十六、Accuracy Validation 应该放在哪里？

理想情况下不是只在最终 Dashboard 检查。

而是：

```text
Raw
↓
Decoder
↓
Decoder Validation
↓
Fact
↓
Business Validation
↓
DWS
```

这样可以尽量在错误传播到下游前拦住。

这属于：

> Shift-left Data Quality.

---

## 三十七、Checkpoint 和 Accuracy Failure

假设当前 Batch：

```text
block 25,000,000 → 25,000,999
```

Decoder Validation 发现：

```text
expected Transfer count = 10,000
decoded valid facts     = 8,300
```

如果这是当前处理范围，

那么：

```text
checkpoint should not advance
```

因为：

> Processing completed, but accuracy validation failed.

这和前面课程一致：

> Checkpoint is a validated-completion claim.

---

## 三十八、历史 Accuracy Bug 和当前 Validation Failure 仍然不同

如果今天发现：

```text
3 months ago decoder was wrong
```

Realtime Checkpoint 已经推进到最新。

那么这是：

> Historical Accuracy Incident.

通常做：

```text
identify range
↓
replay / backfill
↓
replace wrong facts
↓
rebuild downstream
↓
revalidate
```

而不是：

```text
rewind realtime checkpoint by 3 months
```

---

## 三十九、一个完整 Decoder Incident

假设：

```text
USDC decimals
```

被错误配置成：

```text
18
```

而不是：

```text
6
```

链路：

```text
Raw Log correct
↓
amount_raw correct
↓
decimals wrong
↓
amount wrong
↓
DWS volume wrong
↓
Dashboard wrong
```

根因分类：

```text
Source Quality = PASS
Parser = PASS
Decoder raw value = PASS
Metadata Accuracy = FAIL
Derived Accuracy = FAIL
```

这就是为什么事故分析不能只说：

> 数据错了。

而应该说明：

> Which layer became inaccurate?

---

## 四十、本课核心心智模型

可以压缩为：

```text
Raw source
↓
Parse
↓
Decode
↓
Interpret
↓
Enrich
↓
Fact
```

每一层都问：

```text
Did we preserve the correct meaning?
```

所以 Accuracy 的本质不是：

```text
程序有没有跑成功
```

而是：

> Does the stored data faithfully represent the underlying blockchain fact?

---

## 本课核心结论

> Decode Success does not prove Decode Accuracy.

> Raw source data can be correct while derived facts are wrong.

> Parser correctness, ABI decode correctness, business semantic correctness, metadata correctness and enrichment correctness are different layers.

> Accuracy failures are dangerous because wrong values can still look reasonable and pass schema checks.

> Golden Samples, Reference Datasets, Invariant Checks and cross-source comparison provide evidence for Decoder Quality.

> Decoder versioning helps identify the blast radius of historical bugs.

> Fixing decoder code only fixes future processing; historical wrong facts normally require Replay / Backfill and Update / Replace.

> Retaining Raw Logs enables re-decoding and reduces dependence on external Providers during historical repair.

> Current-range Accuracy Validation Failure should block Checkpoint advancement; historical Accuracy incidents should normally use targeted repair.

---

## 理解检查

### 问题一

某个 ERC-20 Log 的 Raw Data 完全正确。

Decoder 也成功执行，没有报错。

但是系统把：

```text
topics[1]
```

解释成：

```text
to_address
```

把：

```text
topics[2]
```

解释成：

```text
from_address
```

实际上两者应该相反。

请回答：

1. Source Quality 是否有问题？
2. Decoder 是否“成功”？
3. 主要属于哪个 Data Quality Dimension？
4. 为什么“没有异常”不能证明数据正确？

### 问题二

某 Token 实际：

```text
amount_raw = 100000000
decimals = 6
```

所以：

```text
amount = 100
```

但系统错误使用：

```text
decimals = 18
```

请回答：

1. `amount_raw` 是否准确？
2. `amount` 是否准确？
3. 这个问题更接近 Raw Decoder Error，还是 Metadata / Enrichment Accuracy Error？
4. 为什么同一行数据中，有些字段可以 Accuracy PASS，而另一些字段 Accuracy FAIL？

### 问题三

你发现一个 Decoder Bug 已经存在两个月。

代码今天已经修复。

历史 Fact Table 中旧数据仍然是错的，并且表有稳定 Unique Key：

```text
chain_id + tx_hash + log_index
```

请回答：

1. 为什么“修代码”还不能算 Data Repair 完成？
2. 你会如何修复历史数据？
3. 为什么这次 Replay 更可能使用 `DO UPDATE / Replace` 而不是 `DO NOTHING`？
4. 修复 Fact 后，为什么还要检查 DWS / ADS？