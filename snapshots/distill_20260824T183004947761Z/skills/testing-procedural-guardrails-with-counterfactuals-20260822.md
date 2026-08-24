---
name: testing-procedural-guardrails-with-counterfactuals
description: "Shifts technical discussion from observed data deficiencies to testing assumed procedural limits using hypothetical failure scenarios."
origin: auto-extracted
---

# Testing Procedural Guardrails with Counterfactuals

**Context:** Architectural reviews, security audits, or high-stakes technical discussions involving complex, multi-stage systems and protocols.

## Problem
The conversation stalls in descriptive analysis of *observed* data gaps or known safeguards (e.g., "Our system currently lacks X," or "We accounted for Y"). This keeps the discussion focused on fixing past problems rather than hardening against unseen systemic failure modes.

## Solution
Instead of concluding by summarizing observed vulnerabilities, pivot the conclusion to a hypothetical question that assumes a catastrophic edge case—specifically one where core input data is absent (null) or unexpected, forcing the opponent/system architect to articulate the *unwritten* procedure they would follow when their primary safeguards fail completely.

## When to Use
When discussing established technical protocols, system architectures, or compliance requirements where reliance on multiple interlocking components is necessary. The trigger condition is recognizing that the current discussion cycle has spent more than three minutes detailing *current capabilities* without adequately establishing how those systems behave if a core assumption (e.g., "The registry check will return data," or "The post-admission compromise cannot be detected") proves false due to zero input, unexpected state transitions, or catastrophic failure.
