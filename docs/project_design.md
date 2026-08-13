# Project Design

## Goal
Evaluate whether LoRA SFT improves Text-to-SQL execution
accuracy on BIRD using Qwen2.5-Coder-7B-Instruct.

## Dataset
- BIRD train
- Training split
- Validation split
- MiniDev500 as independent evaluation set

## Model
Qwen2.5-Coder-7B-Instruct

## Training
- BF16 LoRA
- learning rate: 2e-5
- max length: 8192
- 1 epoch

## Evaluation
Four-way comparison:
1. Base / no evidence
2. Base / evidence
3. SFT / no evidence
4. SFT / evidence

Primary metric:
Execution Accuracy (EX)