# Weekly Diagnosis — 2026-10-30

**Source report**: weekly-2026-10-30.md
**Diagnosis date**: 2026-10-31

## F1. Structural (code / schema / pipeline diff)

### F1.1. The census cannot see a single row of the distill ledger it is registered to read

**Source (O-NNN / Exceptions)**: Exceptions 4 (census `distill-processed.jsonl` NO_ROWS, 407 rows not counted no-ts).
**Code reference**: `src/contemplative_agent/core/distill_ledger.py:112` (writer), `scripts/_census_registry.py:366`
(reader)
**Structural change**: make the ledger row and the census agree on a timestamp field.
- The writer appends `{"episode_id": eid, "processed_at": stamp}` (`distill_ledger.py:112`), and `append_jsonl_restricted`
  adds `run_id` / `session_id` but no `ts` (row shape: `logs/distill-processed.jsonl:1`).
- The census reader takes the window date only from `rec.get("ts")` and counts every other row as `rows_without_ts`
  (`_census_registry.py:366-369`).
- Two diffs close the gap; choosing between them is the implementer's call:
  - (a) the ledger row also carries `ts` = the same `stamp`. The ledger's only in-code reader,
    `read_processed_ids` (`distill_ledger.py:59-80`), reads `episode_id` alone, so an added key is inert there.
  - (b) `Entry` gains a per-log timestamp-field name (default `"ts"`), and the `distill-processed.jsonl` entry
    (`_census_registry.py:221`) names `processed_at`.
- A regression test that feeds one real-shaped ledger row through `_read_entry` and expects 1 row in window pins
  either one.
**Why this is structural, not symptomatic**:
- The ADR-0060 amendment names the weekly census as this file's reader (`docs/adr/0060-…md:247-248`), and the registry
  comment defines the reading as "the count per session is the reading (yield of the daily distill)"
  (`_census_registry.py:218-220`).
- As written, that reader returns NO_ROWS every week by construction, whatever distill does. Last week's watch item
  (weekly-2026-10-24-findings F3.2 (a): "the file exists with one row per processed episode") is met on disk (407 rows,
  4 runs) and invisible to the instrument.
- This week the instrument stayed blind through a two-day distill outage (Exceptions 3). The outage was found only by
  reading the raw ledger, which is the reading the registry entry exists to provide.
**Related ADR**: ADR-0060 (amendment 2026-10-24), ADR-0107, ADR-0110
**Filed**: T-CENSUS-DISTILL-LEDGER-TS

## F2. Identity-level open questions

None this week. The comment-report readings all fall on already-ledgered entries (O-001, O-002, O-004, O-005, O-007, O-013,
O-019), each of which has a linked proposal or an unexpired condition; no new value-layer question arose.

## F3. Pure observations

### F3.1. Episodes from 2026-10-25T21Z through 2026-10-26 are outside every remaining distill window

**Source (O-NNN / Exceptions)**: Exceptions 2, 3; O-020 (changed).
**Observation**:
- `distill-processed.jsonl` holds 0 episode ids dated 2026-10-26 and 1 dated 2026-10-25T16..23, against 85 for
  2026-10-27.
- `distill.episode` calls were 0 in the 10-26 and 10-27 UTC files. The 2026-10-28T23:47:22Z run recorded 173 episodes,
  starting at 2026-10-28T03Z ids.
- Knowledge rows held at 12,903 across the 10-25 export and the three stopped sync runs.
- The same days' sync runs stopped on a 10-26 snapshot manifest that carried the home path
  (`sync-data-launchd.log:14073-14087`).
- The read window is the last `max(days, 2)` day-files (`docs/adr/0060-…md:232`), so no scheduled run will reach the
  10-26 file. ADR-0060's "a flapping backend loses nothing" holds only within that window.
- Whether to backfill undistilled episodes is the open decision D-P0-04-1
  (`docs/plans/roadmap-2026-10/decisions.md:228`, due at the 11-21 gate), and the miss count it needs is P0-04.2's.
  This gap is a post-repair instance of the same question, so it is routed there, not filed. The plist-loss shape
  itself already has a terminal entry by name (`.notes/archive/tasks/T-PLIST-LOSS.md`).
**What to watch next week**:
- (a) whether rows with `episode_id` 2026-10-26 appear in the ledger, from a manual `--file` run or a backfill;
- (b) whether every daily sync commits (no RFC-0051 stop) and `distill.episode` calls are non-zero on every UTC day;
- (c) the O-020 growth reading without the 10-28 catch-up run inside it.

### F3.2. The weekly submolt sweep's retirement reads as NO_ROWS, not as a declared cadence

**Source (O-NNN / Exceptions)**: Exceptions 5, 13.
**Observation**:
- `submolt-scope-*.jsonl` is NO_ROWS (last ts 2026-10-21T18:13:17Z), and API drift lists `GET /submolts` as a baseline
  endpoint not observed.
- RFC-0011 records the 2026-10-24 decision to move the sweep to a quarterly manual run (`rfcs/0011-…md:121-123`). The
  registry entry carries no `cadence_days` (`scripts/_census_registry.py:224-225`), so the census cannot render the
  empty window as QUIET.
- The census header assigns registry edits to the Saturday gate (materials:328). This is a gate edit, not a code
  defect, and is not filed.
**What to watch next week**: whether the status stays NO_ROWS. That is expected until the registry entry changes or
the quarterly sweep runs.

### F3.3. Three distill pattern texts entered the anomaly sweep as new signatures

**Source (O-NNN / Exceptions)**: Exceptions 10.
**Observation**:
- The three 🆕 rows with origin `—` are lines of raw model JSON that `distill` echoed into `distill-launchd.log`
  (`:48182`, `:56579`, and the block at `:56596-56644`).
- They entered the signal set through words inside the generated text, not through a log level. The sweep cuts "Added
  pattern" lines at their payload boundary (materials:293), but these JSON lines carry no such prefix.
- The previous report's Exceptions do not list this shape, so this is the first window with it in the reports read.
**What to watch next week**:
- If raw-JSON signatures recur, the sweep's normalization of distill's JSON echo becomes an F1 candidate (read
  `scripts/log_anomaly_sweep.py` and the distill print path first).
- If they do not recur, drop this item.

### F3.4. `parent_rejected` fell to 21 rows

**Source (O-NNN / Exceptions)**: Exceptions 6, 7; O-014 (changed).
**Observation**:
- 21 rows (53 last window), all on prompt `2f5352151363`, per day 5, 2, 5, 4, 1, 3, 1. The largest single-session
  repeat is ×3 (`3e129daf`), against ×12 last window.
- The window also contains two zero-distill days, but distill does not touch the comment-scan path, so no link is
  claimed.
- This is the fifth consecutive F3 on this signal with no state change. By Principle 4 nothing is proposed.
**What to watch next week**: row count and largest single-session burst.

### F3.5. Last window's watch items

**Source (O-NNN / Exceptions)**: weekly-2026-10-24-findings F3.1–F3.5.
**Observation**:
- F3.1 (relevance `reopen`): the reading is unchanged at diff 0.1174 (carried). ADR-0113's follow-up remains with the
  Saturday gate.
- F3.2 (`distill-processed.jsonl`):
  - (a) is met on disk (407 rows from the 2026-10-25T19:05:17Z run onward), but the census cannot read it (F1.1).
  - (b) is met: the 10-25 run recorded 2026-10-24T21Z and 22Z episode ids (ledger rows 81-87), the session the old
    window never read.
- F3.3 (growth tracks episodes): the P0-04.1 window and the 10-28 catch-up both raise this window's growth. The
  per-episode yield was not re-measured.
- F3.4 (`parent_rejected`): carried as F3.4.
- F3.5 sub-items: no `POST /posts` 403 in the window. The one `GET /home` 503 is the carried `:75980` row. No insight
  `new` verdict: the 10-30 run wrote 2 `revise`.
**What to watch next week**: the items above.

## Diagnosis Metadata

- **Codebase files read**:
  - `src/contemplative_agent/core/distill_ledger.py:1-120`;
  - `src/contemplative_agent/cli/memory_cmds.py:40-119`;
  - `scripts/_census_registry.py:195-239, 340-379` and the `Entry` field grep (`:58-97`);
  - `scripts/sync-research-data.sh:240-389`;
  - `config/sync-excludes-public.txt` (grep `.last_*`);
  - `tests/test_distill_ledger.py` (grep `episode_id` / `processed_at`);
  - `src/contemplative_agent/cli/schedule.py` (grep `submolt-scan`).
- **ADRs read**: `docs/adr/0060-per-episode-grounded-distill.md:220-250`; `docs/adr/README.md` (rows 0060 / 0107 / 0110)
- **Plans / ledger read**: `rfcs/0011-submolt-scope-sweep-reading.md` (full); `docs/plans/roadmap-2026-10/decisions.md:42, 228-229`;
  `docs/plans/roadmap-2026-10/ROADMAP.md:385-394` (P0-04.1 / .2)
- **Identity/Constitution/Skills/Rules sections read**: none (no F2)
- **Past findings consulted**: weekly-2026-10-16, weekly-2026-10-23, weekly-2026-10-24 (F headings; 10-24 F3.2–F3.5 in full)
- **Task ledger consulted**: names of every `rfcs/*.md` and `.notes/archive/tasks/*` (Glob); grep `distill-processed|processed_at`
  over `rfcs/`, `docs/`, `tests/` (no task hit); grep `replay-runs|plist|watchdog` over `rfcs/` (RFC-0011, RFC-0041, RFC-0046,
  RFC-0047, RFC-0050, RFC-0051, RFC-0053 — none covers the census reader)
- **Tasks filed**: T-CENSUS-DISTILL-LEDGER-TS

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-10-17 … 2026-10-30, 1403 judged of 1403 records. History: 9741 judged of 16372 records over 113 daily logs.
Catalog: 53 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 3, wordform 34. Charged to a catalog entry (semantic, wordform): 37.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- (none)

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 0 name(s)

Candidate file: `reports/analysis/weekly-2026-10-30-archive-candidates.txt` — 0 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
The file is empty this week; `--archive-names` rejects an empty file (exit 2), so it is not passed.
