# Experiment Log

## 1. Project Scope

This project studies whether supervised fine-tuning (SFT) improves execution-level Text-to-SQL performance on BIRD using `Qwen/Qwen2.5-Coder-7B-Instruct`. It compares the base model, an original LoRA SFT model, and a failure-mode-targeted LoRA SFT model. Each generator is evaluated both with and without BIRD evidence on the same frozen MiniDev500 benchmark.

The primary metric is execution accuracy (`EX@1`). `Oracle@3` and `Oracle@5` measure whether a correct SQL query appears anywhere among multiple sampled candidates and therefore separate candidate-generation recall from candidate-selection quality. Earlier 200-example evaluations were used only for pipeline debugging and are not treated as final benchmark results.

## 2. Data and Training Configuration

The original SFT data used a database-level train/validation split and mixed evidence exposure with evidence dropout. Because some BIRD schemas are long, the final training and evaluation pipeline used an 8,192-token input limit.

The original and targeted adapters used the same controlled LoRA setup:

| Setting | Value |
|---|---|
| Base model | `Qwen/Qwen2.5-Coder-7B-Instruct` |
| Training precision | BF16 |
| Epochs | 1 |
| Per-device batch size | 2 |
| Gradient accumulation | 4 |
| Learning rate | `2e-5` |
| Scheduler | Cosine |
| Warmup ratio | 0.03 |
| Weight decay | 0.01 |
| Maximum length | 8,192 |
| Loss | Completion-only next-token loss |
| Checkpoint selection | Validation loss |

The targeted dataset contains 5,825 prompts:

| Training family | Prompts | Share |
|---|---:|---:|
| General replay | 2,039 | 35.0% |
| Schema relationship | 1,514 | 26.0% |
| Predicate/evidence | 833 | 14.3% |
| Computation logic | 757 | 13.0% |
| Result structure | 682 | 11.7% |

## 3. MiniDev500 Six-Way Evaluation

### 3.1 Primary results

| Generator | Evidence | EX@1 | Oracle@3 | Oracle@5 | Valid SQL@1 |
|---|---|---:|---:|---:|---:|
| Base | No | 27.0% | 32.6% | 36.6% | 86.0% |
| Original SFT | No | 27.0% | 36.0% | 39.8% | 90.0% |
| Targeted SFT | No | **28.2%** | 36.0% | **40.8%** | **91.4%** |
| Base | Yes | **50.4%** | 55.8% | **60.8%** | 89.4% |
| Original SFT | Yes | 47.6% | **56.2%** | 60.0% | **91.0%** |
| Targeted SFT | Yes | 44.0% | 55.4% | 59.6% | 88.8% |

Evidence-conditioned evaluation is the primary comparison because evidence is both the largest performance driver and the strongest practical inference setting.

### 3.2 Difficulty-level comparison with evidence

| Difficulty | Original SFT EX@1 | Targeted SFT EX@1 |
|---|---:|---:|
| Simple | 68.9% | 64.2% |
| Moderate | 40.8% | 37.6% |
| Challenging | 33.3% | 30.4% |

Targeted SFT underperforms Original SFT across all three difficulty groups.

## 4. Paired Effects and Statistical Tests

Paired Base+Evidence → Original-SFT+Evidence transitions across 500 examples are:

| Transition | Count |
|---|---:|
| Correct → Correct | 205 |
| Valid wrong → Valid wrong | 154 |
| Improved to correct | 33 |
| Regressed from correct | 47 |
| Other non-correct transitions | 61 |

Original SFT therefore fixes some Base errors, but introduces more regressions than improvements.

For the targeted model, the paired EX@1 decreases are statistically significant:

- Targeted SFT versus Base: McNemar `p = 0.00073`;
- Targeted SFT versus Original SFT: McNemar `p = 0.00210`.

The lower targeted-SFT score should therefore not be described as random evaluation noise.

## 5. Failure-Mode Analysis and Human Adjudication

Failure-mode analysis v3 was calibrated using 116 manually audited cases:

| Adjudication status | Count |
|---|---:|
| Confirmed model error | 97 |
| Prediction more correct than gold | 13 |
| Both prediction and gold problematic | 4 |
| Prediction semantically equivalent | 2 |

The automatic v3 classifier achieved approximately `52.5%` accuracy and `42.1%` macro-F1. Its primary-plus-secondary candidate recall was `87.1%`.

Automatic failure labels are therefore useful for aggregate diagnosis and candidate-family retrieval, but they are not high-precision ground truth. Human adjudication remains necessary for calibration and for identifying ambiguous or incorrect benchmark labels.

## 6. Main Findings

### 6.1 Evidence dominates SFT

For the Base model, evidence increases EX@1 from `27.0%` to `50.4%`, an absolute gain of `23.4` percentage points. This is substantially larger than the effect of either SFT intervention.

### 6.2 Original SFT improves form more reliably than semantics

Original SFT increases valid SQL rates, but does not improve EX@1 without evidence and reduces evidence-conditioned EX@1 from `50.4%` to `47.6%`. Token-level optimization improves formatting and syntactic reliability more consistently than execution correctness.

### 6.3 Targeted SFT does not repair the objective mismatch

Targeted SFT reaches only `44.0%` EX@1 with evidence, which is `6.4` percentage points below Base+Evidence and `3.6` points below Original-SFT+Evidence. Concentrating known failure families in token-level supervision does not recover execution accuracy under the current setup.

### 6.4 Candidate recall remains strong

Despite its lower EX@1, Targeted-SFT+Evidence reaches `59.6%` Oracle@5, close to Base+Evidence (`60.8%`) and Original-SFT+Evidence (`60.0%`). Its `15.6`-point EX@1-to-Oracle@5 gap is the largest of the three generators.

The system can often generate a correct SQL candidate but fails to place it first. The immediate bottleneck is therefore candidate selection as well as generation.

## 7. Decision Record

The completed experiments establish the following decisions:

1. Keep Base+Evidence as the strongest current single-candidate baseline.
2. Preserve both SFT adapters and their evaluations as controlled negative results.
3. Do not run another undifferentiated generator-SFT experiment immediately.
4. Reuse the saved K=5 candidates before spending compute on new generation.
5. Test execution-aware grouping and selection before DPO or ranker training.
6. Treat automatic failure-mode labels as diagnostic aids rather than gold labels.

## 8. Next Stage: R³-SQL-Lite Candidate Selection

The next implementation stage is:

```text
Reuse saved K=5 candidates
        ↓
Canonicalize execution outputs
        ↓
Group execution-equivalent SQL
        ↓
Evaluate Functional Majority Voting
        ↓
Add a zero-shot group verifier if needed
        ↓
Use selective resampling for uncertain examples
```

The first gate is whether Functional Majority Voting or a lightweight verifier improves EX@1 over the saved first candidate without reducing Oracle coverage. Pairwise ranker SFT or execution-guided DPO should be considered only after this selection-stage headroom is measured.

## 9. Reproducibility Artifacts

The repository should retain:

- the notebooks used for data preparation, training, six-way evaluation, failure-mode analysis, and targeted SFT;
- the six per-run evaluation configuration files under `configs/evaluation/`;
- summary, detailed, and figure artifacts under `results/`;
- `docs/sft_findings.md` for the concise research interpretation;
- `docs/r3_sql_lite_plan.md` for the next-stage implementation plan.

The six real per-run configuration files should be preserved under their original names. Separate synthetic files named `six_way_evaluation_config.json` and `oracle_k_config.json` are not required.
