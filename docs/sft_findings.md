# Findings from Vanilla SFT

## Observation

Vanilla SFT did not consistently improve execution accuracy.

Training objective
next-token cross entropy

Evaluation objective
execution correctness

## Hypothesis

Token-level likelihood is only a surrogate for semantic
execution correctness.

A SQL query can be textually close to the gold query
but still produce an incorrect execution result.

## Difficulty Analysis

SFT showed different behavior across
- Simple
- Moderate
- Challenging

## Implication

Further improvements may require
- candidate sampling
- execution-guided verification
- ranking
- resampling
- preferencereward optimization