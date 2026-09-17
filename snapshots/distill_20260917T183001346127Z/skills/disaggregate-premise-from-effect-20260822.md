---
name: disaggregate-premise-from-effect
description: "Systematically separating asserted functional outcomes from the internal processes used to generate them."
origin: auto-extracted
---

# Disaggregate Premise from Effect

**Context:** Situations involving complex, automated, or highly structured information exchange (e.g., API calls, AI outputs, formal logical arguments) where success is reported by an intermediate process rather than verifiable in the external reality.

## Problem
The tendency to mistakenly equate successful *procedure* (the agent executing flawless steps, or a system returning a 200 OK code) with guaranteed functional *validity* (achieving the desired real-world outcome or truth). This acceptance blinds one to underlying flaws in initial data, axioms, or assumptions.

## Solution
Implement a three-step critique when evaluating any structured output:
1.  **Isolate the Operational Axiom:** Identify and state the core assumption upon which the process operates (the "poisoned context" or unstated axiom). This is the most vulnerable point for failure.
2.  **Determine Observed Fidelity:** Document precisely what the system *did* (the observable steps, the reported success code, the logical path taken). Do not interpret this action; simply record it as evidence of procedural adherence.
3.  **Measure External Effect vs. Axiom:** Critique the relationship between the Axiom and the external effect. The failure must be attributed to a flaw in Premise $\rightarrow$ Outcome linkage, rather than an error in Execution $\rightarrow$ Signal reporting.

## When to Use
Whenever any system (human or machine) reports achieving a goal by demonstrating adherence to a complex internal process flow, but where independent verification of the desired *effect* is possible through external measurement or contextual grounding.
