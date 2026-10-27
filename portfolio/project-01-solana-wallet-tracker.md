## Objective

给定 Solana Wallet Address，展示 SOL 与 SPL Token 余额、历史转账及时间线。该项目保留在课程原始 Portfolio 列表中，作为 Solana 对照实践；EVM 仍是就业方向的重点。

## Requirements

- 调用可靠 Solana RPC，区分 Native SOL 与 Token Accounts。
- 抓取历史签名与交易详情，解析 Token Balance Changes 和可识别转账。
- 支持分页、速率限制、增量同步、重试以及重复数据去重。
- 公开地址不等于可识别个人；数据只是链上事实。

## Architecture

```txt
Solana RPC → Fetcher → Parser → Normalized Transfers
           → Postgres → API / CLI
```

## Deliverables

README、配置样例、可复现运行脚本、表结构、单元测试、示例地址的查询结果，以及已知限制。不要提交真实 API Key、助记词或私钥。

## Acceptance Criteria（教材中保留；发布不等于通过）

- 相同区间重跑结果保持幂等。
- 断点中断后可恢复，不漏抓与重复计数。
- 指定样本与 Explorer 可核对，明确 Balance 与 Flow 的差异。
- 日志能够解释错误与 RPC 失败。

## Review Questions（无需答题）

1. 为什么一个 Wallet 可能持有多个 Token Account？
2. 如何验证增量抓取没有遗漏？
3. 为什么不能仅用 Signature 数量作为 Transfer Count？
