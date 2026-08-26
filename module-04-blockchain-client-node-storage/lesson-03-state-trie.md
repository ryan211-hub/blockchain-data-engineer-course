# 第3课｜State Trie：Ethereum 为什么不能只用普通 map<Address, Account>？

## 本课目标

从分布式验证与密码学承诺的角度理解 State Trie 的作用，解释为什么 Ethereum 需要 `stateRoot`，以及 State Trie 与底层 Storage Engine 的关系。

## 一、普通 map 能解决什么

如果只考虑单机程序，可以自然地用：

```cpp
map<Address, Account> state;
```

例如：

```text
0xAlice → balance: 10 ETH, nonce: 5
0xBob   → balance: 3 ETH, nonce: 2
```

对于普通业务系统，这已经可以很好地保存和查询账户状态。

## 二、Ethereum 多了一个分布式验证要求

Ethereum 不只是要保存数据，还要让全球不同节点能够验证：

> 大家是否计算出了同一个 State？

如果 Node A 与 Node B 只有一个账户余额不同，不能靠逐个比较所有账户和所有 Contract Storage 来判断。

因此需要把整个 State 映射为一个固定长度的密码学承诺。

## 三、State Root

整个 State 经过 Trie / Hash 结构后得到：

```text
State
  ↓
State Trie
  ↓
State Root
```

可以先把 `stateRoot` 理解为：

> 整个 Ethereum State 的密码学指纹 / 承诺。

只要 State 中关键数据发生变化，最终计算出的 Root 也会发生变化。

## 四、Block 执行与 stateRoot

Block 的状态转换可以理解为：

```text
Previous State
      │
      │ Execute Transactions
      ▼
New State
      │
      ▼
State Trie
      │
      ▼
State Root
```

新的 `stateRoot` 被记录在 Block Header 中。

## 五、验证节点如何使用 stateRoot

验证节点收到 Block 后，可以：

```text
收到 Block
   ↓
自己执行 Transactions
   ↓
计算 New State
   ↓
计算 State Root
   ↓
与 Block Header.stateRoot 比较
```

如果结果一致，说明节点执行后的状态承诺与区块声明一致；如果不一致，则说明执行结果或区块存在问题。

## 六、State Trie 不是为了 SQL 查询

### 【数据库工程师视角】

容易首先想到：

- 查询 Alice 是否快
- 磁盘 IO 是否低
- 数据是否节省空间

### 【Ethereum 协议视角】

State Trie 的核心职责之一是：

> 对整个 State 生成可验证的密码学承诺。

因此不能简单把 State Trie 理解成“更高级的 HashMap”。它承担协议级验证责任。

## 七、局部变化如何导致全局 Root 变化

简化地想象一棵 Merkle 类树：

```text
            Root
           /    \
         H1      H2
        /  \
      H3    H4
     / \
Alice Bob
```

如果 Alice 的状态发生变化：

```text
Alice.balance: 10 ETH → 9 ETH
```

则 Alice 对应节点发生变化，并沿路径逐级影响：

```text
Alice'
  ↓
H3'
  ↓
H1'
  ↓
Root'
```

其他未变化分支可以复用。

核心思想是：

> 局部状态变化，可以得到新的全局密码学承诺，而无需重新 Hash 整个世界状态。

Ethereum 实际使用的数据结构比这个简化示意更复杂，但当前阶段先掌握这一思想。

## 八、State Trie 与 LevelDB / RocksDB 的关系

### 【协议 / 数据结构视角】

```text
Ethereum State
      ↓
State Trie
      ↓
stateRoot
      ↓
Block Header
```

### 【Client 存储实现视角】

Trie 节点数据仍然需要落盘：

```text
Trie Nodes
    ↓
Storage Engine
    ↓
LevelDB / RocksDB
    ↓
SSD
```

因此：

```text
State Trie = 数据结构 / 密码学承诺结构
LevelDB / RocksDB = 持久化 Storage Engine
```

它们处于不同层级，不是替代关系。

## 九、本课理解检查

### 问题1

为什么 Ethereum 不满足于简单使用 `map<Address, Account>` 保存 State？

重点从分布式验证角度回答。

### 问题2

如果 Alice 余额从 10 ETH 变成 9 ETH，为什么整个 `stateRoot` 会变化？

### 问题3

下面哪个描述正确？

A. `Block Header.stateRoot` 直接保存 Ethereum 所有账户余额。

B. `stateRoot` 是整个 State 的密码学承诺 / 指纹，完整 State 数据仍需由节点另外保存。

## 本课状态

✅ 已完成：理解检查通过。

### 理解检查结果

- 能从分布式验证角度解释为什么普通 `map<Address, Account>` 不足以承担 Ethereum State 的协议级验证职责。
- 能解释 State 发生局部变化时，相关 Trie 路径哈希变化并最终导致 `stateRoot` 变化。
- 明确 `stateRoot` 是整个 State 的密码学承诺 / 指纹，而不是完整 State 数据本身。
