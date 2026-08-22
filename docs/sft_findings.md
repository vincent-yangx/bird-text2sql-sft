# Findings from Vanilla and Targeted SFT

## 1. Scope

This document summarizes the supervised fine-tuning findings from the BIRD
Text-to-SQL project using `Qwen/Qwen2.5-Coder-7B-Instruct` and MiniDev500.

The experiments compare:

1. Base model
2. Original LoRA SFT
3. Failure-mode-targeted LoRA SFT

Each generator is evaluated with and without BIRD evidence. The main analysis
focuses on the with-evidence conditions because evidence is the largest
performance driver and represents the strongest practical prompt setting.

## 2. Objective Mismatch

Generator SFT minimizes next-token cross-entropy, while the task is evaluated
using execution correctness.

A SQL query can be:

- syntactically valid but semantically incorrect;
- textually similar to the gold SQL but execution-incorrect;
- textually different from the gold SQL but execution-equivalent.

Therefore, lower training loss, higher normalized exact match, or higher valid
SQL rate does not guarantee higher execution accuracy.

## 3. Main MiniDev500 Results

### With-evidence comparison

| Generator | EX@1 | Oracle@3 | Oracle@5 | Oracle@5 gap | Valid SQL@1 |
|---|---:|---:|---:|---:|---:|
| Base | **50.4%** | 55.8% | **60.8%** | +10.4 pp | 89.4% |
| Original SFT | 47.6% | **56.2%** | 60.0% | +12.4 pp | **91.0%** |
| Targeted SFT | 44.0% | 55.4% | 59.6% | **+15.6 pp** | 88.8% |

### Without-evidence comparison

| Generator | EX@1 | Oracle@3 | Oracle@5 | Valid SQL@1 |
|---|---:|---:|---:|---:|
| Base | 27.0% | 32.6% | 36.6% | 86.0% |
| Original SFT | 27.0% | 36.0% | 39.8% | 90.0% |
| Targeted SFT | **28.2%** | 36.0% | **40.8%** | **91.4%** |

Without evidence, targeted SFT provides only a small EX@1 improvement and
remains far below the evidence-conditioned systems. The project therefore
treats evidence conditioning as part of the primary inference contract.

## 4. Finding 1 — Evidence Dominates SFT

For the base model, evidence improves EX@1 from `27.0%` to `50.4%`, an absolute
gain of `23.4` percentage points.

This effect is substantially larger than any improvement produced by vanilla
or targeted SFT. Correct schema grounding and task-specific evidence are
therefore more important than additional token-level adaptation in the current
setup.

## 5. Finding 2 — Vanilla SFT Improves Form More Than Semantics

Original SFT does not improve EX@1 without evidence and underperforms the base
model with evidence:

```text
Base + evidence:         50.4%
Original SFT + evidence: 47.6%
Delta:                   -2.8 pp
```

At the same time, Original SFT increases valid SQL@1 from `89.4%` to `91.0%`.
This indicates that SFT improves formatting and syntactic reliability more
consistently than execution-level semantic correctness.

## 6. Finding 3 — SFT Produces Both Improvements and Regressions

Paired Base+Evidence → Original-SFT+Evidence transitions on 500 examples show:

| Transition | Count |
|---|---:|
| Correct → Correct | 205 |
| Valid wrong → Valid wrong | 154 |
| Improved to correct | 33 |
| Regressed from correct | 47 |
| Other non-correct transitions | 61 |

The model fixes some errors but introduces more regressions than improvements.
The negative aggregate result is therefore not caused by a total lack of
learning; it is caused by unstable changes across semantic cases.

## 7. Finding 4 — Failure Labels Require Human Calibration

The final adjudication contains 116 audited cases:

| Adjudication status | Count |
|---|---:|
| Confirmed model error | 97 |
| Prediction more correct than gold | 13 |
| Both prediction and gold problematic | 4 |
| Prediction semantically equivalent | 2 |

The automatic v3 classifier achieves approximately `52.5%` accuracy and
`42.1%` macro-F1, although primary-plus-secondary candidate recall reaches
`87.1%`.

Consequently, automatic labels are useful for generating candidate failure
families and building aggregate diagnostics, but they should not be presented
as high-precision ground truth. Human adjudication is required for calibration
and for detecting benchmark ambiguities.

## 8. Finding 5 — Targeted SFT Does Not Recover EX@1

The targeted dataset contains `5,825` prompts:

| Training family | Prompts | Share |
|---|---:|---:|
| General replay | 2,039 | 35.0% |
| Schema relationship | 1,514 | 26.0% |
| Predicate/evidence | 833 | 14.3% |
| Computation logic | 757 | 13.0% |
| Result structure | 682 | 11.7% |

The targeted adapter uses the same one-epoch, learning-rate `2e-5`,
8192-token LoRA setup as the original adapter.

With evidence, targeted SFT reaches `44.0% EX@1`:

- `-6.4` percentage points versus Base+Evidence;
- `-3.6` percentage points versus Original-SFT+Evidence.

The paired EX@1 differences are statistically significant:

- Targeted versus Base: McNemar `p = 0.00073`;
- Targeted versus Original SFT: McNemar `p = 0.00210`.

Concentrating known failure families in token-level supervision therefore does
not repair the execution-level objective mismatch under the current training
design.

## 9. Finding 6 — Candidate Recall Is Stronger Than Top-1 Ranking

Although targeted SFT has lower EX@1, its Oracle@5 is `59.6%`, close to:

- Base+Evidence Oracle@5: `60.8%`;
- Original-SFT+Evidence Oracle@5: `60.0%`.

Its Oracle@5 gap is `15.6` percentage points, the largest of the three
generators. This means the targeted model often generates a correct SQL among
five candidates but fails to place it first.

The result identifies recoverable candidate-selection headroom and motivates
execution grouping and ranking before another generator-training experiment.

## 10. Difficulty-Level Observation

Evidence-conditioned targeted SFT underperforms Original SFT at EX@1 across
all three BIRD difficulty groups:

| Difficulty | Original SFT EX@1 | Targeted SFT EX@1 |
|---|---:|---:|
| Simple | 68.9% | 64.2% |
| Moderate | 40.8% | 37.6% |
| Challenging | 33.3% | 30.4% |

However, the Oracle gaps remain substantial, particularly for moderate and
challenging examples. This reinforces the need to separate candidate recall
from candidate selection.

## 11. Engineering Interpretation

The targeted-SFT experiment is not wasted. It demonstrates a complete
engineering and research loop:

```text
Baseline evaluation
        ↓
Paired failure analysis
        ↓
Human adjudication
        ↓
Targeted data construction with replay
        ↓
Controlled LoRA training
        ↓
Held-out six-way evaluation
        ↓
Statistical and Oracle diagnosis
```

The negative result prevents additional compute from being spent on another
undifferentiated SFT run and provides evidence for changing the system design.

## 12. Final Conclusion

Vanilla and failure-mode-targeted SFT improve some aspects of SQL generation,
especially syntactic validity and candidate recall, but do not improve
execution accuracy over Base+Evidence.

The current priority is therefore:

```text
Reuse saved K=5 candidates
        ↓
Canonicalize execution results
        ↓
Group execution-equivalent SQL
        ↓
Evaluate Functional Majority Voting
        ↓
Add a zero-shot group verifier if needed
        ↓
Use selective resampling only for uncertain examples
```

Further generator training, pairwise ranker SFT, or DPO should be considered
only after execution-aware selection demonstrates measurable headroom.

Detailed run configurations and complete experiment history are maintained in
`EXPERIMENT_LOG.md`. The next-stage implementation plan is maintained in
`docs/r3_sql_lite_plan.md`.
