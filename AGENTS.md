# Project Instructions

This repository is for an ICDE-style IEEE conference paper.

The paper's technical content must be grounded in `/root/autodl-tmp/EviStateBench`. Treat that repository as the source of truth for the benchmark idea, schemas, generator pipeline, public artifacts, reports, baseline sanity results, and current limitations.

Follow `/root/autodl-tmp/EviStateBench/EviStateBench_PAPER_WRITING_BLUEPRINT.md` as the primary paper-writing blueprint for contributions, section structure, reviewer questions, figure/table plan, experiment matrix, related-work strategy, writing guardrails, abstract skeleton, and drafting order.

Before editing LaTeX, paper structure, figures, tables, algorithms, citations, or bibliography, consult `docs/ieeetran-format-guide.md` and preserve IEEEtran formatting conventions.

Also consult `docs/icde-eab-requirements.md` before planning or revising the paper. Project decision: write a full 12-page main paper, excluding references and AI acknowledgement, with no appendix. Do not plan it as a short paper; allocate enough substantive benchmark, artifact, experimental, and limitation content to fill the full page budget.

Consult `docs/icde-eab-section-plan.md` when planning the table of contents, page budget, section order, figure/table placement, or drafting sequence. Use its 10-section main-paper structure as the current working plan unless the user explicitly changes the strategy.

Consult `docs/icde-eab-front-figure-plan.md` before creating or revising non-experimental figures in the Introduction, Motivation, Problem Definition, Benchmark Design, or Artifact sections. The current planned front figures are: motivating temporal state-view example, benchmark architecture/public-hidden artifact boundary, data model and query semantics, and optional generation/validation pipeline.

Use the official ICDE author instructions as the final authority. When they are unavailable, keep the current baseline `\documentclass[conference]{IEEEtran}` and avoid manual changes to margins, fonts, spacing, section styles, and column layout.

Use the fixed positioning from EviStateBench: EviStateBench is the benchmark protagonist; EviStateDB is a reference baseline engine, not the oracle; ground-truth answers come from simulator truth and task specifications; the core problem is temporal task-state view maintenance over embodied observation streams.

When blueprint language mentions appendix, adapt it to ICDE's no-appendix rule: move extra baselines, sweeps, and supplemental workloads into public artifacts/release reports or compress the essential parts into the 12-page main paper.

When drafting or revising prose, prioritize database-conference clarity: explicit problem statement, concrete data model, query semantics, maintenance algorithms, evaluation metrics, and fair baselines.

When checking formatting, pay special attention to: figure/table caption placement, labels after captions, grouped IEEE citations, column-width equations, algorithm floats, IEEEtran bibliography style, and final PDF font embedding.
