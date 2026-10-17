# Module 14 第 5 课｜Aave / Kamino 的借贷业务与链上数据语义

第四阶段 · Web3 Business Understanding｜Lending

前四课，我们已经理解 DeFi Lending 中 Supplier 提供资金、Borrower 支付利息，以及 Collateral、LTV、Health Factor 和 Liquidation 的商业机制。

现在进入本课问题：**如果同一笔抵押借款发生在 Ethereum 的 Aave 和 Solana 的 Kamino 上，业务意义有哪些共同点，链上证据又有哪些不同？**

## 一、本课学习目标

1. 区分 Aave 和 Kamino 的共同借贷业务模型。
2. 理解 Aave 的 Pool、aToken、Debt Token 与 Kamino 的 Reserve、Obligation 的含义。
3. 认识 EVM Transaction / Log / Event 与 Solana Transaction / Instruction / Account State 的区别。
4. 避免将 Wallet、Transaction、Token Transfer 和 Lending Position 当作同一对象。

本课是第四阶段业务理解课程，不要求编写解析程序或设计数据仓库。

## 二、同一笔借款，发生在两条不同的链上

假设两个业务上相似、参数允许相应操作的场景：

| 业务角色 | Aave（Ethereum / EVM） | Kamino（Solana） |
|---|---|---|
| Borrower：Alice | 抵押价值 $30,000 的 WETH，借入 5,000 USDC | 抵押价值 $30,000 的 SOL，借入 5,000 USDC |
| Supplier：Bob | 提供 USDC，获得存款收益 | 提供 USDC，获得存款收益 |
| 协议目标 | 连接资金供求，以抵押品管理风险 | 连接资金供求，以抵押品管理风险 |

金额为教学假设。实际能否借款取决于各市场的资产支持情况、风险配置及当时流动性。

从经济动机看，两笔业务相同：Alice 不出售已有资产就取得 USDC 流动性，Bob 提供闲置资金争取利息收益，协议按风险规则限制借贷。它们都需要考虑超额抵押、价格风险与清算。

**相同金融业务，不等于相同链上数据结构。**

## 三、Aave：由 Pool 合约管理业务，权益与债务通过代币及状态体现

在 Aave V3 中，`Pool` 是执行 Supply、Borrow、Repay、Withdraw、Liquidation 等操作的核心合约入口。

```text
Ethereum Transaction
         ↓
Aave Pool Contract
         ├─ Supply WETH → 建立抵押资产权益
         └─ Borrow USDC → 建立借款债务
         ↓
Current Position State
         ├─ aToken：供应资产权益
         └─ Variable Debt Token：可变利率债务
```

**aToken**：例如 Bob 向池中提供 USDC 后获得 aUSDC，代表在协议中的供应权益；相关余额可按协议规则体现利息累积。

**Variable Debt Token**：例如 Alice 借入 USDC 后，协议记录相应的可变利率债务。随着利息累积，她的实际债务可能增加。

注意：aToken 不是钱包里原本的底层 USDC；Variable Debt Token 也不是借款人收到的普通 USDC。它们分别表达资产权益与负债语义。

分析 Aave 一次借款时至少区分：

- `Transaction`：链上调用和执行过程。
- `Borrow Event` / `Log`：本次借款历史业务动作的证据。
- `Current Debt State`：某一时点真正的剩余债务，包含应计利息及后续操作影响。

**Borrow Event Amount 不等于 Current Outstanding Debt。**

还应关注用户身份：Aave 的 `borrow()` 包含 `onBehalfOf` 参数，在信用委托等场景中发起交易的 `tx.from` 不一定等于承担债务的地址。不能机械地用交易发起者作为债务人。

## 四、Kamino：业务相似，但仓位组织在 Solana Account 中

Kamino Lending 在 Solana 上运行，其链上执行模型不是 EVM Contract Storage + Solidity Event，而是 Program、Instruction 和 Account。

这里首先理解三个对象：

| Kamino 对象 | 业务含义 |
|---|---|
| Lending Market | 借贷市场与相关配置 |
| Reserve | 某种资产的流动性、利率与风险配置等数据所在的储备账户 |
| Obligation | 关联一个借款仓位中的抵押资产和债务的账户 |

在 Kamino 官方 Lending SDK 中，可以加载 Market 与 Reserve 配置、查询资产储备的统计信息、读取用户 Obligation 内的 deposits 和 borrows，并识别相关借款额度。

假设 Alice 在允许 SOL 抵押的 Kamino 市场中借入 USDC：

```text
Alice Wallet
   │
   └─ Kamino Obligation
          ├─ Collateral: SOL
          └─ Debt: USDC

相关 Reserve Accounts
   ├─ SOL Reserve
   └─ USDC Reserve
```

这里 **Wallet 不等于 Obligation**。一个钱包可能关联多个 Obligation；它们可能代表独立的风险仓位，不能只因为属于同一 Wallet 就默认共用抵押品。

Solana 的 SPL Token 余额或转账也不能直接当作完整的 Obligation 状态。

## 五、Aave 与 Kamino：相同的业务，证据与状态不同

| 观察维度 | Aave（EVM） | Kamino（Solana） |
|---|---|---|
| 执行对象 | Smart Contract | Solana Program |
| 主要业务调用 | Pool 合约方法 | Lending Program Instruction |
| 市场数据 | Pool / Reserve 相关状态 | Lending Market / Reserve Account |
| 用户仓位 | 用户的抵押权益及债务状态 | Obligation 与相关 Reserve 状态 |
| 历史业务证据 | Transaction、Receipt、Log、Event | Transaction、Instruction、Program Log 等 |
| 当前状态依据 | 合约状态、aToken 与 Debt Token 等 | 解码的 Obligation、Reserve 及相关 Account State |
| 共同经济行为 | Supply、Borrow、Repay、Withdraw、Liquidation | Supply、Borrow、Repay、Withdraw、Liquidation |

**Solana Program Log 不等于 Ethereum Solidity Event。**

Solana 的业务动作通常要结合已解码的 Program Instruction、交易执行结果、Token Transfer 以及 Account State 进行解释，而不是假设每次 Kamino Borrow 都有一条与 Aave Solidity `Borrow` Event 一一对应的标准化 EVM Log。

同样，单独观察某笔 USDC Token Transfer 不足以断定发生了新的 Borrow：这也可能是 Repay、Supply、Withdraw 或普通转账。

## 六、Wallet 与 Position 是两层不同的业务身份

银行系统中，一个客户可以有多个贷款账户：

```text
Customer
   ├─ Loan Account A
   └─ Loan Account B
```

Lending 数据也必须明确用户级与仓位级的观察视角：

```text
Wallet
   ├─ Lending Position A
   │     ├─ Collateral
   │     └─ Debt
   └─ Lending Position B
         ├─ Collateral
         └─ Debt
```

- 在 Kamino 中，一个 Wallet 可能拥有多个 Obligation。
- 在 Aave 中，要结合用户、市场、资产和借贷状态识别具体风险承担方，而不能只使用 EVM 交易发起者。

**User-level Aggregate 不等于 Position-level Risk。**

对一个钱包下多个仓位的资产和债务做用户级汇总是可以的；但不能仅凭汇总结果，推断这些仓位在协议内自动共享抵押品或合并清算风险。

## 七、链上数据语义：动作、权益、债务、风险要分开

| 分析指标 | 应理解的业务事实 |
|---|---|
| Borrow Volume | 一段时间内新发生的借款业务规模 |
| Repay Volume | 一段时间内实际偿还的债务规模 |
| Outstanding Debt | 特定时点包含应计利息的未偿债务 |
| Supplied Assets | 供应方的协议资产权益 |
| Liquidation Volume | 已执行清算的规模；必须明确以偿债额还是被处置抵押资产价值计量 |
| At-risk Positions | 根据仓位、资产价格和协议风险参数判断的风险状态 |

例如，不能机械地用 `累计 Borrow − 累计 Repay` 表示准确的 Current Debt，因为债务还可能包含利息累积与其他协议状态变化。

同样，不同协议的风险指标必须核对定义、估值时点、适用的市场与仓位范围后才能比较。

**跨链数据分析的关键不是统一字段名称，而是先确认字段是否表达同一种业务事实。**

## 八、本课总结

1. **相同业务，不同实现。** Aave 以 EVM 合约与状态组织借贷，Kamino 以 Solana Program / Account 组织借贷。
2. **历史动作不等于当前仓位。** Event 或 Instruction 可以说明某次 Borrow 发生过，但单靠这次动作不能确认当前债务。
3. **身份不能混淆。** Wallet、Transaction Sender、Debt Owner 和 Lending Position 不一定一一对应。

## 九、理解检查

本课不重复考 LTV、利息公式或 Health Factor 的计算，重点是跨协议的数据理解。

**问题一：业务事实识别**

你发现 Ethereum 上有一笔与 Aave 相关的 5,000 USDC Token Transfer，Solana 上有一笔与 Kamino 相关的 5,000 USDC Token Transfer。

仅凭转账记录，能否确定两笔都是新的 Borrow？还需要什么证据？

**问题二：Wallet 与 Position 的关系**

Alice 在 Kamino 的同一个钱包下有两个 Obligation：

- Obligation A：抵押 SOL，借入 USDC。
- Obligation B：抵押另一种资产，借入 USDT。

为什么不能仅凭这两个 Obligation 属于同一 Wallet，就将它们当作共享抵押品的单一借贷仓位？

**问题三：跨链风险指标的可比性**

分析平台展示：

- Aave：Alice 的 Health Factor = 1.20。
- Kamino：Alice 的仓位当前 LTV = 72%。

能否直接根据 `1.20` 和 `72%` 判断哪笔仓位更安全？为什么？

**可以一次性回答三题，也可以从问题一开始逐题回答。**

## 协议参考

- [Aave V3 Pool](https://aave.com/docs/aave-v3/smart-contracts/pool)
- [Aave Liquidity Pool](https://aave.com/docs/aave-v3/concepts/liquidity-pool)
- [Kamino Lending SDK](https://github.com/Kamino-Finance/klend-sdk)
- [Kamino KLend Program](https://github.com/Kamino-Finance/klend)

所有资产数量和价格均为教学假设，不代表实时市场配置。

## 教师示范解答记录｜第5课
**本轮真实学习行为：** 学员未独立提交问题一、问题二、问题三的作答，而是明确请求：“这三个问题请你给我解答一下”。因此下列答案属于**教师示范解答**，不能标注为“学员独立答题通过”。
### 问题一：如何确认 5,000 USDC Token Transfer 是否为 Borrow？
**教师参考答案：不能仅凭转账断定新借款发生。** Token Transfer 说明资产流动，但不说明 Supply、Borrow、Repay、Withdraw 等具体借贷业务语义。
- **Aave（EVM）：** 结合成功的 Transaction / Receipt、Pool 的 Borrow Event，以及相关债务状态；注意 Event 中的债务承担者与 `tx.from` 不一定相同。
- **Kamino（Solana）：** 结合成功执行的 Lending Program Borrow Instruction、已解码的 Obligation 与 Reserve 等 Account State、相关 USDC Token Transfer；确认债务和对应仓位确实发生变化。
**结论：Asset Transfer ≠ Lending Business Event。**
### 问题二：为什么同一 Wallet 下两个 Obligation 不自动共享抵押品？
**教师参考答案：Wallet 是用户身份层次，Obligation 是特定仓位的风险核算对象。**
假设 Obligation A 抵押 SOL 借 USDC，Obligation B 抵押其他资产借 USDT。即使两者属于同一个钱包，也不能据此推定其抵押品会自动互相担保。是否可以跨仓位担保要由具体协议仓位与风险配置决定。
可汇总 Alice 的全部资产债务进行 User-level Analysis，但清算风险需要按具体协议定义的 Position / Obligation 范围核算，不能用用户级加总数据直接替代。
**结论：Wallet-level Aggregate ≠ Position-level Risk。** 银行类比：一个客户拥有两笔不同贷款，并不意味着两笔贷款的抵押物天然可以交叉担保。
### 问题三：能否直接比较 Aave HF=1.20 与 Kamino LTV=72%？
**教师参考答案：不能直接比较，因为指标定义不同。**
在简化的单抵押资产、单借款资产模型：
```text
LTV = Debt Value / Collateral Value
HF = (Collateral Value × Liquidation Threshold) / Debt Value
因此 HF = Liquidation Threshold / LTV
```
Kamino LTV=72% 时，若假设 Liquidation Threshold=80%，等效 HF 约为 1.111，低于 Aave 的 1.20；若假设 Liquidation Threshold=90%，等效 HF 为 1.25，高于 Aave 的 1.20。
因此只凭 1.20 与 72% 不能判断哪笔仓位更安全。还需统一清算阈值、资产估值时点、抵押品和债务结构、预言机口径、协议风险规则等。等效 HF 仅作为简化教学推导，不代表 Kamino 官方按同一公式展示风险。
**结论：Different Risk Metrics ≠ Directly Comparable Metrics。**
## 第5课完成记录（业务理解课程，教师示范）
- **三道理解题的回答来源：教师示范解答。**
- **学员独立答题结果：无独立作答记录，不标记“三题独立通过”。**
- **课程处理：按照第四阶段 Business Understanding 优先、允许教师示范而无需强制补考的规则，记为“业务理解课程已完成（教师示范解答，未独立考核）”，不做工程设计验收。**
- **下一课（待开始）：Module 14 第6课｜Lending Business Understanding Review（不重复的业务综合检查）。**

## 第5课结课确认｜学员明确认可通过（2026-10-17）
**学员原话：** 本课可以认为通过，请同步GitHub
**最终课程判定：** 根据学员明确确认及第四阶段以业务理解为主的结课规则，Module 14 第5课《Aave / Kamino 的借贷业务与链上数据语义》标记为 **已通过（业务理解课程通过）**。
**答题来源和考核性质保持不变：** 本课三道题均由教师示范解答，学员没有独立提交这三题的答案，因此本次“课程通过”是学员确认的业务理解结课状态，**不是“三题独立考核通过”**。原有教材和教师示范答案保留不改写。
**下一课：** Module 14 第6课｜Lending Business Understanding Review（待开始）。
