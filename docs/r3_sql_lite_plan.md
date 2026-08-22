# R³-SQL-lite: Execution-Aligned Text-to-SQL Plan

## 1. Objective

R³-SQL-lite studies how to improve BIRD Text-to-SQL execution accuracy under a
single-GPU budget using `Qwen/Qwen2.5-Coder-7B-Instruct`.

The project has evolved through three connected stages:

```text
Vanilla SFT + Evidence
        ↓
Failure Analysis + Targeted SFT
        ↓
Execution-Aware Candidate Selection
```

The goal is not to reproduce the compute scale of the full R³-SQL system. The
goal is to identify and recover execution-level headroom using a reproducible,
resource-efficient pipeline.

## 2. Alignment Problem

Generator SFT minimizes next-token cross-entropy, but Text-to-SQL is evaluated
using execution correctness.

A generated SQL query can be:

- syntactically valid but semantically incorrect;
- textually close to the gold SQL but execution-incorrect;
- textually different from the gold SQL but execution-equivalent.

The project therefore separates two bottlenecks:

- **Generation bottleneck:** no correct SQL appears in the candidate pool.
- **Selection bottleneck:** a correct SQL is generated but is not ranked first.

## 3. Experimental Contract

### Model and training

- Base model: `Qwen/Qwen2.5-Coder-7B-Instruct`
- Adaptation: BF16 LoRA
- Maximum training length: `8192` tokens
- Training objective: completion-only loss

### Data

- Training source: BIRD train split
- Train/validation separation: database-level split
- Held-out evaluation: BIRD MiniDev500 (`N = 500`)
- Primary inference condition: question, schema, and BIRD evidence

MiniDev500 is used for evaluation, Oracle measurement, and failure analysis.
Its questions and gold SQL are not copied into the targeted-SFT training set.

### Metrics

- `EX@1`
- `Oracle@3`
- `Oracle@5`
- valid SQL rate
- performance by difficulty and database
- paired improvements and regressions
- candidate diversity and latency

## 4. Completed Stage I — Vanilla SFT

The fixed four-way baseline compares:

1. Base without evidence
2. Base with evidence
3. Original SFT without evidence
4. Original SFT with evidence

The final vanilla-SFT run uses one epoch, learning rate `2e-5`, cosine
scheduling, warmup ratio `0.03`, weight decay `0.01`, gradient checkpointing,
and validation-loss-based checkpoint selection.

### Finding

Evidence is the dominant source of improvement. Base+Evidence reaches
`50.4% EX@1`, whereas Original-SFT+Evidence reaches `47.6%`.

Original SFT improves SQL validity more consistently than execution accuracy.
This demonstrates the mismatch between token-level adaptation and
execution-level correctness.

## 5. Completed Stage II — Failure Analysis and Targeted SFT

Failure analysis compares paired Base+Evidence and Original-SFT+Evidence
predictions. Outputs are first divided into correct, valid-but-wrong, and
invalid SQL states.

Valid-but-wrong cases are analyzed using four training families:

- schema relationships;
- predicates and evidence;
- computation logic;
- result structure.

Automatic failure-mode candidates are calibrated using human adjudication.
They are treated as analysis and retrieval signals rather than perfect
ground-truth labels.

### Targeted dataset

| Training family | Prompts | Share |
|---|---:|---:|
| General replay | 2,039 | 35.0% |
| Schema relationship | 1,514 | 26.0% |
| Predicate/evidence | 833 | 14.3% |
| Computation logic | 757 | 13.0% |
| Result structure | 682 | 11.7% |

The targeted adapter uses the same one-epoch, `2e-5`, 8192-token LoRA setup as
the original adapter to preserve a controlled comparison.

### Finding

Targeted-SFT+Evidence reaches `44.0% EX@1`, below both Base+Evidence and
Original-SFT+Evidence. Concentrating known failure families in token-level
supervision does not recover execution accuracy under the current design.

## 6. Completed Oracle Diagnostic

A single `K = 5` candidate run provides `EX@1`, `Oracle@3`, and `Oracle@5` for
the same MiniDev500 snapshot.

| Generator + Evidence | EX@1 | Oracle@3 | Oracle@5 | Oracle@5 gap | Valid SQL@1 |
|---|---:|---:|---:|---:|---:|
| Base | **50.4%** | 55.8% | **60.8%** | +10.4 pp | 89.4% |
| Original SFT | 47.6% | **56.2%** | 60.0% | +12.4 pp | **91.0%** |
| Targeted SFT | 44.0% | 55.4% | 59.6% | **+15.6 pp** | 88.8% |

### Interpretation

All three generators have meaningful candidate-recall headroom. Targeted SFT
has the weakest top-1 ranking but retains an Oracle@5 close to the other
models. Its `15.6` percentage-point Oracle gap indicates that ranking degraded
more than candidate recall.

This result moves the project toward execution-aware selection before another
generator-training intervention.

## 7. R³-SQL-lite Pipeline

```text
Question + Schema + Evidence
            ↓
     Generate K candidates
            ↓
     Execute each candidate
            ↓
 Canonicalize execution results
            ↓
 Group execution-equivalent SQL
            ↓
     Rank execution groups
            ↓
    Confidence assessment
       ┌────┴────┐
       │         │
   Confident  Uncertain
       │         │
       │     Resample
       │         │
       └────┬────┘
            ↓
       Final SQL
```

The system must never use MiniDev gold execution to select a candidate. Gold
results are used only after selection to compute evaluation metrics and Oracle
upper bounds.

## 8. Next Stage — Execution Grouping

Each candidate execution should store:

```text
example_id
db_id
candidate_index
predicted_sql
execution_valid
execution_result
execution_error
execution_time
```

Execution results must be canonicalized before grouping. The implementation
must handle:

- stable serialization of `NULL`, numbers, and text;
- numeric tolerance where appropriate;
- row and scalar container normalization;
- duplicate rows;
- ordering only when order is not semantically required;
- timeouts and database errors.

Safeguards:

- Successful execution does not imply semantic correctness.
- An empty result must not automatically be rejected.
- Invalid candidates form an error group.
- Result order must be preserved when the question requires ordering or Top-K.

## 9. Immediate Experiment — Functional Majority Voting

Functional Majority Voting selects the largest execution-equivalent candidate
group:

```text
K=5 candidates
       ↓
Execute and group
       ↓
Choose largest valid group
       ↓
Return representative SQL
```

The first FMV experiment should reuse the saved K=5 candidates for all three
with-evidence generators. No new GPU generation run is required.

### Tie breaking

When two execution groups have the same size:

1. prefer the group containing the higher-ranked original candidate;
2. prefer a valid non-error group;
3. use mean generation score if it was saved reliably;
4. otherwise use a deterministic group identifier.

### Evaluation

Compare:

```text
EX@1
FMV@5
Oracle@5
```

Define Oracle gap closed as:

```text
(FMV@5 - EX@1) / (Oracle@5 - EX@1)
```

FMV should be evaluated for Base, Original SFT, and Targeted SFT. Base+Evidence
is the primary deployable baseline because it currently has the best EX@1,
Oracle@5, and inference efficiency.

## 10. Zero-Shot Group Verifier

If FMV does not recover enough Oracle headroom, add a zero-shot verifier. The
verifier receives:

```text
Question
Schema
Evidence
Representative SQL for each group
Normalized execution result
Group size
```

It ranks execution groups by semantic consistency with the question. Group
ranking avoids redundant comparisons among execution-equivalent SQL strings.

The verifier should be tested for candidate-order sensitivity, repeated-call
consistency, latency, and bias toward plausible but semantically wrong
majority groups.

Compare:

```text
EX@1
FMV@5
Zero-shot verifier@5
Oracle@5
```

## 11. Selective Resampling

Additional candidates should be generated only for uncertain examples.
Potential triggers include:

- no dominant execution group;
- all candidates fail execution;
- similar scores for the top two groups;
- verifier disagreement after group-order reversal;
- high candidate diversity but low group confidence.

Initial budget:

```text
Initial K = 5
Additional K = 3–5 only when uncertain
```

Compare selective resampling with fixed `K=8` under similar average candidate
budgets. Report EX, latency, candidates per question, and trigger rate.

## 12. Learned Ranker — Conditional Extension

A trained ranker should be implemented only if the zero-shot verifier closes a
meaningful part of the Oracle gap.

Training pairs can be constructed from BIRD training databases:

```text
Chosen:   execution-correct SQL
Rejected: valid but execution-incorrect SQL
```

Useful hard negatives include wrong joins, filters, aggregation, arithmetic,
ordering, and result structure.

Recommended progression:

1. pairwise ranker SFT;
2. evaluate against FMV and the zero-shot verifier;
3. consider DPO only if pairwise SFT shows additional headroom.

DPO, GRPO, and another generator SFT run are not immediate next steps.

## 13. Experiment Matrix

| Phase | Method | Status | Research question |
|---|---|---|---|
| A | Four-way Base/SFT evaluation | Completed | Does vanilla SFT improve EX? |
| B | Failure analysis and adjudication | Completed | Which semantic errors remain? |
| C | Targeted SFT | Completed | Can targeted token supervision repair them? |
| D | EX@1 / Oracle@3 / Oracle@5 | Completed | Generation or selection bottleneck? |
| E | Execution grouping + FMV | **Next** | Can execution consensus improve top-1? |
| F | Zero-shot group verifier | Planned | Can semantic ranking beat FMV? |
| G | Selective resampling | Planned | Can adaptive compute beat fixed K? |
| H | Hard-negative ranker | Conditional | Is trained selection worthwhile? |

## 14. Decision Gates

### Gate 1 — FMV

- If FMV improves EX, continue to group-level verifier experiments.
- If FMV hurts EX, analyze correlated majority-group errors before sampling
  more candidates.

### Gate 2 — Zero-shot verifier

- If the verifier beats FMV and closes meaningful Oracle gap, proceed to
  selective resampling.
- Otherwise, diagnose verifier failure before ranker training.

### Gate 3 — Selective resampling

- Continue only if selective resampling improves the accuracy/compute frontier
  over fixed sampling.

### Gate 4 — Learned ranking

- Train a ranker only when candidate recall and verifier behavior show
  recoverable selection headroom.

## 15. Implementation Structure

Reusable logic should move gradually out of notebooks:

```text
src/
├── generation.py
├── execution.py
├── grouping.py
├── ranking.py
├── resampling.py
└── metrics.py
```

Suggested next notebooks:

```text
09_execution_grouping_fmv.ipynb
10_zero_shot_group_verifier.ipynb
11_selective_resampling.ipynb
12_ranker_training.ipynb
```

Each notebook should answer one research question and write to a separate run
directory.

## 16. Output Contract

Every run should save:

```text
results/<run_name>/
├── run_config.json
├── summary.json
├── per_example.csv
└── figures/
```

Each configuration should record:

```text
model and adapter
dataset snapshot
evaluation size
seed
temperature
K
max_new_tokens
prompt version
evidence setting
execution normalization version
grouping strategy
ranking strategy
```

## 17. Success Criteria

R³-SQL-lite is successful if it provides reproducible answers to:

1. How much correct-candidate recall exists beyond EX@1?
2. How much Oracle gap can FMV or a verifier recover?
3. Which SQL failure modes are corrected or worsened?
4. Does adaptive resampling improve the accuracy/compute trade-off?
5. Are execution-derived hard negatives useful for learned ranking?

A strong progression would be:

```text
Vanilla and targeted SFT
        ↓
limited or negative EX improvement
        ↓
Oracle analysis reveals candidate headroom
        ↓
execution-aware selection recovers part of the headroom
        ↓
adaptive search improves accuracy per unit of compute
```

Even if FMV or verification does not improve EX, the project remains useful if
it identifies why Oracle headroom cannot be recovered and links the failures
to candidate diversity, schema reasoning, or correlated semantic errors.

## 18. Current Status

```text
[Completed] Database-level split and 8192-token audit
[Completed] Vanilla LoRA SFT
[Completed] MiniDev500 four-way evaluation
[Completed] Failure-mode analysis and human adjudication
[Completed] Targeted-SFT dataset construction
[Completed] Targeted LoRA SFT
[Completed] Six-way Base/SFT/Targeted evaluation
[Completed] EX@1 / Oracle@3 / Oracle@5 diagnostic

[Next]      Execution-result canonicalization and grouping
[Next]      Functional Majority Voting on saved K=5 candidates
[Planned]   Zero-shot group verifier
[Planned]   Selective resampling
[Conditional] Hard-negative ranker SFT / DPO
```
