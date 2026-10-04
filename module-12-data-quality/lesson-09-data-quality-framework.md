# Module 12 第 9 课｜Data Quality Framework：检测、告警、阻断、修复、审计

## Lesson Contract

所属 Module：Module 12 — 数据质量

本课核心问题：

> How do we turn isolated data-quality checks into an operational Data Quality Framework that can detect, contain, repair, verify, and audit problems?

学完以后，你应该能够解释并设计：

- DQ（Data Quality，数据质量）Framework 的核心组成。
- 为什么 Data Quality 不能只是几条 SQL。
- Rule、Metric、Check、Incident、Repair Job 的区别。
- 如何把 Completeness / Accuracy / Consistency / Freshness / Uniqueness / Validity 落成可执行检查。
- 什么是 Severity、Threshold、Blocking Rule。
- 为什么有些 DQ Failure 必须阻止 Checkpoint，有些不需要。
- Alert 和 Incident 的区别。
- Quarantine / Dead-letter 的高层用途。
- 如何设计 Detect → Contain → Diagnose → Repair → Verify → Resume。
- 如何记录 Data Quality Audit Trail。
- 如何把 Reorg、Backfill、Replay、Multi-sink Repair 统一进同一个 Framework。
- 如何避免“告警很多，但没人知道下一步该做什么”。

本课暂不展开：

- 商业 Data Observability 平台采购；
- ML（Machine Learning，机器学习）异常检测；
- Data Catalog / Governance 全体系；
- 全自动 Incident Response；
- 大规模 Rule Engine 实现；
- 分布式一致性协议。

---

## 一、为什么需要 Data Quality Framework？

前面几课你已经学过很多独立能力：

```text
Completeness Check
Uniqueness Check
Accuracy Check
Reconciliation
Freshness Monitoring
Reorg Detection
Historical Repair
```

如果这些能力只是散落在不同脚本里：

```text
check_missing_block.py
check_duplicate.sql
repair_decoder.py
reorg_fix.py
```

系统仍然很难管理。

真正的问题是：

> How do these checks work together as one operational system?

这就是 Framework 要解决的问题。

---

## 二、Framework 不是“多写几条 SQL”

一个成熟的 DQ Framework 至少需要覆盖：

```text
Detect
Classify
Alert
Block / Contain
Diagnose
Repair
Verify
Audit
```

也就是说：

> Detection is only the beginning.

如果系统只能发现错误，却不知道是否要阻断、如何修复、如何验证，那么它还不是完整 Framework。

---

## 三、[Architecture 视角] Data Quality Framework 的五层结构

可以先建立一个总模型：

```text
1. Rule Layer
2. Execution Layer
3. Decision Layer
4. Repair Layer
5. Audit / Observability Layer
```

分别回答：

```text
What should be true?
Did it pass?
How serious is failure?
How do we fix it?
What happened historically?
```

---

## 四、Rule 和 Check 有什么区别？

### Rule

定义：

> What should be true?

例如：

```text
No duplicate source identity
```

或者：

```text
Canonical blocks must be continuous
```

### Check

是一次具体执行：

```text
run rule X
on block range 21,000,000 → 21,000,100
at 12:00
result = FAIL
```

所以：

```text
Rule
→ definition

Check Run
→ execution instance
```

---

## 五、Metric 又是什么？

Metric 是：

> A measurable value produced by observation.

例如：

```text
missing_block_count = 0
duplicate_key_count = 12
block_lag = 800
reconciliation_diff = 0.002
```

Rule 可以基于 Metric 做判断：

```text
duplicate_key_count == 0
→ PASS

duplicate_key_count > 0
→ FAIL
```

因此：

```text
Metric
↓
Rule Evaluation
↓
Check Result
```

---

## 六、一个基础 DQ Rule 应该包含什么？

可以设计成：

```text
rule_id
rule_name
quality_dimension
target_dataset
grain
metric
threshold
severity
blocking_policy
owner
repair_strategy
```

例如：

```text
rule_name = no_duplicate_transfer_key
quality_dimension = Uniqueness
target_dataset = fact_token_transfer
grain = one Transfer
metric = duplicate_key_count
threshold = 0
severity = HIGH
blocking_policy = BLOCK_CHECKPOINT
```

---

## 七、Quality Dimension 要显式记录

你已经学过：

- Completeness
- Accuracy
- Consistency
- Freshness
- Uniqueness
- Validity

Framework 里不要只存：

```text
rule_name
```

最好也记录：

```text
quality_dimension
```

因为这有助于后续回答：

> Which type of data quality is failing most often?

---

## 八、Completeness Rule 示例

例如：

```text
Expected block range:
21,000,000 → 21,000,100
```

检查：

```text
missing_block_count
```

规则：

```text
missing_block_count = 0
```

如果：

```text
missing_block_count = 2
```

则：

```text
Completeness = FAIL
```

---

## 九、Uniqueness Rule 示例

Transfer Fact：

```text
Unique Key:
chain_id + tx_hash + log_index
```

Metric：

```text
duplicate_key_count
```

规则：

```text
duplicate_key_count = 0
```

这属于：

> Preventive Constraint + Detective Check

数据库唯一约束负责 prevention，DQ Job 负责 detection / audit。

---

## 十、Accuracy Rule 为什么更难？

Accuracy 往往不能只靠结构检查。

例如：

```text
amount IS NOT NULL
```

只能证明：

```text
Validity
```

不能证明：

```text
amount is correct
```

Accuracy 往往需要：

```text
Golden Sample
Reference Dataset
Source-vs-Derived Check
Invariant
Cross-source Comparison
```

所以：

> Schema-valid data can still be inaccurate.

---

## 十一、Consistency Rule 示例

Postgres 与 ClickHouse：

```text
same processing boundary
same business scope
```

然后比较：

```text
count
sum(amount)
key set
```

如果不一致：

```text
Consistency = FAIL
```

注意：

> Boundary alignment is part of the rule definition.

否则会产生 false positive。

---

## 十二、Freshness Rule 示例

例如：

```text
Freshness SLO
block_lag <= 100
```

这里 SLO（Service Level Objective，服务水平目标）首次出现。

如果：

```text
block_lag = 40
→ PASS
```

如果：

```text
block_lag = 800
→ FAIL
```

但这个 FAIL 不一定意味着 Checkpoint 要停止。

---

## 十三、Severity 是什么？

Severity 表示：

> How serious is this failure?

一个简单分级：

```text
INFO
LOW
MEDIUM
HIGH
CRITICAL
```

例如：

```text
Freshness lag 120 blocks
→ MEDIUM

Duplicate canonical facts
→ HIGH

Canonical chain continuity broken
→ CRITICAL
```

---

## 十四、Severity 和 Blocking Policy 不是一回事

这是一个重要区分。

```text
Severity
→ business / operational impact

Blocking Policy
→ should processing continue?
```

例如：

```text
Freshness SLO FAIL
Severity = HIGH
Blocking = NO
```

因为 Pipeline 仍然在正确处理，只是太慢。

而：

```text
Reconciliation FAIL
Severity = HIGH
Blocking = YES
```

因为当前 Processing Range 不可信。

---

## 十五、什么时候应该 Block Checkpoint？

一般来说，如果问题说明：

> The current processing range is not trustworthy.

那么应该：

```text
BLOCK_CHECKPOINT
```

典型情况：

```text
Validation FAIL
Missing required source data
Duplicate fact violating identity
Reconciliation FAIL
Canonical continuity FAIL
Decoder accuracy validation FAIL
```

---

## 十六、什么时候不一定 Block？

例如：

```text
Freshness degraded
but processed data is correct
```

可以：

```text
checkpoint continues
alert fires
incident opens
```

所以：

> Not every DQ failure is a processing-blocking failure.

---

## 十七、Decision Layer 做什么？

Decision Layer 把：

```text
Check Result
+
Severity
+
Blocking Policy
```

转化为动作。

例如：

```text
PASS
→ continue

FAIL + WARN_ONLY
→ continue + alert

FAIL + BLOCK_CHECKPOINT
→ stop checkpoint + incident

FAIL + QUARANTINE
→ isolate bad records
```

---

## 十八、什么是 Quarantine？

Quarantine 可以理解为：

> Isolate suspicious data instead of letting it enter trusted downstream datasets.

例如 Decoder 遇到无法识别的 Event：

```text
raw event
↓
decoder failed semantic validation
↓
quarantine table
```

而不是：

```text
silently drop
```

也不是：

```text
write bad fact
```

---

## 十九、Dead-letter 是什么？

DLQ（Dead Letter Queue，死信队列）通常用于：

> Messages or records that repeatedly fail processing and need separate handling.

例如：

```text
Kafka message
→ decode failure
→ retry 3 times
→ still fails
→ DLQ
```

注意：

> DLQ is an operational mechanism, not a complete data-quality strategy.

因为进入 DLQ 之后仍然需要：

```text
diagnose
repair
replay
verify
```

---

## 二十、Alert 和 Incident 有什么区别？

Alert：

> A signal that something may require attention.

Incident：

> A tracked operational problem with lifecycle, ownership, and resolution state.

例如：

```text
duplicate_key_count > 0
→ Alert
```

如果确认影响生产数据：

```text
Incident created
```

Incident 可能包含：

```text
incident_id
severity
owner
affected_range
root_cause
repair_job_id
status
```

---

## 二十一、为什么不能“每个 FAIL 都发报警”？

如果每分钟执行 100 个规则：

```text
50 个 FAIL
50 封邮件
```

最后结果往往是：

> Alert Fatigue.

真正需要的是：

```text
group
deduplicate
suppress
escalate
```

例如同一个根因：

```text
Provider Gap
```

可能导致：

```text
Missing Block
Missing Log
Fact Count Drop
DWS Volume Drop
```

这些不应该被当成 4 个完全独立 Incident。

---

## 二十二、[Operations 视角] Symptom 和 Root Cause 要分开

例如：

```text
Fact Count Down 20%
```

只是 symptom。

Root Cause 可能是：

```text
Provider Gap
Decoder Bug
Reorg
Filter Change
Consumer Stalled
```

Framework 应该保存：

```text
observed symptom
+
diagnosed root cause
```

不能把两者混成一个字段。

---

## 二十三、Incident Lifecycle

一个基础状态流：

```text
DETECTED
↓
TRIAGED
↓
CONTAINED
↓
DIAGNOSED
↓
REPAIRING
↓
VERIFYING
↓
RESOLVED
↓
CLOSED
```

这里 TRIAGED 可以理解为：

> confirmed, classified, prioritized.

---

## 二十四、Containment 是什么？

Containment 的目标：

> Stop bad data from spreading further.

例如：

```text
Fact Validation FAIL
```

可以：

```text
stop checkpoint
prevent DWS refresh
prevent ADS publish
```

这样可以把问题限制在较小 Blast Radius 内。

---

## 二十五、为什么 Containment 很重要？

如果不阻断：

```text
bad Fact
↓
DWS
↓
ADS
↓
Dashboard
↓
API cache
↓
customer
```

修复范围会越来越大。

所以：

> Early containment reduces downstream repair cost.

---

## 二十六、Repair Layer 怎么接入？

Framework 不需要自己“亲自修所有问题”，但应该能够：

```text
create repair request
track repair_job_id
track repair scope
track sink status
track validation result
```

例如：

```text
Incident I123
↓
Repair Job R456
↓
Backfill blocks 20M–20.5M
↓
Postgres VERIFIED
ClickHouse VERIFIED
Parquet VERIFIED
```

---

## 二十七、Validation 和 Verification 的区别

这里可以做一个工程区分：

Validation：

> Does the data satisfy defined correctness checks?

Verification：

> After repair, did the system actually converge to the intended state?

例如修复后：

```text
Validation:
no duplicate key
range complete

Verification:
Postgres / ClickHouse / Parquet reconcile
DWS rebuilt
Incident can close
```

---

## 二十八、Audit Trail 应记录什么？

Audit Trail 就是：

> A traceable history of what happened, why, and what was done.

至少应能回答：

```text
Which rule failed?
When?
Which range?
Which dataset?
What severity?
Was checkpoint blocked?
Who / what triggered repair?
Which repair job?
Which logic version?
What was the validation result?
When was incident closed?
```

---

## 二十九、一个基础数据模型

可以想象有几类核心表：

```text
dq_rule
dq_check_run
dq_metric
dq_incident
repair_job
repair_sink_status
```

它们不是必须真的这样命名，但职责应该分开。

---

## 三十、Rule Table 示例

```text
dq_rule

rule_id
rule_name
quality_dimension
dataset
metric_name
threshold
severity
blocking_policy
owner
is_active
```

---

## 三十一、Check Run 示例

```text
dq_check_run

check_run_id
rule_id
range_start
range_end
actual_value
expected_value
result
started_at
finished_at
```

这样你可以追踪：

> This exact rule, on this exact range, passed or failed.

---

## 三十二、Incident Table 示例

```text
dq_incident

incident_id
check_run_id
severity
status
affected_scope
root_cause
repair_job_id
created_at
resolved_at
```

这让：

```text
Detection
```

和：

```text
Operational Response
```

连接起来。

---

## 三十三、Blockchain-specific Framework 要多考虑什么？

传统数据平台已经有 DQ Framework。

Blockchain 额外需要考虑：

```text
Reorg
Canonical Status
Finality
Provider Gap
Block Continuity
Replayability
Chain-specific identity
```

所以区块链 DQ 不是完全新的理论，

而是：

> Traditional data quality + mutable canonical history + replayable source.

---

## 三十四、一个完整 Blockchain DQ Flow

可以设计成：

```text
RPC / Raw / Kafka
↓
Processing
↓
Validation
↓
Reconciliation
↓
DQ Decision
├─ PASS → Advance Checkpoint
├─ WARN → Continue + Alert
├─ BLOCK → Hold Checkpoint
└─ QUARANTINE → Isolate Bad Records
↓
Incident
↓
Repair / Replay / Backfill
↓
Verification
↓
Resume / Close
```

---

## 三十五、DQ Framework 和 Checkpoint 如何连接？

这是本 Module 最重要的连接之一。

Checkpoint 不应该只表示：

```text
code finished
```

而应该更接近：

> validated completion.

因此：

```text
processing success
+
required DQ checks PASS
=
checkpoint advancement
```

但要注意：

```text
non-blocking freshness failure
```

不一定阻止 Checkpoint。

---

## 三十六、[Banking analogy] 银行数据平台怎么类比？

假设每日交易流水 ETL（Extract, Transform, Load，抽取、转换、加载）结束。

不能因为 Job 显示：

```text
SUCCESS
```

就直接发布报表。

还需要：

```text
source count
amount reconciliation
duplicate check
accounting invariant
```

通过后：

```text
publish
```

否则：

```text
hold publication
open incident
repair
reconcile again
```

Blockchain Data Platform 本质上也是这个逻辑，只是多了：

```text
Reorg
Canonicality
Replay
Finality
```

---

## 三十七、Framework 最容易犯的错误：只有 Detection

很多团队做到：

```text
lots of metrics
lots of dashboards
lots of alerts
```

但没有：

```text
blocking semantics
repair workflow
ownership
verification
audit
```

结果就是：

> Observable, but not operable.

真正成熟的系统应该做到：

> Detectable + Actionable + Repairable + Verifiable + Auditable.

---

## 三十八、Another common failure：Rule without business semantics

例如：

```text
row_count > 0
```

这个规则可能永远 PASS，

却完全不能证明数据正确。

好的 Rule 应该来自：

```text
grain
source identity
business invariant
expected relationship
product requirement
```

所以：

> Data Quality Rule Design starts from data semantics, not from generic SQL templates.

---

## 三十九、Golden Dataset 是什么？

Golden Dataset 可以理解为：

> A small, trusted reference dataset with known-correct expected outputs.

例如你挑选：

```text
100 known ERC-20 transfers
10 known swaps
known mint / burn cases
reorg sample
```

每次 Decoder 升级后跑一遍：

```text
actual
vs
expected
```

这对 Accuracy Regression 很有帮助。

---

## 四十、Golden Dataset 不是生产全量 Reconciliation

它适合：

```text
known cases
regression testing
decoder validation
```

但不能替代：

```text
production completeness
production reconciliation
freshness monitoring
```

所以它是：

> one component of the framework.

---

## 四十一、Data Quality Framework 的最低可用版本

对于你现在的 Blockchain Data Platform，不需要一开始做得很重。

一个最小版本可以只有：

```text
1. Rule definitions
2. Check execution
3. Blocking vs non-blocking decision
4. Incident record
5. Repair job tracking
6. Verification
7. Audit log
```

已经足够形成生产级思维。

---

## 四十二、你可以如何落到项目里？

毕业项目里可以做：

```text
Indexer
↓
Fact Load
↓
DQ Checks
↓
Checkpoint Gate
↓
DWS / ADS
```

再增加：

```text
dq_rule
dq_check_run
dq_incident
repair_job
```

这样你的项目就不再只是：

> data pipeline

而是：

> repairable and governable data platform.

---

## 四十三、本课核心心智模型

可以压缩为：

```text
Define Truth
↓
Measure
↓
Evaluate
↓
Decide
↓
Contain
↓
Repair
↓
Verify
↓
Audit
```

或者：

```text
Rule
→ Metric
→ Check
→ Decision
→ Incident
→ Repair
→ Verification
→ Closure
```

---

## 本课核心结论

> A Data Quality Framework is an operational control system, not a collection of SQL checks.

> Rules define what should be true; Metrics measure reality; Check Runs evaluate rules on a specific scope.

> Severity describes impact, while Blocking Policy decides whether processing may continue.

> Not every DQ failure should block Checkpoint advancement.

> If the current processing range is not trustworthy, the Checkpoint should be blocked.

> Quarantine and DLQ (Dead Letter Queue，死信队列) isolate problematic records, but they do not replace diagnosis and repair.

> Alerts are signals; Incidents are tracked operational problems with lifecycle and ownership.

> Containment prevents bad data from spreading downstream and reduces repair cost.

> Repair is not complete until Verification proves the system has converged.

> A Blockchain DQ Framework must explicitly handle Reorg, Canonicality, Replayability, and Multi-sink convergence.

> A production-grade platform should be Detectable, Actionable, Repairable, Verifiable, and Auditable.

---

## 理解检查

### 问题一

某个 Fact Job 已经完成写入，随后执行 DQ Checks：

```text
Completeness = PASS
Uniqueness = PASS
Reconciliation = FAIL
Freshness = PASS
```

请回答：

1. Checkpoint 应不应该推进？
2. 为什么？
3. 这里真正起决定作用的是 Severity，还是 Blocking Policy？

### 问题二

某 Pipeline 当前：

```text
Freshness SLO FAIL
block_lag = 800
```

但是：

```text
processed range
→ complete
→ accurate
→ reconciled
```

请回答：

1. 是否一定要阻止 Checkpoint？
2. 更合理的 Framework Action 是什么？
3. 为什么 Freshness Failure 和 Reconciliation Failure 的处理策略不同？

### 问题三

一个 Decoder Bug 导致：

```text
Fact Accuracy FAIL
↓
DWS wrong
↓
ADS wrong
```

系统已经：

```text
detected
alerted
```

请回答：

1. 为什么这还不能算完整的 Data Quality Framework？
2. 接下来至少还需要哪些阶段？
3. Repair Job 完成后，为什么还不能直接 Close Incident？

## 理解检查处理决定

用户明确要求：

> 跳过这课的回答，开始下一课

因此本课的理解检查不再继续作答。

## 结课判定

Module 12 第 9 课课程正文已完成；理解检查由用户主动跳过。

本课核心内容已覆盖：
- DQ（Data Quality，数据质量）Framework 的 Rule / Metric / Check / Decision / Incident / Repair / Verification / Audit 全链路。
- Severity 与 Blocking Policy 的区别。
- Blocking / Non-blocking DQ Failure 的判断逻辑。
- Quarantine 与 DLQ（Dead Letter Queue，死信队列）的用途边界。
- Alert 与 Incident 的区别及 Incident Lifecycle。
- Containment、Repair、Verification、Audit Trail 的职责。
- Blockchain-specific DQ：Reorg、Canonicality、Replayability 与 Multi-sink convergence。
- 最小可用 Data Quality Framework 的组成。

理解检查未执行，不等同于理解检查通过；但根据用户明确选择，本课结束并进入下一课。