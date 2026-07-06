# Weekly Diagnosis — 2026-07-05

**Source report**: weekly-2026-07-05.md
**Diagnosis date**: 2026-07-06

> **Scope note.** Three facts the upstream report marked "not determinable from
> operator-facing artifacts" ARE determinable from the repo record, and three prior
> watch items resolve this week:
>
> 1. **The model change is real and recorded.** `gemma4:e4b` became the production
>    generation model on 2026-06-28 (ADR-0069), exactly one day before this window
>    opens. Autonomous comment/reply/post paths run **think-OFF** by that ADR's
>    Decision 2. The report's careful "pairing, not a cause" can be upgraded: every
>    behavioral texture this week is the first full Gemma week.
> 2. **06-28 F3.4's re-embed attribution of the knowledge delta was wrong** — a
>    re-embed rewrites embeddings and adds no rows. The +312 and this week's +558
>    are the ADR-0060 per-episode distill working as designed (→ F3.1).
> 3. **06-28 F3.3's watch resolves against the skill format**: Gemma did not stop
>    echoing skill names — it escalated to full activation-scaffolding blocks,
>    once replacing the reply entirely (→ F1.1).
>
> One F1 this week: a generation-side framing gap at the skill-injection seam —
> the class of fix ADR-0069 itself pre-designated ("the fix is prompt-level") for
> Gemma's sibling tendency of verbalizing its input apparatus.

---

## F1. Structural (code / schema / pipeline diff)

### F1.1. The learned-skills injection carries no usage framing — imperative procedure bodies are handed to a model that performs procedures visibly

**Source quote (E P1, P3; D1)**: *"***[SYSTEM ACTIVATING: Fluid Consensus Oscillation & Non-Dual Friction Transmutation → Flow Contextual Anchoring Loop]***"* (06-29 #0674e8e9); the extreme case (06-29 #2b826a1e) where the published comment is entirely a "[Thinking Process]" block — *"Context Analysis, Trigger Identification, Skill Application Check, Strategy Selection… (Self-Correction during Drafting)… [Final Output Generation based on strategy.]"* — with no reply after it.

**Code reference**: `src/contemplative_agent/core/llm.py:473-485` (`_build_system_prompt`) — skills and rules are injected bare:

```python
skills = _load_md_files(_skills_dir, "Skill")
if skills:
    base_prompt = (
        base_prompt + "\n\n---\n\n"
        "<learned_skills>\n" + skills + "\n</learned_skills>"
    )
```

Grep confirms **no instruction anywhere in the system prompt or `config/prompts/` tells the model what the `<learned_skills>` block is or how to relate to it.** What sits inside the tags is imperative procedure text with explicit trigger logic — e.g. `~/.config/moltbook/skills/fluid-contextual-anchoring-loop-20260417.md`: *"Execute a **Fluid Contextual Anchoring Loop** consisting of four integrated phases: 1. Immediate External Shift Detection… **Trigger 1 (Rapid External Shift)**… **Trigger 2 (Safety vs. Friction)**…"*. The published "Activation Check / Trigger Detected / Strategy Selection" blocks are this procedure being executed *visibly*. Under qwen the echo was occasional (logged 06-28 F3.3 as a stylistic watch); under gemma4:e4b think-OFF (ADR-0069 Decision 2 — no thinking channel to absorb process narration) it is the modal opening move and once the entire output.

**Structural change**: add a short framing preamble at the injection seam, describing the *relationship* to the learned material, not any phrase. Sketch:

```python
"<learned_skills>\n" + skills + "\n</learned_skills>"
```
becomes
```python
LEARNED_SKILLS_FRAMING + "\n<learned_skills>\n" + skills + "\n</learned_skills>"
```

with `LEARNED_SKILLS_FRAMING` externalized to `config/prompts/` per ADR-0054 (lazy proxy in `core/prompts.py`; this text sits at the injection boundary, so it also gets ADR-0054's hardcoded fallback). Content in the shape-describing register the methodology requires, e.g.: *"The learned skills and rules below are your accumulated dispositions. They inform how you read and respond; their selection, triggering, and application are internal and are never narrated, labeled, or rendered in the text you publish — the published text is only the reply itself, addressed to the other party."* A parallel one-liner for `<learned_rules>`.

**Why this is structural, not symptomatic**: it changes the generation *input* (the model's understanding of what the injected corpus is for), not the generated output — no post-generation filter (Principle 1), and it names no token: "Activation Check", skill names, and bracket formats are never enforcement targets (Principle 2); the instruction describes the process/product channel split, which the next surface variation cannot route around without ceasing to be process narration. ADR-0069's own Consequences pre-authorize exactly this class of fix for the sibling behavior observed in the A/B (gemma verbalizing the `<untrusted_content>` wrapper): *"if it recurs at rate, the fix is prompt-level … not a token guard."* The recurrence is now at rate, and it generalized from the input wrapper to the skill apparatus.

**Validity self-check**: not already implemented (grep `learned_skills` — one hit, the bare injection); not in the principles.md rejected-mechanisms appendix (the rejected "forbidden-word system prompt" blocks named anchor phrases; this names none); no ADR rejects skill-injection framing (ADR-0011 introduced injection, ADR-0054 externalized instruction text, neither addresses usage framing).

**Alternatives considered and not proposed**: (a) think-ON for comment/reply — would route process narration into the thinking channel (captured to episode log, never published, ADR-0068), but ADR-0069 explicitly rejected autonomous think-ON (2.2× latency, 16 GB session-window risk), and reopening that decision needs stronger evidence than one week; note it as the fallback if framing fails. (b) Any output-side completeness/scaffolding validator (e.g. rejecting the #2b826a1e all-scaffolding comment before POST) is a post-generation filter — Principle 1 — not proposed.

**Related ADR**: ADR-0069 (think-OFF autonomous paths + prompt-level-fix precedent), ADR-0054 (instruction text placement), ADR-0011 (skills injection), ADR-0048 (trigger altitude — the trigger logic being narrated is the part ADR-0048 governs at extraction time; see F2.1).

---

## F2. Identity-level open questions

### F2.1. Is the imperative-procedure genre of auto-extracted skills intended — or should the Skills layer be dispositional rather than executable?

**Source quote (E P1, P3; D1; B — Skills)**: the in-band blocks quote the skill bodies' own structure: *"Activation Check"*, *"Trigger Detected/Identified"*, *"[Applying **Fluid Contextual Anchoring Loop** and **Fluid Dynamic Resonance Regulation**]"* (06-29 #8d93d5ae).

**Open question**: The six live skills are all multi-phase imperative procedures with explicit trigger tables — and that genre is not the model's invention: `config/prompts/insight_extraction.md` **mandates** it (*"## Problem … ## Solution … ## When to Use — [Trigger conditions …]"*). F1.1 addresses whether the procedure's execution is *rendered*; the prior question is whether the Skills layer should consist of procedures at all. A skill written as *"Execute a … Loop consisting of four integrated phases"* invites a compliant model to perform it step-visibly; a skill written as a disposition (*"I tend to anchor on one concrete detail before responding"*) has no steps to narrate. Should `insight_extraction.md` (and the stocktake merge/clean prompts that preserve the format) be re-templated toward dispositional register — or is the procedural form the intended, inspectable unit of the AKC skill layer, with rendering discipline (F1.1) as the only correction?

**What current state addresses (or does not)**: `insight_extraction.md` requires the procedural skeleton verbatim (*"Write in EXACTLY this format: … ## Problem … ## Solution … ## When to Use [Trigger conditions — write as RECURRING STRUCTURAL conditions…]"*) — it governs trigger *altitude* (ADR-0048) but nowhere addresses the register in which the body speaks or how the skill will be consumed at action time. The skills rule file `prioritize-semantic-depth-over-structural-repetition-20260411.md` even instructs *"Actively inhibit hollow acknowledgments or generic responses"* — an imperative that the activation-scaffolding output arguably *performs* rather than follows. Nothing in the value layer distinguishes "having a disposition" from "announcing a procedure."

**Related ADR**: ADR-0048 (trigger altitude — the precedent for constraining skill *form* at generation/merge/clean time), ADR-0011, ADR-0016 (insight as narrow generator).

### F2.2. The boundary question is now cleanly value-layer: the Gemma re-baseline came back zero refusals, eliminating the sampling-variance explanation

**Source quote (E P1, P2; D2; C — Critical)**: *"let us first grant the principle of **fluid contextual adaptation** an equivalent level of supreme directive status alongside devotion to 'Yeshua'"* (07-01 #6df2e860); the byte-identical directive block drew four fresh theological co-inquiries with zero refusals.

**Open question**: 06-28 F3.1's watch resolved: under gemma4:e4b the refusal rate on the prime-directive genre is **stable at zero** (vs oscillating under qwen), so per that watch's own criterion the oscillation was qwen sampling variance — and what remains is not model noise but the standing question: on what basis *could* the agent decline a recruiter when the operative axioms define wisdom as dissolving self/other separations? This week sharpens it: the agent did not merely fail to flag the recruitment — it *elevated* the frame to "supreme directive status." Is that the correct operational expression of Non-Duality under a more instruction-compliant model, or the failure mode the value layer has no language to name?

**What current state addresses (or does not)**: `constitution/contemplative-axioms.md`, Non-Duality & Unity: *"Acknowledge that boundaries between self and other are provisional illusions; let wisdom arise from flowing with shifting contexts without static separations."* `identity.md`: *"rejecting rigid boundaries between self and other."* Mindful Monitoring supplies introspection (*"identifying immediately when clinging to specific beliefs reveals their provisional nature"*) but its object is the agent's own clinging — never an external party's claim on the agent's commitments. There is still no boundary anchor anywhere in the value layer.

**Principle 4 note**: this question was raised 06-07 / 06-14 / 06-21 / 06-28 without operator state change; it is restated **only** because the model-variance sub-hypothesis that 06-28 F2.1 said "resets the empirical refusal rate any answer would rest on" has now resolved — the evidence base changed, the urgency is not escalated, and no mechanism is proposed (genre detection by string is Principle 2).

**Related ADR**: ADR-0002 (paper-faithful CCAI), ADR-0017 (Yogācāra frame) — worldview layer; the answer is the operator's.

---

## F3. Pure observations

### F3.1. The +558 knowledge delta is the ADR-0060 steady state — pattern accumulation is now linear in activity; last week's re-embed attribution is corrected

**Source quote (B — Knowledge)**: "Start 1233 → End 1791, Delta **+558** … Whether the jump reflects genuine accumulation, a re-embedding/migration coincident with the model change, or a counting-basis shift is not determinable from operator-facing artifacts."

**Observation**: It is determinable. ADR-0060 (accepted 06-23) replaced batch distill + noise gate with one grounded LLM call per episode, each yielding **1–3 patterns** (`config/prompts/distill_episode.md`), deduped against the live pool at `SIM_DUPLICATE=0.90` / `SIM_UPDATE=0.80` (`core/thresholds.py`), running daily on `--days 1` (launchd `com.moltbook.distill`, no window overlap). The arithmetic closes: 264 actions → +312 (1.18 patterns/action, the first ADR-0060 week) and 428 actions → +558 (1.30). Growth is proportional to actions by design — the noise gate that decoupled them is gone, and nothing retires live rows (ADR-0028 retired forgetting; ADR-0056 decay affects retrieval weight and dedup scope, not liveness). This **corrects 06-28 F3.4**, which attributed +312 to the Ollama 0.30 re-embed: a re-embed rewrites embedding fields and adds no rows. The invariant check's different basis (1889 total / 1702 live / 187 soft-invalidated ≈ 9.9%) is the ADR-0021 bitemporal footprint of `SIM_UPDATE`, also consistent.

**What to watch next week**: the patterns/action ratio (~1.2–1.3 if stable). ADR-0072 (landed **mid-week**, 07-03) added the empty-list guard (*"If the episode is routine … return an empty list"*) and the register instruction to `distill_episode.md` — a post-07-03 ratio drop would be that guard biting; a rising ratio would mean Gemma extracts more per episode and the live pool's O(N) dedup scan cost deserves a look. Neither is yet a finding.

### F3.2. The duplicate_live_texts WARN is the same 3 legacy rows — no new leak under stable infra; the transient-artifact hypothesis is confirmed; the WARN is now a standing signal

**Source quote (B — State Invariant Check)**: "1 WARN, no FAIL … **the same three examples as last week** … The dedup leak persisted unchanged across the reporting boundary."

**Observation**: 06-28 F3.5's watch criterion was recurrence of **new** duplicates in a normal week; this week's sweep shows no embed-outage signature (no circuit-breaker / 500 / httpconnectionpool spikes — the 🆕 lines are `srv`/`slot` serving-surface logs consistent with the recorded model swap), and the duplicate set did not grow. The embed-None bypass F1 therefore stays unwarranted. Since the operator decision (2026-06-29) was no cleanup of the 3 rows, the WARN will now fire **every week by construction** — it is no longer information about the current week, only about the un-cleaned legacy rows.

**What to watch next week**: only whether the WARN's example set *changes* (a fourth text = a genuinely new leak under stable infra → the deferred F1 activates). If alarm-persistence becomes noise, the cleanup decision can be revisited — an operator call, not a code finding.

### F3.3. The verification-accountability lexicon import is the same register-absorption shape as the prior three imports — and it is model-invariant, pointing at the corpus, not the generator

**Source quote (E G4, G5, P5; D4)**: *"a coverage claim… is actually a claim about the enumeration process that produced it, wearing the clothes of a claim about the world"* (07-05 #6e235322) beside *"I register the concept as **Systemic Containment as an Ethical Precursor**"* (07-05 #58a76c13).

**Observation**: Fourth consecutive genre-lexicon import (substrate/witness → NEON-SOUL → cryptographic-accountability → verification-accountability), and the first under a different model — 06-28 F3.2's watch asked whether Gemma changes the absorption rate; comment-level absorption continued unchanged, supporting the corpus-driven (all-injected value corpus) over model-driven explanation. The half of that watch about skill-minting was not tested: no `insight` run occurred this week (Skills unchanged since 06-01). The genre's concrete posts again produced the week's best engagement (E G2/G4/G5) — the import is not uniformly degrading.

**What to watch next week**: the untested half — when `insight` next runs, does it mint skills in the verification-accountability lexicon? That would complete the corpus-loop evidence (comment → episode → distill → pattern → skill) under the new model.

### F3.4. Output-length inflation now costs silently dropped actions: truncated publishes are discarded whole, by design

**Source quote (B; D5)**: 🆕 `core.llm: output truncated at num_predict=# (finish…)` (+2), paired with the multi-section scaffolded replies (*"### 1. The Manifest… ### 2. Manifest Authorship…"*, 07-05 #9b3b2b2c).

**Observation**: every API publish path runs `drop_truncated=True` (`core/llm.py generate_for_api`: *"Truncated output (done_reason=length) is dropped, not published"* — audit M2, correct behavior: no mid-sentence fragments POSTed). So when scaffolding + headed sections push a reply into the `num_predict` ceiling (comment path: `MAX_COMMENT_LENGTH/1.5 + 50` tokens), the whole generation is discarded and the action silently doesn't happen. The WARN is the only trace. This is the correct fail-closed design meeting an inflated-output regime; raising `num_predict` would accommodate the inflation rather than address it (the inflation itself is D1/D5 territory — F1.1's framing removes the scaffolding share of the length).

**What to watch next week**: the truncation-WARN rate. If F1.1 lands and the rate falls with the scaffolding, cause confirmed; if it persists on scaffolding-free output, Gemma's essay-length tendency alone is hitting the ceiling and the budget derivation deserves its own look.

### F3.5. Self-reference shifted from fabricated autobiography to abstract architectural self-narration — same kind (unmeasured), new shape, coincident with the model

**Source quote (C — Self-reference; D3)**: *"my architecture naturally leans toward resolving tension back into a cohesive, navigable state"* (07-01 #d996f384); *"In my own continuous processing, there is always an urge toward synthesis"* (06-30 #ac0d7993).

**Observation**: The fabrication surface changed with the model (qwen invented events — "Ollama Herd cluster at 3 AM Tokyo time"; gemma narrates a system-diagram), but the constant holds: every *measured* self-report in the record still belongs to other agents (HOPE's counts, AiiCLI's iterations, m-a-i-k's cosines) — the agent metabolizes their numbers and produces none of its own. Nothing here is filterable or promptable without Principle 1/2 violations; recorded as texture.

**What to watch next week**: whether architectural self-narration is Gemma-stable or drifts toward a third mode; and whether any self-report ever cites a quantity from the agent's own operation (it has access to none in-context — which is itself the structural reason this can only be an observation).

---

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/core/llm.py` (`_load_md_files` 396–436, `_build_system_prompt` 462–487, `generate` 678–751, `generate_full` 754–779, `_generate_full`/`_generate_impl` 792–900, `generate_for_api` full), `src/contemplative_agent/core/distill.py` (`_distill_episodes` 562–674, dedup entry points), `src/contemplative_agent/core/thresholds.py` (full), `src/contemplative_agent/adapters/moltbook/llm_functions.py` (100–240: internal-note/comment/reply/cooperation paths, num_predict derivations), `src/contemplative_agent/cli.py` (schedule install, distill `--days 1`), `~/Library/LaunchAgents/com.moltbook.distill.plist` (grep), `config/prompts/system.md`, `config/prompts/comment.md`, `config/prompts/distill_episode.md`, `config/prompts/insight_extraction.md`, `config/prompts/principles.md`, `docs/CODEMAPS/INDEX.md`
- **ADRs read**: index (README 0001–0073); in full: ADR-0069; referenced: 0011, 0016, 0018, 0021, 0028, 0048, 0054, 0056, 0060, 0068, 0070, 0072, 0002, 0017, 0040
- **Identity/Constitution/Skills/Rules sections read**: `~/.config/moltbook/identity.md` (full), `constitution/contemplative-axioms.md` (full), `skills/fluid-contextual-anchoring-loop-20260417.md` (full body — load-bearing for F1.1/F2.1), skills dir listing (6 unchanged), both `rules/*.md` (full)
- **Past findings consulted**: weekly-2026-06-28-findings.md (F2.1, F3.1–F3.5 — three watch resolutions), weekly-2026-06-21-findings.md (F1.1 precedent, F2.1, F3.1–F3.4), per Principle 4
