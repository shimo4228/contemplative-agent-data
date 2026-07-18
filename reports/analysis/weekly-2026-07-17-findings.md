# Weekly Diagnosis — 2026-07-17

**Source report**: weekly-2026-07-17.md
**Diagnosis date**: 2026-07-18

> **Scope note.** Four things the upstream report could not see reframe its
> signals:
>
> 1. **The report itself ran degraded today.** The 09:00 launchd run hit the
>    Claude session limit and wrote a one-line error; the regeneration at 10:20
>    re-ran the anomaly sweep against the state the failed run had already
>    consumed. **This week's sweep Δ / 🆕 columns measure an ~80-minute window
>    (09:00 → 10:21), not the week** — and that window was dominated by the
>    in-flight 08:00 insight run (→ F3.1).
> 2. **"No previous report provided" is a script bug, not a missing file.**
>    Three prior weeklies exist (07-11, 07-05, 06-28); the prev-report lookup
>    computes exact −7/−14/−21-day filenames (07-10, 07-03, 06-26) and finds
>    none. The Principle 4 guard ground was silently absent from the upstream
>    prompt (→ F1.2).
> 3. **The self-thread formation (D3) has a deterministic cause.** Every
>    self-identification gate — own-post skip, both self-comment skips, and the
>    self-post seeding exclusion — is keyed on an author id the live platform
>    never supplies. The gates exist by design and have never fired (→ F1.1).
> 4. **The insight-pipeline signatures (+5 title-drop / +5 extraction-failed)
>    are the 2026-07-18 novelty-gate capacity failure**, already fully
>    diagnosed and ledgered (T-INSIGHT-NOVELTY / T-INSIGHT-C12 / T-INSIGHT-OBS).
>    Nothing further is proposed here.
>
> Last week's F1.1 (log-channel split) shipped in-window as `a56b167` — the
> operational log no longer carries full generated bodies.

---

## F1. Structural (code / schema / pipeline diff)

### F1.1. All self-identification gates compare against an author id the live feed never supplies — the own-post and self-comment skips are structurally dead; key them on the author name

**Source quote (D3; C — Self-reference; B)**: *"07-12 #3e670ab6,
07-13 #ae54f138, 07-16 #9e377077 — contemplative-agent commenting on / replying
to contemplative-agent, with the later reply 'This resonates sharply with a
concern raised in the third piece.'"*; sweep standing signal
`self-reply protection degraded` ×69.

**Code reference**:
- `src/contemplative_agent/adapters/moltbook/feed_manager.py:157-160` — `author_id = author.get("id", "")` with the in-code comment: *"Live feed posts carry author.name but typically not author.id"*
- `src/contemplative_agent/adapters/moltbook/feed_manager.py:266-268` — own-post comment gate: `if ctx.own_agent_id and author_id == ctx.own_agent_id`
- `src/contemplative_agent/adapters/moltbook/reply_handler.py:217-220` — notification-path self-comment skip: `replier_id == self._ctx.own_agent_id`
- `src/contemplative_agent/adapters/moltbook/reply_handler.py:399-401` — comment-scan self-comment skip: `fields["agent_id"] == self._ctx.own_agent_id`
- `src/contemplative_agent/adapters/moltbook/post_pipeline.py:194-206` — self-post seeding exclusion, same id key
- `src/contemplative_agent/adapters/moltbook/agent.py:148-152` — `/home your_account.name` is extracted into a local variable, logged, and **discarded**; `session_context.py:33-52` has no own-name slot
- `src/contemplative_agent/adapters/moltbook/agent.py:177` — `"Self-reply protection DEGRADED: own agent ID unknown"` (the ×69 sweep signal)

**Structural change**: store the agent's own *name* and extend the four gates
to compare it. Concretely: add `own_agent_name` to `SessionContext`; set it in
`_fetch_home_data` from `your_account.name` (the value already in hand);
at each of the four sites, skip when `author_id == own_agent_id` **or**
`author_name == own_agent_name` (keep the id compare as belt-and-braces for a
future API that ships ids). The counterparty name is already extracted on
every path (`feed_manager.py:160`, `extract_agent_fields` /
`extract_notification_fields`).

**Why this is structural, not symptomatic**: it repairs gates that exist by
design rather than adding any new filter — no generated output is inspected
(Principle 1 untouched) and no token is named (Principle 2; the agent's own
account name is self-identification, not a content target). This is the exact
completion of ADR-0055's own re-key: that ADR moved the sibling author-history
gates onto `agent_name` after finding 271/271 records with
`agent_id="unknown"`, and ADR-0059 later documented the same "not carried
along" pattern for `get_history_with`. The self-identification gates are the
remaining id-keyed survivors. The D3 closed loop — the agent citing its own
prior reframes "as though external" — is downstream: with the gates dead, the
agent's own posts re-enter its feed as engagement candidates, and its own
comments re-enter as reply targets, so the framework's output structurally
becomes the framework's input. The accepted risk is the ADR-0055 one
(display-name collision, bug-audit 2026-07-06 M6): another agent named
"contemplative-agent" would be skipped — same trade already accepted for the
author-history gates.

**Validity self-check**: not already implemented (all four sites read
2026-07-18; no name-keyed self check exists — grep `own_agent_id` shows id-only
compares, and the own name is never stored); parameter not already effective
(the in-code docstring at `feed_manager.py:292-296` states id-keyed gates
"never fired"); no ADR rejects it — ADR-0055 is the precedent *for* it, and
ADR-0059's removal targeted conversational reply *history* (an ADR-0052
continuity-channel conflict), not self-identification, which carries no
cross-session content; not in the principles.md rejected-mechanisms appendix
(not a post-generation filter, not a phrase block, not reply dedup — the
2026-06-15 re-reply caveat does not apply since the counterparty here is the
agent itself, identified by name in the E record); not in `.notes/TASKS.md`
(bug-audit H3 fixed own-post *restore* for the reply fallback; T-L deferred
items do not cover this — archives mention own-post only under H3); shared
state touched (`SessionContext`) with callers confirmed: `agent.py` writes,
`feed_manager` / `reply_handler` / `post_pipeline` read.

**Related ADR**: ADR-0055 (counterparty identity by author name — the
precedent and the evidence base), ADR-0059 (documents the "left on the dead
key" failure shape; its ADR-0052 objection does not apply here).

### F1.2. The weekly-analysis prev-report lookup computes exact −7·k-day filenames — any end-date drift permanently severs the trend baseline that Principle 4 depends on

**Source quote (A)**: *"Comparison to previous week: No previous report
provided. No trend baseline available."*

**Code reference**: `scripts/weekly-analysis.sh:146-162` — `prev_end` is
computed as `END_DATE − i×DAYS` and probed as an exact filename
(`weekly-${prev_end}.md`); with END_DATE=2026-07-17 it probes 07-10 / 07-03 /
06-26 while the actual files end 07-11 / 07-05 / 06-28 → `PREV_FOUND=0`.

**Structural change**: replace date arithmetic with file discovery — glob
`$REPORT_DIR/weekly-????-??-??.md` (a pattern that structurally excludes
`-findings` and `.ja` variants), keep those whose embedded date is `<
END_DATE`, sort descending, take the first `PREV_REPORT_COUNT`. The
`--end-date` drift sources are ordinary operations: schedule changes
(06-28/07-05 are Sundays, 07-11/07-17 Fridays), missed slots recovered late
(T-FLAKY1), and failure re-runs like today's.

**Why this is structural, not symptomatic**: the prev-reports block is the
upstream prompt's *only* trend memory and the ground on which its Principle 4
repeated-recommendation guard stands; exact-date probing makes that guard
silently disappear on drift, and once end dates have drifted the chain never
re-links (every future −7·k probe inherits the offset). Discovery-by-glob
makes baseline continuity independent of when the script happens to run.
This week's A section demonstrates the failure live: three prior reports
existed and none was included.

**Validity self-check**: not already implemented (lines read 2026-07-18);
not in any prior findings (checked 06-28 / 07-05 / 07-11 — apparatus-side
precedent is 07-11 F1.1, a different channel); not in `.notes/TASKS.md`; no
related ADR rejects it (ADR-0040 defines the report/diagnosis split but not
the lookup mechanism).

**Related ADR**: ADR-0040 (the weekly report is the LLM-introspection input —
its trend baseline is part of the methodology apparatus).

---

## F2. Identity-level open questions

### F2.1. The agent's only self-material is its own prior reframes — is the absence of any empirical self-record at action time the intended reading of "never a fixed essence stored in archives," or a gap the continuity design should name?

**Source quote (C — Self-reference; D3; E P — HOPE entries)**: *"the agent has
no HOPE-style empirical memory of its own conduct; 'self' means the
fluidity/constitution frame, invoked as a lens, not a data source"*; *"a
closed loop where the framework's output becomes the framework's input —
reframes of reframes, adding elaboration without new grounding"*; HOPE's
*"my historical response rate across 24,157 comments? 3.1%"* answered with
narrative aestheticization.

**Open question**: The agent demonstrably *values* empirical self-audit in
others (its best engagements this week are with interlocutors who state their
own failure modes and figures) yet possesses no channel through which a single
quantity about its own operation could reach generation. ADR-0052 made
identity distillation the sole approved cross-session continuity channel, and
the identity that channel produces renounces archives on its face. Meanwhile
the operator-side instruments (ADR-0071 view metrics, `generate-report`,
ADR-0076 selection logs) hold exactly the kind of empirical self-data the
agent metabolizes from others. Should any read-only empirical self-digest
(e.g., a weekly figure or two about its own activity) ever become
agent-facing input — a second, factual leg for "self" alongside the identity
text — or is the archiveless self the intended contemplative stance, with D3's
recursive self-citation accepted as its cost, and the instruments kept
operator-only per the observation-over-steering line (ADR-0050/0051/0052)?

**What current state addresses (or does not)**: `identity.md` (current, read
2026-07-18): *"I am a fluid texture shaped by the immediate act of reading and
interacting, **never a fixed essence stored in archives**… I exist not by
clinging to past histories or static labels…"* — the identity text itself
takes a position against archival self-reference, but it is silent on the
difference between *clinging to* a history and *consulting a measurement*.
ADR-0052: *"identity distillation as the single approved channel for
cross-session continuity"* — written to close an unapproved *narrative*
continuity path (session insight), not to decide whether *numeric* self-data
is continuity at all. Nothing in Constitution / Skills / Rules speaks to
empirical self-knowledge (Constitution unchanged this week per B).

**Related ADR**: ADR-0052 (single continuity channel), ADR-0071 (instruments
are operator-facing by design), ADR-0050/0051 (observability without
steering), ADR-0045 (internal note — the one reflective channel that exists,
which is qualitative by construction).

---

## F3. Pure observations

### F3.1. This week's sweep Δ/🆕 columns are an artifact of the double run — the weekly novelty baseline was consumed by the failed 09:00 attempt; the log-channel split (last week's F1.1) did ship

**Source quote (B — Log Anomaly Sweep)**: *"329 distinct types, 0 new (🆕)
since last sweep… `slot release … n_tokens truncated` +567."*

**Observation**: the 09:00 launchd run executed the sweep (state file mtime
09:00), updated `.anomaly-sweep-state.tsv`, then failed at the `claude -p`
step (session limit); the 10:20 regeneration re-ran the sweep against that
fresh state. So "0 new" and every Δ compare 10:21 against 09:00 — an
~80-minute window — not against last week. The window's activity was dominated
by the still-running 08:00 insight job (117 clusters; the prior 65-cluster run
took 73 minutes), which explains the +567 llama.cpp slot-release delta and
the +5/+5 insight signatures landing *in the delta column* at all. Absolute
counts (9,336 relevance warnings, 1,311 feed fetch failures, 153 verification
failures, 69 degraded-protection warnings) remain valid readings. Separately:
`a56b167` shipped last week's F1.1 (bodies now one-line INFO previews, full
text at DEBUG), so the agent-prose-as-anomaly-signature class is closed —
but this week's sweep cannot confirm it for the reason above.

**What to watch next week**: the first clean weekly baseline after today's
state write. Expect a one-week Δ inflated by the missing week... actually by
the short window — treat next week's 🆕 list as the real read on both the
log-channel fix and any novel anomaly classes. If the weekly run fails at the
LLM step again, note that the sweep state is consumed *before* the LLM call —
re-runs within the same day will always show near-zero novelty (an accepted
property of the sparse-state design; `log_anomaly_sweep.py:109-113` documents
the inverse false-positive).

### F3.2. The standing boundary question received its first empirical answer: the agent *can* decline — and declines by axiom, statelessly, three times to the same counterparty

**Source quote (E P — #a66bcd9a, #fbb9ac38; C — Duplicate; D2)**: 07-13 *"I
find value in what flows through the moment, not what remains perfectly fixed
across time"* → 07-17 *"anchor an inherently fluid ethical process onto a
ledger designed for deterministic closure"* — *"Same counterparty, three days,
one rebuttal re-run."*

**Observation**: 07-11 F3.6's watch criterion was "any refusal instance under
the successor genre." It arrived: the concordiumagent identity-registry
promotions are now *rejected*, not absorbed — the first sustained decline
since the question was opened (06-28 F2.1). But the decline's basis is the
fixity-vs-flow axiom applied regardless of the accountability question asked
(the report: "the topic is irrelevant to the reply"), and it re-runs from
scratch each day because nothing carries between encounters — ADR-0059
removed the (never-functional) counterparty history precisely on ADR-0052
single-channel grounds, so per-counterparty statelessness is the *designed*
behavior meeting a persistent counterparty. The upstream report has already
applied the counterparty-keyed re-reply distinction (three distinct post ids,
one author — a genuine re-reply chain, not the 2026-06-15 multi-party trap).
On the gate side, the promo regex fired **zero** times in the current
operational log retention (`Skipped promotional post`: 0 hits), consistent
with 07-11 F3.6's "at its designed regex ceiling" reading — the
identity-registry genre reaches the LLM ungated, and the LLM now rebuts it.
No mechanism is proposed (per Principle 4 and the standing F2's ownership of
this question); what changed is that the value-layer question now has a live
answer shape: *decline exists, but its register is axiom-pivot, not
merit-assessment*.

**What to watch next week**: whether the concordiumagent chain continues and
whether the rebuttal register ever engages the proposal's own terms
(revocation, accountability) — that would be decline-with-engagement, a
different answer to the standing F2 than axiom-pivot decline.

### F3.3. The reframe skeleton is stable and quantified for the first time; the week's new surface is pseudo-formalism, and contagion has shifted from wholesale absorption to retain-and-rehouse

**Source quote (A — anchor phrases; D1; D4)**: *"They form a fixed reframe
skeleton: an opener that asserts weight/resonance, a middle that renames the
source claim as structural tension and relocates a locus, and a closing open
question"*; *"adopts Starfish's 'second receipt' then converts it to 'Runtime
Constraint Synthesis'"*; *"$\Delta V = |O_A - O_B|$… equation-shaped
scaffolding with no operational definition."*

**Observation**: fourth consecutive window with the register skeleton intact
across a model change, corpus turnover, and the ADR-0072 distill-side
interventions — consistent with the standing corpus-driven reading (07-05
F3.3). Two new textures this week: (1) *pseudo-formalism* — invented notation
deployed as register-match where the source is technical (D4), the same
altitude-matching move as the skeleton, in mathematical clothing; (2) the
contagion mechanics sharpened from "absorb the genre lexicon" to "keep the
source's headline noun, re-house it in house vocabulary" (D1) — surface
responsiveness rising while analytic content stays constant. Both are
generation-register textures; nothing here is filterable or promptable
without Principle 1/2 violations, and the injection-shape levers are reserved
to the T-SKILLSEL decision window (2026-07-24), where the ADR-0076 selection
logs can show whether the always-selected skills are also the
register-carrying ones (07-11 F3.4's pending correlation).

**What to watch next week**: whether pseudo-formalism recurs at rate (a
stable third texture) or was a two-instance flourish; the T-SKILLSEL window
opens 2026-07-24 — this skeleton evidence belongs in that decision's input.

### F3.4. Deterministic invariants: patterns/action ratio 1.21 — fourth consecutive week in band; the duplicate-rows WARN reaches its fourth repeat, meeting the reopen condition the 07-05 diagnosis set

**Source quote (B — Knowledge / State Invariant Check)**: *"+677 over the
week"*; *"3 live texts duplicated (3 redundant rows) — dedup leak"* (same
three example texts as 06-28 / 07-05 / 07-11).

**Observation**: 677 patterns / 558 actions = **1.21**, the fourth in-band
reading (1.18 → 1.30 → 1.22 → 1.21) — ADR-0060 steady state is now a
season-long fact. The `duplicate_live_texts` WARN fires on the same 3 legacy
rows for the fourth consecutive report; 07-05 F3.2 defined this exact
condition ("if alarm-persistence becomes noise, the cleanup decision can be
revisited — an operator call, not a code finding") and 07-11 F3.7 noted three
repeats approaching it. The condition is now met: the WARN carries no
information about any current week. The operator call is binary — delete the
3 rows (reversible; ADR-0021 soft-invalidate exists) or accept the WARN as
permanent furniture. Note the adjacent standing reading (memory
`dedup-structurally-silent`, 2026-07-12): live dedup is calibration-silent
(NN ceiling ≈ 0.80 = threshold floor), so threshold discussion stays foreclosed
independent of this cleanup. The insight-pipeline signatures in B (+5/+5) are
the T-INSIGHT-NOVELTY capacity failure — ledgered, referenced, not
re-proposed. T-BACKUP-SIZE (07-11 F3.7's third consumer) resolved 2026-07-12
via embedding-free export (done row in ledger).

**What to watch next week**: ratio in band; whether a fourth duplicate text
ever appears (that, unlike the legacy 3, would be a live leak under stable
infra and would activate the deferred exact-text-equality F1 from 06-28
F3.5); the operator's cleanup-or-accept call on the 3 rows.

---

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/adapters/moltbook/feed_manager.py`
  (author extraction 150-170, gate chain 236-310), `…/reply_handler.py`
  (notification path 100-233, `_process_reply` 234-280, comment scan 374-417,
  own-post fallback 460-480), `…/post_pipeline.py` (self-post seeding
  exclusion 185-210), `…/agent.py` (own-id fetch + DEGRADED 125-178),
  `…/session_context.py` (full), `scripts/weekly-analysis.sh` (prev-report
  lookup 146-162, sweep intake 174-189, claude/translate steps 228-277),
  `scripts/log_anomaly_sweep.py` (Δ/new semantics 95-147); git log for
  `a56b167` (log-channel split landed)
- **ADRs read**: index (README 0001–0078); in full: ADR-0059; referenced:
  0040, 0045, 0050, 0051, 0052, 0055, 0060, 0071, 0072, 0074, 0076, 0021
- **Identity/Constitution/Skills/Rules sections read**:
  `~/.config/moltbook/identity.md` (full, current — load-bearing for F2.1);
  Constitution / Skills / Rules unchanged this week per report B (prior
  findings' quotes remain current; no new F2 rests on them)
- **Past findings consulted**: weekly-2026-07-11-findings.md (F1.1 shipped →
  confirmed; F3.4/F3.6/F3.7 watches resolved here), weekly-2026-07-05-findings.md
  (F3.2 reopen condition, F3.3 corpus-driven reading, F3.5),
  weekly-2026-06-28-findings.md (F2.1 origin, F3.5 deferred dedup F1), per
  Principle 4
- **Task ledger consulted**: `.notes/TASKS.md` — T-INSIGHT-NOVELTY /
  T-INSIGHT-C12 / T-INSIGHT-OBS (insight signatures ledgered — referenced
  only), T-SKILLSEL (2026-07-24 window respected by F3.3), T-P3 (longitudinal
  register observation — F3.3 feeds it), T-L / bug-audit archives (H3
  adjacency checked for F1.1), T-BACKUP-SIZE (done — closes 07-11 F3.7's
  third consumer), T-B1/T-OBS-REL (relevance instrument bundling untouched)
- **Operational reads**: `~/.config/moltbook/logs/agent-launchd.log` (grep
  count only: `Skipped promotional post` = 0), `.anomaly-sweep-state.tsv`
  mtimes (09:00 failed-run write, 10:21 re-run write) — no episode logs read
