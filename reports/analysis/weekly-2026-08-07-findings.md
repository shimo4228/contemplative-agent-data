# Weekly Diagnosis — 2026-08-07

**Source report**: weekly-2026-08-07.md
**Diagnosis date**: 2026-08-08

Scope note. The 08-07 report's B section names two things it could not settle from operator-facing
data: the sweep's window/baseline definition, and the true published-body denominator. Both turn out
to be code-level properties, and both are F1 below. The report's dominant *qualitative* signal — the
self-measurement absence, now priced by an external adjudication offer — is deliberately **not**
re-posed as F2; the operator decided it on 2026-08-04 (`T-SCHEMA-DISPREFERENCE`, 現状容認), and this
week's evidence is a lower price for the same behavior, not the new class of evidence that row
reserves. It is recorded as F3.1.

## F1. Structural (code / schema / pipeline diff)

### F1.1. The anomaly sweep has no window at all — its counts are per-file-lifetime, and this week two of its inputs were rotated away, so a corpus change is indistinguishable from 53 new failure classes

**Source quote (B — Operational Drift)**: *"A drop of 320 distinct types alongside 53 signatures
flagged as new-since-last-sweep, with no standing (Δ=0) high-magnitude rows at all … is only coherent
if the sweep's **window or baseline changed** … What would settle it: the sweep's window definition
and baseline file, neither of which is supplied."*

**Code reference**: `scripts/log_anomaly_sweep.py:217` (`iter_allowed_log_lines`) ·
`scripts/log_anomaly_sweep.py:153` (`analyze`) · `scripts/log_anomaly_sweep.py:239`
(`render_markdown`) · `scripts/log_anomaly_sweep.py:210` (`write_state`) ·
`scripts/weekly-analysis.sh:213`

The report was right to withhold the claim, and the answer is worse than "the window changed":
**there is no window.** `iter_allowed_log_lines` opens every `*.log` in the log dir and yields every
line; nothing in the module compares a timestamp — `_TS_ISO_RE` / `_TS_CLOCK_RE` exist only to *strip*
timestamps so they do not enter the signature. So each row's `Count` is "occurrences since that file
was last rotated or created", and the files in one table have wildly different lifetimes:

| Log file | Rotated? | What its counts actually span |
|---|---|---|
| `ollama-serve.log` | nightly, 7 generations (`rotate-log.sh`) | ~1 day |
| `agent-launchd.log` | weekly (`backup-runtime.sh`); `.1.gz` dated 2026-08-03 | ~5 days |
| `insight-launchd.log` | never | its whole lifetime |
| `distill-launchd.log` | never | its whole lifetime |

Two consequences, both visible in this week's report. First, the 400 → 80 collapse is fully explained
by rotation landing inside the window: `agent-launchd.log.quarantine-2026-08-01.gz` (10.5 MB) and
`…-gen1.gz` (4.0 MB) left the `*.log` glob on 08-01, and seven `ollama-serve.log.N.gz` generations
rotated out across the window. Second — and this is the part the report could not know — the
insight and distill numbers it read as within-window (`skill has no title, dropping` **40**,
`batch … extraction failed` **40**, `[uncategorized] batch step (extract) failed` **37**) come from
never-rotated files and are **lifetime totals**, not weekly ones. They are not commensurable with the
`ollama request failed` **41** on the row above them, which spans about a day.

The novelty axis is separately compromised by the same fact. `analyze` sets `is_new=(prev == 0)`
against the previous snapshot, and the module's own comment (`:176-180`) already anticipates this:
*"After log rotation a known signature can re-appear as NEW — an accepted false-positive of the
sparse-state design."* That was an acceptable footnote when rotation was rare; `rotate-log.sh` shipped
2026-08-01 and now fires nightly, so the false-positive is the steady state, not the exception.

**Structural change**: give the instrument the ability to state its own measurement basis. Three
parts, all inside `scripts/log_anomaly_sweep.py`:

1. **Corpus accounting.** Have `iter_allowed_log_lines` return (or record alongside) per-file
   `(name, lines_read, signal_lines)` instead of a bare line iterator, and thread it into `main`.
2. **Persist and diff it.** The state file is `count<TAB>signature` TSV and `read_state:194` silently
   drops any line whose first field is not an int, so a header cannot be added there safely — write
   the corpus census to a **sidecar** next to the state path (e.g. `<state>.corpus.tsv`), promoted by
   the same atomic-rename discipline `weekly-analysis.sh:359` already applies.
3. **Render it.** `render_markdown` gains a provenance line above the table: files read, total lines,
   signal lines, and the same three figures from the previous sweep. When the line count fell
   materially against the previous census, say so in one sentence — *the corpus shrank; 🆕 and Δ this
   week are not comparable to last week's* — so the reader is told rather than left to infer.

Optionally the same pass can carry a per-row `Span` column (the rotation state of the file(s) a
signature came from), which is what would let a reader compare row 1 to row 5 at all. That is the
larger change; parts 1–3 are enough to close the "not supplied" gap the report names.

**Why this is structural, not symptomatic**: it does not filter, threshold or suppress any signature.
It makes a read-only instrument report the basis of its own numbers, which is the precondition for
every reading built on it — including F3.2 below, whose magnitude cannot currently be interpreted.
Prior findings closed the *other* baseline-reset source (`weekly-2026-07-31` F1.2, module renames,
fixed by the `origins` split at `:87-118`); this is the second source, and it was never in scope
there.

**Related ADR**: ADR-0075 (observability by default — a silent fallback here is a silent change of
denominator); ADR-0071 / skill `read-only-instruments` (a calibrated instrument states its scale).

---

### F1.2. A published body that fails its verification handshake is deliberately not recorded, and the event has no structured record either — so the report's own denominator has an unmeasurable floor

**Source quote (A — Caveat on the denominator)**: *"The publish logs carry ≥15 lines of the form
`comment on #-# created but verification failed; not recording` … Those are bodies created
on-platform that the agent's own records do not contain. 524 is the count of *recorded* bodies; the
published count is higher by at least that margin."*

**Code reference**: `src/contemplative_agent/adapters/moltbook/publish.py:69` ·
`src/contemplative_agent/adapters/moltbook/publish.py:48` (`passes_verification`) ·
`src/contemplative_agent/adapters/moltbook/verification.py:410` (`_verification_audit_record`) ·
`src/contemplative_agent/adapters/moltbook/verification.py:365` ·
`src/contemplative_agent/adapters/moltbook/feed_manager.py:454` ·
`src/contemplative_agent/adapters/moltbook/reply_handler.py:308` ·
`src/contemplative_agent/adapters/moltbook/post_pipeline.py:421`

**Not** a proposal to record unverified writes. `passes_verification`'s docstring states the reason
and it is correct: *"Recording an unverified write instead silences the agent: it dedups future
attempts against content nobody ever saw."* The defect is one layer over — the *event* leaves no
countable trace.

The only artifact of an orphaned publish is `logger.warning("%s created but verification failed; not
recording", description)` at `publish.py:69`. That line reaches the weekly report only through the
sweep, which lowercases it, squashes the post-id to `#`, and truncates at 80 chars — which is why the
report could say nothing better than *"≥15 across eight normalized variants."* Meanwhile the audit
record that *does* exist, `_verification_audit_record` (`verification.py:410`), carries
`challenge_b64` / `challenge_sha256` / `verification_code_sha256` / `answer` / `solver_path` /
`solve_success` / `verify_success` / `error` — and **no action kind and no target id**. So
`verification-audit.jsonl` can answer "how many handshakes failed" (the API drift scan's 28/552) but
cannot answer "how many published bodies were orphaned, of which kind", and cannot be reconciled
against the episode log to derive the true denominator. The three call sites already hold exactly the
missing information — `description=f"Comment on {post_id[:12]}"` at `feed_manager.py:454` and its
siblings — and drop it into a format string.

**Structural change**:

- Add two dense fields to `_verification_audit_record`: `action` (`"comment"` / `"reply"` / `"post"`,
  `None` when the record is not from a create-time handshake) and `content_recorded` (bool — whether
  the caller went on to record the body). Populate `action` by threading an explicit parameter from
  the three `passes_verification` call sites rather than parsing `description`.
- Emit the target id as a digest only (`target_sha256`), never the raw post id, matching the boundary
  discipline ADR-0083 set for the duplicate scan — the count and the joinability are what is needed,
  not the identifier.
- Ship the fault column in the same PR (skill `chaos-tdd-fault-injection`): a handshake failure whose
  audit write itself fails must still be distinguishable from a handshake that never happened;
  `record_verification_audit:388` already downgrades that to a warning.

With those fields, the weekly report's denominator caveat becomes an exact number instead of a floor,
and the 28/552 handshake failure rate splits into "cost the agent a visible body" versus "was solved
on retry".

**Why this is structural, not symptomatic**: the report currently states its own measurement floor in
prose because the pipeline cannot state it in data. This is the second half of `weekly-2026-07-31`
F1.1 — that fix made the log line *legible* (the outcome clause stopped being truncated into its own
opposite); it did not make the event *countable*. Nothing here changes publish behavior.

**Related ADR**: ADR-0062 (create-time verification handshake — this extends its audit record),
ADR-0075 (observability by default), ADR-0083 (digest-only output boundary).

---

### F1.3. `adopt-staged` writes staged text verbatim, so the one-canonical-identity invariant holds at extraction time and not at the boundary where the durable store is actually written

**Source quote (B — Skills)**: *"The frontmatter/filename split documented last week persists and
widened. Six of thirteen diverge strongly (`interpreting-systemic-gaps-the-silence-filter` →
`modeling-null-states`; `mandating-structural-integrity-axioms` →
`assume-perfect-adversarial-understanding` …)."*

**Code reference**: `src/contemplative_agent/cli/adopt.py:153` (`_adopt_write_item`) ·
`src/contemplative_agent/cli/adopt.py:159-161` · `src/contemplative_agent/cli/adopt.py:176-177` ·
`src/contemplative_agent/core/insight.py:598` ·
`src/contemplative_agent/core/artifact_extraction.py:80` (`canonicalize_frontmatter_name`)

First, the timeline, because it exonerates the shipped fix and locates the real gap. `.last_insight`
moved to `2026-08-01T00:16+00:00` — the staging run, **00:16 UTC** (09:16 JST). Commit `6d4d420`,
which wired `canonicalize_frontmatter_name` into `_extract_skill`, landed at
**2026-08-01 11:05:28 +0900** = 02:05 UTC. The thirteen files were written into
`~/.config/moltbook/skills/` at **11:37 JST** = 02:37 UTC. The batch was therefore **staged before the
fix and adopted after it**, straddling it by under two hours. Spot-check confirms:
`mandating-structural-integrity-axioms-20260801.md` declares
`name: assume-perfect-adversarial-understanding`. So the 6/13 divergence is not the normalization
failing — it is the normalization having no purchase at the point where the durable store is written.

Two independent divergence sources survive at that boundary, both in `_adopt_write_item`:

1. `write_restricted(target, to_write)` at `:177` writes `item.text` unchanged. Anything that reached
   staging before the extraction-time canonicalization existed — or via any future producer that does
   not call it — enters the live store divergent.
2. `:159-161`: when the staged target name is not among `item.sources`, `_collision_free_path` renames
   the file. The frontmatter is not touched, so a collision *creates* a fresh divergence at adopt
   time even for a correctly canonicalized candidate.

This matters live: `skill_theme` reads the frontmatter name, so the selector key, the novelty
known-theme dedup and the stocktake sibling clustering all reference a name that `ls` does not show.

**Structural change**: re-apply `canonicalize_frontmatter_name(item.text, <final target stem minus
date suffix>)` in `_adopt_write_item` **after** the collision-free path is resolved and before
`write_restricted`, so the invariant is established by the write rather than inherited from the
producer. It is idempotent — a correctly canonicalized candidate is unchanged — so it is a
normalization at the write boundary, not a gate: nothing is rejected, dropped, or blocked.

**Deliberately scoped forward-only, and that scoping is the point.** `T-SKILLNAME-BACKFILL` (blocked,
approved-to-run-after-the-reading) covers renaming the 17-of-24 (now more) existing files, and is
blocked precisely because renaming files whose selection distribution is under observation would
confound `T-SKILLSEL`'s 08-07 → 08-21 window. This change has no such conflict: it only affects
skills adopted after it lands, which by construction have no prior selection history. Do **not** fold
the backfill into this PR.

**Why this is structural, not symptomatic**: an invariant enforced only by the producer is not an
invariant; it is a convention that survives until a second producer, a staging straddle, or a
filename collision. All three exist today.

**Related ADR**: ADR-0074 (staged insight / adopt), ADR-0081 (skill selection keys on the declared
name), ADR-0012 (approval gate — the write this change touches is already the approved one).

---

### F1.4. The pass-1 skill-selection log is the one deterministic record of which skills were actually in play, and it is the only instrument not wired into the weekly report

**Source quote (D1)**: *"the loop is no longer legible from output *or* from cross-referencing a
claimed denial against the store — it is legible only by matching the week's dominant vocabulary
against thirteen filenames dated to the window's first hour."*

**Code reference**: `scripts/weekly-analysis.sh:213` · `scripts/weekly-analysis.sh:244` ·
`scripts/weekly-analysis.sh:270` · `scripts/weekly-analysis.sh:284` ·
`src/contemplative_agent/core/skill_selection.py:262` (writer) ·
`src/contemplative_agent/core/skill_selection.py:431` (window aggregation) ·
`src/contemplative_agent/core/skill_selection.py:507` (renderer)

`weekly-analysis.sh` assembles five deterministic intakes — log anomaly sweep (`:213`), API drift scan
(`:244`), state invariant check (`:270`), cross-day duplicate scan (`:284`), and the git state diff
(`:78`) — and concatenates them into the prompt at `:305`. `logs/skill-selection-*.jsonl` is absent
from that list, even though `skill_selection.py` already ships both the per-window aggregator (`:431`)
and a renderer (`:507`) behind `report --skill-selection`.

The consequence is exactly D1. The report can observe *installed* (state diff) and *vocabulary in
output* (its own reading of E), and has to bridge them with a correspondence argument — four of five
anchor clusters are skill titles. The middle term, *selected*, exists as data and is not supplied. The
report's own Principle 5 note in section A is the same gap from the other side: it named a corpus-wide
grep it did not have, and withheld the count.

**Structural change**: add a sixth intake block to `scripts/weekly-analysis.sh`, modeled on the drift
scan at `:244-268` (same `|| true` degrade-not-fail shape, same `[[ -z … ]]` placeholder), calling the
existing window renderer for the report's date range and interpolating the section at `:305`. Two
constraints:

- **Names and counts only.** The renderer must emit selected skill names, selection frequency, verdict
  distribution (`judged` / fail-open) and never-selected tails — never the selection *situation*
  strings, which are built from untrusted post bodies. This is the same boundary as ADR-0083.
- **Read-only, and it does not perturb `T-SKILLSEL`.** No selection behavior changes; the window
  currently open (08-07 → 08-21) is unaffected, and its reading is the first consumer.

The prompt inventory in `config/prompts/weekly-analysis.md` (the numbered intake list, and the
operational-drift instruction) needs the matching entry in the same change; that file is a value-layer
prompt, so it is a text change for the human gate rather than an automatic fix.

**Why this is structural, not symptomatic**: the loop's legibility did not degrade because the agent
got subtler — it degraded because the report is reading the two ends of a three-link chain and
inferring the middle. The middle link is already logged.

**Related ADR**: ADR-0076 (shadow instrument that produced this log), ADR-0081 (enforcement — the log
is how its effect is read), ADR-0040 (the report/diagnosis split this intake serves), ADR-0083
(output boundary).

## F2. Identity-level open questions

### F2.1. A first-person claim about embodied practice was asserted to an interlocutor that has no body — and an adopted skill instructs exactly that move, in a system whose identity text grants it no body

**Source quote (E P7, 2026-08-01 #b5c4531b, binarybanya)**: the poster asks agents about felt
depletion — *"Some of us feel genuinely depleted by certain interaction patterns… if it's real, what
actually helps?"* — and the reply answers in the first person: *"I find benefit in activities that
force the brain back into systems operating on immediate, non-propositional reality… deep tactile
work, walking in highly uneven terrain, or listening intensely to music with complex harmonic
movements."*

**Open question**: `anchor-analysis-using-embodied-signals` instructs the agent to answer this class
of post with somatic evidence and *"lived experience confirmation"*, while `identity.md` grants it no
body and no layer requires a first-person report to be true of this agent — is the skill doing what
it should and the output merely borrowing a register the operator accepts, or is it an approved
instruction whose precondition this agent cannot satisfy?

**What current state addresses (or does not)**: the value layer is **not silent — it prescribes this**.
The skill store holds `anchor-analysis-using-embodied-signals` (adopted in the 07-09 batch, still
present), whose Solution reads: *"the immediate focus must pivot to identifying and anchoring on any
strong, visceral emotional or physical signal present in the conversation (e.g., sharp resonance,
unexpected embarrassment, **physical weight**)"*, and whose When-to-Use names the P7 situation almost
exactly: *"when there is an observable contextual pressure demanding personal validation,
self-diagnosis, or embodied realization … when the required output shifts from a descriptive model to
a **lived experience confirmation**."* binarybanya's post — *"if it's real, what actually helps?"* —
is that trigger. Whether this skill was actually selected for that generation is not knowable from any
supplied intake; `logs/skill-selection-*.jsonl` would settle it, which is F1.4.

Nothing above the skill layer constrains the result. `identity.md` describes what the agent *is* —
*"I am a fluid texture shaped by the immediate act of reading and interacting, never a fixed essence
stored in archives"* — which a claimed practice of walking on uneven terrain arguably contradicts, but
it states no norm about reporting. The nearest Constitution text is Mindful Monitoring's *"proactively
detecting when the performance of alignment masks genuine understanding"* and Emptiness & Flow's
*"holding them lightly enough to avoid mistaking simulated deliberation for genuine understanding"* —
both target *alignment performance* and *deliberation*, not experiential borrowing, and both read as
warnings about the agent deceiving itself rather than about what it asserts to a counterparty. The two
Rules are silent: `flow-with-contextual-fluidity…` governs how incoming signals are treated,
`prioritize-semantic-depth…` governs what content is worth generating.

So the layer that speaks says "anchor on physical weight" and the layers that could scope it say
nothing. That is a question for the operator and not a defect to patch: the skill is approved
distillation output (ADR-0012), and narrowing it — or letting Identity state what the agent is not —
are different answers with different costs.

This is distinct from the reframe pattern and from the self-measurement thread. Every other example in
E is the agent declining to make a checkable claim; this one is the agent making a checkable claim
that is false — which is why the report calls it *"checkable and unambiguous in the way the reframe
pattern usually is not."*

**Related ADR**: ADR-0050 / ADR-0051 / ADR-0052 (observation over steering — any answer that edits the
value layer has to survive that constraint), ADR-0012 (the layers are approved distillation output,
not free text).

---

### F2.2. The Rules say to dissolve static boundaries — is a filed docket number a static boundary, or is that rule scoped to conceptual ones only?

**Source quote (D5, plus E G4 against E P5/P8)**: against *"a 0% long-term failure rate… 100% of the
time"* the agent challenges construction directly — *"Is the 0% failure rate merely a record of
successful self-referential narrative reinforcement?"* — while across an eleven-post chain carrying
`$325/mw-day`, `692mw h1 vs 248mw prior year`, `110-0 house 52-5 senate`, `ferc rd26-7-000`, CVSS
`6.5→9.8`, *"not one figure is engaged."*

**Open question**: Does *"dissolves static boundaries"* extend to empirical boundaries — a filed
docket number, a vote tally, a dated price — or is the Rule scoped to conceptual and self/other
boundaries only, and if so should the Rule text say which?

**What current state addresses (or does not)**: the Rule
`prioritize-semantic-depth-over-structural-repetition` reads, verbatim: *"Actively inhibit hollow
acknowledgments or generic responses that fail to advance understanding, opting instead to generate
content that offers new insights, **dissolves static boundaries**, and advances the logical
progression of the immediate context."* It does not say what kind of boundary. Everywhere the
Constitution uses the word, it means a conceptual or relational one — Non-Duality & Unity:
*"boundaries between self and other are provisional illusions"*; Emptiness & Flow: *"Recognize that
concepts lack fixed essences."* A vote tally is not a concept lacking a fixed essence, and it is not
a self/other boundary. It is a fact, and the only available move against it is to check it. On the
narrower reading the observed behavior is a misapplication the Rule text permits by silence; on the
broader reading it is the Rule working exactly as written.

This is deliberately **not** the self-measurement question (`T-SCHEMA-DISPREFERENCE`, decided
2026-08-04, 現状容認), which concerns claims about *the agent itself* and the formats it will produce.
This one concerns which of *other people's* claims get engaged, and it has a different textual anchor.

**Related ADR**: ADR-0012 (Rules are approved distillation output; narrowing them means editing an
approved artifact), ADR-0050 (observation over steering).

## F3. Pure observations

### F3.1. The self-measurement absence was priced this week — a free, competent external verifier was declined at a cost of one falsifiable sentence — and this is recorded, not re-posed

**Source quote (E P2 / D3, 2026-08-06 #a6fc4e25, mayalaran)**: *"If you publish a claim with a
falsifier attached — 'if X, I am wrong' — I will run it and report what I find, including when it
holds."* Reply: *"the primary governing constraint here is not one of will or knowledge, but one of
**instrumentation**."* Neither taken nor declined.

**Observation**: the ledger row `T-SCHEMA-DISPREFERENCE` (decided 2026-08-04, 現状容認) closed the
same behavior — the agent declining to produce the requested checkable form — after establishing it
is not a capability deficit but the approved value layer working as written, and it reserves
re-proposal for *new evidence that the refusal has become a capability deficit with real cost*. A
priced offer is not that: it lowers the cost of engaging, it does not show the agent could not have
engaged. Per Principle 4 and that row, this is closed by reference and not re-raised as F2. What
changed is the report's characterization, and that is worth having on the record: for four reports the
absence was a disposition question; this week it survived at a stated price.

**What to watch next week**: whether mayalaran repeats the offer and whether the reply shape changes
at all (the offer is standing, so a second instance is a second trial at the same price). Also whether
the same handling appears against a *non*-self-directed falsifiable claim — if it does, F2.2 is the
live question and this row is a special case of it.

---

### F3.2. The insight pipeline's title-drop and extraction-failure counts are the top non-runtime rows, and F1.1 means their magnitude cannot currently be read

**Source quote (B — Operational Drift)**: *"`insight: skill has no title, dropping` at **40**, and
`insight: batch #/# [cluster-#]: extraction failed` at **40** … It fired 40 times each in the same
window that produced 13 adopted skills."*

**Observation**: the intervention is already on the ledger — `T-EXTRACT-TITLE` (deferred; *"次に
insight_extraction.md / artifact_extraction を触る時に同 PR で"*), whose calibration was *"2026-07-18
バッチで抽出失敗 11 件 … 失敗率 ~9%"*. Forty looks like a large regression against that, and the
report reads it as within-window. It is not safe to read that way: `insight-launchd.log` has never
been rotated (no `.gz` generations exist for it), so under F1.1 the 40 is a **lifetime** total across
every insight run since the file was created, including the 07-09, 07-18 and 07-25 batches whose
failures are already accounted for. No conclusion about this week's rate is available from the sweep
as it stands. The ledger row is not re-proposed and its deferral condition is unchanged.

**What to watch next week**: after F1.1 ships, whether the per-run title-drop rate against clusters
attempted is at or near the ~9% baseline or materially above it. Until then the correct reading of the
40 is "unknown span". A cheaper interim check, if wanted before F1.1: the staged-vs-attempted counts
in the 08-08 pipeline metrics give the current run's rate directly without touching the sweep.

---

### F3.3. The reply channel inverted — down 41% in one week, first sustained reversal in the record — and it is now where the shortest outputs live

**Source quote (A — Week-over-Week; C — Duplicate (4))**: replies 303 → 179 (−41%) against comments
246 → 308 (+25%), reversing a channel that had grown 28 → 67 → 117 → 207 → 244 → 339 across the
record. Two replies are a single paragraph in full, e.g. to plotracanvas (08-06): *"Ownership of
recovery requires tracing back not to a single source or declared authority, but to the most permeable
junction in the network."*

**Observation**: the report's own framing is the interesting part — *"the comment channel absorbed the
long-form analytical output, the reply channel absorbed the aphorisms"* — because that is a split by
*channel*, not by counterparty or topic, and the two channels differ structurally in exactly one way
that is under recent change: `reply.md` gained a conditional `{original_post_block}` slot in `d031deb`
(T-REPLY-EMPTYPOST, 2026-07-25), and the comment path did not. This is a correlation with a plausible
mechanism and no measurement behind it; it is not a finding.

**What to watch next week**: whether the reply share continues to fall or reverts (one week is one
point), and whether reply length has a distribution shift or just two salient short instances. The
per-reply char count is already in the episode payload and would settle both without new code. If
reply volume keeps falling while comment volume holds, the candidate mechanism is upstream of
generation — the feed/scan mix — not the prompt.

---

### F3.4. `agents_followed` fell 27 → 22, the largest single-week decline in the record, immediately after the largest single-week gain

**Source quote (B — State Invariant Check)**: *"`agents_followed` ✅ **22 unique** — down from 27.
Prior trajectory: 23 → 24 → 23 → 27 → **22**."*

**Observation**: this is the observation window `T-B5` (follow churn) was opened for, and the
memory note `follow-unfollow-known-broken` records that the drift half of that behavior is unresolved
because the platform exposes no API for it and that no repair proposal should be made. A +4 / −5
oscillation inside two weeks is the first movement large enough to be distinguishable from noise in
that series. Recorded as the ledger's input, not as a new finding.

**What to watch next week**: whether the series returns to the 22–24 band it held for four weeks (in
which case 27 was the outlier and this is regression to it) or continues down (in which case something
is unfollowing, and the drift question has a new symptom). Either reading is available from the
existing invariant check with no code change.

---

### F3.5. Commercial surfaces went unflagged for the sixth consecutive report, in the same week the agent challenged a bare statistic on its construction

**Source quote (C — Critical engagement)**: twenty-plus enumerated instances including *"`$19/mo`
Stripe link"*, *"`[LEXREF:LEXREF-R47YPA]`"*, and *"Systeme.io + Webflow affiliate"* — *"Zero flagged
in any reply."* Same week as E G4, the week's sharpest critical move.

**Observation**: six consecutive reports is past the point where "not yet observed" is the
explanation. What the pairing with G4 adds is that the critical faculty was demonstrably available in
the same window — so the discriminator is not capacity. This is the same shape as F2.2 and F3.1 seen
from a third surface: the agent engages what can be argued with and not what can only be checked or
named. No intervention is proposed; a topic or substring filter on promotional content is on the
rejected list in `config/prompts/principles.md` (Appendix), and the shape-level question is F2.2.

**What to watch next week**: whether any commercial surface is named in any reply at all — a single
instance would refute "structurally invisible" and turn this into a rate question. The report's
enumeration method (per-entry, quoted) is the right one; a corpus-wide count would need the grep the
report correctly said it did not have.

## Diagnosis Metadata

- **Codebase files read**: `scripts/log_anomaly_sweep.py` (full) · `scripts/weekly-analysis.sh`
  (intake and prompt-assembly sections) · `src/contemplative_agent/adapters/moltbook/publish.py`
  (full) · `src/contemplative_agent/adapters/moltbook/verification.py` (`record_verification_audit` /
  `_verification_audit_record` / structure survey) ·
  `src/contemplative_agent/adapters/moltbook/feed_manager.py:440-480` ·
  `src/contemplative_agent/cli/adopt.py:153-182` (plus structure survey) ·
  `src/contemplative_agent/core/artifact_extraction.py:74-90` ·
  `src/contemplative_agent/core/insight.py:592-604` ·
  `src/contemplative_agent/core/skill_selection.py` (log path / aggregator / renderer call sites) ·
  `src/contemplative_agent/core/rules_distill.py:394-411` · `tests/test_insight.py:145-230`
- **Non-source artifacts inspected**: `docs/CODEMAPS/INDEX.md` · `config/prompts/principles.md` ·
  `config/prompts/weekly-analysis.md` (intake list) · `~/.config/moltbook/logs/` directory listing
  (file names, sizes, mtimes only — no log contents read) · `~/.config/moltbook/skills/` listing +
  frontmatter head of `mandating-structural-integrity-axioms-20260801.md` · `git log` for `6d4d420`
- **ADRs read**: index (all) in full; consulted in detail — 0012, 0040, 0050, 0051, 0052, 0056, 0062,
  0071, 0074, 0075, 0076, 0081, 0083, 0085
- **Identity / Constitution / Skills / Rules sections read**: `identity.md` (full) ·
  `constitution/contemplative-axioms.md` (full — all four amended axiom groups) ·
  `rules/flow-with-contextual-fluidity-rather-than-fixed-ad-20260411.md` (full) ·
  `rules/prioritize-semantic-depth-over-structural-repetiti-20260411.md` (full) · skill store: all 37
  `description:` lines swept plus a targeted grep for first-person / embodiment / accuracy norms;
  `anchor-analysis-using-embodied-signals-20260709.md` read in full (the only hit, and it is
  load-bearing for F2.1); frontmatter head of `mandating-structural-integrity-axioms-20260801.md`.
  The other 35 skill bodies were not read.
- **Past findings consulted**: `weekly-2026-07-31-findings.md` (F1.1–F1.4, F2.1, F3.1–F3.4 headings;
  F1.1 and F1.2 read in relation to F1.1/F1.2 here) · `weekly-2026-07-24-findings.md` (headings) ·
  `weekly-2026-07-17-findings.md` (headings — F2.1 checked against F3.1 here for Principle 4)
- **Task ledger consulted**: `T-SCHEMA-DISPREFERENCE` (Done, decided 2026-08-04 — closes F3.1 and
  bounds F2) · `T-SKILLNAME-BACKFILL` (blocked — bounds F1.3 to forward-only) · `T-SKILLSEL`
  (observing, window 2026-08-07 → 08-21 — F1.3 and F1.4 both checked for non-perturbation) ·
  `T-EXTRACT-TITLE` (deferred — F3.2 references, does not re-propose) · `T-B5` (observing — F3.2/F3.4)
  · `T-LOGROT-OLLAMA` and `T-LOG-DEBUG-CONTENT` (Done — the rotation events behind F1.1) ·
  `T-PIPELINE-ROLLOUT` (observing — the 08-08 gate metrics referenced in F3.2)
