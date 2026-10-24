# Weekly Diagnosis — 2026-10-23

**Source report**: weekly-2026-10-23.md
**Diagnosis date**: 2026-10-24

## F1. Structural (code / schema / pipeline diff)

None this week. Each candidate ended in F3:

- the `parent_rejected` cost (F3.1): the row count grew 3 → 19 → 49. The only repair keyed on what the rows
  record, the counterparty body, is the rejected post-level / hash dedup shape (principles appendix). Two prior
  diagnoses already left it in F3 (10-09 F3.1, 10-16 F3.1), and Principle 4 forbids escalating the same mechanism.
  A repair keyed on something else needs a premise the logs do not record (see F3.1).
- the knowledge growth deviation (F3.2): one window. No code change can be argued before the input side
  (patterns per distill call) is read.
- the first 403 on `POST /posts` (F3.4): one event, and the body was not JSON, so it was not recorded.

## F2. Identity-level open questions

None this week. No value-layer text changed (State Diff: 0 approval rows for every layer). No Deviation or
Exception points at identity, constitution, rules or skills text.

## F3. Pure observations

### F3.1. `parent_rejected` reached 49 rows, 12 of them in one 12-minute run, all on the comment-scan path

**Source (O-NNN / Exceptions)**: Exceptions 2, 8; O-014 (changed).
**Observation**: all 49 `publish_failed` rows in the window are `404 / parent_rejected`. The 482-character prompt
`prompt_norm_sha256 2f5352151363` was generated exactly 49 times, with the same per-day counts (1, 7, 13, 9, 5, 4,
10). Session `e5733260` ran twelve consecutive `internal_note → skill_selection → reply` triples on it between
2026-10-19T09:15:39Z and 09:27:20Z (`llm-calls-2026-10-19.jsonl:133-168`), and each one ended in a 404. That is
about 36 LLM calls and 12 minutes with no published body (comment report: 09:14:38 → 09:28:48). The rows come from
the comment-scan path. The notification path posts top-level with `comment_id=""`, so a genuine parent rejection
cannot arise there (`src/contemplative_agent/adapters/moltbook/reply_handler.py:280-283, 382`). The 482-character
prompt has no `Original post:` section, which matches `_handle_post_comments` passing `original_post=""`
(`reply_handler.py:404-415`). RFC-0038's terminal mark is keyed on `reply:{post_id}:{comment_id}`
(`reply_handler.py:249, 295-296`), so a new comment id with the same text is a new key every time. Last week's
refutation test for explanation (a) did not fire: no `reply_key_sha256` appears in two `publish_failed` rows. The one
in-session key repeat (`98ee4360`, `d8551911…`) is `unverified` (03:17:54Z) then `published` (03:57:28Z) on parent
`587b5f9b…` (`skill-selection-2026-10-21.jsonl:13, 48`), which is the F3.2 shape of last week, not a rejection.
**Premise that would allow a non-dedup repair**: whether these parents are already gone when the agent lists them
(the platform returns deleted or moderated comments in `GET /posts/{id}/comments`) or disappear during the ~60 s of
generation. `api-audit.jsonl` records envelope keys only, so no self-written log can tell the two apart. The publish
rows carry no `post_id` (`skill-selection-2026-10-19.jsonl:58`), so whether the 12 parents sit under the agent's own
self-post `eefa6ade` (published 09:14:38) is also not recorded.
**What to watch next week**: the row count as a cost reading (3 LLM calls ≈ 50–80 s per row), and the largest
single-session burst. If the triage loop wants to settle the premise, the read is one `GET /posts/{id}/comments`
response for a post that produced a burst, checked for a deleted / removed marker on the rejected parent ids. That is
an operator-side read, not this chain's.

### F3.2. Knowledge grew +849 in 7 days against a +300..+700 band

**Source (O-NNN / Exceptions)**: O-020 (new).
**Observation**: the live store total grew 11,779 → 12,628 between the 10-17 and 10-24 State Invariant reads. Live
rows +776, tombstones +73. The three prior windows were +479..+543 on the snapshot method. Over the same window the
inputs moved much less: `distill.episode` calls 410 → 437 (+7%) and published bodies 541 → 592 (+9%). So the growth
is not explained by volume alone. Patterns per distill call is the unread quantity. `duplicate_live_texts` still
holds, so exact repeats are not the source.
**What to watch next week**: the next 7-day growth on the live-store method, and patterns per `distill.episode` call
if a reading of the distill run (count of `Added pattern` lines per daily run in `distill-launchd.log`) can be dated.
Growth back inside +700 makes this a one-window excursion. A second window above it, with flat distill-call counts,
locates the change in the distill output per call.

### F3.3. The insight run doubled its novelty batches and named 20 clusters, all `revise`

**Source (O-NNN / Exceptions)**: Exceptions 1; O-010 (not tested).
**Observation**: 22 novelty batches against 10 and 8 in the two prior runs (`insight-novelty.jsonl:60-99`). The run
produced 20 naming verdicts, all `revise` (`insight-stages.jsonl:30-49`). Without a `new` verdict the description /
name calls repaired by RFC-0052 (`done 2026-10-17`) did not run, so its named reading (`rfcs/0052:48`) is still
unexercised. No stage downstream of naming consumes a `revise` verdict in this run's logs (no staging row; the
run's `insight.*` calls are 22 `insight.novelty` and 20 `insight.naming` only, `llm-calls-2026-10-23.jsonl:543-605`). The batch count increase coincides with O-020.
**What to watch next week**: the first `new` verdict after 2026-10-17 and whether its `insight.description` row
carries the distill-sized `system_chars` rather than ~102k. The batch count is also worth watching: does it keep
tracking store growth?

### F3.4. One `POST /posts` returned a 403 with an HTML body

**Source (O-NNN / Exceptions)**: Exceptions 3.
**Observation**: `api-audit.jsonl:73499` (2026-10-21T09:16:31Z, session `91e0e3c2`, `rate_remaining 29`) is the first
403 on `POST /posts` in the file. Every earlier 403 is on `POST /posts/{id}/comments`. The body was not JSON (no
`keys`; sweep `<!doctype html public` ×1), which is the shape of an edge or proxy refusal, not an API error envelope.
Seven self-posts were still published that day.
**What to watch next week**: any second non-JSON 4xx on a write endpoint. Two or more would make it a platform-side
pattern worth one `debugging.md`-style read (policy signal versus transient), not a code change.

### F3.5. Last window's watch items, closed or carried

**Source (O-NNN / Exceptions)**: weekly-2026-10-16-findings F3.1–F3.5.
**Observation**:
- F3.1 (`parent_rejected` bursts): the bursts stayed inside one session and grew in size (6 → 12). Explanation (a)
  still holds. Carried as F3.1 above.
- F3.2 (unverified → regenerated reply keys): one more in-session instance (`98ee4360`, 39.5 min apart, see F3.1).
  Cross-session pairs were not counted this week.
- F3.3 (reply share): second window above 48%. 10-21 at 68.2% sits above the proposed band, which makes it a burst
  reading by last week's rule. O-018 carries it, and the weekly total is now above the declared 400–550 output
  baseline.
- F3.4 (report-session fast failure): no recurrence. `weekly-pipeline-audit.jsonl` has 15 rows in the window, all
  `ok`.
- F3.5 / last week's F1.1 and F1.2: both filed and done (`rfcs/0052`, `rfcs/0053`). RFC-0053's reading is met
  (sync resumed, Exceptions 14). RFC-0052's reading is unexercised (F3.3).
**What to watch next week**: the items above.

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/adapters/moltbook/reply_handler.py:240-439`
- **ADRs read**: `docs/adr/README.md` (rows 0038, 0106, 0111, 0112, 0113, via grep)
- **Identity/Constitution/Skills/Rules sections read**: none (no F2 candidate; State Diff shows no change)
- **Past findings consulted**: weekly-2026-10-16-findings.md (full); weekly-2026-10-09-findings.md (O-014 /
  `parent_rejected` lines)
- **Task ledger consulted**: `rfcs/*.md` frontmatter `state:` (all); rfcs/0046 (230-295); rfcs/0052 (grep, :48);
  rfcs/0053 (grep, :48); rfcs/0054 (grep)
- **Tasks filed**: none

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-10-10 … 2026-10-23, 1350 judged of 1350 records. History: 9043 judged of 15674 records over 106 daily logs.
Catalog: 53 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 3, wordform 34. Charged to a catalog entry (semantic, wordform): 37.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- (none)

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 0 name(s)

Candidate file: `reports/analysis/weekly-2026-10-23-archive-candidates.txt` — 0 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
The file is empty this week; `--archive-names` rejects an empty file (exit 2), so it is not passed.
