# Module 15 — NFT

## Module Contract（第四阶段业务理解优先）

### 所属阶段与目标

第四阶段 Web3 Business Understanding。根据 Notion「课程总目录」，本模块的必备主题是 NFT、Metadata、Collection、Marketplace。以下六课为本阶段拟定的教学安排，非原始总目录已有的逐课课程，不要求设计生产级 NFT 数据平台。

### 必须掌握

- NFT（Non-Fungible Token）为何出现、Token 的非同质化身份和转移如何建立可验证的数字资产记录。
- 区分 Token Ownership、Metadata / Media 与 Intellectual Property Rights 或项目实际授权；图片能复制，并不代表 Token 相同。
- ERC-721 与 ERC-1155 的高层业务差异；Mint、Transfer、Burn 与当前 Owner 的关系。
- Metadata、tokenURI、Collection、Token ID 之间的关系，以及链上身份和链下内容的边界。
- NFT Marketplace 中 Listing、Sale、Transfer、成交价、交易费用与 Royalty 的业务语义和风险。
- 不能仅由 NFT Transfer 推断实际 Sale；交易事件、价格、订单和持有者状态须区分。

### 可以了解

- Metadata 的 HTTP、IPFS（InterPlanetary File System）及 On-chain 数据方式与更新性差异。
- NFT 会员、游戏物品、数字艺术品等使用场景；Token 的持有不自动给予作品版权或服务履约保证。
- Collection Floor Price、成交量、挂牌量、版税等市场指标的解释边界。

### 本 Module 不展开

- NFT Indexer、ETL、Streaming、Warehouse、实时数据接口的生产实现。
- Marketplace 订单合约内核、复杂签名与结算函数、NFT 图片托管系统设计。
- 第五阶段 Portfolio 项目实践和工程验收。

### 结束标准

能够解释 NFT 为什么存在、如何发行与流转、为什么有人购买、谁获得收益并承担风险，以及链上 Transfer、Metadata、Collection、Marketplace 记录各自能与不能说明什么。最终采用 3～5 道不重复单课原题的 Business Understanding Review，业务理解已完成后工程实现后置。

## 课程结构（依据 Module 15 总目录方向拟定）

1. 第1课｜为什么需要 NFT：拥有一张图片和拥有一个 Token 有什么不同？
2. 第2课｜Mint、Transfer、Burn：NFT 如何产生与流转
3. 第3课｜Metadata：NFT 图片、属性和 URI 存在哪里
4. 第4课｜Collection：发行、稀缺性与项目价值
5. 第5课｜Marketplace：Listing、Sale、Fee 与 Royalty
6. 第6课｜NFT Business Understanding Review（不重复的业务综合检查）

## 当前学习进度

- 当前阶段：第四阶段 Protocol / Web3 业务理解
- 当前 Module：Module 15 — NFT
- 已完成 Lesson：第1～2课（均三题独立回答并通过，正式结课）
- 当前 Lesson：第3课｜Metadata：NFT 图片、属性和 URI 存在哪里（待开始）
- 当前状态：第1课三道理解检查独立回答并全部通过，正式结课；第2课三题独立回答、校准后通过，正式结课。
- 下一步准确入口：第3课《Metadata：NFT 图片、属性和 URI 存在哪里》（理解检查中），继续第四阶段业务理解。

## 课程路径

- [第1课｜为什么需要 NFT：拥有一张图片和拥有一个 Token 有什么不同？](lesson-01-why-nft-token-ownership-vs-image.md)
- [第2课｜Mint、Transfer、Burn：NFT 如何产生与流转](lesson-02-mint-transfer-burn.md)
- [第3课｜Metadata：NFT 图片、属性和 URI 存在哪里](lesson-03-nft-metadata.md)
- 第4课｜Collection：发行、稀缺性与项目价值（待创建）
- 第5课｜Marketplace：Listing、Sale、Fee 与 Royalty（待创建）
- 第6课｜NFT Business Understanding Review（待创建）
