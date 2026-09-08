# 第9课｜Mini Indexer 实践：用 Python + Ethereum JSON-RPC 实现最小 ERC-20 Transfer Indexer

## Lesson Contract

【Blockchain Data Engineer 视角】

本课核心问题：

> 第 8 课的架构怎样真正落成一套可以运行的代码？

本课先完成最小主链路：

```text
Fetch
→ Parse
→ Decode
→ Normalize
→ Idempotent Write
→ Checkpoint
```

学完以后，你应该能够：

- 看懂一个最小 Ethereum Indexer 的 Python 实现
- 知道 RPC 原始响应如何进入 Parser
- 手工识别和 Decode ERC-20 `Transfer`
- 把 Transfer 写入 SQLite
- 使用 Unique Constraint 实现 Idempotent Write
- 使用 Checkpoint 连续处理 Block
- 理解代码中的每一层分别对应前 8 课哪个概念

本课暂时不实现：

```text
Backfill
Reorg Recovery
并发 Fetch
WebSocket
Postgres
完整 ABI Framework
```

这些会在主流程真正跑通以后再加。

## 一、这次我们不先写“大项目”

先做一个真正能跑通的最小版本：

```text
Ethereum RPC
↓
指定一个 Block Number
↓
获取 Block
↓
获取每笔 Transaction 的 Receipt
↓
遍历 Logs
↓
找到 ERC-20 Transfer
↓
Decode
↓
写 SQLite
```

然后再加：

```text
Checkpoint
```

让它能够连续跑。

这符合一个基本工程原则：

> 先证明 Data Path 正确，再加入 Control Path。

## 二、项目先简化成 5 个文件

第 8 课我们设计了很多模块。

实践第一步没必要马上拆成十几个文件。

先用：

```text
mini-indexer/
├── config.py
├── rpc.py
├── db.py
├── indexer.py
└── main.py
```

职责：

```text
config.py
→ 配置

rpc.py
→ Ethereum RPC

db.py
→ SQLite / Checkpoint / Writer

indexer.py
→ Parser / Decoder / Processing Engine

main.py
→ Execution Controller
```

等代码稳定以后再细拆。

## 三、首先确认 Transfer Event 长什么样

ERC-20：

```solidity
event Transfer(
    address indexed from,
    address indexed to,
    uint256 value
);
```

它生成的 Log：

```text
address
→ Token Contract Address

topics[0]
→ Transfer Event Signature

topics[1]
→ from

topics[2]
→ to

data
→ value
```

Transfer signature：

```text
keccak256("Transfer(address,address,uint256)")
```

对应：

```text
0xddf252ad...
```

所以 Decoder 最核心的判断：

```python
if log["topics"][0] == TRANSFER_TOPIC:
    # ERC-20 Transfer candidate
```

## 四、先看原始 Log

【RPC / Raw Data 视角】

一个典型 Log 类似：

```python
{
    "address": "0xA0b86991...",
    "topics": [
        "0xddf252ad...",
        "0x000000000000000000000000Alice...",
        "0x000000000000000000000000Bob..."
    ],
    "data": "0x000000000000000000000000..."
}
```

这里还没有：

```text
from_address
to_address
amount
```

这些是 Decoder 之后才产生的。

## 五、config.py

先把配置集中：

```python
RPC_URL = "YOUR_ETHEREUM_RPC_URL"

CHAIN_ID = 1

INDEXER_NAME = "erc20_transfer_indexer"

DB_PATH = "indexer.db"

TRANSFER_TOPIC = (
    "0xddf252ad1be2c89b69c2b068fc378daa"
    "952ba7f163c4a11628f55a4df523b3ef"
)
```

这里先不把 API Key 写死在真实项目里。

生产环境一般应该：

```text
Environment Variable
```

但本课先关注 Indexer 主流程。

## 六、rpc.py：最小 JSON-RPC Client

我们甚至暂时不用 `web3.py`。

直接用：

```text
requests + JSON-RPC
```

因为这样更容易看清：

```text
Indexer
↓
RPC
```

到底发生什么。

```python
import requests

from config import RPC_URL


def rpc_call(method, params):
    payload = {
        "jsonrpc": "2.0",
        "id": 1,
        "method": method,
        "params": params,
    }

    response = requests.post(
        RPC_URL,
        json=payload,
        timeout=30,
    )

    response.raise_for_status()

    result = response.json()

    if "error" in result:
        raise RuntimeError(result["error"])

    return result["result"]
```

这里对应 Module 5 学过的一个重要点：

```text
HTTP 200
≠
RPC Success
```

所以必须检查：

```python
if "error" in result
```

不能只看 HTTP 状态码。

## 七、获取 Block

Ethereum JSON-RPC：

```text
eth_getBlockByNumber
```

Block Number 要传：

```text
hex
```

例如：

```python
def get_block(block_number):
    block_hex = hex(block_number)

    return rpc_call(
        "eth_getBlockByNumber",
        [block_hex, True],
    )
```

第二个参数：

```python
True
```

意味着：

> 返回完整 Transaction Objects，而不是只有 tx_hash。

## 八、为什么还要获取 Receipt？

因为：

```text
Transaction
```

里面没有执行后的 Logs。

我们需要：

```text
Transaction Hash
↓
Receipt
↓
Logs
```

所以：

```python
def get_receipt(tx_hash):
    return rpc_call(
        "eth_getTransactionReceipt",
        [tx_hash],
    )
```

现在已经和第 3 课完全对应：

```text
Block
↓
Transactions
↓
Receipt
↓
Logs
```

## 九、这里的 Fetcher 是什么？

目前：

```python
get_block()
get_receipt()
```

组成：

```text
Fetcher
```

Fetcher 的输出还是：

```text
RPC Raw JSON
```

还没有业务语义。

## 十、现在开始 Parser

我们先不设计复杂 class。

简单从 Raw Log 提取结构字段：

```python
def parse_log(raw_log):
    return {
        "block_number": int(raw_log["blockNumber"], 16),
        "block_hash": raw_log["blockHash"],
        "tx_hash": raw_log["transactionHash"],
        "log_index": int(raw_log["logIndex"], 16),
        "contract_address": raw_log["address"].lower(),
        "topics": raw_log["topics"],
        "data": raw_log["data"],
    }
```

注意：

```python
int(raw_log["blockNumber"], 16)
```

因为 RPC 返回：

```text
0x...
```

这是 hexadecimal。

## 十一、Parser 到这里做了什么？

例如：

```text
"0x12ab"
```

变成：

```text
4779
```

并整理字段：

```text
block_number
block_hash
tx_hash
log_index
contract_address
topics
data
```

但仍然没有：

```text
Transfer
Mint
Burn
```

所以再次强调：

```text
Parser = Structure
```

## 十二、现在写 Transfer Decoder

先判断：

```python
def is_transfer_log(log):
    if len(log["topics"]) != 3:
        return False

    return log["topics"][0].lower() == TRANSFER_TOPIC
```

为什么：

```text
3 topics
```

？

ERC-20 Transfer：

```text
topic0 = event signature
topic1 = from
topic2 = to
```

## 十三、Decode Address

Indexed Address 被编码成：

```text
32 bytes
```

而 Ethereum Address 只有：

```text
20 bytes
```

所以取最后 40 个 hex 字符：

```python
def decode_address(topic):
    return "0x" + topic[-40:].lower()
```

例如：

```text
0x000000000000000000000000ABCDEF...
```

变成：

```text
0xABCDEF...
```

## 十四、Decode Amount

`data`：

```text
0x000000000000000000000000...
```

是一个 uint256。

所以：

```python
def decode_uint256(data):
    return int(data, 16)
```

注意：

这里得到的是：

```text
amount_raw
```

不是：

```text
amount = 100 USDC
```

因为还没有应用：

```text
decimals
```

这个我们在 Module 3 已经学过。

当前 Indexer Raw Fact 最安全保存：

```text
amount_raw
```

## 十五、完整 Decoder

```python
def decode_transfer(log):
    if not is_transfer_log(log):
        return None

    return {
        "token_address": log["contract_address"],
        "from_address": decode_address(log["topics"][1]),
        "to_address": decode_address(log["topics"][2]),
        "amount_raw": decode_uint256(log["data"]),
    }
```

这里输出已经有：

```text
Business Meaning
```

所以这是：

```text
Decoder
```

## 十六、Normalizer

现在把 Source Lineage 和业务字段合在一起：

```python
def normalize_transfer(log, decoded):
    return {
        "chain_id": CHAIN_ID,
        "block_number": log["block_number"],
        "block_hash": log["block_hash"],
        "tx_hash": log["tx_hash"],
        "log_index": log["log_index"],
        "token_address": decoded["token_address"],
        "from_address": decoded["from_address"],
        "to_address": decoded["to_address"],
        "amount_raw": str(decoded["amount_raw"]),
        "canonical": 1,
    }
```

为什么：

```python
amount_raw
```

暂时转成 string？

因为 Ethereum `uint256` 最大值可能超过 SQLite 常规 integer 范围。

这是一个很现实的数据工程问题。

所以学习项目里：

```text
TEXT
```

保存 raw uint256 是更安全的简化方式。

## 十七、现在建立数据库

`db.py`：

```python
import sqlite3

from config import DB_PATH


def get_connection():
    return sqlite3.connect(DB_PATH)
```

初始化表：

```python
def init_db():
    conn = get_connection()

    conn.execute("""
        CREATE TABLE IF NOT EXISTS token_transfers (
            chain_id INTEGER NOT NULL,
            tx_hash TEXT NOT NULL,
            log_index INTEGER NOT NULL,
            block_number INTEGER NOT NULL,
            block_hash TEXT NOT NULL,
            token_address TEXT NOT NULL,
            from_address TEXT NOT NULL,
            to_address TEXT NOT NULL,
            amount_raw TEXT NOT NULL,
            canonical INTEGER NOT NULL,

            UNIQUE(chain_id, tx_hash, log_index)
        )
    """)

    conn.execute("""
        CREATE TABLE IF NOT EXISTS indexer_checkpoints (
            chain_id INTEGER NOT NULL,
            indexer_name TEXT NOT NULL,
            block_number INTEGER NOT NULL,
            block_hash TEXT NOT NULL,

            UNIQUE(chain_id, indexer_name)
        )
    """)

    conn.commit()
    conn.close()
```

## 十八、注意这个 Unique Constraint

```sql
UNIQUE(
    chain_id,
    tx_hash,
    log_index
)
```

这不是普通“数据库优化”。

这是：

```text
Indexer Correctness Rule
```

因为：

```text
same Transfer
↓
same chain_id + tx_hash + log_index
```

所以 Retry 不会产生第二行。

## 十九、Idempotent Writer

SQLite 可以：

```sql
INSERT OR IGNORE
```

所以：

```python
def insert_transfer(transfer):
    conn = get_connection()

    conn.execute("""
        INSERT OR IGNORE INTO token_transfers (
            chain_id,
            tx_hash,
            log_index,
            block_number,
            block_hash,
            token_address,
            from_address,
            to_address,
            amount_raw,
            canonical
        )
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """, (
        transfer["chain_id"],
        transfer["tx_hash"],
        transfer["log_index"],
        transfer["block_number"],
        transfer["block_hash"],
        transfer["token_address"],
        transfer["from_address"],
        transfer["to_address"],
        transfer["amount_raw"],
        transfer["canonical"],
    ))

    conn.commit()
    conn.close()
```

这是当前最简单的：

```text
Idempotent Insert
```

## 二十、Processing Engine 出现了

现在把：

```text
Fetcher
Parser
Decoder
Normalizer
Writer
```

串起来。

```python
def process_block(block_number):
    block = get_block(block_number)

    for tx in block["transactions"]:
        receipt = get_receipt(tx["hash"])

        for raw_log in receipt["logs"]:
            log = parse_log(raw_log)

            decoded = decode_transfer(log)

            if decoded is None:
                continue

            transfer = normalize_transfer(
                log,
                decoded,
            )

            insert_transfer(transfer)

    return block
```

这个函数非常重要。

因为它已经基本等于：

```text
Processing Engine
```

## 二十一、现在你应该能看出架构对应关系

```python
get_block()
get_receipt()
```

对应：

```text
Fetcher
```

```python
parse_log()
```

对应：

```text
Parser
```

```python
decode_transfer()
```

对应：

```text
Decoder
```

```python
normalize_transfer()
```

对应：

```text
Normalizer
```

```python
insert_transfer()
```

对应：

```text
Idempotent Writer
```

所以代码不是一堆函数。

它们实际上对应：

```text
Indexer Architecture
```

## 二十二、现在加入 Checkpoint

读取 Checkpoint：

```python
def load_checkpoint(chain_id, indexer_name):
    conn = get_connection()

    row = conn.execute("""
        SELECT block_number, block_hash
        FROM indexer_checkpoints
        WHERE chain_id = ?
          AND indexer_name = ?
    """, (
        chain_id,
        indexer_name,
    )).fetchone()

    conn.close()

    if row is None:
        return None

    return {
        "block_number": row[0],
        "block_hash": row[1],
    }
```

## 二十三、保存 Checkpoint

```python
def save_checkpoint(
    chain_id,
    indexer_name,
    block_number,
    block_hash,
):
    conn = get_connection()

    conn.execute("""
        INSERT INTO indexer_checkpoints (
            chain_id,
            indexer_name,
            block_number,
            block_hash
        )
        VALUES (?, ?, ?, ?)

        ON CONFLICT(chain_id, indexer_name)
        DO UPDATE SET
            block_number = excluded.block_number,
            block_hash = excluded.block_hash
    """, (
        chain_id,
        indexer_name,
        block_number,
        block_hash,
    ))

    conn.commit()
    conn.close()
```

这里 Checkpoint 本身也用了：

```text
UPSERT
```

## 二十四、为什么 Checkpoint 是 UPSERT？

因为：

```text
(chain_id, indexer_name)
```

表示一个 Indexer 的当前进度。

所以不是：

```text
每处理一个 Block 插一行
```

而是：

```text
同一行不断更新当前位置
```

这是：

```text
Current State
```

而不是历史事实表。

## 二十五、main.py

现在真正有 Execution Controller：

```python
from config import CHAIN_ID, INDEXER_NAME
from db import init_db, load_checkpoint, save_checkpoint
from indexer import process_block


def main():
    init_db()

    checkpoint = load_checkpoint(
        CHAIN_ID,
        INDEXER_NAME,
    )

    if checkpoint is None:
        raise RuntimeError(
            "Please configure a start block first."
        )

    next_block = checkpoint["block_number"] + 1

    block = process_block(next_block)

    save_checkpoint(
        CHAIN_ID,
        INDEXER_NAME,
        next_block,
        block["hash"],
    )


if __name__ == "__main__":
    main()
```

现在它每执行一次：

```text
处理一个 Block
```

## 二十六、为什么先做“一次一个 Block”？

因为现在重点是验证：

```text
Correctness
```

不是：

```text
Throughput
```

所以我们故意先设计：

```text
Run Once
→ Process One Block
```

这样容易：

- 看数据库
- 看日志
- 模拟 Crash
- 重复执行
- 验证幂等性

等这些都正确，再写：

```python
while True:
```

## 二十七、真正的循环其实很简单

以后变成：

```python
while True:
    checkpoint = load_checkpoint(
        CHAIN_ID,
        INDEXER_NAME,
    )

    next_block = (
        checkpoint["block_number"] + 1
    )

    block = process_block(next_block)

    save_checkpoint(
        CHAIN_ID,
        INDEXER_NAME,
        next_block,
        block["hash"],
    )
```

但是目前不要急着无限循环。

## 二十八、这里还有一个明显缺陷

注意：

```python
process_block(next_block)
```

里面：

```text
每一个 Transfer
↓
单独 commit
```

如果一个 Block 有：

```text
500 个 Transfer
```

那么就是：

```text
500 次 DB Commit
```

这并不高效。

更好的版本应该：

```text
整个 Block
↓
一个 DB Transaction
↓
写完所有 Facts
↓
Commit
```

甚至：

```text
Data Write
+
Checkpoint Update
```

放在同一个数据库 transaction 中。

## 二十九、为什么这很重要？

假设：

```text
Block 101
```

有：

```text
500 transfers
```

写到：

```text
第 300 条
```

Crash。

数据库可能只存在：

```text
300 rows
```

Checkpoint 仍然：

```text
100
```

重启后重新跑 101。

由于幂等性：

```text
前 300
→ Ignore

后 200
→ Insert
```

最终仍然正确。

所以即使不是单 transaction：

```text
Correctness
```

仍然可以保证。

但：

```text
Atomic Block Write
```

会让系统更干净。

## 三十、这里要区分两个概念

【Correctness 视角】

```text
At-least-once
+
Idempotency
```

可以允许 Partial Write 后恢复。

【Transaction / Performance 视角】

```text
Batch Write
+
DB Transaction
```

可以减少中间状态和提升效率。

两者不是同一概念。

## 三十一、本课先保留简单版本

当前我们暂时接受：

```text
每条 Transfer 独立写
```

因为本课核心是：

```text
Architecture → Code Mapping
```

不是数据库性能调优。

后续可以再把 Writer 改成：

```text
write_block_atomically()
```

## 三十二、还有一个非常重要的缺陷：Reorg 检查没加

现在代码直接：

```text
Checkpoint = 100A
↓
Fetch 101
↓
Process
```

但还没有检查：

```text
101.parent_hash == 100A.hash ?
```

正确应该先：

```python
if block["parentHash"] != checkpoint["block_hash"]:
    raise RuntimeError("Reorg detected")
```

也就是：

```text
Fetch
↓
Verify Chain Continuity
↓
Process
```

这和上一课完全一致。

## 三十三、但是这一课为什么暂时不实现完整 Reorg？

因为：

```text
Detect Reorg
```

简单。

但是：

```text
Find Common Ancestor
Rollback Old Facts
Replay New Branch
```

已经是另一段完整 Control Path。

我们先保证：

```text
Main Processing Engine
```

真正跑通。

然后再加入：

```text
Reorg Recovery Controller
```

## 三十四、现在整个最小实现已经成立

```text
RPC
↓
get_block
↓
get_receipt
↓
parse_log
↓
decode_transfer
↓
normalize_transfer
↓
INSERT OR IGNORE
↓
Checkpoint
```

这不是一个“区块链脚本”。

它已经拥有最基本的：

```text
Indexer Structure
```

## 三十五、如果你把同一个 Block 跑两次会怎样？

第一次：

```text
100 Transfers
↓
100 INSERT
```

第二次：

```text
same:
(chain_id, tx_hash, log_index)
```

触发：

```text
UNIQUE constraint
```

SQLite：

```text
INSERT OR IGNORE
```

所以：

```text
0 new duplicated rows
```

这就是实际验证：

```text
Idempotency
```

## 三十六、如果 Block 数据写完、Checkpoint 更新前 Crash 呢？

例如：

```text
Checkpoint = 100
```

Block 101：

```text
Transfer rows
全部写完
```

程序 Crash。

Checkpoint 仍是：

```text
100
```

重启：

```text
next_block = 101
```

再次跑：

```text
Block 101
```

Unique Constraint：

```text
阻止重复
```

处理完成后：

```text
Checkpoint = 101
```

这就是：

```text
At-least-once
+
Idempotent Write
=
Effectively Once
```

你已经学过，现在第一次真正落到了代码里。

## 三十七、你现在看到 Checkpoint 为什么必须最后写了

如果先：

```python
save_checkpoint(101)
```

再：

```python
process_block(101)
```

而 Process 中途 Crash：

数据库可能：

```text
只写了 300 / 500 Transfers
```

但是 Checkpoint：

```text
101
```

下一次：

```text
next = 102
```

剩余 200 条永久丢失。

所以：

```text
Write Data First
Checkpoint Second
```

不是风格。

是：

```text
Correctness Requirement
```

## 三十八、这一版 Mini Indexer 的完整心智模型

```text
             Execution Controller
                     │
                     ▼
              Load Checkpoint
                     │
                     ▼
              Determine Block
                     │
                     ▼
                  Fetcher
                     │
                     ▼
                   Parser
                     │
                     ▼
                  Decoder
                     │
                     ▼
                Normalizer
                     │
                     ▼
             Idempotent Writer
                     │
                     ▼
                  Database
                     │
                     ▼
             Save Checkpoint
```

## 三十九、这一课最重要的不是记 Python

你不需要背：

```python
requests.post(...)
sqlite3.connect(...)
```

你真正需要看到的是：

```text
每一个函数
=
Indexer 架构中的一个职责
```

以后即使换成：

```text
Rust
Go
Java
TypeScript
```

架构还是：

```text
Fetcher
Parser
Decoder
Normalizer
Writer
Checkpoint
```

不会变化。

## 四十、本课重点总结

第一：

```text
RPC JSON
≠
Business Fact
```

必须经历：

```text
Fetch → Parse → Decode → Normalize
```

第二：

```text
(chain_id, tx_hash, log_index)
```

在代码中真正成为：

```text
UNIQUE KEY
```

第三：

```text
INSERT OR IGNORE
```

让 Retry 变成安全操作。

第四：

```text
Checkpoint = last reliably processed block
```

因此：

```text
next = checkpoint + 1
```

第五：

```text
Data Write
必须先于
Checkpoint Update
```

第六：

这一版本还只是：

```text
Minimal Processing Engine
+
Minimal Realtime Controller
```

下一步才是：

```text
Crash Test
Backfill
Reorg Recovery
```

## 理解检查

### 问题 1

为什么下面这段代码属于：

```text
Decoder
```

而不是 Parser？

```python
if log["topics"][0] == TRANSFER_TOPIC:
    from_address = decode_address(
        log["topics"][1]
    )

    to_address = decode_address(
        log["topics"][2]
    )

    amount_raw = int(
        log["data"],
        16
    )
```

请从：

```text
Structure
vs
Meaning
```

的角度回答。

### 问题 2

为什么：

```sql
UNIQUE(
    chain_id,
    tx_hash,
    log_index
)
```

对于这个 ERC-20 Transfer Indexer 很重要？

如果没有这个 Unique Constraint，而 Block 101 因为 Checkpoint 没更新被执行两遍，数据库会发生什么？

### 问题 3

假设执行顺序写成：

```text
1. Fetch Block 101
2. Save Checkpoint = 101
3. 写 token_transfers
```

然后：

```text
只写了 300 / 500 条 Transfer
↓
程序 Crash
```

重启以后为什么会发生数据丢失？

为什么正确顺序必须是：

```text
Write Data
↓
Save Checkpoint
```

## 理解检查与追问记录

### 问题 1：用户回答

> 问题一：
>
> 这段代码所处理的内容上看，处理的是 receipt 里面的 log 字段内容，这说明对原始数据的结构化处理已经完成。
>
> 然后，这段代码解析的是 log 里面内容实际发生的业务事件：
>
> 1. 通过判断 topics[0] 是否等于 transfer topic（这个 transfer topic 是事先设定好的合约地址，它明确了具体的业务事件）；
> 2. 如果匹配上某个业务事件，那么下一步继续解析 topics[1]、topics[2] 的数据。
>
> 所以说，这段代码是属于 decoder

### 问题 1：老师判断与校准

问题 1 通过，但有一个关键术语需要校准。

整体判断是对的：这段代码已经不再解决“这个字段是什么结构”，而是在判断“这个 Log 表达了什么业务语义”，所以属于 Decoder。

不过：

> `transfer topic` 是事先设定好的合约地址

这个不准确。

`TRANSFER_TOPIC` 不是合约地址，而是 ERC-20 `Transfer` Event 的事件签名哈希：

```text
keccak256("Transfer(address,address,uint256)")
```

也就是：

```text
topics[0]
=
Event Signature
```

而真正的 Token Contract Address 在 Log 的：

```text
log["address"]
```

所以可以这样区分：

```text
log["address"]
→ 是哪个合约发出的 Log

topics[0]
→ 这个 Log 属于哪一种 Event

topics[1]
→ indexed from

topics[2]
→ indexed to

data
→ value
```

从 `Structure vs Meaning` 角度：

```text
Parser
↓
把 Raw Log 整理成：
address / topics / data / tx_hash / log_index ...

Decoder
↓
理解：
topics[0] = Transfer Signature
所以这是 Transfer Event
再进一步解析 from / to / amount
```

### 重要追问：TRANSFER_TOPIC 为什么能和动态参数共存？

用户追问：

> 这里有点不大理解：transfer 函数里面有三个参数，它们的含义应该分别是 from address、to address，然后是 amount 金额。
>
> 我的问题是：参数的值是随机的，而这个 Transfer Topic 宏是事先写在配置文件里的（已经事先就写好了），那它怎么和函数里面随机传入的地址和金额（也就是要 transfer 的地址和金额）匹配得上呢？

老师解释：

这里卡住的点，是把 `topics[0]` 和后面的参数值混在一起理解了。

【EVM Log / Event 视角】

`topics[0]` 不是拿来和 `from / to / amount` 的“值”做匹配的。

它只是在回答一个问题：

> 这条 Log 是什么类型的 Event？

ERC-20 的 Transfer Event 定义是：

```solidity
event Transfer(
    address indexed from,
    address indexed to,
    uint256 value
);
```

Ethereum 会对“事件名称 + 参数类型”做哈希：

```text
keccak256("Transfer(address,address,uint256)")
```

得到一个固定值，这个固定值就是 `TRANSFER_TOPIC`。

所以不管这次是谁转给谁、金额是多少，只要它发出的事件类型仍然是：

```text
Transfer(address,address,uint256)
```

那么：

```text
topics[0]
```

永远都是同一个固定值。

真正会变化的是后面的参数值。

例如第一次：

```text
Alice → Bob
100 USDC
```

Log 大致是：

```text
topics[0] = Transfer Event Signature   ← 固定
topics[1] = Alice                     ← 变化
topics[2] = Bob                       ← 变化
data      = 100                       ← 变化
```

第二次：

```text
Carol → Dave
500 USDC
```

会变成：

```text
topics[0] = Transfer Event Signature   ← 还是同一个固定值
topics[1] = Carol                     ← 变化
topics[2] = Dave                      ← 变化
data      = 500                       ← 变化
```

所以 Decoder 的逻辑其实是：

```text
先看 topics[0]
↓
确认“这是 Transfer Event”
↓
再按照 Transfer 的字段规则解析
topics[1] → from
topics[2] → to
data      → amount
```

这里最重要的一点是：

`TRANSFER_TOPIC` 对应的是“字段结构 / Event 类型”，不是“字段具体值”。

可以类比数据库里的“交易类型码”。比如银行流水：

```text
transaction_type = TRANSFER
```

是固定业务类型，但每一笔流水里的：

```text
from_account
to_account
amount
```

当然每次都不同。

Ethereum Event 也是类似：

```text
topics[0]
≈ transaction_type
```

而：

```text
topics[1]
topics[2]
data
```

才是这笔具体业务里的参数值。

还要区分：

```solidity
transfer(address to, uint256 amount)
```

这是 ERC-20 函数，通常只有两个显式参数；`from` 是当前调用者 `msg.sender`。

执行成功后合约再发出：

```solidity
emit Transfer(from, to, amount)
```

所以当前讨论的：

```text
from / to / amount
```

是在 `Transfer Event` 中看到的，不等于 `transfer()` 函数本身有三个参数。

完整区分：

```text
Function Input
transfer(to, amount)

        ↓ 执行

Event Output
Transfer(from, to, amount)

        ↓ 编码成 Log

topics[0] = Event 类型
topics[1] = from
topics[2] = to
data      = amount
```

用户随后确认：

> 我明白了。topics[0] 对比的其实应该说是 transfer 函数签名的Signature哈希值，和函数的参数具体的值没有关系。

老师进一步校准：

方向对了，但术语再精确一点：`topics[0]` 对比的是 **Event Signature 的哈希值**，不是 `transfer()` 函数签名的哈希值。

也就是：

```text
keccak256("Transfer(address,address,uint256)")
```

它描述的是：

```text
事件名称 + 参数类型
```

而不是：

```text
具体的 from / to / amount 参数值
```

所以：

```text
topics[0]
→ 识别这是哪一种 Event

topics[1] / topics[2] / data
→ 保存这一次 Event 的具体参数值
```

真正的 ERC-20 函数签名是：

```text
transfer(address,uint256)
```

这是另一回事，通常对应 transaction calldata 里的 function selector。

### 问题 2：用户回答

> 因为 chain_id、tx_hash 和 log_index 已经确定了 block 数据里面的最细粒度，所以它能够作为一条具体 Transfer Fact 的稳定身份。
>
> 如果 checkpoint 没更新，被重复处理两次，且数据库没有这个 UNIQUE constraint 的话，那么 token_transfers 表里就会有重复的数据。

### 问题 2：老师判断与校准

问题 2 通过。

核心判断正确：`(chain_id, tx_hash, log_index)` 能定位到一条具体的 Log，因此对于这个 `ERC-20 Transfer Indexer`，它可以作为一条 Transfer Fact 的稳定身份。

再精确一点：

```text
chain_id
→ 指定哪条链

tx_hash
→ 指定哪笔交易

log_index
→ 指定这笔交易执行过程中产生的哪一条 Log
```

所以三者组合后，可以唯一定位到某条链上、某笔交易里的某一条具体 Log。

而我们的 `token_transfers` 是从这个 Log 解码出来的，因此它可以继承这个 Source Identity。

如果没有：

```sql
UNIQUE(chain_id, tx_hash, log_index)
```

那么发生：

```text
Block 101 数据写成功
↓
Checkpoint 更新前 Crash
↓
重启
↓
Block 101 再处理一次
```

数据库就可能出现重复的 Transfer A / Transfer B，最终污染 Transfer Count、Volume、Wallet Activity、Dashboard Aggregation。

所以这里的 Unique Constraint 本质上不是“方便去重”，而是：

> 把链上事实的 Stable Identity 落成数据库层面的 Correctness Constraint。

### 问题 3：用户回答

> 按这个顺序执行的情况下，程序 crash 重启以后，会先检查 checkpoint，会获取到 checkpoint 等于 101，然后程序会获取下一个 102 block 的数据，然后开始 index。这样子的话，实际上 101 block 的数据只写入了一半，导致数据丢失所以，正确的顺序应该是将 save checkpoint 放在执行流程的最后一个步骤上面。
>
> 即使在程序执行过程中发生了 crash，因为该 block 的 checkpoint 并没有更新到数据库里面，所以程序重启之后，它只会将该 block 重新再 index 解析一遍。但是因为我们程序里有 idempotent 这个处理，所以多次执行的话，也不会有任何数据问题。

### 问题 3：老师判断与校准

问题 3 通过，而且这个回答已经把 Checkpoint、Crash Recovery、Idempotency 三者串起来了。

```text
Checkpoint = 101
↓
重启后 next_block = 102
↓
Block 101 剩余未写的数据不会再处理
↓
Silent Data Loss
```

因此 Checkpoint 只能在整个 Block 被可靠处理后推进。

反过来，如果数据先写、Checkpoint 尚未更新就 Crash，系统会 Replay 同一个 Block，再由 Idempotent Writer 保证重复执行安全。

可以压缩成：

```text
Write Data First
↓
Save Checkpoint Second
```

以及：

```text
At-least-once
+
Idempotent Write
=
Effectively Once
```
