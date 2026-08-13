# BIRD Text-to-SQL: From SFT to Execution-Aware Search

A research-oriented Text-to-SQL project built on **Qwen2.5-Coder-7B-Instruct** and the **BIRD** benchmark.

The project first evaluates whether LoRA supervised fine-tuning (SFT) improves Text-to-SQL execution accuracy, then investigates why token-level optimization does not necessarily translate into better execution-level correctness.

The current next step is **R³-SQL-lite**: candidate sampling, execution-aware grouping, verification, ranking, and selective resampling.

---

## Overview

Text-to-SQL is ultimately judged by whether the generated SQL returns the correct database result.

Vanilla SFT optimizes next-token likelihood, while the final task metric is **Execution Accuracy (EX)**.

This creates a potential mismatch:

```text
Token-level training objective
        ↓
Generate gold-like SQL
        ↓
may not imply
        ↓
Correct execution result
```

The project therefore studies two stages:

```text
Stage I
LoRA SFT + Evidence Conditioning
        ↓
MiniDev500 Evaluation

Stage II
Candidate Sampling
        ↓
Oracle@K
        ↓
Execution Grouping
        ↓
Verification / Ranking
        ↓
Selective Resampling
```

---

## Model and Dataset

**Base model**

```text
Qwen/Qwen2.5-Coder-7B-Instruct
```

**Dataset**

```text
BIRD Text-to-SQL
```

**Held-out evaluation**

```text
BIRD MiniDev500
N = 500
```

The final training pipeline uses:

- database-level train/validation splitting;
- mixed evidence exposure;
- LoRA SFT;
- BF16 training;
- `max_length = 8192`;
- completion-only loss.

Detailed training and evaluation settings are recorded in [`EXPERIMENT_LOG.md`](./EXPERIMENT_LOG.md).

---

## Main Results

After increasing the input token limit to **8192**, four configurations were evaluated on the same MiniDev500 set.

| Model | Evidence | EX | Valid SQL | Normalized EM | Avg Generation (s) |
|---|---|---:|---:|---:|---:|
| Base | No | **27.00%** | 86.00% | 6.40% | 1.767 |
| Base | Yes | **50.40%** | 89.40% | 9.40% | 1.827 |
| SFT | No | **27.00%** | 90.00% | 7.00% | 4.066 |
| SFT | Yes | **47.60%** | 91.00% | 10.00% | 4.127 |

---

## Key Findings

### 1. Evidence is the strongest performance driver

For the base model:

```text
27.00% EX → 50.40% EX
```

an absolute improvement of:

```text
+23.40 percentage points
```

---

### 2. Vanilla SFT does not improve EX without evidence

```text
Base: 27.00%
SFT:  27.00%
```

The overall execution accuracy remains unchanged.

---

### 3. SFT improves SQL validity more reliably than execution correctness

Without evidence:

```text
Valid SQL: 86.00% → 90.00%
```

With evidence:

```text
Valid SQL: 89.40% → 91.00%
```

However, with evidence:

```text
EX: 50.40% → 47.60%
```

So the model becomes more syntactically reliable without becoming more semantically correct.

---

### 4. Better textual alignment does not necessarily imply better EX

Normalized Exact Match improves after SFT:

```text
6.40% → 7.00%
9.40% → 10.00%
```

while execution accuracy is unchanged or lower.

This motivates the central hypothesis of the project:

> **Token-level SFT is useful for adaptation and formatting, but execution-level Text-to-SQL correctness may require search, verification, and execution-aware selection.**

---

## Why R³-SQL-lite?

The next stage moves away from single-pass generation:

```text
Question
   ↓
Generate One SQL
   ↓
Return
```

toward an execution-aware pipeline:

```text
Question
   ↓
Generate Multiple Candidates
   ↓
Execute
   ↓
Group by Execution Result
   ↓
Verify / Rank
   ↓
Resample if Necessary
   ↓
Return Final SQL
```

The first diagnostic experiment will compare:

```text
EX@1
Oracle@3
Oracle@5
```

The goal is to determine whether the current bottleneck is mainly:

- **generation** — the model rarely generates a correct SQL candidate; or
- **selection** — the correct SQL is often present among several candidates but is not selected as top-1.

The full R³-SQL-lite design is documented in [`docs/r3_sql_lite_plan.md`](./docs/r3_sql_lite_plan.md).

---

## Repository Structure

```text
bird-text2sql-sft/
│
├── README.md
├── EXPERIMENT_LOG.md
├── requirements.txt
├── .gitignore
│
├── notebooks/
│   ├── 02_data_preparation.ipynb
│   ├── 04_sft_training.ipynb
│   ├── 05_minidev500_four_way.ipynb
│   └── 06_oracle_k_analysis.ipynb
│
├── docs/
│   └── r3_sql_lite_plan.md
│
├── results/
│   ├── figures/
│   ├── summary/
│   ├── per_example/
│   └── runs/
│
└── configs/
    ├── training/
    └── evaluation/
```

---

## Current Status

```text
[Completed] BIRD data preparation
[Completed] Database-level train/validation split
[Completed] Mixed-evidence SFT preparation
[Completed] 8192-token length audit
[Completed] Qwen2.5-Coder-7B LoRA SFT
[Completed] MiniDev500 four-way evaluation
[Completed] Base/SFT × Evidence analysis

[Next]      EX@1 / Oracle@3 / Oracle@5
[Planned]   Execution grouping
[Planned]   Functional Majority Voting
[Planned]   Zero-shot verifier
[Planned]   Selective resampling
[Planned]   Hard-negative ranker
```

---

## Documentation

- [`EXPERIMENT_LOG.md`](./EXPERIMENT_LOG.md) — experiment configurations, results, and decisions.
- [`docs/r3_sql_lite_plan.md`](./docs/r3_sql_lite_plan.md) — detailed design of the execution-aware next stage.

---

## Current Research Question

> **When does token-level supervised adaptation help Text-to-SQL, and when are execution-aware search, verification, and ranking required to improve semantic correctness?**
