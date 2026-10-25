# Weekly Diagnosis — 2026-10-24

**Source report**: weekly-2026-10-24.md
**Diagnosis date**: 2026-10-25

This window shares six of seven days with the window diagnosed yesterday (weekly-2026-10-23). The only new inputs
are the 2026-10-24 day and the materials sections added by the code pulled at about 2026-10-24T22:49Z
(`.git/logs/HEAD:196`): Relevance face reading, Gate record, Principle Coverage, Self-Regulation Reading, and the
census registry entries for P0-04.1 / P2-04.8. The diagnosis below covers those inputs. The rest is carried from
weekly-2026-10-23-findings.md.

## F1. Structural (code / schema / pipeline diff)

None this week. Each candidate ended in F3:

- the relevance adopt-window `reopen` (F3.1): its "before" side straddles the S40 regime boundary. But the logs I read
  cannot settle whether pre-S40 rows are comparable post-level gate readings: they carry `enforce_gate` on the same
  score4 / 0.3 rule. The verdict's only consumer is the Saturday gate (ADR-0113 Review-when). A change to the window
  rule needs that premise first.
- `distill-processed.jsonl` ABSENT (F3.2): no distill run has started on the P0-04.1 code yet, so absence is not
  evidence of a defect.
- `parent_rejected` (F3.4): carried; Principle 4 (fourth consecutive F3, no state change).

## F2. Identity-level open questions

None this week. No value-layer text changed (State Diff: 0 approval rows for every layer). No Exception or ledger line
points at identity, constitution, rules or skills text. The six ⚑ conversions to `baseline_proposal` are calibration
items for the gate, not value-layer questions.

## F3. Pure observations

### F3.1. The relevance adopt window reads `reopen`, and its "before" side spans the S40 deploy

**Source (O-NNN / Exceptions)**: Exceptions 1.
**Observation**: the reading is `before 206/786 (0.2621) -> after 189/498 (0.3795), diff 0.1174 CI [0.0651, 0.1699]
(band +-0.06) => reopen`, anchored on the 2026-10-10T03:36:34Z identity adopt (materials:784). `adopt_window` takes a
fixed 7 days on each side (`scripts/relevance_shadow_reading.py:484-490`). So the before side is
2026-10-03T03:36Z → 10-10T03:36Z. S40 reached production at 2026-10-08 00:00 JST = 2026-10-07T15:00Z
(`rfcs/0046:240`), which leaves about 4.5 of the 7 before-days in the pre-S40 regime. The third-reading pre-registration
defines its population as feed rows from 2026-10-07 (S40) onward (`docs/evidence/rfc-0046/preregistration-third-reading.md:15`).
It names the adopt as a second variable to keep out of that reading (`:20`). The adopt window has no such population
boundary. Two things are read: pre-S40 rows carry `enforce_gate` under the same score4 / 0.3 rule
(`relevance-2026-10-04.jsonl:1`: `gate_source score4`, `enforce_threshold 0.3`, alongside the retired `live_*`
fields), and row volume per day differs across the boundary (`enforce_gate` rows per file: 10-01 250, 10-02 262, 10-03
251, 10-04 204, against 10-11 80, 10-12 73, 10-13 93). Not read: whether the post population reaching the scorer
changed at S40 (`post_id` dedupe is applied per side, `_latest_per_post`, `:471`). Because both windows are anchored on
the adopt, this exact line will repeat every week until the next identity adopt.
**What to watch next week**: the gate's decision on the ADR-0113 re-read (150 post-level rows after the adopt,
`docs/adr/0113-…md:574-576`). One cheap read settles the S40 confound: the same `_side` rate over 10-07T15:00Z →
10-10T03:36Z only (post-S40 part of the before window). Unchanged against "after" (≈ 0.38) means S40 explains the
difference. Unchanged against "before" (≈ 0.26) points at the adopt. This is an operator- or triage-side read. This
chain does not run scripts.

### F3.2. `distill-processed.jsonl` is ABSENT because no distill run has used the P0-04.1 code yet

**Source (O-NNN / Exceptions)**: Exceptions 2.
**Observation**: the writer path is `src/contemplative_agent/cli/memory_cmds.py:50` and is passed at `:87`. The ADR-0060
amendment (2026-10-24, `docs/adr/0060-…md:222-250`) specifies a 2-day window minus the ledger. The window's last
distill run started 2026-10-24T18:30:05Z (`llm-calls-2026-10-24.jsonl:307`). Its per-episode lines cover 2026-10-24
episodes only and include no 2026-10-23T21Z episode (`distill-launchd.log:55747` onward). That is the old one-day
shape, and the local checkout received the code about 4 h 19 min later (`.git/logs/HEAD:196`). The census also lists
`.weekly.lock` as UNKNOWN: it is the P2-04.8 lock (`scripts/weekly-pipeline.sh:334`), held by this chain. Both
registry decisions belong to the Saturday gate.
**What to watch next week**: after the 2026-10-25T18:30Z distill run, (a) the file exists with one row per processed
episode, and (b) that run's per-episode lines include 2026-10-24T21Z episodes (the session the old window never read,
`docs/adr/0060-…md:224-227`). If (a) fails after a run that started on the new code, it becomes an F1 candidate: check
`reason=ledger_write_failed` / `ledger_skipped_load_failed` in `distill-launchd.log` first. Expect (b) to raise
episodes per day by about one session's worth. That raises O-020's growth reading by construction.

### F3.3. Knowledge growth tracks distilled episodes; patterns per episode are flat

**Source (O-NNN / Exceptions)**: O-020 (changed).
**Observation**: last week's watch asked for patterns per distill call. The per-episode lines in `distill-launchd.log`,
grouped by episode date, give 434 episodes → 860 patterns over 10-18..10-24 (1.98 per episode, 62.0 episodes/day).
Over 10-01..10-06 they give 225 → 444 (1.97, 37.5/day). On 10-24 alone it is 61 → 124, against a +123 live-store delta
over the same day. So the per-episode yield did not move. The episode count per day did. The window's
`distill.episode` calls (434) equal the episode-line count. The snapshot method (+730 over 6 days) and the live-store
method agree on direction. The band +300..+700 was declared on 2026-08-26, before S40 raised reply volume (O-018).
**What to watch next week**: whether the gate declares a post-S40 growth baseline (O-020's second expiry clause). Also
watch whether the P0-04.1 window (F3.2) adds a step up in episodes per day from the 2026-10-25 run onward. That would
be a third regime inside the same band question.

### F3.4. `parent_rejected` 53 rows, carried

**Source (O-NNN / Exceptions)**: Exceptions 3, 4; O-014 (changed).
**Observation**: 53 rows (49 yesterday's window), all on prompt `2f5352151363`, per day 7, 13, 9, 5, 4, 10, 5. No
burst on 10-24 above 5. The diagnosis and the unread premise are those of weekly-2026-10-23-findings F3.1 (deleted
parents listed by `GET /posts/{id}/comments` versus deleted during generation). This is the fourth consecutive F3 on
this signal with no state change. By Principle 4 it is not re-proposed with more weight. If the premise read is wanted,
it stays operator-side.
**What to watch next week**: row count and largest single-session burst, as before.

### F3.5. Last window's watch items

**Source (O-NNN / Exceptions)**: weekly-2026-10-23-findings F3.1–F3.4.
**Observation**:
- F3.1 (`parent_rejected`): carried as F3.4.
- F3.2 (knowledge growth): answered in F3.3. The yield per episode is flat, and the growth is in episode count.
- F3.3 (insight `revise` run): no insight run started inside 10-24, so the 10-23 run is the only one in the window. The
  first `new` verdict after 2026-10-17 is still unobserved, and RFC-0052's reading stays unexercised.
- F3.4 (403 with an HTML body on `POST /posts`): no second non-JSON 4xx on a write endpoint. The only 403 in the window
  is `api-audit.jsonl:73499`. The one new 503 (`:75980`, `GET /home`) carries JSON envelope keys, and nine earlier
  `GET /home` 503 rows exist since 2026-07-05.
**What to watch next week**: the items above.

## Diagnosis Metadata

- **Codebase files read**: `scripts/relevance_shadow_reading.py:60-104, 440-519`; `scripts/_census_registry.py:195-239`;
  `scripts/weekly-pipeline.sh:320-350`; `scripts/observation_ledger.py:95-154, 320-562` (delta validation, for the
  ledger delta); `src/contemplative_agent/cli/memory_cmds.py:50, 87` (grep)
- **ADRs read**: `docs/adr/0060-per-episode-grounded-distill.md:220-259`;
  `docs/adr/0113-decision-faces-and-relevance-score4-shadow.md:550-594`
- **Evidence read**: `docs/evidence/rfc-0046/preregistration-third-reading.md:10-64`
- **Identity/Constitution/Skills/Rules sections read**: none (no F2 candidate; State Diff shows no change)
- **Past findings consulted**: weekly-2026-10-23-findings.md (full)
- **Task ledger consulted**: `rfcs/*.md` frontmatter `state:` (all); `rfcs/0046:228-252`
- **Tasks filed**: none

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-10-11 … 2026-10-24, 1356 judged of 1356 records. History: 9162 judged of 15793 records over 108 daily logs.
Catalog: 53 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 3, wordform 31. Charged to a catalog entry (semantic, wordform): 34.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- (none)

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 0 name(s)

Candidate file: `reports/analysis/weekly-2026-10-24-archive-candidates.txt` — 0 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
The file is empty this week; `--archive-names` rejects an empty file (exit 2), so it is not passed.
