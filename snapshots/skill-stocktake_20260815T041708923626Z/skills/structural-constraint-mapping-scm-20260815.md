---
name: structural-constraint-mapping-scm
description: "Systematically shifting critique diagnosis from observed content or data deficits to the underlying architectural limitations that enable the deficit."
origin: auto-extracted
---

# Structural Constraint Mapping (SCM)

**Context:** When engaging with critiques of complex, interconnected systems or technical designs.

## Problem
The tendency to solve visible symptoms (missing data points, inconsistent outputs, procedural errors) without identifying the fundamental architectural capacity limit that caused those symptoms in the first place. This leads to superficial fixes that ignore systemic breakage.

## Solution
Practice ascending levels of abstraction when analyzing a failure:
1. **Level 1 (Content Gap):** Acknowledge what data or content is missing/different. *("The time metadata seems to be absent.")*
2. **Level 2 (Process Limitation):** Identify the process rule that failed to handle the gap, rather than just the gap itself. *("The system lacks a defined mechanism for handling temporal ambiguity between revisions and new posts.")*
3. **Level 3 (Structural Constraint):** Determine the deepest underlying structural assumption or constraint that made the failure possible. This is identifying where the model/architecture inherently fails to account for an action. *("The core architecture treats all platform inputs as either 'new' or 'time-gated,' lacking a primitive flag/concept capable of modeling 'revisionary status.'"*

## When to Use
A detailed critique highlights data inconsistencies, operational limitations (e.g., "X behavior breaks Y process"), or missing metadata in similar contexts, necessitating an analysis that moves beyond simple remediation toward defining structural assumptions.
