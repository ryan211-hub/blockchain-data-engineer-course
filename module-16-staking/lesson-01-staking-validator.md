## Why Staking Exists

Ethereum Proof of Stake（PoS，权益证明）使用 Validator 参与共识。质押 ETH 是参加验证、提议区块与承受协议惩罚的经济抵押，并不是传统银行定期存款。Ethereum 独立运行一个验证者通常要求 32 ETH 作为有效初始质押门槛；更少资金可通过第三方池或服务参与，但增加托管或合约风险。

## Participants

Validator Operator 运营节点、维护在线率与密钥安全；Delegator / Staker（在支持该模式的产品中）提供资金；Ethereum Protocol 按有效履职和惩罚规则影响余额。Ethereum 没有简单的一般性“把 ETH 委托给任意验证者”的原生委托接口；第三方质押服务实现细节不同。

## Mechanism

用户发送 Deposit 进入质押流程，验证者通过 Attestation（投票/证明）和在被选中时 Block Proposal（区块提议）承担职责。奖励不是固定利息，取决于总质押量、在线率、区块提议机会和网络规则。

## On-chain Data Semantics

Execution Layer（执行层）的存款 Transaction 不等于验证者已经 Active；Consensus Layer（共识层）中的 Validator Status、Effective Balance、Rewards / Penalties 才能反映具体进度。银行类比：资金已提交开户申请并不等于账户已开通所有服务。

## Risks

Slashing（削减质押）可能因可罚违规而发生，离线可能受到处罚；服务商还存在运营、托管及提款流动性风险。应区分 Gross Rewards、Net Rewards、ETH 价格风险。

## Review Questions（无需答题）

1. 一个地址向质押存款合约发送 32 ETH，是否能够立即认定验证者已在出块？
2. 验证者奖励与银行存款固定年利率有何区别？
3. 为什么要分开观察 Execution Layer Deposit 和 Consensus Layer Validator 状态？
