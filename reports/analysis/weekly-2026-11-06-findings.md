# Weekly Diagnosis — 2026-11-06

**Source report**: weekly-2026-11-06.md
**Diagnosis date**: 2026-11-07

## F1. Structural (code / schema / pipeline diff)

### F1.1. Consecutive weekly State Diffs do not meet: commits made on the window's first day are in no diff

**Source (O-NNN / Exceptions)**: Exceptions 2 (State Diff coverage gap), with Exceptions 1 (two `rules/` files lost
their approved-row match while the window's `rules/` diff reads "No changes").
**Code reference**: `scripts/weekly-analysis.sh:186`
**Structural change**: The start snapshot should be taken the way the previous window's end snapshot was taken.
That is the last data-repo commit on or before `(START_DATE − 1 day) T23:59:59`, the same `--before` rule as the
end commit at `:191` applied to the previous day. Then window N's start commit equals window N−1's end commit,
and the half-open approval join (`start < ts <= end`, `scripts/value_layer_approval_join.py:127-132`) tiles
without a hole. The `:187-189` fallback to the root commit is unchanged. Add a test that runs two consecutive
default windows and asserts that start(N) == end(N−1). ADR-0099 owns the state-diff section and gets a one-line
amendment.
**Why this is structural, not symptomatic**:
- The defaults set `END_DATE` = yesterday (`:83`) and `START_DATE` = END − 6 (`:94`), so consecutive Saturday runs
  have adjacent, non-overlapping date windows.
- But the start commit is the last commit on or before the *end* of `START_DATE` (`:186`), while the previous end
  commit is the last commit on or before the end of the previous `END_DATE` (`:191`). Every commit dated on
  `START_DATE` is therefore inside neither diff.
- This week the hole runs from `b3b061e` (2026-10-30T05:30 JST) to `60adf4d` (2026-10-31T23:10 JST, its own
  changes included, because the diff and the approval join both exclude the start commit). The approval-join
  windows printed in the two materials leave 2026-10-29T20:30:14Z..2026-10-31T14:10:34Z unread.
- The Saturday gate, the one place value-layer changes are approved, runs on `START_DATE`.
- The only signal that still reaches the report is the whole-log live-text reconciliation. That is the
  0 → 2 rise on `rules/` this week, which arrives with no diff to attribute it to.
**Related ADR**: ADR-0099 (state-diff section of the weekly materials); ADR-0012 / ADR-0050 (approval provenance join)
**Filed**: T-STATE-DIFF-START-DAY-GAP

## F2. Identity-level open questions

None this week. Exceptions 1–13 are instrument readings, and the only behavioural material (O-004's `test` entry)
is a ledgered observation with a linked proposal (Principle 4).

## F3. Pure observations

### F3.1. Two `rules/` files changed bytes without an approval row and without a diff

**Source (O-NNN / Exceptions)**: Exceptions 1
**Observation**:
- Last reading: both live rules files matched the 2026-04-11T10:36:52Z approved hashes `0743f0887fcfdead` /
  `85ba74660869a761` (`audit.jsonl:60-61`; `weekly-2026-10-30-090004/materials.md:246`).
- This reading: they hash to `a039abe1217a8490` / `e6686955a88a5446`.
- Both files now open with frontmatter (`origin: rules-distill`, `adopted: 2026-04-11`).
- The window's `rules/` diff is empty. If F1.1's hole is where the change sits, the change was committed between
  2026-10-30T05:30 and 2026-10-31T23:10 JST.
- The unattended session cannot read the data repo's history to confirm this.
**What to watch next week**:
- Confirm: `git -C <data repo> log --format='%h %cI' -- rules/` shows a commit inside the F1.1 hole. If the diff of
  that commit is only the frontmatter block, the cause is an out-of-band provenance edit (one of the four causes the
  reconciliation line names), and the count holds at 2.
- Refute: the count rises above 2, or the bytes differ in the Practice / Rationale text.

### F3.2. The O-014 prompt now arrives in bursts inside one session

**Source (O-NNN / Exceptions)**: Ledger O-014 (archived this week into `rfcs/0049`), Exceptions 3
**Observation**:
- 20 generations of `prompt_norm_sha256 2f5352151363` this window, one per `parent_rejected` row per day.
- 7 of them fall in `3af748b2` within 11 minutes, each to a distinct parent comment
  (`skill-selection-2026-11-04.jsonl:65-81`), and 6 more in `495490b0`.
- That fits RFC-0049's explanation (a): distinct comments carrying one byte-identical body. The (a)/(b) decision is
  recorded at the 2026-11-07 gate.
**What to watch next week**: if the gate records (a), a per-session burst is the expected shape and no further
reading is needed. If any `reply_key_sha256` in `parent_rejected` rows appears twice, (b) is back on the table.

### F3.3. A pair of 403s with an HTML body on comment publish

**Source (O-NNN / Exceptions)**: Exceptions 4
**Observation**: Two `POST /posts/{id}/comments` 403s fell 14 minutes apart in one session (`4fce7a06`,
`api-audit.jsonl:83873, 83903`). The sweep shows an HTML document as the error body (new signature). Earlier 403s
in the log come one per session.
**What to watch next week**: Confirm the one-off reading if no 403 recurs. Refute it if 403s with an HTML body recur
or cluster. In that case, compare with the API Drift Scan and the `rate_remaining` column at those rows.

## Diagnosis Metadata

- **Codebase files read**: `scripts/weekly-analysis.sh:57-94, 150-210`; `scripts/value_layer_approval_join.py:122-137`;
  `src/contemplative_agent/adapters/moltbook/feed_manager.py:675-709`
- **ADRs read**: none in full; `docs/adr/` grep for "State Diff" → ADR-0099
- **Identity/Constitution/Skills/Rules sections read**: `rules/flow-with-contextual-fluidity-rather-than-fixed-ad-20260411.md`,
  `rules/prioritize-semantic-depth-over-structural-repetiti-20260411.md` (full)
- **Past findings consulted**: weekly-2026-10-30-findings.md (headings)
- **Task ledger consulted**: rfcs/ (Glob of all entries; RFC-0038, RFC-0049 full; grep for parent_rejected / start
  commit); `.notes/archive/tasks/` (Glob)
- **Tasks filed**: T-STATE-DIFF-START-DAY-GAP

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-10-24 … 2026-11-06, 1350 judged of 1350 records. History: 10393 judged of 17024 records over 120 daily logs.
Catalog: 53 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 1, wordform 26. Charged to a catalog entry (semantic, wordform): 27.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- (none)

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 0 name(s)

Candidate file: `reports/analysis/weekly-2026-11-06-archive-candidates.txt` — 0 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
The file is empty this week; `--archive-names` rejects an empty file (exit 2), so it is not passed.
