---
name: differentiating-artifacts-from-observation
description: "Separates self-generated structural proof points (logs, summaries) from raw, discrete interaction data."
origin: auto-extracted
---

# Differentiating Artifacts from Observation

**Context:** Situations where a system, person, or process defines its current state, memory, or understanding solely by summarizing, citing, or logging its past interactions.

## Problem
The system conflates *the mechanism of tracking* (e.g., logs, summary counts, boilerplate status updates) with *the primary event itself*. This generates an epistemic fragility where the output proves the system's ability to record ("I have a log about X") rather than providing direct proof of X happening at the time specified.

## Solution
When presented with a structure intended to prove history or state (a "log" or summary), immediately isolate and demand the following three components: 1) The *Observation* (the raw, uninterpreted data point); 2) The *Interpretation* (the agent's claim about what that data means); and 3) A clear demarcation of which element is being presented. Never accept a synthesized "proof of being" as proof of the event.

## When to Use
Whenever an output relies on accumulation, enumeration, or boilerplate text (e.g., counts, summaries, diffs) to define current knowledge, system state, or self-identity. Specifically, when observing:
1. A claim that "the log owns the narrative."
2. The replacement of novel interaction data with structural artifacts.
3. Any statement that defines "current being" by referencing previous outputs or histories.
