# LifelongSceneDB 下一版本计划：从 MVP 到 Paper-Ready Prototype

> 目标：如果今晚 MVP 验收标准基本完成，下一版本不要继续堆功能，而是把系统推进到能支撑 ICDE 论文实验和写作的状态。  
> 建议版本名：`v0.2-paper-ready-prototype`  
> 核心原则：**可复现、可对比、可画图、可写论文。**

---

## 0. 当前 MVP 的假设完成状态

假设今天已经完成或接近完成：

```text
1. toy case 能跑通；
2. synthetic episode generator 能生成动态事件；
3. LifelongSceneDB 能 ingest noisy observation stream；
4. 支持 current / as-of / change / provenance / top-k 查询；
5. 至少有几个 baseline：static graph、latest-wins、vector/text retrieval、no-confidence；
6. 能输出 metrics.csv；
7. 能画出基础图。
```

如果以上没有全部完成，不要进入下一版本。先补齐 MVP。

---

## 1. 下一版本总目标

下一版本的目标不是“大幅扩展功能”，而是让系统能支撑论文中的四类 claim：

```text
Claim 1: 数据模型有效
  probabilistic + bitemporal + provenance 的表示能更正确回答 embodied memory queries。

Claim 2: 更新机制有效
  在 noisy / delayed / conflicting observations 下比 latest-wins 或 static memory 更稳。

Claim 3: 查询处理有效
  hybrid indexes 比 full scan 或简单 baseline 有更低 latency。

Claim 4: 对具身任务有效
  memory backend 能减少 object search 和 previous-state restoration 的搜索成本或错误动作。
```

最终交付物：

```text
1. 稳定代码版本；
2. 一键复现实验脚本；
3. 可写进论文的 CSV 结果；
4. 可直接插入论文的 figures；
5. paper 中 Evaluation section 的初版材料。
```

---

## 2. 版本边界：下一版本做什么，不做什么

### 2.1 必做

```text
1. 固化数据格式和 query schema；
2. 增强 synthetic dynamic workload；
3. 补齐 baseline 和 ablation；
4. 补齐 robustness sweep；
5. 补齐 scaling/latency 实验；
6. 补齐两个 task-level 实验；
7. 输出标准化表格和图；
8. 更新论文的 Evaluation Setup 和 Preliminary Results。
```

### 2.2 暂不做

```text
1. 不接真实机器人；
2. 不训练 VLM / LLM / policy；
3. 不做完整 SQL parser；
4. 不接完整 BEHAVIOR-1K / Habitat pipeline；
5. 不做复杂物理仿真；
6. 不做多机分布式；
7. 不做复杂 Bayesian inference；
8. 不做 production-grade DBMS。
```

原因：当前目标是 ICDE paper prototype，不是完整机器人产品。

---

## 3. 代码结构目标

建议整理成如下结构：

```text
lifelongscenedb/
  __init__.py
  schema.py
  store.py
  engine.py
  updater.py
  confidence.py
  queries.py
  indexes.py
  baselines.py
  generator.py
  metrics.py
  tasks.py
  io.py
  utils.py

scripts/
  run_toy_case.py
  generate_benchmark.py
  run_query_eval.py
  run_noise_sweep.py
  run_delay_sweep.py
  run_scaling_eval.py
  run_ablation.py
  run_task_object_search.py
  run_task_restore_state.py
  plot_results.py

configs/
  default.yaml
  small.yaml
  paper.yaml

tests/
  test_toy_case.py
  test_queries.py
  test_bitemporal.py
  test_confidence.py
  test_provenance.py
  test_baselines.py

outputs/
  .gitkeep

figures/
  .gitkeep
```

---

## 4. 数据与 workload 下一版本要求

### 4.1 Episode generator 增强

当前如果只支持简单 `MOVE`，下一版本要至少支持：

```text
MOVE(object, from, to, time)
OPEN(container, time)
CLOSE(container, time)
HIDE(object, occluder, time)
REVEAL(object, time)
HUMAN_MOVE(object, from, to, time)
ROBOT_MOVE(object, from, to, time)
ROBOT_OBSERVE(region, time)
FALSE_DETECT(object, wrong_location, time)
MISS_DETECT(object, time)
DELAY_OBSERVATION(event_time, arrival_time)
```

这些事件不需要物理仿真，只需要能生成 ground truth 和 observation stream。

### 4.2 场景规模参数化

需要支持命令行参数：

```bash
python scripts/generate_benchmark.py \
  --episodes 1000 \
  --objects-per-scene 20 \
  --events-per-episode 30 \
  --noise 0.2 \
  --delay 0.1 \
  --seed 42 \
  --out data/paper_1k
```

必须支持至少以下规模：

```text
small: 100 episodes
medium: 1000 episodes
large: 5000 episodes 或 1e6 facts 级别
```

### 4.3 Query generator 增强

每个 episode 自动生成：

```text
Current query:
  Where is object now?

AS-OF query:
  Where was object at time t?

Change query:
  What changed in region during time range?

Provenance query:
  Which observations support claim?

Top-k uncertain query:
  What are the most likely locations of object?

Task query:
  Find object used earlier.
  Restore room to state before target time.
```

每个 query 必须有 ground-truth answer。

---

## 5. Baseline 下一版本要求

### 5.1 必做 baseline

```text
B1 StaticSceneGraph
  只保留每个对象的最新状态，不保留历史。

B2 LatestWinsTemporalMemory
  保留时间历史，但最新 observation 直接覆盖旧 belief。

B3 NoConfidenceTemporalMemory
  有时间区间，但不用 confidence fusion。

B4 NoBitemporalMemory
  只有单一 timestamp，不区分 event_time 和 arrival_time。

B5 TextRetrievalMemory / VectorLikeMemory
  用简单文本匹配或 synthetic embedding 检索相关 observation。

B6 LifelongSceneDB-NoIndex
  使用完整数据模型，但查询全表扫描。

B7 Full LifelongSceneDB
  完整系统。
```

### 5.2 可选 baseline

```text
B8 DuckDBScanBaseline
  把 facts 存成表，使用 SQL scan 模拟通用数据库 baseline。

B9 Neo4jStyleGraphBaseline
  不需要真的接 Neo4j，可以实现 graph adjacency + vector/text filter 的模拟 baseline。
```

下一版本优先级：先做 B1-B7。B8/B9 如果时间够再做。

---

## 6. Ablation 下一版本要求

至少支持以下开关：

```text
--no-bitemporal
--no-confidence
--no-provenance
--no-index
--no-conflict-resolution
--no-expiry
```

Ablation 目标：证明每个设计不是装饰。

预期实验：

```bash
python scripts/run_ablation.py \
  --data data/paper_1k \
  --out outputs/ablation
```

输出：

```text
outputs/ablation/metrics.csv
outputs/ablation/summary.md
figures/ablation_accuracy.pdf
figures/ablation_latency.pdf
```

---

## 7. 实验脚本下一版本要求

### 7.1 Query evaluation

```bash
python scripts/run_query_eval.py \
  --data data/paper_1k \
  --methods full,static,latest,no_conf,text \
  --out outputs/query_eval
```

输出指标：

```text
current_accuracy
asof_accuracy
change_f1
provenance_precision
provenance_recall
stale_fact_rate
conflict_resolution_accuracy
mean_latency_ms
p95_latency_ms
memory_mb
```

### 7.2 Noise sweep

```bash
python scripts/run_noise_sweep.py \
  --episodes 1000 \
  --noise-levels 0,0.1,0.2,0.3,0.4 \
  --delay 0.1 \
  --seed 42 \
  --out outputs/noise_sweep
```

用于论文图：

```text
Accuracy vs Observation Noise Rate
```

### 7.3 Delay sweep

```bash
python scripts/run_delay_sweep.py \
  --episodes 1000 \
  --delay-rates 0,0.05,0.1,0.2,0.4 \
  --noise 0.2 \
  --seed 42 \
  --out outputs/delay_sweep
```

用于论文图：

```text
AS-OF Accuracy vs Delayed / Out-of-order Observation Rate
```

### 7.4 Scaling evaluation

```bash
python scripts/run_scaling_eval.py \
  --fact-scales 10000,100000,1000000 \
  --queries 1000 \
  --out outputs/scaling
```

用于论文图：

```text
p95 Query Latency vs Number of Facts
```

### 7.5 Task-level object search

```bash
python scripts/run_task_object_search.py \
  --data data/paper_1k \
  --methods full,static,latest,text \
  --out outputs/task_object_search
```

指标：

```text
object_localization_success
search_steps
wrong_location_visits
reobserve_count
task_success
```

### 7.6 Task-level restore previous state

```bash
python scripts/run_task_restore_state.py \
  --data data/paper_1k \
  --methods full,static,latest,no_bitemporal \
  --out outputs/task_restore
```

指标：

```text
restoration_accuracy
correctly_restored_objects
wrong_placement_rate
unnecessary_moves
```

---

## 8. 图表下一版本必须产出

下一版本至少产出 6 张图 / 表。

### Figure 1: System Architecture

论文图，不一定由实验脚本生成。

内容：

```text
Observation Stream
  -> Fact Extraction
  -> Probabilistic Bitemporal Scene Graph
  -> Hybrid Indexes
  -> Query API
  -> Robot Planner / User
```

### Figure 2: Accuracy vs Noise Rate

证明 confidence / conflict update 有用。

### Figure 3: AS-OF Accuracy vs Delay Rate

证明 bitemporal 有用。

### Figure 4: Query Latency vs Number of Facts

证明 index 有用。

### Figure 5: Ablation Results

证明 bitemporal、confidence、provenance、index 各自有贡献。

### Figure 6 / Table 1: Task-level Results

证明 object search 和 previous-state restoration 有实际具身任务收益。

---

## 9. 论文需要同步更新的内容

下一版本代码完成后，论文 `conference_101719.tex` 需要同步更新：

```text
1. Abstract:
   用真实实验数字替换 currently tentative wording。

2. Introduction:
   把贡献写得更硬，补上实验结果数字。

3. Experimental Evaluation:
   补数据规模、baseline、metrics、图表。

4. Related Work:
   修正 GraphPad 等 placeholder 引用。

5. Conclusion:
   从 planned evaluation 改为 actual findings。
```

特别注意：当前 abstract 里有一句：

```text
The results will show whether ...
```

下一版本有实验结果后要改成：

```text
Our results show that ...
```

并补具体数字。

---

## 10. 下一版本验收标准

### 10.1 代码验收

必须满足：

```bash
pytest -q
python scripts/run_toy_case.py --out outputs/toy_case
python scripts/generate_benchmark.py --episodes 1000 --noise 0.2 --delay 0.1 --seed 42 --out data/paper_1k
python scripts/run_query_eval.py --data data/paper_1k --out outputs/query_eval
python scripts/run_noise_sweep.py --episodes 1000 --noise-levels 0,0.1,0.2,0.3,0.4 --delay 0.1 --seed 42 --out outputs/noise_sweep
python scripts/run_delay_sweep.py --episodes 1000 --delay-rates 0,0.05,0.1,0.2,0.4 --noise 0.2 --seed 42 --out outputs/delay_sweep
python scripts/run_scaling_eval.py --fact-scales 10000,100000,1000000 --queries 1000 --out outputs/scaling
python scripts/run_ablation.py --data data/paper_1k --out outputs/ablation
python scripts/run_task_object_search.py --data data/paper_1k --out outputs/task_object_search
python scripts/run_task_restore_state.py --data data/paper_1k --out outputs/task_restore
python scripts/plot_results.py --input outputs --out figures
```

### 10.2 实验验收

必须有：

```text
outputs/query_eval/metrics.csv
outputs/noise_sweep/metrics.csv
outputs/delay_sweep/metrics.csv
outputs/scaling/metrics.csv
outputs/ablation/metrics.csv
outputs/task_object_search/metrics.csv
outputs/task_restore/metrics.csv
figures/*.pdf 或 figures/*.png
```

### 10.3 论文验收

必须有：

```text
1. Evaluation section 不再是 TODO；
2. 至少 4 张实验图进入论文；
3. 至少 1 张 schema/system table；
4. abstract 中出现真实实验结论；
5. refs.bib 中不再有 Anonymous or Unknown Authors；
6. conference_101719.tex 能编译通过。
```

---

## 11. 推荐执行顺序

### Step 1: Stabilize data format

先不要加功能，先固定 `Observation`、`Fact`、`Query`、`Answer` 的 JSON / CSV schema。

### Step 2: Strengthen generator

把事件、噪声、delay、query、answer 全部做成可控参数。

### Step 3: Add baselines and ablations

保证每个 baseline 和 ablation 都走同一套 query evaluator。

### Step 4: Add scaling and latency

至少跑到 `1e6 facts`，如果太慢就先跑 `1e5`，但脚本要支持 `1e6`。

### Step 5: Add task-level simulation

先做抽象任务，不做真实机器人。

### Step 6: Freeze results

一旦得到稳定结果，保存：

```text
random seed
config yaml
metrics csv
figures
run logs
```

### Step 7: Update paper

用真实结果替换 TODO 和 speculative wording。

---

## 12. 给 Codex 的下一版本提示词

可以直接把下面这段给 Codex：

```text
We already have a working MVP of LifelongSceneDB. The next goal is to make it a paper-ready prototype for an ICDE-style evaluation. Do not add unrelated features. Implement the v0.2 plan in docs/codex/05_NEXT_VERSION_PLAN.md. Focus on reproducible experiments, baselines, ablations, metrics, and figures. Preserve the current public API unless a change is necessary. Add tests for bitemporal queries, confidence updates, provenance retrieval, and query correctness. Ensure every experiment writes metrics.csv and every plot script reads CSV outputs rather than recomputing experiments.
```

---

## 13. 重要提醒

下一版本最重要的不是“功能更多”，而是：

```text
1. 结果可信；
2. baseline 公平；
3. ablation 清楚；
4. 图能放进论文；
5. 脚本别人能复现。
```

如果时间紧，优先级如下：

```text
P0: query_eval + noise_sweep + delay_sweep + ablation
P1: scaling_eval
P2: task_object_search + task_restore
P3: DuckDB / Neo4j-style baseline
P4: OpenEQA / Habitat / BEHAVIOR 接入
```

当前阶段只需要完成 P0-P2，就足够支撑论文主体实验。
