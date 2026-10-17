# Module 14 — Lending

## Module Contract（第四阶段业务理解优先）

### 所属阶段与目标
第四阶段 Web3 Business Understanding。根据 Notion「课程总目录」，本模块学习 Lending，包括 Aave、Kamino 这类借贷业务为什么存在、参与者为什么使用和如何获利。本 Module 的具体 Lesson 结构为现阶段依据「AI提示词」第14条拟定的教学安排，不声称是历史原目录已有的详细课次。

### 必须掌握
- 为什么用户愿意抵押某种 Token 来借入另一种 Token，而不是直接 Swap。
- Supplier / Depositor、Borrower、Lending Pool、Collateral、Debt、利息的业务关系。
- Overcollateralization、LTV（Loan-to-Value，贷款价值比）、Borrowing Limit、Health Factor 的基本业务含义。
- 利率与资金利用率如何影响 Supplier 和 Borrower 的行为及收益。
- Liquidation 的业务动机、谁承担损失、清算参与者为何愿意参与。
- Supply、Borrow、Repay、Withdraw、Liquidation 相关链上事件及持续变化的 Position / State 的区别。
- Aave（Ethereum / EVM）与 Kamino（Solana）的基本业务定位和跨生态理解边界。

### 可以了解
- Variable Borrow Rate / Stable Rate 的历史或产品层差异（具体支持情况以协议版本为准）。
- Oracle Price、资产风险参数、清算奖励的高层作用。
- Protocol Reserve、治理参数与收入归属的基本机制。

### 本 Module 不展开
- 生产级 Lending Indexer、ETL / Streaming / Warehouse 构建。
- 复杂利率曲线公式与代币的内部计息代码。
- 清算 Bot、MEV、Oracle 合约内核及高级风险建模。
- 第五阶段 Portfolio 项目验收。

### 结束标准
能够解释 Why does it exist / How does it work / Who benefits and who takes risks / What does its on-chain data mean，并运用这些概念判断新的 Lending 业务场景。最后一课采用 3～5 道不重复单课考点的轻量 Business Understanding Review。

## 课程结构（本阶段教学安排）
1. 第1课｜为什么需要 DeFi Lending：不卖 ETH，也能借到 USDC？
2. 第2课｜Collateral、LTV 与 Health Factor：借多少才安全？
3. 第3课｜Supplier、Borrower 与 Interest：谁支付利息，谁得到收益？
4. 第4课｜Liquidation：抵押品价格下跌后，谁承担风险？
5. 第5课｜Aave / Kamino 的借贷业务与链上数据语义
6. 第6课｜Lending Business Understanding Review（不重复的业务综合检查）

## 当前学习进度
- 当前阶段：第四阶段 Protocol / Web3 业务理解
- 当前 Module：Module 14 — Lending
- 已完成 Lesson：第1课｜为什么需要 DeFi Lending：不卖 ETH，也能借到 USDC？（理解检查三题全部通过）；第2课｜Collateral、LTV 与 Health Factor：借多少才安全？（三题通过）；第3课｜Supplier、Borrower 与 Interest：谁支付利息，谁得到收益？（理解检查三题通过）；第4课｜Liquidation：抵押品价格下跌后，谁承担风险？（问题一补充后通过，问题二与三通过）；第5课｜Aave / Kamino 的借贷业务与链上数据语义（学员确认课程通过；教师示范解答，非独立考核通过）
- 当前 Lesson：第6课｜Lending Business Understanding Review（待开始）
- 当前状态：第5课已通过（学员明确确认业务理解课程通过）；三道理解题由教师示范解答，没有学员独立作答，不能记为三题独立考核通过。第6课综合检查待开始。
- 下一步准确入口：学员请求下一课时开始第6课《Lending Business Understanding Review》，设置不重复前5课的3～5道业务综合新题。
 
## 课程路径

- [第1课｜为什么需要 DeFi Lending：不卖 ETH，也能借到 USDC？](lesson-01-why-defi-lending-borrow-usdc-without-selling-eth.md)
- [第2课｜Collateral、LTV 与 Health Factor：借多少才安全？](lesson-02-collateral-ltv-health-factor.md)
- [第3课｜Supplier、Borrower 与 Interest：谁支付利息，谁得到收益？](lesson-03-supplier-borrower-interest.md)
- [第4课｜Liquidation：抵押品价格下跌后，谁承担风险？](lesson-04-liquidation-risks-and-incentives.md)
- [第5课｜Aave / Kamino 的借贷业务与链上数据语义](lesson-05-aave-kamino-lending-data-semantics.md)
- 第6课｜Lending Business Understanding Review（待创建）
