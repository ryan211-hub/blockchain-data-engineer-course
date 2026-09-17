# 第7课｜DAG / Dependency / Scheduler：多个 ETL Job 如何组成一条可运行的数据管道

【Lesson Contract】
所属 Module：Module 9 — ETL。
本课核心问题：
> 单个 ETL Job 我们已经会设计了，但生产环境不是只有一个 Job。几十、几百个 Job 之间怎么确定先后关系、失败影响和调度顺序？

学完本课，你应该能够理解：
- Job、Dependency、DAG、Scheduler / Orchestrator 分别是什么；
- 为什么“每天凌晨依次执行几个脚本”不等于真正的 Pipeline；
- 上游成功为什么是下游执行的重要前提；
- Data Dependency 与 Time Dependency 的区别；
- Retry、Skip、Block、Backfill 如何受 DAG 影响；
- 为什么每个 Job 仍然需要自己的 Checkpoint；
- 怎样把我们前面设计的几张链上表组成一个简单 DAG。

本课只讲基础，不深入 Airflow、Dagster、Prefect 等具体产品。

---

## 一、从单个 Job 到多个 Job

前六课讨论单个 Job：

```text
Input Range
↓
Extract
↓
Transform
↓
Idempotent Load
↓
Validation
↓
Checkpoint
```

生产环境会同时存在 Fact、DWS、ADS 等多个 Job，所以问题变成：谁先跑、谁后跑，以及谁依赖谁。

## 二、Dependency 的本质

Dependency 表示一个 Job 的执行或正确结果依赖另一个 Job 的输出。例如：

```text
A: build_fact_token_transfers
↓
B: build_dws_wallet_token_daily_flow
```

`B depends on A` 的真正含义是：B 的 Input 依赖 A 的 Output。

## 三、Dependency 本质上是数据关系

【Data Engineer 视角】最值得关注的不是“哪个脚本先运行”，而是谁消费谁的数据。

```text
Raw
→ Fact
→ DWS
→ ADS
```

顺序来自数据依赖，而不是人为指定的时钟顺序。

## 四、Time Dependency 与 Data Dependency

固定配置：

```text
01:00 A
02:00 B
03:00 C
```

只是 Time Dependency。如果 A 今天到 02:30 才完成，B 在 02:00 启动就可能读到不完整结果。

可靠做法应是：

```text
A SUCCESS + Validation PASS
↓
B 才允许开始
```

## 五、Scheduler / Orchestrator

Scheduler 决定什么时候尝试启动任务；Orchestrator 管理整个任务依赖与运行状态。实际系统通常还管理 Schedule、Dependency、Execution State、Retry、Backfill Trigger。

## 六、DAG 是什么

DAG = Directed Acyclic Graph，有向无环图。节点是 Job，箭头是 Dependency。

```text
        fact_token_transfers
        /                  \
       ↓                    ↓
wallet_daily_flow     token_daily_volume
```

## 七、Directed 与 Acyclic

`A → B` 表示 B depends on A。Acyclic 表示依赖不能形成环，否则没有任务能成为合法起点。

## 八、DAG Dependency 与 Foreign Key 不同

Foreign Key 是数据实体的引用关系；Pipeline Dependency 是 Job 执行与数据生产的依赖关系。

## 九、Blockchain Pipeline DAG 示例

```text
Raw Ethereum Logs
        ↓
decode_erc20_transfer
        ↓
fact_token_transfers
       /        \
      ↓          ↓
wallet_flow   token_volume
      ↓          ↓
wallet_ads   token_ads
```

A 成功后，互不依赖的 B/C 可以并行。

## 十、Dependency 是最小必要约束

只有存在真实数据依赖时才应该加箭头。把互不依赖的 Job 强行串行，只会增加等待与故障传播。

## 十一、上游失败时下游怎么办

```text
A → B → C
```

若 A FAILED，通常 B/C 应是 BLOCKED / UPSTREAM_FAILED / NOT_RUN，而不是继续执行。FAILED 表示任务自己执行失败；BLOCKED 表示因上游条件不满足而没有执行。

## 十二、Retry 与 Idempotency

Orchestrator 可以 Retry A；A 第二次成功后再触发 B。正因为 Retry 是生产常态，所以 Job 必须可安全重跑。

## 十三、Scheduler 不负责业务正确性

Scheduler 可以看到任务状态，但通常不知道业务结果是否正确。因此应把：

```text
Execution Success + Validation Pass = Job SUCCESS
```

作为下游依赖成立的条件。

## 十四、每个 Job 为什么仍然需要自己的 Checkpoint

DAG 管谁依赖谁；Checkpoint 管每个 Job 自己处理到哪里。多个 Job 的 Checkpoint 可以不同，例如：

```text
Indexer block = 20,010,000
Fact ETL date = 09-15
DWS ETL date = 09-14
ADS ETL date = 09-13
```

所以 DAG 不是一个全局 Checkpoint。

## 十五、Processing Unit Dependency

真实依赖通常不只是 `B depends on A`，而是：

```text
B(09-15) depends on A(09-15)
```

即对应 Processing Unit / Partition 的依赖。B 的 Input Range 必须由上游 Output 覆盖。

## 十六、Backfill 对 DAG 的影响

若 `Fact(09-10)` Backfill，可能需要沿依赖向下修复：

```text
Fact(09-10)
↓
DWS(09-10)
↓
ADS(09-10)
```

但真正 Repair Range 不能只看 Source Changed Range，还要看下游模型是否有跨时间依赖。

## 十七、Independent Partition 与 State-dependent

如果下游是按天独立聚合，Fact(09-10) 变化可能只需修复 DWS/ADS(09-10)。如果下游是状态递推，例如：

```text
balance(09-10)
↓
balance(09-11)
↓
balance(09-12)
```

那么 09-10 的错误会继续影响后续日期，Repair Range 需要向后延伸。

## 十八、Scheduler 与 ETL Logic 的职责边界

Orchestrator 负责何时调用、前置条件、失败后下一步；Job Implementation 负责 Decode、GROUP BY、Upsert 等数据逻辑。

## 十九、核心职责分工

```text
Job
负责做一件数据处理工作

Dependency
描述谁需要谁的 Output

DAG
描述整个 Pipeline 的依赖拓扑

Scheduler / Orchestrator
根据时间、依赖和运行状态决定谁什么时候执行

Checkpoint
记录每个 Job 自己完成到哪里
```

---

## 本课理解检查

### 问题一
有三个 Job：A=build_fact_token_transfers，B=build_wallet_daily_flow，C=build_token_daily_volume；B/C 都读取 A，彼此不依赖。回答是否并行、A 失败时下游状态、为什么不能只用固定时钟顺序。

### 用户回答
问题一，1，A完成后，B和C可以并行。2，如果 A 失败，B和C应该是等待的状态。3，为什么不应该简单安排 A 1点，B2点，C 3点启动？因为 A、B、C，它们是有数据上的依赖关系。这样简单按时间排序执行的话，很有可能 A 会有延迟的情况。那么，B和C到点正常执行，其实没有意义，因为那个小 A 的数据是有问题的。所以正确的方式应该是用 Schedule 来看 A、B、C 的数据依赖关系。这样 A 真正完成以后，才触发 B和C的启动。

### 教师判断与校准
问题一回答正确。A 完成后 B/C 可并行。A 失败时更准确是 BLOCKED / UPSTREAM_FAILED / NOT_RUN，而不是普通等待。另一个术语校准：这里应是 Scheduler / Orchestrator，而不是 Schedule。Schedule 只表示什么时候启动；Dependency 表示谁依赖谁；Scheduler / Orchestrator 根据 Schedule + Dependency + Runtime State 决定谁可以运行。

### 问题二
假设 A checkpoint=09-16，B=09-15，C=09-14，是否说明整个 Pipeline 错误？解释 DAG Dependency 与各 Job Checkpoint 的职责。

### 用户回答
这并不能说明整个 pipeline 状态错误，因为 checkpoint 它保证的是 job 自己的执行完成度。

DAG dependency 是不一样的，它需要保证 DAG 上各个 job 之间的相互依赖。

### 教师判断与校准
问题二正确。Checkpoint = 每个 Job 自己的 Processing State；DAG Dependency = Job 与 Job 之间的依赖关系。更严格地说，DAG 关心的是 B(09-16) 是否依赖 A(09-16)，而不是要求所有 Job 的 Checkpoint 永远一致。

### 问题三
正常依赖 Fact → DWS → ADS。Fact(09-10) 需要 Backfill。回答为什么不能只修 Fact、是否一定从 09-10 重跑到今天、依据什么确定下游 Repair Range。

### 用户回答
一、因为 fact 的数据有变化，那么它的下游 DWS 和 ADS 这些都会受到影响。

二、DWS 和 ADS 不一定要从 09-10 一直重跑到今天。

三、应该依据 fact 数据的变化范围来确定下游的 repair range。比如，如果 fact 修复的数据只在 09-10 这一天，那么下游 DWS 和 ADS 只需要重跑 09-10 这一天的数据就行。

### 教师判断与校准
前两点正确。第三点需要校准：只修 09-10 只在下游按天独立计算时成立。真正的 Repair Range 由 Source Changed Range + Downstream Dependency + Processing Unit / Partition Relationship + 是否存在跨时间依赖共同决定。

### 最终校准问题
如果 `DWS_wallet_balance(09-11)` 是通过 `balance(09-10) + 09-11 net_flow` 算出来的，而 Fact(09-10) 被修正，为什么不能只重跑 09-10？

### 用户回答
因为 balance 余额是由 transfer 交易计算出来的，所以当漏掉一条 transfer 的话，对后续每天的余额计算都会有影响。

如果是这种 DWS 日报余额表，那么它需要重跑 09-10 之后所有的天数

### 最终判断
回答正确。对于状态递推型 DWS，09-10 的错误会沿 previous-day state 继续传播到后续日期，因此 Repair Range 需要从 09-10 向后延伸到当前有效范围。第 7 课理解检查与必要校准全部通过，教学正式结束。