# BIRD Text-to-SQL: From SFT to Execution-Aware Selection

A controlled Text-to-SQL study on the **BIRD** benchmark using **Qwen2.5-Coder-7B-Instruct**. The project evaluates vanilla and failure-mode-targeted LoRA SFT, diagnoses why token-level improvement does not necessarily produce execution-correct SQL, and uses the resulting Oracle@K headroom to motivate an execution-aware R³-SQL-Lite pipeline.

## Research Question

> When does supervised adaptation help Text-to-SQL, and when are execution-aware search and candidate selection required to improve semantic correctness?

Text-to-SQL is ultimately evaluated by execution accuracy, whereas standard SFT minimizes next-token loss. A query can be syntactically valid and textually similar to the gold SQL while still returning the wrong result. This project tests that objective mismatch rather than assuming that lower training loss implies better execution accuracy.

## Experimental Design

- **Base model:** `Qwen/Qwen2.5-Coder-7B-Instruct`
- **Dataset:** BIRD Text-to-SQL
- **Held-out benchmark:** BIRD MiniDev500 (`N = 500`)
- **Generators:** Base, Original LoRA SFT, Targeted LoRA SFT
- **Prompt settings:** with and without BIRD evidence
- **Primary metrics:** EX@1, Oracle@3, Oracle@5, Valid SQL@1
- **Training:** BF16 LoRA, one epoch, `max_length=8192`, completion-only loss

The Targeted SFT dataset contains 5,825 prompts built from human-calibrated failure families plus general replay examples.

## Main Results

| Generator | Evidence | EX@1 | Oracle@3 | Oracle@5 | Valid SQL@1 |
|---|---|---:|---:|---:|---:|
| Base | No | 27.0% | 32.6% | 36.6% | 86.0% |
| Original SFT | No | 27.0% | 36.0% | 39.8% | 90.0% |
| Targeted SFT | No | **28.2%** | 36.0% | **40.8%** | **91.4%** |
| Base | Yes | **50.4%** | 55.8% | **60.8%** | 89.4% |
| Original SFT | Yes | 47.6% | **56.2%** | 60.0% | **91.0%** |
| Targeted SFT | Yes | 44.0% | 55.4% | 59.6% | 88.8% |

## Key Findings

1. **Evidence is the strongest intervention.** Base EX@1 increases from 27.0% to 50.4%, a gain of 23.4 percentage points.

2. **Original SFT improves form more reliably than semantics.** It increases SQL validity, but does not improve EX@1 without evidence and trails Base+Evidence by 2.8 points.

3. **Targeted SFT does not recover execution accuracy.** With evidence, it reaches 44.0% EX@1—6.4 points below Base and 3.6 points below Original SFT. The paired decreases are statistically significant.

4. **SFT changes individual cases rather than simply doing nothing.** Original SFT fixes 33 Base+Evidence failures but regresses 47 previously correct examples.

5. **The candidate set contains recoverable headroom.** Targeted-SFT+Evidence rises from 44.0% EX@1 to 59.6% Oracle@5. Correct SQL is often generated but not ranked first.

6. **Failure labels require calibration.** The v3 classifier is useful for aggregate diagnosis, but human review exposed benchmark ambiguity and limited label precision.

The negative SFT result is part of the contribution: it closes the loop from controlled training through held-out evaluation, paired failure analysis, human adjudication, targeted data construction, retraining, and statistical diagnosis.

## Current Direction: R³-SQL-Lite

The next stage reuses the saved K=5 candidates instead of immediately running another generator-training experiment:

```text
K=5 candidates
      ↓
Canonical execution results
      ↓
Execution-equivalent groups
      ↓
Functional Majority Voting
      ↓
Zero-shot group verification
      ↓
Selective resampling when uncertain
```

Pairwise ranker SFT or execution-guided DPO will be considered only after execution-aware selection demonstrates measurable headroom.

## Repository Structure

```text
bird-text2sql-sft/
├── README.md
├── EXPERIMENT_LOG.md
├── requirements.txt
├── configs/
│   ├── training/
│   └── evaluation/              # Six real per-run configs
├── docs/
│   ├── sft_findings.md
│   └── r3_sql_lite_plan.md
├── notebooks/                   # Preparation, training, evaluation, analysis
└── results/
    ├── summary/
    ├── detailed/
    └── figures/
```

The evaluation directory retains the six actual per-run configuration files. Synthetic aggregate files such as `six_way_evaluation_config.json` and `oracle_k_config.json` are not required.

## Current Status

- [x] Database-level data split and mixed-evidence preparation
- [x] Original BF16 LoRA SFT
- [x] MiniDev500 Base/SFT × Evidence evaluation
- [x] EX@1, Oracle@3, and Oracle@5 diagnostics
- [x] Failure-mode analysis v3 and human adjudication
- [x] Targeted dataset construction and Targeted LoRA SFT
- [x] Held-out six-way comparison and statistical analysis
- [ ] Canonicalize saved candidate execution outputs
- [ ] Evaluate execution grouping and Functional Majority Voting
- [ ] Add a zero-shot group verifier
- [ ] Add selective resampling for uncertain examples

## Documentation

- [`EXPERIMENT_LOG.md`](./EXPERIMENT_LOG.md) — full configurations, results, statistical tests, and decision history.
- [`docs/sft_findings.md`](./docs/sft_findings.md) — concise interpretation of the vanilla and targeted SFT findings.
- [`docs/r3_sql_lite_plan.md`](./docs/r3_sql_lite_plan.md) — implementation plan for execution-aware candidate selection and resampling.
