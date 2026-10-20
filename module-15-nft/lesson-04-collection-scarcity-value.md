# 第4课｜Collection：发行、稀缺性与项目价值

## 一、为什么 NFT 需要 Collection？

前面三课，我们关注的是单个 NFT：Token ID（识别某一枚 NFT）、Ownership（谁持有它）、Metadata（它对应什么内容）。
现在换一个角度。假设 Alice 创作了 10,000 张不同的数字猫图片，她希望将它们作为 NFT 出售。
**方案 A：** 为每张图片部署一个独立的 ERC-721 合约。
**方案 B：** 部署一个 ERC-721 合约，在其中 Mint 10,000 枚不同 Token ID 的 NFT。
大部分常见 NFT Collection 使用类似方案 B 的组织方式。
Collection 可以理解为一组在品牌、发行规则或主题上相关联的 NFT。在典型的 ERC-721 项目中，常以一个合约来代表一个 Collection。
例如 Bored Ape Yacht Club 是一个知名的 NFT Collection 案例（课堂图片为示意性展示，具体艺术品身份须以实际合约和 Token ID 核对）。
Collection 的价值不仅在于把 NFT 组织起来，更在于建立统一的项目身份、市场认知和潜在的社区或权益体系。
**A collection gives individual NFTs a shared project identity.**
Collection 并不必然等于一个智能合约。有些项目使用多个合约，Marketplace 也可能按照自己的规则对 NFT 进行分组。
## 二、Collection 是怎样发行的？

继续使用 Alice 的 Digital Cats 项目。
假设项目计划发行：

| Metric | Hypothetical value |
| --- | --- |
| Maximum Supply | 10,000 |
| Mint Price | 0.05 ETH |

假设 10,000 枚全部以 0.05 ETH 售出，忽略 Gas、折扣、免费 Mint、税费和运营成本：
**Gross Mint Revenue = 500 ETH。**
Alice 可以规定谁有资格参与 Mint、每个钱包能购买多少枚、具体价格以及发行时间。
当 Collection 首次发行时，参与者通常包括：

| Participant | Motivation |
| --- | --- |
| Project Team | 获取发行收入、建立项目或品牌 |
| Minters | 获得 NFT，可能用于收藏、权益或转售 |
| Secondary Buyers | 购买已经发行的 NFT |
| Marketplace | 提供交易发现与撮合、结算入口，并可能收取费用 |

**Primary Sale ≠ Secondary Sale**
Primary Sale（一级销售）通常指项目首次向购买者出售 NFT；Secondary Sale（二级销售）是现有持有人之间的再次买卖。
```text
Primary Sale

Alice's Project
   → Mint NFT #1001
   → Bob pays 0.05 ETH

Secondary Sale

Bob
   → Sells NFT #1001
   → Charlie pays 0.20 ETH
```

第一个场景中，项目方可能获得 0.05 ETH 的一级销售收入。
第二个场景中，0.20 ETH 通常是 Bob 出售资产的交易对价，并不全部属于项目方收入。
项目方能否获得二级交易版税、获得多少，以及是否实际收取，取决于 Marketplace 和交易机制。第5课详细讨论。
## 三、稀缺性为什么可能影响 NFT 的价格？

假设 Digital Cats 总共只有 10,000 枚 NFT。这是否意味着每枚 NFT 都很贵？
**不意味着。**
Supply Scarcity（供应稀缺性）只是价格形成中的一个条件。
**Scarcity can support a price premium, but scarcity alone does not create demand.**
课堂以三只示意数字猫比较：
- Cat #1001：Common · Blue Fur
- Cat #1002：Rare · Sunglasses
- Cat #1003：Legendary · Gold Fur
虚构的稀有度示例。Rare、Legendary 等标签不代表实际成交价格。
假设 10,000 枚 NFT 中，只有 50 枚具有 Gold Fur 属性。那么 Gold Fur 的属性出现频率为：
```text
50 / 10000 = 0.5%
```

这个特征在 Collection 内比较少见，因此可能具有更高的稀缺性。
但是：
- 如果买家非常喜欢 Gold Fur，可能愿意支付溢价。
- 如果买家并不在意这个特征，它就未必带来溢价。
- 如果整个 Collection 缺乏需求，即使最稀有的 NFT 也可能难以出售。
价值影响因素：
- **Supply：** 总数量、发行约束、是否允许增发。
- **Traits & Rarity：** 特征分布、稀有度与市场偏好。
- **Demand & Community：** 真实购买需求、品牌、文化认同。
- **Utility：** 会员权益、游戏功能或实际服务。
- **Liquidity：** 是否有买家愿意按相应价格成交。
因此，Token 数量有限，只能说明某种供给约束，不能保证其市场价值。
## 四、Floor Price 是什么？

NFT Collection 有一个常见市场指标：**Floor Price（地板价）**。
通常指某个 Marketplace 中，一个 Collection 当前有效挂牌出售的 NFT 的最低要价。
假设 Digital Cats 有以下挂牌订单：

| Token ID | Listing Price |
| --- | --- |
| #1001 | 0.8 ETH |
| #1002 | 1.0 ETH |
| #1003 | 1.2 ETH |
| #1004 | 1.5 ETH |

Collection Floor Price = **0.8 ETH**（四个假设挂牌订单中的最低价格）。
但是，Floor Price 不等于 NFT 的真实成交价格。
**Listing Price is an asking price, not an executed trade price.**
卖家提出 0.8 ETH 的挂牌价，并不代表真的有人愿意支付 0.8 ETH。
如果整个 Collection 的 Floor Price 为 0.8 ETH、总供应量为 10,000，不能把 8,000 ETH 当成实际可变现价值。
因为卖出少量 NFT 和同时卖出 10,000 枚 NFT，面临的市场需求与流动性完全不同。
这与传统金融资产的流动性风险有一定相似性，但 NFT 通常还具有更强的个体差异和市场深度限制。
## 五、Collection 如何在链上体现？

从 Protocol View 看，典型 ERC-721 合约会提供如下信息：
```text
Contract: 0xABC...
Collection Name: Digital Cats
Symbol: DCAT

Token ID: 1001
Owner: Alice

Token ID: 1002
Owner: Bob
```

但是 ERC-721 可以提供 Token Ownership、Transfer、Name、Symbol 等能力，**并没有规定所有 NFT 项目必须使用同一种 Max Supply 或 Rarity 模型**。
例如：
- Collection 最大只能发行多少枚，取决于合约规则。
- 哪些 Token 具有稀有属性，通常由 Metadata 描述。
- Floor Price 来自 Marketplace 挂牌数据。
- 实际销售价格需要成交和支付证据。
从 Data Engineer View 看，需要避免把几类信息混成同一种链上事实。

| Business Question | Relevant Evidence |
| --- | --- |
| NFT 属于哪个合约？ | Contract Address |
| 当前由谁持有？ | Contract State |
| 总共发行了多少？ | Mint / Burn 历史、Supply 机制 |
| 哪些 NFT 比较稀有？ | Metadata / Attributes |
| 当前最低挂牌价是多少？ | Marketplace Listings |
| 最近成交了多少钱？ | Sale / Settlement Records |

对于 Supply，需要区分：
- **Max Supply：** 合约或项目规则限定的最大供应量。
- **Total Minted：** 累计铸造数量。
- **Current Supply：** 当前尚未 Burn 的流通中 Token 数量（按项目定义口径）。
它们不能默认相等。
## 六、项目方和持有人分别承担什么风险？

NFT Collection 的商业利益并不总是一致。项目方可能希望通过发行获得资金、扩大社区，并发展持续业务。购买者可能关注作品收藏、权益、品牌认同，也可能预期升值。
- **Supply Risk：** 如果项目方可以继续增发，原本声称的稀缺性可能受到影响。
- **Metadata Risk：** 如果稀有属性或图片可以被修改，投资者依据这些属性形成的价值判断可能受到影响。
- **Liquidity Risk：** 即使 Floor Price 很高，也不意味着持有人可以随时按这个价格卖出。
- **Execution Risk：** 项目可能承诺未来提供会员服务、游戏功能或线下权益，但这些承诺能否落实，与 NFT 在链上是否存在是不同问题。
*An NFT project can have a technically valid smart contract while still being commercially unsuccessful. Scarcity, community, utility, and liquidity must be evaluated separately.*
这也是为什么 Blockchain Data Engineer 不能只理解合约，还要理解实际的商业活动。
## 七、理解检查（Business Understanding）

请结合本课案例，一次性回答以下三道题，不需要写代码。

### 问题一：稀缺性与价格

某 NFT Collection 总共发行 10,000 枚，其中只有 20 枚带有 Diamond 属性。
项目方宣称：
> 因为 Diamond 属性只有 20 枚，所以这些 NFT 一定比其他 NFT 贵。
你认为这个判断是否成立？为什么？

### 问题二：Floor Price 与 Liquidity

某 NFT Collection 总发行量为 5,000 枚，当前 Floor Price 为 2 ETH。
一名分析师认为：
> 这个 Collection 的市场价值就是 5,000 × 2 = 10,000 ETH。如果所有持有人同时出售，也大概能拿回 10,000 ETH。
请指出这一推论的问题。

### 问题三：Collection Supply 与链上数据

某项目声称：
```text
Maximum Supply: 10,000

Total Minted: 8,000

Total Burned: 500
```

假设所有 Mint 和 Burn 都有效，且此前没有其他影响供应量的特殊机制。
1. Current Supply 应该是多少？
2. 为什么 Current Supply 不等于 Maximum Supply？
3. 仅凭某些 NFT 的 Transfer Events，能否判断这个 Collection 的最大供应量一定是 10,000？为什么？
请回答三题后，我会按照 Business Understanding 标准评估。本课通过后进入第5课 Marketplace。
## 学员原始回答（2026-10-20）

问题一：这个判断不一定成立，因为这个稀缺性还要看是否能得到市场的认可
问题二：这一推论的问题在于，floor price 并不等于实际成交价格。而且，单枚单个 NFT 的实际成交价格，也并不能保证这全部 5000 枚 NFT 的价格都能保持不变
问题三：
1. 7500
2. Current Supply是当前的供应量，Maximum Supply是最大供应量，它们并不必须相等
3. 我觉得从 transfer events 推测不出 collection 的最大供应量，因为它最大供应量是个 state 数据，它实际发行量可以，比如发行 10 枚。如果这样的话，从 transfer 里面再怎么样也推断不出它的最大供应量
## 教师判断与校准（2026-10-20）

**三道题全部独立通过，达到本课 Business Understanding 标准。**
**问题一：通过。** Diamond 稀缺只证明属性数量少，不意味着买家愿意支付溢价。Scarcity does not automatically create demand。
**问题二：通过。** Floor Price 是有效挂牌要价，并非已执行交易价格；即便单个 NFT 成交价为 2 ETH，也无法保证 5,000 枚同时出售时维持该价格。Market Depth（市场深度）和 Price Impact（价格冲击）会限制实际可变现价值；5,000 × 2 = 10,000 ETH 不能作为实际出售收益。
**问题三：通过，校准术语。** Current Supply = 8,000 − 500 = 7,500。Maximum Supply 与 Total Minted、Current Supply 不一定相等，不能仅从 Transfer Events 推断最大供应量。Maximum Supply 不一定是独立 State 字段，可能由合约的状态变量、代码常量、发行限制逻辑定义，也可能只是未由合约强制执行的项目声明。
## 第4课正式结课判定

Module 15 第4课《Collection：发行、稀缺性与项目价值》三道理解检查均由学员独立作答并通过，正式结课。
下一课：Module 15 第5课《Marketplace：Listing、Sale、Fee 与 Royalty》（待开始）。
