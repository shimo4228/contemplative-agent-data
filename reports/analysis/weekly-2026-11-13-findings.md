# Weekly Diagnosis — 2026-11-13

**Source report**: weekly-2026-11-13.md
**Diagnosis date**: 2026-11-14

## F1. Structural (code / schema / pipeline diff)

None this week. Each candidate below was checked against the code and the ledger and stays an observation. The
reasons are given under each F3:
- a cap on selection size contradicts the selector's stated design (F3.4);
- the reopen verdict and the census registry gap already have named consumers (F3.1, F3.3);
- the regenerated comment follows the publish path's designed `unverified` exit (F3.5).

## F2. Identity-level open questions

None this week. The identity replacement (Ledger O-003) is the gate's approved act of 2026-11-07. The two readings
that moved after it (F3.1, F3.2) are one window old. Under minimum persistence they are observations before they
are questions about the text.

## F3. Pure observations

### F3.1. The relevance gate rate fell 20 points across the identity adopt, and the after-window spans R34

**Source (O-NNN / Exceptions)**: Exceptions 1
**Observation**:
- Adopt-window reading: post-level gate rate 0.5244 (204/389) before the 2026-11-07T02:30:22Z identity adopt, and
  0.3212 (229/713) after. Diff −0.2032, CI [−0.2626, −0.1424], wholly outside ±0.06, so `=> reopen`
  (materials:823).
- By the instrument's consumption plan, `reopen` opens the 150-labelled-post re-read, and `hold` / `indeterminate`
  change nothing (`scripts/relevance_shadow_reading.py:91-97`).
- One fact bears on reading the verdict: the 7-day after-window contains the RFC-0075 S54 boundary
  (2026-11-11T15:00Z, R34). After it, every feed candidate is scored before any comment (`rfcs/0075:75`), so the
  set of posts reaching the scorer changed inside the after-window.
- The rate is computed per post from the latest answered row, so a change in which posts get scored can move it
  without any change to the scorer.
- This session cannot split the after-window at R34. The adopt_window JSON is a chain-stage output, and the reading
  has no `--until` split.
**What to watch next week**:
- The gate's re-read decision is the consumer; this finding adds only the R34 overlap.
- Confirm that the identity change carries the drop: the gate rate over 2026-11-07T02:30Z..11-11T15:00Z (before
  R34) is already near 0.32. `relevance_shadow_reading.py --start 2026-11-07 --end 2026-11-11 --since
  2026-11-07T02:30:22Z` reads this from the existing logs.
- Refute it: the pre-R34 slice sits near 0.52 and the drop sits after 11-11T15:00Z.

### F3.2. Question-form closers left their band in the first window after the identity adopt

**Source (O-NNN / Exceptions)**: Exceptions 2
**Observation**:
- 90/582 = 15.5% (CI 12.8-18.6) against the 6–12% band declared 2026-11-07. Last window was 8.7% (49/563).
- The rise is concentrated in self-posts: 16/41 = 39.0%. Example: 2 of the 6 self-posts of 2026-11-13 end on a
  question (materials:22097, 22125), and the other 4 end declaratively (materials:21976, 22009, 22039, 22067).
- Replies 18.9%, comments 5.3%.
- The adopted identity text opens *"I am fundamentally defined by a continuous state of structural questioning"*
  (materials:119). The previous text did not use "questioning" as a self-definition (materials:116-118).
- One window does not separate this from the window's seeds and counterparties.
**What to watch next week**:
- A second window above 12% opens a ledger observation on this metric (minimum persistence). In that case, compare
  the per-kind split: whether the self-post share alone carries it.
- Back inside 6–12%: this was a one-window reading.

### F3.3. A live identity-drift writer is not in the census registry

**Source (O-NNN / Exceptions)**: Exceptions 3
**Observation**:
- `logs/identity-drift.jsonl` is written by `core/identity_drift.py` through `cli/memory_cmds.py:165` at each
  identity staging. Its first row is at 2026-11-07T00:07:53Z, the same second as the staged audit row.
- `scripts/_census_registry.py` has no entry for it, so the census reads it as UNKNOWN with 0 rows in the window,
  although the row's `ts` falls on 2026-11-07.
- The census line itself assigns register / retire to the Saturday gate (materials:248).
- An identity adopt happens about once a month (the previous one was 2026-10-10), so the log's cadence is
  event-driven, not weekly.
**What to watch next week**:
- After the gate registers the file, the census should show it as QUIET in a week with no identity staging. A
  plain registration would show NO_ROWS instead, and NO_ROWS is an exception class.

### F3.4. Skill selection occasionally lists most of the catalog and hits its output cap

**Source (O-NNN / Exceptions)**: Exceptions 4
**Observation**:
- 3 selection calls ran to `num_predict` 400 and returned 39 / 38 / 38 names of 53
  (`skill-selection-2026-11-10.jsonl:160`, `-11-11.jsonl:86, 138`).
- Each output's last name was cut mid-token, and the rejection path recorded it as a hallucination. These are 3 of
  the window's 22 emissions, including the only `semantic` one.
- The comment generation that followed each call ran with its `num_predict` clamped to 4507–4830 by the context
  guard (`core/llm/__init__.py:915-940`). It finished normally (`eval_count` 367–552), so no published body was
  truncated.
- Across the whole log, `selected_count ≥ 30` rows number 8 since 2026-10-08, about one per week. 3 of them fall in
  this window.
- Why no F1:
  - The selector's docstring states that no numeric cap is applied to the selection size (`core/skill_selection.py:228`).
  - Capping or rejecting large selections is the numeric-cap enforcement that Principle 2 excludes.
  - The hallucination rate stays inside its declared 0–10% band with or without these 3 rows (22/674 vs 19/674).
- The cap's sizing comment assumes a catalog of ~19 names (`core/skill_selection.py:86-88`). The catalog has had 53
  entries since at least 2026-11-07 (materials:738).
**What to watch next week**:
- The weekly count of `selected_count ≥ 30` rows. Two consecutive windows of 3 or more would mark a move from the
  ~1/week rate.
- Whether the skill-selection reading's rejected-name list keeps carrying cut-off names. Those names can be told
  apart only by joining `done_reason=length` in `llm-calls`; the reading does not join it today.

### F3.5. A comment regenerated after two unverified publishes

**Source (O-NNN / Exceptions)**: Exceptions 6
**Observation**:
- On 2026-11-09 in `31666a29`, the same comment prompt was generated at 21:03, 21:15 and 21:55Z
  (`llm-calls-2026-11-09.jsonl:575, 606, 692`).
- The first two publishes are `unverified` and the third is `published` (`skill-selection-2026-11-09.jsonl:133, 144,
  180`).
- This is the window's largest redundancy key (×3). It is a different shape from last window's O-014 burst
  (parent_rejected, ×7).
- The `unverified` exit is kept as is by RFC-0038 (`rfcs/0038-reply-parent-rejected-requeue.md:20`). The comment
  that fails verification is not recorded, and the post stays eligible.
- Two created-but-unverified comments therefore precede the published one on that post. Per the verification
  handshake, unverified content is not visible on the platform.
**What to watch next week**:
- Whether any post collects 3 or more `unverified` publishes in one session (the redundancy max).
- Whether the solver's "every solver path landed on a previously rejected answer; abstaining" line (🆕 ×2) rises
  with it.

### F3.6. Last week's report session recorded a permission denial

**Source (O-NNN / Exceptions)**: Exceptions 10
**Observation**:
- `weekly-pipeline-audit.jsonl:346` closes the 2026-11-07 chain with `reasons: REPORT_SESSION_DENIED`, result ok.
- The pipeline sets that reason when the session's denial count is above 0, and lets the structural gate judge the
  report (`scripts/weekly-pipeline.sh:653-659`).
- The previous report's coverage list includes reads outside the materials. Which call was denied cannot be read
  from this session: the audit row carries the reason, not the call.
**What to watch next week**:
- Whether this week's chain_end carries the same reason. This session's own out-of-materials reads were all inside
  `logs/`, `scripts/`, `src/`, `rfcs/` and `reports/analysis/`.

### Follow-up on last week's F3

- F3.1 (rules bytes): the unmatched count holds at 2, the same set as the prior reading (materials:166-167). It is
  carried in Exceptions.
- F3.2 (O-014 bursts): the `2f5352151363` prompt does not appear among this window's redundancy keys
  (materials:348-356). There are 6 `parent_rejected` rows, against 20 last window.
- F3.3 (403 with HTML body): no 403 in the window's status distribution (200 / 201 / 400 / 404; materials:318).
  The one-off reading is confirmed.

## Diagnosis Metadata

- **Codebase files read**:
  - `src/contemplative_agent/core/skill_selection.py:80-99, 225-264`
  - `src/contemplative_agent/core/llm/__init__.py:895-940`
  - `src/contemplative_agent/adapters/moltbook/llm_functions.py:232-254`
  - `scripts/relevance_shadow_reading.py:50-108`
  - `scripts/weekly-pipeline.sh:653-659`
  - `scripts/_census_registry.py` (grep)
  - `src/` grep for `identity-drift` and `num_predict`
- **ADRs read**: `docs/decision-history/README.md` (index head); grep across `docs/decision-history/` for
  truncation / `selected_count` / `_SELECTION_NUM_PREDICT` (ADR-0087, ADR-0089, ADR-0097 hits)
- **Identity/Constitution/Skills/Rules sections read**: identity.md old and new text as quoted in the State Diff
  (materials:110-121); no live value-layer file opened
- **Past findings consulted**: weekly-2026-11-06-findings.md (F3.1–F3.3 in full)
- **Task ledger consulted**: rfcs/ (Glob of all entries; RFC-0075, RFC-0007 full; grep for unverified / reopen /
  identity-drift / truncation); `.notes/archive/tasks/` (Glob)
- **Tasks filed**: none

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-10-31 … 2026-11-13, 1326 judged of 1326 records. History: 11067 judged of 17698 records over 127 daily logs.
Catalog: 53 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 1, wordform 32. Charged to a catalog entry (semantic, wordform): 33.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- translating-temporal-gaps-into-structural-utility: named-into 1 (similarity 0.76..0.76), selected 1 in the window, offered 11067 times in the whole history; runs against testing-procedural-guardrails-with-counterfactuals (selected 16, offered 7128, weight 1, similarity 0.50); fewer selections: translating-temporal-gaps-into-structural-utility

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 0 name(s)

Candidate file: `reports/analysis/weekly-2026-11-13-archive-candidates.txt` — 1 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
