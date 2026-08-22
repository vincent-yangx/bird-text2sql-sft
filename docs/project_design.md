# Project Design

## 1. Goal

Build and evaluate a resource-efficient Text-to-SQL pipeline on BIRD using
`Qwen/Qwen2.5-Coder-7B-Instruct`.

The project studies when supervised fine-tuning improves SQL generation and
when execution-aware search and selection are required. The primary objective
is execution correctness rather than textual similarity to a reference SQL
query.

## 2. Research Questions

1. Does LoRA SFT improve execution accuracy over the base model?
2. How much does BIRD evidence improve schema grounding and execution accuracy?
3. Can failure-mode-targeted SFT repair the errors introduced or retained by
   vanilla SFT?
4. When multiple candidates are generated, is the main bottleneck candidate
   generation or top-1 selection?
5. Can execution grouping, verification, and selective resampling close the
   gap between `EX@1` and `Oracle@K`?

## 3. Model and Data

### Model

- Base model: `Qwen/Qwen2.5-Coder-7B-Instruct`
- Adaptation: BF16 LoRA
- Training objective: completion-only next-token cross-entropy

### Data

- Training source: BIRD train split
- Train/validation separation: database-level split
- Training format: question, schema, optional evidence, and gold SQL
- Final held-out evaluation: BIRD MiniDev500 (`N = 500`)

MiniDev500 is reserved for evaluation and failure analysis. Its gold SQL is not
used as targeted-SFT training data.

## 4. Stage I — Vanilla SFT and Evidence Conditioning

The first stage establishes a stable four-way baseline:

1. Base model without evidence
2. Base model with evidence
3. Vanilla SFT without evidence
4. Vanilla SFT with evidence

The final vanilla-SFT configuration uses:

- one epoch;
- learning rate `2e-5`;
- cosine learning-rate scheduling;
- warmup ratio `0.03`;
- weight decay `0.01`;
- maximum sequence length `8192`;
- gradient checkpointing;
- validation-loss-based checkpoint selection.

This stage tests whether token-level adaptation improves execution-level
correctness and quantifies the independent effect of evidence conditioning.

## 5. Stage II — Failure Analysis and Targeted SFT

The second stage analyzes paired Base+Evidence and SFT+Evidence predictions.
Each example is assigned an execution state:

- correct;
- valid but execution-incorrect;
- invalid SQL.

Valid-but-wrong cases are further analyzed using a failure-mode taxonomy that
covers schema relationships, predicates and evidence use, computation logic,
and result structure. Automated labels are calibrated against human
adjudication before being used for aggregate analysis.

The targeted-SFT dataset contains `5,825` prompts:

- `2,039` general-replay prompts to preserve broad capability;
- `3,786` targeted prompts selected from related BIRD training examples.

Targeted examples are selected from the training split using structural and
failure-family signals. MiniDev500 questions and gold SQL are never copied into
the training set.

The targeted adapter is trained with the same one-epoch, `2e-5`, 8192-token
LoRA configuration so that its effect can be compared with the original SFT
adapter under a controlled setup.

## 6. Stage III — Execution-Aware Candidate Selection

The final stage moves from single-pass generation to an execution-aware
pipeline:

```text
Question + Schema + Evidence
            ↓
     Generate K SQL candidates
            ↓
       Execute candidates
            ↓
   Group by execution result
            ↓
     Verify and rank groups
            ↓
   Selective resampling if uncertain
            ↓
          Final SQL
```

The initial diagnostic measures:

- `EX@1`: execution accuracy of the first candidate;
- `Oracle@3`: whether any of the first three candidates is correct;
- `Oracle@5`: whether any of the first five candidates is correct.

A large Oracle gap indicates that the generator can produce correct SQL but
cannot reliably rank it first. This motivates Functional Majority Voting,
execution-group ranking, a verifier, and selective resampling before additional
generator fine-tuning.

## 7. Evaluation

### Primary metrics

- Execution Accuracy at rank 1 (`EX@1`)
- `Oracle@3`
- `Oracle@5`

### Supporting metrics

- valid SQL rate;
- normalized exact match;
- performance by BIRD difficulty;
- paired improvement and regression counts;
- bootstrap confidence intervals and McNemar tests;
- candidate diversity and inference latency.

All main model comparisons use the same MiniDev500 snapshot, prompt contract,
SQL extraction logic, execution matcher, and evaluation size.

## 8. Current Findings

- Evidence conditioning is the largest source of improvement.
- Vanilla SFT improves SQL validity more consistently than execution accuracy.
- Targeted SFT does not improve `EX@1` over either Base+Evidence or the original
  SFT model.
- Targeted SFT retains an `Oracle@5` close to the other models despite its lower
  top-1 accuracy, indicating a substantial selection bottleneck.
- The next implementation priority is execution grouping and candidate
  selection rather than another round of undifferentiated SFT.

Detailed configurations and numerical results are maintained in
`EXPERIMENT_LOG.md`; the execution-aware roadmap is maintained in
`docs/r3_sql_lite_plan.md`.

## 9. Current Status

```text
[Completed] Data preparation and database-level splitting
[Completed] Vanilla LoRA SFT
[Completed] MiniDev500 four-way evaluation
[Completed] EX@1 / Oracle@3 / Oracle@5 evaluation
[Completed] Failure-mode analysis and human adjudication
[Completed] Targeted dataset construction
[Completed] Targeted LoRA SFT
[Completed] Six-way Base/SFT/Targeted evaluation

[Next]      Execution-result grouping
[Next]      Functional Majority Voting
[Planned]   Verifier or group-level ranker
[Planned]   Selective resampling
```
