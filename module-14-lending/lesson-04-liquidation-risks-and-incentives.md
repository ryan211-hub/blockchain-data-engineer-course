# Module 14 第 4 课｜Liquidation：抵押品价格下跌后，谁承担风险？

第四阶段 · Web3 业务理解｜Lending

前三课，我们已经了解了 Lending 的参与者、借款利率、抵押规则以及 Health Factor。

现在进入借贷协议中最重要的风险处置机制之一：**Liquidation（清算）**。

本课仍然使用 Alice 抵押 WETH 借入 USDC 的案例，重点回答一个商业问题：

**当 Alice 的抵押品价值下降到危险水平时，谁来偿还她的债务？谁能够获得收益？又是谁承担损失？**

## 一、Lesson Contract

本课需要掌握：

- 为什么 Lending Protocol 必须建立清算机制。
- Borrower、Liquidator（清算人）和 Supplier 在清算过程中的不同角色。
- Liquidation Bonus（清算奖励）为什么存在。
- 清算如何改变 Borrower's Debt 与 Collateral Position。
- Liquidation Event 能够说明哪些业务事实，不能说明哪些结果。

本课采用 Aave V3 的典型机制。不展开 Liquidation Bot、交易抢先执行及智能合约实现。

---

## 二、Alice 的借款仓位进入危险状态

回到第 2 课。

Alice 最初提供了 10 WETH，借出 5,000 USDC。

但市场出现大幅下跌，WETH 的价格从 $3,000 跌至 $600。

此时仓位为：

| 参数 | 数值 |
| --- | --- |
| 抵押品价值 | $6,000（10 WETH × $600） |
| 债务价值 | $5,000（5,000 USDC） |
| Liquidation Threshold | 80% |
| Health Factor | 0.96 |

计算过程：

**HF = (6,000 × 80%) / 5,000 = 0.96**

由于 HF 已低于 1，这个仓位在我们假设的 Aave V3 规则下，进入可以被清算的状态。

问题来了：Alice 可能没有注意到市场下跌，也没有主动还款。

协议能不能一直等她？

显然不能。否则，如果 WETH 继续下跌，最终可能无法覆盖她的债务，风险会进一步传递给整个 Lending Pool。

因此，协议需要允许其他参与者主动处理这些高风险仓位。

## 三、Liquidator：为什么有人愿意替 Alice 还钱？

假设 Charlie 是一名清算参与者。

他并没有借钱给 Alice，却发现 Alice 的仓位已经符合清算条件。

按照 Aave V3 的机制，Charlie 可以通过协议提供的清算操作，替 Alice 偿还允许范围内的一部分或全部 USDC 债务。

作为交换，Charlie 从 Alice 的抵押品中获得相应价值的 WETH，并享有协议设定的 Liquidation Bonus。

这构成一种经济激励：

```text
Charlie（Liquidator）
提供 USDC，替 Alice 偿还部分债务
             ↓
       Lending Protocol
减少 Alice 的债务，并处置相应抵押品
             ↓
    Charlie 收到 WETH
取得用于偿债的相应抵押价值及清算奖励
```

在 Aave V3 中，符合清算条件的仓位可以由外部参与者执行清算，并非必须由某个指定机构完成。

关键是：**Liquidator 并不是无偿帮助 Borrower，而是在经济奖励的驱动下帮助协议降低债务风险。**

## 四、清算奖励究竟来自哪里？

我们用一笔具体的清算来解释。

### 业务前提

继续假设：

- Alice 债务：5,000 USDC。
- Charlie 本次允许并决定偿还其中 2,500 USDC。
- Liquidation Bonus：5%。
- WETH 价格：$600。

这里的 50% 偿债比例与 5% 奖励均用于教学演示；实际允许清算比例与奖励取决于协议版本、仓位和资产参数。

### 清算过程

Charlie 偿还：**2,500 USDC**。

假设忽略协议可能从奖励中收取的费用，Charlie 获得的 WETH 参考价值为：

**2,500 × (1 + 5%) = 2,625 USD**

按照 $600/WETH 的假设价格，他获得：

**2,625 / 600 = 4.375 WETH**

### 计算结果

| Charlie 的清算交易 | 数量或参考价值 |
| --- | --- |
| 支付 | 2,500 USDC |
| 获得 | 4.375 WETH（参考价值 $2,625） |
| 理论清算奖励 | $125 |

### 经济含义

以上 $125 不是保证利润，尚未考虑 Gas、交易滑点、价格变化及可能的协议费用。

为什么要设置 Liquidation Bonus？

因为清算人需要付出资金，还面临执行成本和市场风险。如果完全没有经济激励，就可能没有人愿意及时帮助协议处理危险债务。

请注意：

**这 $125 的理论奖励来自 Alice 被处置的抵押品价值，而不是 Bob 额外支付给 Charlie 的资金。**

这意味着清算对 Alice 存在实际的经济代价。

---

## 五、清算完成以后，Alice 的仓位发生了什么？

这是本课最重要的部分。

清算前：

| 项目 | 清算前 |
| --- | --- |
| WETH Collateral | 10 WETH |
| Collateral Value | $6,000 |
| USDC Debt | $5,000 |
| Health Factor | 0.96 |

清算后，按照前面的简化示例：

| 项目 | 清算后 |
| --- | --- |
| WETH Collateral | 5.625 WETH |
| Collateral Value | $3,375 |
| USDC Debt | $2,500 |
| Health Factor | 1.08 |

新的 Health Factor：

**HF = (3,375 × 80%) / 2,500 = 1.08**

该结果假设清算后价格及参数不变，也不考虑舍入误差和额外费用。

可以观察到：

1. Alice 的债务减少了 2,500 USDC。
2. Alice 的抵押品也减少了 4.375 WETH。
3. Alice 的 Health Factor 从 0.96 上升到 1.08。

因此，有一个容易误解的地方：

**清算并不意味着 Alice 的全部债务都被清除，也不意味着她所有的抵押品都被没收。**

对于部分清算，仓位仍然可以存在，只是债务和抵押资产都发生了变化。Aave V3 的清算比例取决于 Health Factor 和仓位规模等规则，并不总是固定为 50%。

而且，清算后 HF 改善，也不代表 Alice 以后不会再次面临清算。

## 六、到底是谁承担了清算带来的经济后果？

我们分别从四个角色来看。

| 参与者 | 清算带来的影响 |
| --- | --- |
| Alice（Borrower） | 一部分债务被偿还，同时失去相应抵押品及清算奖励对应的价值 |
| Charlie（Liquidator） | 提供偿债资金，取得抵押品并争取清算奖励；承担执行和市场风险 |
| Bob（Supplier） | 协议回收部分借出资金，有助于降低资金池风险；不意味着 Bob 直接收到清算奖励 |
| Lending Protocol | 减少危险债务、保护资金池偿付能力；极端情况下仍可能出现 Bad Debt（坏账） |

这里要重点区分两件事：

**Liquidation 是风险处置机制，不是对所有损失的绝对保险。**

如果市场崩跌过快、抵押品流动性不足、预言机或交易执行出现问题，即使存在清算机制，也可能无法完全避免坏账。

所以，不能认为“有超额抵押和自动清算，Supplier 就一定不会损失”。

## 七、从链上数据看 Liquidation 的业务含义

在前面的案例中，我们可以看到一系列不同的业务事实。

```text
Alice:
Supply 10 WETH
Borrow 5,000 USDC

Market:
WETH price falls

Charlie:
LiquidationCall
  Repay 2,500 USDC
  Receive WETH collateral
```

在 Aave V3 的链上数据中，清算操作会涉及相关的债务、抵押品状态变化及 LiquidationCall Event。

对于 Blockchain Data Engineer，至少需要区分三个层次：

| 数据对象 | 业务意义 |
| --- | --- |
| Liquidation Event | 一次清算实际发生，记录被清算用户、清算人及相关资产数量等 |
| Debt State | 清算后 Alice 还剩多少未偿债务 |
| Collateral State | 清算后 Alice 还有多少有效抵押资产 |

还有一个重要区别：

**Liquidation Bonus 不等于 Liquidator Net Profit（清算人的净利润）。**

即使我们可以通过清算事件计算 Charlie 获得的抵押资产价值，也不能直接认定他净赚了 $125。

真正分析盈利，还需要考虑资产估值、Gas、兑换损耗和其他执行成本。

这与 DEX 模块中学过的“Gross Fee 不等于 Net Profit”是相同的经济分析思路。

---

## 八、本课总结

本课要建立四个核心认识：

1. Liquidation 是借贷协议维护抵押安全与资金池偿付能力的机制。
2. 清算人通过替借款人偿还债务，获得抵押资产及经济奖励。
3. Borrower 通常承担清算奖励对应的抵押资产价值损失，但债务也会随清算减少。
4. 清算可以降低系统风险，却不能保证消除所有坏账或让供应者绝对安全。

## 九、理解检查

继续使用本课 Alice、Bob 和 Charlie 的场景。

**问题一：Liquidator 的经济动机**

Charlie 原本并没有借钱给 Alice。为什么他会愿意使用自己的 USDC 替 Alice 偿还债务？他拿到清算奖励，就一定代表最终获得了净利润吗？

**问题二：Alice 在清算后是否就没有债务了？**

Charlie 偿还了 Alice 的 2,500 USDC 债务，并取得相应 WETH 抵押品。

为什么不能说“Alice 的借款已经全部还清”？她的风险状况可能发生怎样的变化？

**问题三：Supplier 的风险**

Bob 认为：“Lending Protocol 有自动清算机制，所以我提供给协议的 USDC 完全没有资金损失风险。”

你同意这个说法吗？为什么？

**请从问题一开始回答，也可以一次性回答三题。**

## 协议参考

- [Aave｜Health Factor & Liquidations](https://aave.com/help/borrowing/liquidations)
- [Aave V3｜Pool](https://aave.com/docs/aave-v3/smart-contracts/pool)

所有金额、资产价格和 5% 奖励均为教学假设，不代表实时协议配置。
