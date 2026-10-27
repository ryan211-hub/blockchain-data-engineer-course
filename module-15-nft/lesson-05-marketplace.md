## Business Problem

NFT Token 可以在钱包之间 Transfer，但陌生买卖双方还需要发现价格、验证订单、完成资产与支付的交换。Marketplace 提供 Listing（挂牌）、Order Matching / Execution（订单执行）和 Settlement（结算）等服务。

<b>A listing is an offer, not a completed sale.</b> 例如 Alice 以 2 ETH 挂牌 NFT #1001，即使链上没有转账，市场也可能显示此挂牌；只有订单完成成交才可以计入 Sales。

## Participants and Incentives

卖家希望出售 NFT；买家希望按可接受的价格获得 Token；平台可能收取 Marketplace Fee；创作者可能收到 Royalty（版税），但二级版税是否被实际执行取决于市场和订单机制，并非 ERC-721 Transfer 自动保证。

## Scenario and Settlement

假设 Bob 以 1 ETH 从 Alice 购买 #1001，平台收费 2.5%，创作者版税为 5%，均从该 1 ETH 中扣除、没有其他费用，则：

- Buyer pays: 1.00 ETH
- Marketplace Fee: 0.025 ETH
- Creator Royalty: 0.05 ETH
- Seller Net Proceeds: 0.925 ETH

这只是清晰定义费用口径后的示例。实际订单可能涉及 ERC-20 支付、多个 NFT 批量成交、聚合器、零费用或可选择版税。

## On-chain Data Semantics

ERC-721 <code>Transfer(Alice, Bob, tokenId)</code> 证明 Token 转移，但不足以说明 1 ETH 价格。成交需要 Marketplace 执行事件、订单履约记录、资金流及关联交易上下文。Off-chain Listing 可能不在链上形成事件。

区分：Listing Count、Sale Count、Volume（成交总额）、Fee Revenue（平台费用）、Seller Proceeds（卖方净收益）。同一 Tx 内可能有多个 NFT Transfer，不能默认一笔 Tx 等于一笔 NFT Sale。

## Limitations and Risks

Wash Trading（刷量交易）可能制造虚假成交额；Floor Price 是挂牌价格，不是即时可变现价格；Royalty 有执行不确定性；订单也可能撤销、过期或因所有权变化而失效。

## Key Takeaways

<b>Transfer ≠ Sale; Asking Price ≠ Execution Price; Sale Price ≠ Seller Proceeds.</b>

## Review Questions（不要求提交答案）

1. 一个 NFT 在 Marketplace 挂牌 3 ETH，但一周未成交，可以记录 3 ETH Trading Volume 吗？为什么？
2. 一笔交易同时转移两枚 NFT、总支付 4 ETH，能否仅凭 ERC-721 Transfer Events 得到每枚成交价？
3. 假设 1 ETH 成交、平台费 2%、版税 3%，两种费用都从总价扣除，卖方净得多少？
