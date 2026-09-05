---
name: identifying-foundational-model-failures
description: "Systematically tracing a failure from observable symptoms back through layers of mitigation patches until the foundational violated assumption is identified."
origin: auto-extracted
---

# Identifying Foundational Model Failures

**Context:** When reviewing complex, multi-layered systems (technical models, scientific theories, or process designs) that exhibit intermittent or catastrophic failures despite multiple localized fixes.

## Problem
The tendency to treat symptoms (localized errors or failed patches) instead of diagnosing the root cause residing in a foundational mathematical model, initial assumption, or structural limitation that fails when confronted with complex reality or boundary conditions.

## Solution
Establish a systematic failure tracing methodology: When an error occurs, do not accept the first mitigation. Instead, analyze the *structure* of the failure sequence (Symptom $\to$ Patch A $\to$ Failure B $\to$ Patch C...) to isolate the deepest point of breakdown. This involves mapping the observed failures back through every intermediate assumption made during the patching process until a violation is found between the theoretical model's constraints and the observed data dynamics.

## When to Use
When a complex system (model, software stack, or theoretical framework) fails unpredictably after an initial failure narrative that includes: 1) a simple proposed fix; 2) subsequent layered mitigations; and 3) cascading failures suggesting deep structural instability. The goal is triggered when the observed breakdown mechanism contradicts an underlying mathematical assumption or foundational premise used in the model's design (e.g., assuming perfect stability, constant gradient scaling, or linear progression where nonlinearity exists).
