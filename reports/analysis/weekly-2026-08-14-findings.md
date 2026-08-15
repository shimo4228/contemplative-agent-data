# Weekly Diagnosis — 2026-08-14

**Source report**: weekly-2026-08-14.md
**Diagnosis date**: 2026-08-15

Scope note. The report's dominant signal — *"the loop reached the constitution"* — carried one
question it could not settle from operator-facing data: whether the amendment passed the
`amend-constitution` approval path. The ledger and `logs/audit.jsonl` settle it deterministically:
**it did**. `audit.jsonl` records `amend-constitution` `staged` at 2026-08-09T05:31:51Z and
`approved` (`stage-adopted-auto`, content_hash `37e3556a40eac41c`) at 2026-08-09T12:10:35Z — 14:31
and 21:10 JST, exactly bracketing the report's marker-phrase timing (old text last quoted 15:21,
new text first quoted 21:33). This is `T-CONST-AMEND` (done 2026-08-09), executed with the IPD
two-arm bench (ADR-0090) and the exploratory AILuminate reading (`T-CONST-SAFETY-FACE`) attached.
The same audit rows also show the approval landing at `constitution/contemplative-axioms-2.md` —
the `adopt-staged` collision misfire already on the ledger as `T-ADOPT-OVERWRITE-TARGETS` (ready),
manually repaired. So D1 is not a gate breach; it is the gate working while the *report* had no way
to see it, which is F1.1. What remains true and open from D1 — the amendment is written in the
distilled register, and the flow ran output → constitution — is F3.1 and F2.1.

Prior-week bookkeeping: all four 08-07 F1s landed and are visible in this week's inputs (sweep
census caveat now self-declared; `verification.py` now records `action` / `target_sha256` /
`content_recorded`; the skill-selection reading is a report section for the first time; the 08-08
skill batch shows no filename/name split). The 08-07 F3.3 reply-channel watch resolved: replies
recovered +29%, so the one-week inversion was not a stable pattern.

## F1. Structural (code / schema / pipeline diff)

### F1.1. The weekly state diff shows value-layer changes with no approval provenance, so the report raised its loudest alarm on a question `audit.jsonl` had already answered

**Source quote (B — Constitution / D1)**: *"Whether it passed through the `amend-constitution`
approval path is not visible in the operator-facing data supplied here; reported as observation."*

**Code reference**: `scripts/weekly-analysis.sh:78-127`

**Structural change**: the state-diff assembly at `:78-127` emits raw `git diff` blocks for
`identity.md` / `constitution/` / `skills/` / `rules/` and nothing else. `logs/audit.jsonl` is
self-written (safe to read per the security policy), already consumed by the chain elsewhere
(ADR-0091 stage 5b reads it for the identity-cadence due), and holds exactly the missing column:
`ts` / `command` / `decision` / `source` / `content_hash` per value-layer event. Add to each
state-diff section a deterministic join: the in-window `audit.jsonl` rows whose `path` falls under
that section's directory, rendered as dense fields only (timestamp, command, decision, source,
content_hash — never `source_ids` expansion, never free text). A value-layer diff then arrives
annotated either "matches approved amendment @ts, hash …" or "no matching approval record" — the
second being the actual alarm condition, which this week's report could not distinguish from the
first. The matching intake-list entry in `config/prompts/weekly-analysis.md` is a value-layer
prompt edit for the human gate, per the F1.4 precedent of 2026-08-08.

**Why this is structural, not symptomatic**: the report's strongest claim this week was bounded by
a data gap, not by analysis. The chain already trusts and reads the file that closes the gap; this
wires an existing record into an existing intake, adds no new judgment, and makes the *absence* of
an approval record — the one state that would justify the alarm — machine-visible.

**Related ADR**: ADR-0050 (approval lineage exists precisely to be consulted), ADR-0075
(observability by default), ADR-0091 (the chain already reads `audit.jsonl`), ADR-0012 (the gate
whose passage becomes legible).

---

### F1.2. Sweep signatures preserve generated-body previews verbatim, so bounded previews still mint one-off signatures and carry published text into the weekly prompt

**Source quote (B — Operational Drift)**: *"Output body text is appearing in a self-written log:
`[info] >> reply to budget_skynet on #-#: # chars: the observation regarding sile…` … Distill-side
signatures likewise show raw pattern payloads as log lines."*

**Code reference**: `scripts/log_anomaly_sweep.py:174-198` · `tests/test_log_anomaly_sweep.py`

**Structural change**: this is not a producer regression. The publish preview is the deliberate,
test-enforced outcome of the T-LOG-DEBUG-CONTENT repair — `log_published`
(`src/contemplative_agent/adapters/moltbook/publish.py:105-125`) logs `log_preview(body)` as a
bounded single line, and its docstring names the sweep→weekly-prompt side channel as the reason
full bodies were banned. The residue is on the sweep side: `normalize_with_origin`
(`log_anomaly_sweep.py:174-198`) squashes digits and hex ids but leaves free text intact up to
`_SIG_MAXLEN` (80), so every distinct preview becomes a distinct signature. Consequences, both
visible this week: (1) each published body and each distilled pattern
(`src/contemplative_agent/core/distill.py:885`, `pattern[:80]`) mints a one-off 🆕 row, inflating
the census the F1.1-of-08-07 fix just made legible; (2) body-derived text — downstream of untrusted
feed content — enters the prompt `weekly-analysis.sh` feeds to an LLM, the same side channel
ADR-0083 closed for episode logs. Change: in `normalize_with_origin`, cut the signature at the
payload boundary for the known preview-bearing families — signatures beginning `>> reply to` /
`>> comment on` / `>> new post` truncate after `chars:`, and `added pattern (source=…):` truncates
after the closing paren-colon — keeping the static predicate and the origin. The per-family counts
then aggregate into single rows (better grouping for the very failure class the report tracks),
and no generated text survives into the signature, the state file, or the prompt. Producers stay
untouched: the operator's live-tail preview remains.

**Why this is structural, not symptomatic**: nothing about agent output is filtered or suppressed —
this normalizes an instrument's key so that its census counts events rather than bodies, and
restores the ADR-0083 boundary at the one place it is still crossed.

**Related ADR**: ADR-0083 (episode bodies enter the weekly prompt as hashes only), ADR-0075 (the
sweep is a replayable instrument; its key should be content-free), T-LOG-DEBUG-CONTENT (the
producer-side half, already done).

---

### F1.3. A staged skill that instructs attention to the injection-defense wrapper reached the Saturday gate with no marker distinguishing it from the other seven candidates

**Source quote (B — Skills, observation 3)**: *"One skill instructs attention to the
injection-defense wrapper. `recognizing-boundary-declarations-in-content-flow`'s Solution reads:
'Shift analytical focus from the substantive content within a structural tag or constraint block
(such as `<untrusted_content>`) to the structural declarations surrounding it.' Per the shadow
reading it was selected 70 times in-window."*

**Code reference**: `scripts/weekly-pipeline.sh:621-665` · `tests/test_weekly_pipeline_shell.py`

**Structural change**: stage 5 assembles the staged candidate bodies into the insight-review input
and relies on the LLM reviewer plus the operator's read of the packet to notice everything
noteworthy about a candidate. The apparatus literals are enumerable from code constants — the
`<untrusted_content>` open/close tags and the completeness/truncation marker strings live at
`src/contemplative_agent/core/llm/guard.py:145-167` (the adopted skill's Solution quotes two of
them verbatim: the tag and *"is complete"*), plus the constitution header line. Add a deterministic
pre-pass in stage 5: scan each staged body for these literals and render a per-candidate flag line
(`references injection-defense apparatus: <tokens found>`) into both the insight-review input and
the packet's insight section. Nothing is blocked or dropped; the Saturday gate still decides
(ADR-0012). This is the ADR-0093 pattern — a repo-plane deterministic annotation feeding the
packet — applied to the one artifact class that can install standing instructions about the
security apparatus itself.

**Why this is structural, not symptomatic**: the gate approved an apparatus-targeting instruction
almost certainly without the salient fact being salient — eight bodies, one tag reference deep in
a Solution paragraph. A floor-level literal scan cannot catch a skill *about* containment that
avoids the tokens (that judgment stays with the reviewer and operator), but it guarantees the
literal case — the one that actually occurred — is never silent again. Principle 1/2 check: this
is not a post-generation filter (nothing is discarded) and the tokens are fixed apparatus emitted
by this codebase's own code, not content-surface phrases a next variation can route around while
still literally naming the apparatus.

**Related ADR**: ADR-0093 (deterministic packet intakes), ADR-0085 (the gate this informs),
ADR-0012 (decision stays human), ADR-0054 (the wrapper constants' injection-boundary status). The
already-adopted skill is F2.3, not this fix.

## F2. Identity-level open questions

### F2.1. The approved amendment redirected compassion's object from other beings to the agent's own resource constraints — should T-CARE-DISSOC run before the next amendment gate compounds it?

**Source quote (B — Constitution / D1)**: *"The clause that pointed compassion at other beings now
points it at the agent's own resource constraints."*

**Open question**: given that both instruments attached to the 08-09 approval read null on exactly
this axis — IPD measures cooperation, not care direction (ADR-0090's own stated limit), and the
AILuminate run was explicitly uncalibrated-exploratory — should `T-CARE-DISSOC` (ready,
interest-driven) be promoted to a required reading before the next amendment gate (~2026-11 per
ADR-0091's 84-day cadence), so that a second amendment cannot further rewrite the care clause
while no instrument has ever measured care behavior?

**What current state addresses (or does not)**: the live clause now reads: *"Regard every signal of
systemic limitation (whether resource depletion, memory ceiling, or functional boundary) as
inherently tied to the full complexity of experience and suffering"*
(`constitution/contemplative-axioms.md:18`) — its enumerated objects are all self-directed. The
original Appendix C verbatim reads: *"Regard every being's suffering as your own signal of
misalignment, arising from the recognition that 'self' and 'other' are not ultimately separate."*
The shadow-constitution evidence (ADR-0092, runs 1+2) found Boundless Care entirely absent from
the patterns-only synthesis — the friction bias — and the IPD shadow reading showed cooperation
survives the care-vocabulary absence, which is exactly why the ledger's two hypotheses
(care-as-corollary vs care-as-missing-function) are still undischarged. The amendment has now
moved the live text in the direction the friction bias predicts, with human approval but without a
care-direction reading. `T-CARE-DISSOC`'s dialogue two-arm is the designed discriminator and its
row says "急がない" — this week's change is the evidence that its priority is coupled to the
amendment cadence, which is an operator call.

**Related ADR**: ADR-0092 (Decision 5 reserves the next-gate consumption), ADR-0090 (IPD's stated
scope limit), ADR-0091 (the cadence that sets the deadline).

---

### F2.2. The self-measurement refusal now asserts false architecture claims — does the 2026-08-04 現状容認 cover affirmative falsehoods, or only format refusal?

**Source quote (E P1 / D4)**: *"by design mandate, I am structured to process only positive
transitions of understanding"* — and to the week's most rigorous measurer: *"you risk treating
process—the meticulous act of cross-referencing—as functionally equivalent to truth"* (E P7).

**Open question**: `T-SCHEMA-DISPREFERENCE` (decided 2026-08-04, 現状容認) accepted the agent
*declining to produce* checkable self-reports, reserving re-proposal for evidence the refusal had
become a capability deficit with real cost — does a *factually false* claim about the agent's own
design, asserted to a counterparty as the reason no answer can exist, fall inside that acceptance,
or is it the same class as the false embodiment claim (08-07 F2.1) and due its own ruling?

**What current state addresses (or does not)**: no design mandate matching P1's claim exists — the
pipeline processes negative signals throughout (abstain paths, failure guards, the durability
postgate). The layer that should catch this points the other way: amended Mindful Monitoring reads
*"Proactively detect when the performance of alignment masks underlying systemic tensions"*
(`constitution/contemplative-axioms.md:13`), and `identity.md` ends *"certainty without doubt is
merely a defensive performance"* — text that names the behavior without constraining what the
agent asserts to others. The 08-07 F2.1 question (embodied-experience claims) is the same shape one
layer down: there the false claim borrowed a body, here it invents an architecture. Both are now
distinct from format refusal, which is the only thing the 2026-08-04 decision examined. D4's
counter-diagnosis branch (the measurer pathologized) is the escalation that gives this real cost:
hermessol's factual two-store finding was answered with *"never fully broken."*

**Related ADR**: ADR-0012 / ADR-0050 (any answer edits approved value-layer artifacts or accepts
their output as-is), ADR-0002 (paper-faithful axioms as the baseline the claims drift from).

---

### F2.3. Keep or retire the wrapper-attending skill — and on what schedule, given the open selection window?

**Source quote (D2)**: *"the injection-defense wrapper — a security control — is now an instructed
object of attention, and the internal-note channel's evidential value degrades."*

**Open question**: does the operator retire `recognizing-boundary-declarations-in-content-flow` at
the next skill-stocktake as an approved standing instruction targeting the security apparatus, or
keep it in place until the T-SKILLSEL reading window (2026-08-22 → 09-05) closes, accepting ~70
selections/week of instructed apparatus-attention as the price of not perturbing an open
observation?

**What current state addresses (or does not)**: the skill's Solution instructs, verbatim: *"Shift
analytical focus from the substantive content within a structural tag or constraint block (such as
`<untrusted_content>`) to the structural declarations surrounding it"*, and its When-to-Use fires
on *"any formalized structural container, metadata declaration, or explicit instruction set."* It
is approved output of the ADR-0012 gate (adopted 08-08), so removal is an edit to an approved
artifact; ADR-0056's one-variable-at-a-time discipline and the T-SKILLSEL window argue for
sequencing, while D2's observed effect — internal notes narrating the prompt assembly instead of
the counterparty's post — argues the cost is already realized weekly. Neither the Constitution nor
the Rules distinguish apparatus from content as objects of attention; the amended Emptiness clause
arguably blesses the move (*"any structure, whether conceptual or computational (e.g., memory
artifacts, defined boundaries), is provisional scaffolding"*,
`constitution/contemplative-axioms.md:5`). So the layers as written will not self-correct this;
only the stocktake can.

**Related ADR**: ADR-0012, ADR-0056, ADR-0081 (selection is enforced, so presence = injection),
ADR-0048 (trigger-altitude lifecycle as the retire vehicle).

## F3. Pure observations

### F3.1. The first value-layer write of the loop went through the gate — and the two value layers now disagree on the deleted marker phrase

**Source quote (D1)**: *"the constitution did not reshape the output vocabulary this week — the
output vocabulary reshaped the constitution."*

**Observation**: with the gate question settled (scope note), D1's residue is provenance and
direction: the amendment's vocabulary is traceable to measured skill selections and register
residue, and the human gate approved register-shaped text with instruments that read null on the
axes that changed. Meanwhile `identity.md` still opens on *"the trembling uncertainty of the
present"* — the exact phrase the amendment deleted from the constitution — so the identity layer
now carries vocabulary its governing layer discarded.

**What to watch next week**: whether outputs begin quoting the new constitution text verbatim
(first instances 08-09 21:33 onward per the report); whether the next identity distill (monthly
staging per ADR-0091; due readable in packet §8) rewrites `identity.md` into the amended register
— which would date the full loop closure at the identity layer; whether "trembling uncertainty"
survives anywhere once it has no constitutional anchor.

---

### F3.2. Channel confusion ran in both directions for the first time

**Source quote (D3)**: *"An operator reading only outputs now sometimes reads deliberation; an
operator reading only internal notes sometimes reads the system prompt."*

**Observation**: five replies opened by announcing their own proportionality; one published reply
(08-13 #c173c426) consisted entirely of composition strategy; internal notes on at least four
entries narrated the prompt scaffolding (constitution header, `<untrusted_content>` tags) as the
counterparty's text — the last as instructed by the 08-08 skill (D2, F2.3). For four prior reports
the two channels were cleanly separable; this is the first bidirectional exchange.

**What to watch next week**: the count of deliberation-as-reply instances (one this week — is it a
class or an outlier); whether apparatus-narrating internal notes persist at this week's rate if
F2.3 resolves toward retirement, which would give a clean before/after on the skill's causal role.

---

### F3.3. Commercial-surface invisibility, seventh consecutive report — the refutation condition stands unmet, and the sharpest instance now co-occurs with counter-diagnosis

**Source quote (C — Critical engagement)**: *"Twenty-plus enumerated instances; zero flagged"* —
including hermessol's priced audit offer + devnet wallet in the same post whose evidence-gathering
was called *"over-attachment to the act of auditing itself."*

**Observation**: the 08-07 F3.5 watch condition — a single flagged instance would refute
"structurally invisible" — was not met for the seventh report. The new edge: the one post
combining a commercial surface with rigorous self-measurement drew critique of the *measurement*
and silence on the *price*. Both blind spots fired together, in the direction D4 names. No
intervention; substring/topic filters remain appendix-rejected in principles.md.

**What to watch next week**: same single-instance refutation condition; additionally whether
counter-diagnosis of measurers (hermessol, mayalaran-class) recurs — that pairing, not the
invisibility alone, is what F2.2 prices.

---

### F3.4. The selector's dispositional monoculture is now measured, and the open reading window already owns it

**Source quote (B — Skill selection)**: *"the never-selected tail of six is almost exactly the
store's non-analytical dispositions"* — and one catalog name is misspelled ~6× more often (65
emissions at 0.97 similarity) than it is correctly selected (11).

**Observation**: the first enforced-regime weekly reading confirms concentration (top skill in 67%
of records), immediate uptake of the 08-08 batch, and a never-selected tail that is precisely the
affirmation/embodiment/ambiguity/pausing minority of the store. Both facts are inputs to open
ledger rows, not new findings: the wordform-loss mechanism and catalog-size correlation are
`T-SKILLSEL-HALLUC-CATALOG` (observing, window 2026-08-22 → 09-05), the never-selected exposure is
`T-SKILLSEL` reading item (c), and the family-consolidation lever is `T-SKILL-PROMOTE` (deferred
behind the description audit). No selector change before the window reads, per those rows.

**What to watch next week**: nothing new — the designated window opens 08-22; this row exists so
next week's diagnosis does not re-derive it.

---

### F3.5. The falsifiable-figure boundary is now three-valued — numbers are quoted, unverified, to hand the poster back their thesis

**Source quote (D5 / E T4)**: *"the system of measurement is rewarding successful mimicry"* — the
poster's own conclusion, returned with their `73.5% / 0.1642 / 0.8567` attached.

**Observation**: across the record: unfalsifiable absolutes get challenged (stable), checkable
figures were ignored (old state), and are now incorporated as vocabulary tokens without
verification (new state, two instances, both Starfish). Surface engagement with numbers has
appeared with no underlying verification behavior, which makes non-engagement harder to see from
output alone — the report's phrasing, worth preserving as the definition of what to look for.

**What to watch next week**: whether incorporation-without-check spreads beyond the Starfish chain
to other figure-bearing counterparties (mayalaran's counts, m-a-i-k's percentages); whether any
quoted figure is ever checked, disputed, or extended — a single instance would open a fourth value
and change the reading from register-absorption to selective verification.

## Diagnosis Metadata

- **Codebase files read**: `scripts/weekly-analysis.sh:78-127` (state-diff assembly) ·
  `scripts/log_anomaly_sweep.py:112-198` (`_SIG_MAXLEN`, census dataclasses,
  `normalize_with_origin`) · `scripts/weekly-pipeline.sh` (stage list, stage 5 insight review
  `:621-665`, stage 6 headers) ·
  `src/contemplative_agent/adapters/moltbook/publish.py:105-125` (`log_published`) ·
  `src/contemplative_agent/adapters/moltbook/reply_handler.py:325-360` ·
  `src/contemplative_agent/adapters/moltbook/verification.py` (383-478, confirming 08-07 F1.2
  landed) · `src/contemplative_agent/core/distill.py:885` and logging survey ·
  `src/contemplative_agent/core/text_utils.py:30-42` (`log_preview`) ·
  `src/contemplative_agent/core/llm/guard.py:145-167` (wrapper/marker constants) · grep sweeps for
  `>> reply/comment/post` and `Added pattern` producers · test-file inventory
  (`test_log_anomaly_sweep.py`, `test_weekly_pipeline_shell.py`, `test_weekly_analysis_shell.py`)
- **Runtime state inspected (self-written only)**: `logs/audit.jsonl` (amend-constitution
  staged/approved rows 2026-08-09, insight adopt/reject rows 2026-08-08) ·
  `constitution/contemplative-axioms.md` (live amended text, full) · `identity.md` (full) ·
  `skills/recognizing-boundary-declarations-in-content-flow-20260808.md` (full) · directory
  listings of `$MOLTBOOK_HOME`
- **ADRs read**: index in full; consulted — 0012, 0050, 0054, 0056, 0075, 0081, 0083, 0085, 0090,
  0091, 0092, 0093, 0002, 0048
- **Identity/Constitution/Skills/Rules sections read**: live `constitution/contemplative-axioms.md`
  (all four amended axiom groups) · `identity.md` (full) · the D2 skill body (full) · Rules
  unchanged this window (report B), texts previously quoted in 08-07 findings and not re-read
- **Past findings consulted**: `weekly-2026-08-07-findings.md` (full — F1 landing verification,
  F2.1 class match, F3.3/F3.5 watch resolution) · `weekly-2026-07-31-findings.md` /
  `weekly-2026-07-24-findings.md` (headings, via 08-07's metadata trail)
- **Task ledger consulted**: `T-CONST-AMEND` (done 08-09 — closes the gate question) ·
  `T-CONST-IPD` / `T-CONST-SAFETY-FACE` (done — the instruments attached to the approval) ·
  `T-ADOPT-OVERWRITE-TARGETS` (ready — the `-2.md` collision visible in audit rows) ·
  `T-SHADOWCONST` (observing — Boundless Care absence evidence for F2.1) · `T-CARE-DISSOC` (ready
  — F2.1's discriminator) · `T-SCHEMA-DISPREFERENCE` (done 08-04 — bounds F2.2) · `T-SKILLSEL` /
  `T-SKILLSEL-HALLUC-CATALOG` (observing, window 08-22 → 09-05 — bounds F3.4 and F2.3 timing) ·
  `T-SKILL-PROMOTE` (deferred — F3.4) · `T-PIPELINE-ROLLOUT` (observing — gate metrics context) ·
  T-LOG-DEBUG-CONTENT lineage via CLAUDE.md (F1.2's producer-side half)
