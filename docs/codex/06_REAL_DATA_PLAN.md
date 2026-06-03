# 06 Real Data Plan: From Synthetic MVP to Real-Data Validation

> 目标：在 synthetic MVP 已经打通后，规划真实数据接入。  
> 重点：真实数据不是立刻替代 synthetic main benchmark，而是补强论文的 external validity。  
> 结论：今晚可以先下载 OpenEQA metadata / code、DROID 小样本、EPIC-KITCHENS annotations 或 manifest；不要盲目下载全量 DROID、BridgeData、EPIC 视频。

---

## 0. 为什么需要 real-data plan

当前 synthetic workload 的优势是：

```text
1. 有精确 ground truth；
2. 可以控制 noise / delay / out-of-order；
3. 可以系统性验证 bitemporal、confidence、provenance 和 indexing；
4. 适合作为 ICDE 主实验。
```

但 synthetic workload 的风险是：

```text
1. 审稿人可能质疑是不是过于人造；
2. embodied 场景的真实性不足；
3. 真实图像、真实语言指令、真实轨迹没有进入系统。
```

因此需要真实数据验证，但要明确：

```text
Synthetic benchmark = 主实验，用来验证数据库语义和算法；
Real data = 外部验证，用来证明系统能接真实 embodied observations。
```

不要为了真实数据牺牲主实验的可控性。

---

## 1. 真实数据源选择

### 1.1 首选今晚下载：OpenEQA

#### 数据定位

OpenEQA 是 open-vocabulary embodied question answering benchmark。它包含 1600+ human-generated question-answer pairs 和 episode histories，覆盖 180+ real-world environments，并支持 episodic memory 和 active exploration settings。

#### 适合我们的用途

```text
用途：下游 embodied QA validation。
不是用于证明 bitemporal / change / provenance 数据库正确性的主数据。
```

可以用它测试：

```text
1. LifelongSceneDB 能否把 episode history 转成结构化 memory context；
2. 和 vector memory / raw frame retrieval 相比，是否能减少 retrieved frames 或 token cost；
3. 对 where / what / attribute / relation 类问题是否能返回更结构化证据。
```

#### 今晚下载什么

```bash
git clone https://github.com/facebookresearch/open-eqa.git data/raw/open-eqa
```

或者只下载 QA json：

```bash
mkdir -p data/raw/open-eqa
wget -O data/raw/open-eqa/open-eqa-v0.json \
  https://raw.githubusercontent.com/facebookresearch/open-eqa/main/data/open-eqa-v0.json
```

#### 不要今晚做什么

```text
1. 不要今晚跑完整 GPT-4 / VLM evaluation；
2. 不要把 OpenEQA 当作 current/as-of/change query 的主 benchmark；
3. 不要假装 OpenEQA 有完整动态世界状态 ground truth。
```

---

### 1.2 首选真实机器人样本：DROID

#### 数据定位

DROID 是 large-scale in-the-wild robot manipulation dataset，包含真实机器人轨迹、图像、动作和语言指令。官方 quickstart 使用 TensorFlow Datasets 从 `gs://gresearch/robotics` 加载 `droid`，每个 episode 有 steps、images、actions、language instructions。

#### 适合我们的用途

```text
用途：真实 robot observation ingestion + qualitative / secondary validation。
```

可以用它证明：

```text
1. LifelongSceneDB 可以接真实机器人轨迹数据；
2. observation stream 可以由 images + actions + language instructions 构成；
3. provenance 可以落到真实 frame_id / step_id；
4. task-oriented query 可以检索与语言任务、对象、动作相关的 episodes 或 segments。
```

#### 不适合作为什么

DROID 通常没有直接给出：

```text
1. 每个物体每个时刻的真实 3D location；
2. 完整 object-relation-object ground truth；
3. 物体被移动的精确 valid-time interval；
4. 每个 claim 的人工 provenance 标注。
```

所以它不适合作为主实验来评估：

```text
current-state accuracy
AS-OF accuracy
change detection F1
```

除非我们额外做 detector/VLM extraction 和人工/弱标签验证。

#### 今晚下载什么

不要下载全量。先流式导出 10-100 个 episode。

建议 Codex 写脚本：

```bash
python scripts/realdata/export_droid_sample.py \
  --episodes 50 \
  --out data/real/droid_sample_50 \
  --image-every 5
```

内部逻辑参考：

```python
import tensorflow_datasets as tfds

ds = tfds.load("droid", data_dir="gs://gresearch/robotics", split="train", shuffle_files=False)
for episode in ds.take(num_episodes):
    for step in episode["steps"]:
        image = step["observation"]["exterior_image_1_left"]
        wrist_image = step["observation"].get("wrist_image_left")
        action = step["action"]
        instruction = step["language_instruction"]
```

#### 需要的依赖

```bash
pip install tensorflow tensorflow-datasets apache-beam[gcp]
```

如果 AutoDL 访问 Google Cloud 不稳定，脚本要优雅失败，并保存 error log。不要让整个 pipeline 崩掉。

---

### 1.3 次选真实机器人数据：BridgeData V2

#### 数据定位

BridgeData V2 是大规模机器人 manipulation dataset，包含 60,096 trajectories，覆盖 24 environments 和 13 skills；每条 trajectory 有自然语言指令。它包含 toy kitchens、tabletop、sinks、microwaves、drawers 等环境，和我们的 household manipulation 场景比较贴合。

#### 适合我们的用途

```text
用途：真实机器人 trajectory + initial/final state style validation。
```

可以用它做：

```text
1. task-oriented query：retrieve episodes matching object/action language；
2. before/after visual evidence：initial frame 和 final frame；
3. weak change query：根据 instruction 推断 task-level change，例如 put X on Y / open drawer。
```

#### 不适合作为什么

```text
1. 不适合作为精确 bitemporal world-state benchmark；
2. 不适合直接评估 exact AS-OF location accuracy；
3. 不适合今晚全量下载。
```

#### 今晚下载策略

BridgeData V2 full JPEG zip 可能很大。今晚建议：

```text
1. 先下载或保存 official download page / manifest；
2. 如果有小样本 zip，再下载小样本；
3. 如果没有小样本，不要全量下载；
4. Codex 先实现 adapter，可以读取本地 BridgeData-style folders。
```

建议脚本：

```bash
python scripts/realdata/prepare_bridgedata_manifest.py \
  --out data/real/bridgedata_manifest
```

如果手动拿到 zip：

```bash
python scripts/realdata/export_bridgedata_sample.py \
  --root data/raw/bridgedata_v2 \
  --episodes 50 \
  --out data/real/bridgedata_sample_50
```

---

### 1.4 真实动态 egocentric 数据：EPIC-KITCHENS / VISOR

#### 数据定位

EPIC-KITCHENS 是 egocentric kitchen video benchmark。核心数据有大量 unscripted egocentric footage、action segments、object annotations 和 narrations。VISOR 是基于 EPIC-KITCHENS 的 hand-object segmentation / contact relation annotations。

#### 为什么它重要

虽然 EPIC-KITCHENS 不是机器人数据，但它非常适合验证：

```text
1. 真实厨房动态场景；
2. 人与物体交互；
3. 动作时段 ground truth；
4. object / hand / contact 相关 annotation；
5. query/answer 可以从 action segments 自动构造。
```

对于 LifelongSceneDB，它比 DROID 更适合构造真实数据上的：

```text
when was object interacted with?
what object changed during this time segment?
which frames support the action claim?
```

#### 今晚下载什么

不要下载全量视频。先下载 annotations / metadata。

建议目录：

```text
data/raw/epic/
  annotations/
  visor_annotations/       # optional
  sample_videos/           # optional, only if small sample is available
```

建议 Codex 写 adapter，先支持读本地 CSV/JSON：

```bash
python scripts/realdata/export_epic_annotations.py \
  --root data/raw/epic \
  --out data/real/epic_annotations
```

如果 annotations 无法自动下载，先输出 instructions 和 expected paths，不要阻塞主流程。

---

### 1.5 暂不建议今晚下载的数据

#### Open X-Embodiment full dataset

原因：

```text
1. 数据量太大；
2. 格式复杂；
3. 很多子数据集差异大；
4. 今晚不能稳定转换成 query/answer benchmark。
```

#### RoboMIND full dataset

原因：

```text
1. 很有价值，但需要先确认下载方式和许可；
2. 数据较大；
3. 接入成本高；
4. 可作为后续扩展，不适合今晚。
```

#### 全量 EPIC / DROID / BridgeData 视频

原因：

```text
1. 下载时间不可控；
2. 存储占用大；
3. 今晚主要需要 adapter 和小样本，不需要全量。
```

---

## 2. 真实数据统一格式

无论来自 OpenEQA、DROID、BridgeData 还是 EPIC，最终都要转换成统一格式。

### 2.1 统一 observation jsonl

文件：

```text
data/real/<dataset>/observations.jsonl
```

每行：

```json
{
  "obs_id": "droid_ep0001_step0005_cam_ext",
  "dataset": "droid",
  "episode_id": "ep0001",
  "event_time": 5.0,
  "arrival_time": 5.0,
  "agent_id": "robot_or_human",
  "room": "unknown_or_metadata",
  "subject": "cup",
  "predicate": "seen|interacted|moved|placed|opened|closed|instruction_mentions",
  "object": "table_or_sink_or_unknown",
  "location": "view_left|workspace|kitchen|unknown",
  "confidence": 0.7,
  "frame_id": "frame_0005.jpg",
  "evidence_ref": "path/to/frame_0005.jpg",
  "source": "language|action|detector|annotation|manual",
  "raw": {}
}
```

### 2.2 统一 query jsonl

文件：

```text
data/real/<dataset>/queries.jsonl
```

每行：

```json
{
  "query_id": "q_ep0001_001",
  "dataset": "droid",
  "episode_id": "ep0001",
  "query_type": "current|asof|change|provenance|retrieval|qa|task",
  "natural_language": "Which frames support the instruction put the cup on the table?",
  "structured": {
    "object": "cup",
    "predicate": "placed",
    "time": null,
    "region": null
  }
}
```

### 2.3 统一 answer jsonl

文件：

```text
data/real/<dataset>/answers.jsonl
```

每行：

```json
{
  "query_id": "q_ep0001_001",
  "answer_type": "object|location|time|frames|text|episode_ids",
  "answer": "cup",
  "evidence_refs": ["frame_0005.jpg", "frame_0010.jpg"],
  "confidence": 1.0,
  "label_quality": "strong|weak|heuristic|manual"
}
```

### 2.4 metadata

文件：

```text
data/real/<dataset>/metadata.json
```

内容：

```json
{
  "dataset": "droid",
  "source_url": "https://droid-dataset.github.io/",
  "num_episodes": 50,
  "num_observations": 12345,
  "num_queries": 1000,
  "label_quality": "weak",
  "created_by": "scripts/realdata/export_droid_sample.py",
  "notes": "Language/action-derived weak labels; not used for main AS-OF accuracy."
}
```

---

## 3. Observation extraction from real data

### 3.1 DROID extraction

Input fields:

```text
step image
wrist image
action
language_instruction
timestep
episode id
```

Extract observations:

```text
1. instruction_mentions(object/action): from language instruction parsing;
2. frame_seen(object): optional detector/VLM extraction;
3. action_phase: from action/gripper heuristics if available;
4. provenance: frame path + step id + camera id;
5. weak task fact: e.g., instruction says put X on Y.
```

Minimum extractor tonight:

```text
language instruction -> object/action tokens;
every Nth frame -> evidence observation;
action step -> action observation;
confidence = 0.6 for language-derived facts;
confidence = 0.5 for action-derived facts;
confidence = detector score if detector is used.
```

Optional later:

```text
Use OWL-ViT / GroundingDINO / YOLO-World to detect open-vocabulary objects from frames.
Use sentence-transformers for semantic query matching.
Use CLIP embeddings for image/frame retrieval baseline.
```

### 3.2 BridgeData extraction

Input fields:

```text
trajectory images
natural language instruction
initial/final frames
camera view metadata if available
```

Extract observations:

```text
1. instruction-derived task facts;
2. initial/final evidence frames;
3. weak before/after change facts if instruction has verbs like put, move, open, close;
4. optional detector-derived object observations.
```

Example:

```text
instruction: "put the carrot on the plate"
observations:
  instruction_mentions(carrot)
  instruction_mentions(plate)
  intended_relation(carrot, on, plate)
  evidence(initial_frame)
  evidence(final_frame)
```

### 3.3 EPIC-KITCHENS extraction

Input fields:

```text
action segment start/end
verb
noun
narration
bounding boxes or masks if available
video/frame ids
```

Extract observations:

```text
1. interacted(noun) during [start, end];
2. action(noun, verb) during [start, end];
3. provenance frames sampled from segment;
4. if VISOR masks are available: seen(noun) with bbox/mask evidence;
5. if contact relations are available: contact(hand, object).
```

EPIC is useful because action segment labels give stronger real-data ground truth than robot trajectory language alone.

### 3.4 OpenEQA extraction

Input fields:

```text
episode history H
question Q
ground-truth answer A*
metadata
```

Extract observations:

```text
1. episode history entries -> memory observations;
2. question-answer pairs -> QA queries and answers;
3. if frames/video are available locally, add frame evidence;
4. otherwise store textual episode-history evidence.
```

OpenEQA is mainly for downstream QA, not for exact world-state query correctness.

---

## 4. Ground truth strategy

### 4.1 Strong ground truth

Can be used for paper results with high confidence.

```text
Synthetic workload:
  exact current/as-of/change/provenance labels.

OpenEQA:
  human-generated QA answers for embodied QA evaluation.

EPIC-KITCHENS annotations:
  action segment start/end, verb, noun, narration, and optionally object masks/contact if VISOR is used.
```

### 4.2 Weak ground truth

Useful for qualitative or secondary results, but should not be overclaimed.

```text
DROID:
  language instructions, action sequences, success/failure metadata if available, frame provenance.

BridgeData V2:
  language instructions, initial/final frames, task labels.
```

Weak labels can support:

```text
demonstration retrieval;
task-oriented query examples;
provenance retrieval;
real-data ingestion throughput;
case studies.
```

Weak labels should not be used to claim:

```text
exact object location accuracy;
exact AS-OF world-state correctness;
precise change detection F1;
```

unless manually verified.

### 4.3 Manual spot-check option

If time permits, manually annotate 20-50 short real episodes:

```text
object of interest;
start location;
end location;
event time range;
evidence frames.
```

This can support a small real-data case study table.

---

## 5. Query and answer construction

### 5.1 OpenEQA

Use original QA pairs:

```text
query_type = qa
question = original Q
answer = original A*
evidence = episode history H or retrieved frame ids
```

Optional map to query categories:

```text
where questions -> locate_current / locate_from_history
what questions -> attribute/object query
relation questions -> graph query
```

Metrics:

```text
exact/semantic answer match if evaluator available;
retrieved evidence precision;
number of retrieved memory items;
token/context size;
latency.
```

### 5.2 DROID / BridgeData

Construct task-oriented queries:

```text
Find episodes where the robot was instructed to put X on Y.
Which frames support the instruction involving object X?
Retrieve demonstrations involving object X and action A.
What is the most likely task object in this episode?
```

Answers:

```text
episode ids;
step/frame ids;
instruction objects/actions;
weak evidence frames.
```

Metrics:

```text
retrieval recall@k using language label;
evidence coverage;
query latency;
ingestion throughput;
case-study correctness by manual inspection.
```

### 5.3 EPIC-KITCHENS

Construct annotation-derived queries:

```text
What object is interacted with during time t?
When was object X interacted with?
Which action segment supports verb V on noun N?
What changed/interacted in this time range?
Which frames support action claim (verb, noun)?
```

Answers:

```text
verb/noun labels;
segment start/end;
video/frame ids;
mask/bbox refs if available.
```

Metrics:

```text
action/object query accuracy;
time segment retrieval recall;
evidence precision;
latency.
```

---

## 6. Fair baseline comparison on real data

### 6.1 Same input principle

Every method must receive the same extracted observations.

```text
Same frames;
Same language instructions;
Same detected objects if detector is used;
Same action annotations;
Same query set;
Same answer labels.
```

If we use a detector, detector outputs must be precomputed once and shared across all systems.

### 6.2 Baselines

Use the same baselines as synthetic experiments:

```text
Vector / Text Retrieval Memory
Static Scene Graph
Latest-wins Temporal Memory
No-confidence Temporal Memory
No-bitemporal Variant
LifelongSceneDB Full
LifelongSceneDB No-index
```

For real data, comparison focus changes:

```text
OpenEQA:
  compare retrieval/evidence/context quality, not exact AS-OF state.

DROID / BridgeData:
  compare task-demonstration retrieval and evidence retrieval.

EPIC:
  compare action/object/time-segment query accuracy and evidence retrieval.
```

### 6.3 No leakage rule

Do not let LifelongSceneDB see ground truth answers during ingestion.

Allowed:

```text
language instruction;
frame;
action;
detector output;
annotation as observation only if all baselines also get it.
```

Not allowed:

```text
query answer injected as fact only for LifelongSceneDB;
manual labels used only by our method;
post-hoc correction after seeing test queries.
```

---

## 7. Which real-data results can go into the paper

### 7.1 Main experiments

These should remain synthetic / controlled:

```text
current-state accuracy;
AS-OF accuracy;
change detection F1;
noise robustness;
delay robustness;
ablation;
latency scaling.
```

Reason:

```text
Only synthetic controlled data gives clean ground truth for all memory facts and event times.
```

### 7.2 Real-data results suitable for paper main body

Can include one subsection called:

```text
Real-Data Validation
```

Recommended results:

```text
R1. DROID sample ingestion:
  number of episodes, observations, facts, ingestion throughput, query latency.

R2. OpenEQA downstream retrieval/QA support:
  evidence retrieval precision or context reduction.

R3. EPIC-KITCHENS annotation query:
  action/object segment retrieval accuracy or evidence retrieval.

R4. Qualitative case study:
  real frames/segments showing instruction -> observations -> facts -> query result.
```

### 7.3 Results not recommended for main claim

Avoid claiming from real robot data unless manually verified:

```text
LifelongSceneDB improves exact current-state object location accuracy on DROID.
LifelongSceneDB improves exact AS-OF accuracy on BridgeData.
LifelongSceneDB detects real object movement times exactly from raw videos.
```

Those require strong object-level annotations or manual labeling.

---

## 8. Tonight's Codex tasks

### 8.1 Create real-data module skeleton

Codex should create:

```text
lifelongscenedb/real/__init__.py
lifelongscenedb/real/schema.py
lifelongscenedb/real/openeqa_adapter.py
lifelongscenedb/real/droid_adapter.py
lifelongscenedb/real/bridgedata_adapter.py
lifelongscenedb/real/epic_adapter.py
scripts/realdata/download_real_data.py
scripts/realdata/export_droid_sample.py
scripts/realdata/export_openeqa.py
scripts/realdata/export_epic_annotations.py
scripts/realdata/run_real_ingestion_smoke.py
```

### 8.2 Download commands tonight

Run only safe downloads first:

```bash
# OpenEQA code + QA json
python scripts/realdata/download_real_data.py --dataset openeqa --out data/raw/open-eqa

# DROID small sample, if TFDS/GCS access works
python scripts/realdata/export_droid_sample.py --episodes 10 --image-every 10 --out data/real/droid_sample_10

# EPIC annotations only; if automatic download unavailable, create expected folder and instructions
python scripts/realdata/download_real_data.py --dataset epic_annotations --out data/raw/epic
```

Do not run full dataset downloads tonight unless storage and time are confirmed.

### 8.3 Smoke test

After downloading/converting:

```bash
python scripts/realdata/run_real_ingestion_smoke.py \
  --input data/real/droid_sample_10 \
  --out outputs/real_droid_smoke
```

Expected outputs:

```text
outputs/real_droid_smoke/summary.json
outputs/real_droid_smoke/observations_count.txt
outputs/real_droid_smoke/query_examples.jsonl
```

### 8.4 Minimum success tonight

今晚最低成功标准：

```text
1. OpenEQA json 下载成功；
2. 至少一个真实数据 adapter 可以输出 observations.jsonl；
3. LifelongSceneDB 可以 ingest observations.jsonl；
4. 能跑一个 provenance / retrieval query；
5. 输出 summary.json。
```

如果 DROID 下载失败，但 OpenEQA 成功，也算 real-data pipeline 第一版成立。

---

## 9. Suggested implementation details

### 9.1 RealDataObservation schema

```python
@dataclass
class RealDataObservation:
    obs_id: str
    dataset: str
    episode_id: str
    event_time: float
    arrival_time: float
    agent_id: str
    room: str | None
    subject: str
    predicate: str
    object: str | None
    location: str | None
    confidence: float
    frame_id: str | None
    evidence_ref: str | None
    source: str
    raw: dict
```

### 9.2 Adapter output contract

Each adapter must implement:

```python
def export_observations(input_root: str, output_root: str, limit: int | None = None) -> None:
    """Writes observations.jsonl, queries.jsonl, answers.jsonl, metadata.json."""
```

### 9.3 Integration with LifelongSceneDB

Add a loader:

```python
def load_observations_jsonl(path: str) -> list[Observation]:
    ...
```

Then run:

```python
db = LifelongSceneDB()
for obs in load_observations_jsonl("data/real/droid_sample_10/observations.jsonl"):
    db.ingest_observation(obs)
```

---

## 10. Paper usage plan

### 10.1 In the paper

Add a small section:

```text
Real-Data Validation
```

Possible text:

```text
In addition to the controlled synthetic workload, we evaluate whether LifelongSceneDB can ingest real embodied observations. We build adapters for OpenEQA and DROID/EPIC-KITCHENS-style data, converting episode histories, robot trajectories, language instructions, and action annotations into the same observation schema used by the synthetic benchmark. Since these datasets do not always provide complete object-level bitemporal world-state ground truth, we use them to evaluate ingestion throughput, evidence retrieval, and downstream QA/retrieval support rather than exact AS-OF world-state accuracy.
```

### 10.2 Do not overclaim

Do not write:

```text
Real data proves our AS-OF memory is correct.
```

Write:

```text
Real data validates that the observation schema and query interface can ingest real embodied data and support evidence/retrieval workloads.
```

---

## 11. Final recommendation

今晚 Codex 可以开始下载/准备：

```text
1. OpenEQA repo/json: yes, high priority, low risk.
2. DROID first 10-50 episodes via TFDS streaming: yes, if network works.
3. EPIC-KITCHENS annotations: yes, if accessible without manual account steps.
4. BridgeData V2 full zip: no, only manifest or small sample.
5. Full videos/full robot datasets: no.
```

真实数据下一步的定位：

```text
Synthetic workload remains the ICDE main experiment.
Real data becomes a credibility and external-validation section.
```
