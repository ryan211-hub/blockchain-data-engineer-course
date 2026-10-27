## Business Problem

Nansen、Arkham 等链上情报产品需要把链上行为转换为可理解的实体、业务意图和风险线索。常用方法是 Heuristic（启发式规则）、Clustering（地址聚类）与人工核验。

## Mechanism

例如资金多次归集到已知交易所热钱包，形成关联线索。但从地址 A 给地址 B 转账不能直接得出两个地址同属一个人；协议路由、托管、交易所充值和桥接都可能产生类似形态。

## Confidence

建议将标签区分为 Verified、High Confidence、Inferred、Unknown，并保存 Evidence Type、As-of Date、Source 和可推翻条件。规则输出是推断，而不是链上原生事实。

## Bank Analogy

银行反洗钱规则可产生可疑交易告警，但告警不等于司法认定。链上地址归因同样需要解释证据与误报率。

## Review Questions（无需答题）

1. 两个地址频繁互转能否证明同一所有者？
2. 为什么应保存 Label Confidence 和 Evidence Source？
3. 一个聚类规则在跨链桥场景可能产生什么误判？
