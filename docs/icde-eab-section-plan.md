# ICDE EAB Section Plan for EviStateBench

This document records the current section plan for the ICDE EAB paper.
It complements:

- `docs/icde-eab-requirements.md`
- `docs/ieeetran-format-guide.md`
- `/root/autodl-tmp/EviStateBench/EviStateBench_PAPER_WRITING_BLUEPRINT.md`

## Sources Checked

Official ICDE guidance:

- ICDE 2027 Research Papers CFP: https://icde2027.github.io/cf-research-papers.html
- ICDE 2027 Submission Guidelines: https://icde2027.github.io/submission-guidelines.html
- ICDE 2026 Accepted Research Papers: https://icde2026.github.io/accepted-papers.html

Representative EAB / benchmark-style papers inspected for structure:

- SQLMorph: Query Mutation and Fine-Grained Metrics for Text-to-SQL Evaluation, ICDE 2026 EAB.
- BEACON: A Benchmark for Efficient and Accurate Counting of Subgraphs, ICDE 2026 EAB.
- Systematic Evaluation of Plan-based Adaptive Query Processing, ICDE EAB preprint.
- XRAG: eXamining the Core -- Benchmarking Foundational Components in Advanced Retrieval-Augmented Generation, ICDE 2026 EAB listing and public preprint.
- SparqLog: A System for Efficient Evaluation of SPARQL 1.1 Queries via Datalog, DB EAB-style reference.

## Observed EAB Structure Patterns

EAB papers usually do not follow a narrow IMRaD template. They tend to use one of three related shapes.

1. Contribution-centric evaluation framework
   - Introduction
   - Background / target pipeline
   - Experimental setup
   - One section per benchmark/methodological contribution
   - Related work and challenges
   - Conclusion

   SQLMorph uses this shape: setup appears early, then the main contribution sections each contain design plus analysis.

2. Benchmark-centric artifact paper
   - Introduction
   - Notation / problem definition
   - Related work
   - Proposed benchmark framework
   - Experimental evaluation
   - Conclusion

   BEACON uses this shape: the benchmark ecosystem and datasets are central, and the evaluation demonstrates how the benchmark reveals method trade-offs.

3. System-analysis EAB paper
   - Introduction
   - Preliminaries and related work
   - System/module design
   - Experimental setup
   - Experimental evaluation
   - Conclusion or embedded takeaways

   Plan-based AQP and SparqLog use variants of this shape, where the system or module decomposition is needed before the experiments make sense.

For EviStateBench, the best fit is the benchmark-centric artifact paper with a system-analysis evaluation section. EviStateBench is the protagonist; EviStateDB is a reference baseline and diagnostic subject, not the oracle.

## Recommended Main-Paper Table of Contents

Target: 12 pages excluding references and AI acknowledgement, with no appendix. Plan for about 11.8-12.0 pages of substantive main-paper content and keep only a small final formatting buffer.

| Part | Target Pages | Role |
| --- | ---: | --- |
| Abstract | 0.25 | Compact problem, benchmark, artifact, and result summary. |
| I. Introduction | 1.15 | Establish task-state view maintenance as a data-engineering problem; state benchmark gap and contributions. |
| II. Motivation and Benchmark Scope | 0.85 | Explain why embodied observation streams need temporal state views; define what EviStateBench evaluates and what it deliberately excludes. |
| III. Problem Definition | 1.05 | Formalize observations, event time, arrival time, task specs, state variables, temporal state views, queries, answers, and metrics. |
| IV. EviStateBench Benchmark Design | 1.55 | Present public/hidden artifact boundary, tasks, observation streams, query families, perturbation regimes, evaluator, and metrics. |
| V. Benchmark Generation and Artifacts | 1.25 | Describe simulator-driven generation, hidden truth extraction, clean/mixed streams, validation gates, and released artifact statistics. |
| VI. Reference Engine and Experimental Protocol | 1.15 | Explain EviStateDB and baselines, reproducibility settings, fairness rules, and why the reference engine is not the oracle. |
| VII. Evaluation and Analysis | 2.70 | Main EAB payload: benchmark characterization, main accuracy results, query-family breakdown, robustness, efficiency, ablations, and error analysis. |
| VIII. Lessons, Limitations, and Artifact Availability | 0.90 | Distill what the benchmark reveals; state limits such as action-source gap and simulator scope; point to reproducibility artifacts. |
| IX. Related Work | 0.90 | Compare against database benchmarks, temporal/stream processing, embodied-state benchmarks, VLM/RAG-style evaluation, and state tracking. |
| X. Conclusion | 0.20 | Close with the benchmark's contribution and intended community use. |

## Section Details

### I. Introduction

Jobs:

- Open with the mismatch between embodied observation streams and the task-state abstractions that downstream systems query.
- Frame the problem as temporal task-state view maintenance over imperfect evidence, not as robotics control or perception.
- State the evaluation gap: existing benchmarks do not test database-style maintenance of time-indexed task-state views under missing, noisy, or conflicting observations.
- List contributions: benchmark, generator/artifact protocol, query/evaluator semantics, reference baseline, empirical findings.
- Mention reproducibility and EAB artifact availability.

Avoid:

- Overclaiming general embodied intelligence.
- Calling EviStateDB the oracle.
- Spending first-page space on implementation minutiae.

### II. Motivation and Benchmark Scope

Jobs:

- Use one running example showing how a task state changes over event time and how observation noise creates conflicting evidence.
- Define the benchmark's scope: task-state maintenance, temporal query answering, robustness to imperfect observations, and artifact reproducibility.
- Explicitly place out-of-scope items: low-level robot control, perception model training, and open-ended planning.

Primary figure:

- Figure 1: motivation example from observation stream to temporal task-state answer.

### III. Problem Definition

Jobs:

- Define task specifications, predicates, objects, observations, evidence records, event time, arrival time, repairs, hidden truth, and answer sets.
- Define TemporalStateView as the conceptual maintained view.
- Define query classes and metrics at a level that makes the evaluator understandable.
- Make the oracle boundary precise: simulator truth plus task specs, not EviStateDB output.

Primary figure/table:

- Figure 3 or a compact schema diagram for data model and query semantics.
- Table 3 can summarize query families and metrics if space permits here; otherwise place it in Section IV.

### IV. EviStateBench Benchmark Design

Jobs:

- Describe the benchmark as artifacts plus evaluator, not just a dataset.
- Explain public artifacts: task specs, observation streams, query files, metadata, validation reports.
- Explain hidden artifacts: hidden timelines and answer sets used only by the evaluator.
- Present query workload families, perturbation regimes, and metrics.
- Make the no-leakage rule explicit: systems under test receive public inputs, not hidden truth.

Primary figure/table:

- Figure 2: benchmark architecture.
- Table 3: query workloads.
- Table 4: perturbation regimes.

### V. Benchmark Generation and Artifacts

Jobs:

- Describe episode generation, simulator state capture, hidden truth construction, observation extraction, perturbation injection, and public sanitization.
- Report the current main artifact statistics: number of episodes, tasks, queries, clean observations, mixed observations, and validation status.
- Explain quality gates and what PASS / PASS_WITH_LIMITS means.
- If v4 local-action data is used, present it as supplemental validation or a controlled additional regime, not as the main benchmark.

Primary table:

- Table 2: artifact statistics.

### VI. Reference Engine and Experimental Protocol

Jobs:

- Describe EviStateDB as a reference temporal-state maintenance engine.
- Define baselines and ablations.
- State implementation and runtime settings.
- Explain evaluation protocol and fairness constraints.
- Preserve the distinction: hidden truth evaluates all systems; EviStateDB is one system under test.

Primary table:

- Table 5: baselines / systems under test.

### VII. Evaluation and Analysis

Jobs:

- E1: characterize benchmark difficulty and main clean vs mixed performance.
- E2: break down results by query semantics, predicate family, and task family.
- E3: analyze robustness under missing, noisy, conflicting, delayed, and mixed observations.
- E4: report efficiency and ablations if stable enough.
- Include error analysis and concrete takeaways after each major result group.

Primary figure:

- Figure 5: results overview.

Writing rule:

- This should be the largest section. EAB reviewers expect the scientific contribution to appear in the analysis, not only in the benchmark description.

### VIII. Lessons, Limitations, and Artifact Availability

Jobs:

- Summarize lessons about temporal view maintenance under embodied observation uncertainty.
- State limitations honestly: simulator scope, action-source gap where applicable, finite task set, and public/hidden split.
- Explain artifact availability, reproducibility commands, and what is in the supplemental release.
- Since ICDE allows no appendix, put overflow details in public artifacts or release reports, not in an appendix.

### IX. Related Work

Jobs:

- Compare with database benchmarks and EAB-style benchmark papers.
- Compare with temporal databases, stream processing, event sourcing, and provenance/evidence-aware data management.
- Compare with embodied AI / VLM benchmarks only through the lens of state tracking and evaluation artifacts.
- End by positioning EviStateBench as a database benchmark for temporal task-state view maintenance.

Primary table:

- Table 1: related benchmark comparison can appear here or near Section II if it strengthens the motivation.

### X. Conclusion

Jobs:

- Re-state EviStateBench's benchmark contribution and the main empirical message.
- Close with community-facing artifact use: evaluate systems, compare methods, and expose failure modes.

## Blueprint Mapping

The EviStateBench blueprint proposes a rich 12-section skeleton. For ICDE's 12-page limit, use fewer main sections and push the blueprint content into subsections:

| Blueprint Content | Planned Location |
| --- | --- |
| Introduction | I |
| Background and Motivation | II |
| Problem Definition | III |
| EviStateBench Benchmark | IV |
| Benchmark Generator | V |
| EviStateDB Reference Baseline Engine | VI |
| Experimental Setup | VI |
| Evaluation and Analysis | VII |
| External / Supplemental Validation | VIII or VII, depending on result stability |
| Lessons Learned | VIII |
| Related Work | IX |
| Conclusion | X |

## Drafting Order

Recommended writing order:

1. III. Problem Definition
2. IV. EviStateBench Benchmark Design
3. V. Benchmark Generation and Artifacts
4. VI. Reference Engine and Experimental Protocol
5. VII. Evaluation and Analysis
6. VIII. Lessons, Limitations, and Artifact Availability
7. II. Motivation and Benchmark Scope
8. IX. Related Work
9. I. Introduction
10. Abstract and Conclusion

This order keeps early prose grounded in the actual benchmark semantics and current artifacts.
