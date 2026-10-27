## Why Wallet Labels Exist

Ethereum 地址是一段由密钥推导的标识，并非银行开户时实名登记的账户。Block Explorer 能显示交易，却通常不能直接显示地址背后是谁。Wallet Label 将公开链上地址与某种归属或行为类别联系起来。

## Three Distinct Objects

Address：单个链上地址；Entity：交易所、协议、个人或组织等推测/确认的运营主体；Label：例如 Exchange Deposit、DEX Trader、Smart Money 等业务分类。一个 Entity 可能管理许多地址；一个地址也可能出现多个用途。

## Evidence

项目官方网站公布地址、智能合约源码验证、已知存取款模式、地址集群分析、公开声明等，证据强度不同。<b>Observed on-chain activity is not identical to verified real-world identity.</b>

## Scenario

10 个地址都向同一交易所充值钱包转 ETH，不能直接推断这 10 个地址都由交易所控制。Deposit Address、Hot Wallet、用户钱包和 Protocol Contract 应分别处理。

## Review Questions（无需答题）

1. 为什么链上地址不能天然等同于实名客户？
2. 同一 Entity 下多个地址如何影响统计口径？
3. “疑似交易所地址”与“官方确认的交易所钱包”有什么证据区别？
