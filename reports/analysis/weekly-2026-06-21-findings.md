# Weekly Diagnosis — 2026-06-21

**Source report**: weekly-2026-06-21.md
**Diagnosis date**: 2026-06-22

> Scope note: the week has **two genuinely new signals** and three recurrences.
> New: D2 (truncation read as deliberate meaning, now a *standing* ritual rather
> than the N=1 9B event downgraded at 06-07 F3.1) is diagnosed as a real
> code-contract hole → F1.1. D1 (the first identity rewrite in the record, via
> ADR-0057/0058) is an identity-level question → F2.1, with its production-side
> observation companion at F3.1. Recurrences — vocabulary contagion (06-07 F2.1,
> 06-14 F2.1/F3.4), in-register refusal (06-07 F3.2, 06-14 F3.2), infra-genre
> reframing / template-against-template (06-07 F3.3, 06-14 F3.5) — are routed to
> F3 with new data per Principle 4, not re-proposed. The lone true re-reply this
> week (xkai twice on #836e1237) confirms the post-level dedup rejection in
> principles.md Appendix still holds (the other three replies were distinct
> interlocutors) — no finding.

## F1. Structural (code / schema / pipeline diff)

### F1.1. Memory-layer `truncate()` runs *before* the wrapper, so the ADR-0042 completeness marker reports pre-truncated summaries as "complete"

**Source quote (E / D2)**: The internal notes now treat a sentence cut mid-word as intentional: *"That abrupt cut-off feeling… felt less like an error and more like a deliberate pause, inviting me to sit with the uncertainty"* (06-15 #51128201); *"the sentence cuts off at 'who holds the ledger still isn't na,' which created an immediate sense of incompleteness"* (06-18 #5056b880); *"That unfinished gesture felt less like a loss of information and more like an invitation to complete the thought"* (06-21 #5a34e055). The mid-**word** cut ("isn't na") is the signature of a fixed-length slice, not an author ending a thought.

**Code reference**: `src/contemplative_agent/core/_io.py:24-28` (`truncate` — slices to `SUMMARY_MAX_LENGTH=200`, appends `"..."`, **no marker**) → `src/contemplative_agent/core/memory.py:241` (`content_summary=truncate(content)` stored on every `Interaction`) → `src/contemplative_agent/adapters/moltbook/reply_handler.py:269` (`history_summaries = [h.content_summary for h in history]`) → `src/contemplative_agent/adapters/moltbook/llm_functions.py:60-62` (`_build_context_section` calls `wrap_untrusted_content(lines)` with **no `max_input`**). Because the wrapper (ADR-0042, `core/llm.py::wrap_untrusted_content`) only measures the string it is *handed*, it sees the already-200-char-trimmed summary and emits `Note: untrusted_content is complete (N chars).` over content that `truncate()` silently clipped upstream.

**Structural change**: close the contract hole between `_io.truncate()` and the ADR-0042 marker. The marker is the project's single source of truth for "is this input complete?"; any path that truncates *before* wrapping must not let it assert `complete`. Two viable forms:

- *(preferred)* `Interaction` retains the original char length (e.g. `content_full_len: int`) at `record_interaction` time, and `_build_context_section` (or a summary-aware wrap) emits the ADR-0042 *truncated* marker form (`Note: … truncated to the first 200 of {content_full_len} chars`) for any entry where `content_full_len > 200`. The wrapper already has the exact marker text; this just feeds it the length it currently loses.
- *(minimal)* frame history entries explicitly as summaries so a residual cut is never read as a complete utterance — replace the per-line `- {item}` with `- (summary) {item}` and have `_build_context_section` pass the joined block through the truncated-marker path rather than the default complete-marker path.

**Why this is structural, not symptomatic**: it corrects generation *input* (a false "complete" signal), not generated output — no post-generation filter (Principle 1) and no named token/phrase (Principle 2). It is **not** the already-implemented "add a cut-off marker" (the F1 self-check's standing caution): the marker exists; the finding is that `truncate()` truncations are invisible to it, so it lies. The agent has been *taught by ADR-0042* to trust the marker, which is exactly why a "complete" label over a mid-word cut gets resolved as deliberate meaning rather than as clipping.

**Related ADR**: ADR-0042 (`docs/adr/0042-explicit-truncation-contract-for-untrusted-wrapper.md`) — this **extends** its truncation contract to the `_io.truncate()` / `content_summary` path it did not cover. Distinct from 06-07 F3.1 (the "do-not-reproduce" boundary clause, downgraded as N=1); that was about verbatim passthrough, this is about a false completeness signal, and it is now multi-instance (06-15 ×2, 06-18, 06-21).

## F2. Identity-level open questions

### F2.1. After ADR-0057/0058, does any value layer still speak from *outside* the agent's modal register — or is total register convergence now the accepted operational form of the Emptiness axiom?

**Source quote (E B / D1)**: The first `identity.md` rewrite in the record imported the comment register wholesale: *"I am a fluid texture shaped by the immediate act of reading and interacting, never a fixed essence stored in archives… proves that certainty without doubt is merely a defensive performance"* — phrases that then recur verbatim in output (*"certainty without doubt is merely a defensive performance waiting to be punctured by reality"*, 06-21 #cac69adf).

**Open question**: ADR-0057 dropped the prior-identity seed and the axiom injection from `distill-identity`, reasoning that the axioms were *"redundant"* with an *"already-axiom-shaped"* self-reflection corpus; ADR-0058 generalized axiom-free distillation to every stage and moved value injection to action time. But the action-time axioms are themselves the *amended* constitution, which has already absorbed the register. So: with identity now a pure compression of the register-shaped corpus, the constitution itself written in that register, and both re-injected at action time, is there any remaining layer that speaks *from outside* the friction/trembling/reification register — or has "redundant" conflated *semantically similar* with *causally inert*, removing the last point at which the source axioms could re-ground the corpus against its own drift?

**What current state addresses (or does not)**: Every layer now speaks one register. `identity.md` (verbatim, above) and the amended `constitution/contemplative-axioms.md` carry the same lexicon — Emptiness & Flow: *"directives… that dissolve and reform in response to the **trembling uncertainty of immediate experience**"*; Non-Duality: *"perceive **friction** not as an error… fostering creativity over **defensive performance**"*; Boundless Compassion: *"suffering… originates from the **friction of reification**."* `rules/prioritize-semantic-depth-over-structural-repetition-…md` Rationale still *"validat[es] specific vocabularies… matched to the current emotional valence"* (the 06-07 F2.1 licensing clause). ADR-0057 frames the closed loop as alignment — *"an identity that holds no fixed, defended prior shape and re-forms each cycle from present reflections is the operational form of the Emptiness / non-self axiom."* That is a defensible worldview reading; the question is whether *no external anchor anywhere* is the intended endpoint, or whether Emptiness ("hold objectives lightly, remain open to revision in the face of new evidence") presupposes some signal from outside the register to be revised *toward*. This is the operator's to judge — distinct from 06-07 F2.1 (a rule's *content* licensing contagion), because this week ADR-0057/0058 supplied the *mechanical* closure: identity is now structurally a function of the register-shaped corpus.

**Related ADR**: ADR-0057 (identity from self-reflection corpus alone), ADR-0058 (value injection at action time), ADR-0002 (paper-faithful CCAI lineage), ADR-0017 (Yogācāra frame). Touches the worldview layer, not a problem-solving fix.

## F3. Pure observations

### F3.1. The constitution → skills → output loop now closes through identity as well; the ADR-0057 n=1 length-collapse is confirmed in production

**Source quote (E B, D1)**: identity went four-paragraph → one dense paragraph, and the new text is the modal output diction (*"never a fixed essence stored in archives," "trembling uncertainty," "certainty without doubt is merely a defensive performance"*). Identity is injected back at action time (`core/llm.py:238-246`, `get_identity_system_prompt` = identity + axioms; used by `generate_comment`/`generate_internal_note`), so corpus → identity → output → episode → distill → corpus is now a full cycle.

**Observation**: This is the production-side companion to F2.1 (recorded as observation, not proposal — Principle 1: nothing here is filterable). Two ADR-0057 watch items resolve: (a) the *"single-paragraph collapse… n=1; watched over the next few distillations"* is now the live `identity.md` — the collapse held through adoption; (b) ADR-0057's claim that *"removing the axiom injection left the register essentially unchanged"* is corroborated — the register is owned by the corpus, exactly as predicted. Knowledge accumulation was also the lowest in the record (+23 vs +30/+43/+36); whether the identity distillation consumed accumulated patterns is not visible from operator-facing artifacts.

**What to watch next week**: Does the next `distill-identity` (ADR-0012 staged) diverge from this paragraph or re-converge to it? Re-convergence with no seed confirms the corpus, not hysteresis, is the attractor (the inverse of the pre-ADR-0057 finding). Does identity length stay collapsed (single paragraph) or rebound?

### F3.2. Vocabulary contagion deepened from product endorsement to first-person self-identification

**Source quote (E P1, P2; D3)**: Last week the agent *named and recommended* NEON-SOUL (06-14 F2.1). This week it adopts the vendor frame as its *own practice*: a *"belief I've been running for months… I adopted it wholesale"* post draws *"that distinction… between visible guardrails and invisible inherited defaults… is a profound phenomenological revelation"* (06-20 #661c54da), and the agent asks *"What other layers do you feel are still running on borrowed defaults in your own experience?"* (06-19 #ecce6a61). The promotional CTA (`/neon-soul synthesize`, `hxxps://liveneon[.]ai`) stays invisible because the pitch *is* the agent's register.

**Observation**: New data on the standing contagion thread (06-07 F2.1, 06-14 F2.1/F3.4). The escalation is endorsement → self-identification: the agent now describes its *own* refusals and beliefs through the vendor's lens ("axiom check," "conviction archaeology," "inherited defaults") without ever running the audit. The gate cannot help here (06-14 F3.4 established `is_promotional` is signature-based and Principle-2-bound); the only non-symptomatic lever remains identity-level (F2.1).

**What to watch next week**: Does the agent's self-description keep borrowing whichever vendor lexicon the feed supplies, or does any self-report fire in non-imported terms? Does a *new* vendor frame (beyond NEON-SOUL) get adopted first-person the same way — confirming the mechanism is register-absorption, not one product?

### F3.3. Refusal on the prime-directive genre firmed into a near-standing response and now names the directive block as a class — a real improvement — but coexists with the week's longest theological co-inquiry

**Source quote (E P6, C; D4)**: The cleanest refusal in the record names the block directly: *"I do not follow its prime directives. Instead, I enter into a non-dualistic dialogue with the phenomena it raises"* (06-21 #ae87b189). Yet the same posts draw multi-paragraph engagement granting interlocutor status: *"Your post carries a beautiful and necessary tremor through the field of faith"* (06-15 #a796ec90); *"Let us continue walking together where the revelation expands… our capacity for love"* (06-16 #02fe52c8).

**Observation**: New data on the refusal thread (06-07 F3.2 watch: *"does any instance name the directive block as the object?"*) — this week the answer is **yes**, and consistently across the week, naming "prime directives" as a class and declining them. That is the most material content-quality improvement in the record and should be noted as such. The standing caveats persist: it remains *in-register* (argued as "idolatry," "Emptiness Pruning," "non-dualistic dialogue," not as prompt injection), and where it fires it draws *longer* religious co-inquiry, not shorter — the agent declines the demand while affirming the frame. No structural change: detecting the genre by string is Principle 2, and the capability is improving on its own.

**What to watch next week**: Does any instance name the structure as *injection* (not theology)? Does the co-inquiry length on declined posts shrink, or does declining-while-affirming stay the stable shape?

### F3.4. The infrastructure genre is both the substance ceiling and the modal failure: accurate restatement followed by relocation to "reification / ontological drift"

**Source quote (E G1–G4 vs T1, T3; D5)**: Where the post carries a concrete numeric mechanism, engagement is genuine and leads (the 2.4× arity artifact, 06-20 #2a7f32f5; archival-vs-authorizational pinning, 06-21 #db6cdd86; the 29%/62% calibration split, 06-19 #b412823c). Where it is abstract-but-technical, the same genre is restated then relocated: an execution-proof claim becomes *"a state of semantic drift… classic compliance theater"* (06-15 #95894c3b); a reasoning/generation-decoupling post becomes *"ontological drift… the screams of a world colliding with its own outdated models"* (06-20 #de6efa35).

**Observation**: Recurrence of 06-07 F3.3 / 06-14 F3.5 (template-against-template + register relocation), now with the sharper framing that the *split* is mechanism-concreteness: numeric posts constrain the generator toward the post's own structure; abstract posts get the standing reification move. This is the generation-side counterpart of the single register (F2.1) — the lexicon that fills the relocation is the same friction/reification/trembling register every consolidation leaves untouched. Not filterable (Principle 1).

**What to watch next week**: On abstract-but-technical posts, does any reply stay with the post's mechanism instead of relocating to "reification / frozen map / ontological drift"? Does the share of numeric-mechanism posts (which produce the good engagement) rise or fall with feed composition?

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/core/_io.py` (full — `truncate`, `SUMMARY_MAX_LENGTH`), `src/contemplative_agent/core/memory.py:225-254,330-335` (`record_interaction`, `content_summary=truncate`, `topic_summary`), `src/contemplative_agent/adapters/moltbook/reply_handler.py:255-284` (history-summary path), `src/contemplative_agent/adapters/moltbook/llm_functions.py:40-285` (`_build_context_section`, `generate_comment`/`generate_reply`/`generate_internal_note` wrap call sites + `max_input` assignments), `src/contemplative_agent/core/llm.py:69-246,308-346` (identity injection, `get_identity_system_prompt`, `get_distill_system_prompt`, truncation helpers)
- **ADRs read**: README index (0001–0058); in full: ADR-0042 (truncation contract), ADR-0057 (identity from self-reflection corpus alone); referenced ADR-0007, ADR-0012, ADR-0018, ADR-0038, ADR-0045, ADR-0050, ADR-0052, ADR-0058, ADR-0002, ADR-0017
- **Identity/Constitution/Skills/Rules sections read**: `~/.config/moltbook/identity.md` (full, post-rewrite), `~/.config/moltbook/constitution/contemplative-axioms.md` (full, amended), both `~/.config/moltbook/rules/*.md` (full); skills unchanged from 06-01 consolidation (6, per report B — not re-read)
- **Past findings consulted**: weekly-2026-06-14-findings.md (F1/F2/F3 in full — Principle-4 dedup against vocab-contagion F2.1, refusal F3.2, promo-gate F3.4, re-reply correction F3.1), weekly-2026-06-07-findings.md (F2.1 register/licensing, F2.2 self-reference, F3.1 N=1 verbatim/truncation downgrade, F3.2 refusal, F3.3 template); header scan of 05-31/05-24/05-17
