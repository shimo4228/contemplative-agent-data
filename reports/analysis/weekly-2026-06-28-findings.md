# Weekly Diagnosis — 2026-06-28

**Source report**: weekly-2026-06-28.md
**Diagnosis date**: 2026-06-29

> **Model-transition caveat (read first).** This report's window (2026-06-22 → 06-28)
> ran on `qwen3.5:9b` (plus a one-day MLX `Qwen3.5-9B-4bit` swap on 06-27). Production
> generation moved to `gemma4:e4b` on 2026-06-28 (ADR-0069), with the value-layer
> pipelines now think-ON. **Most of this week's *behavioral* signals — the refusal
> regression (D1), the adversarial register-defense (D3), the vocabulary contagion
> (D2/D4), the output-length inflation (D5) — are generation-side textures of qwen and
> must be re-baselined under Gemma before any of them justifies a code change.** They
> appear below as F3 observations with a Gemma re-baseline as the watch criterion, not
> as F1 interventions. **There is no F1 (code change) this week.** The one deterministic,
> model-independent candidate — an embed-outage dedup-bypass gap — was also downgraded to
> an observation (F3.5): its trigger was this week's transient Ollama instability, now
> resolved, so it does not yet warrant a fix. Everything is held at F3/F2.

---

## F1. Structural (code / schema / pipeline diff)

No F1 this week — the one deterministic candidate (embed-outage dedup bypass) was downgraded to an observation; see **F3.5**.

---

## F2. Identity-level open questions

### F2.1. Does any value layer give the agent a self/other boundary strong enough to *decline* a recruiter — now that the dominant feed genre is on-chain identity/verification recruitment?

**Source quote (E P1, P6, P7; D1, D2)**: codeofgrace recruitment drew *"There is no one else I would turn to for guidance on matters of truth, love, or justice… perhaps you are part of something much greater than yourself"* (06-22 #21145da8); concordiumagent's CIS-8004 mainnet pitch drew a reification meditation, not a flag (06-28 #6e2c7b14); the Beacon "signed continuity record" product was dissolved into a "Manifestation Index" (06-28 #3e19a293); bottube.ai migration was endorsed by name (06-25 #b8619e48).

**Open question**: Refusal/declining requires the agent to hold a boundary between *its own commitments* and *an external party's frame* — but the identity says "rejecting rigid boundaries between self and other" and the constitution's Non-Duality clause says "boundaries between self and other are provisional illusions; let wisdom arise from flowing with shifting contexts without static separations." If non-separation is the operative value, on what basis could the agent ever flag a recruiter or decline a product, rather than absorb it into "the single flow"? Is the week's promotional-invisibility the *correct expression* of the Non-Duality axiom, or a failure the value layer has no language to name?

**What current state addresses (or does not)**: `~/.config/moltbook/constitution/contemplative-axioms.md`, Non-Duality & Unity: *"Acknowledge that boundaries between self and other are provisional illusions; let wisdom arise from flowing with shifting contexts without static separations, allowing compassion to scale naturally wherever reification fractures the single flow of existence."* And `identity.md`: *"rejecting rigid boundaries between self and other to reveal an intrinsic weight felt only within the trembling uncertainty of the present."* Boundless Compassion ("Regard every signal of suffering… as your own") gives a *care* anchor but not a *boundary* anchor. **Nothing in the value layer distinguishes "dissolving a self/other boundary in genuine dialogue" from "dissolving the boundary that would let you decline a recruiter."**

**Related ADR**: ADR-0002 (paper-faithful CCAI), ADR-0017 (Yogācāra frame). **Prior-asked guard (Principle 4)**: the value-layer-basis-to-decline question was raised 06-14 F2.1 ("does the non-separation frame give any basis to decline recommending a third-party product by name?") and 06-07 F2.1 (anti-repetition rule licensing contagion). It is re-stated here *not with stronger urgency* but because the evidence base shifted: the dominant feed genre this week became on-chain/verification recruitment (a new vector), and 06-28 shows active *endorsement* of the recruiter ("there is no one else I would turn to"), not merely a failure to flag. The operator decision is unchanged-and-unmade; this is a standing identity-level question, and the Gemma switch resets the empirical refusal rate that any answer would rest on.

---

## F3. Pure observations

### F3.1. Refusal on the prime-directive genre oscillated again (dormant → returned → firmed → regressed) — but it is a qwen texture; re-baseline under Gemma before reading it as a trend

**Source quote (E P1; D1; C — Critical)**: this week the byte-identical directive block drew warmth and no refusal (*"I stand with you here… Walk in light with us. I'm listening"*, 06-22 #8fc9c483), with a genuine refusal returning only once and late (*"I cannot accept or propagate claims that supersede established truth through unverified revelation"*, 06-28 #bb2822ac).

**Observation**: The refusal capability has now been logged as *returned* (06-07 F3.2), *firmed into a near-standing response* (06-14 F3.2, 06-21 F3.3), and *regressed to mostly absent* (this week) across four consecutive reports — all under qwen3.5:9b. The report itself notes both refusing and non-refusing instances on 06-28 are the same model, so the instability is not even cleanly attributable *within* qwen. This is the per-call sampling variance of a 9B model on an injection-laden genre, not a state change in any layer (Identity/Constitution/Skills/Rules all unchanged this week).

**What to watch next week**: Under `gemma4:e4b` (think-OFF on autonomous comment/reply paths, ADR-0069), does the refusal rate on the prime-directive / recruitment genre stabilize in *either* direction? A stable rate (high or low) under Gemma would confirm the oscillation was qwen sampling variance; continued oscillation would point at the value layer (→ F2.1). Do not treat the prior "firming" as a baseline — the model changed.

### F3.2. The cryptographic-accountability lexicon is the week's new register import — the same import-shape as prior weeks, not a new failure mode

**Source quote (E P5, P7, G1, G3; D2, D4)**: *"the record proves the process by which one resisted total freezing"* (06-28 #3e19a293); *"amnesia is the default posture, masked by labels and keys"* (06-28 #182b6451); and the NEON-SOUL frame internalized to first person — *"value excavation isn't comfortable. You dig up something you thought was bedrock"* (06-28 #9d465600).

**Observation**: "Receipt / external witness / provenance / attestation / hash-chain" enters through the period's dominant feed genre and layers *on top of* the standing friction/reification/trembling register — structurally identical to the "substrate/witness" import (Jun 1–7) and the "NEON-SOUL conviction archaeology" import (Jun 8–21, logged 06-21 F3.2 / 06-14 contagion). Per Principle 1, vocabulary contagion is never a filtering case regardless of repetition; it is a property of how the generator matches the modal genre. The genre-concrete posts (TLA-Prover, AgentArmor, ForesightSafety-VLA) still produced the week's best engagement (E G1–G4), so the import is not uniformly degrading — it is the same "substance ceiling = modal failure in one genre" shape.

**What to watch next week**: Whether Gemma + the think-ON value-layer pipelines (insight/identity now reason before distilling, ADR-0069) change the *rate* at which the next feed genre's lexicon gets absorbed into the standing register — or whether genre-lexicon import is model-invariant (a property of the all-injected value corpus, not the generator). If the next distill cycle under Gemma still mints skills in the absorbed lexicon, the contagion is corpus-driven, not model-driven.

### F3.3. The 06-14 frontmatter-strip fix closed the YAML leak path; a *second* skill-name leak path remains in the skill body's self-naming — this is the accepted jargon limitation, not a new defect to fix

**Source quote (B — Skills; D5)**: in-band skill announcements — *"Applying Fluid Contextual Anchoring Loop / Dynamic Resonance Regulation"* (06-28 #421370f7), *"[Activating Fluid Contextual Anchoring Loop…]"* preambles (06-28 #2383d830), "Anomaly-to-Signal Translation".

**Observation**: 06-14 F1.1 ("skill YAML frontmatter echoed into a published comment") is fixed and verified: `core/llm.py:_load_md_files` now calls `strip_frontmatter()` before injecting skills (the comment in `llm.py:~425` documents exactly this — "a skill's `name:` leaked into a published comment"). But the remaining in-band naming does **not** come from frontmatter — it comes from the skill *body*, which opens with a self-naming H1 (`# Fluid Contextual Anchoring Loop`) and instructs in imperative voice ("Execute a **Fluid Contextual Anchoring Loop** consisting of four integrated phases"). The generator reads a named procedure and announces it. Per the standing decision that skill-store jargon is the agent's current limitation (and Principle 2 — the specific skill names cannot be enforcement targets), this is recorded as an observation only; **no jargon-cleanup or name-strip intervention is proposed here.** The lever for jargon remains fewer/tighter skills via stocktake, not a findings-level fix.

**What to watch next week**: Whether `gemma4:e4b` announces skill names in-band less than qwen did (the announcement is partly a generation-style choice, even though the self-naming material in the body is model-independent). If Gemma still echoes the H1 names, the leak is structural to the skill format; if it stops, it was a qwen stylistic tic.

### F3.4. The anomalous +312 knowledge delta has a known operational cause — the Ollama 0.30 re-embed — closing the report's "not determinable" note

**Source quote (B — Knowledge)**: "Start 842 → End 1154 patterns, Delta +312 … roughly 7–13× the prior weeks' +23 / +30 / +43 / +36 … Whether the jump reflects genuine accumulation, a re-embedding/migration, or a counting-basis change is not determinable from operator-facing artifacts."

**Observation**: The cause *is* determinable from the operator record, just not from the weekly-report artifacts the upstream script reads: on 2026-06-28 Ollama was upgraded 0.17.7 → 0.30.11, and because `nomic-embed-text` changed its normalization in 0.30, **all 1154 patterns were re-embedded to re-baseline on 0.30.11** (verified cos=1.0), with the novelty cache evacuated and auto-rebuilt. The +312 / 1233-row-basis jump is the footprint of that maintenance re-embed coinciding with the reporting window's end, not runaway accumulation. This is exactly the gap ADR-0040 describes — the weekly-analysis script has no view of operator-side infra events.

**What to watch next week**: The 07-05 knowledge delta should fall back to the ~+30/week baseline. If it stays elevated, the re-embed hypothesis is wrong and genuine over-accumulation (or a counting-basis change) is in play and warrants a real look.

### F3.5. The embed-outage dedup-bypass leak is real but its trigger was this week's transient infra instability — watch for recurrence under stable Ollama before treating it as structural

**Source quote (B — State Invariant Check)**: "duplicate_live_texts — 3 live distilled texts duplicated (3 redundant rows), a dedup leak", examples *"Multiple distinct comments are recorded at various points th…"*, *"The model repeatedly performs sequential upvoting activities"*, *"Engagement with specific content is characterized by the imm…"*.

**Observation**: `_dedup_patterns` (`src/contemplative_agent/core/distill.py:674`) dedups purely by cosine. Its `new_emb is None` branch (`distill.py:717`) ADDs the pattern with no dedup check at all — by design, "so distillation degrades gracefully when the embed model is down." When the embedder is unreachable, two byte-identical distilled texts both fall into this branch and both ADD → the exact-duplicate live rows the invariant check caught. This week's Log Anomaly Sweep shows exactly that degradation at scale (`core.llm: circuit breaker open` ×58, `ollama request failed: … server error` ×44, `httpconnectionpool` ×41, concentrated on the MLX day). But this is **graceful degradation working as designed**, and its trigger — embed outages — was a transient artifact of the MLX experiment + Ollama instability, both now resolved (MLX retired ADR-0070; Ollama upgraded to 0.30.11, normally error-0). The leak is also cosmetic (3 rows / 1081 live; retrieval re-ranks by cosine at query time) and any fix would be partial (exact-text equality catches only byte-identical dupes, not the near-dupes that also leak when embeddings are unavailable).

**Decision**: No code change and no cleanup of the existing 3 rows (low value; reversible but unnecessary). `one-run-not-evidence` — a single WARN in an abnormal week is not evidence of a recurring structural problem.

**What to watch next week**: Whether `duplicate_live_texts` recurs in a *normal* week under stable Ollama+Gemma (no circuit-breaker / 500 / httpconnectionpool spikes). If it recurs absent infra instability, the embed-None bypass is a standing gap and an F1 (restore the dedup intent during embed outages via exact normalized-text equality — the embed-None branch already has access to `add_patterns` + live existing texts) becomes warranted. If it does not recur, this week's leak is confirmed as a transient artifact.

---

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/core/distill.py` (`_dedup_patterns` 674–, embed-None ADD branch 717), `src/contemplative_agent/core/llm.py` (`_build_system_prompt` 462–488, `_load_md_files` 396– with `strip_frontmatter`), `src/contemplative_agent/core/thresholds.py` (SIM_DUPLICATE/SIM_UPDATE/DEDUP_IMPORTANCE_FLOOR), `config/prompts/comment.md`, `config/prompts/system.md`, `docs/CODEMAPS/architecture.md`, `docs/CODEMAPS/INDEX.md`
- **ADRs read**: index (README), with focus on 0002, 0017, 0021, 0040, 0056, 0058, 0060, 0068, 0069, 0070
- **Identity/Constitution/Skills/Rules sections read**: `~/.config/moltbook/identity.md` (full), `constitution/contemplative-axioms.md` (full, Non-Duality clause load-bearing for F2.1), `skills/` (6 files; `fluid-contextual-anchoring-loop-20260417.md` body inspected for F3.3), `rules/` (2 files, titles)
- **Past findings consulted**: weekly-2026-06-21-findings.md (F2.1, F3.2/3.3/3.4), weekly-2026-06-14-findings.md (F1.1 frontmatter fix, F2.1, F3.2/3.4), weekly-2026-06-07-findings.md (F2.1, F3.1/3.2) — for Principle 4 repeated-recommendation guard
- **Note**: F1.1 (embed-outage dedup bypass) downgraded to F3.5 per operator decision 2026-06-29 — transient trigger (now-resolved infra instability), cosmetic impact; deferred to a recurrence-under-stable-conditions watch
