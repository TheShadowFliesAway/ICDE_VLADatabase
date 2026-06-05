# 08 Final Frozen Spec: EviStateBench / EviStateDB

> 状态：定稿版。  
> 后续只允许补实现细节、实验数字、引用和图表，不再改变问题中心。  
> 论文形态：ICDE EAB 优先。  
> 论文主角：EviStateBench。  
> Reference engine：EviStateDB。

---

## 1. 最终标题

```text
[Experiment, Analysis, and Benchmark] EviStateBench:
Evaluating Temporal Task-State View Maintenance over Embodied Observation Streams
```

参考系统名：

```text
EviStateDB
```

---

## 2. 最终一句话

**EviStateBench 是一个面向数据工程的 benchmark，用来评测数据系统如何从带噪声、延迟和冲突的具身观察流中，维护并查询时态任务状态视图。**

**EviStateDB 是该 benchmark 的 reference materialized-view engine。**

---

## 3. 以后不再改变的核心定位

本项目不再定位为：

```text
embodied memory database
long-term robot memory system
task-conditioned retrieval system
symbolic state estimation system
robot planning system
VLM / perception system
```

最终定位固定为：

```text
temporal task-state view maintenance benchmark + reference engine
```

也就是说，论文不是说“我们做了一个新的机器人记忆系统”，而是说：

```text
已有 embodied memory、state estimation、goal reasoning 和 retrieval 方法很多，
但缺少一个面向数据系统社区的 benchmark，系统性评测：
在 noisy / delayed / conflicting embodied observation streams 上，
如何维护 temporal task-state views，
如何回答 CHECK / AS_OF / DIFF / WHY / GOAL / PRECONDITION queries，
以及不同 baseline 在这个 workload 下各自失败在哪里。
```

---

## 4. 视图的最终定义

这是最重要的定义，后续不再改。

### 4.1 Observation Log

Observation 是原始证据，不是 view。

```text
StateObservation(
  obs_id,
  event_time,
  arrival_time,
  source,
  predicate_name,
  arguments,
  observed_value,
  confidence,
  polarity,
  evidence_ref
)
```

它的含义是：

```text
某个来源在某个时间以某个置信度声称某个任务状态成立、不成立或需要修正。
```

### 4.2 Temporal State View

Temporal State View 是从 Observation Log 派生出来的、物化维护的时态状态关系。

```text
TemporalStateView(
  state_id,
  predicate_name,
  arguments,
  value,
  valid_start,
  valid_end,
  transaction_start,
  transaction_end,
  confidence,
  support_observations,
  contradict_observations,
  status,
  revision_history
)
```

它的含义是：

```text
系统在某个有效时间区间内，对某个任务状态的维护判断。
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

### 4.3 Task Views

Task Views 是从 Temporal State Views 再派生出来的任务级视图。

```text
GoalView(task_id, time)
PreconditionView(action_id, time)
StateDiffView(scope, t1, t2)
FailureView(episode_id)
UncertainStateView(task_id)
```

例子：

```text
GoalView(task_7, now):
  satisfied = false
  satisfied_predicates = [inside(cup_1, cabinet_1)]
  violated_predicates = [open(cabinet_1)]
  uncertain_predicates = [grasped(robot_1, cup_1)]
  evidence = [...]
```

### 4.4 和 memory 的最终区别

```text
Memory stores or retrieves observations.
EviStateDB maintains materialized task-state views derived from observations.
```

白话版：

```text
memory 像监控录像库，回答“我看过什么”；
state view 像库存系统，回答“业务状态现在是什么、什么时候变了、为什么相信”。
```

---

## 5. 为什么 noisy / delayed / conflicting 是必要条件

这三个条件不是修辞，而是把问题从“读取仿真状态”变成“数据工程问题”的关键。

### 5.1 Noisy

真实 observation 来自 detector、VLM、tracker、动作日志、深度估计、语言解释或仿真传感器，可能误检、漏检、错判关系或数值偏差。因此系统维护的是 uncertain state views，而不是直接读取真值。

### 5.2 Delayed

具身系统常有异步 pipeline。一个状态证据可能发生在 08:05，但 08:30 才被数据库收到。因此必须区分：

```text
event_time / valid_time
arrival_time / transaction_time
```

### 5.3 Conflicting

不同观察会互相反驳。系统必须维护 support 和 contradict evidence，并能关闭旧 interval、修正历史状态、回答 WHY_STATE。

如果没有这三者，问题会退化为：

```text
从 OmniGibson ground truth 读状态，然后写表。
```

那不是 ICDE 贡献。

---

## 6. 最终贡献点

### Contribution 1: Benchmark Problem

提出并形式化：

```text
Temporal Task-State View Maintenance over Embodied Observation Streams
```

这是论文主贡献。

### Contribution 2: State Observation / State View Model

定义统一数据模型，支持：

```text
variable-arity predicates
boolean / categorical / numeric values
valid time
arrival / transaction time
confidence
support provenance
contradiction provenance
revision history
```

### Contribution 3: Query Workload

定义并实现以下 query templates：

```text
CHECK_STATE
AS_OF_STATE
STATE_DIFF
WHY_STATE
CHECK_GOAL
CHECK_PRECONDITION
FIND_UNCERTAIN_STATES
FAILURE_LOCALIZATION
```

### Contribution 4: EviStateBench Generator

基于 BEHAVIOR / OmniGibson-derived predicates 和 object states，生成：

```text
ground-truth state timelines
noisy observations
delayed / out-of-order observations
conflicting observations
query sets
ground-truth answers
```

### Contribution 5: Baseline Analysis + Reference System

实现代表性 baseline 和 EviStateDB reference engine，系统性分析它们在不同 perturbation regimes、query workloads 和 scale 下的表现。

---

## 7. 数据源最终决策

### 7.1 主实验

主实验使用：

```text
BEHAVIOR / OmniGibson-derived EviStateBench
```

原因：

```text
1. BEHAVIOR / BDDL 有 initial conditions 和 goal conditions；
2. OmniGibson 有丰富 object states；
3. simulator 可以给出 ground-truth state timeline；
4. 可以系统性注入 noise、delay、conflict；
5. 能覆盖多种 predicate 类型和任务视图查询。
```

### 7.2 外部验证

可选加一个小节：

```text
External Data Validation
```

候选数据：

```text
DROID small sample
EPIC-KITCHENS annotations
OpenEQA episode histories
```

这部分只证明：

```text
schema can ingest real embodied observations;
provenance can point to real frames / annotations;
query interface can run on real data.
```

不承担主 correctness claim。

---

## 8. Benchmark 内容最终版

### Predicate Categories

```text
Unary boolean:
  open(drawer), cooked(food), frozen(food), folded(cloth)

Binary / spatial relation:
  inside(object, container), ontop(object, surface), nextto(object, object), under(object, object)

Material / particle state:
  filled(container, substance), covered(object, substance), saturated(object, substance), contains(container, substance)

Robot-specific:
  grasped(robot, object), objects_in_fov(robot)

Numeric:
  temperature(object), pose(object), max_temperature(object)
```

### Perturbation Regimes

```text
clean
noisy
delayed
out-of-order
conflicting
missing
mixed
```

### Query Workloads

```text
W1 CHECK_STATE
W2 AS_OF_STATE
W3 STATE_DIFF
W4 WHY_STATE
W5 CHECK_GOAL
W6 CHECK_PRECONDITION
W7 FIND_UNCERTAIN_STATES
W8 FAILURE_LOCALIZATION
```

### Metrics

Correctness:

```text
state truth accuracy
state interval F1
state diff precision / recall
goal satisfaction accuracy
precondition checking accuracy
failure state localization accuracy
numeric state error
```

Evidence:

```text
support evidence precision / recall
contradict evidence precision / recall
why-query correctness
```

Robustness:

```text
accuracy vs noise rate
accuracy vs delay rate
accuracy vs conflict rate
late-arrival repair accuracy
uncertain-state calibration
```

System:

```text
update throughput
p50 / p95 query latency
memory footprint
index size
out-of-order repair cost
```

---

## 9. Baselines 最终版

### B1 Latest Observation

每个状态直接采用最新 observation。

### B2 Temporal Log + Voting

保存所有 observations，查询时按时间窗口过滤并投票。

### B3 Static Symbolic State

只维护当前 symbolic state，不维护 history / provenance。

### B4 Recall Memory Baseline

模拟 eMEM / STaR 类型系统：存 observation，按语义/时间/空间检索 top-k，再用 heuristic 聚合回答 state query。

### B5 SQL Scan Baseline

用 DuckDB / SQLite 存 observation log，每次查询 scan + group-by + heuristic。

### B6 EviStateDB

维护 materialized temporal state views、support/contradict evidence、valid-time intervals、arrival-time repair 和 indexes。

---

## 10. EviStateDB Reference Engine 最终职责

EviStateDB 不作为“最强系统”主张，而作为 benchmark 的 reference engine。

它实现：

```text
1. StateObservation ingestion
2. predicate schema validation
3. confidence update
4. support / contradiction provenance tracking
5. valid-time interval maintenance
6. delayed / out-of-order repair
7. derived goal / precondition views
8. indexes for predicate-args, time, evidence, confidence, task
9. CHECK / AS_OF / DIFF / WHY / GOAL / PRECONDITION queries
```

---

## 11. 论文实验最终版

### Experiment 1: State Tracking

回答：不同方法维护基础状态视图的准确性如何？

### Experiment 2: Robustness

回答：noise、delay、conflict、missingness 如何影响不同方法？

### Experiment 3: Goal Satisfaction Monitoring

回答：不同方法能否正确判断任务目标是否满足？

### Experiment 4: Precondition Checking

回答：不同方法能否正确判断下一步动作前提？

### Experiment 5: State Diff / Restoration

回答：不同方法能否正确找出两个时间点之间的状态差异？

### Experiment 6: WHY_STATE / Evidence Query

回答：不同方法能否返回正确支持和反驳证据？

### Experiment 7: Query Processing and Scalability

回答：materialized state views 相比 log scan 的 latency / update trade-off 如何？

### Experiment 8: Ablation

回答：valid time、arrival time、confidence、contradict provenance、index、late repair 各自是否必要？

---

## 12. 论文结构最终版

```text
1. Introduction
2. Background and Motivation
3. Problem Definition
4. EviStateBench Benchmark
5. EviStateDB Reference Engine
6. Experimental Setup
7. Evaluation and Analysis
8. Lessons Learned
9. Related Work
10. Conclusion
```

EAB 论文必须有：

```text
Benchmark design
Baseline taxonomy
Systematic analysis
Lessons learned
Reproducible artifact
```

---

## 13. Lessons Learned 目标

最终论文至少产出这些分析结论：

```text
L1. Recall-memory methods can retrieve evidence, but do not reliably maintain goal/precondition state views.
L2. Latest-observation methods are fast but fragile under noisy and conflicting evidence.
L3. Temporal-log methods can recover history but pay high query cost and handle late repair poorly.
L4. Bitemporal semantics matter most under delayed and out-of-order observations.
L5. Support/contradict provenance is necessary for WHY_STATE and failure localization.
L6. Materialized state views trade update overhead for lower query latency and better goal/precondition correctness.
```

---

## 14. 需要避免的 claim

不要写：

```text
We introduce task predicates for embodied agents.
We solve symbolic state estimation.
We build the first embodied memory database.
We outperform all embodied memory systems.
We solve real-world robot state tracking.
```

固定写法：

```text
We introduce a benchmark and reference engine for temporal task-state view maintenance over embodied observation streams.
```

---

## 15. 白话对比表

| 工作 / 系统 | 它主要在干什么 | 容易和我们撞的点 | 最终区分 |
|---|---|---|---|
| eMEM | 给 embodied agent 做可查询长期记忆库，按语义、空间、时间 recall 记忆 | 长期记忆、查询、索引 | eMEM 问“我看过什么”；EviStateBench 问“任务状态是否成立，何时失效，由什么支持/反驳” |
| STaR | 从长期多模态记忆里检索与当前任务相关的 memory subset | task-conditioned memory | STaR 做 retrieval；我们做 materialized task-state view maintenance |
| STAR | 动态开放世界物体检索，支持 past-state object retrieval | 过去状态、具身动作 | STAR 主任务是找对象；我们主任务是 goal/precondition/diff/why 查询 |
| BEHAVIOR / OmniGibson | 提供任务谓词、目标条件、object states 和仿真真值 | 我们使用它们 | 它们是 benchmark source；EviStateBench 研究 noisy/delayed/conflicting observations 下如何维护视图 |
| Symbolic State Estimation | 从 noisy multimodal observations 估计 symbolic state | noisy observation + symbolic state | 它是机器人状态估计算法；我们是数据系统 benchmark / view maintenance / query provenance |
| GSR / world-state reasoning | 建模 world-state evolution、preconditions、goal satisfaction | 状态、前提、目标 | 它们关注 reasoning / agent；我们关注状态视图作为数据对象如何被维护、查询和评测 |
| GoalNet | 从语言和示范推断 goal predicates | goal predicates | 我们不推断 goal；我们维护 goal predicates 是否满足 |
| DBSP / IVM | 通用 incremental view maintenance | view maintenance | 我们不发明 IVM；我们提出 embodied task-state view workload 和 support/contradict evidence benchmark |
| SQL / DuckDB / SQLite scan | 把 observation log 存表后查询 | 通用数据系统 baseline | 它们没有专门 temporal state views、late repair、goal/precondition view 和 support/contradict semantics |
| EviStateBench / EviStateDB | 评测并参考实现 temporal task-state view maintenance | 与 memory、state estimation、IVM 都有交叉 | 贡献固定在 ICDE：benchmark、data model、query workload、baseline analysis、reference engine、artifact |

---

## 16. 最终判断

这是最终版本：

```text
EviStateBench 是论文主角；
EviStateDB 是 reference engine；
核心问题是 temporal task-state view maintenance；
主实验来自 BEHAVIOR / OmniGibson-derived benchmark；
论文形态优先 ICDE EAB；
不再使用 LifelongSceneDB / embodied memory database 作为主定位。
```
