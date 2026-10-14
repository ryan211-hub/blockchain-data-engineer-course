# Module 14 第 3 课｜Supplier、Borrower 与 Interest：谁支付利息，谁得到收益？

第四阶段 · Web3 业务理解｜Lending
上一课，我们理解了 LTV、Liquidation Threshold 和 Health Factor 如何控制借贷风险。
现在回到 Lending 最基本的商业问题：
**Bob 把 USDC 提供给协议，是为了赚取利息；Alice 借出 USDC，则需要支付利息。两者的利率是否相同？利率又由谁决定？**
本课继续以 Ethereum 上的 Aave V3 资金池机制为例。
## 一、Lesson Contract
本课主要掌握：
- Supplier 与 Borrower 的收益和成本关系。
- Borrow Rate（借款利率）与 Supply Rate（存款利率）的区别。
- Utilization Rate（资金利用率）为什么影响利率。
- 利息如何体现在链上的 Debt State 和 Supply Position 中。
本课不展开复杂利率曲线、复利实现算法或智能合约源码。
---
## 二、Alice 支付的利息去了哪里？
继续上一课的场景：
```text
Bob:
Supply 20,000 USDC

Alice:
Collateral = 10 WETH
Borrow = 5,000 USDC
```
Alice 借款后，需要按照协议规定的 Borrow Rate 支付利息。
Bob 则可以按照相应的 Supply Rate 获得存款收益。
但请注意，**Bob 不是直接收取 Alice 的利息**。
在 Aave V3 这种资金池模式中：
```text
Borrowers（Alice 等借款人）
        │ 借款债务随时间累积利息
        ▼
   USDC Lending Pool
   按协议规则核算利息
        │
        ├──→ Suppliers：获得存款利息
        │
        └──→ Protocol Reserve：获得规则规定的份额
```
这里展示的是经济归属，不代表每秒都有一笔实际 Token Transfer。
借款人增加的应计债务和存款人增加的应计资产权益，主要通过协议的会计状态体现。
因此可以先建立这样的理解：
**Borrower Interest 是资金成本；Supplier Interest 是资金收益；Protocol Reserve 是协议按规则分配的部分。**
## 三、为什么 Borrow Rate 不等于 Supply Rate？
我们把资金池扩大一点。
假设 USDC Pool 中有：
```text
Total Supplied = 100,000 USDC
Total Borrowed = 60,000 USDC
```
为了理解经济关系，假设：
- Borrow APR（Annual Percentage Rate，年化借款利率）：8%
- Reserve Factor（储备金比例）：10%
- 利率、资金规模全年保持不变，忽略复利及其他变化。
那么，在这个简化示例中：
| 一年期示意 | USDC |
| --- | --- |
| 借款人合计产生的利息 | 4,800 |
| 分配至协议储备的份额 | 480 |
| 分配给存款人的利息 | 4,320 |
存款人总共提供了 100,000 USDC，因此整体存款年化利率约为：
**4,320 / 100,000 = 4.32%**
而不是 8%。
如果 Bob 提供了其中的 20,000 USDC，他在这个简化条件下，一年获得的利息约为 864 USDC。
于是：
| 指标 | 教学假设 | 含义 |
| --- | --- | --- |
| Borrow APR | 8% | 借款方的年化资金成本 |
| Supply APR | 4.32% | 资金提供者的年化收益率 |
以上数字都是教学假设，不是 Aave 当前 USDC 市场利率或收益承诺。
这里的关键不是记住 4.32%，而是理解为什么两个利率不同：
**并不是所有提供给协议的资金都处于借出状态，而且一部分借款利息可能分配给协议储备。**
真实协议中的年化利率和复利收益还需要区分 APR 与 APY，不能直接把某一时点显示的利率当成未来一整年保证不变的收益。
## 四、Utilization Rate：为什么资金利用率决定利率？
现在出现一个新的问题：
为什么借款利率可能今天是 5%，过一段时间却变成 10%？
一个主要原因是资金的供给和需求发生变化。
Aave 使用 **Utilization Rate（资金利用率）** 描述资金池中有多少流动性已经被借出。
在当前简化案例中：
**U = 60,000 / 100,000 = 60%**
也就是说，大约 60% 的 USDC 资金已经被借出。
我们观察两个不同场景：
| 情况 | 利用率 | 流动性 |
| --- | --- | --- |
| 场景 A：较低利用率 | 30% | 大量 USDC 尚未被借出，资金相对充裕 |
| 场景 B：较高利用率 | 95% | 绝大部分 USDC 已借出，可用流动性很少 |
对于场景 B，协议希望：
- 抑制进一步借款。
- 鼓励现有借款人还款。
- 吸引新的 Supplier 提供资金。
- 维持一定的可用流动性，支持提款及其他协议操作。
因此，随着利用率上升，借款利率通常也会上升。
在 Aave V3 中，利率模型还存在一个 Optimal Utilization（目标利用率）附近的转折点：低于它时，利率通常较平缓地变化；超过它后，借款利率可能更快上升。
不需要记住内部曲线公式，只需要理解：
**Interest Rate is a mechanism for balancing supply and demand for liquidity.**
利率不只是一个收费数字，也是协议调节资金供需的工具。
这与银行的存贷款业务有相似之处，但 DeFi 借贷利率由协议模型及参数控制，而不是简单由银行工作人员逐笔协商。
## 五、资金利用率高，对 Bob 一定更好吗？
不一定。
如果 Utilization 上升，Supply Rate 可能提高，Bob 的预期利息收益也可能增加。
但与此同时：
**更多资金被借走，可能意味着资金池中可立即提取的 USDC 更少。**
例如，USDC Pool 供应了 100,000 USDC，其中 95,000 USDC 已被借出。
在不考虑其他资金流动时，池中只有约 5,000 USDC 可用。
如果 Bob 希望一次性提取 20,000 USDC，即使他的供应仓位账面价值已经达到 20,000 USDC 以上，也不代表池中立刻有足够的可用 USDC 支持全部提款。
因此要区分：
- **Supply Position Value**：Bob 在协议中的资产权益价值。
- **Available Liquidity**：当前可用于借款或提款的流动性。
这也说明较高利率不代表没有风险。
## 六、从链上数据看利息：为什么不一定每天有一条 Interest Event？
回到 Alice 的 Borrow Position：
```text
Day 1:
Borrow 5,000 USDC

Day 30:
没有新的 Borrow Event
```
但 Alice 的债务仍可能高于 5,000 USDC。
原因是利息随着时间累积。
在 Aave V3 中，借款人的债务可以通过 Variable Debt Token（可变利率债务代币）及相关借款指数体现；供应方的资产权益则由 aToken 等协议资产表示。
不必把它理解成每天都有一笔 Alice 向 Bob 支付利息的链上转账。
从 Blockchain Data Engineer 的视角，我们要区分：
| 对象 | 业务语义 |
| --- | --- |
| `Borrow Event` | 某次新增借款的历史动作 |
| `Current Debt` | 包含应计利息及后续操作影响的当前债务 |
| `Supply Position` | 存款人的资产权益及累计收益 |
| `Borrow Rate / Supply Rate` | 某时点或某期间对应的利率状态 |
| `Utilization Rate` | 某资金池的资金使用状况 |
这也是为什么分析 Lending 业务时，不能仅靠累计 Borrow Event 的 Amount，就准确计算某一时点的未偿债务与资金提供者收益。
---
## 七、本课总结
从 Bob 与 Alice 的业务案例，我们建立了四个认识：
1. Borrower 支付资金成本，Supplier 获得资金收益，但他们通常不是一对一的借贷关系。
2. Borrow Rate 与 Supply Rate 不同，因为资金并非全部借出，协议还可能分配部分利息到储备。
3. Utilization Rate 影响借贷利率和可用流动性，是协议平衡资金供给与需求的重要机制。
4. 利息累积会改变 Debt State 与 Supply Position，不必每次变化都对应一条独立的 Interest Event。
## 八、理解检查
请仍然使用 Bob 和 Alice 的业务案例回答下面三题。
**问题一：为什么存款利率低于借款利率？**
假设 USDC Pool 的 Borrow APR 为 8%，但 Bob 的 Supply APR 只有约 4%。
为什么不能认为协议一定把另外 4% 全部拿走了？
**问题二：资金利用率升高意味着什么？**
如果 USDC Pool 的 Utilization 从 40% 上升到 95%，对新的 Borrower、现有 Supplier，以及资金池可用流动性，分别可能产生什么影响？
**问题三：链上利息的业务语义**
Alice 30 天前借入了 5,000 USDC，之后没有新的 Borrow Event，也没有 Repay Event。
她当前的 Debt 是否一定仍是 5,000 USDC？为什么？
**可以三题一起回答，也可以逐题提交。**
## 协议参考
- [Aave Supply Tokens](https://aave.com/help/supplying/supply-tokens)
- [Aave Borrow Tokens](https://aave.com/help/borrowing/borrow-tokens)
- [Aave V3 Interest Rate Strategy](https://www.aave.com/docs/aave-v3/smart-contracts/interest-rate-strategy)
课程中的利率和资金规模均为教学假设。

## 学员回答（第3课理解检查｜问题一至三）
问题一： 因为这里还涉及到资金利用率的问题，以及资金的储备金。
问题二：资金的利用率升高，对新的Borrower，资金利率升高，资金的使用成本增加。对Supplier来说，利息收益提高，但是不能提现的风险增大。资金池可用流动性降低，因为可使用的资金减少了。
问题三：不一定，因为他当前的 debt 还涉及到借款利息的累加
## 教师判断与校准（第3课）
**判定：三道题全部通过。** 已掌握 Lending 中利息分配、资金利用率和债务累积的基本业务逻辑。
### 问题一：为什么 Supply APR 低于 Borrow APR？
**正确。** 学员指出资金利用率和储备金比例两个关键因素。
在本课教学假设下：
```text
Borrow APR × Utilization Rate × (1 − Reserve Factor)
= 8% × 60% × 90%
= 4.32% Supply APR
```
Borrow Rate 为 8%，但只有约 60% 的资金被借出、产生借款利息；协议再将其中 10% 的借款利息划入储备。因此 Borrow APR 与 Supply APR 的差值并不全是协议收入。
### 问题二：资金利用率从 40% 上升到 95%
**正确，分别覆盖了 Borrower、Supplier 和 Lending Pool。**
| 观察对象 | 学员判断 | 教师校准 |
| --- | --- | --- |
| 新 Borrower | 借款利率升高、使用成本增加 | 正确，通常如此 |
| Supplier | 利息收益提高 | 正确，但不是保证收益 |
| Lending Pool | 可用流动性减少 | 正确 |
学员提出的提款风险成立，但应更精确表述为：**可能无法立即全额提款，不等于永久无法提款。** 借款人还款或新的资金进入后，流动性可能恢复。
### 问题三：没有新借款或还款 Event，债务是否固定？
**正确。** 即使 Alice 没有新的 Borrow Event 或 Repay Event，借款利息仍可能累积，使 Current Debt 高于最初的 5,000 USDC。
**Borrow Event 是历史借款动作；Current Debt 是包括应计利息的当前债务。**
## 第3课结课判定
**三道理解检查全部通过，Module 14 第3课正式结课，不需要补考。**
学员已能够解释资金池中 Borrow APR 与 Supply APR 的差异、利用率升高对参与者和流动性的影响，以及历史借款动作与当前债务之间的时间语义。符合第四阶段业务理解标准，工程实践后置。
**下一课（待开始）：Module 14 第4课｜Liquidation：抵押品价格下跌后，谁承担风险？**
