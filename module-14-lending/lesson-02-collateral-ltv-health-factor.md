# Module 14 第 2 课｜Collateral、LTV 与 Health Factor：借多少才安全？

第四阶段 · Web3 业务理解｜Lending
上一课，我们学习了 Alice 如何通过抵押 WETH，从 Lending Protocol 借出 USDC。
现在出现一个新的业务问题：
**既然 Alice 提供了价值 $30,000 的 WETH，协议为什么不能直接借给她 $30,000 USDC？又如何判断这笔借款是否安全？**
本课继续使用 Alice 的案例，理解借贷协议最重要的三个风险控制概念。
## 一、Lesson Contract
**本课核心目标：** 理解 Lending Protocol 如何限制借款额度，并判断一个借款仓位的清算风险。
学完后，你应该能够：
- 区分 Collateral Value（抵押资产价值）与 Debt Value（债务价值）。
- 理解 LTV（Loan-to-Value，贷款价值比）的用途。
- 理解 Liquidation Threshold（清算阈值）与 LTV 的区别。
- 使用 Health Factor（健康因子）判断仓位风险。
本课不展开清算人的执行流程、清算奖励或复杂利率模型；这些留在后面的课程。
---
## 二、为什么不能按抵押品的全部价值放款？
假设 Alice 在 Aave 这类借贷协议中提供了：
```text
Collateral = 10 WETH
1 WETH = $3,000
Collateral Value = $30,000
```
如果协议允许她借出 $30,000 USDC，一旦 WETH 下跌 20%，抵押品就只值 $24,000，但债务仍约为 $30,000，甚至还会因利息增加。
这会让协议和资金提供者面临损失风险。
因此，协议必须在抵押品和债务之间保留安全空间（Safety Buffer）。
这就引出了 LTV。
## 三、LTV：最多允许借多少？
LTV 全称是 Loan-to-Value，中文是贷款价值比。
它定义借贷协议允许使用某种抵押品支持的借款比例。
假设协议给 WETH 设置：
**Maximum LTV = 70%**
那么 Alice 的最高初始借款额度为：
**$30,000 × 70% = $21,000**
这意味着：
- Alice 的抵押品价值 $30,000。
- 在这一假设下，最高初始借款额度是 $21,000。
- Alice 只借 $5,000，距离允许的初始借款上限还很远。
这里的 70% 是教学假设，不是声称 Aave 当前所有 WETH 市场都采用这一参数。真实参数由市场、协议版本、资产风险配置等因素决定。
有一点需要强调：
**Maximum Borrowing Limit 不等于 Safe Borrowing Amount。**
协议允许你最多借 $21,000，并不表示借满 $21,000 是安全的。市场价格变化可能迅速侵蚀安全空间。
## 四、Liquidation Threshold：什么时候有可能被清算？
有了 LTV，我们已经知道借款额度上限。
但这还不能直接回答另一个问题：
> WETH 下跌以后，协议什么时候可以启动清算？
这就需要第二个参数：**Liquidation Thre