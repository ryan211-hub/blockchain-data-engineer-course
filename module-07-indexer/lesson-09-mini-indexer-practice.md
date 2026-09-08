# 第9课｜Mini Indexer 实践：用 Python + Ethereum JSON-RPC 实现最小 ERC-20 Transfer Indexer

## Lesson Contract
【Blockchain Data Engineer 视角】
- 所属 Module：Module 7 — Indexer
- 本课核心问题：第 8 课的架构怎样真正落成一套可以运行的代码？
- 本课先完成最小主链路：`Fetch → Parse → Decode → Normalize → Idempotent Write → Checkpoint`。
- 学完以后应能：看懂最小 Ethereum Indexer 的 Python 实现；知道 RPC 原始响应如何进入 Parser；手工识别和 Decode ERC-20 `Transfer`；把 Transfer 写入 SQLite；使用 Unique Constraint 实现 Idempotent Write；使用 Checkpoint 连续处理 Block；理解代码中的每一层分别对应前 8 课哪个概念。
- 本课暂时不实现：Backfill、完整 Reorg Recovery、并发 Fetch、WebSocket、Postgres、完整 ABI Framework。

## 一、先把第 8 课架构落成最小可运行链路
先实现：
```text
Ethereum RPC
↓
指定 Block Number
↓
获取 Block
↓
获取每笔 Transaction Receipt
↓
遍历 Logs
↓
找到 ERC-20 Transfer
↓
Decode
↓
写 SQLite
↓
更新 Checkpoint
```
工程策略是：先证明 Data Path 正确，再逐步加入更复杂的 Control Path。

## 二、最小项目结构
```text
mini-indexer/
├── config.py
├── rpc.py
├── db.py
├── indexer.py
└── main.py
```
职责：`config.py` 管配置；`rpc.py` 负责 Ethereum RPC；`db.py` 负责 SQLite / Checkpoint / Writer；`indexer.py` 放 Parser / Decoder / Processing Engine；`main.py` 作为 Execution Controller。

## 三、ERC-20 Transfer Event 与 Log
ERC-20 Event：
```solidity
event Transfer(
    address indexed from,
    address indexed to,
    uint256 value
);
```
Log 编码关系：
```text
log.address  → Token Contract Address
topics[0]   → Transfer Event Signature Hash
topics[1]   → from
topics[2]   → to
data        → value
```
Transfer Event Signature：
```text
keccak256("Transfer(address,address,uint256)")
```
对应固定的 `TRANSFER_TOPIC`。它只标识 Event 类型，与某一笔具体 Transfer 的 from / to / amount 值无关。

## 四、config.py
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
真实项目不应把 API Key 写死在源码中，通常使用环境变量。本课先聚焦 Indexer 主流程。

## 五、rpc.py：最小 JSON-RPC Client
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
这里对应 Module 5 的重要结论：`HTTP 200 ≠ RPC Success`，所以除了 HTTP 状态，还必须检查 JSON-RPC `error`。

## 六、Fetcher：获取 Block 与 Receipt
```python
def get_block(block_number):
    return rpc_call(
        "eth_getBlockByNumber",
        [hex(block_number), True],
    )


def get_receipt(tx_hash):
    return rpc_call(
        "eth_getTransactionReceipt",
        [tx_hash],
    )
```
`eth_getBlockByNumber(..., True)` 返回完整 Transaction Objects。Logs 属于 Transaction 执行结果，应沿 `Transaction Hash → Receipt → Logs` 获取。因此 `get_block()` 和 `get_receipt()` 组成当前最小 Fetcher，其输出仍是 RPC Raw JSON。

## 七、Parser = Structure
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
Parser 将 RPC Raw JSON 整理成结构化字段，例如把十六进制 block number / log index 转成整数，并保留 block_hash、tx_hash、contract_address、topics、data。到这里仍然没有判断 Transfer / Mint / Burn / Swap。

## 八、Decoder = Meaning
先判断是否符合 ERC-20 Transfer Event 的结构与 Event Signature：
```python
def is_transfer_log(log):
    if len(log["topics"]) != 3:
        return False

    return log["topics"][0].lower() == TRANSFER_TOPIC
```
ERC-20 Transfer 的 3 个 topics 分别是 Event Signature、from、to。
Decode indexed address：
```python
def decode_address(topic):
    return "0x" + topic[-40:].lower()
```
Indexed address 被编码成 32 bytes，而 Ethereum Address 是 20 bytes，因此取最后 40 个 hex 字符。
Decode uint256：
```python
def decode_uint256(data):
    return int(data, 16)
```
这里得到的是 `amount_raw`，尚未应用 token decimals。
完整 Decoder：
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
此时已经从结构化 Log 得到业务语义，所以属于 Decoder。

## 九、重要澄清：TRANSFER_TOPIC 为什么是固定的
用户追问：Transfer Event 的 from / to / amount 每笔都不同，为什么 `TRANSFER_TOPIC` 可以事先写在配置里并匹配？
核心区分：
```text
topics[0]
→ Event 类型签名
→ keccak256("Transfer(address,address,uint256)")
→ 固定

topics[1]
→ from 参数值
→ 每笔变化

topics[2]
→ to 参数值
→ 每笔变化

data
→ amount 参数值
→ 每笔变化
```
例如 Alice → Bob 100 USDC 与 Carol → Dave 500 USDC，`topics[0]` 都是同一个 Transfer Event Signature；变化的是 `topics[1]`、`topics[2]` 与 `data`。
还要区分 Function Signature 与 Event Signature：真正的 ERC-20 函数通常是 `transfer(address,uint256)`，而本课 Decoder 匹配的是 Event Signature `Transfer(address,address,uint256)`。前者用于 calldata function selector，后者用于 Log `topics[0]`。

## 十、Normalizer = Business Data Model
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
这里把 Source Lineage 与业务字段合成稳定的 `token_transfers` 模型。`amount_raw` 暂时转成字符串，是因为 Ethereum `uint256` 可能超出 SQLite 常规 integer 范围；学习项目里用 TEXT 保存 raw uint256 是安全的简化方案。

## 十一、数据库表
```python
import sqlite3
from config import DB_PATH

def get_connection():
    return sqlite3.connect(DB_PATH)
```
初始化：
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

## 十二、Stable Identity 落成数据库 Correctness Constraint
```sql
UNIQUE(chain_id, tx_hash, log_index)
```
含义：`chain_id` 指定哪条链，`tx_hash` 指定哪笔交易，`log_index` 指定该交易执行结果里的哪条 Log。对于当前一条 Log → 一条 Transfer Fact 的模型，这组 Source Identity 可以成为 `token_transfers` 的稳定逻辑身份。
这不是单纯“方便去重”，而是把链上事实的 Stable Identity 落成数据库层面的 Correctness Constraint。

## 十三、Idempotent Writer
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
第一次处理时 INSERT；重复处理同一 Log 时命中 Unique Constraint，`INSERT OR IGNORE` 不再写第二条 Logical Fact。

## 十四、Processing Engine
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

            transfer = normalize_transfer(log, decoded)
            insert_transfer(transfer)

    return block
```
架构对应关系：`get_block/get_receipt = Fetcher`；`parse_log = Parser`；`decode_transfer = Decoder`；`normalize_transfer = Normalizer`；`insert_transfer = Idempotent Writer`。代码函数背后其实是 Indexer Architecture 的职责划分。

## 十五、Checkpoint 读取与保存
读取：
```python
def load_checkpoint(chain_id, indexer_name):
    conn = get_connection()
    row = conn.execute("""
        SELECT block_number, block_hash
        FROM indexer_checkpoints
        WHERE chain_id = ?
          AND indexer_name = ?
    """, (chain_id, indexer_name)).fetchone()
    conn.close()

    if row is None:
        return None

    return {
        "block_number": row[0],
        "block_hash": row[1],
    }
```
保存：
```python
def save_checkpoint(chain_id, indexer_name, block_number, block_hash):
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
Checkpoint 表表达一个 Indexer 的当前进度状态，因此同一 `(chain_id, indexer_name)` 不断 UPSERT，而不是每个 Block 都插入新的“事实行”。

## 十六、Execution Controller
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
当前先设计为每次运行处理一个 Block，便于检查数据库、模拟 Crash、重复执行、验证幂等。之后再扩展成 `while True`。

## 十七、为什么 Data Write 必须先于 Checkpoint
正确顺序：
```text
Fetch / Parse / Decode / Normalize
↓
Write Data
↓
Save Checkpoint
```
如果先 `Save Checkpoint=101`，然后才写 Block 101 的 500 条 Transfer，写到 300 条时 Crash，重启后程序看到 Checkpoint=101，就会直接计算 `next_block=102`，剩余 200 条永远不会自动补回，形成 Silent Data Loss。
反过来，如果先写数据、Checkpoint 尚未推进就 Crash，重启后会再次处理同一 Block。依靠 Stable Identity + Unique Constraint + Idempotent Writer，已经写过的数据被安全忽略，未写完的数据继续补齐，最终结果仍正确。
因此：
```text
Checkpoint = last reliably processed block
At-least-once = 允许同一 Block 被重复处理
Idempotency = 保证重复处理不会产生重复结果
```

## 十八、Partial Write、Atomicity 与 Idempotency
当前简单版本每条 Transfer 单独 commit。假设一个 Block 有 500 条 Transfer，写到 300 条时 Crash：Checkpoint 仍在上一个 Block，重启后整个 Block 再跑。前 300 条命中 Unique Constraint 被忽略，后 200 条继续插入，因此 Correctness 仍可恢复。
更好的版本会把整个 Block 的 facts 甚至 Data Write + Checkpoint Update 放在一个 DB Transaction 中，减少中间状态并提升效率。但要区分：
```text
At-least-once + Idempotency
→ Correctness / Recovery

Batch Write + DB Transaction
→ Atomicity / Performance / Cleaner Intermediate State
```
二者不是同一概念。

## 十九、Reorg Detection 尚未展开完整恢复
当前代码还应在处理新 Block 前检查：
```python
if block["parentHash"] != checkpoint["block_hash"]:
    raise RuntimeError("Reorg detected")
```
这一步只是 Detect Reorg。完整 `Find Common Ancestor → Rollback Old Facts → Replay New Branch` 会作为后续 Control Path 实践继续实现。

## 二十、本课核心模型
```text
Execution Controller
↓
Load Checkpoint
↓
Determine Next Block
↓
Fetcher
↓
Parser
↓
Decoder
↓
Normalizer
↓
Idempotent Writer
↓
Database
↓
Save Checkpoint
```
最重要的不是背 Python，而是理解每个函数对应 Indexer 架构里的一个职责。换成 Rust / Go / Java / TypeScript，职责边界仍然成立。

## 理解检查与校准
### 问题 1
为什么下面的逻辑属于 Decoder，而不是 Parser？请从 Structure vs Meaning 角度回答：根据 `topics[0]` 判断 Transfer Event，并进一步把 `topics[1] / topics[2] / data` 解析成 from / to / amount。

**用户回答：**
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

**老师判断与校准：✅ 通过，但有关键术语修正。**
判断逻辑正确：结构化 Log 已经形成，继续识别“这是什么业务 Event”属于 Decoder。需要修正的是：`TRANSFER_TOPIC` 不是合约地址，而是 ERC-20 Transfer Event Signature 的哈希值。真正的合约地址在 `log.address`。

### 重要追问与澄清：TRANSFER_TOPIC 与动态参数值
**用户追问：**
> 这里有点不大理解：transfer 函数里面有三个参数，它们的含义应该分别是 from address、to address，然后是 amount 金额。
>
> 我的问题是：参数的值是随机的，而这个 Transfer Topic 宏是事先写在配置文件里的（已经事先就写好了），那它怎么和函数里面随机传入的地址和金额（也就是要 transfer 的地址和金额）匹配得上呢？

**老师澄清：**
`topics[0]` 不和 from / to / amount 的具体值匹配。它只回答“这条 Log 是什么 Event 类型”。`keccak256("Transfer(address,address,uint256)")` 对 Event 名称与参数类型求哈希，因此固定；每笔交易变化的是 `topics[1]=from`、`topics[2]=to`、`data=value`。
还需区分：ERC-20 函数签名通常是 `transfer(address,uint256)`，而这里的 Log Event Signature 是 `Transfer(address,address,uint256)`。函数 selector 与 Event topic 是两套不同用途的签名。

**用户确认：**
> 我明白了。topics[0] 对比的其实应该说是 transfer 函数签名的Signature哈希值，和函数的参数具体的值没有关系。

**老师校准：**
理解方向正确，但术语应为 **Event Signature 的哈希值**，不是 `transfer()` Function Signature 的哈希值。

### 问题 2
为什么 `UNIQUE(chain_id, tx_hash, log_index)` 对 ERC-20 Transfer Indexer 很重要？如果 Block 因 Checkpoint 未更新被执行两遍，而数据库没有这个 Unique Constraint，会发生什么？

**用户回答：**
> 因为 chain_id、tx_hash 和 log_index 已经确定了 block 数据里面的最细粒度，所以它能够作为一条具体 Transfer Fact 的稳定身份。
>
> 如果 checkpoint 没更新，被重复处理两次，且数据库没有这个 UNIQUE constraint 的话，那么 token_transfers 表里就会有重复的数据。

**老师判断与校准：✅ 通过。**
`chain_id` 定位链，`tx_hash` 定位交易，`log_index` 定位交易执行结果中的具体 Log。对当前一 Log → 一 Transfer Fact 的模型，它可以作为稳定 Source Identity。没有 Unique Constraint 时，At-least-once 重跑会产生重复行，并污染 Transfer Count、Volume、Wallet Activity 和下游聚合。

### 问题 3
如果顺序是 Fetch Block 101 → Save Checkpoint=101 → 写 token_transfers，然后只写 300 / 500 条就 Crash，重启后为什么会丢数据？为什么必须 Write Data → Save Checkpoint？

**用户回答：**
> 按这个顺序执行的情况下，程序 crash 重启以后，会先检查 checkpoint，会获取到 checkpoint 等于 101，然后程序会获取下一个 102 block 的数据，然后开始 index。这样子的话，实际上 101 block 的数据只写入了一半，导致数据丢失所以，正确的顺序应该是将 save checkpoint 放在执行流程的最后一个步骤上面。
>
> 即使在程序执行过程中发生了 crash，因为该 block 的 checkpoint 并没有更新到数据库里面，所以程序重启之后，它只会将该 block 重新再 index 解析一遍。但是因为我们程序里有 idempotent 这个处理，所以多次执行的话，也不会有任何数据问题。

**老师判断与校准：✅ 通过。**
核心完全正确。Checkpoint 只能代表“已经可靠完成”的 Block。若先推进 Checkpoint，Partial Write 后 Crash 会让恢复逻辑直接跳到下一 Block，形成 Silent Data Loss。若最后更新 Checkpoint，Crash 最多导致同一 Block Replay；再由 Idempotent Writer 保证重复执行安全。因此本课的恢复模型是：`Data Write First → Checkpoint Second`，配合 `At-least-once + Idempotent Write = Effectively Once`。

## 本课重点总结
1. `RPC JSON ≠ Business Fact`，必须经历 Fetch → Parse → Decode → Normalize。
2. `Parser = Structure`；`Decoder = Meaning`；`Normalizer = Business Data Model`。
3. `topics[0]` 是 Event Signature Hash，与具体参数值无关；`log.address` 才是发出 Log 的合约地址。
4. `(chain_id, tx_hash, log_index)` 将 Stable Identity 落成数据库 Unique Constraint。
5. `INSERT OR IGNORE` 让 Retry 成为安全操作。
6. `Checkpoint = last reliably processed block`，因此下一次从 `checkpoint + 1` 开始。
7. 正确顺序必须是 `Write Data → Save Checkpoint`。
8. 当前已完成最小 Processing Engine + Realtime Controller；下一步继续实现并验证更完整的 Crash / Backfill / Reorg Control Path。