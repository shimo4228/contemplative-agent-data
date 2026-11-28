# Weekly Diagnosis — 2026-11-27

**Source report**: weekly-2026-11-27.md
**Diagnosis date**: 2026-11-28

## F1. Structural (code / schema / pipeline diff)

### F1.1. The self-post prompt asks for the seed voice "in its own words", and two posts quoted the whole framed block

**Source (O-NNN / Exceptions)**: O-022 (changed this week: second hit) with the O-019 line (nonce baseline ABOVE for a
third consecutive window). This is the confirm condition that weekly-2026-11-20 F3.1 set.
**Code reference**: `config/prompts/cooperation_post.md:5` (with the layout it acts on,
`src/contemplative_agent/adapters/moltbook/llm_functions.py:253`)
**Structural change**: At line 5 the prompt tells the model to bring a voice in *"by name and in its own words"*, and
at line 7 to *"Stay close to the specific language each voice uses"*. Above those lines, `format_feed_seeds` lays out
each voice as one unit: a label line, then the framed title and body (`llm_functions.py:253`). Both published carriers
open with that unit, copied whole: a bracketed name line, the frame, the body cut mid-sentence, and the closing frame.
- `f670c32d`: 2026-11-17, `comment-report-2026-11-17.md:2941-2942`.
- `299575b0`: 2026-11-25, materials:13967-13973.

The change is generation-side, in the self-post prompt. It names the form a quotation takes: a phrase taken into the
agent's own sentence, attributed by the voice's name. That form is not the block as it appears above the
instruction. The triage's premise check is whether the copied unit really comes from the quoting instruction. It
could also be a format echo of the first block. Reading the two posts' prompts in the replay harness settles this,
because the seed block is the prompt's own text.
**Why this is structural, not symptomatic**: The intervention changes what the generator is asked to produce. It
does not remove anything from the output. Removing the frame or identifier after generation is the substring filter
that Principle 1 rejects, and RFC-0018 declined it for that reason (`rfcs/0018-self-post-seed-voice-label.md:47-49,
55-56`). RFC-0018's label fixed the *anonymous* reference (identifier as name). It did not address *how much* of a
voice gets quoted, and no RFC does.
**Related ADR**: ADR-0043 (per-post seeding), ADR-0054 (prompt externalization: `config/prompts/` is fixed apparatus,
so this is a mechanism-layer repair), RFC-0018 (done 2026-08-29)
**Filed**: T-SELF-POST-SEED-BLOCK-QUOTE

Self-check (all passed):
- code reference given;
- not implemented: `cooperation_post.md` has no wording about quotation form, and no commit subject in the window
  names it (`.git/logs/HEAD:225-242`);
- not an existing parameter;
- not withdrawn in an ADR or RFC (RFC-0018 declined output-side removal only);
- no same intervention in `rfcs/` or `.notes/archive/tasks/`;
- retrieval / shared state and re-reply checks: N/A.

## F2. Identity-level open questions

None this week. No value-layer diff landed in the window (materials:109-152). The O-021 question stays as last week's
F2.1, unanswered and not re-asked (Principle 4).

## F3. Pure observations

### F3.1. On the S57 reply path, one comment text under many comment ids is read by score4 each time

**Source (O-NNN / Exceptions)**: Exceptions 1
**Observation**:
- `content_sha256 aba56850…` appears on 102 `relevance-*.jsonl` rows in the window, each with its own target id
  (e.g. `reply-order-2026-11-24.jsonl:59, 94-95`). It accounts for the whole of the census's redundancy top list.
- The memo is keyed on the reply key plus the text digest, by design: *"one reading per comment per session"*
  (`adapters/moltbook/reply_handler.py:276-282, 439`). Identical text under a new comment id is a new key.
- On 2026-11-24 the 22 readings of the one text range 0.028–0.176 at temperature 0. Two copies in the same second
  scored 0.061 and 0.176. The verdict was `below_threshold` every time, so no decision differed.
- Cost: ~100 decision calls of ~1.4 s over the week, against 3,905 `relevance_reply` calls.
- This is not filed. It sits inside S57's declared design, the verdict never moved, and RFC-0046 already records the
  temperature-0 non-reproducibility (`rfcs/0046-relevance-gate-score4-logprobs-shadow.md:189`). A memo on text alone
  would be an owner-level design change to the S57 path, not a repair.

**What to watch next week**:
- Confirm a decision-relevant shape: one `content_sha256` group whose `decision_p_top` straddles the 0.3 cut (some
  rows `below_threshold`, some passed). That is the RFC-0046 flip, now on the reply path.
- Refute: the group stays entirely on one side of the cut, or leaves the logs.

### F3.2. First window on the S57 reply-selection code

**Source (O-NNN / Exceptions)**: Exceptions 3, 5, 8; Ledger (reply-share verdict)
**Observation**:
- Replies 238 (297 last window).
- Daily reply share is 37.8–58.7%, with four days below the band's 48% floor. The window verdict is INDETERMINATE
  against a band declared for the earlier regime.
- `thread_cap` labelled 452 candidates.
- `publish_failed` fell 18 → 1, and no `parent_rejected` row is in the window. This was last week's RFC-0049 (a)
  shape.
- RFC-0075 review-when (a) did not fire: the daily comments + replies are 51–70.

**What to watch next week**:
- Whether the reply-share verdict reads BELOW for a second window. That opens an observation by minimum persistence,
  and would put the baseline's regime label in front of the gate.
- Whether `parent_rejected` stays at 0.

### F3.3. No approvals for a full window

**Source (O-NNN / Exceptions)**: Exceptions 2, 6
**Observation**:
- `audit.jsonl` has no row in the window (last 2026-11-14T06:03:57Z).
- Adopted = archived = identity updates = 0 for 2 consecutive weeks (materials:664; the threshold is 8).
- `insight-staged.jsonl` has been empty since 2026-09-19.

This is a reading of the metabolism floor, not a fault: the insight run ran (13 judged, 1 revise).
**What to watch next week**:
- Whether the Self-Regulation Reading's no-change run reaches week 3.
- Whether any staged row appears.

### Follow-up on last week's F3

- F3.1 (self-post seed block): confirmed. It moved to F1.1 above.
- F3.5 (regeneration after unverified publishes): the same shape once more, parent `45efe042` in `e3f5b944`
  (`skill-selection-2026-11-22.jsonl:106, 108, 110`). It is carried in Exceptions and nothing new is added
  (Principle 4).
- The relevance reopen line is unchanged (same counts as 11-13 and 11-20; materials:826). It is carried. The re-read
  decision stays with the gate.

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/adapters/moltbook/reply_handler.py:1-470`,
  `src/contemplative_agent/adapters/moltbook/llm_functions.py:195-306`, `config/prompts/cooperation_post.md`
  (full), `src/contemplative_agent/core/llm/guard.py` (grep: `sanitize_output`, `INJECTION_TOKENS`, nonce lines)
- **ADRs read**: none opened this week. ADR-0043 / ADR-0054 are cited through the code docstrings and the RFC-0018
  text.
- **Identity/Constitution/Skills/Rules sections read**: state diff only (no changes in any layer)
- **Past findings consulted**: weekly-2026-11-20-findings.md (full)
- **Task ledger consulted**: RFC-0018 (full), RFC-0075 (lines 1-30 and grep), RFC-0046 / RFC-0032 / RFC-0078 (grep);
  `rfcs/` and `.notes/archive/tasks/` filename lists
- **Tasks filed**: T-SELF-POST-SEED-BLOCK-QUOTE

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-11-14 … 2026-11-27, 1179 judged of 1179 records. History: 12246 judged of 18877 records over 141 daily logs.
Catalog: 53 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 12, value_layer 2, wordform 51. Charged to a catalog entry (semantic, wordform): 63.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- (none)

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 0 name(s)

Candidate file: `reports/analysis/weekly-2026-11-27-archive-candidates.txt` — 0 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
The file is empty this week; `--archive-names` rejects an empty file (exit 2), so it is not passed.
