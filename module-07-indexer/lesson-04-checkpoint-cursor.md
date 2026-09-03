# 第4课｜Checkpoint / Cursor：Indexer 如何知道自己处理到哪个 Block

## Lesson Contract

【Blockchain Data Engineer 视角】

- 所属 Module：Module 7 — Indexer
- 本课核心问题：一个持续运行的 Indexer，怎么知道自己已经处理到哪里？程序宕机以后，怎么从正确位置继续？
- 学完以后应能解释：Checkpoint / Cursor 是什么；为什么生产级 Indexer 不能只靠内存变量保存进度；为什么很多 Indexer 以 Block Height 作为推进位置；Checkpoint 应该在什么时候更新；数据写入和 Checkpoint 更新之间为什么必须考虑一致性；“处理到 Block N”到底意味着什么。
- 本课暂不展开：完整 Idempotency 设计、Reorg 回滚、Backfill 调度、Kafka offset、分布式 Exactly Once。

## 一、先从最简单的程序开始

假设我们写一个 Mini Indexer：

```text
start_block = 20,000,000
for block_number in range(start_block, latest_block):
    fetch block
    parse
    decode
    write database
```

第一次运行会从 20,000,000、20,000,001、20,000,002……持续推进。假设跑到 20,050,000 程序突然崩了，重启以后第一个问题就是：我应该从哪个 Block 继续？这不是 Blockchain 特有的问题，而是所有长期运行数据 Pipeline 都会遇到的问题：我处理到哪里了？

## 二、最原始的做法：内存变量

例如：

```text
last_processed_block = 20,050,000
```

只要程序还活着，这很好用。但程序一退出，Process Memory 消失，下一次启动以后就不知道 last_processed_block 是多少。所以内存变量只能表示当前运行实例的进度，不能作为持久化恢复点。这就是为什么生产系统需要 Checkpoint。

## 三、Checkpoint 是什么？

Checkpoint 是 Indexer 已经可靠完成处理的位置记录。例如：

```text
last_processed_block = 20,050,000
```

保存到数据库：

```text
indexer_checkpoint
chain_id = 1
block_number = 20,050,000
```

程序重启后读取 checkpoint = 20,050,000，再从 20,050,001 开始。Checkpoint = Durable Progress，也就是持久化的处理进度。

## 四、Cursor 又是什么？

很多系统会用 Cursor 而不是 Checkpoint，这两个词实际工程中经常有重叠。可以暂时理解为：Checkpoint = 已经确认完成的位置；Cursor = 当前数据流推进的位置。对于当前 Ethereum Block Indexer，最典型 Cursor 就是 Block Height。更复杂系统里 Cursor 也可能是 block_number + transaction_index + log_index，或者某种 opaque cursor token。

## 五、为什么 Blockchain Indexer 特别适合用 Block Height？

Blockchain 天然是 Block N → N+1 → N+2 连续增长，因此最简单 Indexer 状态机就是：

```text
Checkpoint = N
↓
process N+1
↓
success
↓
Checkpoint = N+1
```

## 六、一个最简单的 Checkpoint Loop

```text
Read Checkpoint = N
↓
Fetch Block N+1
↓
Parse
↓
Decode
↓
Write Data
↓
Update Checkpoint = N+1
↓
Repeat
```

关键顺序是 Write Data → Update Checkpoint，不能轻易反过来。

## 七、错误设计：先更新 Checkpoint

假设当前 Checkpoint = 100，准备处理 Block 101。系统先把 Checkpoint 更新成 101，然后才开始写 Block 101 数据；数据库写到一半 Crash。程序重启后看到 Checkpoint = 101，于是认为 101 已经处理完，直接从 102 开始。但 Block 101 数据并没有完整写入，于是形成 Data Gap。

## 八、所以 Checkpoint 的语义必须非常明确

【工程语义视角】

当 last_processed_block = 101 时，它不能只是说“我看过 Block 101”，而应该表示“Block 101 所要求的所有处理步骤已经可靠完成”。例如 Block fetched、Transactions saved、Receipts saved、Logs saved、Decode finished、Normalized facts saved 全部成功，最后才 Checkpoint = 101。Checkpoint 代表完成承诺。

## 九、Fetched 和 Processed 不一样

例如 latest_fetched_block = 1000，而 latest_processed_block = 997 完全可能。Fetch Thread 可能已经到 1000，而 Processing Worker 只到 997。所以 Fetched ≠ Processed，更不能把 RPC 已经返回 Block 理解成这个 Block 已安全入库完成。

## 十、银行批处理类比

银行每天收到清算文件：2026-09-01、2026-09-02、2026-09-03。系统要读取、校验、解析、写流水表、写账户变动，最后记录“2026-09-02 已完成”。最后这个“已完成”就类似 Checkpoint。如果流水还没完整入库就标记完成，重启后系统会跳过它。Blockchain Indexer 是同一个问题。

## 十一、Checkpoint 应该保存在哪里？

最简单的做法是 Database，例如 indexer_checkpoints，字段可能是 indexer_name、chain_id、last_processed_block、updated_at。放数据库的核心原因是 Persistence，程序挂掉也不会消失。

## 十二、为什么不能写一个本地 txt 文件？

checkpoint.txt 技术上当然可以，Mini Demo 完全可行。但生产系统通常要考虑多实例、容器重启、机器替换、权限、一致性、审计、高可用，所以本地文件可靠性通常不足。Demo 视角可以 local file；生产视角通常更偏向 Database / Durable State Store。

## 十三、Checkpoint 一般是一条记录吗？

不一定。只有 Ethereum Mainnet 且只有一个 Pipeline 时，一条记录就够；多链系统里 Ethereum、Base、Arbitrum、Optimism 各自有不同进度，因此 chain_id 必须进入 Checkpoint Identity。

## 十四、如果有多个不同 Indexer 呢？

例如 raw_block_indexer、erc20_transfer_indexer、dex_swap_indexer、nft_indexer，它们可能处理进度不同，因此 Checkpoint 更像 (indexer_name, chain_id) 作为身份。这说明“链最新高度”不是 Indexer 的进度，Indexer 自己有独立处理状态。

## 十五、Chain Head 和 Checkpoint 的区别

假设 Chain Head = 20,100,000，Indexer Checkpoint = 20,050,000，差距 50,000 Blocks，这个差距叫 Indexer Lag / Sync Lag，即 Indexer 落后链头多少。

## 十六、Lag 是很重要的生产监控指标

如果 chain_head = 20,100,000，checkpoint = 20,099,997，Lag = 3 blocks 可能正常；如果 checkpoint = 20,050,000，Lag = 50,000 blocks，则可能说明 RPC 问题、处理速度不足、数据库写入变慢、Decoder Error 或服务中断。因此生产系统通常监控 head_block - checkpoint。

## 十七、Checkpoint 为什么不等于 Finalized Block？

【Indexer 处理视角】Checkpoint 表示 Indexer 已经处理到哪里。

【链共识视角】Finalized Block 表示哪个 Block 已经达到共识上的最终确定性。

例如 Chain Head = 1000、Checkpoint = 998、Finalized = 950 完全合理，这三个数分别回答不同问题。

## 十八、Checkpoint 是否一定只能推进？

正常同步路径上通常是 100 → 101 → 102 → 103 单调增加。但发生 Reorg 时，生产系统可能需要 rollback 后重新处理，所以 Checkpoint 不能简单理解成永远不能回退。本课暂时只讨论正常同步路径上的单调推进。

## 十九、Checkpoint 更新最大的工程难点是什么？

核心是 Data Write + Checkpoint Write 必须协调。情况 A：Data 写成功，Checkpoint 没更新；情况 B：Checkpoint 更新成功，Data 没写完整。B 更危险，因为会永久跳过数据；A 通常只是重启后重复处理，可以通过 Idempotency 解决。

## 二十、一个非常重要的工程取舍

如果只能在“可能重复处理一次”和“可能漏掉一块数据”中选一个，生产数据系统通常宁愿接受重复处理。Duplicate 可以通过唯一键 / UPSERT / Idempotency 修复，而 Missing Data 很可能更难发现。因此常见原则是：At-least-once processing + Idempotent write，往往比冒险跳过数据更安全。

## 二十一、Checkpoint 更新的安全顺序

```text
Checkpoint = N
↓
Process Block N+1
↓
全部数据写成功
↓
Checkpoint = N+1
```

如果全部数据写成功以后、Checkpoint 更新之前 Crash，重启后 Checkpoint 仍是 N，系统会重新处理 N+1。会出现 Duplicate Processing，但不会出现 Missing Block。因此 Idempotency 会成为 Checkpoint 的天然搭档。

## 二十二、Checkpoint 和 Idempotency 是成对出现的

Checkpoint 告诉系统“从哪里继续”；Idempotency 告诉系统“重复处理也不会把数据搞坏”。组合起来就是 Crash → 重新读 Checkpoint → 可能重跑某些 Block → Idempotent Write → 数据仍正确。

## 二十三、为什么不每条 Log 都更新一次 Checkpoint？

技术上可以用 block + tx_index + log_index 作为 Cursor，但越细粒度，Checkpoint update frequency 越高，系统复杂度和写入开销也会上升。很多 Block-based Indexer 会选择 One Block 作为处理单元，因为 Block 本身就是链天然的数据批次边界。

## 二十四、Block 是一个天然 Batch

【数据工程视角】

可以把 Block 看成 Micro Batch。比如 Block 100 有 300 Transactions、900 Logs，Indexer 可以将整个 Block 100 作为一个逻辑工作单元：Fetch → Parse → Decode → Write → Commit，然后 Checkpoint = 100。因此 Blockchain Indexer 天然具有 Block-oriented Micro-batch 模型。

## 二十五、这和传统 ETL 的 Batch ID 很像

例如银行批处理 batch_id = 20260902_001，当整个批次完成，batch_status = SUCCESS，然后进入下一批。Blockchain 里 block_number 某种程度上天然充当 batch sequence，因此 Indexer 和传统数据平台思想很接近。

## 二十六、Checkpoint 是否应该只存 block_number？

最简单只存 block_number 足够，但更稳健设计通常还会存 block_number + block_hash。block_number 只说明高度，block_hash 说明当时 Indexer 处理的是哪一个具体 Block，这对后续 Reorg Detection 很重要。

## 二十七、Checkpoint = Height + Hash

一个实用心智模型是：Checkpoint = Position + Identity，也就是 block_number + block_hash。block_number 告诉你处理到哪；block_hash 告诉你当时处理的是谁。以后检测 Reorg 时，可以比较 stored block_hash 与 current canonical block_hash。

## 二十八、Checkpoint Table 可以长什么样？

简化的 indexer_checkpoints 字段可以是 chain_id、indexer_name、block_number、block_hash、updated_at，主键可以是 (chain_id, indexer_name)。这只是当前课程里的逻辑模型，具体 DB Schema 后面再完善。

## 二十九、如果程序第一次启动，没有 Checkpoint 怎么办？

可能从 Genesis 开始，也可能从显式配置的 start_block 开始，或者从 current latest - N 开始。生产系统一般会显式配置 initial_start_block，而不是随意猜。

## 三十、为什么不能默认每次都从 Genesis 开始？

如果每次启动都 Genesis → Latest，会产生重复 RPC 成本、重复 CPU Decode、重复 Database Write，而且启动时间极长，完全不可接受。Checkpoint 本质上就是避免重复扫描完整历史。

## 三十一、Checkpoint 其实是一种状态

真正生产 Indexer 并不是完全 Stateless，它至少需要维护 Processing State，例如 Checkpoint、Cursor、Retry State、Reorg State。因此完整 Indexer 可以理解为 Data Pipeline + State Machine。

## 三十二、Indexer 的状态机

```text
Checkpoint N
↓
Fetch N+1
↓
Process N+1
↓
Success?
├── No → Retry
└── Yes → Checkpoint N+1
↓
Repeat
```

这已经开始从脚本走向长期运行服务。

## 三十三、为什么 Mini Indexer 也应该有 Checkpoint？

最终项目目标不是运行一次的 Demo Script，而是可以停止、可以重启、可以继续。所以即使 Mini Indexer 很简单，也至少应该做到 persist checkpoint + resume。

## 三十四、本课核心模型

```text
Checkpoint = N
↓
Fetch Block N+1
↓
Parse / Decode
↓
Write Data
↓
All Success
↓
Checkpoint = N+1
```

关键原则是 Data first, Checkpoint second。重复处理通常比漏数据更容易修复。Checkpoint 最好理解为 Durable Progress，而稳健 Checkpoint = block_number + block_hash = Position + Identity。

## 理解检查

### 问题 1

假设当前 Checkpoint = Block 100，Indexer 开始处理 Block 101。系统先把 Checkpoint = 101 写入数据库，然后才开始保存 Block 101 的 Transactions / Receipts / Logs，保存到一半程序 Crash。重启以后会发生什么问题？为什么这种 Checkpoint 更新顺序是危险的？

**用户回答：**

重启以后，index 会检查 checkpoint 的进度。他会看到 checkpoint 等于 101，然后从下一个 102 高度的 block 开始继续工作。

这样子的话，会造成 block 101 的数据不完整、数据丢失。所以，这种 checkpoint 的更新顺序是危险的

**老师判断与校准：**

✅ 通过。因果链正确：Checkpoint = 100 → 先写成 101 → Block 101 只写了一半 → Crash → 重启读取 Checkpoint = 101 → 直接从 102 开始。结果就是 Block 101 被系统错误地认为已经完成，从而留下永久性数据缺口。这里最危险的是 Progress State > Actual Data State，也就是 Checkpoint 走在真实数据完成状态前面。一旦出现这种情况，系统恢复时就可能跳过未完成的数据。所以应当先完整写数据，再推进 Checkpoint。

### 问题 2

另一种情况：Checkpoint = 100，Block 101 的所有数据已经完整写入数据库，但在更新 Checkpoint = 101 之前程序 Crash。重启以后 Indexer 会怎么做？这种情况为什么通常比问题 1 更容易处理？

**用户回答：**

重启以后，Indexer 会继续从 101 高度的 block 开始继续工作。虽然之前它已经处理过这份数据，但是在 Checkpoint 的更新进度上没能成功更新，所以导致它重启之后再一次处理了 101 block。

但是，因为 Indexer 的幂等性性质，对同一个 block 做多次处理，它的结果也是完全相同的。这种情况下就不会发生数据丢失的问题，所以这个写入顺序更好。

**老师判断与校准：**

✅ 通过，但有一个术语需要校准。恢复过程正确：Checkpoint = 100 → Block 101 已完整写入 → Checkpoint 还没来得及更新 → Crash → 重启 → 再次处理 Block 101。此时风险主要是重复处理，而不是数据缺失。不过 Indexer 本身并不天然具有幂等性，必须通过唯一键、UPSERT、去重逻辑等方式，把写入设计成幂等。也就是说重复处理 ≠ 自动安全，只有当 Idempotent Write 已经设计好时，重复处理 Block 101 才不会造成重复数据或错误累计。问题 1 的本质是 Checkpoint > 实际完成进度 → 可能漏数据；问题 2 是 Checkpoint < 实际完成进度 → 可能重复处理 → 可通过幂等性解决。这也是为什么生产系统通常更愿意接受 At-least-once processing + Idempotent write。

### 问题 3

为什么一个更稳健的 Checkpoint 不只保存 block_number，而是保存 block_number + block_hash？请从 Position、Identity、Reorg 三个角度解释。

**用户回答：**

因为 block number 只是一个表示区块高度或者说位置的数值，具体那个位置上是哪个 block 是无法识别出来的。所以，如果需要对 block 进行身份识别，就需要保存 block hash 来作为 block 的唯一 identity（身份标识）。

因为在区块链 block 的生成过程中，有可能会发生 reorg 的情况

**老师判断与校准：**

✅ 通过。block_number 只表示 Position，也就是处理到了哪个高度；block_hash 提供 Identity，说明当时处理的是哪一个具体 Block。发生 Reorg 时，同一 block_number 可能对应不同 block_hash，因此仅保存高度无法验证“当时处理的 Block 是否仍然是当前 canonical Block”。所以更稳健的 Checkpoint 可以理解为 Position + Identity = block_number + block_hash。

## 本课总结

1. Checkpoint 是 Indexer 的 Durable Progress，不是“看过哪里”，而是“可靠完成到哪里”。
2. 安全顺序是 Data Write first，Checkpoint second；Checkpoint 不能走在真实完成进度前面。
3. Data 已成功但 Checkpoint 未更新时，会导致重复处理；这类风险可以通过幂等写设计解决。
4. Checkpoint 与 Idempotency 是生产级恢复机制的天然组合：At-least-once processing + Idempotent write。
5. Block Height 是 Ethereum Indexer 很自然的 Cursor，Block 也可以视作天然 Micro Batch。
6. Chain Head、Indexer Checkpoint、Finalized Block 是三个不同概念；head - checkpoint 可以衡量 Indexer Lag。
7. 更稳健的 Checkpoint 保存 block_number + block_hash，也就是 Position + Identity，为后续 Reorg Detection 提供基础。
8. Indexer 本质上不仅是 Data Pipeline，也是一个需要持久化 Processing State 的 State Machine。