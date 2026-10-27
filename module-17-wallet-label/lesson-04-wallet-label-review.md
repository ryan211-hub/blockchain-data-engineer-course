## Integrated Case

某数据平台给 0xAAA 标注为 Exchange Hot Wallet（置信度高），0xBBB 标注为 Whale（仅行为推断），0xCCC 则未知。某日 0xBBB 将 1,000 ETH 转至 0xAAA，平台显示 Exchange Inflow。第二天分析员发现 0xAAA 原来是与交易所合作的托管商地址。

## Interpretation

Transfer 是链上事实；“交易所流入”依赖标签；“抛售压力”是更进一步的行为解释。若归因错误，下游交易信号可能全部偏移。标签需要版本、证据、时间和纠错机制。

## Bank Analogy

开户识别、受益所有人识别与交易异常检测各有独立证据条件，不能把监控规则的输出当作已审定身份。

## Review Questions（无需答题）

1. 本例哪些信息是链上事实，哪些属于推断？
2. 0xAAA 标签改变后，需要重新审查哪些历史业务结论？
3. 如果 Whale Label 仅依据余额阈值，为什么它不能证明真实世界是一位富有个人？
4. 设计一个面向客户的标签置信度说明，应包含什么信息？
