# R³-SQL-lite: Execution-Aligned Text-to-SQL Plan

## 1. Motivation

The first stage of this project investigated whether supervised fine-tuning (SFT) improves Text-to-SQL performance on BIRD using Qwen2.5-Coder-7B-Instruct.

The initial experiments compared four configurations:

1. Base model without evidence
2. Base model with evidence
3. SFT model without evidence
4. SFT model with evidence

The experiments showed that vanilla SFT does not consistently translate lower token-level training loss into higher execution accuracy.

This motivates a shift in perspective.

SFT optimizes token-level likelihood:

[
\mathcal{L}_{SFT}
=================

-\sum_t \log P_\theta(y_t \mid x, y_{<t})
]

where the target is the gold SQL sequence.

However, the final Text-to-SQL objective is execution-level correctness:

[
EX =
\mathbf{1}
\left[
Exec(\hat{y}) = Exec(y^*)
\right]
]

These objectives are related but not identical.

A SQL query can be:

* syntactically valid but semantically incorrect;
* textually different from the gold SQL but execution-equivalent;
* very close to the gold SQL in token space while differing in a critical operator such as `DISTINCT`, `JOIN`, `GROUP BY`, or a filtering condition.

Therefore, the next stage of the project focuses on **execution-aware search and verification rather than relying only on additional generator fine-tuning**.

The proposed system is called **R³-SQL-lite**, inspired by the ranking-and-resampling paradigm of R³-SQL but redesigned for a resource-constrained single-GPU setting.

---

# 2. Main Research Question

The central question is:

> Is the current Text-to-SQL bottleneck primarily a generation problem or a selection problem?

More specifically:

### Generation problem

The model rarely generates a correct SQL query even after multiple attempts.

If this is the case, improvements should focus on:

* better schema grounding;
* reasoning-oriented training;
* targeted SFT;
* hard-example training;
* preference optimization;
* improved prompting.

### Selection problem

The model frequently generates a correct SQL query among several candidates, but the top-1 prediction is often incorrect.

If this is the case, improvements should focus on:

* candidate sampling;
* execution grouping;
* verification;
* ranking;
* selective resampling.

R³-SQL-lite begins by explicitly measuring which of these two problems dominates.

---

# 3. High-Level Pipeline

The planned pipeline is:

```text
Question + Schema + Optional Evidence
                │
                ▼
        Candidate Generation
                │
                ▼
       K SQL Candidates
                │
                ▼
         SQL Execution
                │
                ▼
       Execution Filtering
                │
                ▼
       Execution Grouping
                │
                ▼
     Candidate / Group Ranking
                │
        ┌───────┴────────┐
        │                │
   Confident         Uncertain
        │                │
        │                ▼
        │        Selective Resampling
        │                │
        │                ▼
        │         New Candidate Pool
        │                │
        └───────┬────────┘
                ▼
        Final SQL Selection
```

The implementation will be introduced incrementally rather than building the full system at once.

---

# 4. Stage 0 — Existing Baseline

Before introducing R³-SQL-lite, the following systems have already been evaluated:

```text
Base Model
├── without evidence
└── with evidence

SFT Model
├── without evidence
└── with evidence
```

These experiments serve as the fixed baseline for all subsequent execution-aware experiments.

The main metric is:

* Execution Accuracy (EX)

Additional metrics may include:

* valid SQL rate;
* normalized exact match;
* inference latency;
* performance by difficulty;
* performance by database.

The existing four-way evaluation should remain unchanged so that later R³-SQL-lite results can be compared against a stable reference point.

---

# 5. Stage 1 — Candidate Recall Analysis

## 5.1 Goal

Before building a verifier or ranking model, determine whether multiple sampling already exposes significant hidden capability in the generator.

The key diagnostic is:

> Can the model generate a correct SQL query among K attempts even when its top-1 answer is wrong?

---

## 5.2 Candidate Sampling

For each evaluation example, generate multiple SQL candidates.

Initial configuration:

```text
K = 5
temperature ≈ 0.7–0.8
do_sample = True
```

A resource-efficient first experiment should use approximately 200 MiniDev examples before scaling to the full MiniDev500 set.

One generation call should return several sequences whenever possible.

Example:

```python
outputs = model.generate(
    **inputs,
    do_sample=True,
    temperature=0.8,
    num_return_sequences=5,
    max_new_tokens=MAX_NEW_TOKENS,
)
```

This is preferable to repeatedly invoking the model five separate times for the same prompt.

---

# 6. Stage 2 — EX@1 and Oracle@K

## 6.1 EX@1

EX@1 is the normal execution accuracy of the first candidate:

[
EX@1 =
\frac{1}{N}
\sum_{i=1}^{N}
\mathbf{1}
[
Exec(y_{i,1}) =
Exec(y_i^*)
]
]

Existing top-1 evaluation results should be reused whenever possible rather than recomputed.

---

## 6.2 Oracle@K

Oracle@K measures whether **at least one** candidate among the first K candidates is execution-correct.

[
Oracle@K =
\frac{1}{N}
\sum_{i=1}^{N}
\mathbf{1}
[
\exists j \leq K :
Exec(y_{i,j}) =
Exec(y_i^*)
]
]

Planned metrics:

```text
EX@1
Oracle@3
Oracle@5
```

A single K=5 sampling run is sufficient to compute all three metrics.

---

## 6.3 Interpretation

### Case A — Large Oracle Gap

Example:

```text
EX@1       = 48%
Oracle@3   = 57%
Oracle@5   = 63%
```

Interpretation:

The model frequently produces the correct solution, but does not reliably place it at rank 1.

This indicates a **selection bottleneck**.

Priority should move toward:

```text
Execution Grouping
→ Verification
→ Ranking
→ Selective Resampling
```

### Case B — Small Oracle Gap

Example:

```text
EX@1       = 48%
Oracle@3   = 50%
Oracle@5   = 51%
```

Interpretation:

The model rarely produces a correct SQL query even after multiple attempts.

This indicates a **generation bottleneck**.

Priority should move toward:

```text
Schema Grounding
→ Targeted Training
→ Reasoning
→ Hard-Example SFT
→ Preference Optimization
```

This diagnostic is therefore the first decision point of R³-SQL-lite.

---

# 7. Stage 3 — Execute Every Candidate

Each generated SQL candidate should be executed against its corresponding SQLite database.

For each candidate, store:

```text
example_id
db_id
candidate_index
predicted_sql
execution_valid
execution_result
execution_error
execution_match
generation_time
```

Possible execution states:

```text
1. Valid SQL + non-empty result
2. Valid SQL + empty result
3. Execution error
```

Execution errors can safely identify invalid candidates.

However:

> An empty result must not automatically be treated as incorrect.

The correct answer to some database questions may legitimately be an empty set.

Likewise:

> Successful execution does not imply semantic correctness.

A syntactically valid SQL query can still answer the wrong question.

---

# 8. Stage 4 — Execution Grouping

Multiple SQL queries may have different surface forms but produce the same result.

For example:

```sql
SELECT COUNT(*) FROM students WHERE age > 20;
```

and

```sql
SELECT COUNT(student_id) FROM students WHERE age > 20;
```

may produce the same execution output.

Instead of treating all SQL strings independently, R³-SQL-lite groups candidates according to their execution results.

Example:

```text
Candidate 1 → Result A
Candidate 2 → Result A
Candidate 3 → Result B
Candidate 4 → Result A
Candidate 5 → Execution Error
```

This produces:

```text
Execution Group A
├── Candidate 1
├── Candidate 2
└── Candidate 4

Execution Group B
└── Candidate 3

Invalid
└── Candidate 5
```

The key idea is:

> Ranking execution-equivalent groups is often more aligned with Text-to-SQL semantics than ranking SQL strings independently.

---

# 9. Stage 5 — Functional Majority Voting

The first selection baseline will require no additional model.

For every example:

1. execute all candidates;
2. group candidates by execution result;
3. count the number of candidates in each group;
4. choose the SQL from the largest group.

This method is called **Functional Majority Voting (FMV)**.

Pipeline:

```text
K Candidates
     ↓
Execution
     ↓
Group by Result
     ↓
Largest Group
     ↓
Final SQL
```

The result should be compared against:

```text
EX@1
Oracle@K
FMV@K
```

This establishes three levels:

```text
EX@1
│
│ actual top-1 generator performance
│
FMV@K
│
│ simple execution-based selection
│
Oracle@K
│
│ theoretical upper bound of current candidate pool
```

The gap between these metrics identifies where improvement is possible.

---

# 10. Stage 6 — Zero-Shot LLM Verifier

FMV assumes that the largest execution group is the most likely answer.

This assumption may fail when several candidates independently make the same semantic mistake.

The next step is therefore an LLM-based verifier.

The verifier receives:

```text
Question
Schema
Optional evidence

Candidate A
Execution Result A

Candidate B
Execution Result B
```

and predicts which candidate more faithfully answers the original question.

Example verifier prompt:

```text
You are evaluating two candidate SQL queries for a
Text-to-SQL task.

Given the database schema, user question, SQL queries,
and their execution results, determine which candidate
more accurately answers the question.

Do not judge based only on SQL syntax.
Focus on the intended semantics of the question.

Return only:
A
or
B
```

Initially, the existing Qwen2.5-Coder-7B-Instruct model can serve as the verifier without additional training.

This provides a low-cost experiment before investing in a dedicated ranker.

---

# 11. Stage 7 — Group-Level Ranking

Instead of ranking individual SQL queries, ranking should operate primarily on execution groups.

For each group, useful signals include:

```text
group_size
representative_sql
execution_result
generation_probability, if available
verifier_score
candidate diversity
```

A simple initial score could be conceptually represented as:

[
Score(G)
========

\alpha \cdot GroupFrequency(G)
+
\beta \cdot VerifierScore(G)
]

The exact formula does not need to be fixed initially.

The goal is to compare:

```text
Top-1 Generation
vs
FMV
vs
LLM Ranking
vs
Oracle@K
```

If LLM ranking closes a significant fraction of the Oracle gap, ranking becomes a promising direction.

---

# 12. Stage 8 — Selective Resampling

Generating more SQL queries for every example can be expensive and may introduce additional noisy candidates.

Therefore, additional generation should only occur when the current candidate pool appears unreliable.

Possible uncertainty signals include:

### Signal 1 — No dominant execution group

Example:

```text
Group A: 2
Group B: 2
Group C: 1
```

There is no strong consensus.

### Signal 2 — Verifier disagreement

The verifier gives inconsistent decisions when candidate ordering changes.

### Signal 3 — Low ranking confidence

The top two candidate groups receive very similar scores.

### Signal 4 — All candidates fail execution

The model needs another generation attempt.

### Signal 5 — Candidate pool appears semantically inconsistent

Several plausible SQL queries produce substantially different answers.

---

## Selective Resampling Flow

```text
Initial K Candidates
        ↓
Execution + Ranking
        ↓
Confidence Check
   ┌────┴─────┐
   │          │
 High         Low
   │          │
Select        ▼
          Generate More
          Candidates
              ↓
          Re-execute
              ↓
          Re-rank
              ↓
          Final SQL
```

The lite version should use a small additional sampling budget.

For example:

```text
Initial K = 5

If uncertain:
additional K = 3–5
```

The goal is not to reproduce the very large inference budget of the full R³-SQL system.

The goal is to determine whether **adaptive computation** produces better accuracy/compute trade-offs than fixed sampling.

---

# 13. Stage 9 — Hard-Negative SQL Ranker

If zero-shot verification shows promising results, the next stage is to train a dedicated verifier/ranker.

Training examples can be automatically constructed from BIRD training databases.

For each training question:

```text
1. Generate multiple SQL candidates.
2. Execute every candidate.
3. Compare its execution result against the gold SQL.
4. Label execution-correct SQL as positive.
5. Label execution-incorrect SQL as negative.
```

The most valuable negatives are not obviously broken SQL queries.

They are **hard negatives**.

Examples:

```sql
Correct:
SELECT COUNT(DISTINCT student_id)
FROM enrollment;
```

```sql
Hard Negative:
SELECT COUNT(student_id)
FROM enrollment;
```

or:

```sql
Correct:
SELECT AVG(salary)
FROM employees
WHERE department = 'Engineering';
```

```sql
Hard Negative:
SELECT AVG(salary)
FROM employees;
```

These SQL queries:

* are syntactically valid;
* look plausible;
* may be close to the gold query;
* fail because of a specific semantic reasoning error.

Such examples directly target the failure modes that token-level SFT may not sufficiently penalize.

---

# 14. Stage 10 — Ranker SFT

The first trained verifier should use a simple supervised objective.

Input:

```text
Question
Schema

Candidate A
Execution A

Candidate B
Execution B
```

Target:

```text
A
```

or:

```text
B
```

The resulting model learns:

[
P(
SQL_A > SQL_B
\mid
Question, Schema, Results
)
]

This is different from generator SFT.

Generator SFT learns:

```text
Question → SQL
```

Verifier SFT learns:

```text
Question + SQL A + SQL B
→ which SQL is semantically better?
```

This may align more closely with the downstream selection problem.

---

# 15. Stage 11 — Preference Optimization

If pairwise ranker SFT shows measurable gains, preference optimization can be explored.

Training pairs can be constructed as:

```text
Chosen:
execution-correct SQL

Rejected:
execution-incorrect hard negative
```

Possible approaches include:

```text
DPO
GRPO
other pairwise or reward-based optimization
```

The purpose of this stage is not simply to optimize token similarity.

Instead, execution feedback is used to construct supervision that better reflects semantic correctness.

This stage should only be attempted after simpler ranking baselines have demonstrated that candidate selection is a meaningful bottleneck.

---

# 16. Error-Aware Analysis

Every stage should be analyzed by SQL difficulty and error type.

Suggested error taxonomy:

```text
Schema Linking
├── Wrong table
├── Wrong column
└── Ambiguous schema mapping

Aggregation
├── COUNT vs COUNT DISTINCT
├── AVG / SUM / MAX / MIN
└── Incorrect GROUP BY

Join Reasoning
├── Missing join
├── Wrong join path
└── Wrong join condition

Filtering
├── Missing condition
├── Incorrect operator
└── Incorrect value grounding

Nested Reasoning
├── Incorrect subquery
├── Incorrect HAVING
└── Incorrect ordering / LIMIT

Set / Comparison Logic
├── IN / NOT IN
├── EXISTS
├── UNION / INTERSECT / EXCEPT
└── comparison logic

Execution / Syntax
├── Invalid SQL
├── Missing table
└── Missing column
```

The goal is to determine not only whether R³-SQL-lite improves EX, but also **which categories of errors it actually fixes**.

---

# 17. Evaluation Metrics

The project should track more than final execution accuracy.

## Generator Metrics

```text
EX@1
Oracle@3
Oracle@5
```

## Selection Metrics

```text
FMV@K
Verifier EX
Ranker EX
Oracle Gap Closed
```

Define:

[
OracleGap = Oracle@K - EX@1
]

and:

[
GapClosed =
\frac{
EX_{method} - EX@1
}{
Oracle@K - EX@1
}
]

This metric measures how much of the available candidate-recall headroom the selection method successfully recovers.

---

## Efficiency Metrics

Track:

```text
average candidates per question
average generation time
average execution time
average total latency
resampling trigger rate
GPU inference cost
```

This is important because execution-aware methods trade additional inference computation for higher accuracy.

The final system should therefore be evaluated on both:

```text
Accuracy
and
Compute / Latency
```

---

# 18. Planned Experiment Matrix

## Phase A — Diagnostic

| Experiment | Sampling | Ranking | Training |
| ---------- | -------: | ------- | -------- |
| EX@1       |        1 | None    | None     |
| Oracle@3   |        3 | Oracle  | None     |
| Oracle@5   |        5 | Oracle  | None     |

Goal:

Determine candidate recall.

---

## Phase B — Execution-Aware Selection

| Experiment         | Sampling | Selection                |
| ------------------ | -------: | ------------------------ |
| Top-1              |        1 | First candidate          |
| FMV@5              |        5 | Majority execution group |
| Zero-shot verifier |        5 | LLM verifier             |
| Oracle@5           |        5 | Oracle                   |

Goal:

Measure the candidate-selection gap.

---

## Phase C — Adaptive Search

| Experiment           | Initial K | Resampling           |
| -------------------- | --------: | -------------------- |
| Fixed K=5            |         5 | No                   |
| Fixed K=8            |         8 | No                   |
| Selective Resampling |         5 | Only uncertain cases |

Goal:

Determine whether adaptive search provides a better accuracy/compute trade-off.

---

## Phase D — Learned Ranking

| Experiment        | Ranker                          |
| ----------------- | ------------------------------- |
| Zero-shot LLM     | Base Qwen                       |
| Ranker SFT        | Hard-negative pairwise training |
| Preference Ranker | DPO / GRPO, if justified        |

Goal:

Determine whether execution-derived supervision improves semantic candidate selection.

---

# 19. Resource-Aware Implementation Strategy

The original R³-SQL framework uses substantially more inference and model capacity than available in this project.

R³-SQL-lite intentionally avoids reproducing that scale.

Recommended progression:

```text
Step 1:
MiniDev200
K = 5
one model

Step 2:
If Oracle@5 shows substantial improvement,
scale to MiniDev500.

Step 3:
Implement FMV.

Step 4:
Implement zero-shot verifier.

Step 5:
Only if ranking works,
introduce selective resampling.

Step 6:
Only if verifier performance shows headroom,
train a dedicated ranker.
```

This prevents unnecessary GPU expenditure before the bottleneck is understood.

---

# 20. Data Leakage Rules

Oracle@K requires access to the gold SQL execution result.

Therefore:

> Oracle@K is an analysis metric only.

Gold execution results must never be used to select the final SQL during actual inference.

Correct usage:

```text
Candidate generation
        ↓
system selects candidate without gold
        ↓
final prediction
        ↓
gold result used only for evaluation
```

Incorrect usage:

```text
generate K candidates
        ↓
compare each against gold execution
        ↓
choose correct candidate
```

The second procedure is valid only for computing the Oracle upper bound, not for reporting a deployable system.

---

# 21. Proposed Code Structure

The R³-SQL-lite implementation should gradually move reusable logic out of notebooks.

```text
src/
├── generation.py
├── execution.py
├── grouping.py
├── ranking.py
├── resampling.py
└── metrics.py
```

Suggested responsibilities:

### `generation.py`

```text
prompt construction
candidate sampling
batched generation
SQL extraction
```

### `execution.py`

```text
safe SQLite execution
timeouts
error handling
result normalization
```

### `grouping.py`

```text
execution result canonicalization
execution-equivalent grouping
group statistics
```

### `ranking.py`

```text
FMV
zero-shot verifier
pairwise comparison
group scoring
```

### `resampling.py`

```text
confidence estimation
resampling trigger
candidate-pool update
```

### `metrics.py`

```text
EX@1
Oracle@K
FMV
Oracle gap
gap closed
latency statistics
```

---

# 22. Proposed Notebook Structure

The next experimental notebooks can be organized as:

```text
06_oracle_k_analysis.ipynb
07_execution_grouping_fmv.ipynb
08_zero_shot_verifier.ipynb
09_selective_resampling.ipynb
10_ranker_training.ipynb
```

Each notebook should address one research question rather than combining the complete pipeline into a single large notebook.

---

# 23. Expected Experimental Outputs

For every major experiment, save:

```text
results/
├── summary/
├── per_example/
├── figures/
└── runs/
```

Example R³-SQL-lite outputs:

```text
oracle_k_summary.csv
oracle_k_per_example.csv

fmv_summary.csv
fmv_per_example.csv

verifier_summary.csv
verifier_per_example.csv

resampling_summary.csv
resampling_per_example.csv
```

Each run should also save its configuration:

```text
model
dataset split
seed
temperature
K
max_new_tokens
prompt version
evidence setting
sampling strategy
ranking strategy
```

---

# 24. Main Ablation Questions

The final project should answer the following questions.

### RQ1

Does vanilla SFT improve execution accuracy over the base code model?

### RQ2

How much higher is Oracle@K than EX@1?

### RQ3

Is the current bottleneck generation or candidate selection?

### RQ4

Does execution grouping improve selection over top-1 generation?

### RQ5

Does an LLM verifier outperform functional majority voting?

### RQ6

Does selective resampling outperform fixed additional sampling under similar compute budgets?

### RQ7

Can a hard-negative trained ranker recover a larger fraction of the Oracle gap?

### RQ8

Which SQL error categories benefit most from execution-aware methods?

---

# 25. Success Criteria

R³-SQL-lite does not need to reproduce the absolute performance of large-scale R³-SQL.

The project is successful if it demonstrates a clear and reproducible progression such as:

```text
Vanilla SFT
    ↓
limited EX improvement
    ↓
Oracle@K reveals hidden candidate recall
    ↓
execution-aware selection improves top-1 accuracy
    ↓
verification closes part of the Oracle gap
    ↓
selective search improves the accuracy/compute trade-off
```

A particularly strong result would demonstrate that:

> A significant portion of Text-to-SQL errors comes not from the inability to generate a correct SQL candidate, but from the inability to identify the correct candidate among plausible alternatives.

Alternatively, if Oracle@K remains close to EX@1, that is also an important finding:

> The primary limitation is generation capability rather than candidate selection, motivating reasoning- or training-oriented improvements.

Both outcomes provide actionable conclusions.

---

# 26. Project Evolution

The project can therefore be viewed as two stages.

## Stage I — Token-Level Adaptation

```text
Qwen2.5-Coder-7B-Instruct
        ↓
LoRA SFT
        ↓
Evidence Conditioning
        ↓
Four-Way Evaluation
```

Research question:

> How much can conventional supervised adaptation improve Text-to-SQL?

---

## Stage II — Execution-Level Alignment

```text
Candidate Sampling
        ↓
Execution
        ↓
Oracle Analysis
        ↓
Execution Grouping
        ↓
Verification / Ranking
        ↓
Selective Resampling
        ↓
Learned Ranker
```

Research question:

> Can execution-aware search and verification improve semantic correctness beyond token-level SFT?

---

# 27. Immediate Next Experiment

The immediate next experiment is:

## Oracle@K Diagnostic

Initial setup:

```text
Dataset:
MiniDev subset (~200 examples)

Model:
Current best-performing 7B configuration

Candidate Count:
K = 5

Sampling:
temperature ≈ 0.7–0.8
do_sample = True

Metrics:
EX@1
Oracle@3
Oracle@5
```

Decision rule:

```text
Large Oracle@5 - EX@1 gap
        ↓
Proceed with execution grouping and ranking

Small Oracle@5 - EX@1 gap
        ↓
Return focus to generation quality and training
```

This experiment is intentionally placed before building additional agent or ranking infrastructure.

---

# 28. Long-Term Goal

The final goal of R³-SQL-lite is not to build a larger generator.

It is to investigate whether Text-to-SQL performance can be improved by changing the system from:

```text
Question
   ↓
Generate One SQL
   ↓
Return
```

into:

```text
Question
   ↓
Generate Candidates
   ↓
Interact with Database
   ↓
Observe Execution
   ↓
Compare Semantic Alternatives
   ↓
Verify
   ↓
Search Again if Necessary
   ↓
Return Final SQL
```

The broader hypothesis is:

> For strong pretrained code models, the next major improvement in Text-to-SQL may come not only from increasing the probability of the gold SQL token sequence, but from better search, execution feedback, semantic verification, and candidate selection.
