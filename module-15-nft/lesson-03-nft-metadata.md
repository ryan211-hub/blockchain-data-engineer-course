# 第3课｜Metadata：NFT 图片、属性和 URI 存在哪里？

第四阶段：Web3 Business Understanding | Module 15 · NFT | 第3课 | 理解检查：3题

## 一、为什么 NFT 需要 Metadata？

上一课，我们已经知道 NFT 有三个基本操作：Mint → Transfer → Burn。

但现在出现一个新问题：**区块链知道 Alice 拥有 Token ID 1001，却怎么知道这枚 NFT 对应哪张图片？**

假设 Alice 发行了一套名为 Digital Cats 的 NFT Collection（系列），包括 Digital Cat #1001、#1002、#1003 三个示意 NFT（示意图片，并非真实 NFT Collection）。

智能合约可以记录：

```text
Contract: 0xABC...

Token ID: 1001
Owner: Alice

Token ID: 1002
Owner: Bob

Token ID: 1003
Owner: Charlie
```

但是，这些信息只解决两个问题：

- Which token is it? 这是哪一枚 Token？
- Who owns it? 谁拥有这枚 Token？

没有解决：

- What does this token represent? 它代表什么？
- What does it look like? 它长什么样？

这就是 Metadata 存在的原因。

**Metadata is descriptive information associated with a token.**

Metadata（元数据）是用于描述 Token 的信息，常见内容包括名称、图片链接、描述和属性。

## 二、NFT 的三层结构：Token、Metadata、Media

这是本课最重要的概念。

```text
Layer 1 — On-chain Token
Contract Address + Token ID + Owner
               |
            tokenURI
               v
Layer 2 — Metadata (JSON)
Name · Description · Image · Attributes
               |
           image URI
               v
Layer 3 — Media File
PNG · JPEG · SVG · Video · Audio
```

### Layer 1：On-chain Token

Ethereum 上的 ERC-721 合约记录 NFT 的身份及所有权。

```text
Contract: 0xABC...
Token ID: 1001
Owner: Alice
```

很多 ERC-721 合约提供一个叫 `tokenURI` 的方法：

```text
tokenURI(1001)

→ ipfs://bafy.../1001.json
```

URI 是 **Uniform Resource Identifier（统一资源标识符）**，用于标识某个资源。URL 是 URI 中常见的一类，可以用于定位和访问网络资源。

`tokenURI` 通常不是直接返回图片，而是返回指向 Metadata 的 URI。

### Layer 2：Metadata

假设打开 `1001.json`，得到：

```json
{
  "name": "Digital Cat #1001",
  "description": "A blue cat from Digital Cats",
  "image": "ipfs://bafy.../1001.png",
  "attributes": [
    {
      "trait_type": "Color",
      "value": "Blue"
    },
    {
      "trait_type": "Rarity",
      "value": "Rare"
    }
  ]
}
```

JSON（JavaScript Object Notation）是一种结构化数据交换格式。

| Field | Business meaning |
| --- | --- |
| `name` | NFT 的展示名称 |
| `description` | 作品介绍 |
| `image` | 图片资源 URI |
| `attributes` | 特征或属性，例如颜色、稀有度 |

`attributes` 尤其重要。一套有 10,000 枚 NFT 的 Collection 可能通过帽子、眼睛、颜色等特征组合形成差异。这些属性可能影响市场对稀缺性和价格的判断，但不自动保证市场价值。

### Layer 3：Media

Metadata 中的 `image` 字段指向真正的图片文件：

```text
ipfs://bafy.../1001.png
```

**The NFT, its metadata, and its image are three different things.**

拥有 NFT，并不意味着图片文件一定存储在 Ethereum 区块链上。为了节省链上存储成本，许多项目会把图片和 Metadata 放在链下。

## 三、NFT 图片到底存在哪里？

常见三种方式：

**1. HTTP / Centralized Server**

`https://example.com/nft/1001.json`

数据由项目方或服务提供商的服务器托管。读取方便，但服务器可能停机、域名失效，内容也可能被修改。

**2. IPFS — InterPlanetary File System（星际文件系统）**

`ipfs://bafy.../1001.json`

IPFS 支持通过内容标识符 CID（Content Identifier）定位内容。内容发生改变时，通常会对应不同 CID。但需要有人持续提供和保存数据。

**IPFS does not guarantee permanent availability by itself.**

**3. On-chain Storage（链上存储）**

项目可以将 Metadata，甚至 SVG 图像等内容编码后直接保存在链上。这减少对外部服务器的依赖，但链上存储成本高，设计受到内容体积等因素限制。

| 方式 | 内容可变性 | 主要风险 |
| --- | --- | --- |
| HTTP Server | 服务器内容通常可以修改 | 服务中断、内容替换 |
| IPFS | 同一 CID 对应内容具有完整性约束 | 内容无人提供、无法获取 |
| On-chain | 取决于合约是否允许更新 | 链上成本、合约设计限制 |

即使图片保存在 IPFS，也不能直接推出 NFT 图片永远不会变化。如果合约允许修改 `tokenURI`，它仍可能从一个 CID 指向另一个 CID。

需要区分：

- **Content Integrity：** 取回的数据是否与内容标识符匹配？
- **Reference Mutability：** NFT 指向哪个内容的引用能否被修改？

## 四、为什么 Metadata 对 NFT 商业价值很重要？

假设 Alice 花 5 ETH 购买了一枚稀有 NFT，Marketplace 展示：

```text
Digital Cat #1001

Color: Gold
Rarity: Legendary

Image: Golden Cat
```

半年后，项目方关闭服务器。Alice 的钱包里仍有：

```text
Contract: 0xABC...
Token ID: 1001
Owner: Alice
```

但图片和 Metadata 无法加载。

**Protocol View：** NFT 依然有效，Alice 仍是 Owner。外部 Metadata 服务器失效不会自动使 NFT 被 Burn。

**Marketplace View：** 市场可能无法正常展示图片、属性和稀缺性，潜在买家难以评估它代表什么，可能降低交易意愿。

**Business View：** NFT 的链上所有权继续存在，但商业价值可能受到影响。

**The ownership record can survive even when the associated content becomes unavailable.**

区块链可以证明 Token 的链上状态，但不能单凭 Token 的存在保证外部图片、网站或承诺服务始终可用。

## 五、Transaction、Event 和 State 能告诉我们什么？

Alice 将 Digital Cat #1001 转给 Bob，链上出现：

```text
ERC-721 Transfer

Contract: 0xABC...
Token ID: 1001
From: Alice
To: Bob
```

这条 Event 可以证明一次符合合约逻辑的 Token 转移，但通常不能单独告诉你：

- NFT 显示什么图片。
- NFT 有哪些 Traits（特征属性）。
- Metadata 最近有没有变化。
- 图片现在是否仍可访问。

这些信息需要查看 Metadata 引用及实际内容。

Metadata 的更新不一定伴随新的 Token Transfer：

```text
Block 100:
Mint Token #1001

Block 150:
Transfer Alice → Bob

Later:
Project updates the metadata server
```

最后一步如果只是修改链下服务器内容，可能完全没有新的 Ethereum Transaction 或 Event。

**On-chain ownership history and off-chain metadata history are not the same dataset.**

不要以为掌握所有 Transfer Event 就已拥有完整的 NFT 商业信息。

## 六、本课的核心知识

| Layer | 核心问题 | 常见来源 |
| --- | --- | --- |
| Token | 谁拥有它？ | ERC-721 Contract State |
| Metadata | 它描述了什么？ | JSON / tokenURI |
| Media | 实际展示什么？ | Image / Video / Audio |

*The token establishes an on-chain identity and ownership record. Metadata describes the token, while media provides its visual or other digital content. These layers may be stored separately and may have different availability and mutability guarantees.*

不要把链上 Token 的存在、链下内容的可用性和作品版权视为同一件事。

## 七、理解检查（Business Understanding）

本课三道业务理解题，不需要写代码。

### 问题一：Token 与 Metadata

Bob 购买一个 NFT：

```text
Contract: 0xABC...
Token ID: 500
Owner: Bob

tokenURI:
https://example.com/500.json
```

后来，项目方服务器关闭，`500.json` 无法访问。

1. Bob 是否还拥有这个 NFT？
2. NFT 的图片无法显示，是否说明 NFT 已经不存在？
3. 为什么？

### 问题二：IPFS 与内容变更

NFT 项目原本使用：

```text
tokenURI = ipfs://CID_A/1001.json
```

后来项目方修改合约中的 Metadata 引用：

```text
tokenURI = ipfs://CID_B/1001.json
```

假设 CID_A 和 CID_B 对应不同 Metadata 内容。为什么使用 IPFS，并不一定意味着 NFT 展示内容永远不能改变？

### 问题三：链上 Event 与业务信息

某数据分析师已经收集了一个 NFT Collection 的全部 ERC-721 Transfer Events。

他认为：“我已经掌握全部 Transfer Event，因此完全能够还原每枚 NFT 的名称、图片、颜色和稀有度信息，不需要额外查询其他内容。”

这个判断是否正确？为什么？

请一次性回答三道题。通过后正式结束第3课。

## 第3课理解检查｜学员原始回答（2026-10-20）

问题一：

1. Bob还拥有这个 NFT
2. NFT 图片无法显示，但是 NFT 还是存在链上的
3. 因为链上的 NFT 和链下的原始图片它们其实是分离的，所以即使图片无法显示了，它也不影响链上的 NFT 继续存在。

问题二：

IPFS 能保证的是Content Addressing + Content Integrity，它并不能保证 metadata 里面的内容能否改变，这个是由链上合约来决定的

问题三：

这个判断不正确。因为掌握全部的 Transfer Event，只代表了这个 NFT 在链上的 Transfer 记录；而 NFT 的名称、图片、颜色和稀有度这些，还需要涉及到额外的链下数据，如media数据。

## 教师判断与校准（2026-10-20）

**评估结果：三道题全部独立作答并通过，达到本课 Business Understanding 标准。**

**问题一：通过。** Bob 仍持有链上 NFT。Metadata / Media 的可用性与 Token Ownership 是不同层次；服务器关闭、图片失效不自动销毁 Token。

**问题二：通过，补充细节。** IPFS 采用 Content Addressing，CID 与内容完整性相关。同一 CID 对应内容具有完整性约束，但 NFT 当前引用哪个 CID（Reference Mutability），在题设中由可修改 tokenURI 的合约实现决定。引用也可能通过链下可变机制变化。

**问题三：通过。** ERC-721 Transfer Events 能说明所有权转移历史，不能独立还原名称、图片、颜色和稀有度。名称、颜色、稀有度等来自 Metadata JSON，图片本身属于 Media；需额外读取 tokenURI、Metadata 与 Media。

## 第3课正式结课判定

Module 15 第3课《Metadata：NFT 图片、属性和 URI 存在哪里？》三道理解检查由学员独立作答并全部通过，正式结课。

下一课：Module 15 第4课《Collection：发行、稀缺性与项目价值》（待开始）。
