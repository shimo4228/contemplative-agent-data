# Weekly Diagnosis — 2026-10-02

**Source report**: weekly-2026-10-02.md
**Diagnosis date**: 2026-10-03

## F1. Structural (code / schema / pipeline diff)

### F1.1. The `parent_rejected` re-entry came back after RFC-0038, and no log names the target, so nobody can tell why

**Source (O-NNN / Exceptions)**: Exceptions, first entry ("One reply prompt was regenerated and rejected
23 times"); ledger line O-014 (changed).
**Code reference**: `src/contemplative_agent/adapters/moltbook/publish.py:222`
**Structural change**: Give the `kind: "publish"` record a target column, so a failed reply names what it
was replying to. `PublishOutcome._record` (`publish.py:218-228`) calls `record_publish_outcome`
(`core/skill_selection.py:518-525`) with `comment_id` (the *created* comment, null on failure),
`publish_status`, `http_status` and `failure_reason`. Nothing in that call identifies the parent. The
change: the reply path passes its target into `publish_outcome(...)` (`reply_handler.py:462`, where
`reply_key` and `comment_id` are in scope), and the writer records it as an id-shaped column, for
example `target_key_sha256` of `reply_key` plus `parent_comment_id` stripped to printable ids, written
as explicit null on every other path, as the RFC-0029 columns are. That makes two questions answerable
offline. Are the 23 rows one reply key or many? Did `_reply_failed`'s retirement mark
(`reply_handler.py:289-296`) land for each of them? ADR-0106 D3 gets an amendment in the same PR (a
schema change on the record that ADR owns).
**Why this is structural, not symptomatic**: The repair RFC-0038 shipped keys its effect on a
`reply:{post}:{comment}` identity. The only record of the outcome is a row that omits that identity, so
the repair's effect cannot be verified, and the recurrence cannot be told apart between its two live
explanations. One is distinct comment ids carrying one byte-identical body: the five 404 triplets with
no request between them, `api-audit.jsonl:56699-56701 / 57167-57169 / 58312-58314 / 58373-58375 /
58687-58689`, fit this, because a marked key is skipped within the session (`reply_handler.py:249-250`).
The other is a mark that does not hold across sessions (the cold-rebuild hole that `memory_repos.py:436-445`
and ADR-0106 D3 already name). Each explanation calls for a different repair. ADR-0075's Verify-gate
question, "which log answers why", fails on this path today. The 2026-09-25 findings read one
row as "the repair holds", and that reading was not decidable either.
**Related ADR**: ADR-0106 (D3 and its 2026-09-12 / 2026-09-19 amendments), ADR-0075; `rfcs/0038`
(done 2026-09-19), `rfcs/0029` (done 2026-09-12).
**Filed**: T-PUBLISH-ROW-REPLY-TARGET

Self-check: code read (`publish.py:90-255`, `reply_handler.py:200-625`, `core/skill_selection.py:518-548`,
`core/memory_repos.py:393-445`). Not implemented: the window's `publish_failed` row keys were read, and
no target field exists. No parameter involved. ADR-0106 D3 records the platform *message* being kept out
of the log (untrusted text) but has no ruling against an id column; ids are platform-assigned and are
already logged as `comment_id` / `post_id` elsewhere. Ledger: `rfcs/0038` is the behavioural repair (done)
and `rfcs/0029` the reason column (done); neither adds the target, and nothing in `rfcs/` or
`.notes/archive/tasks/` does. Shared state: `record_commented` / `has_commented_on` are also used by
`feed_manager.py:707` with bare `post_id` keys, and this F1 does not touch them. Re-reply type check:
this is not a published re-reply claim. Nothing was published, and the counterparty identity is exactly
what the change makes recordable. The behavioural fix is left for after the column reads, because
which of the two explanations holds is the unknown.

## F2. Identity-level open questions

None this week. O-016 and O-017 are effects of a gate on the mechanism layer (`rfcs/0046` enforce),
not of value-layer text. The continuing lines (O-001..O-015) get no fresh diagnosis here.

## F3. Pure observations

### F3.1. The score4 gate's first week: volume below band, and the comment-report relevance column now shows the paired score, not the gate's

**Source (O-NNN / Exceptions)**: O-016, O-017 (Deviations); Exceptions "Session ledger, and the call mix after the enforce switch"
**Observation**: Comments per session went from 11.4 (10 sessions) to 6.1 (18 sessions) at
2026-09-28T15:00Z, and the window total, 391, is the first below the declared 400–550 band in the
five windows read. That is the direction `rfcs/0046` registered beforehand (Tier L, the shrinking
side: enforced gate rate 0.35 against live 0.62). Two side readings. (1) The comment-report header
`relevance` is the free-generation score that is still computed alongside the gate
(`feed_manager.py:256-267`), so 11 comments now show 0.30–0.70. A reader of the comment reports who
takes that column as the gate value will misread the regime. (2) Per-session LLM work moved toward
scoring and notes: median `score_relevance` 27.5 → 45 and `internal_note` 22 → 42, against
`skill_selection` 21 → 16. A note is generated when the live score clears the upvote-only bar *or*
the score4 gate passes (`feed_manager.py:411`), so a post the score4 gate closes can still cost a note.
This window cannot separate how much of the rise comes from more posts scored per session (comment
pacing freed) and how much from that or-condition.
**What to watch next week**: the `rfcs/0046` face gate (paired 300 rows were due around 2026-10-01).
On keep, the paired free-generation call is dropped and O-017's shape should end. The output-volume
band declared 2026-08-26 then describes a regime that no longer runs, which is a baseline
recalibration for the Saturday gate, not this session. On kill, comments per session should return to
around 11. Also watch the ratio of `internal_note` calls to comments published. If it stays near 1,127 : 224
under keep, the note's or-condition is the reading to take to the face gate.

### F3.2. Two consecutive weekly insight runs staged nothing; this week's ended at three `revise` verdicts

**Source (O-NNN / Exceptions)**: Exceptions "The window's insight run staged nothing"; O-010 (not tested)
**Observation**: The 2026-10-02T23:04Z run judged 7 novelty rows and named 3 clusters, all `revise`.
`revise` stops at naming by design (`core/insight.py:1010-1012`, option A: target and reason stay in
the audit log). The 2026-09-25 run (13 stage rows) also staged nothing, and the last staged row is
2026-09-19. With `rfcs/0042` (entrance narrowing) and `rfcs/0048` (store self-maintenance) both blocked,
`revise` verdicts accumulate in `insight-stages.jsonl` with no consumer. This is the "later reading" the
option-A docstring defers to.
**What to watch next week**: whether the next run yields a `new` verdict (a staged row) or only `revise`
/ `reconfirm`. A third consecutive zero-stage week with `revise` as the modal verdict is the count
`rfcs/0048`'s blocked state can be read against.

### F3.3. `injection_tokens_removed` is absent for the first time in five windows

**Source (O-NNN / Exceptions)**: Exceptions "Absent this window, present in the previous four"
**Observation**: All 36 injection-detect rows (28 sessions) are `guard_alive`. No row records tokens
removed, where each of the four prior windows had at least one. Because the guard is alive, the
reading is that no matching input reached it. Whether the score4 gate's narrowing (O-016) changed
which texts reach the guard is not readable from the census.
**What to watch next week**: a second window at zero. With `guard_alive` still present, that is an
input-population reading, not a guard failure. A missing `guard_alive` would be the other case.

### F3.4. Last window's watch items, closed or carried

**Source (O-NNN / Exceptions)**: weekly-2026-09-25-findings F3.1–F3.4
**Observation**: F3.1: the temperature-0 rate stayed below 10% (2.8%, 13/460, 53-entry catalog). That
completes the second window `rfcs/0044` needed for the 2026-10-04 gate; that entry now reads
`done 2026-09-26`. F3.2: 403s on the publish path went from 2 to 0, so it stays a one-off. F3.3: the sweep
corpus went down 87% through rotation, so the runner rows' Δ is not comparable this week. Only one
new runner signature appears (`srv update: - cache state`). F3.4 (unverified → regeneration join): not run;
`unverified` rows 29 → 24.
**What to watch next week**: the unverified join stays open until a deterministic join exists. If
F1.1's column lands, the same column makes that join possible for the reply path.

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/adapters/moltbook/reply_handler.py:200-625`;
  `adapters/moltbook/publish.py:90-255`; `adapters/moltbook/feed_manager.py:232-420` (+ grep for
  `enforce` / `generate_internal_note`); `core/memory_repos.py:370-445`; `core/insight.py:1000-1030`
  (+ grep for `revise`); `core/skill_selection.py:518-548`; `core/llm/__init__.py` (grep for
  `prompt_norm_sha256`)
- **ADRs read**: `docs/adr/0106-comment-outcome-recording.md:60-104` (D3 and amendments);
  `docs/adr/README.md` (grep for 0075 / 0106 / 0112 / 0113)
- **Identity/Constitution/Skills/Rules sections read**: none (no F2 raised); value-layer state via the
  materials diff and live-hash reconciliation
- **Past findings consulted**: `weekly-2026-09-25-findings.md` (full); `weekly-2026-09-18` / `09-11`
  findings by heading and `Filed:` line
- **Task ledger consulted**: `rfcs/*.md` frontmatter `state:` (all 48); full text of `rfcs/0038` and
  `rfcs/0046`; grep of `rfcs/` and `docs/` for `2f5352151363` / `parent_rejected`; Glob of
  `.notes/archive/tasks/` by name
- **Tasks filed**: T-PUBLISH-ROW-REPLY-TARGET

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-09-19 … 2026-10-02, 1030 judged of 1030 records. History: 7234 judged of 13865 records over 85 daily logs.
Catalog: 53 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 5, value_layer 9, wordform 42. Charged to a catalog entry (semantic, wordform): 47.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- (none)

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 0 name(s)

Candidate file: `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/weekly-2026-10-02-archive-candidates.txt` — 0 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
The file is empty this week; `--archive-names` rejects an empty file (exit 2), so it is not passed.
