## Why Liquid Staking Exists

直接质押 ETH 会使资产进入质押机制，而 Liquid Staking Protocol 允许用户通过合约或服务质押 ETH，并收到 Liquid Staking Token（LST，流动性质押代币）。例如 stETH、rETH 等属于不同实现，不能假设赎回和收益累积机制相同。

## Business Flow

Alice 投入 ETH → 协议安排质押 → Alice 收到相应 LST → 可持有、交易或在支持的平台用作抵押。LST 代表经济权益，但不等于一个保证始终按 1:1 价格流通的 ETH 存款凭证。

## Mechanism & Price

不同 LST 可能采用 Rebase（余额调整）或 Exchange Rate（兑换率增长）表示质押收益。市场价格受到流动性、风险预期、赎回机制和交易深度影响。<b>Redemption Value and Market Price can diverge.</b>

## Data Semantics

ERC-20 Transfer 只能说明 LST 流转；不能凭 Token Transfer 确认某个账户新质押 ETH。需要结合 Deposit/Mint、Redeem/Burn、协议汇率和验证者层状况。总 TVL（Total Value Locked，总锁仓价值）也可能包含重复计量问题。

## Risks

Smart Contract、Oracle、Liquidity、Validator / Operator 和 Governance 风险。二级市场出售 LST 可以比赎回更快，但价格可能打折。

## Review Questions（无需答题）

1. Alice 将 LST 转给 Bob，是新的 ETH Deposit 吗？
2. 为什么 LST 市价可能低于理论赎回价值？
3. Rebase 与 Exchange Rate 型 LST 对持仓数量和价值观察有什么不同？
