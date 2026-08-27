# 第6课｜节点同步：为什么新节点不必总从 Genesis 重放？Full Sync、Snap Sync 与状态恢复

## 本课目标

理解 Ethereum 节点首次启动时如何获得可用的 Current State，区分“验证历史”“恢复当前状态”和“保存历史状态”三个问题，并建立 Full Sync / Snap Sync 的基础心智模型。

## 一、理论上的最朴素同步方式

从状态机角度，节点可以从 Genesis State 开始，依次下载、验证并执行全部历史 Block：

```text
Genesis State
    ↓
Execute Block 1
    ↓
State 1
    ↓
...
    ↓
Execute Latest Block
    ↓
Current State
```

理论上完全可行，但现代 Ethereum 历史很长，成本涉及 CPU、EVM 执行、网络、磁盘 I/O、数据库写入与 Trie 更新，因此工程上需要更快的状态恢复方式。

## 二、必须区分的三个任务

### 节点实现视角

节点同步至少包含三个不同问题：

1. History Sync：获得并验证区块链历史数据。
2. State Sync / State Recovery：获得一个可信、可用的 Current State。
3. Live Sync：追上 Chain Head 后持续验证、执行最新 Block 并更新 State。

这三件事相关，但不是同一个任务。

## 三、Full Sync 的核心思想

```text
Download Headers / Blocks
        ↓
Validate
        ↓
Execute Transactions
        ↓
Build State
        ↓
Reach Chain Head
```

即通过历史执行逐步构建当前 State。

## 四、Snap Sync 的核心思想

Snap Sync 的目的不是跳过验证，而是避免本机从 Genesis 重放全部历史执行。

```text
获得近期可信链位置
        ↓
高效获取对应 State 数据
        ↓
通过 stateRoot 等密码学承诺验证
        ↓
获得可信 State@N
        ↓
执行 N 之后的 Block
        ↓
Current State
```

核心模型：

```text
Snapshot + Delta = Current State
```

## 五、为什么不能直接相信别人的 current_state.json

其他 Peer 可以给出 State 数据，但节点不能因为“别人这么说”就相信。

```text
State Data
   ↓
State Trie / Commitment
   ↓
stateRoot
   ↓
Block Header.stateRoot
```

因此 `stateRoot` 将下载到的状态数据与已经确认的 Block Header 建立密码学关联。

## 六、Block 数据验证与 State 验证是两套不同问题

### History / 数据结构完整性视角

Block Header 通过 `parentHash` 形成 Hash-linked chain；Header 中的 `transactionsRoot`、`receiptsRoot` 等承诺分别用于校验 Block Body 和 Receipt 数据与 Header 是否一致。

### State / Execution 视角

`stateRoot` 承诺的是 Block 执行完成后的整个 State。

```text
tx_hash          → 单笔 Transaction 的身份 / 内容承诺
transactionsRoot → 一个 Block 中整组 Transactions 的承诺
blockHash        → Block Header 的身份 / 承诺
receiptsRoot     → Block 执行 Receipts 的承诺
stateRoot        → Block 执行完成后整个 State 的承诺
```

“Transaction 数据完整”不能推出“Current State 正确”，因为：

```text
Previous State
+
Transactions
+
EVM Rules
=
New State
```

## 七、为什么历史 Block 不一定要先全部执行

如果已经获得并验证 `State@Block N`，那么得到更高状态可以直接执行后续 Delta：

```text
State@N
+
Block N+1
+
...
+
Block M
=
State@M
```

因此更早的历史链数据可以继续同步和验证，但不要求先把 Genesis 到 N 的所有历史 Transaction 都由本机重新执行一遍，节点才能获得可用 Current State。

## 八、Consensus 与 Execution 的视角区别

### Consensus Client 视角

PoS Ethereum 中，canonical chain / head 不是单靠 Execution Layer 的 `parentHash` 判断。Consensus Client 依据共识规则、attestation、fork choice、finality 等确定节点应该追随的链头。

### Execution Client 视角

Execution Client 负责验证执行层 Header / Block Body / Receipt / State，并获得正确的 Current State。

```text
Consensus Layer
→ 确定可信 Chain Target / Head
        ↓
Execution Layer
→ 同步并验证 History
→ 获取 / 恢复 State
→ 执行后续 Blocks
→ Current State
```

## 九、银行 / 数据工程类比

```text
Full Sync
≈ 从 10 年交易流水重放到今天余额

Snap Sync
≈ 导入一个可信余额快照
   + 消费之后的增量流水
```

对应常见数据工程模式：

```text
Snapshot + CDC / WAL / Delta
```

## 十、Blockchain Data Engineer 为什么要理解 Node Sync

实际工程中会遇到：

- node is syncing
- latest block lag
- state unavailable
- historical state unavailable
- RPC 数据暂时落后

这些问题可能来自 Chain Sync、State Sync、Live Sync 或节点能力差异，而不一定是 API 或数据 Pipeline 自身出错。

## 十一、理解检查结果

1. 不能直接相信其他节点提供的 Current State；需要通过 `stateRoot` 等密码学承诺验证 State 与可信 Block Header 是否对应。回答正确。
2. Snap Sync 不是“跳过验证”，而是通过更高效的状态获取与验证方式避免从 Genesis 重放全部历史执行。回答正确。
3. 已有并验证 `State@20,000,000` 时，可以顺序执行后续 100 个 Block 得到 `State@20,000,100`，即 Snapshot + Delta。回答正确。
4. 进一步理解了 Block History 完整性验证与 State 正确性验证不能互相替代；`transactionsRoot` 与 `stateRoot` 承诺的是不同对象。

## 本课状态

✅ 已完成。
