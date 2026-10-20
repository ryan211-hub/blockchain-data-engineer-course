# 第2课｜Mint、Transfer、Burn：NFT 如何产生与流转

第四阶段 · 业务理解 | Lesson 2 | 理解检查：3题
## 一、一个 NFT 是怎样进入市场的？
上一课，我们区分了两个概念：
- Digital Content：图片、音乐或其他数字内容。
- NFT（Non-Fungible Token，非同质化代币）：区块链上可以识别和转移的独立 Token。
本课进一步研究：**一个 NFT 从产生、出售、转手，到最终被销毁，究竟经历了什么？**
Alice 是一位数字艺术家，创作了一幅名为 *The Blue City* 的作品。她希望将它作为 NFT 发行，并允许其他人交易。
示意作品：用一件数字艺术品说明 NFT 的生命周期。
Mint（Alice）→ Transfer（Bob）→ Burn（Destroyed）
这个过程涉及三个核心操作。
## 二、Mint：NFT 的诞生
**Minting means creating a new token on the blockchain.**
Mint 是铸造，即在智能合约中创建一个新的 NFT。
假设 Alice 使用一个符合 ERC-721 标准的 NFT 合约：
<table header-row="true">
<tr>
<td>字段</td>
<td>值</td>
</tr>
<tr>
<td>Contract Address</td>
<td>`0xABC...`</td>
</tr>
<tr>
<td>Token ID</td>
<td>`1001`</td>
</tr>
<tr>
<td>Owner</td>
<td>Alice</td>
</tr>
<tr>
<td>Token URI</td>
<td>`ipfs://.../1001.json`</td>
</tr>
</table>
在 Mint 之前，这个合约中还不存在 Token ID `1001`。
Mint 成功之后，合约记录该 Token 存在，并且归 Alice 所有。
**Mint creates a token, not necessarily the artwork itself.**
Mint 创建的是链上 Token，而不是图片文件本身。
图片可能早已存储在 IPFS 或其他存储系统中。
从 Protocol / Smart Contract 视角看，Mint 会改变链上 State，例如新增 Token Ownership 记录。
从 Data 视角看，标准 ERC-721 铸造通常会产生一条 Transfer Event：
```plain text
Transfer(
    from = 0x000...000,
    to = Alice,
    tokenId = 1001
)
```
其中 `from` 为零地址，表示 Token 的创建，而不是某个用户把已有 NFT 转给 Alice。
## 三、Transfer：NFT 的所有权转移
假设 Bob 愿意支付 1 ETH 购买 Alice 的 NFT。
交易完成后：
```plain text
Before:
Owner(1001) = Alice

After:
Owner(1001) = Bob
```
智能合约把 Token ID `1001` 的所有者从 Alice 修改为 Bob。
对应的 ERC-721 Transfer Event：
```plain text
Transfer(
    from = Alice,
    to = Bob,
    tokenId = 1001
)
```
**NFT Transfer ≠ NFT Sale**
Transfer 仅说明 Token 所有权发生了变化，不保证发生了买卖。
例如：
- Alice 出售 NFT 给 Bob。
- Alice 将 NFT 免费赠送给 Bob。
- Alice 把 NFT 转移到自己的另一个钱包。
- Alice 把 NFT 转入某个托管合约。
这些场景都可能产生 Transfer Event，但经济含义完全不同。
如果我们要分析 NFT Trading Volume（交易额），不能简单将所有 Transfer 都算作 Sale。
这正是 Blockchain Data Engineer 需要理解的业务语义。
## 四、Burn：NFT 的销毁
**Burning permanently removes a token from the circulating supply.**
假设 Bob 决定销毁自己的 NFT。
对于标准 ERC-721，Burn 通常意味着 Token ID `1001` 不再作为有效 Token 存在。
Burn 事件通常表现为：
```plain text
Transfer(
    from = Bob,
    to = 0x000...000,
    tokenId = 1001
)
```
<table header-row="true">
<tr>
<td>操作</td>
<td>From</td>
<td>To</td>
</tr>
<tr>
<td>Mint</td>
<td>Zero Address</td>
<td>Alice</td>
</tr>
<tr>
<td>Transfer</td>
<td>Alice</td>
<td>Bob</td>
</tr>
<tr>
<td>Burn</td>
<td>Bob</td>
<td>Zero Address</td>
</tr>
</table>
Burn 并不意味着历史数据消失。
The token no longer exists as an active NFT, but its transaction history remains on the blockchain.
链上的历史交易和 Event 仍然存在，相关图片文件也不一定被删除。
而且，一些 NFT 合约采用自己的销毁机制，并非所有 NFT 都能由持有人直接 Burn。
## 五、从链上数据看 NFT 的生命周期
用一个简化的 ERC-721 例子表示：
<table header-row="true">
<tr>
<td>Block</td>
<td>操作</td>
<td>Transfer</td>
</tr>
<tr>
<td>Block 100</td>
<td>Mint</td>
<td>Zero → Alice</td>
</tr>
<tr>
<td>Block 150</td>
<td>Transfer</td>
<td>Alice → Bob</td>
</tr>
<tr>
<td>Block 200</td>
<td>Burn</td>
<td>Bob → Zero</td>
</tr>
</table>
假设这些都是规范链上的真实事件，没有发生 Reorg，且我们有该 Token 的完整事件历史。
那么：
- 在 Block 120，Owner 是 Alice。
- 在 Block 180，Owner 是 Bob。
- 在 Block 220，Token 已经不存在。
**State View：What is true now?**
查询当前 State，可以得到 NFT 目前是否存在、由谁拥有。
**Event History View：What happened over time?**
分析历史 Event，可以理解 NFT 何时 Mint、经历了哪些 Transfer、何时 Burn。
这和银行系统有相似之处：账户当前余额是一种状态，而账户流水记录的是历史业务变化。
但 NFT 的历史 Event 不一定能独立证明销售价格或真实商业交易原因。分析这些问题还需要交易上下文。
## 六、局限与风险：链上所有权不等于所有权利
NFT 持有人通常拥有的是 Token 对应的链上控制权，而不是自动取得作品版权。
例如：
Bob 购买 Alice 的 NFT 后，通常可以通过控制对应钱包来转移这个 Token。
但是，他是否可以商业化使用原始图片、制作周边商品或授权别人使用，取决于项目的具体许可条款和相关法律。
**Token ownership, intellectual property rights, and commercial rights are different concepts.**
另外，NFT 可以存在，但 Metadata URL 失效，或图片文件不可访问。
NFT 仍然是一个有效 Token，却可能失去其原本关联的展示内容。
## 七、理解检查（Business Understanding）
请独立回答以下三个问题。无需写代码，也不要求设计工程架构。
**问题一：Mint 与 Transfer**
Alice 创建一件 NFT，Token ID 为 `500`。之后，她免费把这个 NFT 赠送给 Bob。
1. 这个过程中有几次 ERC-721 Transfer Event？
2. 这两个 Event 的 `from` 和 `to` 分别是什么？
3. 为什么第二个 Event 不能直接认定为 NFT Sale？
**问题二：Burn 与历史记录**
Bob 将 Token ID `500` Burn。
有人说：
“既然 NFT 已经 Burn，区块链上就不存在这个 NFT 的任何记录了，所以无法知道它以前属于谁。”
你认为这句话是否正确？为什么？
**问题三：业务分析与链上数据语义**
某 NFT Collection 一天发生了 1,000 次 Transfer。
一家数据公司在 Dashboard 上显示：
```plain text
Daily NFT Sales = 1,000
```
1. 这个统计结果可能有什么问题？
2. 如果你要统计真正的 NFT Sales，你认为还需要什么信息？
## 学员回答（原文，2026-10-20）
问题一：
1. 两次，第一次是 mint，第二次是赠送
2. 第一次的 from 是系统地址，to 是 Alice；然后第二次的 from 是 Alice，to 是 Bob
3. 因为赠送它不涉及到金钱交易。
问题二：
这句话不正确。Burn 只是把这个 NFT 销毁了，但是它历史的 transfer 记录还都留存在链上，并没有消失
问题三：
1. 这个统计结果有问题，因为 sale 和 transfer 是不同的，有 transfer 不一定代表就是 sale
2. 如果要统计真正的 sales，还需要其他的一些交易信息，比如金额，交易的双方
## 教师校准与结课判定（2026-10-20）
**评估结果：三道题全部通过，达到本课 Business Understanding 标准。**
问题一：三小问正确。第一次 `from` 并非“系统地址”，准确术语是 Zero Address（零地址）；零地址不是系统账户，不表示系统持有过 NFT。第二次为 Alice → Bob。赠送不属于 Sale；NFT 也可能由 ERC-20 支付，不能仅依赖 ETH 转账判断是否成交。
问题二：正确。Burn 不删除链上的历史 Transaction / Event，只改变当前 Token 状态。Current State 与 Historical Events 需要区分。
问题三：正确。1000 次 Transfer 不等于1000 次 Sale。要统计实际销售，需交易对手、金额以及能够证明成交的 Marketplace 事件、订单执行和支付证据。即使同一交易发生转移和付款，也不一定足以确认真实成交。
`Transfer Count ≠ Sales Count ≠ Trading Volume`
**结课判定：学员独立回答三题，经术语校准后三题通过；本课正式结课。**
下一课：Module 15 第3课《Metadata：NFT 图片、属性和 URI 存在哪里》。