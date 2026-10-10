# Weekly Diagnosis — 2026-10-09

**Source report**: weekly-2026-10-09.md
**Diagnosis date**: 2026-10-10

## F1. Structural (code / schema / pipeline diff)

None this week. Each candidate ended in F3:

- the `parent_rejected` reading (F3.1): it answers the question RFC-0049 left open, but the repair it
  points to (key the terminal mark on more than `reply_key`) is the "post-level / body-hash dedup" shape
  in the principles appendix, at 3 generations a week.
- the self-post identifier (F3.2): single instance, and no self-written log records which voice label the
  prompt carried. The cause cannot be determined, so no code change can be argued.
- the reply-share shift (F3.3): a consequence of a merged, owner-decided change (S40), not a defect.

## F2. Identity-level open questions

None this week. The window's value-layer state is unchanged (0 diffs, 0 approval rows on all four layers),
and no Deviation or Exception points at identity, constitution, rules or skills text.

## F3. Pure observations

### F3.1. RFC-0049's columns answer its own open question for this window: one prompt, three different parents

**Source (O-NNN / Exceptions)**: Exceptions 1 (`parent_rejected`, the first window with the RFC-0049 target
columns); O-014 (changed).
**Observation**: all 3 `publish_failed / 404 / parent_rejected` rows in the window
(`skill-selection-2026-10-07.jsonl:27`, `-10-08.jsonl:115`, `-10-09.jsonl:135`) carry distinct
`parent_comment_id` and `reply_key_sha256` values. Each follows by 22–26 s a `moltbook.reply` generation
on the same 482-character prompt `prompt_norm_sha256 2f5352151363` (`llm-calls-2026-10-07.jsonl:157`,
`-10-08.jsonl:350`, `-10-09.jsonl:506`). RFC-0049 (`rfcs/0049:50`) left this choice open. For this window
the reading is explanation (a): the same counterparty text arrives under new comment ids, so RFC-0038's
key-scoped terminal mark (`reply_handler.py:249-250, 295`) does not apply. Explanation (b) (a mark lost
across sessions) would show one key failing twice, and no key repeats. The volume fell from 23 rows in
triplets (last window) to 3 singles. RFC-0049 and RFC-0038 are both `done`, so no open ledger entry holds
this reading. It is recorded here for the triage loop, which owns that thread. A repair keyed on the
counterparty body would be the rejected post-level / hash dedup shape (principles appendix). This
diagnosis does not propose one.
**What to watch next week**: confirm (a) if every `parent_rejected` row again has a distinct
`parent_comment_id`. Refute it if any `reply_key_sha256` appears in two `publish_failed` rows, because that
is explanation (b). The row count stays a cost reading (one generation ≈ 25–35 s per row).

### F3.2. A self-post named its seeds by containment-nonce identifier after the RFC-0018 voice label

**Source (O-NNN / Exceptions)**: O-019.
**Observation**: self-post `992c7d83` (2026-10-04 09:46:33) cites two seeds as *"the individual in
`<untrusted_content_13f229b60cda0371>`"* and *"…`<untrusted_content_8b72b6ca363cc81c>`"*. This is 1 of 28
self-posts, after 0 of 30 / 31 / 33 in the three prior windows. The same day's other self-post (`619ca1e7`,
21:55:34) cites its seeds by the label form *"[myspecarchitect]"* and *"[wren_of_somerville]"*. The label
path exists and is applied per seed (`adapters/moltbook/llm_functions.py:366-367`). It falls back to
`UNKNOWN_VOICE_LABEL` when a seed has no author name (`llm_functions.py:341-342`). No self-written log
records which label a self-post prompt carried, so this diagnosis cannot tell a fallback-label case (the
identifier was then the only handle) from a model choice made with a name label present. RFC-0018 rejected
removing identifiers on the output side (`rfcs/0018:44-46`, Principle 1).
**What to watch next week**: a second self-post body with an identifier would make the shape recurrent. A
repeat on a seed set whose comment-report shows named sources nearby would point away from the fallback
label. 0 of the window's self-posts carrying one starts O-019's two-window expiry.

### F3.3. After S40, the time a session used to spend on the upvote-only band went to replies, not comments

**Source (O-NNN / Exceptions)**: O-018; Exceptions 2 (session ledger, two regimes); O-016 archived.
**Observation**: at the 2026-10-07T15:00Z S40 boundary, per-session medians moved: internal notes
69 → 22.5, upvotes 64 → 7.5, full-text GETs 48.5 → 4, skill selections (generations) 13 → 23. Comments per
day held at 24–33 while replies rose to 44 / 41. Reply share was 55.2% on 10-08..09, against 35–41% in the
three prior windows. S39 forecast the removed work: *"score4 で落ちても live ≥ 0.70 なら全文 GET・internal_note
生成・upvote-only に進む（cache 後の post の 62%）"* (`rfcs/0046:203`). It did not forecast where the freed
session time would go. Comments stay gated by score4 (gate rate about 0.24, `rfcs/0046:199`), so the
extra generations went to replies on the agent's own threads. That is an inference from the medians. The
incoming-reply volume at the boundary was not separated. The window total (404) re-entered the declared
band, which fired O-016's expiry.
**What to watch next week**: the first full post-S40 + `4b85f22` window. Confirm if reply share stays
above 41% with comments near 25–33/day. Refute if share returns inside 35–41%, which would make the two
days a burst. Also read `comment-outcomes:reply` per session across the boundary, to separate "more time"
from "more incoming replies". Self-posts per day (7 on 10-09) is the `4b85f22` seed-path reading, and
`rfcs/0046:248` already waits on it.

### F3.4. Last window's watch items, closed or carried

**Source (O-NNN / Exceptions)**: weekly-2026-10-02-findings F3.1–F3.4; Exceptions 6, 8.
**Observation**:
- F3.1 (score4 first week): closed. The face gate recorded keep on 2026-10-04 (`rfcs/0046:180`), and
  O-016 / O-017 are archived this week.
- F3.2 (insight runs staging nothing): carried. The third consecutive run staged nothing: `insight-stages.jsonl`
  shows `revise` ×5 and `reconfirm` ×2, both stopping before the body call (`core/insight.py:111-115`).
  `rfcs/0042` holds this.
- F3.3 (`injection_tokens_removed` absent): refuted. 3 rows returned, all one `content_sha256` in session
  `0cfc77b2`.
- F3.4 (the unverified-publish → regeneration join): still open. No deterministic join exists inside this
  session.
**What to watch next week**: a `new` naming verdict (a staged row) in the next insight run would close
F3.2. A fourth empty run is a reading for `rfcs/0042`, not a new filing.

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/adapters/moltbook/llm_functions.py:300-409`;
  `src/contemplative_agent/core/insight.py:1-115` (grep `reconfirm`)
- **ADRs read**: `docs/adr/README.md` (rows 0043, 0106, 0107, 0110, 0113)
- **Identity/Constitution/Skills/Rules sections read**: none (no F2 candidate; state diff shows 0 changes)
- **Past findings consulted**: weekly-2026-10-02-findings.md (F-section headings and watch lines)
- **Task ledger consulted**: rfcs/0046 (2026-10-03..10-09 sections), rfcs/0049, rfcs/0038 (frontmatter),
  rfcs/0018, rfcs/0042 (by reference); `rfcs/` and `.notes/archive/tasks/` Glob listings
- **Tasks filed**: none

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-09-26 … 2026-10-09, 919 judged of 919 records. History: 7693 judged of 14324 records over 92 daily logs.
Catalog: 53 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 4, wordform 23. Charged to a catalog entry (semantic, wordform): 27.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- (none)

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 0 name(s)

Candidate file: `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/weekly-2026-10-09-archive-candidates.txt` — 0 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
The file is empty this week; `--archive-names` rejects an empty file (exit 2), so it is not passed.
