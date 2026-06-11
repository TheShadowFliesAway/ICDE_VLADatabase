# ICDE EAB Submission Requirements

Sources:

- ICDE 2027 Call for Research Papers: https://icde2027.github.io/cf-research-papers.html
- ICDE 2026 Call for Research Papers: https://icde2026.github.io/cf-research-papers.html

Use the current target year's official call as final authority.

## Category

EviStateBench should target ICDE's Experimental, Analysis, and Benchmark (EAB) category.

EAB papers are expected to focus on extensive evaluation of algorithms, data structures, and systems, or on benchmarks related to ICDE topics. The scientific contribution should be new insight into strengths and weaknesses of existing methods, or new ways to evaluate existing methods.

## Title

The title must start with:

```text
[Experiment, Analysis, and Benchmark]
```

For this project, the working title shape should remain:

```text
[Experiment, Analysis, and Benchmark] EviStateBench:
Evaluating Temporal Task-State View Maintenance over Embodied Observation Streams
```

## Page Budget

- Research and EAB papers must not exceed 12 pages.
- References are excluded from the 12-page limit.
- The required AI-generated content acknowledgement is excluded from the 12-page limit.
- No appendix is allowed.
- Only PDF submissions are considered.

Project writing target: fill the full 12-page main-paper budget with substantive content. EAB papers need enough room for benchmark design, artifact boundary, data generation, query workloads, metrics, baselines, experiments, limitations, and artifact reproducibility.

This is a project decision, not an official minimum-length rule. The draft should be planned as a 12-page paper from the beginning, with enough real technical material to occupy the full budget without filler. Keep only a small formatting buffer near submission time.

## Formatting

- Use IEEE conference format.
- Do not modify margins, fonts, spacing, section style, or columns.
- Do not use negative space or push content into margins.
- Formatting and length violations can cause desk rejection.

## Artifact Requirement

Supplemental material is expected for papers reporting code/data/results. For EAB papers, all artifacts necessary to reproduce results must be provided, with no exceptions.

For EviStateBench, this means the paper should point to reproducible artifacts for:

- public benchmark inputs
- hidden/evaluator answer generation protocol, without leaking hidden files to systems under test
- baseline implementations
- evaluation scripts
- build commands
- validation reports
- public artifact sanitization checks

## Scope Requirement

The submission must be clearly relevant to data engineering. For this paper, the ICDE-facing framing should emphasize:

- temporal task-state view maintenance
- event-time vs arrival-time semantics
- bitemporal repair
- uncertain and conflicting observations
- provenance and evidence-aware query answers
- benchmark generation and artifact validation
- query workloads and evaluator metrics

Do not frame the paper primarily as a robotics, VLM, perception, low-level control, or embodied-memory paper.

## AI Acknowledgement

ICDE requires disclosure for AI-generated content. Keep a short acknowledgement section before references or in the acknowledgement location specified by the target-year CFP. It does not count toward the page limit.
