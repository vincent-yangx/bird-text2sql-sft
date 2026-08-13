# Experiment Log

## Previous Work Summary

This project evaluates whether supervised fine-tuning (SFT) can improve execution-level Text-to-SQL performance on BIRD using `Qwen/Qwen2.5-Coder-7B-Instruct`. The BIRD training data was prepared with a database-level train/validation split to reduce leakage across questions from the same database, and the final SFT pipeline used mixed evidence exposure with approximately 50% evidence dropout. The processed SFT dataset contained 9,428 training examples and 1,485 validation examples. A token-length audit showed that some BIRD schemas are long enough to make 2K/4K context windows unsafe, so the final v2 training configuration used `max_length=8192`; the audited training examples had p50=668, p90=2,978, p95=6,667, p99=6,719, max=6,823, with 0 examples exceeding 8,192 tokens, while the validation set had p50=741, p90=1,366, p95=1,387, p99=1,423, max=1,470, also with 0 examples exceeding 8,192 tokens. The final LoRA SFT run used BF16 on an NVIDIA A100 40 GB GPU, 1 epoch, batch size 2, gradient accumulation 4, learning rate `2e-5`, cosine scheduling, warmup ratio 0.03, weight decay 0.01, gradient checkpointing, completion-only loss, and validation-loss-based checkpoint selection. Earlier small 200-example development evaluations are treated only as legacy debugging runs and are not used as final benchmarks. The main project conclusion from the SFT stage is that token-level likelihood optimization does not necessarily translate into higher execution accuracy, motivating the next stage of execution-aware candidate generation, Oracle@K analysis, execution grouping, verification, ranking, and selective resampling.

## MiniDev500 Four-Way Evaluation — 8192-Token Input Limit

After increasing the input token-length limit to `8192`, the model was evaluated on the same MiniDev500 set under four conditions: Base/SFT × with/without evidence.

| Experiment | Model | Evidence | EX | Valid SQL | Normalized EM | Avg Generation (s) | Avg Input Tokens | Avg Output Tokens |
|---|---|---|---:|---:|---:|---:|---:|---:|
| `baseline_no_evidence` | Base | No | **27.00%** | 86.00% | 6.40% | 1.767 | 940.0 | 55.9 |
| `baseline_with_evidence` | Base | Yes | **50.40%** | 89.40% | 9.40% | 1.827 | 971.8 | 57.7 |
| `sft_no_evidence` | SFT | No | **27.00%** | 90.00% | 7.00% | 4.066 | 940.0 | 58.9 |
| `sft_with_evidence` | SFT | Yes | **47.60%** | 91.00% | 10.00% | 4.127 | 971.8 | 59.7 |

### Key Findings

- **Evidence is the dominant source of improvement.** The Base model improves from 27.00% EX without evidence to 50.40% with evidence, a gain of **+23.40 percentage points**.
- **Vanilla SFT does not improve EX without evidence.** Base and SFT both achieve **27.00% EX**.
- **With evidence, SFT slightly underperforms the Base model.** EX decreases from 50.40% to 47.60%, a change of **-2.80 percentage points**.
- **SFT improves SQL validity but not execution correctness.** Valid SQL increases from 86.00% to 90.00% without evidence and from 89.40% to 91.00% with evidence, suggesting that SFT improves output well-formedness more reliably than task-level semantic correctness.
- **Normalized EM also increases slightly after SFT**, from 6.40% to 7.00% without evidence and from 9.40% to 10.00% with evidence, while EX remains unchanged or decreases. This further supports the distinction between textual alignment and execution-level correctness.
- **SFT inference is substantially slower.** Average generation latency rises from approximately 1.8 seconds for the Base model to approximately 4.1 seconds for the SFT model, while output lengths remain similar.

## Current Conclusion

The 8192-token evaluation strengthens the current working hypothesis: the main limitation is not SQL formatting or syntactic validity, but execution-level semantic reasoning. SFT makes the generated SQL slightly more valid and textually aligned, but it does not consistently improve the final execution result. The next experiment will therefore measure `EX@1`, `Oracle@3`, and `Oracle@5` to determine whether the current bottleneck is primarily candidate generation or candidate selection before implementing the full R³-SQL-lite ranking and resampling pipeline.
