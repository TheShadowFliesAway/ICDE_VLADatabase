# ICDE EAB Front-Figure Plan for EviStateBench

This document summarizes what non-experimental figures EviStateBench should include before the evaluation section. It is based on a quick deep-research pass over ICDE 2026 EAB papers and nearby database EAB-style benchmark/system papers.

Scope: front-section figures only. Experimental result plots are intentionally excluded.

## Sources Checked

Official ICDE accepted-paper list:

- ICDE 2026 Accepted Research Papers: https://icde2026.github.io/accepted-papers.html

Representative papers inspected:

- SQLMorph: Query Mutation and Fine-Grained Metrics for Text-to-SQL Evaluation, ICDE 2026 EAB. Public PDF: https://amine.io/papers/2026-icde-sqlmorph.pdf
- BEACON: A Benchmark for Efficient and Accurate Counting of Subgraphs, ICDE 2026 EAB listing / public preprint: https://arxiv.org/abs/2504.10948
- XRAG: eXamining the Core -- Benchmarking Foundational Components in Advanced Retrieval-Augmented Generation, ICDE 2026 EAB listing / public preprint: https://arxiv.org/abs/2412.15529
- Systematic Evaluation of Plan-based Adaptive Query Processing, ICDE 2026 EAB listing / public preprint: https://arxiv.org/abs/2511.16455
- One Size Does NOT Fit All: On the Importance of Physical Representations for Datalog Evaluation, ICDE 2026 EAB listing / public preprint: https://arxiv.org/abs/2602.05651
- SparqLog: A System for Efficient Evaluation of SPARQL 1.1 Queries via Datalog, database EAB-style reference: https://arxiv.org/abs/2307.06119

## Pattern From Similar Papers

Similar ICDE EAB / benchmark-style papers usually use front figures for four jobs:

1. Explain the task with a concrete example.
2. Show the system, benchmark, or framework architecture.
3. Define the data/query/workload abstraction visually.
4. Explain a generation, mutation, translation, or evaluation pipeline.

Examples:

| Paper | Front-section non-result figures |
| --- | --- |
| SQLMorph | Text-to-SQL pipeline; multi-stage join-query expansion pipeline. |
| BEACON | Example input graph and query pattern; benchmark framework plus usage scenarios. |
| XRAG | Framework overview; dataset/context distribution before experiments. |
| Plan-based AQP | QuerySplit example; DAG representation; unified AQP module representation; modular AQP design; plan-merge example. |
| Datalog physical representations | Background/motivation examples and operation-level pseudo-code; later decision-tree/signature diagrams. |
| SparqLog | Example SPARQL query; translated Datalog rules; property-path example; translation-rule diagrams. |

Takeaway: front figures should teach the reader how to read the rest of the paper. They should not duplicate tables, and they should not become decorative screenshots.

## Recommended Figures for EviStateBench

Use 3 required front figures, plus 1 optional figure if space permits.

### Figure 1: Motivating Temporal State-View Example

Placement: Section I or early Section II.

Purpose:

- Make the problem concrete before any formalism.
- Show an embodied observation stream with event time and arrival time.
- Show imperfect observations: missing, noisy, delayed, or conflicting evidence.
- Show the maintained task-state view and one example query answer.

Suggested visual structure:

```text
Observation stream
  obs_1: cup inside cabinet at t1
  obs_2: cabinet open at t2
  obs_3: noisy/conflicting state at t3
        |
        v
Temporal task-state view
  open(cabinet, [t2, ...])
  inside(cup, cabinet, [t1, ...])
        |
        v
Example query
  "Was the cup accessible before the goal step?"
```

Design notes:

- This should be an explanatory diagram, not a plot.
- It can be one column if compact; use two columns only if event-time/arrival-time conflict needs more room.
- Avoid robotics-heavy visual language. The database concept is temporal view maintenance over evidence.

### Figure 2: Benchmark Architecture and Artifact Boundary

Placement: Section IV, after the benchmark-design opening.

Purpose:

- Show EviStateBench as a benchmark package, not merely a dataset.
- Separate public artifacts from hidden evaluator artifacts.
- Show systems under test consuming public inputs and evaluator comparing against hidden answer sets.
- Make clear that EviStateDB is a reference baseline, not the oracle.

Suggested visual structure:

```text
Simulator truth + task specs
          |
          v
Benchmark generator
          |
          +--> Public artifacts:
          |      task specs, observation streams, queries, metadata
          |
          +--> Hidden evaluator artifacts:
                 hidden timelines, answer sets

Public artifacts --> Systems under test --> Predictions
Hidden artifacts + Predictions --> Evaluator --> Metrics/report
```

Design notes:

- This is likely the most important front figure.
- Make it page-wide if needed.
- Use color or line style to distinguish public vs hidden, but keep it printable in grayscale.

### Figure 3: Data Model and Query Semantics

Placement: Section III or IV.

Purpose:

- Visually define the core objects readers need for the formal section.
- Connect observations, evidence, predicates, time intervals, state variables, queries, and answer sets.
- Clarify event-time vs arrival-time semantics.

Suggested visual structure:

```text
Observation
  episode_id, object_id, predicate, value,
  event_time, arrival_time, confidence, source
        |
        v
TemporalStateView
  key: object/predicate
  value over intervals
  provenance/evidence links
        |
        v
Query families
  point, interval, transition, goal predicate, repair/conflict
```

Design notes:

- If the formal definitions become dense, this figure will save space.
- This can be one column if drawn as a compact schema.
- Do not overload it with all predicates; put predicate/task counts in a table instead.

### Optional Figure 4: Generation and Validation Pipeline

Placement: Section V.

Purpose:

- Explain how benchmark artifacts are produced and validated.
- Show simulator state capture, hidden truth construction, clean/mixed observation stream generation, query generation, public sanitization, and validation gates.

Suggested visual structure:

```text
Task library -> Simulator episodes -> Hidden timeline
                                    -> Clean observations
                                    -> Perturbation injection -> Mixed observations
                                    -> Query generator -> Answer sets
                                    -> Public sanitizer + validation reports
```

Use this figure if:

- Section V feels hard to follow in prose.
- We need to emphasize reproducibility and artifact validation.
- We have enough room after placing Figures 1-3.

Skip this figure if:

- Figure 2 already includes generator details.
- Space pressure becomes severe.
- The same information is clearer as an artifact-statistics table plus short prose.

## What Should Be Tables Instead

Use tables rather than figures for:

- Artifact statistics: episodes, tasks, queries, clean/mixed observations, validation status.
- Query workload taxonomy.
- Perturbation regimes.
- Baseline/system comparison.
- Related-benchmark comparison.

These are scan-and-compare objects; figures would waste space.

## Recommended Final Front-Figure Set

For the 12-page ICDE paper, start with this exact set:

1. Figure 1: motivating temporal state-view example.
2. Figure 2: benchmark architecture and public/hidden artifact boundary.
3. Figure 3: data model and query semantics.
4. Optional Figure 4: benchmark generation and validation pipeline.

If we need to cut one figure, cut Optional Figure 4 first and fold its content into Figure 2 plus the artifact-statistics table.

If we need a figure for EviStateDB, do not add a separate large architecture figure by default. EviStateDB is not the paper protagonist. A small inset inside Figure 2 or a compact baseline table is enough unless the reference engine becomes central to the claims.

## Drafting Order for Figures

1. Sketch Figure 2 first, because it fixes the benchmark boundary and prevents oracle/baseline confusion.
2. Sketch Figure 1 second, because it determines the running example used in Introduction and Motivation.
3. Sketch Figure 3 third, because it should align with formal notation.
4. Decide whether Figure 4 is needed only after Section V is drafted.

## Figure Quality Rules

- Every front figure must introduce a concept used later in the text.
- Captions should be explanatory enough that the reader can understand the figure without reading the whole paragraph.
- Avoid screenshots unless they show a real artifact interface that matters to reproducibility.
- Avoid result bars, line charts, or ablation plots before Section VII.
- Prefer clean boxes, arrows, timelines, and schema-like diagrams over illustrative scenes.
- Keep terminology consistent with the paper: EviStateBench, EviStateDB, public artifacts, hidden truth, evaluator, TemporalStateView, event time, arrival time.
