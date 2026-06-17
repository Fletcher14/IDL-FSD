# Validation Methodology

> Evaluation rigor is treated as a first-class feature of this project.
> This document describes the practices that make the reported results trustworthy.

## The problem: data leakage inflates results

In imitation-learning and perception pipelines it is easy to over-report
performance by evaluating on data that overlaps with training — same maps, same
scenarios, or targets the model can trivially copy from its input. The result is
a number that looks good but does not reflect generalization.

## Practices used here

### 1. Leakage-free validation by construction
A dedicated world **seed is reserved** and never placed in the training set. Because
the world is fully procedural and deterministic by seed, this guarantees the
evaluation maps are genuinely novel — not a resampling of training maps. Every
recording session is **stamped with provenance** (seed + creation metadata) so the
training/holdout boundary is auditable.

### 2. Sensor-realistic targets that can't be copied
During training and evaluation, the model's **input** is a degraded sensor
observation (occlusion + noise), while the **targets** are clean ground truth. The
relevant channels are handled so the model cannot simply echo its input — it must
infer the world state.

### 3. Identical train/eval conditions
The evaluation harness **mirrors the training validation loop exactly**: same dataset
class, same loss functions and weightings, same observation regime. A checkpoint's
evaluation score therefore matches what training reported, confirming the harness
is faithful.

### 4. Head-to-head comparison on the same holdout
Model improvements are reported only when a new model and the prior model are scored
on the **same reserved novel map under identical conditions**. A model's score on its
own (easier) holdout is not used as the comparison baseline — doing so is one of the
most common ways AV/ML results are unintentionally overstated.

### 5. Closed-loop run tracking
Driving evaluations (not just perception metrics) are logged to a separate
experiment tracker: destinations reached, emergency-braking events, cost profile,
and throughput — keeping closed-loop driving quality distinct from offline
perception metrics.

## Why this matters

The discipline above is what separates "the model scored well on a number I chose"
from "the model generalizes to conditions it has never seen." The latter is the
only claim worth making for a safety-critical domain.
