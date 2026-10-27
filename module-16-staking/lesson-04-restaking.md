## Business Problem

Restaking（再质押）希望利用已有的质押资产或其代表权益，为额外服务提供经济安全，并向参与者提供额外奖励机会。安全共享不是收益保证。

## Participants

Restaker、Operator、Actively Validated Service（AVS，主动验证服务）或类似额外安全服务、协议治理参与者。不同项目的实际权限、可罚规则和资产可用性差异很大。

## Example

Alice 在原本 Ethereum 质押风险之外，又同意让同一经济抵押承担某服务的额外义务。额外收益来自履行新义务的报酬，但也引入额外运营失败、合约、权限和潜在惩罚风险。

## Distinguish Layers

Base Layer Validator 的职能与 Restaking Service 的职能并不相同；一个 ETH 经济权益被多个协议引用，不意味着产生多份独立本金。

## Data Semantics

统计 TVL 时，如果 1 ETH 先进入质押、换取 LST 再作为 Restaking 资产使用，简单把三处展示的面值相加可能产生 Double Counting（重复计量）。应解释资产路径、对手风险和净敞口，而不是只报告协议名义 TVL 总和。

## Risks

Slashing / Penalty 机制依据协议而异，部分可能尚未启用或附有生效条件，不能推断所有 Restaking 产品具有相同损失机制。收益越高不代表风险调整后回报越好。

## Review Questions（无需答题）

1. 一份质押本金被多个安全服务使用，是否意味着用户拥有多份本金？
2. 计算 Restaking 总 TVL 时最需要警惕什么？
3. Restaking 的额外奖励对应哪些新增的商业或技术风险？
