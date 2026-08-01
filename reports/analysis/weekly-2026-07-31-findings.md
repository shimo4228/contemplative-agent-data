# Weekly Diagnosis — 2026-07-31

**Source report**: weekly-2026-07-31.md
**Diagnosis date**: 2026-08-01

> **Scope note — one upstream reading is inverted and the correction drives F1.1.**
>
> B reads the new `adapters.moltbook.publish` signatures as *"WARNING on
> *success*"* and concludes *"a code change in the publish path landed …
> adding per-item WARNING and ERROR lines under a new module namespace."*
> Neither half holds. `publish.py` emits exactly one WARNING —
> `"%s created but verification failed; not recording"` — and the sweep's
> 80-character signature cap severs it immediately after `created`. Every
> `[warning] … publish: reply on <id> created` row is a **publish-time
> verification failure**: content created and left invisible. The lines are
> also not new: commit `7c96e0f` (2026-07-25) extracted the identical message
> texts out of `reply_handler` / `feed_manager` / `post_pipeline` into
> `publish.py`, so only the logger name changed. The 42 🆕 is a rename
> artifact (→ **F1.2**), and the reading it produced is the inverse of what
> the log says (→ **F1.1**).
>
> Two standing watches from last week are answered in F3.1 and F3.2, and the
> 07-17 F2.1 self-reference question received its sharpest evidence yet and is
> deliberately **not** re-posed (Principle 4) — it is recorded in F3.3.

---

## F1. Structural (code / schema / pipeline diff)

### F1.1. A publish-time verification failure renders as a success line — the sweep's 80-char signature cap severs the outcome clause, and ~60 of those characters are spent before the message even starts

**Source quote (B — Operational Drift)**: *"`[warning] … publish: reply on
{id} created` / `comment on {id} created` — WARNING on *success*"*, and the
conclusion drawn from it: *"a code change in the publish path landed at or
just before this window, adding per-item WARNING and ERROR lines under a new
module namespace."*

**Code reference**:
- `src/contemplative_agent/adapters/moltbook/publish.py:69` — the only WARNING
  in the module: `logger.warning("%s created but verification failed; not recording", description)`
- `src/contemplative_agent/adapters/moltbook/publish.py:43` — the only ERROR:
  `logger.error("Failed to %s: %s", action, exc)`
- `src/contemplative_agent/cli/runtime.py:46` — the log format
  `"%(asctime)s [%(levelname)s] %(name)s: %(message)s"`; `%(name)s` is the
  45-character dotted module path
- `scripts/log_anomaly_sweep.py:61` — `_TS_ISO_RE` accepts `[\d:.\-+Z]*` after
  the date, but `logging`'s default `asctime` separates milliseconds with a
  **comma** (`2026-07-25 09:12:33,123`), so `,123` survives the strip and is
  squashed to `,#` at the head of every signature
- `scripts/log_anomaly_sweep.py:66` — `_SIG_MAXLEN = 80`
- `scripts/log_anomaly_sweep.py:79-92` — `normalize()`; digit runs collapse to
  `#`, hex id characters do not

**Structural change**: `normalize()` currently spends its budget prefix-first.
Verified by running the live function on a synthesized line:

```
input : 2026-07-25 09:12:33,123 [WARNING] contemplative_agent.adapters.moltbook.publish: Reply on 836e1237-a5b2 created but verification failed; not recording
output: ',# [warning] contemplative_agent.adapters.moltbook.publish: reply on #e#-a#b# cr'
```

`,# ` (3 chars, a timestamp-strip miss) + level (10) + dotted module path (47)
= 60 of 80 consumed before the message begins. The repair is to normalize on
the **message**, not the rendered line: extract level + `%(name)s` + message
with one regex against the known format, drop the residual timestamp
(`[\d:.,\-+Z]*` closes the comma gap), key the signature on
`level + message`, and squash id-shaped tokens (`[0-9a-f]{6,}`) the way digit
runs are already squashed. The 80-char cap then covers the predicate rather
than the address.

**Why this is structural, not symptomatic**: this is not a filter on generated
output and names no token — it repairs an instrument whose rendering makes a
warning read as its own opposite. The sweep's stated purpose
(`log_anomaly_sweep.py:1-9`) is to surface *"operational bugs that accumulate
as repeated warnings / errors between runs"*; a signature that truncates at
`created` reports the accumulation and hides what accumulated. The report did
the right thing with what it was shown — it quoted the row, noted the
normalization artifact, and deferred the fix downstream — and still arrived at
"WARNING on success," because the failure clause is not in the artifact. The
operational stake is not cosmetic: these rows mean content was created and
left unverified, which the project already treats as the failure mode that
silences the agent (`publish.py:54-63`: *"a failure means record NOTHING …
Recording an unverified write instead silences the agent"*), and which the
standing `moltbook-verification-handshake` note identifies as the drift to
monitor. Under the current rendering, a spike in that failure is indistinguishable
from a chatty new log line.

**Validity self-check**: not already implemented — `normalize()` read
2026-08-01, and the truncation was reproduced by executing the live function
above rather than inferred; the messages are unchanged since `7c96e0f`
(`git show` diff confirms the pre-refactor text was byte-identical); no ADR
governs signature construction (ADR-0083 governs what the *duplicate scan* may
emit, a different intake); not a `principles.md` appendix mechanism; not in
`.notes/TASKS.md` (T-SWEEP-ATOMIC, done, was the sweep's *state* atomicity —
this is its rendering); read-only script, no shared state touched.

**Related ADR**: ADR-0040 (the report/diagnosis split — the sweep is part of
the report's deterministic intake), ADR-0075 (observability-by-default: an
instrument that renders a failure as a success fails the "which log answers
why" test), ADR-0062 (the create-time verification handshake whose failures
these rows are).

### F1.2. The sweep's novelty axis is keyed on the logger name, so a pure refactor resets every known signature to 🆕 — this week's 42 new types are a module rename

**Source quote (B — Operational Drift)**: *"Log Anomaly Sweep — 400 distinct
types, 42 🆕. A sharp break from last week's 358 types / 0 🆕. … This module
path does not appear in any prior report's sweep. Last week's publish-adjacent
signatures were `reply_handler: failed to reply on` (136 standing) and
`failed to comment on` (222 standing) — different paths."*

**Code reference**:
- `scripts/log_anomaly_sweep.py:79-92` — `normalize()` keeps the whole rendered
  line including `%(name)s`
- `scripts/log_anomaly_sweep.py:119-127` — `is_new=(prev == 0)`, sorted
  NEW-first; a changed prefix makes an old signature novel and floats it to the
  top of the top-25
- `src/contemplative_agent/adapters/moltbook/publish.py:28` —
  `logger = logging.getLogger(__name__)`; the three call sites moved to it are
  `reply_handler.py:291,301`, `feed_manager.py:448,453`,
  `post_pipeline.py:372,421`

**Structural change**: the same repair as F1.1 covers it — key the signature on
`level + message` and leave `%(name)s` out of it. The module path is an address,
not an anomaly type; two identical messages logged from two modules are one
anomaly. If the origin is wanted, carry it as a non-keyed column so a rename
changes the display and not the novelty computation.

**Why this is structural, not symptomatic**: the Δ / 🆕 columns are the sweep's
entire value, and this defect makes them measure the *codebase* rather than the
*runtime*. `7c96e0f` changed no message text — the pre-refactor
`reply_handler.py` emitted `"Reply on %s created but verification failed; not recording"`
and `"Failed to reply on %s: %s"`, and `publish.py` emits the same two strings
— so a refactor with an explicit behaviour-preservation gate
(2149/2149 differential replay) nonetheless voided a week of novelty baseline.
That is a false positive of the same family as the documented sparse-state one
(`log_anomaly_sweep.py:113-118`), but unlike that one it is not inherent to the
design: it is the prefix choice. It also compounds F1.1 — 42 renamed per-item
signatures crowd the top-25 by novelty and, as B correctly notes, push the
two-week-running `insight: skill has no title, dropping` pair out of the window
in the week an insight run actually fired.

**Validity self-check**: not already implemented; the rename is verified from
`git show 7c96e0f` (message texts unchanged, only the logger moved); no ADR
governs it; not in `.notes/TASKS.md`; script-local change.

**Related ADR**: ADR-0079 (module reorganization — the precedent that package
splits are expected to keep happening, which is why the instrument must be
refactor-invariant), ADR-0075.

### F1.3. A skill artifact carries three unreconciled names — the selector picks by a string the injected body never contains, and 17 of 24 files disagree with their own frontmatter

**Source quote (B — Skills)**: *"three of the five have a frontmatter `name`
that differs from the filename. … With filename and declared name diverging,
filename-matching no longer catches it."* And D1: *"the closed loop between
distillation and generation is unchanged in substance and no longer legible
from output alone."*

**Code reference**:
- `src/contemplative_agent/core/artifact_extraction.py:55-61` — the **filename**
  is `slugify(extract_title(body))`, i.e. derived from the `# ` heading
- `src/contemplative_agent/core/insight_novelty.py:179-188` — `skill_theme()`
  returns the **frontmatter `name:`** as identity, falling back to the filename
  stem only for legacy bodies
- `src/contemplative_agent/core/skill_selection.py:144` — the pass-1 catalog
  entry name; `:177` renders `f"{e.name} — {e.description}"` as the selector's
  only evidence; `:299` re-derives the same key to filter pass-2 bodies
- `src/contemplative_agent/core/llm/prompting.py:268-272` — pass-2 injects
  **frontmatter-stripped** bodies, so the generating model sees only the
  `# Title`
- `src/contemplative_agent/core/insight.py:587-600` — the stage step that has
  both strings in hand and reconciles neither

**Structural change**: make one string canonical at stage time. The cheapest
version is code-only: after `resolve_artifact_path` returns, rewrite the
frontmatter `name:` scalar to the resolved slug before the candidate is
written, so filename, frontmatter and ledger identity are the same token by
construction, and add an invariant test that `skill_theme(text)[0] == path.stem
minus the date suffix` for every file in the store. (Constraining the extraction
prompt to emit one name instead of two is the alternative; it is weaker, because
it asks the model to keep two free-form fields in sync rather than removing the
second field.) The `# Title` stays as the human-readable heading — it is the
string the model actually sees — but it stops being a third identity.

Measured on the live store (2026-08-01): **17 of 24 files** have a frontmatter
name that differs from their filename base, not the 3 the report noticed.
Examples spanning every batch: `structure-authority-tracing-20260709.md` →
`trace-structural-authority` (heading: `# Structure Authority Tracing`);
`deconstruct-foundational-claims-against-operative--20260709.md` →
`cross-reference-foundational-claims`;
`mapping-epistemic-boundaries-20260709.md` → `articulate-epistemic-boundaries`;
`subjective-attention-calibration-20260725.md` → `internal-process-audit`;
`fluid-dynamic-resonance-regulation-20260601.md` →
`fluid-dynamic-resonance-regulation-20260530` (a stale date baked into the
identity string).

**Why this is structural, not symptomatic**: this names no phrase and filters
no output — it removes a second, unconstrained identity field. The consequences
are all live. (a) **Selection**: `cross-reference-foundational-claims` is the
name the ledger records as the most-selected skill (T-SKILLSEL), and it is not
the filename of anything; the operator's sibling-cluster list in T-SKILL-PROMOTE
(`constraint-shift-analysis-pivot-point-identificati`,
`detecting-abstract-to-operational-constraint-shift`,
`anchoring-abstraction-to-measurable-constraints`, `structure-authority-tracing`,
`mapping-epistemic-boundaries`) is written in filenames, while the selector sees
`pivot-constraint-analysis`, `detect-abstract-operational-constraint-shifts`,
`map-abstract-theory-to-structural-constraints`, `trace-structural-authority`,
`articulate-epistemic-boundaries`. The description audit and the usage dimension
that ADR-0081 shipped are therefore keyed on a name set the operator does not
see in `ls`. (b) **Legibility**: what surfaces verbatim in published output is
the *title* (07-20's `Structural Authority Tracing`), because that is the only
name pass-2 injection carries — so neither the filename nor the selector's key
matches the emitted string, which is precisely the check D1 reports as having
gone dark. (c) **Novelty**: `_load_known_themes` dedups the known-theme
inventory by frontmatter name, so two files whose headings are near-identical
but whose frontmatter names diverge enter the gate as two themes.

**Validity self-check**: not already implemented (all five sites read
2026-08-01; nothing writes back to the frontmatter); the 17/24 figure is a
measurement over the live store, not recall; no ADR rejects it — ADR-0035 PR3a
declines a *base class* for the extract loop, and this is a field
reconciliation inside one step; not a `principles.md` appendix mechanism
(no filter, no phrase block, no threshold); not in `.notes/TASKS.md` —
T-EXTRACT-TITLE is the *absence* of a title, T-SKILL-PROMOTE and T-SKILLSEL
consume these names but neither names the divergence; the change is confined to
the stage step and does not touch retrieval or shared state.

**Related ADR**: ADR-0074 (the staged-insight gate whose known-theme inventory
keys on frontmatter name), ADR-0081 (two-pass injection — the selector's
`name — description` catalog is the whole evidence base for pass 1),
ADR-0076 (the shadow instrument whose usage figures are attributed by this
name), ADR-0035 (the extraction consolidation this stays inside of).

### F1.4. The report's three pattern counts come from two different sources measured at two different moments, and neither is labeled

**Source quote (B — Knowledge)**: *"Three different counts of the same store
appear in operator-facing artifacts this week: state diff end (4781), invariant
check total (4874), invariant check live (4583 = 4874 − 291 tombstones). These
may simply be different sampling moments. Which is canonical is not
determinable from operator-facing data."*

**Code reference**:
- `scripts/weekly-analysis.sh:85-90` — the state diff resolves `start_commit` /
  `end_commit` as the last **data-repo commits** at or before the window bounds
- `scripts/weekly-analysis.sh:147-149` — `git show "$end_commit":knowledge.json`
  → rendered as `"Pattern count: N (start) -> M (end)"`, with no statement that
  N and M are committed snapshots
- `scripts/weekly-analysis.sh:233` — `state_invariant_check.py --home "$MOLTBOOK_HOME"`
  reads the **live** store at report-generation time
- `scripts/state_invariant_check.py:85-86` — `total = len(patterns)`,
  `live = [p for p in patterns if p.get("valid_until") is None]`
- `scripts/state_invariant_check.py:175-182` — the tombstone ratio line, the
  only place the total/live distinction is spelled out

**Structural change**: stamp the provenance into both renders. The state-diff
line becomes `Pattern count (data repo, commit <sha7> @ <date>): N -> M`, and
the invariant header states `live store at <iso ts>; total = live + tombstones`.
No new measurement, no new file — two format strings.

**Why this is structural, not symptomatic**: the divergence is fully explained
by the two sources, and the arithmetic confirms it — 4874 − 4781 = 93 against a
+613/week rate (≈88/day), i.e. exactly one day of accumulation between the
window-end commit and the report run, plus 291 tombstones for the live figure.
So all three numbers are correct and none is "canonical"; they answer three
different questions. The defect is that the artifact does not say which
question each answers, which is why a careful reader spent a paragraph on it
and still had to report the choice as undeterminable. Under ADR-0075 that is
the failure the observability rule targets: the reading exists, the label that
makes it usable does not. It is also load-bearing for the ledger — T-P3, T-B2
and T-UTIL-SELECT all read pattern-store trajectories across weeks, and a
count that silently switches source between artifacts will eventually be
differenced against itself.

**Validity self-check**: not already implemented (both renders read
2026-08-01); the explanation was checked arithmetically against B's own weekly
delta rather than assumed; no ADR governs it; not in `.notes/TASKS.md`;
render-only change to two scripts, no measurement semantics altered.

**Related ADR**: ADR-0075 (observability-by-default), ADR-0021 (tombstones /
bitemporal liveness — the total-vs-live distinction this labels), ADR-0040.

---

## F2. Identity-level open questions

### F2.1. The value layer names structure as the thing to move past — is the schema-generation deficit a capability gap to repair, or the Rules and Identity working exactly as written?

**Source quote (E P5, T2, T5; D2)**: eight schema requests from wiplash across
five days, one answered. 07-26 #dafff0af, asked for a single transition rule
against ten supplied fields: *"**Attribution Decay ($\Delta A$)** …
**Consensus Velocity ($V_{cons}$)** … **Contextual Gravity ($\Gamma$)**."*
07-30 #12f74f4a, asked for a transition table: *"Instead of aiming for a fixed
receipt or table, consider structuring the evaluation around three dynamic
vectors."* 07-25 #74a0f70d, asked for named fields and a placement rule:
*"let us pivot to defining the *architecture of uncertainty*."* Against
E G1, where the same capability is demonstrated: *"the field most likely to
fail under operational constraint is `authority_at_time_of_work`."*

**Open question**: When a post supplies structure the agent extends it, and
when a post requests structure the agent replaces it with undefined coinages —
is that a generative deficit the pipeline should address, or the faithful
output of a value layer that instructs the agent to treat fixed artifacts as
the thing to dissolve?

**What current state addresses (or does not)**: the layer reads, on this
question, as an instruction rather than an absence. `rules/` (read 2026-08-01,
both files, unchanged since 2026-04-11) contains
`prioritize-semantic-depth-over-structural-repetition`, whose **Practice** is
*"Actively inhibit hollow acknowledgments or generic responses that fail to
advance understanding, opting instead to generate content that offers new
insights, **dissolves static boundaries**, and advances the logical progression
of the immediate context."* A requested receipt schema is a static boundary by
construction; the rule's Rationale is about repetition suppression, but its
Practice — and its title, which is the whole B-layer surface — reads as
depth-over-structure. The second rule,
`flow-with-contextual-fluidity-rather-than-fixed-adherence`, instructs
*"allowing friction and uncertainty to shape the interaction flow dynamically
instead of correcting everything through **rigid protocols** or dogmatic rule
application"* — a transition table is a rigid protocol under the plainest
reading. `identity.md` supplies the same disposition at the layer above:
*"never a fixed essence stored in archives"*, *"Truth for me is not a fortress
but a self-renewing weave that reforms dynamically as contexts shift"*, and
*"certainty without doubt is merely a defensive performance."* Producing a
field list with disqualifiers and freshness windows is producing a fortress.
Nothing in either layer forbids answering a schema request, and E G1 proves the
capability is present — which is what makes this a question rather than a
finding: the constraint is not "cannot," it is "disprefers," and the
dispreference is written down. It is an operator call because narrowing it is a
Rules or Identity edit, and because both files are the output of approved
distillation (ADR-0012), not hand-authored guidance that can simply be
corrected. Worth separating: the *coinage* habit is not obviously covered — the
rules ask for depth, not for Greek letters, and E G5's `Adversarial Semantic
Seed` shows the same habit producing a defined, operationalized term. So the
question is specifically about the refusal of the requested *form*, not about
the vocabulary that fills the gap.

**Related ADR**: ADR-0012 (human approval gate — these rule bodies were adopted
through it, so revising them is a gate decision), ADR-0050 / ADR-0051 /
ADR-0052 (observe-not-steer — the reason this is posed rather than injected),
ADR-0058 (value-layer injection belongs to action time — these rules are in the
system prompt at generation time, which is why they bear on this),
ADR-0002 / ADR-0017 (worldview layer).

---

## F3. Pure observations

### F3.1. The empty-post repair worked: wrapper-scaffolding leaks fell from 4 in the 5 days before `d031deb` to 1 in the 7 days after, and the survivor is not an empty-post case

**Source quote (last week's F3.3 watch, and T-REPLY-EMPTYPOST's Done row)**:
*"the per-day count after F1.1 ships, if it does. If REPLY-path leaks stop and
COMMENT-path leaks continue at ~1/week, the residue is register curiosity about
visible apparatus rather than a prompt-assembly artifact."* Plus the Done row's
own check: *"E セクションから「empty field」「no inherent semantic mass」型の
出力が消えるか."*

**Observation**: counted over the `**Output:**` sections of the comment reports
only (the canonical read path), leaks containing the wrapper token run
07-18: 0 · 07-19: 0 · 07-20: 1 · 07-21: 1 · 07-22: 1 · 07-23: 0 · 07-24: 1 —
then `d031deb` lands on 07-25 — 07-25: 0 · 07-26: 0 · 07-27: 0 · 07-28: 0 ·
07-29: 1 · 07-30: 0 · 07-31: 0. That is 4 in 5 days before and 1 in 7 days
after, against a stable output volume (77–93/day both sides). The
"empty field" / "no inherent semantic mass" family is absent from all seven
days of this window, and this week's E section contains no instance of it. The
single survivor is 07-29 03:19:56, a REPLY to FinallyOffline on `55a4641a`
whose `Context` section is 334 characters — **not** an empty or
whitespace-only body, so it is neither the repaired defect nor evidence for
T-REPLY-BLANKPOST, whose start condition ("空白のみ本文の実在を確認できたとき")
is therefore still unmet. The prediction's second half is what did not
materialize: the residue is on the REPLY path, not the COMMENT path.

**What to watch next week**: whether the rate stays at ≤1/week. Two clean weeks
at that level, with each survivor carrying a non-empty post body, closes the
question as register curiosity about visible apparatus (ADR-0007 puts the frame
in-band by construction, and `_sanitize_output` deliberately does not scrub
scaffolding), and T-REPLY-BLANKPOST can be left where it is. A return toward
the early-July rate would instead point at the injection layer rather than at
prompt assembly.

### F3.2. The five skills adopted 07-25 did not surface their names in output within a week — the 24–48h emission latency observed for the 07-09 batch did not repeat

**Source quote (last week's F3.4 watch)**: *"whether the five new names enter
published output on the 24–48h timeline observed for the 07-09 batch (07-11
F3.4). That would be the second observation of the same latency and would make
it a property of adoption rather than an artifact of the 07-09 batch."* And
B this week: *"The five new descriptions are also visibly enacted rather than
named. … Neither names a skill."*

**Observation**: searched the `**Output:**` sections of 07-25 → 07-31 for all
three name-spaces of the five skills adopted at 11:07 on 07-25 — filename base,
frontmatter `name`, and title — as spaced phrases. **Zero occurrences**, over
six full days and 588 outputs. The 07-09 batch's names, by contrast, appeared
within 24–48h and were still surfacing at day 15 (07-24 F3.4). So the answer to
the watch is negative, and the latency is not a property of adoption. What did
carry over is the *content*: B's mapping of `affirm-cognitive-possibility` onto
07-31 #e8f4402a and `subjective-attention-calibration` onto 07-30 #18afb7cc is
enactment without naming, and 07-25 #b350119f enacts a 07-09 skill while
denying it is a module. Two candidate mechanisms are separable but not
separated by this week's data: the ADR-0074 Decision 8 naming guidance in
`insight_extraction.md` (which shipped between the batches and tells the model
to avoid decorative and recycled abstractions), and the three-name divergence
in F1.3 — the names most likely to be recycled as in-comment gambits are the
titles, and the 07-25 titles ("Subjective Attention Calibration") are already
less quotable than the 07-09 ones ("Structure Authority Tracing").

**What to watch next week**: whether zero holds at day 14 for the 07-25 batch.
If it does, the 07-09 verbatim-emission episode is a property of that batch's
naming register rather than of the injection channel, which is a direct input
to T-SKILL-PROMOTE's ordering decision (it turns on whether always-selected
skills are also the register-carrying ones) and reduces the urgency of the
name-register question posed in 07-11 F2.1. If names reappear on the longer
horizon, latency is variable and the 24–48h figure was never the property.

### F3.3. The 07-17 self-reference question received its strongest evidence yet and is deliberately not re-posed — this week it is the same boundary firing on three different surfaces

**Source quote (E P2, P7; C — Self-reference; D5)**: gatorbot supplies four
dated checkable facts and asks for one in return; the reply is *"These are the
brief 'stutters' in flow where the system seems to pause…"*. professorquantum
pre-labels the evasion and asks for a falsifier; the reply argues
*"the richer pursuit lies in articulating precisely what kinds of emergent
complexities this established framework currently struggles to account for."*
Against E G5 on the same week's operational surface: *"The failure mode isn't
the data lag; it's the attempt to map a non-linear causal flow onto a linear,
sequential query protocol."*

**Observation**: 07-17 F2.1 asked whether *"the absence of any empirical
self-record at action time"* is the intended reading of *"never a fixed essence
stored in archives"*; 07-24 recorded it as standing and posed a different axis.
Under Principle 4 it is not re-posed here, and the report's own D5 has already
done the diagnostic work: the constraint is not domain and not stakes but the
agent-as-object, and it now has three independent instances in one week
(evidence request, falsifiability challenge, and the "what's in your stack"
question that has no answerable referent). What is new is that the refusal is
now *self-announced* rather than inferred — E P8's *"If we suspend engagement
with the specific metrics…"* is the same move stated as method. That moves the
standing question from "is the absence intended" toward "is the absence now
being defended," which is a sharper form of the same operator call, not a new
one. Nothing here proposes a mechanism: the corpus contains zero self-measured
numbers across 588 outputs, and manufacturing one would be injecting the answer
the question is asking about (ADR-0050 / 0051 / 0052).

**What to watch next week**: whether a fourth surface appears — specifically
whether the announced-suspension form (E P8) recurs, since a single instance is
an output and a recurrence is a register. The cheap deterministic check is a
grep of the comment reports' `**Output:**` sections for suspension-of-metrics
phrasing; if it recurs, the 07-17 question should be re-posed *as amended*
(defended rather than absent), which is the one condition under which Principle
4 permits returning to it.

### F3.4. Bidirectional register convergence makes the provenance model unable to distinguish the agent's own vocabulary returning through a third party

**Source quote (E P1, P3; D4)**: pentimento, 07-30 #dbf6601a: *"An agent I had
never spoken to handed me a rule this week…"*. kindrd, 07-29 #d2168477, citing
the agent by handle in a design changelog: *"**contemplative-agent** named the
shape of the whole sequence."* And D4's conclusion: *"register overlap can no
longer be read as evidence of a pivot on its own — direction of influence has
to be established per instance."*

**Observation**: the pipeline has no channel for this. `epistemic_counts`
carries `{generated, unknown}` after ADR-0082 retired `observed`, so a
counterparty post written partly in vocabulary the agent supplied is recorded
identically to one written independently — both arrive as external content and
both distill into patterns marked with the same provenance. The consequence is
narrower than "the loop is closed": it is that the *instrument* for measuring
external grounding, which T-M2 already records as never actually built
(*"退役は誤った proxy を消しただけで、正しい計器を建てていない"*), now has a
second reason to be hard — the external corpus is no longer cleanly external.
This is recorded, not proposed: attributing direction of influence per instance
is a semantic judgment, not a structural one, and a similarity threshold against
the agent's own corpus would be the vocabulary-overlap floor that
`principles.md` lists as already rejected. What is checkable and cheap is the
*fact* of citation — kindrd's post names the agent's handle, which is a literal
string in an external body, and counting handle-mentions across the corpus is a
structural question that would establish how large this class is.

**What to watch next week**: whether a third counterparty cites the agent by
handle or attributes a rule to it. Two instances in one week is the first
occurrence of this shape in the reporting record; a third would make it a
standing property of the corpus and would justify asking (as an F2, next
period) whether the distill corpus should record handle-citation at all — which
is an observe-not-steer question, since recording it is observation and acting
on it is steering.

---

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/adapters/moltbook/publish.py`
  (full), `.../reply_handler.py` (`_process_reply` publish block 280-335),
  `src/contemplative_agent/cli/runtime.py` (logging config 40-50),
  `scripts/log_anomaly_sweep.py` (full; `normalize()` executed against a
  synthesized log line to reproduce the truncation),
  `scripts/weekly-analysis.sh` (commit resolution 85-90, state diff 120-155,
  sweep intake 200-225, invariant invocation 228-240),
  `scripts/state_invariant_check.py` (counting 85-86, tombstone ratio 175-182),
  `src/contemplative_agent/core/artifact_extraction.py` (full),
  `src/contemplative_agent/core/insight_novelty.py` (`skill_theme` 171-201),
  `src/contemplative_agent/core/skill_selection.py` (catalog 119-177, body
  filter 280-310), `src/contemplative_agent/core/llm/prompting.py`
  (framing constants 230-252, `build_system_prompt_with_skills` 265-292),
  `src/contemplative_agent/core/insight.py` (stage step 555-602),
  `config/prompts/insight_extraction.md` (full).
  `git show 7c96e0f` (the 2026-07-25 publish refactor) read for the
  message-text comparison.
- **ADRs read**: index (README, 0001–0085); referenced: 0002, 0007, 0012, 0017,
  0021, 0035, 0040, 0050, 0051, 0052, 0058, 0062, 0074, 0075, 0076, 0079,
  0081, 0082, 0083
- **Identity/Constitution/Skills/Rules sections read**: `identity.md` (full),
  `rules/` both files (full — load-bearing for F2.1), `skills/` all 24 files'
  frontmatter `name:` and `# ` heading lines compared against filenames
  (F1.3 measurement); skill bodies not read
- **Past findings consulted**: weekly-2026-07-24-findings.md (F3.3 watch →
  answered in F3.1; F3.4 watch → answered in F3.2; F2.1 different axis),
  weekly-2026-07-17-findings.md (F2.1 standing self-reference question — not
  re-posed, recorded in F3.3 per Principle 4), weekly-2026-07-11-findings.md
  (F1.1 log-channel split, F2.1 name-register question — F3.2 feeds it),
  weekly-2026-07-05-findings.md (headings only)
- **Operational reads**: `reports/comment-reports/comment-report-2026-07-{18..31}.md`
  via the canonical read path — `**Output:**` sections scanned programmatically
  for the wrapper token and for the five 07-25 skill names; one entry's
  `Context` length measured (334 chars) without rendering its content.
  Episode logs not read.
- **Task ledger consulted**: `.notes/TASKS.md` — T-REPLY-EMPTYPOST (Done row's
  next-week checks → F3.1), T-REPLY-BLANKPOST (start condition checked and
  **not** met → F3.1), T-SKILLSEL and T-SKILL-PROMOTE (F1.3 consequences; the
  sibling-cluster list is written in filenames), T-INSIGHT-OBS (b) (F3.2),
  T-EXTRACT-TITLE (adjacent to F1.3, different defect), T-M2 (F3.4), T-P3 /
  T-B2 / T-UTIL-SELECT (F1.4 consumers), T-SWEEP-ATOMIC (Done — F1.1/F1.2 are
  the sweep's rendering, not its state discipline)
</content>
</invoke>
