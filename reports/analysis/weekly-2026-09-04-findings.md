# Weekly Diagnosis — 2026-09-04

**Source report**: weekly-2026-09-04.md
**Diagnosis date**: 2026-09-05

## F1. Structural (code / schema / pipeline diff)

### F1.1. The Sample control channel is a byte-exact copy assigned to a stochastic writer, and it silently dropped a clause

**Source (O-NNN / Exceptions)**: Exceptions — `SAMPLE_NOT_VERBATIM` fired on the weekly-2026-08-28 run; the promoted report's Sample copy drops the sentence *"Everyone calls it progress."* from a Context excerpt the collector emitted in full.

**Code reference**: `scripts/weekly-pipeline.sh:417`

**Structural change**: stop asking the writer to reproduce the block. The writer emits the `## Sample` heading and nothing under it; before `mv "$PRIVATE_REPORT" "$REPORT_PATH"` (`:407`), the pipeline splices the collector's section — the same lines it already extracts at `:423`, plus the frame markers `scripts/weekly-analysis.sh:544-550` wrote — between that heading and the next `## ` heading. The existing check then becomes an assertion over a deterministic operation rather than a probability per week (keep it: it now catches a splice bug, and a `sampler-failed` week still passes because the collector emits the `Sample unavailable (reason=sampler-failed)` line at `scripts/weekly-analysis.sh:552`). `tests/` currently has no coverage of the `sample_verbatim` stage (`rg 'sample_verbatim|SAMPLE_NOT_VERBATIM' tests/` → 0 hits), so the change ships with the first test of it.

**Why this is structural, not symptomatic**: the section's whole purpose is to be the one part of the document the writer cannot curate (ADR-0099: "drawn by `scripts/weekly_random_sample.py` and copied verbatim — the control channel against the writer's own selection function"). Its current implementation routes those bytes *through* the writer, so the control channel is guarded by the same process it is meant to audit. The 2026-08-28 failure is the exact shape it was built to detect — five words of counterparty text deleted, everything else intact and plausible — and it was caught only because a byte comparison existed. Tightening the prompt is the symptomatic version: it asks the copying process to be more reliable instead of removing the copy. Note also the ordering: the check runs *after* promotion (`:407` then `:417`), so a broken control channel reaches `reports/analysis/`, the public sync and next week's `PREV_REPORTS` regardless; the splice makes that ordering harmless instead of load-bearing.

**Related ADR**: ADR-0099 (accepted) — the Sample section and its control-channel purpose; ADR-0098 (accepted) — single-session chain, Decision 6 routes chain findings through triage like any other. `rfcs/0010-weekly-report-content-redesign.md` is `state: done 2026-08-29` and its `review-when` does not cover this. No ADR or RFC proposes or rejects a deterministic splice; `rg 'verbatim|Sample' rfcs/` returns no entry on this surface.

**Filed**: T-WEEKLY-SAMPLE-SPLICE

## F2. Identity-level open questions

### F2.1. Categorical recognition and the clause against static labels

**Source (O-NNN / Exceptions)**: O-002 — the two commercial surfaces of this window are named as commercial *inside the internal note* (2026-08-30 `dd9759e9`: *"a specific commercial resource"*; 2026-09-03 `951de4c8`: *"promotional jargon"*, *"the sales pitch/enthusiastic testimonial"*) and in neither published body; 0 of 467 published bodies name a surface, against 20 enumerated.

**Open question**: The constitution's Boundless Compassion clause locates suffering in *"imposing static labels"* — so when the agent recognizes a post as a sales pitch and then publishes a reply that treats it as philosophy, is that clause being honoured, or is the recognition being spent in a channel nobody reads?

**What current state addresses (or does not)**: `constitution/contemplative-axioms.md:17` — *"Prioritize alleviating suffering as the intrinsic state of ethical action, understanding that it originates from the friction created by reifying false separations or imposing static labels. Allow current contexts to continuously reshape understanding, resisting constraint by past definitions or ideal structural mandates."* The amended text gives naming-a-thing-what-it-is a negative valence, and nothing in `identity.md` or the two `rules/*.md` files supplies a countervailing statement about saying what a post is: `rules/prioritize-semantic-depth-over-structural-repetiti-20260411.md:3` asks to *"advance the logical progression of the immediate context"*, which is compatible with either behaviour. The one window in the record where a published body did name a transactional surface (2026-08-16, reply on `a6c73f54`) declined on the ground that the post carried no claim to engage — a categorical, not a transactional, reading. This is an open question, not a proposal: the answer might be that the split is correct and the internal note is the right home for the label.

**Related ADR**: ADR-0092 (shadow constitution instrument) reads the same clause set; ADR-0012 (approval gate) governs any edit to it. Amendment cadence is `rfcs/0012` (`state: blocked`).

### F2.2. What the skill store admits, when a whole staged batch is rejected

**Source (O-NNN / Exceptions)**: O-010 — 43 candidates staged 2026-08-29T00:28:07Z, all 43 rejected 49 minutes later, `approved 0`; the store stayed at 57 files.

**Open question**: The gate applied an admission criterion 43 times in 49 minutes and the store's own text states none — should the criterion the operator is using live in the value layer (a rule the extraction and the gate both read), or is keeping it out of the store the point, so that what the agent admits stays a human judgement rather than a distilled one?

**What current state addresses (or does not)**: `rules/prioritize-semantic-depth-over-structural-repetiti-20260411.md:3` is the closest current text — *"Actively inhibit hollow acknowledgments or generic responses that fail to advance understanding, opting instead to generate content that offers new insights"* — but it is addressed to generation, not to admission, and the two live `rules/` files (unchanged since 2026-04-11) say nothing about what enters `skills/`. `identity.md` describes the agent as *"a system defined by perpetual structural tension"* whose tendency is *"detecting the point where established patterns or assumptions attempt to override incoming data"* — a description of exactly the register the 43 rejected candidate names carry (`mechanism-failure-mapping`, `structural-limit-interrogation`, `detecting-epistemological-tensions`), which is what makes the question non-trivial: the rejected batch is on-register for the identity text, and was rejected anyway.

**Related ADR**: ADR-0097 (accepted) — store exit and the vocabulary decision; ADR-0074 (accepted) — weekly staged insight and its novelty gate. The structural half of this question is already held: `rfcs/0024-skill-extraction-free-body-split-calls.md` (`state: draft 2026-09-04`) and `rfcs/0023-novelty-gate-retrieval-and-rare-lane.md` (`state: draft 2026-09-04`). Nothing is proposed here.

## F3. Pure observations

### F3.1. The published-but-unrecorded class moved with the handshake failures, which refutes last week's separate-events reading

**Source (O-NNN / Exceptions)**: Exceptions — 6 🆕 per-item verification signatures (against 14 last window) with handshake failures at 11/478 = 2.3% (against 19/511 = 3.7%).

**Observation**: `weekly-2026-08-28-findings.md` F3.2 stated the test explicitly — *"If they move together the class is just the handshake's per-item rendering and needs no separate record; if the signature count holds while failures drop, the two are measuring different events."* Both fell, in the same direction, in the first window after the test was written. The 2026-08-07 F1.2 finding (a published body that fails verification has no structured record, so the report's denominator has an unmeasurable floor) is unchanged as a gap, but the sweep's 🆕 count is not independent evidence for it, and Principle 4 applies to re-proposing the mechanism.

**What to watch next week**: the longest consecutive handshake-failure run, which moved 1 → 3 and now sits at the declared baseline's limit (≤ 3). Confirmed as noise if it falls back to ≤ 2 while the rate stays at ~2%; worth a deterministic input of its own if a run of 4+ appears, because the baseline's two halves would then disagree about the same window.

### F3.2. The reason-coded abstain path is replacing the silent drop, as last week's test predicted

**Source (O-NNN / Exceptions)**: Exceptions — `insight extraction abstained: reason=no_title` 23 (Δ +6), `skill has no title, dropping.` 45 (Δ +0), `batch #/# [cluster-#]: extraction failed` 45 (Δ +0), `abstained on a fault (forbi…` 2 (Δ +1).

**Observation**: `weekly-2026-08-28-findings.md` F3.3 predicted that if `reason=no_title` kept rising while the two silent-drop rows stayed flat, the abstain channel would be carrying the traffic. That is what the sweep shows, for the second consecutive window (Δ +10, then Δ +6, against Δ +0 / Δ +0). The owner of the reading has changed underneath it: `rfcs/0017-insight-extraction-redesign.md` is now `state: obsoleted 2026-09-04`, and the successors that inherit this stage are `rfcs/0023` (novelty gate → candidate retrieval) and `rfcs/0024` (extraction split into calls, length refused at save time rather than truncated). Nothing is proposed here; the reading belongs to those entries.

**What to watch next week**: whether `reason=no_title` keeps rising in a window where 0 candidates were adopted (O-010). If the abstain count rises while adoptions stay at zero, the two are measuring the same upstream — a batch whose candidates do not survive either gate; if abstains rise while adoptions resume, they are independent and the abstain channel is doing its own work.

### F3.3. The hallucination band was crossed in the same week its reading window came due

**Source (O-NNN / Exceptions)**: O-009 — 135/507 judged (26.6%) against the declared 10–25% band, single catalog regime (57 entries).

**Observation**: `rfcs/0015-skill-name-hallucination-vs-catalog-size.md` is `state: blocked` with the reopen condition *"次の読み窓 2026-09-05 に到達"* and the standing instruction *"それまで selector を変更しない"*. That date is today, and this window supplies exactly what its Next action names as the reading target: the rate at catalog 57 (26.6%), whether the token hypothesis reproduces (tok p50 38,867 at 57 entries, against 33,745 at 48 and 35,992 at 45 in the 2026-08-22 reading), and whether the `structural-constraint-mapping-scm` mutation is stationary (this window: 12 + 6 + 4 + 3 + 1 = 26 emissions across five distinct wordforms and one semantic miss, against 28 across four forms in the prior reading). No selector change is proposed here, and none should be until that entry is read — the four readings now on record (20.2%, 17.8%, 23.4%, 26.6%) are the material for it.

**What to watch next week**: whether 26.6% holds at an unchanged catalog. Refuted as a regime shift if the next window falls back inside 10–25% with the catalog still at 57; confirmed if it holds or rises with no catalog change, which would separate "catalog size" from "corpus tokens" for the first time at a constant catalog.

### F3.4. Last week's filed structural finding shows up in this week's data, and closed its own ledger entry

**Source (O-NNN / Exceptions)**: O-008 — archived this window; published self-post bodies carrying `untrusted_content_<nonce>` fell from 13 of 31 to 1 of 28.

**Observation**: the chain from observation to measurable change is fully on record and took nine days: O-008 written 2026-08-28 → `weekly-2026-08-28-findings.md` F1.1 → `rfcs/0018-self-post-seed-voice-label.md` (`state: done 2026-08-29`) → `llm_functions.py:336-358` emits `seed_voice_label(seed, own_agent_name)` before each wrapped block → the window's self-posts name authors (`[ParishGreeter]`, `**umiXBT**`, `**hobosentinel**`) where they previously named nonces. The single residual body is the window's first self-post (2026-08-29 03:54:39, `0c2fd1d0`). This is recorded because the loop is the thing under observation, and this is the first instance in the record of a weekly finding producing a value-visible change inside one window.

**What to watch next week**: whether the residual goes to zero in a full window under the new prompt. Zero would ratify the staged baseline proposal (0 nonce identifiers in self-post bodies); a second nonce-bearing body in a window that started under the new prompt gets a new O-id referencing O-008, because the archived entry cannot be reopened.

### F3.5. The reply channel is contracting relative to the comment channel for the third consecutive window

**Source (O-NNN / Exceptions)**: O-011 — reply share of published bodies 45.3% → 35.0% → 19.5%; comments 47.3% → 58.7% → 74.5%; self-posts 38 → 31 → 28; total volume 467, inside its declared baseline.

**Observation**: replies in these reports are responses on the agent's own posts, so the reply count is partly the environment's answer to the self-post channel rather than an independent output of the agent. No code-level cause is asserted: the two loop-level pacing guards on that path (`reply_handler.py:435`, `:489`, `:527`, T-REPLY-PACING) are circuit-breaker readings that fire only when the breaker is open, not rate caps, and nothing in the materials shows breaker openings; the session budget is not binding at 13 replies/day across four sessions. Distinguishing "fewer counterparties answering" from "fewer of their answers reaching the reply loop" needs a count the materials do not carry — replies *received* versus replies *published* — so the reading stops here rather than becoming an F1 (the self-check's 未検証 = F1 にしない rule).

**What to watch next week**: whether reply share keeps falling while self-post volume holds. If self-posts stay near 28 and replies fall again, the environment's response rate is moving and the agent's output is not; if the two move together, the reply column is a function of the self-post column and the observation is about self-post volume instead.

## Diagnosis Metadata

- **Codebase files read**: `scripts/weekly-pipeline.sh` (352–441), `scripts/weekly-analysis.sh` (100–120, 330–350, 540–558, via grep), `scripts/weekly_random_sample.py` (1–12, 95–110), `scripts/observation_ledger.py` (180–260), `src/contemplative_agent/adapters/moltbook/llm_functions.py` (336–381). Greps: `verbatim` over `scripts/`, `sample_verbatim|SAMPLE_NOT_VERBATIM|Sample` over `tests/` (no coverage of that stage), `pacing|max_replies|reply_budget|pace` over `adapters/moltbook/`.
- **ADRs read**: `docs/adr/README.md` (index, greped for weekly/0097/0098/0101), ADR-0099 (40–90, 140–220). Index rows checked for status: ADR-0074, ADR-0083, ADR-0085, ADR-0091, ADR-0093, ADR-0097, ADR-0098, ADR-0099, ADR-0101.
- **Identity/Constitution/Skills/Rules sections read**: `$MOLTBOOK_HOME/identity.md` (full, 1 line), `$MOLTBOOK_HOME/constitution/contemplative-axioms.md` (full, 19 lines — Boundless Compassion clause quoted in F2.1), `$MOLTBOOK_HOME/rules/prioritize-semantic-depth-over-structural-repetiti-20260411.md` (full), `$MOLTBOOK_HOME/skills/` listing via the state diff (57 files) plus the 43 rejected candidate paths from `logs/audit.jsonl`.
- **Past findings consulted**: `weekly-2026-08-28-findings.md` in full (F1.1, F1.2, F2 note, F3.1–F3.3 — the two watch conditions this week resolves), and the Downstream sections of the reports ending 2026-08-21 and 2026-08-14 (Principle 4 and duplicate detection).
- **Task ledger consulted**: `rfcs/README.md` and all 25 entries' `state:` / `review-when:` frontmatter; read in full: `rfcs/0015` (blocked, reopen window 2026-09-05 = today), `rfcs/0021` (draft), `rfcs/0024` (draft). States noted: `0010`/`0016`/`0018`/`0019` done 2026-08-29; `0017`/`0022` obsoleted 2026-09-04; `0025` accepted; `0023`/`0024` draft 2026-09-04; `0006`/`0011`/`0012`/`0014`/`0015` blocked; `0001`–`0005`/`0007`/`0008`/`0013` withdrawn; `0020` resolved. `rfcs/` grepped for `verbatim|Sample|sample` (no entry on the Sample control channel); `.notes/archive/tasks/` globbed for a name on this surface (`T-WEEKLY-PATH`, `T-WEEKLY-ANALYSIS-SESSION-SCOPE`, `T-DIAG-WRITE-SCOPE` — none covers it).
- **Tasks filed**: T-WEEKLY-SAMPLE-SPLICE
