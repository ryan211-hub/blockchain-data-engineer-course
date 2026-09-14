# Module 8 结课综合检查｜数据建模结束标准验收

## 验收目标

验证 Module 8 的结束标准：能够根据链上分析需求确定 Business Object 与 Grain，设计 Fact / Dimension、Unique Key 与关联关系，区分 Source Identity / Business Identity，并判断 Normalized Fact / DWS 等层级职责。

## 综合题 1｜Wallet Token Analytics

业务需求：用户输入一个 Ethereum 钱包地址后，查看最近 30 天收到哪些 ERC-20 Token、每种 Token 收发总金额与次数，并显示 Token Symbol 和 Decimals。

### 用户第一次回答

1. 这个需求的 fact 应该是 transfers 。为什么不是 fact_transactions？因为 transactions 无法获取到 token 的金额，只能从 transfer 中获取
2. 这个 fact grain 的 token unique key 应该是 chain ID, wallet address, token。
3. Token symbol 和 decimals 放在 fact 或者 dimension 都可以。因为这个需求可以认为是 DWS，它有稳定的口径，还有大量重复的查询.直接放在 fact 的话，可以更简便一点
4. 会涉及一张 DWS，它的 grain、Chain ID、Wallet Address、Token 三个字段
5. 首先需要的是 transfer 事实表，用来统计 token 的转入、转出金额和次数。然后还需要 token dimension 表，获取 token 的 symbol 和 decimals。

### 老师校准

- 核心 Fact 应为 `fact_token_transfers`，根本原因是 Business Object 为 Token Transfer，而不是 Transaction。
- `fact_token_transfers` Grain = one canonical Token Transfer Event；Unique Key = `(chain_id, tx_hash, log_index)`。
- `symbol`、`decimals` 的语义归属为 `dim_tokens`；物理冗余属于 denormalization，不改变语义归属。
- `dws_wallet_token_daily_flow` 的 Grain 应包含时间：`(chain_id, date, wallet_address, token_address)`。
- 第一次回答中主要问题是把 Transfer Fact Grain 与 DWS Grain 混在了一起。

### 校准题

```text
fact_token_transfers
Grain =
Unique Key =

dws_wallet_token_daily_flow
Grain =
Unique Key =
```

### 用户校准回答

fact_token_transfers，它的 grain 是One row, one transfer. The unique keys: chain ID, TX hash, log index.
dws_wallet_token_daily_flow, grain_one_token，它的 unique key：chain ID, wallet address, token, date

### 老师判定

通过。

- `fact_token_transfers`：Grain = one Token Transfer Event；Unique Key = `(chain_id, tx_hash, log_index)`。
- `dws_wallet_token_daily_flow`：Grain = one wallet + one token + one day；Unique Key = `(chain_id, wallet_address, token_address, date)`。

固定原则：

> 同一个业务需求里可以同时存在多个不同 Grain；关键是先明确当前讨论的是哪一张表、哪个 Business Object。

## 综合题 2｜USDC 多链身份建模

业务需求：统计 Ethereum 和 Base 上 USDC 的每日 Transfer Volume，并在 Dashboard 上统一显示成 “USDC”，后台仍能区分 Ethereum USDC 和 Base USDC；进一步考虑 USDC.e。

### 用户回答

1.底层 dim_tokens 中，Ethereum USDC 和 Base USDC 应该保存两条记录。因为底层作为可追溯、可审计的数据，它需要真实保留原始数据。

2.他们各自的 source identity 是 chain_id token，虽然它们的 token 名称一样，但是它们的 chain_id 是不同的

3. Dashboard 统一逻辑应该发生在更高业务语义层
4. 如果以后又加入 USDC.e，在 Dashboard 上面可以统一显示成 USDC，但是在后台不能把他们三者当成完全同一个 token

### 老师校准

- Ethereum USDC 与 Base USDC 在 `dim_tokens` 中保存两条记录，结论正确；原因不是 `dim_tokens` 属于 Raw，而是它们具有不同 Source Identity。
- Source Identity 应完整表示为 `(chain_id, token_address)`；`chain_id` 是命名空间，`token_address` 才能具体识别 Token Contract。
- Dashboard 统一为 “USDC” 应发生在更高业务语义层，正确。
- USDC.e 可以在某些 Dashboard 语义下归入 USDC family，但后台仍必须保留独立 Source Identity；Business Identity 的统一不能抹掉链上真实身份差异。

固定原则：

> Source Identity 保留链上真实差异，Business Identity 负责业务归类；业务统一不能抹掉源身份。

## Module 8 结束判定

结合第 1～8 课学习结果与本次综合检查，Module 8 已达到 Module Contract 结束标准：

- 能从 Query Semantics 判断 Business Object；
- 能为 Transaction / Transfer / Swap / Wallet / Token 等对象确定 Grain 与 Unique Key；
- 能区分 Fact / Dimension 及其语义职责；
- 能区分 Source Identity 与 Business Identity；
- 能解释 Raw / Normalized Fact / DWS / ADS 的层级边界；
- 能识别 Mixed Grain、Join 后 Measure 重复和万能宽表问题；
- 能设计支持 Wallet / Token / Transfer / Swap 查询的最小链上数据模型。

**结论：Module 8 — 数据建模正式完成。**