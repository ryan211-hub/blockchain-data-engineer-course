## Business Problem

参与验证者运营为什么能够获得收益，又为什么需要惩罚机制？PoS 需要激励参与者按规则提供安全性和可用性。

## Rewards & Penalties

Attestation、Sync Committee 和 Block Proposal 等职责可能带来奖励；未及时履职可能发生 Missed Reward 或 Penalty。Slashing 是针对可处罚行为的额外严厉惩罚，不等同于日常暂时离线。

<b>Rewards are conditional protocol incentives, not guaranteed yield.</b>

## Validator Lifecycle

Deposit → Pending → Active → Exiting → Withdrawable。这个过程涉及激活与退出队列，状态变化不必与单笔执行层交易一一对应。提款和退出也不同：部分余额可提取不一定代表验证者已经退出。运营方还需要关注有效余额、提款凭证与适用的协议规则。

## Case

某运营商报告本月验证收益为 1 ETH，但支付服务费 0.1 ETH、经历处罚 0.05 ETH，那么不含本金价格波动的相关净额为 0.85 ETH。只有明确时间范围和费用口径，才能比较运营绩效。

## Evidence and Limitations

仅观察钱包 ETH 余额变动，可能遗漏共识层奖励与处罚；只观察宣传 APR（Annual Percentage Rate，年化收益率）不能证明实际投资者收益。应区分 Validator-level 指标与 Investor-level 净收益。

## Review Questions（无需答题）

1. Missed Attestation 与 Slashing 在业务风险上有什么不同？
2. 为什么 Validator Exit 不等于用户能立即收到全部 ETH？
3. 同一 APR 下，运营费与处罚如何改变 Staker 的净回报？
