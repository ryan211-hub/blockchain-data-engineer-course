# Module 14 第 1 课｜为什么需要 DeFi Lending：不卖 ETH，也能借到 USDC？

第四阶段 · Web3 业务理解｜Lending（借贷协议）

我们已经完成 Module 13 — DEX，现在进入 **Module 14 — Lending**。

根据新的第四阶段教学规则，本模块首先学习借贷业务的商业逻辑、参与者、协议机制和链上数据语义。复杂的利率计算、清算算法和数据管道实现暂时不展开。

本课以 Ethereum 生态的 **Aave** 为主要案例。Solana 生态的 Kamino 可以在理解基础机制后进行对比。

## 一、Lesson Contract

**本课核心问题：**

> Alice 持有 ETH，看好 ETH 的长期价值，不想卖掉，但现在需要 5,000 USDC。她能否通过 DeFi Lending 解决资金需求？

学完后，你应该能够解释：

- Lending 为什么存在，它解决了什么资金需求。
- Supplier（存款方）和 Borrower（借款方）是什么关系。
- Collateral（抵押品）与 Borrowed Asset（借入资产）有什么区别。
- Lending Protocol 的收益来自哪里，风险由谁承担。
- 一笔 Supply / Borrow 在链上分别意味着什么。

本课不计算 Health Factor（健康因子），也不讨论清算执行算法。这些内容后面会结合相同业务场景继续学习。

---

## 二、从 Alice 的资金需求开始

假设 Alice 拥有：

```text
Alice Wallet

10 WETH
0 USDC
```

WETH 是 Wrapped Ether，封装后的 ETH。为便于讨论，我们假设当前：

```text
1 WETH = 3,000 USDC

10 WETH ≈ $30,000
```

Alice 现在需要 5,000 USDC。

最直接的办法当然是通过 DEX 卖掉一部分 WETH。

但是 Alice 认为 ETH 将来可能上涨，所以她不想减少 WETH 持仓。

于是产生了一个业务需求：

**Can Alice borrow USDC without selling her WETH?**

可以。一个选择是使用 DeFi Lending Protocol（去中心化借贷协议）。

但出现了一个新问题：

Alice 与协议运营方互不认识。没有工资证明、银行流水，也没有传统银行的信用审批。

协议为什么愿意把 USDC 借给 Alice？

## 三、引入 Collateral：为什么要抵押？

解决方法是：**Overcollateralized Lending（超额抵押借贷）**。

Alice 先向 Lending Protocol 提供有价值的资产作为抵押，才能在协议允许的范围内借出 USDC。

```text
Alice：Borrower
持有 10 WETH，想借 5,000 USDC
            ↓
Lending Protocol
接受 10 WETH 作为 Collateral
在允许的借款额度内提供 5,000 USDC
            ↓
Alice：借款完成
拥有可使用的 5,000 USDC，同时保留对抵押 WETH
的经济敞口，并承担债务
```

注意，10 WETH 的估值是 $30,000，但这不代表 Alice 能够借满 $30,000。

协议会按照资产风险设置借款上限，并要求抵押资产的价值保持足够的安全空间。

而且 Alice 的 WETH 已经进入协议并受到抵押规则约束，**不能像未抵押的钱包余额那样自由转走**。

所以这个业务并不是凭空创造财富：

Alice 获得了 USDC 流动性，同时也获得了一笔需要偿还的债务。

## 四、协议的 USDC 从哪里来？

现在我们发现一个新的问题。

Alice 通过 Lending Protocol 借出了 5,000 USDC，但谁提供了这些 USDC？

假设另一位用户 Bob：

```text
Bob Wallet

20,000 USDC
```

Bob 不准备马上使用这些 USDC，希望通过提供资金赚取收益。

于是 Bob 将 20,000 USDC 存入 Lending Protocol。

这种操作称为 **Supply（向协议提供资产）**。

```text
Bob（Supplier）             Alice（Borrower）
提供 USDC                  提供 WETH 抵押
       ↓                          ↓
           Lending Pool / Protocol
     管理供应资产、借款额度、债务与抵押约束
       ↓                          ↓
Bob：获得 Supply 收益的权利    Alice：借出 5,000 USDC
```

这里的关键是：

**Bob 并不需要找到 Alice，也不需要直接与 Alice 签订借款合同。**

在 Aave 这类 Pool-based Lending Protocol 中，资金由协议统一管理，借款人从相应资产的可用流动性中借款。

这和传统银行有一定相似性。

| 传统银行 | DeFi Lending |
|---|---|
| 存款人把资金存入银行 | Supplier 向协议 Supply 资产 |
| 银行向符合条件的客户贷款 | Protocol 根据抵押品与风险规则允许 Borrow |
| 银行管理存贷款及风险 | Smart Contract 执行协议设定的规则 |
| 利息构成银行经营的重要收入来源 | 借款利息支持 Supplier 收益及协议可能收取的费用 |

但不能简单地认为 Lending Protocol 就是银行。

银行可以通过信用评估发放贷款，并承担相应的法律和资产负债责任。这里讨论的 DeFi Lending 则主要通过链上抵押品和自动执行的协议规则管理风险。

---

## 五、三类参与者为什么愿意参与？

回到 Alice 和 Bob。

**Alice — Borrower**

她获得了 5,000 USDC，而不必立即卖掉 WETH。

但需要支付借款利息，并承担 WETH 价格下跌可能引发的清算风险。

**Bob — Supplier**

他通过向协议提供 USDC，获得相应的存款收益。

但这不等于银行保本存款。他仍然承担智能合约、抵押品市场、资产本身及协议流动性等风险。

**Lending Protocol**

协议负责组织资金供给与借贷需求，并按照规则管理风险。

借款人支付的利息，是 Supplier 收益的重要来源。协议也可能按规则将其中一部分分配给协议储备或其他指定用途。

因此，**Borrow Interest 不等于 Protocol Revenue，也不等于 Supplier 的净利润**。

这一点与上个模块中区分 Pool Gross Swap Fee 和 LP Net Profit 的方法相似。

## 六、Alice 如果不还款怎么办？

这是整个 Lending 业务最重要的风险问题之一。

假设 Alice 用 WETH 作为抵押品借入 USDC。

如果 WETH 价格大幅下跌，那么：

- 她的 USDC 债务不会因为 WETH 跌价而自动消失。
- 但抵押品的美元价值会下降。
- 一旦抵押状况不再满足协议的安全要求，就可能触发 Liquidation（清算）。

清算并非简单地等同于银行催收。

在 DeFi 中，协议可以按照预先约定的规则，允许清算参与者偿还部分债务并取得相应抵押资产，以降低坏账风险。

具体什么时候触发、谁可以清算、清算人为什么愿意参与，是后续 Lending 课程要解决的核心问题。

本课只需要建立一个直觉：

**超额抵押是借贷协议控制信用风险的主要手段之一，但并不能消除所有风险。**

---

## 七、作为 Blockchain Data Engineer，应该理解哪些链上数据？

仍然使用 Alice 和 Bob 的业务。

现在假设我们在链上看到了这些业务动作：

```text
Bob:
Supply 20,000 USDC

Alice:
Supply 10 WETH
Borrow 5,000 USDC
```

它们对应不同的业务含义：

| 链上业务动作 | 真实业务含义 |
|---|---|
| Supply | 用户向协议提供资产 |
| Borrow | 用户建立或增加借款债务，并取得借入资产 |
| Repay | 用户偿还部分或全部债务 |
| Withdraw | 用户从可提取的供应资产中取回资产 |
| Liquidation | 抵押不足等条件下，按照协议规则进行债务与抵押品处置 |

在 Aave 中，这些业务动作可以通过相关 Protocol Events 观察到。

但是，要注意一个重要区别：

**Event records what happened. State tells us what the position looks like now.**

例如 Alice 借入 5,000 USDC 的 Borrow Event，是一项历史借款事实。

随着时间经过，债务可能因利息累积而增加；Alice 也可能部分还款。

因此，我们不能永远把她当前的 Debt 当成初始 Borrow Event 中记录的 5,000 USDC。

本阶段只需要理解这个业务语义。如何重建历史债务余额、如何处理利率累积，将根据后续课程和第五阶段项目需要安排。

---

## 八、本课总结

我们从 Alice 的资金需求出发，发现：

- DEX 解决的是资产兑换问题。
- Lending 解决的是资金借贷与资产流动性需求。
- Supplier 提供可借资金，Borrower 通过抵押资产获得借款。
- Borrower 付利息，Supplier 寻求收益，协议负责规则执行与风险管理。
- 借贷在链上不仅表现为 Token Transfer，还涉及抵押状态、债务状态与利率等业务对象。

因此，**DeFi Lending 不是简单的 Token Transfer，而是一个持续存在、不断变化的债权债务关系。**

## 九、理解检查

继续使用本课中 Alice 和 Bob 的场景，请回答三个问题。

**问题一：资金来源**

Bob Supply 20,000 USDC，Alice 抵押 10 WETH 并 Borrow 5,000 USDC。

为什么不能说“Bob 直接借给了 Alice 5,000 USDC”？Lending Protocol 在其中扮演什么角色？

**问题二：业务利益与风险**

Alice 为什么宁愿抵押 WETH 借 USDC，也不愿直接通过 DEX 卖出 WETH？这种选择为她带来了哪两类主要负担或风险？

**问题三：链上数据语义**

链上有一条记录：

```text
Borrow Event:
user = Alice
asset = USDC
amount = 5,000
```

半年后，我们能否直接根据这条 Borrow Event，认定 Alice 当前仍欠 5,000 USDC？为什么？

**现在从问题一开始回答即可，可以逐题提交。**

## 用户回答（理解检查｜问题一至三）
问题一
因为Bob和Alice并没有直接的交易，他们是通过中间商Lending Protocol来完成这个交易的。
问题二
因为Alice感觉wETH有可能会上涨，所以她不愿意卖出。这个选择带来主要有借款产生的利率，另外是wETH如果下跌的话，所带来的清算风险。
问题三
不能，因为在这个过程中有可能有其他的事件发生，比如爱丽丝可能部分还款。
## 老师判断与校准（第 1 课）
**判定：三道题全部通过。** 第一题有一个术语需要校准，其余两题正确。
### 问题一：Bob 与 Alice 的借贷关系
**基本正确。**
Bob 和 Alice 没有建立直接的一对一借贷关系。不过，“中间商”可以表达协议处于双方之间，但更准确的称呼是 Lending Protocol（借贷协议）或 Pool-based Lending（资金池式借贷）。
它不是简单撮合 Bob 和 Alice，而是通过智能合约统一管理：
- Supplier 提供的可借资金；
- Borrower 的抵押品和借款额度；
- 债务、利息与风险约束。
Alice 借出的 5,000 USDC 来自协议管理的可用流动性，不一定能够对应到 Bob 单独提供的某一笔资金。
### 问题二：Alice 为什么选择抵押借款？
**正确。**
Alice 认为 WETH 可能继续上涨，不愿卖掉它，所以希望在保留 WETH 价格敞口的同时获得 USDC。
她主要承担：
1. Interest Cost（利息成本）：借款会产生利息，债务可能随时间增加。
2. Liquidation Risk（清算风险）：WETH 价格下跌时，抵押品价值下降，可能触发清算。
即使 WETH 最终上涨，借款策略也不一定比直接持有更好，因为她仍然需要支付利息并承担协议风险。
### 问题三：Borrow Event 是否等于当前债务？
**正确。**
你指出 Alice 可能已经部分还款，这是最直接的反例。此外，还存在 Accrued Interest（累积利息）。
```text
初始 Borrow Event：5,000 USDC

之后可能发生：
- 借款利息累积
- 部分还款
- 再次借款
- 全部还清
```
Borrow Event 记录的是一次历史借款动作，而不是永久有效的当前债务余额。
**Event = 发生过的业务事实；Current State = 当前债务与抵押仓位的状态。**
## 第 1 课结课判定
**三道理解检查题均通过，Module 14 第 1 课正式结课。**
已理解资金池式借贷、抵押借款动机、利息与清算风险，以及 Borrow Event 不等于当前债务的时间语义。符合第四阶段 Business Understanding 的本课完成标准，不要求工程实现或补考。
**下一课（待开始）：Module 14 第 2 课｜Collateral、LTV 与 Health Factor：借多少才安全？** 继续使用 Alice 抵押 WETH 借 USDC 的场景。

## 同步去重记录（2026-10-14）

第二次重复追加的原始问答与简版教师校准，已转存到 [Notion 课程同步重复内容归档](https://app.notion.com/p/3f919faa63d681b3ac52e13ff96f6110)，本文件保留第一份原始回答、完整校准与结课判定。
