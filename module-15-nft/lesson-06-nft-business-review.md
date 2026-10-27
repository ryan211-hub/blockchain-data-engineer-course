## 综合业务案例

项目 Blue Pass 发行 2,000 枚 ERC-721 会员 NFT，定价 0.05 ETH。发行后，Marketplace 上的最低有效挂牌价为 0.12 ETH，过去一天发生 300 次 Transfer，其中平台经关联成交事件确认 80 笔 Sale。项目将图片保存在中心化服务器，Metadata 包含会员等级。会员权益取决于项目实际服务规则。

## Business Understanding

NFT 的链上身份由 Contract Address + Token ID 表达；当前 Owner 由合约状态表示。Metadata 和 Media 可能位于链下，其可用性不是链上所有权的保证。初级发行收入与二级交易流量不同；Marketplace 的挂牌与成交不同；高 Floor Price 不意味着有足够买盘接下全部供应。

## Evidence Boundary

- 300 Transfers：链上 Token 所有权变化的次数，不意味着 300 次销售。
- 80 Confirmed Sales：需说明平台覆盖范围、订单与结算口径，并防范刷量。
- 0.12 ETH Floor：最低挂牌要价而非成交均价。
- 会员级别：需要 Metadata 和服务协议，不能只看 Transfer。
- 总供应量：项目声明需要核查合约约束及 Mint/Burn 记录。

## Risks

服务器关闭时 Token 仍可能存在但图片无法展示；项目未兑现权益时链上 Token 也不会自动消失；合约若允许改变 Metadata 或新增 Mint，商业稀缺性可能变化。

## Review Questions（不要求提交答案）

1. Blue Pass 的 300 次 Transfer 与确认的 80 笔 Sale，应该如何在 Dashboard 分别命名？还缺少哪些覆盖范围说明？
2. 若项目方关闭图片服务器且会员系统停服，NFT 所有权、可展示性与权益履行分别怎样变化？
3. Collection 有 2,000 枚 NFT、Floor 0.12 ETH。有人宣称项目至少可立即变现 240 ETH，应如何评价？
4. 若 NFT 卖方收到 0.90 ETH，但买方支付 1 ETH，应如何调查差额是 Fee、Royalty 还是其他结算安排？
