# EviStateDB / EviStateBench：从具身观察流维护时态任务状态视图

> 本文档是不考虑短期实现时间限制后的完整构思。  
> 目标：重新定义一个更契合 ICDE 的具身数据管理选题，避免落入“又一个 embodied memory / recall system”的叙事。  
> 推荐论文形态：优先 EAB（Experiment, Analysis, and Benchmark），同时保留 regular system paper 的可能性。  
> 推荐标题：`[Experiment, Analysis, and Benchmark] EviStateBench: Evaluating Temporal Task-State View Maintenance over Embodied Observation Streams`  
> 参考系统名：`EviStateDB`

---

## 0. 一句话结论

**EviStateDB 研究的问题不是机器人怎么记忆，而是数据系统如何从带噪声、延迟和冲突的具身观察流中，维护并查询任务状态视图。**

这里的“任务状态视图”指的是：

```text
open(drawer_1) = true during [t1, t2)
inside(cup_1, cabinet_1) = false during [t3, t4)
temperature(pan_1) = 84.6 at t5
goal_satisfied(task_7) = false because inside(cup_1, cabinet_1) is uncertain
precondition(open(drawer_1)) = violated before action pick(cup_1)
```

系统要回答的不是“我看过什么”，而是：

```text
某个状态现在是否成立？
某个状态过去是否成立？
状态什么时候变了？
为什么相信这个状态？
哪些观察支持它？哪些观察反驳它？
任务目标哪些已经满足？哪些还没满足？
下一步动作的前提是否满足？
哪些状态不确定，需要重新观察？
```

---

## 1. 为什么这个问题更适合 ICDE？

ICDE 关注的是 data engineering，而不是机器人控制策略、视觉识别模型或 embodied agent 规划方法。EviStateDB 的核心正好落在以下数据工程问题上：

```text
Data model:
  如何统一表示不同 arity、不同 value type 的具身任务状态。

Temporal semantics:
  如何表示状态成立的 valid time，以及 observation 到达系统的 transaction / arrival time。

Stream processing:
  observation 是连续到达的，系统必须在线维护状态视图。

Uncertain / probabilistic data:
  observation 可能错，状态置信度需要持续更新。

Provenance:
  每个状态判断需要能追溯 support / contradict evidence。

Incremental view maintenance:
  任务状态视图是 observation stream 上的 materialized views。

Query processing:
  CHECK / AS_OF / DIFF / WHY / GOAL / PRECONDITION 查询需要被高效执行。

Benchmarking:
  需要系统性评测不同 memory / log / DB baseline 在该 workload 下的表现。
```

ICDE 2027 CFP 明确覆盖 data models、query processing、storage/indexing、benchmarking、data streams、temporal/spatial data、uncertain/probabilistic data、provenance、graph/vector/multimodal data 和 domain-specific data engineering。EAB 类别也明确接受 benchmark / evaluation method，并要求可复现实验 artifact。

因此，EviStateDB 的安全表述不是：

```text
我们做一个 embodied memory system。
```

而是：

```text
我们定义并评测 embodied observation streams 上的 temporal task-state view maintenance workload，
并提供一个 reference data system。
```

---

## 2. 为什么要强调 noisy、delayed、conflicting observation？

这三个词不是装饰，而是 EviStateDB 成为数据工程问题的关键。

### 2.1 如果 observation 是干净的，问题会退化

如果系统直接读取 OmniGibson 的 ground-truth object states：

```text
open(drawer) = true
inside(cup, cabinet) = false
temperature(pan) = 84.6
```

那它只是把 simulator state 写进表。这样不需要复杂的数据模型、置信度、冲突处理或 provenance，ICDE 贡献会很弱。

### 2.2 Noisy：真实输入是状态证据，不是状态真值

具身 agent 的输入来自 detector、VLM、tracker、动作日志、深度估计、语言解释或仿真传感器。它们会产生：

```text
漏检
误检
错误物体类别
错误空间关系
错误开关状态
错误数值状态
置信度不校准
```

因此 observation 应该被建模为：

```text
在某个时间，某个来源，以某个置信度，声称某个状态成立或不成立。
```

不是：

```text
状态本身已经被确定。
```

### 2.3 Delayed：事件发生时间和系统知道时间不同

许多 embodied pipelines 是异步的：

```text
VLM 离线处理视频；
检测器晚于控制循环返回结果；
日志批量上传；
多机器人或多传感器数据乱序到达。
```

因此必须区分：

```text
event_time: 状态证据对应的真实世界时间；
arrival_time / transaction_time: 数据库收到或写入该证据的时间。
```

这直接对应 temporal database / streaming data 的问题。

### 2.4 Conflicting：状态证据之间会互相反驳

例子：

```text
obs_1: inside(cup, cabinet) = true, confidence 0.83
obs_2: ontop(cup, table) = true, confidence 0.76
obs_3: inside(cup, cabinet) = false, confidence 0.91
```

系统要判断：

```text
哪个状态当前成立？
哪个状态已失效？
旧状态的 valid interval 何时关闭？
哪些 observation 是 support？
哪些 observation 是 contradiction？
是否需要重新观察？
```

这不是普通 append-only log，也不是 retrieval memory，而是 uncertain temporal state-view maintenance。

---

## 3. 当前调研后的边界判断

### 3.1 不能声称“首次提出状态谓词”

BEHAVIOR / BDDL 已经用 first-order logic predicates 定义任务的 initial conditions 和 goal conditions。OmniGibson 也提供丰富 object states，例如 `Open`、`Inside`、`OnTop`、`Temperature`、`Saturated`、`IsGrasping` 等。

所以不能写：

```text
We introduce task predicates for embodied agents.
```

应该写：

```text
We use task predicates as data views to be maintained under noisy, delayed, and conflicting observations.
```

### 3.2 不能声称“首次从 observation 估计 symbolic state”

机器人领域已有 symbolic state estimation 工作，例如用 predicate classifiers 和 Bayesian state estimation 从 noisy multimodal observations 中估计 high-level symbolic states。

所以不能把贡献写成：

```text
We estimate symbolic states from observations.
```

应该写：

```text
We benchmark and analyze temporal state-view maintenance as a data management workload, including update semantics, query semantics, provenance, and baseline comparison.
```

### 3.3 不能声称“首次做 embodied memory database”

eMEM 已经非常接近 database-style embodied memory：它有 graph memory、Observation / Episode / Gist / Entity、SQLite、HNSW、R-tree、多层 consolidation 和 agent-facing recall tools。

所以不能写：

```text
We build the first queryable embodied memory database.
```

应该写：

```text
Unlike recall-oriented embodied memories, we maintain task-state views that answer whether a task predicate currently or historically holds, with support and contradiction evidence.
```

### 3.4 不能把 task-conditioned retrieval 当核心创新

STaR 已经做 task-conditioned retrieval from long-horizon multimodal memory。STAR 也做 open-world object retrieval with past states and embodied action loops。

所以不能把主实验设计成：

```text
Find the cup used earlier.
Retrieve memories relevant to this task.
```

主实验应该是：

```text
state tracking
as-of state query
state diff
goal satisfaction monitoring
precondition checking
failure state localization
support / contradiction provenance
```

---

## 4. 论文最终定位

### 4.1 推荐论文形态

最稳的形态：

```text
[Experiment, Analysis, and Benchmark] EviStateBench:
Evaluating Temporal Task-State View Maintenance over Embodied Observation Streams
```

理由：

```text
1. 该方向与已有 embodied memory / state estimation / planning 工作交叉较多；
2. 单纯 regular system novelty 风险较高；
3. ICDE EAB 明确接受新的 benchmark / evaluation method；
4. 这个问题目前缺少系统性数据工程 benchmark。
```

### 4.2 参考系统

系统可以叫：

```text
EviStateDB
```

它不是论文唯一贡献，而是 EviStateBench 的 reference engine。

### 4.3 核心一句话

英文：

```text
We benchmark and analyze how data systems maintain and query temporal task-state views from noisy, delayed, and conflicting embodied observation streams.
```

中文：

```text
我们评测和分析数据系统如何从带噪声、延迟和冲突的具身观察流中维护并查询时态任务状态视图。
```

---

## 5. 系统抽象

### 5.1 State Predicate Schema

每类状态先定义 schema：

```text
predicate_name
arity
argument_types
value_type
persistence_policy
contradiction_policy
```

例子：

```text
inside(object, container) -> boolean
ontop(object, surface) -> boolean
open(container) -> boolean
temperature(object) -> numeric
saturated(object, substance) -> boolean
gr apped(robot, object) -> boolean
objects_in_fov(robot) -> set<object>
pose(object) -> numeric tuple
```

注意：实际代码里应使用 `grasped`，上方 `gr apped` 是为了避免一些编辑器自动纠错时粘连；正式文档和代码都用 `grasped`。

### 5.2 State Observation

Observation 是状态证据，不是状态真值。

```text
StateObservation:
  obs_id
  event_time
  arrival_time
  source
  predicate_name
  arguments
  observed_value
  polarity: support / contradict / revise
  confidence
  evidence_ref
  raw_payload
```

例子：

```text
obs_17:
  event_time = 08:20
  arrival_time = 08:23
  source = visual_detector
  predicate = inside
  arguments = [cup_1, cabinet_1]
  observed_value = true
  polarity = support
  confidence = 0.81
  evidence_ref = frame_020
```

### 5.3 Temporal State View

系统维护 materialized temporal state views：

```text
TemporalStateView:
  state_id
  predicate_name
  arguments
  value
  valid_start
  valid_end
  transaction_start
  transaction_end
  confidence
  support_observations
  contradict_observations
  status: active / expired / contradicted / uncertain
  revision_history
```

例子：

```text
inside(cup_1, cabinet_1) = true
valid_time = [08:20, now)
transaction_time = [08:23, now)
confidence = 0.86
support = [obs_17, obs_18]
contradict = [obs_21]
status = active
```

### 5.4 Evidence Graph

除了状态表，还维护证据图：

```text
Observation -> supports -> StateView
Observation -> contradicts -> StateView
StateView -> revises -> StateView
StateView -> contributes_to -> GoalState
StateView -> violates -> Precondition
```

这使 WHY_STATE、failure diagnosis 和 provenance query 可以被系统化回答。

---

## 6. 查询接口

### 6.1 CHECK_STATE

```text
CHECK_STATE predicate(args) AT time
```

例子：

```text
CHECK_STATE inside(cup_1, cabinet_1) AT now
```

返回：

```text
value: true / false / unknown / numeric
confidence
valid_interval
support evidence
contradict evidence
status
```

### 6.2 AS_OF_STATE

```text
AS_OF_STATE open(drawer_1) AT 08:10
```

用于历史状态查询。

### 6.3 STATE_DIFF

```text
STATE_DIFF region=kitchen BETWEEN 08:00 AND now
```

返回：

```text
changed states
old values
new values
change intervals
support / contradict evidence
```

### 6.4 WHY_STATE

```text
WHY_STATE inside(cup_1, cabinet_1) = true
```

返回支持与反驳证据。

### 6.5 CHECK_GOAL

输入任务目标谓词集合：

```text
Goal:
  inside(cup_1, cabinet_1) = true
  open(cabinet_1) = false
```

输出：

```text
satisfied predicates
violated predicates
uncertain predicates
goal completion confidence
explanation
```

### 6.6 CHECK_PRECONDITION

输入动作前提集合：

```text
Action: place(cup_1, cabinet_1)
Preconditions:
  grasped(robot_1, cup_1) = true
  open(cabinet_1) = true
```

输出：

```text
safe_to_execute: true / false / uncertain
violated preconditions
uncertain preconditions
required reobservations
```

### 6.7 FIND_UNCERTAIN_STATES

```text
FIND_UNCERTAIN_STATES task_id threshold=0.6
```

用于决定下一步重新观察哪些关键状态。

---

## 7. EviStateDB 系统架构

### 7.1 Layer 1: Observation Ingestion

输入来自：

```text
OmniGibson state sampling
simulated noisy detectors
VLM/detector outputs
robot action logs
task execution logs
manual / annotation sources
```

统一转成 StateObservation。

### 7.2 Layer 2: State Normalization

负责：

```text
predicate schema validation
argument normalization
value type checking
source-specific confidence calibration
polarity detection
```

### 7.3 Layer 3: Temporal State View Maintenance

负责：

```text
support update
contradiction update
interval closure
late-arrival repair
numeric value revision
state expiry
uncertainty propagation
```

### 7.4 Layer 4: Evidence and Provenance Store

维护：

```text
support evidence
contradict evidence
revision history
source metadata
evidence references
```

### 7.5 Layer 5: Indexing

建议索引：

```text
predicate-argument index:
  predicate + args -> state views

temporal interval index:
  valid intervals for AS_OF / DIFF

evidence index:
  state_id -> support / contradict observations

confidence index:
  low-confidence states, top-k uncertain states

goal/precondition index:
  task_id / action_id -> relevant states
```

### 7.6 Layer 6: Query Engine

支持：

```text
CHECK_STATE
AS_OF_STATE
STATE_DIFF
WHY_STATE
CHECK_GOAL
CHECK_PRECONDITION
FIND_UNCERTAIN_STATES
```

---

## 8. 维护语义

### 8.1 支持证据更新

如果 observation 支持某个状态：

```text
p_new = 1 - (1 - p_old) * (1 - c_obs)
```

也可以替换为更原则化的 log-odds update。

### 8.2 反驳证据更新

如果 observation 反驳某个状态：

```text
p_new = p_old * (1 - gamma * c_obs)
```

当置信度低于阈值时：

```text
close valid interval
mark status = contradicted / expired
attach contradiction evidence
```

### 8.3 乱序观察修复

如果 observation 的 event_time 早于当前已维护 interval：

```text
1. 找到受影响的 state intervals；
2. 判断该 observation 支持还是反驳历史状态；
3. 分裂、合并或修正 valid intervals；
4. 保留 transaction_time，记录系统何时知道该修正。
```

### 8.4 数值状态维护

对于 temperature、pose 等 numeric state：

```text
use weighted update / smoothing
track error bounds
support AS_OF numeric query
report numeric error in evaluation
```

### 8.5 Goal / Precondition 视图

Goal 和 precondition 可以看作 derived views：

```text
GoalSatisfied(task, t) = conjunction / formula over state views
PreconditionSatisfied(action, t) = formula over state views
```

系统需要返回：

```text
true / false / uncertain
which predicate causes failure
which evidence supports the judgment
```

---

## 9. EviStateBench Benchmark 设计

### 9.1 主数据来源

主数据来源建议是：

```text
BEHAVIOR / OmniGibson-derived workload
```

原因：

```text
1. BEHAVIOR 任务天然有 BDDL initial / goal conditions；
2. OmniGibson 有丰富 object states；
3. simulator 可提供 ground-truth state timeline；
4. 可以系统性注入 noise / delay / conflict；
5. 可以覆盖 boolean、categorical、numeric、relation、robot-specific states。
```

系统抽象不能写死到 BEHAVIOR；BEHAVIOR 只是 benchmark source。

### 9.2 Predicate categories

Benchmark 至少覆盖：

```text
Unary boolean:
  open(drawer), cooked(food), frozen(food), heated(food), folded(cloth)

Binary relation:
  inside(object, container), ontop(object, surface), nextto(object, object), under(object, object)

Material / particle state:
  filled(container, substance), covered(object, substance), saturated(object, substance), contains(container, substance)

Robot-specific:
  grasped(robot, object), objects_in_fov(robot)

Numeric:
  temperature(object), pose(object), max_temperature(object)
```

### 9.3 Perturbation regimes

每个 episode 可生成多个 observation regime：

```text
clean:
  no noise, no delay

noisy:
  false positive, false negative, wrong value, wrong predicate

delayed:
  observation arrival_time > event_time

out-of-order:
  older event arrives after newer events

conflicting:
  mutually exclusive state claims both appear

missing:
  key states are unobserved for some intervals

mixed:
  all perturbations combined
```

### 9.4 Query workloads

```text
W1: CHECK_STATE
W2: AS_OF_STATE
W3: STATE_DIFF
W4: WHY_STATE
W5: CHECK_GOAL
W6: CHECK_PRECONDITION
W7: FIND_UNCERTAIN_STATES
W8: FAILURE_LOCALIZATION
```

### 9.5 Metrics

#### Correctness

```text
state truth accuracy
state interval F1
state diff precision / recall
goal satisfaction accuracy
precondition checking accuracy
failure state localization accuracy
numeric state error
```

#### Evidence quality

```text
support evidence precision
support evidence recall
contradict evidence precision
contradict evidence recall
why-query correctness
```

#### Robustness

```text
accuracy vs noise rate
accuracy vs delay rate
accuracy vs conflict rate
late-arrival repair accuracy
uncertain-state calibration
```

#### System performance

```text
update throughput
p50 / p95 query latency
memory footprint
index size
out-of-order repair cost
```

---

## 10. Baselines

### B1: Latest Observation

每个状态使用最新 observation 直接覆盖。

优点：简单、快速。  
缺点：噪声和误检下容易错；没有证据管理和历史 interval。

### B2: Temporal Log + Voting

保存所有 observations，查询时按时间窗口过滤并投票。

优点：有历史，能处理部分 AS_OF。  
缺点：每次查询重算，latency 高；处理 contradiction、late repair、WHY 查询较弱。

### B3: Static Symbolic State

只维护当前 symbolic state，不维护历史和 provenance。

优点：适合当前状态查询。  
缺点：AS_OF、DIFF、WHY、delayed correction 都弱。

### B4: Recall Memory Baseline

模拟 eMEM / STaR 类 recall memory：

```text
存 observations；
按语义、时间、空间过滤 top-k；
用 heuristic aggregation 回答 state query。
```

优点：对“我看过什么”强。  
缺点：没有 materialized state views，对 goal / precondition / state diff 需要临时推断。

### B5: SQL Scan Baseline

用 DuckDB / SQLite 存 observation log，每次查询 scan + group-by + heuristic。

优点：通用数据库基线，公平。  
缺点：缺少状态视图维护、专用 indexes、support/contradict semantics。

### B6: EviStateDB

维护 materialized temporal state views，支持 evidence、confidence、valid-time intervals、late-arrival repair 和 indexed queries。

---

## 11. 实验设计

### Experiment 1: State Tracking

问题：系统能否从 noisy observation stream 中维护正确状态？

输入：不同 predicate categories 的 observation stream。  
指标：

```text
state truth accuracy
state interval F1
numeric error
uncertain-state calibration
```

### Experiment 2: Robustness to Noise / Delay / Conflict

横轴：

```text
noise rate
delay rate
conflict rate
missing observation rate
```

纵轴：

```text
state accuracy
AS_OF accuracy
goal check accuracy
late repair accuracy
```

### Experiment 3: Goal Satisfaction Monitoring

输入：BEHAVIOR-style goal predicates。  
系统判断：

```text
satisfied / violated / uncertain
```

指标：

```text
goal satisfaction accuracy
false completed rate
false incomplete rate
uncertain goal rate
```

### Experiment 4: Precondition Checking

输入：动作前提谓词。  
系统判断下一步动作是否可执行。  
指标：

```text
precondition accuracy
unsafe-action false positive
unnecessary reobserve rate
```

### Experiment 5: State Diff / Restoration

比较 target time 和 current time。  
输出需要恢复的状态差异。  
指标：

```text
state diff precision
state diff recall
restore plan correctness
change-time error
```

### Experiment 6: WHY_STATE / Evidence Query

系统返回支持和反驳证据。  
指标：

```text
support evidence precision / recall
contradict evidence precision / recall
why-query correctness
```

### Experiment 7: Query Processing and Scalability

规模：

```text
number of tasks
number of objects
number of predicates
number of observations
number of state views
number of queries
```

指标：

```text
p50 / p95 latency
update throughput
memory footprint
index size
out-of-order repair cost
```

### Experiment 8: Ablation

Ablation variants：

```text
Full EviStateDB
No-valid-time
No-arrival-time / no-bitemporal
No-confidence
No-contradict-provenance
No-index
No-late-repair
```

---

## 12. BEHAVIOR / OmniGibson 是否足够？

### 12.1 构建系统：足够

BEHAVIOR / OmniGibson 足够构建第一版 EviStateDB / EviStateBench。

原因：

```text
1. BEHAVIOR 有 BDDL task definitions；
2. 每个任务有 initial / goal predicates；
3. OmniGibson 有丰富 object states；
4. simulator 能提供精确 ground truth；
5. 可以可控注入 noise、delay、conflict；
6. predicate 类型足够复杂，覆盖布尔、关系、数值、物质、机器人状态。
```

### 12.2 主实验：基本足够

如果把它做成系统化 benchmark，而不是几个 demo task，主实验可以主要基于 BEHAVIOR / OmniGibson。

需要覆盖：

```text
多任务
多 predicate 类型
多扰动 regime
多查询 workload
多 baseline
多规模设置
```

### 12.3 不应把系统写死到 BEHAVIOR

系统定义应该是通用的：

```text
predicate schema + state observation + temporal state view + evidence graph
```

BEHAVIOR 只是 benchmark generator 的来源。

### 12.4 建议加小规模外部验证

为了避免“只在仿真上成立”的质疑，可以加一个 small real-data validation：

```text
DROID small sample:
  转换 language / action / frames 为 weak state observations。

EPIC-KITCHENS annotations:
  转换 action segments 为 interaction-state observations。

OpenEQA episode histories:
  转换 history entries 为 textual evidence observations。
```

这部分不做主 correctness claim，只报告：

```text
ingestion feasibility
schema coverage
provenance examples
query examples
```

---

## 13. 可写成 regular paper 吗？

可以，但风险比 EAB 高。Regular paper 需要更明确的方法创新，例如：

```text
1. 支持 variable-arity typed predicates 的 uncertain temporal view maintenance algorithm；
2. 支持 late-arriving evidence 的 interval repair；
3. support / contradiction provenance-aware query processing；
4. 针对 CHECK_GOAL / CHECK_PRECONDITION 的 incremental derived-view maintenance。
```

如果只做 benchmark + 简单 reference engine，更适合 EAB。

推荐优先：

```text
EAB paper: EviStateBench + EviStateDB reference engine
```

备选：

```text
Regular paper: EviStateDB with stronger update/index algorithm
```

---

## 14. 建议论文结构

```text
1. Introduction
   动态具身任务为什么需要 temporal task-state views。

2. Background and Motivation
   BEHAVIOR / OmniGibson states；现有 memory / retrieval / state estimation 的不足。

3. Problem Definition
   observation stream、state predicate schema、temporal state view、query workload。

4. EviStateBench
   benchmark generation、predicate categories、perturbation regimes、query templates、metrics。

5. EviStateDB Reference System
   ingestion、normalization、view maintenance、evidence store、indexes、query engine。

6. Experimental Setup
   tasks、baselines、metrics、implementation details。

7. Evaluation
   state tracking、robustness、goal/precondition、evidence、scalability、ablation。

8. Related Work
   embodied memory, task-conditioned retrieval, symbolic state estimation, BEHAVIOR/OmniGibson, IVM/provenance.

9. Conclusion
```

---

## 15. 需要避免的过度 claim

不要写：

```text
We introduce task predicates for embodied agents.
We solve symbolic state estimation.
We build the first embodied memory database.
We outperform all embodied memory systems.
We solve real-world robot state tracking.
```

建议写：

```text
We define a data engineering benchmark for temporal task-state view maintenance over embodied observation streams.
We provide a unified state observation and temporal state view model for typed embodied predicates.
We analyze how recall memories, temporal logs, latest-observation baselines, SQL scans, and a reference materialized-view engine behave under noise, delay, and conflict.
We show which design choices are necessary for CHECK, AS_OF, DIFF, WHY, GOAL, and PRECONDITION queries.
```

---

## 16. 白话对比表

| 工作 / 系统 | 它主要在干什么 | 和我们相似的地方 | 我们怎么区分 |
|---|---|---|---|
| **eMEM** | 给 embodied agent 做可查询长期记忆库，能按语义、空间、时间 recall 记忆；有 SQLite、HNSW、R-tree 和 agent-facing recall tools | 都和长期具身记忆、查询、索引有关 | eMEM 关心“我看过什么、在哪里看过、怎么 recall”；EviStateDB 关心“任务状态是否成立、何时失效、由什么支持/反驳” |
| **STaR** | 从长期多模态记忆里检索和当前任务最相关的 memory subset | 都强调 task relevance | STaR 解决“检索哪些记忆给当前任务用”；EviStateDB 解决“当前任务状态视图如何被持续维护和查询” |
| **STAR** | 动态开放世界 object retrieval，支持过去状态、空间上下文和 embodied action loop | 都涉及过去状态和具身动作 | STAR 主任务是找对象；EviStateDB 主任务是维护 state views，做 goal/precondition/diff/why 查询 |
| **BEHAVIOR / OmniGibson** | 提供任务谓词、初始/目标条件、object states 和仿真 ground truth | 我们使用它们作为 benchmark 来源 | 它们提供状态和任务定义；EviStateDB 研究当状态只能通过 noisy/delayed/conflicting observations 得到时，数据系统如何维护视图 |
| **Symbolic State Estimation with Predicates** | 从 noisy multimodal observations 中估计 symbolic state，用于机器人 manipulation | 都涉及 noisy observation 和 symbolic state | 它是机器人状态估计算法；EviStateDB 是数据管理 benchmark / view maintenance / query provenance 系统 |
| **GSR / world-state reasoning 类工作** | 显式建模 world-state evolution、precondition、consequence、goal satisfaction | 都涉及状态、前提、目标 | 它们关注推理模型或 agent 规划；EviStateDB 关注状态视图作为数据对象如何被维护、查询、评测 |
| **GoalNet** | 从语言和示范中推断 conjunctive goal predicates，并交给 symbolic planner | 都涉及 goal predicates | GoalNet 研究目标谓词推断；EviStateDB 假设 goal predicates 给定，研究如何维护它们是否满足 |
| **DBSP / IVM 系统** | 通用数据流上的 incremental view maintenance | 都涉及 view maintenance | DBSP 是通用 IVM 框架；EviStateDB 是面向具身任务状态的 workload、typed state semantics、support/contradict provenance 和 benchmark |
| **SQL / DuckDB / SQLite scan baseline** | 把 observation log 存表，查询时 scan、filter、group-by | 都是数据系统方法 | SQL scan 能做通用查询，但没有专门的 temporal state view、late repair、evidence semantics 和 goal/precondition query support |
| **EviStateDB / EviStateBench** | 从具身观察流维护 temporal task-state views，并评测 CHECK、AS_OF、DIFF、WHY、GOAL、PRECONDITION workload | 与 embodied memory、state estimation、IVM 都有交叉 | 贡献落在 ICDE：数据模型、查询语义、benchmark、baseline 分析、reference system 和可复现实验 |

---

## 17. 相关论文与资源

### ICDE / 数据工程定位

1. **ICDE 2027 Call for Research Papers**  
   https://icde2027.github.io/cf-research-papers.html  
   说明 ICDE 覆盖 data models、query processing、benchmarking、data streams、temporal/spatial data、uncertain/probabilistic data、provenance、domain-specific data engineering 等方向；EAB 类别接受 benchmark / evaluation method。

2. **DBSP: Automatic Incremental View Maintenance for Rich Query Languages**  
   https://arxiv.org/abs/2203.16684  
   通用 incremental view maintenance 相关背景。EviStateDB 不应声称发明 IVM，而应定位为 embodied task-state view workload 与 reference system。

### BEHAVIOR / OmniGibson 状态与任务

3. **BEHAVIOR Important Concepts**  
   https://behavior.stanford.edu/getting_started/important_concepts.html  
   说明 BEHAVIOR task 是 1000+ long-horizon household activities 的 first-order logic formalization，每个任务由 BDDL 文件定义 object scope、initial conditions 和 goal conditions。

4. **OmniGibson Object States**  
   https://behavior.stanford.edu/omnigibson/object_states.html  
   列出 Open、Inside、OnTop、Temperature、Saturated、IsGrasping 等 object states，以及 get_value / set_value 接口。

5. **BEHAVIOR Task Definitions**  
   https://behavior.stanford.edu/behavior_components/behavior_tasks.html  
   可作为任务 predicate 和 goal/precondition workload 的来源。

### Embodied memory / retrieval 相关工作

6. **eMEM: A Hybrid Spatio-Temporal Memory System For Embodied Agents**  
   https://arxiv.org/abs/2606.03374  
   最接近 database-style embodied recall memory 的工作。用于说明 EviStateDB 不能只做 memory recall / multi-index。

7. **eMEM GitHub**  
   https://github.com/automatika-robotics/emem

8. **eMEM PyPI**  
   https://pypi.org/project/emem/

9. **STaR: Scalable Task-Conditioned Retrieval for Long-Horizon Multimodal Robot Memory**  
   https://arxiv.org/abs/2602.09255  
   做 task-conditioned long-horizon multimodal memory retrieval。用于区分 retrieval 和 state-view maintenance。

10. **STaR Project Page**  
    https://trailab.github.io/STaR-website/

11. **STaR GitHub**  
    https://github.com/TRAILab/STaR

12. **STAR: SpatioTemporal Active Retrieval for Object Retrieval in Dynamic Open Worlds**  
    https://arxiv.org/abs/2511.14004  
    做 object retrieval with past states / spatial context / active loop。用于说明 object-search 不是 EviStateDB 的主创新点。

### Symbolic state / goal / reasoning 相关工作

13. **Symbolic State Estimation with Predicates for Contact-Rich Manipulation Tasks**  
    https://arxiv.org/abs/2203.02468  
    从 noisy multimodal observations 估计 symbolic states。用于区分 robotics state estimation 和 data-system state-view maintenance。

14. **GoalNet: Inferring Conjunctive Goal Predicates from Demonstrations and Language**  
    https://arxiv.org/abs/2205.07081  
    从语言和示范中推断 goal predicates。EviStateDB 不做 goal inference，而是假设目标谓词给定，维护其是否满足。

15. **GSR: A General World Model for Embodied Reasoning**  
    https://arxiv.org/abs/2602.01693  
    显式建模 object states、spatial relations、preconditions、consequences 和 goal satisfaction。用于区分 embodied reasoning model 和 data management benchmark。

### 下游 / 可选真实验证

16. **OpenEQA: Embodied Question Answering in the Era of Foundation Models**  
    https://open-eqa.github.io/  
    可作为下游 QA / evidence retrieval 外部验证，不适合作为主 state-view correctness benchmark。

17. **DROID Dataset**  
    https://droid-dataset.github.io/  
    可作为真实机器人 observation ingestion 小样本验证。

18. **EPIC-KITCHENS / VISOR**  
    https://epic-kitchens.github.io/  
    可作为 egocentric dynamic interaction annotation 的外部验证来源。

---

## 18. 最终判断

EviStateDB / EviStateBench 可以作为 ICDE 选题，但必须遵守以下边界：

```text
1. 不讲 embodied memory database；
2. 不讲 predicate 本身是创新；
3. 不讲 symbolic state estimation 是创新；
4. 不讲 task-conditioned retrieval 是创新；
5. 主体讲 temporal task-state view maintenance workload；
6. 贡献放在 data model、query semantics、benchmark、baseline analysis、reference system；
7. 主实验使用 BEHAVIOR / OmniGibson-derived workload；
8. 可加小规模 real-data validation，但不替代主实验；
9. 论文形态优先 EAB，regular 需要更强算法贡献。
```

最稳的最终题目仍然是：

```text
[Experiment, Analysis, and Benchmark] EviStateBench:
Evaluating Temporal Task-State View Maintenance over Embodied Observation Streams
```

如果转成 regular system paper，则题目可以是：

```text
EviStateDB: Evidence-Aware Temporal State View Maintenance for Embodied Observation Streams
```
