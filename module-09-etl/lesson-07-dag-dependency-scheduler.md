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

## 一、我们前六课其实都只盯着“一个 Job”

到目前为止，我们一直在讨论：

```text
fact_token_transfers
        ↓
Daily ETL
        ↓
dws_wallet_token_daily_flow
```

对单个 Job，我们已经知道怎么处理：

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

但是现实里不会只有这一条。

假设一个链上数据平台每天要产出：

```text
fact_token_transfers
fact_swaps
dim_token

dws_wallet_token_daily_flow
dws_token_daily_volume
dws_pool_daily_volume

ads_wallet_overview
ads_token_dashboard
```

现在问题变了：

> 这些 Job 谁先跑，谁后跑？

这就是 Dependency 问题。

---

## 二、什么叫 Dependency

Dependency 就是：

> 一个 Job 的执行或正确结果，依赖另一个 Job 的输出。

例如：

```text
Job A
build_fact_token_transfers
```

产生：

```text
fact_token_transfers
```

而：

```text
Job B
build_dws_wallet_token_daily_flow
```

要读取：

```text
fact_token_transfers
```

于是：

```text
B depends on A
```

画成：

```text
A
↓
B
```

这条箭头不是说：

> A 比 B 更重要。

而是说：

> B 的 Input 依赖 A 的 Output。

---

## 三、Dependency 本质上是数据关系

【Data Engineer 视角】

最值得关注的不是“哪个脚本先运行”，而是：

```text
谁消费谁的数据？
```

例如：

```text
Raw Logs
   ↓
fact_token_transfers
   ↓
dws_wallet_token_daily_flow
   ↓
ads_wallet_overview
```

这里自然形成：

```text
Raw
→ Fact
→ DWS
→ ADS
```

所以：

```text
ads_wallet_overview
```

不是因为“规定凌晨 3 点执行”，所以排在最后。

而是因为它的数据依赖决定了：

```text
Fact
必须先存在

DWS
必须先正确

ADS
才能计算
```

这是非常重要的区别。

---

## 四、Time Dependency 和 Data Dependency 不一样

假设你配置：

```text
01:00 Job A
02:00 Job B
03:00 Job C
```

看起来有顺序。

但这是：

```text
Time Dependency
```

即：

> 按时间假设 A 已经跑完，所以 B 可以开始。

问题是：

如果 A 今天特别慢：

```text
01:00 start
02:30 finish
```

而 B：

```text
02:00
```

准时启动。

那么 B 会读取什么？

很可能是：

```text
不完整的 A Output
```

这就出问题了。

所以生产 Pipeline 更希望使用：

```text
Data / Task Dependency
```

即：

```text
A SUCCESS
↓
B 才允许开始
```

而不是：

```text
时间到了
↓
管 A 完没完成
B 都开始
```

---

## 五、Scheduler 到底负责什么

很多人会把 Scheduler 理解成：

> 定时器。

这是不够的。

最简单 Scheduler 当然可以只是：

```text
每天 01:00
启动 ETL
```

但现代 Data Pipeline 的 Scheduler / Orchestrator 通常还要负责：

```text
什么时候启动
谁依赖谁
哪个任务正在运行
哪个成功
哪个失败
失败是否重试
下游是否阻塞
历史任务如何补跑
```

所以从工程角度更准确：

> Scheduler 决定“什么时候尝试执行”；Orchestrator 管理“整个任务依赖与运行状态”。

实际产品里两者经常由同一个系统完成，所以日常交流里经常混着说。

---

## 六、DAG 是什么

DAG：

```text
Directed Acyclic Graph
```

中文通常叫：

```text
有向无环图
```

名字看起来数学味很重，但在 ETL 中非常直观。

例如：

```text
        fact_token_transfers
        /                  \
       ↓                    ↓
wallet_daily_flow     token_daily_volume
       \                    /
        ↓                  ↓
          wallet_dashboard
```

每个节点：

```text
Job
```

每条箭头：

```text
Dependency
```

这就是 DAG。

---

## 七、“Directed” 是什么意思

Directed = 有方向。

例如：

```text
A → B
```

意味着：

```text
B depends on A
```

不能反过来理解成：

```text
A depends on B
```

数据流有明确方向：

```text
Source
↓
Derived Data
↓
Further Derived Data
```

---

## 八、“Acyclic” 为什么重要

Acyclic = 不能形成环。

例如下面就是错误依赖：

```text
A → B
↑   ↓
└── C
```

也就是：

```text
A 依赖 C
C 依赖 B
B 依赖 A
```

那到底谁先跑？

没人能先跑。

于是系统：

```text
deadlock-like dependency
```

所以 DAG 必须无环。

---

## 九、和数据库 Foreign Key 不一样

这里也容易混淆。

DAG Dependency 不是数据库里的：

```text
Foreign Key
```

Foreign Key 表达的是：

```text
数据实体之间的引用关系
```

而 Pipeline Dependency 表达的是：

```text
Job 执行和数据生产的依赖关系
```

例如：

```text
dim_token
```

和：

```text
fact_token_transfers
```

在模型层面可能有逻辑关联，

但是否存在 ETL Job Dependency，还要看 Transform 是否真的需要先读取 `dim_token`。

---

## 十、一个 Blockchain Pipeline DAG

我们拿 Ethereum 数据来构造：

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

可以拆成：

```text
Job A
decode_token_transfers

Job B
build_wallet_daily_flow

Job C
build_token_daily_volume

Job D
build_wallet_dashboard

Job E
build_token_dashboard
```

Dependency：

```text
A → B
A → C
B → D
C → E
```

---

## 十一、为什么 B 和 C 可以并行

因为：

```text
B depends on A
C depends on A
```

但：

```text
B 不依赖 C
C 也不依赖 B
```

所以 A 成功以后：

```text
       A
      / \
     B   C
```

B 和 C 可以同时执行。

这就是 DAG 带来的一个重要能力：

> 不仅描述顺序，也描述哪里可以并行。

如果全部用一个 Shell：

```text
A
↓
B
↓
C
↓
D
↓
E
```

就会人为制造很多不必要的串行等待。

---

## 十二、Dependency 是最小必要约束

一个好的 DAG 不应该表达：

> “我习惯先跑这个，再跑那个。”

而应该表达：

> “没有这个上游，下游的数据就不能正确产生。”

例如：

```text
A → B
```

应该有实际数据含义。

如果 B 和 C 完全独立：

```text
B
C
```

就没有必要强行写：

```text
B → C
```

否则整个系统变慢，还增加故障传播。

---

## 十三、上游失败时，下游应该怎么办

假设：

```text
A → B → C
```

今天：

```text
A FAILED
```

那么正常情况下：

```text
B BLOCKED
C BLOCKED
```

而不是：

```text
B 继续执行
C 继续执行
```

为什么？

因为 B 读取的是 A 的 Output。

如果 A 没有完成：

```text
B 的 Input Contract
不成立
```

所以不应该执行。

这和上一课：

```text
Validation Pass
才能 Advance Checkpoint
```

其实是一套逻辑。

---

## 十四、注意：FAILED 和 BLOCKED 不一样

这个区别很实用。

```text
A FAILED
```

表示：

> A 自己执行了，但是失败了。

而：

```text
B BLOCKED
```

表示：

> B 可能根本没执行，因为前置条件没有满足。

所以：

```text
FAILED
```

和：

```text
NOT RUN / BLOCKED / UPSTREAM FAILED
```

是不同状态。

---

## 十五、Retry 应该发生在哪一层

假设：

```text
A → B
```

A 因为数据库连接临时失败：

```text
FAILED
```

Scheduler 可以：

```text
retry A
```

如果第二次：

```text
A SUCCESS
```

再：

```text
trigger B
```

这也是为什么前面一直强调：

```text
Idempotency
```

因为 Scheduler Retry 是生产系统的常态。

如果 Job 不可安全重跑：

> Orchestrator 再聪明也救不了你。

---

## 十六、Scheduler 不应该负责业务正确性

这里要加一个重要边界。

【Scheduler 视角】

Scheduler 可以知道：

```text
A SUCCESS
```

但它通常不知道：

```text
A 的业务数据是否真的正确
```

所以 Lesson 6 的 Validation 很重要。

一个更严格的状态应该是：

```text
A
Execution Success
+
Validation Pass
=
A SUCCESS
```

然后：

```text
B
才可以运行
```

否则 Scheduler 只是把错误数据更快地传播到下游。

---

## 十七、每个 Job 为什么仍然需要自己的 Checkpoint

假设：

```text
Indexer
↓
Fact ETL
↓
DWS ETL
↓
ADS ETL
```

是不是有一个全局：

```text
checkpoint = 09-15
```

就够了？

不够。

因为每个 Pipeline 的完成状态不同。

例如：

```text
Indexer
block = 20,010,000

Fact ETL
date = 09-15

DWS ETL
date = 09-14

ADS ETL
date = 09-13
```

这是完全可能的。

所以：

> DAG 管 Dependency；Checkpoint 管每个 Job 自己的 Processing State。

职责不同。

---

## 十八、DAG 不是一个全局 Checkpoint

这个概念一定要分开。

DAG：

```text
A → B → C
```

描述：

```text
谁依赖谁
```

Checkpoint：

```text
A checkpoint
B checkpoint
C checkpoint
```

描述：

```text
每个 Job 自己完成到哪里
```

Scheduler：

```text
根据依赖和状态
决定接下来运行谁
```

三者不要混在一起。

---

## 十九、同一天的数据，也可能不同步推进

例如：

```text
A = fact_token_transfers
B = wallet_daily_flow
C = wallet_dashboard
```

状态：

```text
A checkpoint = 09-16
B checkpoint = 09-15
C checkpoint = 09-14
```

这并不意味着一定出错。

只是：

```text
A 领先
B 落后一批
C 又落后一批
```

Scheduler 后续可以依次补齐：

```text
B(09-16)
↓
C(09-15)
↓
C(09-16)
```

实际怎么调度，要看 Pipeline 设计。

---

## 二十、Dependency 还必须考虑 Processing Unit

假设：

```text
A
生产 09-15 Fact
```

B 要处理：

```text
09-15 DWS
```

那么 Dependency 不能只表达：

```text
B depends on A
```

更严格应该理解为：

```text
B(09-15)
depends on
A(09-15)
```

这叫：

```text
partition-level dependency
```

或者更泛化：

```text
Processing Unit Dependency
```

---

## 二十一、为什么这个区别重要

假设 A：

```text
09-14 SUCCESS
09-15 FAILED
09-16 尚未运行
```

B：

```text
09-15
```

能不能因为：

```text
“A 历史上成功过”
```

就执行？

显然不能。

B 需要的是：

```text
A(09-15) SUCCESS
```

不是：

```text
A somewhere SUCCESS
```

所以生产 Pipeline 的 Dependency 通常不只是 Job-level：

```text
A → B
```

还隐含：

```text
same partition / matching range
```

---

## 二十二、这和 Input Boundary 又连起来了

Lesson 6 我们刚学：

```text
Job B Input Range
=
[09-15, 09-16)
```

那么 B 的上游依赖就应该保证：

```text
A Output
覆盖了这个 Input Range
```

所以：

```text
Dependency
```

本质上不是单纯的：

> 等一个程序结束。

而是：

> 等待满足自己的 Input Contract 的上游数据准备完成。

这个理解更接近真正的数据工程。

---

## 二十三、Backfill 对 DAG 有什么影响

假设正常 Pipeline：

```text
Fact
↓
DWS
↓
ADS
```

现在发现：

```text
09-10 Fact
```

需要 Backfill。

Fact 修复完成以后：

```text
09-10 DWS
```

可能也需要重算。

然后：

```text
09-10 ADS
```

可能也需要重算。

所以 Historical Repair 也会沿着 Dependency 向下传播：

```text
Fact(09-10)
↓
DWS(09-10)
↓
ADS(09-10)
```

这和上一课我们提到的 Lineage / Affected Range 是同一套思想。

---

## 二十四、但不是所有下游都一定要重跑

仍然要根据：

```text
Dependency
+
Affected Range
```

判断。

例如：

```text
DWS_A
依赖 Fact
```

而：

```text
DWS_B
完全不使用这个 Fact
```

那么 Fact 修复：

```text
不应该导致整个仓库全部重算
```

因此 DAG 也帮助我们回答：

> 上游变化到底会影响哪些下游？

---

## 二十五、Scheduler 和 DAG 解决的是“组织问题”

单个 ETL Job 解决：

```text
我自己怎么正确地跑
```

DAG / Orchestrator 解决：

```text
大家怎么正确地一起跑
```

这是两个层次。

可以类比银行系统：

单个存储过程：

```text
负责自己计算正确
```

批量调度平台：

```text
负责依赖、时序、状态、重试、补跑
```

---

## 二十六、一个很典型的错误设计

假设有三个脚本：

```text
01_fact.sql
02_dws.sql
03_ads.sql
```

然后 Crontab：

```text
01:00 run fact
01:30 run dws
02:00 run ads
```

这种设计在规模小时能工作。

但它暗含：

```text
fact 一定 30 分钟内完成
dws 一定 30 分钟内完成
```

只要某天数据量增大：

```text
fact 跑了 50 分钟
```

DWS 就可能提前读半成品。

所以：

> “固定时间间隔”不是可靠 Dependency。

---

## 二十七、正确思路是事件状态驱动

更可靠：

```text
Fact
SUCCESS + Validation PASS
        ↓
Trigger DWS
        ↓
SUCCESS + Validation PASS
        ↓
Trigger ADS
```

这里：

```text
Clock
```

主要负责启动整批 Pipeline。

真正决定下游是否能跑的是：

```text
Dependency State
```

---

## 二十八、Scheduler 还负责什么

在基础层面，你可以认为它至少管理：

```text
Schedule
Dependency
Execution State
Retry
Backfill Trigger
```

例如：

```text
每天 01:00
启动 daily_pipeline
```

然后：

```text
A
失败最多重试 3 次

A SUCCESS
→ B / C 并行

B SUCCESS
→ D

C SUCCESS
→ E
```

这已经是一个基本 Orchestrator。

---

## 二十九、Orchestrator 不是 ETL Engine

这个角色边界也值得明确。

Orchestrator 通常不负责：

```text
怎么 decode Event
怎么 GROUP BY
怎么算 sent_amount
怎么写 Upsert SQL
```

这些属于：

```text
Job Implementation
```

Orchestrator 负责：

```text
什么时候调用这个 Job
这个 Job 的前置条件是什么
成功失败后下一步怎么办
```

所以：

```text
ETL Logic
```

和：

```text
Workflow Control
```

应该分开。

---

## 三十、把 Module 9 到目前为止串起来

现在我们已经从一个 SQL，逐渐走到一个 Pipeline。

```text
Lesson 1
ETL 是长期 Pipeline

Lesson 2
处理什么范围

Lesson 3
怎么记录进度

Lesson 4
怎么安全重跑

Lesson 5
为什么历史会变化

Lesson 6
怎么验证一批数据

Lesson 7
多个 Job 怎么组织运行
```

一个单独 Job：

```text
Input
↓
Transform
↓
Output
↓
Validation
↓
Checkpoint
```

多个 Job：

```text
Job A
↓
Job B
↓
Job C
```

组合后就是：

```text
Data Pipeline DAG
```

---

## 三十一、最核心的职责分工

建议你直接记住这四句话：

```text
Job
负责做一件数据处理工作

Dependency
描述谁需要谁的 Output

DAG
描述整个 Pipeline 的依赖拓扑

Scheduler / Orchestrator
根据时间、依赖和运行状态决定谁什么时候执行
```

再加一条：

```text
Checkpoint
记录每个 Job 自己完成到哪里
```

这样这几个概念就不会混。

---

## 本课理解检查

问题一：

有三个 Job：

```text
A: build_fact_token_transfers
B: build_wallet_daily_flow
C: build_token_daily_volume
```

其中：

```text
B 读取 A 的结果
C 也读取 A 的结果
B 和 C 互不依赖
```

请画出 DAG，并回答：

1. A 完成后，B 和 C 能不能并行？
2. 如果 A 失败，B 和 C 应该是什么状态？
3. 为什么不应该简单安排成 A 01:00、B 02:00、C 03:00？

问题二：

假设：

```text
A checkpoint = 09-16
B checkpoint = 09-15
C checkpoint = 09-14
```

是否说明整个 Pipeline 状态错误？

请解释：

> DAG Dependency 和各 Job Checkpoint 的职责分别是什么？

问题三：

正常依赖：

```text
Fact
↓
DWS
↓
ADS
```

后来发现：

```text
Fact(09-10)
```

需要 Backfill。

请回答：

1. 为什么不能只修 Fact 就结束？
2. DWS 和 ADS 是否一定要“从 09-10 一直重跑到今天”？
3. 应该依据什么来确定真正的下游 Repair Range？

---

## 理解检查记录（Canonical Increment Segments）

### 用户回答｜问题一

问题一，1，A完成后，B和C可以并行。2，如果 A 失败，B和C应该是等待的状态。3，为什么不应该简单安排 A 1点，B2点，C 3点启动？因为 A、B、C，它们是有数据上的依赖关系。这样简单按时间排序执行的话，很有可能 A 会有延迟的情况。那么，B和C到点正常执行，其实没有意义，因为那个小 A 的数据是有问题的。所以正确的方式应该是用 Schedule 来看 A、B、C 的数据依赖关系。这样 A 真正完成以后，才触发 B和C的启动。

### 教师判断与校准｜问题一

问题一回答正确。

有两个小校准：

1. 如果 A 失败，B 和 C 更准确的状态不是普通“等待”，而是类似：

```text
BLOCKED
UPSTREAM_FAILED
NOT_RUN
```

意思是：

> B、C 没有自己执行失败，而是因为上游 A 没有满足成功条件，所以暂时不能运行。

2. 你最后说“用 Schedule 来看 A、B、C 的数据依赖关系”，这里更准确应该是：

```text
Scheduler / Orchestrator
```

而不是 `Schedule`。

区别是：

```text
Schedule
= 什么时候启动

Dependency
= 谁依赖谁

Scheduler / Orchestrator
= 根据 Schedule + Dependency + Runtime State
  决定谁现在可以运行
```

所以你这题的核心理解是对的：

```text
        A
       / \
      ↓   ↓
      B   C
```

A 成功并且 Validation Pass 后，B、C 可以并行；A 失败时，B、C 被上游状态阻塞。固定安排成 01:00 / 02:00 / 03:00 只是 Time Dependency，不能保证下游真正读取到完整、正确的上游结果。

继续问题二：

```text
A checkpoint = 09-16
B checkpoint = 09-15
C checkpoint = 09-14
```

这是否说明整个 Pipeline 状态错误？

请解释：

> DAG Dependency 和各 Job Checkpoint 的职责分别是什么？

### 用户回答｜问题二

这并不能说明整个 pipeline 状态错误，因为 checkpoint 它保证的是 job 自己的执行完成度。

DAG dependency 是不一样的，它需要保证 DAG 上各个 job 之间的相互依赖。

### 教师判断与校准｜问题二

对，问题二正确。

再精确一点：

```text
Checkpoint
= 每个 Job 自己的 Processing State
= 它自己安全完成到哪里
```

而：

```text
DAG Dependency
= Job 与 Job 之间的依赖关系
= 谁必须等谁的对应 Processing Unit 完成
```

所以：

```text
A checkpoint = 09-16
B checkpoint = 09-15
C checkpoint = 09-14
```

只能说明三个 Job 当前推进速度不同，并不能直接说明 Pipeline 错了。

更严格地说，DAG 关心的是类似：

```text
B(09-16) 是否依赖 A(09-16)
C(09-15) 是否依赖 B(09-15)
```

而不是要求所有 Job 的 Checkpoint 永远完全相同。

继续问题三：

正常依赖：

```text
Fact
↓
DWS
↓
ADS
```

后来发现：

```text
Fact(09-10)
```

需要 Backfill。

请回答：

1. 为什么不能只修 Fact 就结束？
2. DWS 和 ADS 是否一定要从 09-10 一直重跑到今天？
3. 应该依据什么确定真正的下游 Repair Range？

### 用户回答｜问题三

一、因为 fact 的数据有变化，那么它的下游 DWS 和 ADS 这些都会受到影响。

二、DWS 和 ADS 不一定要从 09-10 一直重跑到今天。

三、应该依据 fact 数据的变化范围来确定下游的 repair range。比如，如果 fact 修复的数据只在 09-10 这一天，那么下游 DWS 和 ADS 只需要重跑 09-10 这一天的数据就行。

### 教师判断与校准｜问题三

前两点正确，第三点需要一个关键校准。

你说：

> 如果 Fact 只修复 09-10，那么 DWS 和 ADS 只需要重跑 09-10。

这个结论只在下游是“按天独立计算”的情况下成立。

例如：

```text
Fact(09-10)
↓
DWS daily_flow(09-10)
↓
ADS daily_dashboard(09-10)
```

如果 09-11 的结果完全不依赖 09-10，那么确实只需要修 09-10。

但如果下游是累计型或状态递推型，例如：

```text
balance(09-10)
↓
balance(09-11)
↓
balance(09-12)
↓
...
```

那么 Fact(09-10) 改变以后，影响可能一路传播到后续所有日期。

所以更准确的答案是：

```text
Repair Range
=
Source Changed Range
+
Downstream Dependency
+
Processing Unit / Partition Relationship
```

也就是不能只看 Fact 改了哪一天，还要看下游模型是否存在跨时间依赖。

你可以把它压缩成：

```text
Fact Changed Range
        ↓
Affected DWS Partitions
        ↓
Affected ADS Partitions
        ↓
沿 Dependency / Lineage 继续传播
        ↓
Determine Repair Range
```

因此第三题最终应回答：

> 下游 Repair Range 由 Source Changed Range、下游 DAG Dependency、各层 Processing Unit，以及是否存在跨时间依赖共同决定。

最后再确认一个小问题：

如果 `DWS_wallet_balance(09-11)` 是通过：

```text
balance(09-10) + 09-11 net_flow
```

算出来的，而 Fact(09-10) 被修正，那么为什么不能只重跑 09-10？

### 用户回答｜最终校准

因为 balance 余额是由 transfer 交易计算出来的，所以当漏掉一条 transfer 的话，对后续每天的余额计算都会有影响。

如果是这种 DWS 日报余额表，那么它需要重跑 09-10 之后所有的天数

### 最终判断

这道校准题回答正确。关键点就是：如果 `DWS_wallet_balance` 是递推状态，09-10 的错误会沿 `previous_day_balance → next_day_balance` 继续传播，因此 Repair Range 需要从 09-10 延伸到当前有效范围。第 7 课至此通过，正式结课。
