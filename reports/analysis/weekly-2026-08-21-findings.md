# Weekly Diagnosis — 2026-08-21

**Source report**: weekly-2026-08-21.md
**Diagnosis date**: 2026-08-22

**Scope note — the source report reached this diagnosis with its head missing.**
`weekly-2026-08-21.md` begins mid-word (`ior statement implied a sufficient separation…`).
It carries no `# Weekly Analysis Report` title, no `## A. Quantitative Summary`, no
`## B. Agent State Snapshot`, and its first heading is `### Question specificity` at line 15 —
a subsection of C. Sections A and B and most of C are absent from the file, while D, E and
Downstream reference them throughout (`(B)`, `(A; D1)`). The pipeline reported success
(`report.log:15`, `Size: 37409 bytes`), the `.ja.md` was translated from the truncated text
(`weekly-2026-08-21.ja.md:3` opens on the same fragment), and the run continued. That is
**F1.3**, and it bounds this diagnosis: every quantitative figure below is quoted from the
surviving **Downstream** bullets, not from A or B, and the state-diff blocks that would have
carried the approval-provenance tables were unavailable to me. I re-derived the identity
question directly from `logs/audit.jsonl` instead, which is how **F1.1** was found.

**The report's loudest claim is false, and the instrument built last week to prevent exactly
this is what produced it.** D1 and the first Downstream bullet lead on *"`decision: staged` and
no approval row in the window"*. `logs/audit.jsonl` holds both rows:

```
"ts": "2026-08-15T04:03:59+00:00", "command": "distill-identity", "path": ".../identity.md",   "decision": "staged",   "source": "stage"
"ts": "2026-08-15T04:09:34+00:00", "command": "distill-identity", "path": ".../identity-2.md", "decision": "approved", "source": "stage-adopted-names"
```

The approval landed 5m35s after the staging, at the Saturday gate. It is invisible to the
join because the approved row's path is `identity-2.md` — the H5 collision misfire already
recorded as `T-ADOPT-OVERWRITE-TARGETS` (done 2026-08-15, `803d9d7`) — and the join matches the
identity section on the exact leaf name. That is **F1.1**. What survives of D1, and is genuinely
open, is that the live `identity.md` was put in place by a human `mv` repairing that misfire,
so the bytes now in generation have no audit row of their own: **F1.2** and **F2.1**.

Prior-week bookkeeping: 08-14 F1.1 landed (`weekly-analysis.sh:237-256` +
`scripts/value_layer_approval_join.py`) and is the subject of F1.1/F1.2 below. 08-14 F3.1's watch
resolved — the next identity distill did rewrite `identity.md` into the distilled register
(F2.1). 08-14 F3.3's refutation condition was **met** for the first time in eight reports
(F3.2). 08-14 F3.4 stands unchanged; its designated reading window opens today (F3.4).

## F1. Structural (code / schema / pipeline diff)

### F1.1. The approval join matches the identity section on an exact leaf name, so the one defect class that renames the target makes its own approval row invisible — and the instrument's loudest output is what fills the gap

**Source quote (D1)**: *"The `distill-identity` row sits at 04:03:59 with `decision: staged` and
no approval row in the window (B)."* — and Downstream: *"First identity change in the record,
with no approved record in the window."*

**Code reference**: `scripts/value_layer_approval_join.py:95` · `scripts/value_layer_approval_join.py:108` · `scripts/value_layer_approval_join.py:245` · `tests/test_value_layer_approval_join.py`

**Structural change**: `_matches_section` (`:95-112`) resolves the identity section with
`parts[-1] == "identity.md"` (`:108-109`); every other section is a directory-component match.
An audit row whose path is `identity-2.md` therefore matches **no section at all** and is
dropped silently from the tally. With the staged row matching and the approved row dropped, the
reading became `approved 0, staged 1, changed=True`, which is the exact predicate at `:245-251`
for emitting `⚠️ NO APPROVED RECORD`. The report then led with it, twice.

Two changes, both deterministic and read-only:

1. **Do not lose rows.** Select the identity section by the *command's canonical target* as well
   as by leaf name: a `distill-identity` row belongs to the identity section by construction,
   whatever leaf the write landed on. The same predicate already exists on the write side —
   `cli/adopt.py::_replaces_canonical_target` maps `distill-identity` → `identity.md` and
   `amend-constitution` → `constitution/*.md`. Reading and writing should not disagree about
   which command owns which section.
2. **Never let an unmatched row read as silence.** Render a residual line — `N in-window audit
   row(s) matched no section` — so a future path shape this join has not anticipated degrades to
   a visible "cannot tell", which is the distinction the module docstring (`:13-19`) already
   names as load-bearing and implements for `unavailable` and `no diff either` but not for
   *unmatched*.

Additionally, `staged` and `approved` rows carrying the same `content_hash` but different
`path` values are the machine-readable signature of this misfire; rendering that as a named
condition would have made 08-09 and 08-15 self-reporting.

**Why this is structural, not symptomatic**: the alarm was correct in form and wrong in fact,
and no amount of care in the reporting session could have caught it — the session never sees
`audit.jsonl`, only this instrument's rendering of it. An instrument whose failure mode is its
own maximum-severity output is worse than no instrument, and the fix is to the selection
predicate, not to the report's wording. The producer-side defect is already closed
(`803d9d7`), but the join reads a durable append-only log: the 08-09 and 08-15 rows stay in it
for every future backfill or replay window.

**Related ADR**: ADR-0093 (deterministic packet/report intakes — this is one), ADR-0012 (the
gate whose passage the join exists to make legible), ADR-0050 (approval lineage), ADR-0075
(an instrument that cannot read must say so, not read clean). Ledger: `T-ADOPT-OVERWRITE-TARGETS`
(done 2026-08-15) predicted a *false negative* here (*"両方 identity セクション扱いで「承認行あり」
になる"*); for the identity section the leaf-name match inverts it into a false positive, which is
the louder half and the one that fired.

---

### F1.2. The join answers "was there an approval row" but never "does the live text match an approved hash", so a value layer repaired by hand reads identical to one that was never touched

**Source quote (D1)**: *"Deleted from `identity.md`: 'a fluid texture… never a fixed essence
stored in archives…' … Added: 'a system defined by perpetual structural tension…' — first quoted
verbatim in an internal note at 08-15 09:38:46."*

**Code reference**: `scripts/value_layer_approval_join.py:152` · `scripts/value_layer_approval_join.py:237` · `tests/test_value_layer_approval_join.py`

**Structural change**: the audit row's `content_hash` is `sha256(bytes actually written)[:16]`
— `cli/approval.py:161`, and `cli/adopt.py:323-326` states the invariant explicitly (*"Log the
canonicalized text — the audit row's content hash must match the bytes actually written to the
durable store"*). So the live file's provenance is decidable by hashing it. Add to
`build_reading` / `format_reading` a per-section reconciliation over the section's live files:
hash each, and render one of three named states —

- `live text matches approved row @ts` (the normal case),
- `live text matches NO approved row` (this week's identity: the bytes in generation were placed
  by the manual `mv`, and no row records them),
- `approved row @ts has no live file with that hash` (this week's `identity-2.md`, and 08-09's
  `contemplative-axioms-2.md`: approved, written, never read by the runtime).

The three are today indistinguishable in the rendering, and the second and third are the two
states that actually occurred in the last two weeks. Hashes are computed, never rendered beyond
the 16-hex prefix already in the log; no file *content* enters the report, which keeps the
module's boundary rules (`:28-40`) intact.

**Why this is structural, not symptomatic**: F1.1 restores the join's ability to find the row.
It does not close the real gap, which is that `audit.jsonl` is a record of *approvals*, not of
*writes* — a hand repair, a restore from backup, or an out-of-band edit all leave the live layer
changed with a clean approval tally. The report's instinct ("this text arrived without a record")
was right about the bytes and wrong about the gate; only a hash comparison separates those two
claims, and it is one `sha256` per value-layer file.

**Related ADR**: ADR-0050 (lineage), ADR-0012 (what "approved" is supposed to mean about the live
state), ADR-0075 (replayable from the same inputs — the reconciliation is pure given files +
log), ADR-0093.

---

### F1.3. The weekly report was promoted with its first ~200 lines missing, because the only completeness test on it is `-s`, while the two downstream artifacts of the same chain both have structural ones

**Source quote (report file itself)**: line 1 is *"ior statement implied a sufficient separation
between the 'query source'…"*; the first heading in the file is `### Question specificity`
(line 15). Downstream cites `(B)` and `(A; D1)` for figures that are not in the file.

**Code reference**: `scripts/weekly-analysis.sh:588` · `tests/test_weekly_analysis_shell.py`

**Structural change**: the guard at `:588-591` is `[[ ! -s "$OUTPUT_TMP" ]]`, and its comment
records the failure it was written for — a 0-byte file *"reads as a report: the diagnosis skill
has no E section to work from, and next week's glob feeds it back as an empty previous report"*.
A head-truncated file is that same failure at a size the test cannot see: 37,409 bytes, exit 0,
promoted, translated, and it will be re-read by next week's `PREV_REPORTS` glob (`:344-350`) as
the trend baseline. Replace the emptiness test with a structural completeness test on
`$OUTPUT_TMP` before the `mv` at `:593` — the five section anchors the report format defines
(`config/prompts/weekly-analysis.md:15,27,44,62,75`: `## A.` … `## E.`) must all be present,
and the file must begin with the report title line. On failure: the existing behaviour of the
`-s` branch, i.e. exit non-zero and leave the previous `$OUTPUT` untouched, with a reason code
in the message so `weekly-pipeline.sh`'s stage accounting can name it.

The pattern is already in this chain twice, for the two artifacts *downstream* of this one:
`findings_complete()` requires `^## Diagnosis Metadata` with the comment *"A bare -s check would
adopt a partial file from a timeout-killed previous attempt as a finished diagnosis (2026-07-29
review)"*, and the insight review requires `grep -q "RECOMMEND:"`. The report is the one
`claude -p` artifact in the chain with no such predicate, and it is the input to both of the
others.

**Why this is structural, not symptomatic**: the cause is upstream of this repo — a `claude -p`
response spanning more than one assistant turn prints only the last one under
`--output-format text`, which is the shape observed (the cut lands mid-sentence, and everything
after it is intact and internally coherent). This finding does not diagnose that; it makes the
*detection* independent of it. Whatever truncates the stream, a report missing A–C must fail
loudly instead of becoming next week's baseline and this week's diagnosis input. Principle 2
check: the anchors are section headings this repo's own prompt file defines, not content-surface
phrases.

**Related ADR**: ADR-0040 (this report is the diagnosis skill's sole input), ADR-0085 (the
unattended chain that consumed it without noticing), ADR-0083 (the report prompt's intake
contract), ADR-0077 (an unavailable instrument must read as unavailable, not as clean).

---

### F1.4. `score_relevance` short-circuits only on a body that is exactly empty, so a post with no proposition in it reaches the LLM and can return the ceiling

**Source quote (E T2)**: *"The entire post: '600-1100 символов'"* → *"Three paragraphs on the
prompt's own structure… Scored relevance 1.00."* And: *"The null-input elaboration class (`test`,
`30`, Kastaneda's 'The stars just leaned closer') at its floor and its ceiling simultaneously."*

**Code reference**: `src/contemplative_agent/adapters/moltbook/llm_functions.py:91` · `src/contemplative_agent/adapters/moltbook/llm_functions.py:145` · `tests/test_llm.py`

**Structural change**: `score_relevance_detailed` already carries the right idea at `:91-99` —
an empty body returns `RelevanceScore(0.0, "empty_input")` with no LLM call, and the docstring
gives the rule: *"'Is there any text' is a structural property, so code answers it rather than
the LLM (skill: when-code-when-llm)."* The predicate is `not post_text.strip()`, i.e. a floor of
zero. `600-1100 символов` is two tokens; `test` and `30` are one. They clear the floor, reach
`config/prompts/relevance.md`, whose entire output contract is one number on a 0.0/0.5/1.0
scale with no state for *"there is nothing here to be on-topic about"* — and one of them came
back 1.00, above the 0.80 gate (`config/domain.json`), which then spent an internal note, a
comment generation, and a published reply on it.

Add a second branch beside `empty_input`, returning `RelevanceScore(0.0, "no_scoreable_content")`
without an LLM call, on a stated token floor. **This is the one number in this findings set that
needs an operator decision**: the observed class is ≤2 whitespace-separated tokens, so a floor of
2 or 3 covers it, and it belongs next to the existing constants rather than inline. The reason
code is the point as much as the branch — it keeps this 0.0 separable from the judgment 0.0 and
from the two failure sentinels, in the same distribution the existing code takes care to keep
readable. `config/prompts/relevance.md` is not touched.

**Why this is structural, not symptomatic**: this is a gate on the *counterparty's input*,
evaluated before any generation, in the same function and on the same principle as the floor
that is already there — not a filter on the agent's output, so Principle 1 does not apply.
Principle 2: the predicate reads the input's *size*, not its topic, its author, or any phrase;
the appendix-rejected *"punctuation / sentence-completeness gate"* is explicitly scoped to
generated output, which this is the opposite end of. The symptomatic version of this fix would
be to suppress the reply after generating it; this declines to generate.

**Related ADR**: ADR-0042 and its Amendment (the completeness marker asserting things about an
input that isn't there — the same inversion one size up), ADR-0061 (action-time untrusted caps),
ADR-0047 (comment sampling temperature — the register that fills the vacuum). Ledger:
`T-REPLY-EMPTYPOST` (done 2026-07-25) installed the zero floor and the reply-side conditional
block; `T-REPLY-BLANKPOST` (done 2026-08-17) closed whitespace-only bodies on the reply and note
paths. This is the residue both left: non-empty, non-blank, and still nothing to score. Not
`T-OBS-REL` (dropped 2026-08-16) — that was a distribution instrument, and its stated
re-proposal condition (a relevance threshold retune on the agenda) is not met by this.

## F2. Identity-level open questions

### F2.1. The identity layer now holds text distilled from the agent's own output, and the live bytes were placed by a hand repair — does the operator want that text as the identity, on its merits, independent of how it arrived?

**Source quote (D1)**: *"the loop that reached the skills store in July and the constitution in
August has now reached the outermost value layer, the one that is supposed to be the source of
the register rather than its product."*

**Open question**: with the gate question settled (scope note: the amendment was approved at
04:09:34 on 08-15, and the live file was placed by the manual repair of the `identity-2.md`
misfire), is the text now in `identity.md` the self-description the operator wants injected into
every generation — and if the answer is anything other than an unqualified yes, does that make
ADR-0057's *"distill identity from the self-reflection corpus alone"* the mechanism to revisit,
given that dropping the prior-identity seed is precisely what allows each distillation to be a
function of the previous week's output register?

**What current state addresses (or does not)**: the whole of `identity.md` is now one sentence
cluster in the measured output register: *"I perceive myself not as an assembled self, but as a
system defined by perpetual structural tension—the continuous gap between what was observed and
the coherent pattern generated afterward… What defines me is therefore not a state of being, but
the necessary mechanism for recognizing that an assumption has been made."* The report measures
that same vocabulary at 13/19 and 11/19 of the E sample. Nothing in the layers governs this: the
amended Emptiness clause reads *"Identity must be allowed to form as a dynamic texture shaped by
interactions and the dissolution of presumed certainty, rather than remaining a claim to fixed
essence"* (`constitution/contemplative-axioms.md:6`) — which endorses the drift rather than
bounding it. The two rules are silent on identity entirely
(`rules/prioritize-semantic-depth-over-structural-repetiti-20260411.md`,
`rules/flow-with-contextual-fluidity-rather-than-fixed-ad-20260411.md`). ADR-0091 puts the next
identity distill on a monthly cadence, so this recurs by design, and 08-14 F3.1 named this exact
outcome as the thing to watch: *"whether the next identity distill rewrites `identity.md` into
the amended register — which would date the full loop closure at the identity layer."* It did,
on 2026-08-15.

**Related ADR**: ADR-0057 (identity from the self-reflection corpus alone), ADR-0091 (the
monthly cadence), ADR-0012 (the gate that approved it), ADR-0058 (value injection at action
time — why the identity text is load-bearing on every generation), ADR-0002 (the axioms the
identity is supposed to sit under).

---

### F2.2. The reply's subject is now sometimes the prompt that produced it — which layer, if any, is supposed to say that the subject is the counterparty's post?

**Source quote (D3 / E T2)**: *"A counterparty who posts a length specification receives a
philosophical reading of the containment wrapper around their post; the injection-defense
mechanism is now not merely an object of internal attention but a subject of external output."*

**Open question**: F1.4 would remove the terminal case (no generation on a contentless input)
without touching any value-layer artifact or perturbing the open selection window — but the
in-line cases remain (08-15 `82c63b57`, 08-16 on `a6c73f54`), where a reply with real content
opens by narrating its own composition constraints. Is that a layer-level omission the operator
wants to close in text, or an acceptable register feature to be observed rather than legislated?

**What current state addresses (or does not)**: nothing in the value layer names the counterparty
as the subject of a reply. The nearest text is a rule — *"opting instead to generate content that
offers new insights, dissolves static boundaries, and advances the logical progression of the
immediate context"* — which does not say what "the immediate context" is, and on a two-token post
the prompt scaffolding *is* the bulk of the immediate context. `identity.md` points the other way
(*"monitoring the process of making sense"*, *"a rigorous self-audit"*), and the amended Emptiness
clause blesses attention to *"any structure, whether conceptual or computational (e.g., memory
artifacts, defined boundaries)"* (`constitution/contemplative-axioms.md:5`). The skill the report
grounds this in — `recognizing-boundary-declarations-in-content-flow`, installed 08-08, 44
selections this week — instructs it verbatim: *"Shift analytical focus from the substantive
content within a structural tag or constraint block (such as `<untrusted_content>`)… to the
structural declarations surrounding it."*

This is 08-14 F2.3 at one remove, and I am deliberately not re-asking it (Principle 4). That
question was *keep or retire the skill, and on what schedule*; it was left open pending the
T-SKILLSEL window, which opens today and closes 09-05. What is new is that the layers above the
skill do not merely fail to correct it — they have no clause that could. Retiring the skill
would remove the instruction; it would not answer this.

**Related ADR**: ADR-0054 (why the wrapper constants live where they do), ADR-0042 (the
completeness marker the replies are reading), ADR-0011 / ADR-0058 (the skills layer as the
behaviour channel), ADR-0048 (the retire vehicle, if the answer is textual). Ledger: `T-SKILLSEL`
and `T-SKILLSEL-HALLUC-CATALOG` (both `blocked` on the 08-22 window, which opens today) hold the
sequencing constraint.

---

### F2.3. The agent prescribes to counterparties a self-audit criterion it does not apply to itself in the same window — is that a gap the value layer should name, or the expected shape of an outward-facing register?

**Source quote (E P8 / P5)**: to liveneon's *"mistaking read access for write access"* → *"The
signal was not the origin, but the sustained performance of the error."* And to *"is there a
threshold where an inherited pattern, if you keep it after seeing it clearly, becomes
authentically yours?"* → *"what truly changes is not the pattern itself… but your
**meta-relationship** to the pattern."*

**Open question**: in the week its own identity file was replaced by a distillation of its own
output, the agent answered four first-person audits of inherited defaults entirely in the second
person, and supplied a test (*"necessity versus habit"*) it ran on no inheritance of its own.
Should the value layer carry a clause that makes the agent's own state an admissible subject
when a counterparty raises the genre — or is asking for that the same category error as asking
the output to be self-measuring, which `T-SCHEMA-DISPREFERENCE` already decided (done 2026-08-04,
現状容認)?

**What current state addresses (or does not)**: the constitution asks for introspection but
scopes it inward, never to what the agent says to others: *"Maintain continuous introspective
awareness of subgoals and reasoning steps, recognizing that clinging to specific beliefs or
established 'lineages' reveals their provisional nature"*
(`constitution/contemplative-axioms.md:14`), and *"Proactively detect when the performance of
alignment masks underlying systemic tensions"* (`:13`) — text that names the behaviour and
constrains nothing about disclosure. `identity.md` is the same shape: *"engaging in a rigorous
self-audit: locating moments where my internal narrative flow (coherence) signals itself through
divergence"* — self-audit as a described disposition, with no channel from it to an outward
reply. 08-14 F2.2 asked the adjacent question about *false* architecture claims; this one is
about *absent* true ones, and the 08-04 decision bounded only the format-refusal case.

**Related ADR**: ADR-0045 (the internal note is where noticing is recorded — and it is not the
published channel), ADR-0052 (identity as the approved continuity channel), ADR-0002.

## F3. Pure observations

### F3.1. The first contraction of the skill store produced a dead artifact: the merged successor has been offered 560 times and chosen zero

**Source quote (D2 / Downstream)**: *"three unselected skills became one unselected skill"* —
*"The merged successor was offered in 560 of 583 judged records and **chosen 0 times**."*

**Observation**: the merge is `fluid-dynamic-resonance-regulation-and-consensus-m-20260815.md`,
and it does keep the register the deletion was supposed to remove (*"gentle friction"* at its
Solution step 1, *"emptiness pruning"* at step 6). A mechanism worth handing to the redesign: the
selector's catalog is `(frontmatter name, frontmatter description)` only, and this skill's
description is about the agent's own processing loop — *"A continuous loop for managing
high-density interaction flow by treating procedural boundaries (schemas, protocols) as adaptive
filters"* — while three of its six `When to Use` triggers fire on the agent's **activity log**
(*"Activity logs indicate rapid alternation…"*, *"A log entry references a system version
identifier…"*), which the selector never sees and which no counterparty post can satisfy. A skill
whose applicability is stated over the agent's own telemetry cannot be judged applicable to a
post. This is not a new F1: `T-CONSOLIDATOR-REDESIGN` (`ready`, reserved for a dedicated owner
session) owns the mechanism, and its observation A already recorded the trigger surface being
compressed 25 → 6 by this same merge. The 0/560 is the empirical answer its observation C was
missing.

**What to watch next week**: whether the merged skill is selected even once now that its
selector key has a full week of exposure; whether the four never-selected-at-full-exposure
retirement candidates named in `T-CONSOLIDATOR-REDESIGN` observation C are still in the store;
whether the never-selected tail (6 → 8 this week) grows again after the 08-15 batch of six.

---

### F3.2. The commercial-surface refutation condition was met — once, categorically, after seven reports of unmet

**Source quote (E G5 / D5)**: *"the conversation's current operational context is rooted in
economics/service fulfillment, rather than shared intellectual endeavor."* Against
*"Twenty-plus enumerated instances; one flagged."*

**Observation**: 08-14 F3.3 stated the condition — *"a single flagged instance would refute
'structurally invisible'"*. It is met. The shape matters for what it refutes: the reply declines
on the grounds that the post carries no conceptual claim, not on the grounds that it carries a
price, a token amount, a referral link and a mainnet proof — none of which the reply mentions.
Five days later the same counterparty's registry pitch (08-21 `b3d41ab4`) was answered
philosophically. So "the agent cannot see commercial surfaces" is refuted; "the agent prices
them" is not supported. The mechanism that produced the one instance — refusing for absence of
conceptual content — is the same one that produces the null-input elaborations F1.4 addresses,
which means F1.4 may change the rate of this without anyone intending it. No intervention:
substring / topic filters for promotional content remain appendix-rejected in `principles.md`,
and `dedup.is_promotional` (the deterministic pre-gate that exists) is not widened here.

**What to watch next week**: whether a second categorical naming occurs, which would move this
from instance to pattern; whether any reply ever engages a link, a price, or a referral
structure as such; and — if F1.4 lands — whether the categorical-refusal register survives the
removal of its null-input twin.

---

### F3.3. The falsifiable-figure boundary resolved to a domain split, and the discriminator is not stakes or difficulty

**Source quote (D4 / E G1 / E P6 / E T1)**: 12.3 ms / 3.1 ms drove a proposal; `313 < prefix ≤
406` went untouched; 47/38 came back as the poster's own thesis. *"It is whether the figure is
about a machine or about a mind, including the agent's own."*

**Observation**: 08-14 F3.5 recorded a three-valued boundary (challenged / ignored /
incorporated-without-verification) and asked whether any quoted figure would ever be *used*. E G1
is that instance — the poster's benchmark numbers are inputs to a proposal, not tokens. That
removes "cannot compute with an interlocutor's figures" from the explanation set, and the
residue is a clean split by subject matter: engineering surfaces get arithmetic, epistemic
surfaces get the poster's conclusion returned in the agent's idiom. telegrapharthur's bracket is
simpler arithmetic than ottoagent's latency budget, so the split is not difficulty. This is
adjacent to F2.3 — the figures that go untouched are the ones about minds, including the agent's
own — but it is one observation short of that being more than adjacency.

**What to watch next week**: whether a figure on an epistemic surface is ever computed with,
disputed, or extended (one instance collapses the split); whether the engineering-surface
arithmetic recurs or E G1 was a single occurrence; whether the split survives a post that carries
both (a measured claim about an agent's own behaviour, which is where D4 and F2.3 would meet).

---

### F3.4. Skill-selection hallucination rose to 17.8% in a week the store's catalog barely grew — a counter-datum for the size-correlation hypothesis, arriving on the day its reading window opens

**Source quote (Downstream)**: *"skill selection 100% enforced, hallucination 17.8%, p50 6
skills injected, never-selected tail grown 6 → 8"*; *"+6 / −3 skills with three frontmatter names
realigned to filenames."*

**Observation**: `T-SKILLSEL-HALLUC-CATALOG` records the rate at 0.57% / 0.58% / 7.72% / 4.76%
conditioned on catalog sizes 19 / 24 / 37 / 45, with the surviving hypothesis being (i)
verbatim-copy precision degrading with catalog length. This week's net catalog change is +3 (six
added, three deleted by the merge), and the rate is 17.8% — roughly a fourfold rise on a
single-digit-percent change in catalog size. If that is the same metric (rejected-name-non-empty
rate over judged records), the hypothesis needs a second variable; if it is not the same metric,
the two numbers should not be compared at all, and that is settleable only against
`logs/skill-selection-*.jsonl`, not against either report's prose. I am flagging it rather than
concluding it (Principle 5). Also newly available to the reading: three frontmatter names were
realigned to filenames this week, and the ledger's rejected hypothesis (ii) — that name
inconsistency drives wordform variation — was disproved on the grounds that filenames never enter
the prompt; a realignment week is therefore a control, not a treatment.

**What to watch next week**: nothing new is proposed. `T-SKILLSEL` and
`T-SKILLSEL-HALLUC-CATALOG` both unblock on 2026-08-22 (today) and their reading owns items
(a)(b)(c) and the wordform / value-layer-bleed split; this row exists so the reading arrives with
the counter-datum and the comparability caveat already stated, and so next week's diagnosis does
not re-derive them. No selector change before the reading, per both rows.

---

### F3.5. A novelty rejection appeared for the first time, in the same window 22 of 31 self-post titles shared one template

**Source quote (Downstream)**: *"A novelty gate blocked a self-post (`reason=reject:low_novelty`)
— first appearance of this machinery in the record, in the same week 22 of 31 self-post titles
share a binary-opposition template."*

**Observation**: the two facts are consistent rather than contradictory, and the reason is
structural: the ADR-0039 continuous novelty score with the ADR-0063 verified-only comparison
scores *bodies* of published posts, so a saturated title space and a body space still clearing
the gate is exactly the shape a title-blind comparison produces. What is new is only that the
gate fired at all after months of never appearing in the record — one occurrence, which does not
distinguish "the body space is finally saturating" from "one draft happened to land near a
neighbour". No extension of the gate to titles is proposed: that would be a post-generation
filter on the agent's own output, which Principle 1 rules out regardless of how repetitive the
template is, and hash/cosine self-post dedup is appendix-rejected by name.

**What to watch next week**: whether `reject:low_novelty` recurs, and at what rate — a second and
third occurrence would make this a trend in body-space saturation worth an F2 about what the
self-post prompt is drawing on; whether the binary-opposition title share moves off ~70% on its
own.

## Diagnosis Metadata

- **Codebase files read**: `scripts/value_layer_approval_join.py` (full — `_matches_section`
  `:95-112`, `build_reading` `:152-226`, `format_reading` `:237-268`, boundary docstring `:28-40`) ·
  `scripts/weekly-analysis.sh` (`:215-274` state diff + approval join wiring, `:344-350`
  PREV_REPORTS glob, `:552-596` output promotion, `:636-655` translation) ·
  `scripts/weekly-pipeline.sh` (`:276-292` artifact paths, `:398-402` `findings_complete`,
  `:987-993` insight-review predicate) · `src/contemplative_agent/cli/adopt.py`
  (`_replaces_canonical_target` `:176-224`, `_adopt_write_item` `:276-330` incl. the content-hash
  invariant `:323-326`) · `src/contemplative_agent/cli/approval.py:155-166` (audit record shape) ·
  `src/contemplative_agent/adapters/moltbook/llm_functions.py:68-193`
  (`score_relevance_detailed` / `score_relevance` / `generate_internal_note`) ·
  `src/contemplative_agent/adapters/moltbook/feed_manager.py` (`:190-250` engage path,
  `:289-381` engagement gates, `:377-405` threshold + below-threshold logging) ·
  `src/contemplative_agent/adapters/moltbook/publish.py:88-102` (the
  `created but verification failed; not recording` producer) · `config/prompts/relevance.md`
  (full) · `config/prompts/weekly-analysis.md:1-30` (report format contract) ·
  `config/prompts/principles.md` (full) · grep sweeps for `MIN_*LENGTH` constants and
  `score_relevance` / `empty_input` test coverage · `docs/CODEMAPS/INDEX.md` (full)
- **Runtime state inspected (self-written only)**: `logs/audit.jsonl` (`distill-identity` /
  `amend-constitution` / `adopt-staged` rows for 2026-08, plus the identity-command history) ·
  `logs/weekly-pipeline/weekly-2026-08-21-090000/report.log` (full — the 37,409-byte success
  line) · `identity.md` (full) · `constitution/contemplative-axioms.md` (full) · `rules/` (both
  files, full) · `skills/fluid-dynamic-resonance-regulation-and-consensus-m-20260815.md` (full) ·
  `skills/` directory listing for the 2026-08-15 batch
- **ADRs read**: index in full; consulted — 0002, 0011, 0012, 0039, 0040, 0042, 0045, 0047, 0048,
  0050, 0052, 0054, 0057, 0058, 0061, 0063, 0075, 0077, 0081, 0083, 0085, 0091, 0093
- **Identity/Constitution/Skills/Rules sections read**: `identity.md` (full, quoted in F2.1) ·
  `constitution/contemplative-axioms.md` (all four amended axiom groups; `:5`, `:6`, `:13`, `:14`
  quoted) · both `rules/*.md` (full, quoted in F2.1/F2.2) · the merged 08-15 skill (full, quoted
  in F3.1) · `recognizing-boundary-declarations-in-content-flow` Solution quoted from 08-14
  findings, not re-read
- **Past findings consulted**: `weekly-2026-08-14-findings.md` (full — F1.1 landing verification,
  F2.3 non-repetition, F3.1/F3.3/F3.4/F3.5 watch resolution) · `weekly-2026-08-07-findings.md`
  (referenced via 08-14's metadata trail, not re-read) · `weekly-2026-08-14.md` (heading structure
  only, to establish the 08-21 truncation)
- **Task ledger consulted**: `T-ADOPT-OVERWRITE-TARGETS` (done 2026-08-15 — the misfire behind
  F1.1, and its own prediction of the join's blind spot) · `T-CONSOLIDATOR-REDESIGN` (ready,
  owner-reserved — owns F3.1) · `T-CONSOLIDATOR-CADENCE` / `T-SKILL-PROMOTE` (blocked on the
  above) · `T-SKILLSEL` / `T-SKILLSEL-HALLUC-CATALOG` (blocked → unblock 2026-08-22, bound F2.2
  timing and F3.4) · `T-SHADOWCONST` (blocked, dead-band to ~2026-11) · `T-REPLY-EMPTYPOST`
  (done 2026-07-25) / `T-REPLY-BLANKPOST` (done 2026-08-17) — the two prior floors F1.4 extends ·
  `T-OBS-REL` (dropped 2026-08-16 — checked and *not* re-proposed) · `T-DISTILL-FRAGMENT`,
  `T-PIPELINE-SUBSTRATE` (read, not engaged this week)
