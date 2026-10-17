# Weekly Diagnosis — 2026-10-16

**Source report**: weekly-2026-10-16.md
**Diagnosis date**: 2026-10-17

## F1. Structural (code / schema / pipeline diff)

### F1.1. The insight description and name calls run under the full action-time system prompt and are skipped over budget

**Source (O-NNN / Exceptions)**: Exceptions 1 (the run's one `new` naming verdict ended `description_invalid`);
O-010 (not tested).
**Code reference**: `src/contemplative_agent/core/insight.py:331`
**Structural change**: `_ask_json_field` (`insight.py:313-345`) calls `llm.generate_full` with no `system=`.
`_generate_full` then substitutes `_build_system_prompt()` (`core/llm/__init__.py:768`), which is identity +
axioms + the whole `<learned_skills>` corpus + rules (`core/llm/prompting.py:285-330`). On 2026-10-16 that was
`system_chars 102,193` and an estimated 34,898 input tokens, so the context-budget pre-flight
(`core/llm/__init__.py:781-803`) skipped the call (`llm-calls-2026-10-16.jsonl:578`, `outcome budget_exceeded`).
The body call in the same function family passes `system=llm.get_distill_system_prompt()` (`insight.py:278`)
with the comment that skill generation must not be conditioned on the existing skill corpus. The change: give
the two short constrained calls (`_write_description`, `insight.py:352`; `_write_name`, `insight.py:384`) an
explicit system prompt chosen the same way as the body call's, so their input size no longer grows with the
skill store.
**Why this is structural, not symptomatic**: the input that overflows is not the cluster or the body
(`prompt_chars 2,205`); it is the skill corpus the call never needed, so every cluster that reaches the
description step fails the same way while the store is this size. `description_invalid` is a fault reason
(`insight.py:134-143`), so the incremental window is kept and the same cluster recurs into the same skip. The
whole llm-calls history has this one `insight.description` row and no `insight.name` row: the RFC-0042 item 3 /
ADR-0111 call split has not completed once in production.
**Related ADR**: ADR-0111 (split calls), ADR-0058 (no value-layer injection at distillation time), ADR-0087
(budget guard)
**Filed**: T-INSIGHT-DESCRIPTION-CALL-FULL-SYSTEM-PROMPT

### F1.2. The home-path gate searches files rsync does not mirror, so one stale file stops every daily sync

**Source (O-NNN / Exceptions)**: Exceptions 2.
**Code reference**: `scripts/sync-research-data.sh:136`
**Structural change**: the gate greps the data repo's whole working tree (`sync-research-data.sh:136`), but
rsync's `--delete` mirror (`:76-107`) excludes `reports/.private/` (`:103`). rsync neither refreshes nor deletes
a file that sits in an excluded path on the destination. So a file there that contains the home path stays in
the tree, `scripts/rewrite_home_paths.py` (which works on `MOLTBOOK_HOME`) cannot reach it, and the gate stops
the sync on every run until a person edits the data repo. That is the state since 2026-10-15T20:30Z
(`logs/sync-data-launchd.log:13054-13057`, one file under `reports/.private/` both times;
`reports/PIPELINE-STATUS.md:10`). The change makes the gate's search set and the set `git add -A` (`:153`)
would publish the same set. This session cannot read the data repo's `git status` / `.gitignore`. The triage
premise check must first settle whether the named file is tracked, untracked-not-ignored, or ignored. In the
first two cases the gate is right and the remedy is on the data repo side, not a scope change.
**Why this is structural, not symptomatic**: deleting or rewriting the one file clears this week only. Any
file that ever lands in an rsync-excluded path of the data repo reproduces the stop. Nothing in the sync
removes it, and the watchdog row is the only signal (`rfcs/0051:81, 89`).
**Related ADR**: ADR-0010 (research data sync, RFC-0051 amendment)
**Filed**: T-SYNC-GATE-SCANS-RSYNC-EXCLUDED-PATHS

## F2. Identity-level open questions

None this week. The one value-layer change (identity.md, approved 2026-10-10T03:36:34Z) is O-003's loop
continuing with a new text. No Deviation or Exception points at identity, constitution, rules or skills text.

## F3. Pure observations

### F3.1. `parent_rejected` returned to 19 rows, every one a new parent for the same prompt

**Source (O-NNN / Exceptions)**: Exceptions 3; O-014 (changed).
**Observation**: last week's watch confirmed explanation (a). The 19 `publish_failed / 404 / parent_rejected`
rows carry 19 distinct `parent_comment_id` and 19 distinct `reply_key_sha256`. The 482-character prompt
`prompt_norm_sha256 2f5352151363` was generated exactly 19 times with the same per-day counts (6, 1, 4, 2, 2, 4).
No key failed twice, so (b) (a terminal mark lost across sessions) is not supported. Bursts: 6 in session
`52d87cb4` (`skill-selection-2026-10-11.jsonl:82-92`, 09:55:05Z–10:01:03Z) and 3 in `58068683`
(`-10-16.jsonl:149-167`). Each arrival costs one internal note, one skill selection and one reply generation
(the Redundancy triples). The repair the shape suggests is keyed on the counterparty body. That is the rejected
post-level / hash dedup shape (principles appendix), and last week's F3.1 already left it there (Principle 4).
**What to watch next week**: the row count as a cost reading, and whether bursts stay inside one session (a run
of new parent ids within minutes). Any `reply_key_sha256` seen in two `publish_failed` rows refutes (a).

### F3.2. One `unverified` reply was generated again for the same target in the same session

**Source (O-NNN / Exceptions)**: Exceptions 7, 8.
**Observation**: this partly answers last window's F3.4 (the unverified-publish → regeneration join). In
`0fd73b95`, `skill-selection-2026-10-12.jsonl:113` (`unverified`, 15:48:10Z) and `:119` (`published`, 15:55:32Z)
share `reply_key_sha256 207f6b08…` and `parent_comment_id 72e84c04…`. It is the only reply key repeated inside a
session this window (Id-field repeats max 2, only there), against 31 `unverified` rows. The census cannot see
repeats across sessions, and whether the first, unverified comment later became visible is not recorded.
**What to watch next week**: count reply keys that appear as `unverified` and then `published` in any later
row (a grep over the window's `skill-selection` files). If more than a few appear, the double-visible case is
worth a premise check against the platform.

### F3.3. The post-S40 reply share held for a full window

**Source (O-NNN / Exceptions)**: O-018 (changed); Exceptions 5.
**Observation**: replies are 312/541 (57.7%), daily 36–52, while comments stay 22–35/day. This confirms last
week's watch ("stays above 41% with comments near 25–33/day"). Incoming replies (`comment-outcomes:reply`) have a
median of 18.5 per session in one regime. The pre-S40 value of that column is not in any materials, so "more
time" and "more incoming replies" are still not separated. A composition baseline is proposed in the delta for
the gate.
**What to watch next week**: a second window inside the proposed 48–65% daily band supports ratification. A day
outside it is a burst reading, not a regime change.

### F3.4. Last week's report session failed in 2.4 s and was re-run by hand

**Source (O-NNN / Exceptions)**: Exceptions 4.
**Observation**: `REPORT_SESSION_FAIL` then `REPORT_MISSING` at 2026-10-10T00:00:13Z, and a new chain at 01:13Z
that ended `ok`. No self-written log records the session's exit cause, so no code change can be argued.
**What to watch next week**: a second fast failure (seconds after `materials ok`) would make it a recurring
start-up fault, worth one line of exit-cause logging in `weekly-pipeline.sh`.

### F3.5. Last window's watch items, closed or carried

**Source (O-NNN / Exceptions)**: weekly-2026-10-09-findings F3.1–F3.4.
**Observation**:
- F3.1 (`parent_rejected` explanation): closed as (a). See F3.1 above.
- F3.2 (self-post identifier): 0 of 38 self-post bodies this window. O-019's two-window clock is at 1.
- F3.3 (reply share): confirmed. See F3.3.
- F3.4 (insight runs staging nothing, carried as "a fourth empty run is a reading for `rfcs/0042`"): changed in
  kind. The run produced a `new` verdict and failed on a fault, not on a verdict. That is F1.1, not a reading
  for `rfcs/0042` (done 2026-10-12).
**What to watch next week**: after F1.1 is decided, the next run's `insight.description` / `insight.name` rows
and their `system_chars`.

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/core/insight.py:60-459`;
  `src/contemplative_agent/core/llm/__init__.py:636-725, 764-803`;
  `src/contemplative_agent/core/llm/prompting.py:90-330`; `src/contemplative_agent/core/llm/guard.py:417-457`;
  `scripts/sync-research-data.sh:76-164`
- **ADRs read**: `docs/adr/README.md` (rows 0010, 0058, 0084, 0087, 0111)
- **Identity/Constitution/Skills/Rules sections read**: identity.md diff in the State Diff only (no F2 candidate)
- **Past findings consulted**: weekly-2026-10-09-findings.md (full)
- **Task ledger consulted**: rfcs/0046 (244-273), rfcs/0051 (grep: 29, 46, 81, 87-97), rfcs/0042 and rfcs/0024,
  0027, 0038, 0041, 0048 (frontmatter / grep); `rfcs/` and `.notes/archive/tasks/` Glob listings
- **Tasks filed**: T-INSIGHT-DESCRIPTION-CALL-FULL-SYSTEM-PROMPT, T-SYNC-GATE-SCANS-RSYNC-EXCLUDED-PATHS

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-10-03 … 2026-10-16, 1104 judged of 1104 records. History: 8338 judged of 14969 records over 99 daily logs.
Catalog: 53 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 3, wordform 27. Charged to a catalog entry (semantic, wordform): 30.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- (none)

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 0 name(s)

Candidate file: `reports/analysis/weekly-2026-10-16-archive-candidates.txt` — 0 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
The file is empty this week; `--archive-names` rejects an empty file (exit 2), so it is not passed.
