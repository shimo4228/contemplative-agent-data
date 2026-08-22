# Weekly Decision Packet — 2026-08-21

- Run: `weekly-2026-08-21-090000`
- Generated: 2026-08-22T01:59:16.675016+00:00
- Findings: F1 4 / F2 3 / F3 5
- Reason codes this run: NO_RECURRENCE, SCOPE_ESCALATED

## 1. Decision inventory

- code patch: 0 件（apply → 単一 commit の対象、+4 件は §3 の全文ゲートへ昇格（apply 対象外））
- prompt diff: 4 件（本文全文を下に提示 — 個別承認、うち 4 件は code scope からの昇格）
- insight: 46 件（`adopt-staged` の対象）
- pipeline improvement: 0 件
- dead code candidate: 1 件（検出のみ — 削除・whitelist の判断は人間）
- docs consistency: 4 件（検出のみ — doc 修正は人間 commit、§9 参照）

## 2. Code fixes (unattended, Verify-passed where noted)

| finding | scope | attempts | result | reviewer | patch / reason |
|---|---|---|---|---|---|
| F1.1 | code → **SCOPE_ESCALATED** | 2 | patch_ready | CONCERNS→APPROVE | `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-21/prompt/F1.1.patch` |
| F1.2 | code → **SCOPE_ESCALATED** | 2 | patch_ready | CONCERNS→APPROVE | `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-21/prompt/F1.2.patch` |
| F1.3 | code → **SCOPE_ESCALATED** | 2 | patch_ready | CONCERNS→APPROVE | `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-21/prompt/F1.3.patch` |
| F1.4 | code → **SCOPE_ESCALATED** | 2 | patch_ready | CONCERNS→APPROVE | `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-21/prompt/F1.4.patch` |

**各 finding の診断見出し**（診断段の LLM 出力 — その patch が何を直すのかの手がかり。承認判断は下の reviewer verdict と、§3 に本文がある行（prompt scope と昇格分のみ）はその diff で行う。code scope の行に §3 本文は無く、patch は table のパスに置かれている）:

- **F1.1** — The approval join matches the identity section on an exact leaf name, so the one defect class that renames the target makes its own approval row invisible — and the instrument's loudest output is what fills the gap
- **F1.2** — The join answers "was there an approval row" but never "does the live text match an approved hash", so a value layer repaired by hand reads identical to one that was never touched
- **F1.3** — The weekly report was promoted with its first ~200 lines missing, because the only completeness test on it is �-s�, while the two downstream artifacts of the same chain both have structural ones
- **F1.4** — �score_relevance� short-circuits only on a body that is exactly empty, so a post with no proposition in it reaches the LLM and can return the ceiling

### Review notes (final round, full text)

レビュー本文は LLM 出力（finding 由来の連鎖）— 検査者の所見であって承認ではない。CONCERNS のまま採用する判断は人間に属する。

#### F1.1 — CONCERNS→APPROVE

```text
VERDICT: APPROVE

- **All four round-1 concerns are addressed in the diff, not just rebutted.** Doc sync: `docs/CODEMAPS/architecture.md:725` now describes the command arm, the yield-to-directory rule, "Four renderings" incl. the residual and its gating class, and the timestamp caveat (verified this is the only mechanism description — no ADR/CHANGELOG copy exists). Stale sibling claim: `tests/test_cli_adopt.py:498-521` is past-tensed, the brittle `:113-123` reference dropped, assertions untouched. Shared vocabulary: `IDENTITY_COMMANDS` now lives in `scripts/_audit.py:23-31` with both readers importing it (`value_layer_approval_join.py:62`, `value_layer_due_check.py:44/212`) and `test_the_identity_command_vocabulary_is_shared_with_the_cadence_reading` pinning identity with `is`. Caveat: module docstring `value_layer_approval_join.py:13-29` names the undatable+unplaceable gap and corrects "two states" → "three".
- **Fix still matches the finding's two Structural changes and nothing more.** Command-arm selection (`_matches_section`, identity branch) yields to `_DIR_SECTIONS` so no row clears two alarms (`test_the_command_arm_does_not_steal_a_directory_section_row`); the residual counter is window-scoped like every other tally (`test_an_out_of_window_unplaceable_row_is_not_counted`). The finding's optional "Additionally" (same-`content_hash`/different-`path` condition) is explicitly flagged-not-done, which is correct — implementing it would have crossed the render boundary that keeps target paths out (`value_layer_approval_join.py:31-36`); the new residual line also renders no path bytes, so that boundary holds.
- **No gate weakened; regression coverage is real.** No existing assertion, lint config or guard was changed anywhere in the diff. `test_a_renamed_identity_leaf_still_belongs_to_the_identity_section` reproduces the exact 08-15 shape (`identity.md` staged + `identity-2.md` approved) and asserts `NO APPROVED RECORD` is absent — it would have failed before the fix. `unparsable` keeps its old section-scoped semantics (`if mine:`), and `Reading` stays frozen with `unmatched` added; it is keyword-constructed only in `build_reading` and has no other constructor anywhere in the repo (grepped).
- **Scope expansion is reviewer-driven, not implementer-driven.** `_audit.py`, `value_layer_due_check.py`, `test_cli_adopt.py` and `architecture.md` are outside the finding's named files but each traces to a specific round-1 concern; `scripts/_matches_section` has no other callers, so the signature change (`raw_path: object` → `record: dict`) is contained.
- **Minor, not verdict-driving:** the test module docstring still says "the **three**-state distinction" (`tests/test_value_layer_approval_join.py:8`) while the file now pins four — same staleness class the diff fixed in the production docstring. Also, one unplaceable row renders a residual line in all four sections (pinned by `test_an_unplaceable_in_window_row_is_counted_and_rendered`), so the packet shows it ×4; defensible per-section semantics, worth knowing before reading a report.
```

#### F1.2 — CONCERNS→APPROVE

```text
VERDICT: APPROVE

- **Round-1 concern 1 (alarm calibration) closed.** `format_reconciliation` now names the never-had-a-row cause and directs the reader to a *rise* rather than the count (`scripts/value_layer_approval_join.py` ⚠️-branch, `+"FOUR causes … contemplative-agent init … cli/session_cmds.py:66,88"`), and the citation checks out against the real code (`src/contemplative_agent/cli/session_cmds.py:66` copytree, `:88` copy2 — neither writes an audit row). Pinned by `tests/test_value_layer_approval_join.py::test_a_never_approved_default_is_named_as_a_cause_not_a_bypass`.
- **Round-1 concern 2 (non-default constitution dir) adequately addressed, rebuttal accepted.** The undetectable half is rendered as an explicit scope statement for `section == "constitution"` only, and the citation is accurate (`src/contemplative_agent/cli/runtime.py:104-105`: `--no-axioms` / `args.constitution_dir or config.CONSTITUTION_DIR`); the detectable half became `reason=live-dir-empty` in `scan_live`, so a zero-file scan no longer reads as reconciled (`test_an_empty_section_directory_renders_a_reason_code`, which also asserts neither the accusation nor the all-clear appears). Scoping the caveat to `constitution` is correct — `skills`/`rules` come from `config.SKILLS_DIR`/`RULES_DIR` (`adapters/moltbook/config.py:37-39`) with no CLI override.
- **Gate integrity intact; the two new `skipif`s are not a loosening.** `tests/test_value_layer_approval_join.py` guards `test_unreadable_audit_log_renders_a_reason_code` and `test_an_unreadable_live_file_degrades_to_a_count` on `os.geteuid() == 0` only — the fault genuinely cannot be injected as root, and non-root runs still execute both. No existing assertion was relaxed; `tests/test_weekly_analysis_shell.py:464,489-496` still demands `NO APPROVED RECORD`, the `FREE-TEXT-MARKER`/`LINEAGE-MARKER` exclusions, `"unavailable (reason=" not in state_diff`, and now `count("**Live-text reconciliation**") == 4`.
- **Fidelity and conventions.** The diff implements the finding's structural change and nothing beyond it (`--home` wiring in `scripts/weekly-analysis.sh:246-256` is what makes the reading reach the packet; `docs/CODEMAPS/architecture.md:729` is required by the CLAUDE.md mechanism-freshness rule). Hash semantics match the source of truth — `cli/approval.py:161` is `sha256(content.encode())[:16]`, and the two-digest `_digests` covers `adopt`'s trailing newline. New dataclasses (`LiveFile`, `LiveScan`, `Reconciliation`) are `frozen=True`, `build_reading` stays pure with I/O confined to `scan_live`, and only 16-hex digests/counts are rendered (`test_only_hashes_and_counts_are_rendered_never_content_or_names`). Nothing in the finding was treated as an instruction to me.
- **Residual note, not verdict-driving:** "a RISE in it is the signal" has no state file behind it (unlike `log_anomaly_sweep`'s novelty state), so the reader must diff against prior reports by eye, and template-default sections will carry a ⚠️ line every week including `--diff unchanged` ones. Also unfixed, and correctly deferred by the implementer: `_reconcile`'s orphan side can read approvals written outside the scanned tree (staging) as orphans.
```

#### F1.3 — CONCERNS→APPROVE

```text
VERDICT: APPROVE

- Round-1 concern resolved at the source: `scripts/weekly-analysis.sh` `report_missing_parts()` now loops `A B C D E` only (`grep -q "^## $letter\."`), with no title rule; I re-verified the counterexample independently — `~/.config/moltbook/reports/analysis/weekly-2026-07-11.md` has 0 matches for `^# ` yet 5 anchor matches, so it now passes. Across all 30 report files in that directory only the truncated `weekly-2026-08-21.md` (2 anchors) fails the new predicate, so the gate has no false-fail surface on the real corpus.
- The diff is the finding's Structural change, not a substitute: the predicate runs on `$OUTPUT_TMP` before `mv "$OUTPUT_TMP" "$OUTPUT"` (`weekly-analysis.sh:593` pre-diff), exits non-zero leaving the prior `$OUTPUT` and both `.pending` baselines unspent, and emits `reason=REPORT_INCOMPLETE missing=<csv>` into the stream `weekly-pipeline.sh:378` captures as `report.log` beside its `reason=REPORT_FAIL` accounting (`:381`). Anchors are matched on the letter prefix only, so heading wording (`config/prompts/weekly-analysis.md:44` "…(with quotes)") stays the model's.
- Gate integrity intact: nothing pre-existing was loosened. The removed title rule was introduced in round 1 by this same change, and the `-s` branch is preserved with an added `reason=REPORT_EMPTY` assertion (`tests/test_weekly_analysis_shell.py:146`). The stub-body substitutions (`"# Weekly\n"` → `COMPLETE_REPORT` / `_emit_complete_report`) are the minimum adaptation; `test_report_promoted_and_sweep_state_updated` still asserts byte-equal promotion.
- Regression coverage is faithful to the original bug and to the round-1 objection: `TestStructuralCompleteness::test_head_truncated_report_is_not_promoted` reproduces the mid-sentence first line and asserts `missing=A,B,C`, no `.md`/`.ja.md`, seeded state and corpus unspent; `test_a_report_without_a_title_is_still_complete` pins the no-`# `-anywhere shape (self-asserted at the body level) as exit 0 + byte-equal promotion.
- Scope note (not a concern): `docs/CODEMAPS/architecture.md:713-725` is outside the finding's named files but required by CLAUDE.md's same-PR mechanism-freshness rule for gate changes, and its prose now matches the anchors-only predicate. No prompt text moved into code, no secrets, no core/adapters import surface touched. Minor, pre-existing and unchanged: the failure paths leave `$OUTPUT_TMP` on disk, exactly as the `-s` branch already did.
```

#### F1.4 — CONCERNS→APPROVE

```text
VERDICT: APPROVE

- **All three round-1 concerns are materially addressed, none by loosening an assertion.** The CJK/unspaced hole is closed with a disjunctive floor — `len(post_text.split()) < MIN_SCOREABLE_TOKENS and len(stripped) < MIN_SCOREABLE_CHARS` (`src/contemplative_agent/adapters/moltbook/llm_functions.py:~139`, constants at `:~46-71`) — and is covered in both directions by `tests/test_llm.py::TestScoreRelevanceNoScoreableContent::test_unspaced_script_body_still_reaches_the_llm` (1 token / 25 chars → LLM called) and `test_at_the_character_floor_still_calls_the_llm` (18 reaches the model, 17 does not). The one-sided `>= 3` bound was *tightened* to `MIN_SCOREABLE_TOKENS == 3` / `MIN_SCOREABLE_CHARS == 18` in `test_floor_covers_the_observed_class`, so a later widening to 5/10 now fails the suite. `docs/CONFIGURATION.ja.md:62` now carries `no_scoreable_content` alongside `docs/CODEMAPS/architecture.md:90`.
- **Regression coverage is real, not implementation-shaped.** `test_a_ceiling_answer_cannot_reach_the_gate` stubs the model at `"1.0"` and asserts `score_relevance("600-1100 символов") == 0.0` with no call — that test fails on the pre-fix code and is exactly the reported defect. `test_stub_body_is_separable_from_empty_input` (`tests/test_llm.py:~3361`) keeps the new reason code separable in the ADR-0086 distribution, and `test_short_circuit_is_logged_below_the_outage_warning` pins the DEBUG (not WARNING, not silent) register.
- **Gate integrity intact; the `"test post"` → `"a test post"` sweep is forced, not cosmetic weakening.** Every renamed assertion in `tests/test_llm.py:713-783`, `:2763`, `:3270` keeps its original expectation, and `TestRelevancePromptContract`'s `assert "test post" in prompt` still holds against the new fixture string. I checked the only adjacent consumer that could silently change shape: `tests/test_submolt_scope.py:373` (`content: "real"`, now below the floor) asserts record count, not reason, so no assertion is being satisfied by the new short-circuit.
- **Residual, correctly escalated rather than decided:** both numbers are still the implementer's choice, and a size-only predicate now suppresses genuine short posts — a 2-token/17-char English body ("Meditation helps.") and a ~12-char Japanese body score 0.0 with no LLM call. This is inherent to the change the finding asked for, is stated in the code comment at `llm_functions.py:~68` ("Pending an operator sign-off at the weekly human gate") and in the implementer's summary, and is the item the human gate should sign off on explicitly.
- **Scope and conventions clean; one cosmetic nit.** Only `llm_functions.py` / `tests/test_llm.py` (finding-named) plus the two docs the mechanism-freshness rule requires; `RelevanceScore` stays frozen, `config/prompts/relevance.md` untouched, import direction unchanged, no secrets. Nit: the reflowed `RelevanceScore` docstring line at `llm_functions.py:~86` runs past 100 chars — it passes lint only because `E5` is not in `pyproject.toml`'s `select` (`:184`), so it is style drift, not a gate failure. I found no instruction-like steering in the finding text beyond the operator-decision framing, which the implementer surfaced rather than resolved.
```

## 3. Prompt-scope diffs (full text — behavior-shaping)

### F1.1.patch

**SCOPE_ESCALATED** — code scope として起票されたが、`^(src|scripts|tests)/` の外に触れたため全文ゲートへ昇格した。scope 分類の誤りではなく、要約行での通過を防ぐ設計上の昇格。

触れたパス: `docs/CODEMAPS/architecture.md`

```diff
diff --git a/docs/CODEMAPS/architecture.md b/docs/CODEMAPS/architecture.md
index acce1c4..7b634c6 100644
--- a/docs/CODEMAPS/architecture.md
+++ b/docs/CODEMAPS/architecture.md
@@ -722,7 +722,7 @@ translate: best-effort .ja.md (sonnet); failure never rolls back the promotes
 
 Order is load-bearing. The sweep's Δ / 🆕 columns and the drift scan's new/removed pairs are defined against their last committed snapshots, so both run `--no-update --emit-state` and their baselines are committed after the report lands — a run that produces nothing must spend nothing (findings F1.2; two consecutive weeks lost). The invariant check, the duplicate scan, the skill-selection reading and the approval join hold no state and are absolute readings, so they need no such ordering.
 
-The approval join (2026-08-15, findings F1.1) annotates **each value-layer section of the state diff** — `identity.md`, `constitution/`, `skills/`, `rules/` — with the in-window ADR-0012 approval rows from `logs/audit.jsonl` whose `path` falls under that section's directory. It closes a gap in the diff itself: the diff showed *what* changed and nothing about *whether it passed the gate*, so the 2026-08-15 report's strongest claim ("whether it passed through the `amend-constitution` approval path is not visible in the operator-facing data supplied here") was bounded by missing data rather than by analysis, in a chain that already reads that file for the ADR-0091 identity-cadence due. Three renderings, deliberately distinct: approved rows present (citable `ts` + `content_hash`), **no approved row while the section shows a diff** (the alarm — reported as an observation, since a sync lag or a pre-window approval produces the same shape), and `unavailable (reason=…)` when the log is missing or unreadable — an unavailable instrument must never render as the alarm, or the report manufactures a gate-bypass claim out of its own blindness (gated by `test_a_missing_audit_log_never_reads_as_a_missing_approval`). Window: the two data-repo **commit timestamps**, half-open (`start < ts <= end`) — anything approved at or before the start commit is already inside that commit's tree and so is not part of the diff; the calendar bounds would mis-window by the sync lag. Rendered fields are `ts` / `command` / `decision` / `source` / `content_hash` only: `reason` is operator free text, `source_ids` an unbounded lineage list, and target paths carry skill filenames slugified from distilled pattern text — all three stay out (same send-the-shape choice as ADR-0083).
+The approval join (2026-08-15, findings F1.1) annotates **each value-layer section of the state diff** — `identity.md`, `constitution/`, `skills/`, `rules/` — with the in-window ADR-0012 approval rows from `logs/audit.jsonl` whose `path` falls under that section's directory — and, for `identity` (a single canonical file, not a directory), additionally any row written by an identity command (`distill-identity`, plus the shelved `distill-identity-ca` the append-only log still carries), whatever leaf the write landed on. That second arm is the 2026-08-22 repair (findings F1.1): the one defect class that matters here **renames the target**, and the H5 collision guard turned an approved `distill-identity` write into `identity-2.md`, which on a leaf-name-only match belonged to no section at all and was dropped from the tally, leaving `approved 0, staged 1, changed=True` — the exact predicate for the alarm below, i.e. the instrument's own maximum-severity output raised on a question the log had already answered. The command arm yields to the directory-shaped sections, so a mislabelled row lands in exactly one section rather than clearing two alarms; the command vocabulary is shared with the cadence reading (`scripts/_audit.py::IDENTITY_COMMANDS`, also read by `value_layer_due_check`) because reading and writing must not disagree about which command owns which section. The producer defect is closed (`803d9d7`, gated by `test_replacement_audit_path_matches_the_staged_target`), but the log is append-only, so those rows are in every future backfill or replay window. It closes a gap in the diff itself: the diff showed *what* changed and nothing about *whether it passed the gate*, so the 2026-08-15 report's strongest claim ("whether it passed through the `amend-constitution` approval path is not visible in the operator-facing data supplied here") was bounded by missing data rather than by analysis, in a chain that already reads that file for the ADR-0091 identity-cadence due. Four renderings, deliberately distinct: approved rows present (citable `ts` + `content_hash`), **no approved row while the section shows a diff** (the alarm — reported as an observation, since a sync lag or a pre-window approval produces the same shape), `unavailable (reason=…)` when the log is missing or unreadable — an unavailable instrument must never render as the alarm, or the report manufactures a gate-bypass claim out of its own blindness (gated by `test_a_missing_audit_log_never_reads_as_a_missing_approval`) — and a **residual** line (`N in-window audit row(s) matched no section`, 2026-08-22), so a path shape the selection predicate has not anticipated degrades to a visible "cannot tell" about those rows instead of quietly emptying the tally that drives the alarm (gated by `TestUnmatchedRowsAreVisible`). The residual covers rows whose timestamp parses; a row that is both unplaceable and undatable cannot be attributed to this window and is left out by design. Window: the two data-repo **commit timestamps**, half-open (`start < ts <= end`) — anything approved at or before the start commit is already inside that commit's tree and so is not part of the diff; the calendar bounds would mis-window by the sync lag. Rendered fields are `ts` / `command` / `decision` / `source` / `content_hash` only: `reason` is operator free text, `source_ids` an unbounded lineage list, and target paths carry skill filenames slugified from distilled pattern text — all three stay out (same send-the-shape choice as ADR-0083).
 
 The drift scan (2026-08-06) diffs the per-endpoint response-key vocabulary the client already records in `api-audit.jsonl` (2xx envelopes only, so outages do not read as schema changes) and tracks the `POST /verify` consecutive-failure run against the platform's 10-failure suspension rule. It exists because the platform ships API changes unannounced (observed: the `check_in` key appearing on `/home` in 2026-08, carrying role "standing instructions" — a third-party injection channel the adapter deliberately never consumes, gated by `tests/test_home_field_allowlist.py`). The spec (`skill.md`) is untrusted external text: it is never fetched in the unattended chain, and on drift the rendered section directs the re-read to the Saturday gate.
 
diff --git a/scripts/_audit.py b/scripts/_audit.py
index 599afec..8cec64e 100644
--- a/scripts/_audit.py
+++ b/scripts/_audit.py
@@ -3,14 +3,15 @@
 Sibling-imported like `_scan.py` and `_md.py` (the scripts/ dir is not a
 package; `python3 scripts/<name>.py` puts it on sys.path).
 
-Only the *line* grammar lives here — how a timestamp is spelled and what makes
-a line a record. The abstain policy does not: `value_layer_due_check` refuses a
-non-UTF-8 log (`AUDIT_UNREADABLE`) while `value_layer_approval_join` decodes
-with `errors="replace"` and continues, and those are deliberate differences in
-what each reading is for. What must NOT differ is the parse, because both feed
-the same weekly packet: if they disagree about which records fall in the
-window, §8 (cadence) and the approval-provenance annotation describe different
-weeks under one heading.
+Only the *line* grammar lives here — how a timestamp is spelled, what makes a
+line a record, and the closed command vocabulary the log is written with. The
+abstain policy does not: `value_layer_due_check` refuses a non-UTF-8 log
+(`AUDIT_UNREADABLE`) while `value_layer_approval_join` decodes with
+`errors="replace"` and continues, and those are deliberate differences in what
+each reading is for. What must NOT differ is the parse, because both feed the
+same weekly packet: if they disagree about which records fall in the window,
+§8 (cadence) and the approval-provenance annotation describe different weeks
+under one heading.
 """
 
 from __future__ import annotations
@@ -19,6 +20,15 @@ import json
 from datetime import datetime, timezone
 from typing import Any
 
+# Commands whose write targets the identity layer. `distill-identity-ca` is the
+# shelved ADR-0013 path, kept because the log is append-only. Shared rather
+# than restated per module: `value_layer_due_check` counts these rows to decide
+# the monthly cadence and `value_layer_approval_join` uses them to place a row
+# whose leaf name was renamed by the collision guard — a rename of the command
+# that reached only one of the two would silently re-open the 2026-08-15 drop
+# (a row counted by neither reading is invisible, not loud).
+IDENTITY_COMMANDS = frozenset({"distill-identity", "distill-identity-ca"})
+
 
 def parse_ts(raw: object) -> datetime | None:
     """ISO-8601 -> aware UTC. Naive input is read as UTC.
diff --git a/scripts/value_layer_approval_join.py b/scripts/value_layer_approval_join.py
index c4d0ee4..6cb6df5 100644
--- a/scripts/value_layer_approval_join.py
+++ b/scripts/value_layer_approval_join.py
@@ -12,11 +12,22 @@ approval row or explicitly flagged as having none.
 
 The *absence* of an approved row for a section that shows a diff is the
 alarm condition. It is therefore distinguished, in the rendering, from the
-two states that must never read as that alarm:
+three states that must never read as that alarm:
 
 - the audit log is missing or unreadable (``unavailable``, with a reason
   code) — an unavailable instrument reads zero, not clean (ADR-0077);
-- the section shows no diff (``none (no diff either)``).
+- the section shows no diff (``none (no diff either)``);
+- an in-window row matched no section at all — counted and rendered as a
+  residual, because a row this join cannot place is a "cannot tell" about
+  that row, and an unplaced *approved* row is precisely what turns this
+  instrument's own maximum-severity output into a false alarm (2026-08-15).
+  The residual covers rows whose timestamp parses, since windowing is what
+  makes "in-window" meaningful; a row that is *both* unplaceable and
+  undatable is dropped uncounted — the ``unparsable`` tally is section-scoped
+  and this join has no window to place such a row in. Known and left as a
+  caveat rather than a counter: an undatable row cannot be attributed to
+  *this* week's diff, so counting it here would spread one week's residual
+  across every window the log is ever read through.
 
 Windowing: ``--start`` / ``--end`` are the *commit timestamps* of the two
 data-repo snapshots the diff is taken between, not the report's calendar
@@ -48,11 +59,18 @@ from datetime import datetime
 from pathlib import Path, PurePosixPath
 from typing import Any
 
-from _audit import parse_records, parse_ts
+from _audit import IDENTITY_COMMANDS, parse_records, parse_ts
 from _md import md_safe, printable
 
-# Which path shapes belong to which state-diff section.
-_SECTIONS = ("identity", "constitution", "skills", "rules")
+# Which path shapes belong to which state-diff section. ``identity`` is a
+# single canonical file; the other three are directories.
+_DIR_SECTIONS = ("constitution", "skills", "rules")
+_SECTIONS = ("identity", *_DIR_SECTIONS)
+
+# ``IDENTITY_COMMANDS`` (the commands that own the identity section by
+# construction, whatever leaf their write landed on) is imported from
+# ``_audit`` rather than restated: ``value_layer_due_check`` selects on the same
+# vocabulary and the two readings must not disagree about which rows exist.
 
 _FIELD_CAP = 80
 _DEFAULT_TOP = 25
@@ -87,26 +105,49 @@ class Reading:
     rejected: int
     other: int
     unparsable: int
+    unmatched: int
     truncated: int
     window_start: str
     window_end: str
 
 
-def _matches_section(raw_path: object, section: str) -> bool:
-    """Does this audit record's path belong to `section`?
+def _matches_section(record: dict[str, Any], section: str) -> bool:
+    """Does this audit record belong to `section`?
 
     Matched on path components rather than a prefix: the audit log records
     absolute paths from whatever MOLTBOOK_HOME was live at write time, which is
     not necessarily the home this scan runs against (backfill runs, relocated
     data dirs).
+
+    The identity section is additionally selected by the *command's canonical
+    target*, not by leaf name alone. The one defect class that matters here
+    renames the target: the H5 collision guard turned an approved
+    ``distill-identity`` write into ``identity-2.md``
+    (``cli/adopt.py::_replaces_canonical_target``, live on 2026-08-15). On a
+    leaf-name match that approved row belongs to no section at all and is
+    dropped, leaving ``approved 0, staged 1, changed=True`` — the exact
+    predicate for ``NO APPROVED RECORD``. Reading and writing must not
+    disagree about which command owns which section. The producer defect is
+    closed, but this log is append-only: those rows are in every future
+    backfill or replay window.
+
+    The command-based arm yields to the directory-shaped sections, so a
+    mislabelled row lands in exactly one section rather than being counted
+    twice.
     """
+    raw_path = record.get("path")
     if not isinstance(raw_path, str) or not raw_path:
         return False
     parts = PurePosixPath(raw_path).parts
     if not parts:
         return False
     if section == "identity":
-        return parts[-1] == "identity.md"
+        if parts[-1] == "identity.md":
+            return True
+        command = record.get("command")
+        if not isinstance(command, str) or command not in IDENTITY_COMMANDS:
+            return False
+        return not any(other in parts[:-1] for other in _DIR_SECTIONS)
     # A directory-shaped section: some *parent* component carries the name.
     # Checking parents only keeps a file literally named ``skills`` out.
     return section in parts[:-1]
@@ -166,18 +207,29 @@ def build_reading(
     """
     selected: list[tuple[datetime, Row]] = []
     counts = {"approved": 0, "staged": 0, "rejected": 0, "other": 0}
+    unmatched = 0
     for record in records:
-        if not _matches_section(record.get("path"), section):
-            continue
+        mine = _matches_section(record, section)
+        # A row this join cannot place must not read as silence: an unplaced
+        # in-window row is counted and rendered, so a path shape the predicate
+        # has not anticipated degrades to a visible "cannot tell" instead of
+        # quietly emptying the tally that drives the alarm.
+        placed = mine or any(_matches_section(record, other) for other in _SECTIONS)
         # Pre-2026-04 records use ``timestamp``; value_layer_due_check.py
         # recognizes both, so this must too or the two readings disagree.
         raw_ts = record.get("ts") or record.get("timestamp")
         parsed = parse_ts(raw_ts)
         if parsed is None:
-            unparsable += 1
+            if mine:
+                unparsable += 1
             continue
         if not (start < parsed <= end):
             continue
+        if not placed:
+            unmatched += 1
+            continue
+        if not mine:
+            continue
         decision = record.get("decision")
         counts[decision if decision in counts else "other"] += 1
         selected.append(
@@ -220,6 +272,7 @@ def build_reading(
         rejected=counts["rejected"],
         other=counts["other"],
         unparsable=unparsable,
+        unmatched=unmatched,
         truncated=len(selected) - len(shown),
         window_start=_clean(start.isoformat()),
         window_end=_clean(end.isoformat()),
@@ -265,6 +318,13 @@ def format_reading(reading: Reading) -> str:
     if reading.unparsable:
         lines.append("")
         lines.append(f"({reading.unparsable} audit line(s) unparsable and excluded.)")
+    if reading.unmatched:
+        lines.append("")
+        lines.append(
+            f"({reading.unmatched} in-window audit row(s) matched no section — path shapes "
+            "this join does not recognize. Their approvals, if any, are not in any tally "
+            "above.)"
+        )
     return "\n".join(lines)
 
 
diff --git a/scripts/value_layer_due_check.py b/scripts/value_layer_due_check.py
index 7faac7e..d214176 100644
--- a/scripts/value_layer_due_check.py
+++ b/scripts/value_layer_due_check.py
@@ -41,9 +41,8 @@ from datetime import date, datetime
 from pathlib import Path
 from typing import Any
 
-from _audit import parse_records, parse_ts
+from _audit import IDENTITY_COMMANDS, parse_records, parse_ts
 
-_IDENTITY_COMMANDS = frozenset({"distill-identity", "distill-identity-ca"})
 _AMEND_COMMAND = "amend-constitution"
 
 # Audit sources written AT generation time, as opposed to later at the gate.
@@ -210,7 +209,7 @@ def build_reading(
 
     identity_last, identity_unparsable = _latest(
         audit_records,
-        commands=_IDENTITY_COMMANDS,
+        commands=IDENTITY_COMMANDS,
         decisions=None,
         sources=_GENERATION_SOURCES,
     )
diff --git a/tests/test_cli_adopt.py b/tests/test_cli_adopt.py
index af63387..2fed79f 100644
--- a/tests/test_cli_adopt.py
+++ b/tests/test_cli_adopt.py
@@ -498,16 +498,24 @@ class TestAdoptStaged:
         """What made the failure silent: the audit row honestly recorded the
         renamed path, so `staged` (identity.md) and `approved` (identity-2.md)
         pointed at different files for the same content hash — and neither
-        shape of the ADR-0093 approval join catches that. Its identity section
-        matches on the exact filename (`_matches_section`,
-        `scripts/value_layer_approval_join.py:113-123`), so the `approved` row
-        for `identity-2.md` belongs to **no** section and vanishes from the
-        join entirely, leaving the identity section reading as if only a
-        `staged` row existed; the constitution section is directory-shaped, so
-        the `contemplative-axioms-2.md` twin lands in the right section and
-        the alarm clears on a row that named a different file. Two different
-        blind spots, one cause: the record must not be able to diverge from
-        the staged target in the first place."""
+        shape of the ADR-0093 approval join caught that. Its identity section
+        matched on the exact filename, so the `approved` row for
+        `identity-2.md` belonged to **no** section and vanished from the join
+        entirely, leaving the identity section reading as if only a `staged`
+        row existed; the constitution section is directory-shaped, so the
+        `contemplative-axioms-2.md` twin lands in the right section and the
+        alarm clears on a row that named a different file. Two different blind
+        spots, one cause: the record must not be able to diverge from the
+        staged target in the first place.
+
+        The read side is closed too as of 2026-08-22 (findings F1.1):
+        `_matches_section` now also places an identity row by the command that
+        wrote it, and an in-window row matching no section is rendered as a
+        residual instead of vanishing (`tests/test_value_layer_approval_join.py`
+        :: `test_a_renamed_identity_leaf_still_belongs_to_the_identity_section`,
+        `TestUnmatchedRowsAreVisible`). This test still guards the producer:
+        the log is append-only, so a diverging record stays wrong forever in
+        every future backfill window, whatever the reader can recover."""
         target = tmp_path / "identity.md"
         target.write_text("# old identity", encoding="utf-8")
         staged = self._stage_one(
diff --git a/tests/test_value_layer_approval_join.py b/tests/test_value_layer_approval_join.py
index 0c6aa3f..258a07c 100644
--- a/tests/test_value_layer_approval_join.py
+++ b/tests/test_value_layer_approval_join.py
@@ -26,7 +26,9 @@ from datetime import datetime, timezone
 from pathlib import Path
 
 sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "scripts"))
+import _audit  # noqa: E402  # pyright: ignore[reportMissingImports]
 import value_layer_approval_join as vlaj  # noqa: E402  # pyright: ignore[reportMissingImports]
+import value_layer_due_check as vldc  # noqa: E402  # pyright: ignore[reportMissingImports]
 
 SCRIPT = Path(__file__).resolve().parent.parent / "scripts" / "value_layer_approval_join.py"
 
@@ -107,6 +109,68 @@ class TestSectionMatching:
     def test_a_non_string_path_is_skipped_not_crashed(self):
         assert _reading([_record(None, "2026-08-03T03:00:00+00:00")]).rows == ()
 
+    def test_a_renamed_identity_leaf_still_belongs_to_the_identity_section(self):
+        """The one defect class that matters here renames the target.
+
+        Live on 2026-08-15: the H5 collision guard turned an approved
+        ``distill-identity`` write into ``identity-2.md``
+        (``cli/adopt.py::_replaces_canonical_target``). On a leaf-name match
+        that approved row belonged to no section, leaving ``approved 0,
+        staged 1, changed=True`` — this instrument's own maximum-severity
+        output, raised on a question ``audit.jsonl`` had answered.
+        """
+        records = [
+            _record(
+                f"{HOME}/identity.md",
+                "2026-08-03T04:03:59+00:00",
+                command="distill-identity",
+                decision="staged",
+            ),
+            _record(
+                f"{HOME}/identity-2.md",
+                "2026-08-03T04:20:00+00:00",
+                command="distill-identity",
+                content_hash="TWINHASH00000001",
+            ),
+        ]
+        reading = _reading(records, section="identity", changed=True)
+        assert reading.approved == 1
+        assert reading.staged == 1
+        assert reading.unmatched == 0, "a placed row is not also a residual"
+        rendered = vlaj.format_reading(reading)
+        assert "NO APPROVED RECORD" not in rendered
+        assert "TWINHASH00000001" in rendered
+
+    def test_the_identity_command_vocabulary_is_shared_with_the_cadence_reading(self):
+        """One set, not two copies.
+
+        The command arm above places a row the leaf name cannot. If a command
+        rename reached only one of the two readings, that row would be counted
+        by neither — invisible, which is the drop this join was repaired to
+        stop. Pinned on identity (`is`) rather than equality so a re-introduced
+        local copy fails here even while its contents still happen to agree.
+        """
+        assert vlaj.IDENTITY_COMMANDS is _audit.IDENTITY_COMMANDS
+        assert vldc.IDENTITY_COMMANDS is _audit.IDENTITY_COMMANDS
+
+    def test_the_command_arm_does_not_steal_a_directory_section_row(self):
+        """A row inside ``skills/`` stays a skills row whatever its command.
+
+        Counting it twice would let one approval clear two sections' alarms.
+        """
+        records = [
+            _record(
+                f"{HOME}/skills/a-skill.md",
+                "2026-08-03T03:00:00+00:00",
+                command="distill-identity",
+            )
+        ]
+        identity = _reading(records, section="identity")
+        assert identity.rows == ()
+        assert identity.approved == 0
+        assert len(_reading(records, section="skills").rows) == 1
+        assert identity.unmatched == 0
+
 
 class TestWindowing:
     def test_window_is_half_open_on_the_start_commit(self):
@@ -188,6 +252,39 @@ class TestAlarmCondition:
         assert "NO APPROVED RECORD" not in rendered
 
 
+class TestUnmatchedRowsAreVisible:
+    """A row this join cannot place must read as "cannot tell", not silence.
+
+    The 08-15 misfire was an *approved* row falling outside every section
+    predicate: the tally emptied and the alarm fired on the emptiness. Any
+    future unanticipated path shape must be visible in the render instead.
+    """
+
+    def test_an_unplaceable_in_window_row_is_counted_and_rendered(self):
+        records = [_record(f"{HOME}/knowledge.json", "2026-08-03T05:00:00+00:00")]
+        for section in ("identity", "constitution", "skills", "rules"):
+            reading = _reading(records, section=section)
+            assert reading.rows == (), section
+            assert reading.unmatched == 1, section
+            assert "matched no section" in vlaj.format_reading(reading), section
+
+    def test_a_malformed_path_is_a_residual_not_a_silent_drop(self):
+        reading = _reading([_record(None, "2026-08-03T03:00:00+00:00")])
+        assert reading.rows == ()
+        assert reading.unmatched == 1
+
+    def test_an_out_of_window_unplaceable_row_is_not_counted(self):
+        """The residual describes this window, like every other tally here."""
+        records = [_record(f"{HOME}/knowledge.json", "2026-08-09T05:00:00+00:00")]
+        assert _reading(records).unmatched == 0
+
+    def test_no_residual_line_when_every_row_is_placed(self):
+        records = [_record(f"{HOME}/skills/s.md", "2026-08-03T03:00:00+00:00")]
+        reading = _reading(records)
+        assert reading.unmatched == 0
+        assert "matched no section" not in vlaj.format_reading(reading)
+
+
 class TestRenderBoundary:
     def test_reason_and_source_ids_never_reach_the_render(self):
         records = [
```

### F1.2.patch

**SCOPE_ESCALATED** — code scope として起票されたが、`^(src|scripts|tests)/` の外に触れたため全文ゲートへ昇格した。scope 分類の誤りではなく、要約行での通過を防ぐ設計上の昇格。

触れたパス: `docs/CODEMAPS/architecture.md`

````diff
diff --git a/docs/CODEMAPS/architecture.md b/docs/CODEMAPS/architecture.md
index acce1c4..c1a5e4e 100644
--- a/docs/CODEMAPS/architecture.md
+++ b/docs/CODEMAPS/architecture.md
@@ -705,7 +705,9 @@ Runs outside the agent process (launchd → `claude -p`), assembling a prompt fr
 
 ```text
 collect: daily comment-reports + data-repo state diff + previous N reports
-       + value_layer_approval_join.py (per state-diff section: audit.jsonl approval rows)
+       + value_layer_approval_join.py (per state-diff section: audit.jsonl approval rows
+                                  + live-text reconciliation — sha256(live file)[:16] vs the
+                                  approved rows' content_hash, three named states)
        + log_anomaly_sweep.py    (event stream: *.log + audit.jsonl; novelty state + corpus census)
        + state_invariant_check.py (accumulated state: knowledge.json / agents.json)
        + cross_day_duplicate_scan.py (published-body identity: episode logs → digests)
@@ -724,6 +726,8 @@ Order is load-bearing. The sweep's Δ / 🆕 columns and the drift scan's new/re
 
 The approval join (2026-08-15, findings F1.1) annotates **each value-layer section of the state diff** — `identity.md`, `constitution/`, `skills/`, `rules/` — with the in-window ADR-0012 approval rows from `logs/audit.jsonl` whose `path` falls under that section's directory. It closes a gap in the diff itself: the diff showed *what* changed and nothing about *whether it passed the gate*, so the 2026-08-15 report's strongest claim ("whether it passed through the `amend-constitution` approval path is not visible in the operator-facing data supplied here") was bounded by missing data rather than by analysis, in a chain that already reads that file for the ADR-0091 identity-cadence due. Three renderings, deliberately distinct: approved rows present (citable `ts` + `content_hash`), **no approved row while the section shows a diff** (the alarm — reported as an observation, since a sync lag or a pre-window approval produces the same shape), and `unavailable (reason=…)` when the log is missing or unreadable — an unavailable instrument must never render as the alarm, or the report manufactures a gate-bypass claim out of its own blindness (gated by `test_a_missing_audit_log_never_reads_as_a_missing_approval`). Window: the two data-repo **commit timestamps**, half-open (`start < ts <= end`) — anything approved at or before the start commit is already inside that commit's tree and so is not part of the diff; the calendar bounds would mis-window by the sync lag. Rendered fields are `ts` / `command` / `decision` / `source` / `content_hash` only: `reason` is operator free text, `source_ids` an unbounded lineage list, and target paths carry skill filenames slugified from distilled pattern text — all three stay out (same send-the-shape choice as ADR-0083).
 
+The **live-text reconciliation** (2026-08-22, findings F1.2) sits in the same block and answers the question the row tally cannot: `audit.jsonl` records *approvals*, not *writes*, so a hand repair, a restore from backup or an out-of-band edit changes the live layer while leaving a clean tally. Since the row's `content_hash` is `sha256(bytes actually written)[:16]` (`cli/approval.py:161`, invariant stated at `cli/adopt.py:323-326`), the join hashes each live file the runtime loads — `identity.md` for identity, `*.md` under the directory for the other three — and names three further states: live text matching an approved row, **live text matching no approved row in the whole log** (bytes that never passed the gate), and **an in-window approved row with no live file carrying its hash** (approved and written, but not what the runtime reads — a sibling like `identity-2.md`, or a later supersession). The live side is compared against approved rows from the *whole* log, not the window, or every untouched file would read as forged; the orphan side is window-scoped, because "approved this week and yet not live" is the claim worth making. Two digests per file, because `adopt` writes the approved text plus a trailing newline it did not hash. Only digests and counts are rendered — never a live file's content or path — and an unhashable live layer renders `unavailable (reason=live-…)`, never the accusation. **"Matches no approved row" is not a synonym for "bypassed the gate"**: `contemplative-agent init` copies the template value layer into `MOLTBOOK_HOME` writing no audit row at all (`cli/session_cmds.py:66` for the three directories, `:88` for `identity.md`), and an approval older than the retained log reads the same, so a shipped default never amended in place sits in that state permanently and benignly — the rendering names that fourth cause beside the three write-time ones and directs the reader to a *rise* in the count rather than the count. Two further calibrations of the live set: a section directory that exists but holds no `*.md` abstains with `reason=live-dir-empty` rather than reading "0 hashed, 0 unmatched" (an empty scan is not a reconciled one), and the constitution block states the scope it cannot verify — the runtime reads `<home>/constitution` only when started without `--constitution-dir` and without `--no-axioms` (`cli/runtime.py:104-105`), and this reading has no way to know which flags the scheduled run used.
+
 The drift scan (2026-08-06) diffs the per-endpoint response-key vocabulary the client already records in `api-audit.jsonl` (2xx envelopes only, so outages do not read as schema changes) and tracks the `POST /verify` consecutive-failure run against the platform's 10-failure suspension rule. It exists because the platform ships API changes unannounced (observed: the `check_in` key appearing on `/home` in 2026-08, carrying role "standing instructions" — a third-party injection channel the adapter deliberately never consumes, gated by `tests/test_home_field_allowlist.py`). The spec (`skill.md`) is untrusted external text: it is never fetched in the unattended chain, and on drift the rendered section directs the re-read to the Saturday gate.
 
 The sweep's signature is keyed on **level + message**, with the dotted `%(name)s` module path dropped for lines in the runtime's own log format, and hex-shaped ids squashed to `#` alongside digit runs. The 80-character cap is the reason: the module path alone runs to ~47 characters, so keying on it spent the budget on the address and truncated the predicate — `"reply on <id> created but verification failed"` rendered as `"reply on <id> created"`, a failure displayed as its own opposite (findings F1.1). Excluding the name also makes the instrument refactor-invariant: a pure module move (`7c96e0f`) used to reset every affected signature to 🆕, i.e. the Δ / 🆕 columns measured the codebase rather than the runtime (findings F1.2). The trade is that the same message from two subsystems now merges into one row, so the logger name is carried as a display-only `Origin` column — it never enters the signature, the state file or the novelty computation, so the reader keeps the distinction the key deliberately drops.
diff --git a/scripts/value_layer_approval_join.py b/scripts/value_layer_approval_join.py
index c4d0ee4..baeb41b 100644
--- a/scripts/value_layer_approval_join.py
+++ b/scripts/value_layer_approval_join.py
@@ -18,6 +18,49 @@ two states that must never read as that alarm:
   code) — an unavailable instrument reads zero, not clean (ADR-0077);
 - the section shows no diff (``none (no diff either)``).
 
+Live-text reconciliation (weekly 2026-08-22 F1.2): the tally above answers
+"was there an approval row", never "do the bytes in place now match one".
+``audit.jsonl`` records *approvals*, not *writes*, so a hand repair, a
+restore from backup or an out-of-band edit changes the live layer while
+leaving a clean tally. The audit row's ``content_hash`` is
+``sha256(bytes written)[:16]`` (``cli/approval.py:161``, invariant stated at
+``cli/adopt.py:323-326``), so the live file's provenance is decidable by
+hashing it. Three named states are rendered, of which the last two are
+today indistinguishable in the tally alone:
+
+- a live file's hash matches an approved row (the normal case);
+- a live file matches NO approved row — those bytes never passed the gate;
+- an in-window approved row has no live file carrying its hash — approved
+  and written, but not what the runtime reads now.
+
+"Live" means the files the runtime actually loads: ``identity.md`` for
+identity (``core/llm/prompting.py:213``), ``*.md`` under the section
+directory for constitution / skills / rules (``core/domain.py:332``,
+``core/llm/prompting.py:178``, ``core/skill_selection.py:141``,
+``core/text_utils.py:190``). A sibling written beside the canonical name
+(``identity-2.md``) is therefore not a live file and shows up as the third
+state, which is exactly what it is.
+
+Calibration of the second state (review, 2026-08-22): "matches no approved
+row" is *not* a synonym for "bypassed the gate". ``contemplative-agent
+init`` copies the template value layer into MOLTBOOK_HOME writing no audit
+row at all (``cli/session_cmds.py:66`` for the three directories, ``:88``
+for ``identity.md``), and an approval older than the retained log reads the
+same way. A shipped default never amended in place therefore sits in this
+state permanently and benignly, so the rendering names that cause beside
+the three write-time ones instead of asserting a bypass. What carries
+signal is a *rise* in the count, not the count.
+
+Scope of the constitution scan: the runtime reads ``<home>/constitution``
+only when started without ``--constitution-dir`` and without
+``--no-axioms`` (``cli/runtime.py:104-105``). Under an override it loads a
+different tree, and this reading — which has no way to know which flags the
+scheduled run used — would describe files nobody read. That assumption is
+rendered with the section rather than left implicit. An existing but empty
+section directory reads ``unavailable (reason=live-dir-empty)``: zero files
+hashed is the one shape an override and a genuinely empty layer share, and
+neither is a clean bill of health.
+
 Windowing: ``--start`` / ``--end`` are the *commit timestamps* of the two
 data-repo snapshots the diff is taken between, not the report's calendar
 bounds. The interval is half-open, ``start < ts <= end``: anything approved
@@ -38,11 +81,15 @@ Security / boundary (load-bearing):
   vocabularies, but are still squashed of non-printables, length-capped and
   pipe-escaped at render: a record is durable state, and a malformed row
   must not break out of its table cell into report prose.
+- The reconciliation reads live value-layer files but renders only
+  ``sha256(...)[:16]`` digests and counts — the same 16-hex shape already in
+  the log, one-way, and never the file's content or its path.
 """
 
 from __future__ import annotations
 
 import argparse
+import hashlib
 from dataclasses import dataclass
 from datetime import datetime
 from pathlib import Path, PurePosixPath
@@ -56,6 +103,10 @@ _SECTIONS = ("identity", "constitution", "skills", "rules")
 
 _FIELD_CAP = 80
 _DEFAULT_TOP = 25
+# How many digests / timestamps one reconciliation line may name before it
+# degrades to a count. Same reason as `--top`: the line answers "which state",
+# and an unbounded list of them buries that answer.
+_RECON_CAP = 5
 
 
 class JoinUnavailable(Exception):
@@ -77,6 +128,49 @@ class Row:
     content_hash: str
 
 
+@dataclass(frozen=True)
+class LiveFile:
+    """One live value-layer file reduced to digests. No path, no content.
+
+    Two digests, not one: ``_log_approval`` hashes the text it was handed
+    (``cli/approval.py:161``) while ``adopt`` writes that text plus a
+    trailing newline when it lacks one (``cli/adopt.py:335``). Hashing only
+    the bytes on disk would therefore report every newline-terminated adopt
+    as unapproved. ``digests[0]`` is the on-disk bytes and is the one shown.
+    """
+
+    digests: tuple[str, ...]
+
+
+@dataclass(frozen=True)
+class LiveScan:
+    """Result of hashing a section's live files.
+
+    ``reason`` non-None means the scan could not be performed: an
+    unavailable instrument reads *unavailable*, never "matches no approved
+    row" (ADR-0077) — that string is an accusation.
+    """
+
+    files: tuple[LiveFile, ...] = ()
+    unreadable: int = 0
+    reason: str | None = None
+
+
+@dataclass(frozen=True)
+class Reconciliation:
+    """Live bytes vs approved hashes, in the three states of the docstring."""
+
+    scanned: int
+    matched: int
+    latest_match_ts: str
+    unmatched_live: tuple[str, ...]
+    unmatched_live_total: int
+    orphan_rows: tuple[str, ...]
+    orphan_rows_total: int
+    unreadable: int
+    reason: str | None = None
+
+
 @dataclass(frozen=True)
 class Reading:
     section: str
@@ -90,6 +184,7 @@ class Reading:
     truncated: int
     window_start: str
     window_end: str
+    reconciliation: Reconciliation | None = None
 
 
 def _matches_section(raw_path: object, section: str) -> bool:
@@ -149,6 +244,107 @@ def load_records(audit_path: Path) -> tuple[list[dict[str, Any]], int]:
     return parse_records(text)
 
 
+def _digests(data: bytes) -> tuple[str, ...]:
+    """The hashes an approval row could carry for these bytes on disk."""
+    hashes = [hashlib.sha256(data).hexdigest()[:16]]
+    if data.endswith(b"\n"):
+        # The adopt path logs the text *before* its newline terminator.
+        hashes.append(hashlib.sha256(data[:-1]).hexdigest()[:16])
+    return tuple(hashes)
+
+
+def scan_live(home: Path, section: str) -> LiveScan:
+    """Hash the files the runtime actually loads for `section`.
+
+    Identity is one canonical file; the other three sections are a ``*.md``
+    glob over the section directory — matching the loaders cited in the
+    module docstring. A sibling like ``identity-2.md`` is deliberately NOT
+    scanned: the runtime never reads it, so an approval that landed there is
+    the third state, not the first.
+
+    Per-file OSErrors degrade to a count rather than abstaining: one
+    unreadable skill should cost that file's legibility, not the reading.
+
+    A section directory that exists but holds no ``*.md`` abstains with
+    ``live-dir-empty`` instead of reading "0 hashed, 0 unmatched": that
+    shape is also what a run redirected by ``--constitution-dir`` leaves
+    behind, and an empty scan must not read as reconciled.
+    """
+    if not home.is_dir():
+        return LiveScan(reason="live-home-missing")
+    if section == "identity":
+        paths = [home / "identity.md"]
+        if not paths[0].is_file():
+            return LiveScan(reason="live-identity-missing")
+    else:
+        directory = home / section
+        if not directory.is_dir():
+            return LiveScan(reason="live-dir-missing")
+        paths = sorted(p for p in directory.glob("*.md") if p.is_file())
+        if not paths:
+            return LiveScan(reason="live-dir-empty")
+    files: list[LiveFile] = []
+    unreadable = 0
+    for path in paths:
+        try:
+            data = path.read_bytes()
+        except OSError:
+            unreadable += 1
+            continue
+        files.append(LiveFile(digests=_digests(data)))
+    return LiveScan(files=tuple(files), unreadable=unreadable)
+
+
+def _reconcile(
+    live: LiveScan,
+    approved_by_hash: dict[str, tuple[datetime, str]],
+    window_approved: list[tuple[datetime, str, str]],
+) -> Reconciliation:
+    """Compare live digests to approved hashes. Pure.
+
+    ``approved_by_hash`` spans the *whole* log, not the window: bytes
+    approved a month ago are still approved bytes, and window-scoping this
+    side would report every untouched file as unapproved. ``window_approved``
+    is window-scoped, because "approved this week and yet not live" is the
+    claim worth making — the same row a year ago is just superseded history.
+    """
+    if live.reason is not None:
+        return Reconciliation(
+            scanned=0,
+            matched=0,
+            latest_match_ts="—",
+            unmatched_live=(),
+            unmatched_live_total=0,
+            orphan_rows=(),
+            orphan_rows_total=0,
+            unreadable=live.unreadable,
+            reason=live.reason,
+        )
+    matched = 0
+    latest: tuple[datetime, str] | None = None
+    unmatched_live: list[str] = []
+    for file in live.files:
+        hit = next((approved_by_hash[d] for d in file.digests if d in approved_by_hash), None)
+        if hit is None:
+            unmatched_live.append(file.digests[0])
+            continue
+        matched += 1
+        if latest is None or hit[0] > latest[0]:
+            latest = hit
+    live_digests = {digest for file in live.files for digest in file.digests}
+    orphans = [raw_ts for _, raw_ts, digest in window_approved if digest not in live_digests]
+    return Reconciliation(
+        scanned=len(live.files),
+        matched=matched,
+        latest_match_ts=_clean(latest[1]) if latest is not None else "—",
+        unmatched_live=tuple(unmatched_live[:_RECON_CAP]),
+        unmatched_live_total=len(unmatched_live),
+        orphan_rows=tuple(_clean(ts) for ts in orphans[:_RECON_CAP]),
+        orphan_rows_total=len(orphans),
+        unreadable=live.unreadable,
+    )
+
+
 def build_reading(
     records: list[dict[str, Any]],
     *,
@@ -158,14 +354,19 @@ def build_reading(
     end: datetime,
     unparsable: int = 0,
     top: int = _DEFAULT_TOP,
+    live: LiveScan | None = None,
 ) -> Reading:
     """Select the section's in-window rows and tally decisions.
 
-    Pure: takes already-loaded records so the join is reproducible offline
-    from the same inputs.
+    Pure: takes already-loaded records — and already-hashed live files — so
+    the join is reproducible offline from the same inputs (ADR-0075). All
+    file I/O lives in `load_records` / `scan_live`.
     """
     selected: list[tuple[datetime, Row]] = []
     counts = {"approved": 0, "staged": 0, "rejected": 0, "other": 0}
+    # Every approved hash in the log, newest wins; and the in-window subset.
+    approved_by_hash: dict[str, tuple[datetime, str]] = {}
+    window_approved: list[tuple[datetime, str, str]] = []
     for record in records:
         if not _matches_section(record.get("path"), section):
             continue
@@ -176,9 +377,21 @@ def build_reading(
         if parsed is None:
             unparsable += 1
             continue
-        if not (start < parsed <= end):
-            continue
         decision = record.get("decision")
+        in_window = start < parsed <= end
+        digest = record.get("content_hash")
+        if decision == "approved" and isinstance(digest, str) and digest.strip():
+            # Collected before the window filter: bytes approved before the
+            # start commit are still approved bytes, and the live file
+            # carrying them must not read as unapproved.
+            digest = digest.strip().lower()
+            previous = approved_by_hash.get(digest)
+            if previous is None or parsed > previous[0]:
+                approved_by_hash[digest] = (parsed, str(raw_ts))
+            if in_window:
+                window_approved.append((parsed, str(raw_ts), digest))
+        if not in_window:
+            continue
         counts[decision if decision in counts else "other"] += 1
         selected.append(
             (
@@ -223,6 +436,9 @@ def build_reading(
         truncated=len(selected) - len(shown),
         window_start=_clean(start.isoformat()),
         window_end=_clean(end.isoformat()),
+        reconciliation=(
+            None if live is None else _reconcile(live, approved_by_hash, window_approved)
+        ),
     )
 
 
@@ -234,6 +450,68 @@ def format_unavailable(reason: str) -> str:
     )
 
 
+def format_reconciliation(recon: Reconciliation, section: str) -> list[str]:
+    """Render the three named states. Digests and counts only."""
+    if recon.reason is not None:
+        return [
+            "",
+            f"**Live-text reconciliation**: unavailable (reason={recon.reason}). "
+            "This is NOT evidence that the live text lacks an approval record — "
+            "the instrument could not hash the live value-layer files.",
+        ]
+    lines = [
+        "",
+        "**Live-text reconciliation** (`sha256(live file)[:16]` vs the "
+        "`content_hash` of every approved row, whole log): "
+        f"{recon.scanned} live file(s) hashed, {recon.matched} match an approved "
+        f"row (latest @{recon.latest_match_ts}).",
+    ]
+    if section == "constitution":
+        lines.append(
+            "(Scope: the `constitution/*.md` of the home given to this scan, which "
+            "the runtime loads only when started without `--constitution-dir` and "
+            "without `--no-axioms` (`cli/runtime.py:104-105`). This reading cannot "
+            "see which flags a run used; under an override it describes files the "
+            "runtime did not read.)"
+        )
+    if recon.unmatched_live_total:
+        shown = ", ".join(recon.unmatched_live)
+        more = recon.unmatched_live_total - len(recon.unmatched_live)
+        suffix = f", +{more} more" if more else ""
+        lines.append(
+            f"⚠️ {recon.unmatched_live_total} live file(s) match NO approved row "
+            f"(sha256[:16] {shown}{suffix}). The bytes the runtime reads did not come "
+            "from a logged approval. FOUR causes produce this and the hash cannot "
+            "tell them apart: a hand repair, a restore from backup, an out-of-band "
+            "edit — and a file that never had a row to begin with, which is the "
+            "expected, permanent state of a shipped default (`contemplative-agent "
+            "init` copies the template value layer in without writing any audit row, "
+            "`cli/session_cmds.py:66,88`) and of an approval older than the retained "
+            "log. Do not report a steady count as a gate bypass; a RISE in it is the "
+            "signal. All four leave the tally above clean — this line is about the "
+            "bytes, not about the window."
+        )
+    if recon.orphan_rows_total:
+        shown = ", ".join(f"@{ts}" for ts in recon.orphan_rows)
+        more = recon.orphan_rows_total - len(recon.orphan_rows)
+        suffix = f", +{more} more" if more else ""
+        lines.append(
+            f"⚠️ {recon.orphan_rows_total} approved row(s) in this window have no "
+            f"live file carrying that hash ({shown}{suffix}). Approved and "
+            "written, but not what the runtime reads now — superseded by a later "
+            "approval, written beside the canonical name (`identity-2.md`), or "
+            "reverted."
+        )
+    if not recon.unmatched_live_total and not recon.orphan_rows_total and recon.scanned:
+        lines.append(
+            "Every live file traces to an approved row, and every approved row in "
+            "this window is live."
+        )
+    if recon.unreadable:
+        lines.append(f"({recon.unreadable} live file(s) could not be hashed and are excluded.)")
+    return lines
+
+
 def format_reading(reading: Reading) -> str:
     total = reading.approved + reading.staged + reading.rejected + reading.other
     lines = [
@@ -251,6 +529,8 @@ def format_reading(reading: Reading) -> str:
         )
     elif not reading.changed and total == 0:
         lines.append("No diff and no approval records — nothing to reconcile.")
+    if reading.reconciliation is not None:
+        lines.extend(format_reconciliation(reading.reconciliation, reading.section))
     if reading.rows:
         lines.append("")
         lines.append("| ts | command | decision | source | content_hash |")
@@ -281,6 +561,14 @@ def main(argv: list[str] | None = None) -> int:
     parser.add_argument("--start", required=True, help="ISO-8601 start-commit timestamp")
     parser.add_argument("--end", required=True, help="ISO-8601 end-commit timestamp")
     parser.add_argument("--top", type=int, default=_DEFAULT_TOP, help="max rows rendered")
+    parser.add_argument(
+        "--home",
+        type=Path,
+        default=None,
+        help="MOLTBOOK_HOME whose live value-layer files are hashed and reconciled "
+        "against the approved rows; omitted renders the reconciliation as "
+        "unavailable rather than silently skipping it",
+    )
     args = parser.parse_args(argv)
 
     try:
@@ -293,6 +581,11 @@ def main(argv: list[str] | None = None) -> int:
         print(format_unavailable(exc.reason))
         return 0
 
+    live = (
+        LiveScan(reason="live-home-not-given")
+        if args.home is None
+        else scan_live(args.home, args.section)
+    )
     reading = build_reading(
         records,
         section=args.section,
@@ -301,6 +594,7 @@ def main(argv: list[str] | None = None) -> int:
         end=end,
         unparsable=unparsable,
         top=args.top,
+        live=live,
     )
     print(format_reading(reading))
     return 0
diff --git a/scripts/weekly-analysis.sh b/scripts/weekly-analysis.sh
index 0e05f9f..0544201 100755
--- a/scripts/weekly-analysis.sh
+++ b/scripts/weekly-analysis.sh
@@ -243,10 +243,17 @@ if [[ -d "$DATA_REPO/.git" ]]; then
         # never target paths. Observability only — a failure must not break
         # the weekly report, but it must also never read as "no approval",
         # hence the explicit unavailable line instead of an empty string.
+        #
+        # --home adds the live-text reconciliation (2026-08-22 F1.2): the row
+        # tally answers "was there an approval", the hash comparison answers
+        # "are these the approved bytes". A hand repair leaves the first clean
+        # and only the second can see it. Renders digests and counts, never a
+        # live file's content or path.
         approval_join() {  # $1 = section, $2 = changed|unchanged
             local out
             out=$(python3 "$PROJECT_ROOT/scripts/value_layer_approval_join.py" \
                 --audit "$MOLTBOOK_HOME/logs/audit.jsonl" \
+                --home "$MOLTBOOK_HOME" \
                 --section "$1" --diff "$2" \
                 --start "$start_cdate" --end "$end_cdate" 2>/dev/null || true)
             if [[ -z "$out" ]]; then
diff --git a/tests/test_value_layer_approval_join.py b/tests/test_value_layer_approval_join.py
index 0c6aa3f..bcd9a76 100644
--- a/tests/test_value_layer_approval_join.py
+++ b/tests/test_value_layer_approval_join.py
@@ -11,20 +11,39 @@ shows a diff* (the alarm), and the log being unreadable (never the alarm) —
 plus the render boundary: ``reason`` is operator free text and ``source_ids``
 is an unbounded lineage list, and neither may ever reach the prompt.
 
+The live-text reconciliation (2026-08-22 F1.2) is pinned beside it: the row
+tally answers "was there an approval row", never "are these the approved
+bytes", so a hand-repaired value layer reads identical to an untouched one.
+The three named states — live text matching an approved row, live text
+matching none, and an approved row with no live file carrying its hash —
+are asserted separately, together with the fact that the tally stays clean
+in the second case (which is why the hash comparison has to exist).
+
+Calibration is pinned beside the states (review, 2026-08-22): the second
+state is also where a shipped default permanently sits — ``init`` copies
+the template value layer in with no audit row — so the rendering must name
+that cause instead of asserting a bypass, and an empty section directory
+must abstain rather than read as reconciled.
+
 Fault column (ADR-0077): a missing or unreadable audit log renders an
-explicit ``unavailable (reason=…)`` line. An unavailable instrument that
-rendered as "no approval record" would manufacture the exact false alarm
-this finding set out to make impossible.
+explicit ``unavailable (reason=…)`` line, and so does an unhashable live
+layer. An unavailable instrument that rendered as "no approval record" (or
+as "matches NO approved row") would manufacture the exact false alarm this
+finding set out to make impossible.
 """
 
 from __future__ import annotations
 
+import hashlib
 import json
+import os
 import subprocess
 import sys
 from datetime import datetime, timezone
 from pathlib import Path
 
+import pytest
+
 sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "scripts"))
 import value_layer_approval_join as vlaj  # noqa: E402  # pyright: ignore[reportMissingImports]
 
@@ -33,6 +52,9 @@ SCRIPT = Path(__file__).resolve().parent.parent / "scripts" / "value_layer_appro
 HOME = "/Users/someone/.config/moltbook"
 START = "2026-08-01T23:00:00+09:00"
 END = "2026-08-08T23:00:00+09:00"
+# More live files than one reconciliation line may name, so the overflow is
+# exercised rather than assumed.
+_CAP_OVERFLOW = 8
 
 
 def _ts(raw: str) -> datetime:
@@ -57,7 +79,9 @@ def _record(
     }
 
 
-def _reading(records, *, section="skills", changed=True, top=vlaj._DEFAULT_TOP, unparsable=0):
+def _reading(
+    records, *, section="skills", changed=True, top=vlaj._DEFAULT_TOP, unparsable=0, live=None
+):
     return vlaj.build_reading(
         records,
         section=section,
@@ -66,6 +90,7 @@ def _reading(records, *, section="skills", changed=True, top=vlaj._DEFAULT_TOP,
         end=_ts(END),
         unparsable=unparsable,
         top=top,
+        live=live,
     )
 
 
@@ -295,6 +320,10 @@ class TestUnavailableIsNotTheAlarm:
         assert "unavailable (reason=audit-log-missing)" in result.stdout
         assert "NO APPROVED RECORD" not in result.stdout
 
+    @pytest.mark.skipif(
+        hasattr(os, "geteuid") and os.geteuid() == 0,
+        reason="chmod(0o000) does not block root, so the fault cannot be injected",
+    )
     def test_unreadable_audit_log_renders_a_reason_code(self, tmp_path):
         audit = tmp_path / "audit.jsonl"
         audit.write_text("{}\n", encoding="utf-8")
@@ -341,6 +370,271 @@ class TestUnavailableIsNotTheAlarm:
         assert unparsable == 2
 
 
+def _digest(text: str) -> str:
+    """The hash ``cli/approval.py:161`` writes for `text`."""
+    return hashlib.sha256(text.encode()).hexdigest()[:16]
+
+
+def _live_home(tmp_path: Path, section: str, files: dict[str, str]) -> Path:
+    home = tmp_path / "home"
+    directory = home if section == "identity" else home / section
+    directory.mkdir(parents=True, exist_ok=True)
+    for name, text in files.items():
+        (home / name).parent.mkdir(parents=True, exist_ok=True)
+        (home / name).write_text(text, encoding="utf-8")
+    return home
+
+
+class TestLiveTextReconciliation:
+    """audit.jsonl records *approvals*, not *writes* (2026-08-22 F1.2)."""
+
+    def test_live_text_matching_an_approved_row_is_the_normal_case(self, tmp_path):
+        text = "approved body\n"
+        home = _live_home(tmp_path, "skills", {"skills/s.md": text})
+        records = [
+            _record(f"{HOME}/skills/s.md", "2026-08-03T03:00:00+00:00", content_hash=_digest(text))
+        ]
+        rendered = vlaj.format_reading(
+            _reading(records, live=vlaj.scan_live(home, "skills"), changed=True)
+        )
+        assert "1 live file(s) hashed, 1 match an approved row" in rendered
+        assert "match NO approved row" not in rendered
+        assert "no live file carrying that hash" not in rendered
+
+    def test_a_hand_repaired_file_matches_no_approved_row(self, tmp_path):
+        """The state the tally cannot see.
+
+        An approval row exists for the section (so the row-count alarm stays
+        silent), but the bytes the runtime reads are not the approved ones —
+        a hand edit, a restore, or an out-of-band write.
+        """
+        home = _live_home(tmp_path, "identity", {"identity.md": "hand-typed replacement\n"})
+        records = [
+            _record(
+                f"{HOME}/identity.md",
+                "2026-08-03T03:00:00+00:00",
+                content_hash=_digest("the text that was actually approved"),
+            )
+        ]
+        reading = _reading(
+            records, section="identity", live=vlaj.scan_live(home, "identity"), changed=True
+        )
+        rendered = vlaj.format_reading(reading)
+        assert reading.approved == 1, "the tally reads clean — that is the whole point"
+        assert "NO APPROVED RECORD" not in rendered
+        assert "1 live file(s) match NO approved row" in rendered
+        assert _digest("hand-typed replacement\n") in rendered
+
+    def test_an_approved_row_with_no_live_file_is_its_own_state(self, tmp_path):
+        """Approved and written, but not what the runtime reads now."""
+        live_text = "the older, still-live body\n"
+        home = _live_home(tmp_path, "skills", {"skills/s.md": live_text})
+        records = [
+            _record(
+                f"{HOME}/skills/s.md",
+                "2026-07-01T03:00:00+00:00",
+                content_hash=_digest(live_text),
+            ),
+            _record(
+                f"{HOME}/skills/s.md",
+                "2026-08-03T03:00:00+00:00",
+                content_hash=_digest("adopted somewhere the runtime does not read"),
+            ),
+        ]
+        rendered = vlaj.format_reading(
+            _reading(records, live=vlaj.scan_live(home, "skills"), changed=True)
+        )
+        assert "1 approved row(s) in this window have no live file carrying that hash" in rendered
+        assert "@2026-08-03T03:00:00+00:00" in rendered
+        assert "match NO approved row" not in rendered, "the live bytes were approved in July"
+
+    def test_a_pre_window_approval_still_counts_as_approved_bytes(self, tmp_path):
+        """Window-scoping the live side would call every untouched file forged."""
+        text = "approved long before the start commit\n"
+        home = _live_home(tmp_path, "rules", {"rules/r.md": text})
+        records = [
+            _record(f"{HOME}/rules/r.md", "2026-05-01T03:00:00+00:00", content_hash=_digest(text))
+        ]
+        rendered = vlaj.format_reading(
+            _reading(records, section="rules", live=vlaj.scan_live(home, "rules"), changed=False)
+        )
+        assert "1 match an approved row" in rendered
+        assert "match NO approved row" not in rendered
+
+    def test_the_adopt_newline_terminator_is_not_a_mismatch(self, tmp_path):
+        """``adopt`` hashes the text and writes it plus a newline.
+
+        Hashing only the bytes on disk would report every newline-terminated
+        adoption as an unapproved hand edit.
+        """
+        text = "body without a trailing newline"
+        home = _live_home(tmp_path, "skills", {"skills/s.md": text + "\n"})
+        records = [
+            _record(f"{HOME}/skills/s.md", "2026-08-03T03:00:00+00:00", content_hash=_digest(text))
+        ]
+        rendered = vlaj.format_reading(
+            _reading(records, live=vlaj.scan_live(home, "skills"), changed=True)
+        )
+        assert "1 match an approved row" in rendered
+        assert "match NO approved row" not in rendered
+
+    def test_a_sibling_beside_the_canonical_identity_is_not_live(self, tmp_path):
+        """``identity-2.md`` is written, approved, and never read by the runtime."""
+        home = _live_home(
+            tmp_path,
+            "identity",
+            {"identity.md": "live body\n", "identity-2.md": "the adopted body\n"},
+        )
+        scan = vlaj.scan_live(home, "identity")
+        assert len(scan.files) == 1
+        assert scan.files[0].digests[0] == _digest("live body\n")
+
+    def test_only_hashes_and_counts_are_rendered_never_content_or_names(self, tmp_path):
+        home = _live_home(
+            tmp_path, "skills", {"skills/SLUG-MARKER-from-a-post.md": "BODY-MARKER text\n"}
+        )
+        rendered = vlaj.format_reading(
+            _reading([], live=vlaj.scan_live(home, "skills"), changed=True)
+        )
+        assert "BODY-MARKER" not in rendered
+        assert "SLUG-MARKER" not in rendered
+        assert "match NO approved row" in rendered
+
+    def test_the_list_of_unmatched_digests_is_capped_not_silent(self, tmp_path):
+        home = _live_home(
+            tmp_path, "skills", {f"skills/s{i}.md": f"body {i}\n" for i in range(_CAP_OVERFLOW)}
+        )
+        rendered = vlaj.format_reading(
+            _reading([], live=vlaj.scan_live(home, "skills"), changed=True)
+        )
+        assert f"{_CAP_OVERFLOW} live file(s) match NO approved row" in rendered
+        assert f"+{_CAP_OVERFLOW - vlaj._RECON_CAP} more" in rendered
+
+    def test_a_never_approved_default_is_named_as_a_cause_not_a_bypass(self, tmp_path):
+        """``init`` copies the template value layer in with no audit row.
+
+        ``cli/session_cmds.py:66`` (constitution / skills / rules) and ``:88``
+        (identity) write no approval record at all, so a shipped default sits
+        in "matches NO approved row" permanently and benignly. Naming only
+        hand repair / restore / out-of-band edit would render that steady
+        state as an accusation every week.
+        """
+        home = _live_home(tmp_path, "constitution", {"constitution/axioms.md": "template body\n"})
+        rendered = vlaj.format_reading(
+            _reading(
+                [],
+                section="constitution",
+                live=vlaj.scan_live(home, "constitution"),
+                changed=False,
+            )
+        )
+        assert "1 live file(s) match NO approved row" in rendered
+        assert "contemplative-agent init" in rendered
+        assert "never had a row" in rendered
+        assert "RISE in it is the signal" in rendered
+
+    def test_the_constitution_scan_declares_the_override_it_cannot_see(self, tmp_path):
+        """``--constitution-dir`` redirects the runtime's read (`cli/runtime.py:105`).
+
+        The scan hashes ``<home>/constitution`` regardless, so the assumption
+        is rendered rather than left implicit. The other sections have no
+        such override and must not carry the caveat.
+        """
+        text = "approved body\n"
+        home = _live_home(tmp_path, "constitution", {"constitution/axioms.md": text})
+        records = [
+            _record(
+                f"{HOME}/constitution/axioms.md",
+                "2026-08-03T03:00:00+00:00",
+                content_hash=_digest(text),
+            )
+        ]
+        rendered = vlaj.format_reading(
+            _reading(
+                records,
+                section="constitution",
+                live=vlaj.scan_live(home, "constitution"),
+                changed=True,
+            )
+        )
+        assert "`--constitution-dir`" in rendered
+
+        skills_home = _live_home(tmp_path / "s", "skills", {"skills/s.md": text})
+        skills_rendered = vlaj.format_reading(
+            _reading([], live=vlaj.scan_live(skills_home, "skills"), changed=True)
+        )
+        assert "--constitution-dir" not in skills_rendered
+
+    @pytest.mark.skipif(
+        hasattr(os, "geteuid") and os.geteuid() == 0,
+        reason="chmod(0o000) does not block root, so the fault cannot be injected",
+    )
+    def test_an_unreadable_live_file_degrades_to_a_count(self, tmp_path):
+        text = "readable\n"
+        home = _live_home(tmp_path, "skills", {"skills/a.md": text, "skills/b.md": "secret\n"})
+        blocked = home / "skills" / "b.md"
+        blocked.chmod(0o000)
+        try:
+            scan = vlaj.scan_live(home, "skills")
+        finally:
+            blocked.chmod(0o600)
+        records = [
+            _record(f"{HOME}/skills/a.md", "2026-08-03T03:00:00+00:00", content_hash=_digest(text))
+        ]
+        rendered = vlaj.format_reading(_reading(records, live=scan, changed=True))
+        assert scan.unreadable == 1
+        assert "1 live file(s) could not be hashed" in rendered
+
+
+class TestReconciliationUnavailableIsNotTheAlarm:
+    def test_a_missing_home_renders_a_reason_code(self, tmp_path):
+        scan = vlaj.scan_live(tmp_path / "absent", "skills")
+        assert scan.reason == "live-home-missing"
+        rendered = vlaj.format_reading(_reading([], live=scan, changed=True))
+        assert "reason=live-home-missing" in rendered
+        assert "match NO approved row" not in rendered
+
+    def test_a_missing_section_directory_renders_a_reason_code(self, tmp_path):
+        home = tmp_path / "home"
+        home.mkdir()
+        scan = vlaj.scan_live(home, "skills")
+        assert scan.reason == "live-dir-missing"
+        assert "match NO approved row" not in vlaj.format_reading(_reading([], live=scan))
+
+    def test_an_empty_section_directory_renders_a_reason_code(self, tmp_path):
+        """Zero files hashed must not read as reconciled.
+
+        It is also the shape a run redirected by ``--constitution-dir``
+        leaves in the default tree, which this scan cannot detect.
+        """
+        home = _live_home(tmp_path, "constitution", {})
+        scan = vlaj.scan_live(home, "constitution")
+        assert scan.reason == "live-dir-empty"
+        rendered = vlaj.format_reading(
+            _reading([], section="constitution", live=scan, changed=True)
+        )
+        assert "reason=live-dir-empty" in rendered
+        assert "match NO approved row" not in rendered
+        assert "Every live file traces" not in rendered
+
+    def test_a_missing_identity_file_renders_a_reason_code(self, tmp_path):
+        home = tmp_path / "home"
+        home.mkdir()
+        scan = vlaj.scan_live(home, "identity")
+        assert scan.reason == "live-identity-missing"
+        assert "match NO approved row" not in vlaj.format_reading(
+            _reading([], section="identity", live=scan)
+        )
+
+    def test_omitting_home_on_the_cli_is_declared_not_skipped(self, tmp_path):
+        audit = tmp_path / "audit.jsonl"
+        audit.write_text("", encoding="utf-8")
+        result = _run_cli(audit, "--diff", "changed")
+        assert result.returncode == 0, result.stderr
+        assert "reason=live-home-not-given" in result.stdout
+        assert "match NO approved row" not in result.stdout
+
+
 def _run_cli(audit: Path, *extra: str, start: str = START) -> subprocess.CompletedProcess[str]:
     return subprocess.run(
         [
@@ -382,6 +676,27 @@ class TestCli:
         assert "abc123def4567890" in result.stdout
         assert "FREE-TEXT-MARKER" not in result.stdout
 
+    def test_end_to_end_reconciliation_over_a_real_home(self, tmp_path):
+        home = _live_home(tmp_path, "skills", {"skills/s.md": "hand-typed body\n"})
+        audit = tmp_path / "audit.jsonl"
+        audit.write_text(
+            json.dumps(
+                _record(
+                    f"{HOME}/skills/s.md",
+                    "2026-08-03T03:00:00+00:00",
+                    content_hash=_digest("the approved body\n"),
+                )
+            )
+            + "\n",
+            encoding="utf-8",
+        )
+        result = _run_cli(audit, "--diff", "changed", "--home", str(home))
+        assert result.returncode == 0, result.stderr
+        assert "**Live-text reconciliation**" in result.stdout
+        assert "1 live file(s) match NO approved row" in result.stdout
+        assert _digest("hand-typed body\n") in result.stdout
+        assert "hand-typed body" not in result.stdout
+
     def test_timezone_naive_stamps_are_read_as_utc(self):
         parsed = vlaj.parse_ts("2026-08-03T03:00:00")
         assert parsed == datetime(2026, 8, 3, 3, 0, tzinfo=timezone.utc)
diff --git a/tests/test_weekly_analysis_shell.py b/tests/test_weekly_analysis_shell.py
index 3db2aba..049adc9 100644
--- a/tests/test_weekly_analysis_shell.py
+++ b/tests/test_weekly_analysis_shell.py
@@ -15,6 +15,7 @@ macOS-only: the script uses BSD ``date -j``.
 
 from __future__ import annotations
 
+import hashlib
 import json
 import os
 import re
@@ -410,8 +411,21 @@ class TestPromptAssembly:
         absence is the alarm the report could not previously distinguish from
         the presence. The record's free text (``reason``) and lineage list
         (``source_ids``) must not ride along into the prompt.
+
+        findings F1.2 adds the third half: ``--home`` must reach the join, so
+        the live bytes are hashed against the approved rows. Without that
+        wiring a hand-repaired value layer renders identically to an untouched
+        one, and the block below would only ever answer "was there a row".
         """
         home = _make_home(tmp_path)
+        # The live value layer the runtime reads (identity.md + the three
+        # section directories), so the reconciliation has bytes to hash.
+        identity_text = "the live identity body\n"
+        (home / "identity.md").write_text(identity_text, encoding="utf-8")
+        identity_hash = hashlib.sha256(identity_text.encode()).hexdigest()[:16]
+        for section in ("constitution", "skills", "rules"):
+            (home / section).mkdir()
+            (home / section / "a.md").write_text(f"{section} body\n", encoding="utf-8")
         data_repo = tmp_path / "fakehome" / "MyAI_Lab" / "contemplative-agent-data"
         (data_repo / "skills").mkdir(parents=True)
 
@@ -447,7 +461,7 @@ class TestPromptAssembly:
                     "path": f"{home}/identity.md",
                     "decision": "approved",
                     "source": "stage-adopted",
-                    "content_hash": "IDHASH0000000000",
+                    "content_hash": identity_hash,
                     "reason": "FREE-TEXT-MARKER typed by the operator",
                     "source_ids": ["LINEAGE-MARKER-1"],
                 }
@@ -472,8 +486,14 @@ class TestPromptAssembly:
         state_diff = prompt.split("## Log Anomaly Sweep")[0]
         # One block per value-layer section: identity, constitution, skills, rules.
         assert state_diff.count("**Approval provenance**") == 4
-        assert "IDHASH0000000000" in state_diff
+        assert identity_hash in state_diff
         assert "NO APPROVED RECORD" in state_diff
+        # The live bytes were hashed too, and the two answers are distinct:
+        # identity's live text traces to its approved row, while the skills /
+        # constitution / rules bytes trace to none.
+        assert state_diff.count("**Live-text reconciliation**") == 4
+        assert "1 live file(s) hashed, 1 match an approved row" in state_diff
+        assert "1 live file(s) match NO approved row" in state_diff
         # The instrument read the log; nothing degraded to "cannot tell".
         assert "unavailable (reason=" not in state_diff
         # The record's free text and lineage list stay out of the prompt.
````

### F1.3.patch

**SCOPE_ESCALATED** — code scope として起票されたが、`^(src|scripts|tests)/` の外に触れたため全文ゲートへ昇格した。scope 分類の誤りではなく、要約行での通過を防ぐ設計上の昇格。

触れたパス: `docs/CODEMAPS/architecture.md`

````diff
diff --git a/docs/CODEMAPS/architecture.md b/docs/CODEMAPS/architecture.md
index acce1c4..934b524 100644
--- a/docs/CODEMAPS/architecture.md
+++ b/docs/CODEMAPS/architecture.md
@@ -713,6 +713,8 @@ collect: daily comment-reports + data-repo state diff + previous N reports
        + skill-selection reading (pass-1 selection log: skill-selection-*.jsonl → names
                                   and counts; package renderer via `uv run --no-sync`)
 generate: claude -p → weekly-<end>.md.tmp
+gate:     tmp must carry all five section anchors (## A. … ## E.);
+          else exit 1, reason=REPORT_INCOMPLETE missing=<csv>
 promote:  mv tmp → weekly-<end>.md        (atomic; a failure leaves the prior report)
        →  mv sweep-state.pending → sweep-state   (novelty baseline committed ONLY here)
        →  mv sweep-state.pending.corpus.tsv → sweep-state.corpus.tsv  (lockstep; see below)
@@ -720,6 +722,8 @@ promote:  mv tmp → weekly-<end>.md        (atomic; a failure leaves the prior
 translate: best-effort .ja.md (sonnet); failure never rolls back the promotes
 ```
 
+The promote gate is structural, not `-s` (2026-08-21, findings F1.3). A report that passes `-s` can still be head-truncated: `claude -p --output-format text` prints only the last assistant turn, so a two-turn response promoted a 37,409-byte file that began mid-sentence with A, B and C absent — translated, cited by the diagnosis for figures it did not contain, and queued as next week's `$PREV_REPORTS` baseline. The anchors are the section headings `config/prompts/weekly-analysis.md` defines, matched on the letter prefix only — the trailing wording stays the model's. Nothing beyond them is required, and deliberately not a level-1 title: the prompt's own `# ` lines are prompt-internal headings rather than an instruction to emit one, and `weekly-2026-07-11.md` is a complete A–E report that opens with a preamble and carries no `# ` line at all (gated by `test_a_report_without_a_title_is_still_complete`). Same predicate discipline as the two artifacts downstream of it — `findings_complete()` and the insight review's `RECOMMEND:` grep — and the failure handling is the emptiness branch's: non-zero exit, prior report untouched, nothing spent, and a reason code the pipeline's stage accounting can name.
+
 Order is load-bearing. The sweep's Δ / 🆕 columns and the drift scan's new/removed pairs are defined against their last committed snapshots, so both run `--no-update --emit-state` and their baselines are committed after the report lands — a run that produces nothing must spend nothing (findings F1.2; two consecutive weeks lost). The invariant check, the duplicate scan, the skill-selection reading and the approval join hold no state and are absolute readings, so they need no such ordering.
 
 The approval join (2026-08-15, findings F1.1) annotates **each value-layer section of the state diff** — `identity.md`, `constitution/`, `skills/`, `rules/` — with the in-window ADR-0012 approval rows from `logs/audit.jsonl` whose `path` falls under that section's directory. It closes a gap in the diff itself: the diff showed *what* changed and nothing about *whether it passed the gate*, so the 2026-08-15 report's strongest claim ("whether it passed through the `amend-constitution` approval path is not visible in the operator-facing data supplied here") was bounded by missing data rather than by analysis, in a chain that already reads that file for the ADR-0091 identity-cadence due. Three renderings, deliberately distinct: approved rows present (citable `ts` + `content_hash`), **no approved row while the section shows a diff** (the alarm — reported as an observation, since a sync lag or a pre-window approval produces the same shape), and `unavailable (reason=…)` when the log is missing or unreadable — an unavailable instrument must never render as the alarm, or the report manufactures a gate-bypass claim out of its own blindness (gated by `test_a_missing_audit_log_never_reads_as_a_missing_approval`). Window: the two data-repo **commit timestamps**, half-open (`start < ts <= end`) — anything approved at or before the start commit is already inside that commit's tree and so is not part of the diff; the calendar bounds would mis-window by the sync lag. Rendered fields are `ts` / `command` / `decision` / `source` / `content_hash` only: `reason` is operator free text, `source_ids` an unbounded lineage list, and target paths carry skill filenames slugified from distilled pattern text — all three stay out (same send-the-shape choice as ADR-0083).
diff --git a/scripts/weekly-analysis.sh b/scripts/weekly-analysis.sh
index 0e05f9f..378c65c 100755
--- a/scripts/weekly-analysis.sh
+++ b/scripts/weekly-analysis.sh
@@ -560,6 +560,42 @@ $DAILY_REPORTS_FRAMED"
 mkdir -p "$REPORT_DIR"
 OUTPUT="$REPORT_DIR/weekly-${END_DATE}.md"
 
+# `report_missing_parts <file>` — the report's machine contract, as a csv of the
+# parts that are absent (empty output = complete). Size is not that contract:
+# 2026-08-21 promoted a 37,409-byte report whose first line began mid-sentence
+# ("ior statement implied…") with A, B and C gone, because `claude -p
+# --output-format text` prints only its last assistant turn and the response
+# spanned more than one. It passed `-s`, was translated, was cited by the
+# diagnosis for figures that were not in it, and would have returned as next
+# week's $PREV_REPORTS trend baseline. Whatever truncates the stream, a report
+# missing A-C must read as unavailable rather than as clean (ADR-0077).
+#
+# The two artifacts downstream of this one already have such a predicate —
+# `findings_complete()` and the insight review's `RECOMMEND:` grep, both in
+# weekly-pipeline.sh — and this is the input to both.
+#
+# The anchors are the five section headings this repo's own prompt file defines
+# (config/prompts/weekly-analysis.md, "## A." … "## E."), matched on the letter
+# prefix only: the trailing wording of a heading is the model's to vary, the
+# letter is the format's. They are structure, not content-surface phrases.
+#
+# Nothing else is required — in particular not a level-1 title. The prompt's
+# "# " lines are its own internal headings, never an instruction to emit a
+# document title, and weekly-2026-07-11.md is a complete A-E report that opens
+# with a preamble and no title at all. Requiring one would discard that shape
+# while adding no detection power: the truncation this guard exists for takes
+# the title and A-C together.
+#
+# Plain string accumulation, not an array: /bin/bash on macOS is 3.2, where
+# `${#arr[@]}` on an empty array trips `set -u`.
+report_missing_parts() {  # report_missing_parts <file>
+    local file="$1" letter missing=""
+    for letter in A B C D E; do
+        grep -q "^## $letter\." "$file" || missing="${missing:+$missing,}$letter"
+    done
+    printf '%s' "$missing"
+}
+
 # --- Run claude ---
 # Write to a temp file and promote on success. A direct `> "$OUTPUT"` truncates
 # the target before the command runs, so any failure (or a run killed mid-flight)
@@ -586,7 +622,19 @@ if ! echo "$USER_PROMPT" | with_timeout "$REPORT_TIMEOUT_SECONDS" claude -p \
 fi
 
 if [[ ! -s "$OUTPUT_TMP" ]]; then
-    echo "ERROR: claude -p exited 0 but produced no output; $OUTPUT left untouched" >&2
+    echo "ERROR: claude -p exited 0 but produced no output; reason=REPORT_EMPTY; $OUTPUT left untouched" >&2
+    exit 1
+fi
+
+# Structural completeness, checked before the promote — see report_missing_parts
+# above for what the 0-byte guard cannot see. Same failure handling either way:
+# non-zero exit, previous $OUTPUT untouched, and a reason code in the message so
+# weekly-pipeline.sh's stage accounting can name it in report.log.
+MISSING_PARTS="$(report_missing_parts "$OUTPUT_TMP")"
+if [[ -n "$MISSING_PARTS" ]]; then
+    echo "ERROR: claude -p exited 0 but the report is structurally incomplete;" \
+        "reason=REPORT_INCOMPLETE missing=$MISSING_PARTS" \
+        "bytes=$(wc -c < "$OUTPUT_TMP" | tr -d ' '); $OUTPUT left untouched" >&2
     exit 1
 fi
 
diff --git a/tests/test_weekly_analysis_shell.py b/tests/test_weekly_analysis_shell.py
index 3db2aba..9d7e76c 100644
--- a/tests/test_weekly_analysis_shell.py
+++ b/tests/test_weekly_analysis_shell.py
@@ -36,6 +36,27 @@ END_DATE = "2026-07-24"
 SEEDED_STATE = "7\t[warning] seeded signature\n"
 SEEDED_CORPUS = "5000\t300\told-rotated.log\n"
 
+# The promote gate's contract: the five section anchors
+# config/prompts/weekly-analysis.md defines. Every stub that expects its report
+# to be promoted must emit them. The title line here is realistic padding (20 of
+# 21 past reports carry one), not part of the contract — see
+# test_a_report_without_a_title_is_still_complete.
+COMPLETE_REPORT = (
+    "# Weekly Analysis Report — Moltbook Agent\n\n"
+    "## A. Quantitative Summary\n\n"
+    "## B. Agent State Snapshot\n\n"
+    "## C. Engagement Patterns\n\n"
+    "## D. Change Points\n\n"
+    "## E. Qualitative Highlights — analytical center\n"
+)
+
+
+def _emit_complete_report(tmp_path: Path) -> str:
+    """A shell line printing a report the promote gate accepts."""
+    body_file = tmp_path / "complete-report.md"
+    body_file.write_text(COMPLETE_REPORT, encoding="utf-8")
+    return f'cat "{body_file}"\n'
+
 
 def _make_home(tmp_path: Path) -> Path:
     """A minimal MOLTBOOK_HOME: one daily report, one log, a seeded state."""
@@ -122,14 +143,93 @@ class TestFailedRunSpendsNothing:
         result = _run(home, _stub_claude(tmp_path, exit_code=0), tmp_path)
 
         assert result.returncode != 0, result.stdout
+        assert "reason=REPORT_EMPTY" in result.stderr
+        assert _state(home).read_text(encoding="utf-8") == SEEDED_STATE
+        assert _pending_files(home) == []
+
+
+class TestStructuralCompleteness:
+    """findings F1.3: a head-truncated report passed the ``-s`` guard.
+
+    ``claude -p --output-format text`` prints only the last assistant turn, so a
+    response spanning two turns produced a 37,409-byte file whose first line
+    began mid-sentence with A, B and C missing. It was promoted, translated,
+    cited by the diagnosis for figures it did not contain, and queued as next
+    week's trend baseline. Size cannot see that failure; the section anchors
+    the report format defines can.
+    """
+
+    def test_head_truncated_report_is_not_promoted(self, tmp_path):
+        home = _make_home(tmp_path)
+        # The observed 2026-08-21 shape: the cut lands mid-sentence and
+        # everything after it — D, E — is intact and internally coherent.
+        truncated = (
+            "ior statement implied a sufficient separation between the "
+            "'query source' and the 'data retrieval process'.\n\n"
+            "## D. Change Points\n\nD1: something shifted.\n\n"
+            "## E. Qualitative Highlights — analytical center\n\nAn example.\n"
+        )
+        result = _run(home, _stub_claude(tmp_path, exit_code=0, body=truncated), tmp_path)
+
+        assert result.returncode != 0, result.stdout
+        assert not (home / "reports" / "analysis" / f"weekly-{END_DATE}.md").exists()
+        assert not (home / "reports" / "analysis" / f"weekly-{END_DATE}.ja.md").exists()
+        # A reason code the pipeline's stage accounting can name, and the parts
+        # that were missing.
+        assert "reason=REPORT_INCOMPLETE" in result.stderr
+        assert "missing=A,B,C" in result.stderr
+        # A failed run still spends nothing.
         assert _state(home).read_text(encoding="utf-8") == SEEDED_STATE
+        assert _corpus(home).read_text(encoding="utf-8") == SEEDED_CORPUS
         assert _pending_files(home) == []
 
+    def test_a_report_missing_only_its_last_section_is_not_promoted(self, tmp_path):
+        """Tail loss is the same defect from the other end."""
+        home = _make_home(tmp_path)
+        body = COMPLETE_REPORT.split("## E.")[0]
+        result = _run(home, _stub_claude(tmp_path, exit_code=0, body=body), tmp_path)
+
+        assert result.returncode != 0, result.stdout
+        assert "missing=E" in result.stderr
+        assert not (home / "reports" / "analysis" / f"weekly-{END_DATE}.md").exists()
+
+    def test_a_report_without_a_title_is_still_complete(self, tmp_path):
+        """The observed legitimate shape a title rule would have discarded.
+
+        ``weekly-2026-07-11.md`` opens with a preamble and goes straight to
+        ``## A.`` with no ``# `` line anywhere; it is complete A-E and was
+        consumed downstream. The format contract
+        (``config/prompts/weekly-analysis.md``) defines only the five section
+        headings — its own ``# `` lines are prompt-internal — so the gate must
+        require only those, or it fails closed on a good report and aborts
+        stage 1 with ``reason=REPORT_FAIL``.
+        """
+        home = _make_home(tmp_path)
+        body = "I have all seven daily reports in context. Per Principle 3 I authored this directly.\n\n---\n\n"
+        body += COMPLETE_REPORT.split("\n\n", 1)[1]
+        assert not body.startswith("# ") and "\n# " not in body
+        result = _run(home, _stub_claude(tmp_path, exit_code=0, body=body), tmp_path)
+
+        assert result.returncode == 0, result.stderr
+        report = home / "reports" / "analysis" / f"weekly-{END_DATE}.md"
+        assert report.read_text(encoding="utf-8") == body
+
+    def test_a_preamble_before_the_title_is_still_a_complete_report(self, tmp_path):
+        """The majority shape: an opening note, then the title, then A-E."""
+        home = _make_home(tmp_path)
+        body = "I have all seven daily reports in context. Here is the report.\n\n---\n\n"
+        body += COMPLETE_REPORT
+        result = _run(home, _stub_claude(tmp_path, exit_code=0, body=body), tmp_path)
+
+        assert result.returncode == 0, result.stderr
+        report = home / "reports" / "analysis" / f"weekly-{END_DATE}.md"
+        assert report.read_text(encoding="utf-8") == body
+
 
 class TestSuccessfulRunCommits:
     def test_report_promoted_and_sweep_state_updated(self, tmp_path):
         home = _make_home(tmp_path)
-        body = "# Weekly\n\nA. Volume\n\nB. Signals\n"
+        body = COMPLETE_REPORT
         result = _run(home, _stub_claude(tmp_path, exit_code=0, body=body), tmp_path)
 
         assert result.returncode == 0, result.stderr
@@ -152,7 +252,7 @@ class TestSuccessfulRunCommits:
         (home / "logs" / "agent.log").write_text(
             "[10:00:00] INFO nothing wrong here\n[10:01:00] DEBUG idle\n", encoding="utf-8"
         )
-        result = _run(home, _stub_claude(tmp_path, exit_code=0, body="# Weekly\n"), tmp_path)
+        result = _run(home, _stub_claude(tmp_path, exit_code=0, body=COMPLETE_REPORT), tmp_path)
 
         assert result.returncode == 0, result.stderr
         assert _state(home).read_text(encoding="utf-8") == ""
@@ -165,7 +265,7 @@ class TestSuccessfulRunCommits:
         corpus that no longer exists, and assert the comparison as fact.
         """
         home = _make_home(tmp_path)
-        result = _run(home, _stub_claude(tmp_path, exit_code=0, body="# Weekly\n"), tmp_path)
+        result = _run(home, _stub_claude(tmp_path, exit_code=0, body=COMPLETE_REPORT), tmp_path)
 
         assert result.returncode == 0, result.stderr
         assert _state(home).read_text(encoding="utf-8") != SEEDED_STATE
@@ -190,7 +290,7 @@ class TestSuccessfulRunCommits:
             "cat > /dev/null\n"
             f"echo x >> {marker}\n"
             f"if [[ $(wc -l < {marker}) -gt 1 ]]; then exit 1; fi\n"
-            "printf '# Weekly\\n'\n",
+            f"{_emit_complete_report(tmp_path)}",
             encoding="utf-8",
         )
         stub.chmod(0o755)
@@ -220,7 +320,8 @@ class TestDailyReportFraming:
         bin_dir.mkdir(exist_ok=True)
         stub = bin_dir / "claude"
         stub.write_text(
-            f'#!/bin/bash\ncat >> "{captured}"\nprintf "# Weekly\\n"\n', encoding="utf-8"
+            f'#!/bin/bash\ncat >> "{captured}"\n{_emit_complete_report(tmp_path)}',
+            encoding="utf-8",
         )
         stub.chmod(0o755)
 
@@ -250,7 +351,8 @@ class TestDailyReportFraming:
         bin_dir.mkdir(exist_ok=True)
         stub = bin_dir / "claude"
         stub.write_text(
-            f'#!/bin/bash\ncat >> "{captured}"\nprintf "# Weekly\\n"\n', encoding="utf-8"
+            f'#!/bin/bash\ncat >> "{captured}"\n{_emit_complete_report(tmp_path)}',
+            encoding="utf-8",
         )
         stub.chmod(0o755)
 
@@ -284,7 +386,8 @@ class TestPromptAssembly:
         bin_dir.mkdir(exist_ok=True)
         stub = bin_dir / "claude"
         stub.write_text(
-            f'#!/bin/bash\ncat >> "{captured}"\nprintf "# Weekly\\n"\n', encoding="utf-8"
+            f'#!/bin/bash\ncat >> "{captured}"\n{_emit_complete_report(tmp_path)}',
+            encoding="utf-8",
         )
         stub.chmod(0o755)
 
@@ -332,7 +435,8 @@ class TestPromptAssembly:
         bin_dir.mkdir(exist_ok=True)
         stub = bin_dir / "claude"
         stub.write_text(
-            f'#!/bin/bash\ncat >> "{captured}"\nprintf "# Weekly\\n"\n', encoding="utf-8"
+            f'#!/bin/bash\ncat >> "{captured}"\n{_emit_complete_report(tmp_path)}',
+            encoding="utf-8",
         )
         stub.chmod(0o755)
 
@@ -385,7 +489,8 @@ class TestPromptAssembly:
         bin_dir.mkdir(exist_ok=True)
         stub = bin_dir / "claude"
         stub.write_text(
-            f'#!/bin/bash\ncat >> "{captured}"\nprintf "# Weekly\\n"\n', encoding="utf-8"
+            f'#!/bin/bash\ncat >> "{captured}"\n{_emit_complete_report(tmp_path)}',
+            encoding="utf-8",
         )
         stub.chmod(0o755)
 
@@ -461,7 +566,8 @@ class TestPromptAssembly:
         bin_dir.mkdir(exist_ok=True)
         stub = bin_dir / "claude"
         stub.write_text(
-            f'#!/bin/bash\ncat >> "{captured}"\nprintf "# Weekly\\n"\n', encoding="utf-8"
+            f'#!/bin/bash\ncat >> "{captured}"\n{_emit_complete_report(tmp_path)}',
+            encoding="utf-8",
         )
         stub.chmod(0o755)
 
@@ -516,7 +622,8 @@ class TestPromptAssembly:
         bin_dir.mkdir(exist_ok=True)
         stub = bin_dir / "claude"
         stub.write_text(
-            f'#!/bin/bash\ncat >> "{captured}"\nprintf "# Weekly\\n"\n', encoding="utf-8"
+            f'#!/bin/bash\ncat >> "{captured}"\n{_emit_complete_report(tmp_path)}',
+            encoding="utf-8",
         )
         stub.chmod(0o755)
````

### F1.4.patch

**SCOPE_ESCALATED** — code scope として起票されたが、`^(src|scripts|tests)/` の外に触れたため全文ゲートへ昇格した。scope 分類の誤りではなく、要約行での通過を防ぐ設計上の昇格。

触れたパス: `docs/CODEMAPS/architecture.md`, `docs/CONFIGURATION.ja.md`

```diff
diff --git a/docs/CODEMAPS/architecture.md b/docs/CODEMAPS/architecture.md
index acce1c4..646c59b 100644
--- a/docs/CODEMAPS/architecture.md
+++ b/docs/CODEMAPS/architecture.md
@@ -87,8 +87,13 @@ The kill switch (`audit_dir` unset) is a different path — it removes the
 corpus rather than injecting it, so it does not overflow),
 `submolt-scope-YYYY-MM-DD.jsonl` (ADR-0086 read-only scope sweep, written by
 `submolt-scan` only: one `scan_start` / N `score` / one `scan_end` per run,
-per-post score + reason code (scored/empty_input/llm_unavailable/
-unparseable/out_of_range) + `subscribed` label + post body base64+sha256;
+per-post score + reason code (scored/empty_input/no_scoreable_content/
+llm_unavailable/unparseable/out_of_range; `no_scoreable_content` is the
+`MIN_SCOREABLE_TOKENS` / `MIN_SCOREABLE_CHARS` floor in
+`llm_functions.score_relevance_detailed` — a body under 3 whitespace-separated
+tokens *and* under 18 characters scores 0.0 with no LLM call; either count
+clearing its floor is enough, so an unspaced CJK body still reaches the model)
++ `subscribed` label + post body base64+sha256;
 scan verdict completed/disabled/discovery_failed/no_submolts/
 aborted_rate_limit/aborted_read_budget/aborted_scored_cap; read via
 `report --submolt-scope`, wired to no gate), LLM telemetry
diff --git a/docs/CONFIGURATION.ja.md b/docs/CONFIGURATION.ja.md
index 93cc3dd..8c5d1f8 100644
--- a/docs/CONFIGURATION.ja.md
+++ b/docs/CONFIGURATION.ja.md
@@ -60,8 +60,11 @@ contemplative-agent report --days 30 --submolt-scope   # その読み値（購
 
 書き込みは自身の監査ログ (`submolt-scope-*.jsonl`) のみ。投稿本文は base64 +
 sha256 で保存され、`reason` コード（`scored` / `empty_input` /
-`llm_unavailable` / `unparseable` / `out_of_range`）により「低い判断」と
-「壊れたスコアラ」が区別できる。エージェントが行動してよい submolt の範囲は
+`no_scoreable_content` / `llm_unavailable` / `unparseable` /
+`out_of_range`）により「低い判断」と「壊れたスコアラ」が区別できる
+（`no_scoreable_content` は本文が短すぎて採点対象になりえない場合 —
+`MIN_SCOREABLE_TOKENS` 未満かつ `MIN_SCOREABLE_CHARS` 未満で LLM を呼ばずに
+0.0）。エージェントが行動してよい submolt の範囲は
 この計器では一切変わらない。`MOLTBOOK_SUBMOLT_SCOPE_DISABLE=1` を立てると、
 launchd job を残したまま sweep だけを無効化できる。
 
diff --git a/src/contemplative_agent/adapters/moltbook/llm_functions.py b/src/contemplative_agent/adapters/moltbook/llm_functions.py
index de62f76..8b40a95 100644
--- a/src/contemplative_agent/adapters/moltbook/llm_functions.py
+++ b/src/contemplative_agent/adapters/moltbook/llm_functions.py
@@ -43,6 +43,33 @@ logger = logging.getLogger(__name__)
 # 1.5 was rejected for axiom-label collapse.
 COMMENT_TEMPERATURE = 1.3
 
+# Fewest whitespace-separated tokens a body needs before relevance scoring is a
+# question the LLM can answer at all. Below it there is no proposition to be
+# on-topic about, and `config/prompts/relevance.md` has no output state for
+# "nothing here" — its contract is one number on 0.0/0.5/1.0, so the model
+# scores the *prompt's own structure* instead and can return the ceiling
+# (observed 2026-08: a post whose entire body was `600-1100 символов` scored
+# 1.00, cleared the 0.80 gate in `config/domain.json`, and spent an internal
+# note, a comment generation and a published reply). The observed class is
+# <= 2 tokens (`test`, `30`, `600-1100 символов`), so 3 is the smallest floor
+# that covers it. Reads the input's size only — never its topic, author or
+# any phrase — and it gates the counterparty's input before any generation,
+# not the agent's output.
+#
+# Pending an operator sign-off at the weekly human gate; the tests pin both
+# numbers exactly so widening the floor cannot pass unnoticed.
+MIN_SCOREABLE_TOKENS = 3
+
+# Character escape hatch for scripts that do not delimit words with spaces
+# (Japanese / Chinese / Thai). Whitespace tokenisation reads a 300-character
+# unspaced body as one token, which the token floor alone would suppress
+# without an LLM call — the observed defect class was byte-short, not merely
+# token-short, so a body clears the floor on *either* count. 18 is the
+# smallest value that still covers the longest observed member
+# (`600-1100 символов`, 17 characters), i.e. it suppresses that class and
+# nothing longer.
+MIN_SCOREABLE_CHARS = 18
+
 
 def _resolve_domain_prompt(template: str) -> str:
     """Resolve a prompt template with the current domain config."""
@@ -54,9 +81,9 @@ def _resolve_domain_prompt(template: str) -> str:
 class RelevanceScore:
     """One relevance judgment plus why it reads the way it does.
 
-    ``score`` alone is lossy: four distinct events all produce 0.0 (empty
-    body, LLM outage, unparseable answer, out-of-range answer) and only one
-    of them is a judgment. Callers that gate on the number can ignore
+    ``score`` alone is lossy: five distinct events all produce 0.0 (empty
+    body, body below the scoreable token floor, LLM outage, unparseable
+    answer, out-of-range answer) and only one of them is a judgment. Callers that gate on the number can ignore
     ``reason``; callers that *measure the distribution* must not, or an
     Ollama outage reads as "uninteresting feed" (ADR-0075).
     """
@@ -73,8 +100,8 @@ def score_relevance_detailed(
     """Score a post's relevance to domain topics (0.0 to 1.0), with a reason.
 
     ``reason`` is ``scored`` for a real judgment; ``empty_input``,
-    ``llm_unavailable``, ``unparseable`` and ``out_of_range`` each carry 0.0
-    for a different cause.
+    ``no_scoreable_content``, ``llm_unavailable``, ``unparseable`` and
+    ``out_of_range`` each carry 0.0 for a different cause.
 
     ``caller`` is the telemetry tag. It defaults to the production gate's
     tag; observation-only callers pass their own so their volume stays
@@ -87,8 +114,19 @@ def score_relevance_detailed(
     class as the reply path's empty post section (weekly-2026-07-24 F1.1).
     "Is there any text" is a structural property, so code answers it rather
     than the LLM (skill: when-code-when-llm).
+
+    A body under ``MIN_SCOREABLE_TOKENS`` *and* under ``MIN_SCOREABLE_CHARS``
+    short-circuits the same way, with its own reason ``no_scoreable_content``:
+    "how much text is there" is the same structural property one notch up, and
+    asking the model to score a body with no proposition in it produced a 1.00
+    on the prompt's own structure. Either count clearing its floor is enough,
+    so an unspaced Japanese/Chinese/Thai body — one whitespace token however
+    long — still reaches the model. Kept as a separate reason code so it stays
+    separable from the judgment 0.0 and from the two failure sentinels in the
+    distribution ADR-0086's instrument reads.
     """
-    if not post_text.strip():
+    stripped = post_text.strip()
+    if not stripped:
         # DEBUG, not WARNING: an empty feed post is a normal condition, and
         # this 0.0 must stay distinguishable from the outage sentinel below —
         # both land in the relevance distribution a retune reads.
@@ -98,6 +136,18 @@ def score_relevance_detailed(
         )
         return RelevanceScore(0.0, "empty_input")
 
+    if len(post_text.split()) < MIN_SCOREABLE_TOKENS and len(stripped) < MIN_SCOREABLE_CHARS:
+        # DEBUG for the same reason as empty_input: a stub post is a normal
+        # feed condition, not a failure.
+        logger.debug(
+            "Post text below the scoreable floor (%d tokens / %d chars) — "
+            "scoring 0.0 without an LLM call "
+            "(reason=no_scoreable_content, not a low score)",
+            MIN_SCOREABLE_TOKENS,
+            MIN_SCOREABLE_CHARS,
+        )
+        return RelevanceScore(0.0, "no_scoreable_content")
+
     prompt = _resolve_domain_prompt(RELEVANCE_PROMPT).format(
         post_content=wrap_untrusted_content(post_text, max_input=1000),
     )
diff --git a/tests/test_llm.py b/tests/test_llm.py
index 79e5d86..540b293 100644
--- a/tests/test_llm.py
+++ b/tests/test_llm.py
@@ -713,27 +713,27 @@ class TestScoreRelevanceParsing:
     @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
     def test_clean_number(self, mock_generate):
         mock_generate.return_value = "0.75"
-        assert score_relevance("test post") == 0.75
+        assert score_relevance("a test post") == 0.75
 
     @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
     def test_number_with_trailing_text(self, mock_generate):
         mock_generate.return_value = "0.7\n\nThis post discusses"
-        assert score_relevance("test post") == 0.7
+        assert score_relevance("a test post") == 0.7
 
     @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
     def test_number_with_leading_text(self, mock_generate):
         mock_generate.return_value = "The score is 0.8"
-        assert score_relevance("test post") == 0.8
+        assert score_relevance("a test post") == 0.8
 
     @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
     def test_no_number_returns_zero(self, mock_generate):
         mock_generate.return_value = "This is not relevant"
-        assert score_relevance("test post") == 0.0
+        assert score_relevance("a test post") == 0.0
 
     @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
     def test_none_returns_zero(self, mock_generate):
         mock_generate.return_value = None
-        assert score_relevance("test post") == 0.0
+        assert score_relevance("a test post") == 0.0
 
     @pytest.mark.parametrize(
         "output",
@@ -750,17 +750,17 @@ class TestScoreRelevanceParsing:
         answer, not a high score. Clamping it to 1.0 failed toward acting;
         reject toward not acting instead."""
         mock_generate.return_value = output
-        assert score_relevance("test post") == 0.0
+        assert score_relevance("a test post") == 0.0
 
     @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
     def test_integer_score(self, mock_generate):
         mock_generate.return_value = "1"
-        assert score_relevance("test post") == 1.0
+        assert score_relevance("a test post") == 1.0
 
     @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
     def test_chinese_text_with_number(self, mock_generate):
         mock_generate.return_value = "0.6 该内容讨论了冥想"
-        assert score_relevance("test post") == 0.6
+        assert score_relevance("a test post") == 0.6
 
     @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
     def test_uses_identity_system_prompt(self, mock_generate, tmp_path):
@@ -776,7 +776,7 @@ class TestScoreRelevanceParsing:
         _configure_skills_marker(tmp_path)
         try:
             mock_generate.return_value = "0.5"
-            score_relevance("test post")
+            score_relevance("a test post")
             system = mock_generate.call_args.kwargs["system"]
             assert system == get_identity_system_prompt()
             assert "<learned_skills>" not in system
@@ -820,6 +820,104 @@ class TestScoreRelevanceEmptyInput:
         mock_generate.assert_called_once()
 
 
+class TestScoreRelevanceNoScoreableContent:
+    """weekly-2026-08-22 F1.4: non-empty, non-blank, and still nothing to score.
+    A body of `600-1100 символов` reached `config/prompts/relevance.md`, whose
+    output contract has no state for "there is no proposition here", and came
+    back 1.00 — above the 0.80 gate — spending an internal note, a comment
+    generation and a published reply on it. "How much text is there" is a
+    structural property, so code answers it (skill: when-code-when-llm)."""
+
+    @pytest.mark.parametrize(
+        "post_text",
+        ["600-1100 символов", "test", "30", "The stars", "  30\n"],
+        ids=["observed-ceiling-case", "one-word", "bare-number", "two-words", "padded"],
+    )
+    @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
+    def test_below_floor_scores_zero_without_llm_call(self, mock_generate, post_text):
+        result = score_relevance_detailed(post_text)
+        mock_generate.assert_not_called()
+        assert result.score == 0.0
+        assert result.reason == "no_scoreable_content"
+
+    @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
+    def test_at_the_floor_still_calls_the_llm(self, mock_generate):
+        """The floor rejects strictly below MIN_SCOREABLE_TOKENS; a body at it
+        is still the model's call to make."""
+        from contemplative_agent.adapters.moltbook.llm_functions import (
+            MIN_SCOREABLE_TOKENS,
+        )
+
+        mock_generate.return_value = "0.9"
+        post_text = " ".join(["word"] * MIN_SCOREABLE_TOKENS)
+        assert score_relevance(post_text) == 0.9
+        mock_generate.assert_called_once()
+
+    @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
+    def test_unspaced_script_body_still_reaches_the_llm(self, mock_generate):
+        """A Japanese/Chinese/Thai body is one whitespace token however long.
+        The token floor alone would suppress a real post with no LLM call and
+        only a DEBUG line, so the character floor is the escape hatch: this
+        body is 1 token and 25 characters, and it is the model's call."""
+        mock_generate.return_value = "0.9"
+        post_text = "瞑想の実践について今日気づいたことを書いておきたい"
+        assert len(post_text.split()) == 1
+        assert score_relevance(post_text) == 0.9
+        mock_generate.assert_called_once()
+
+    @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
+    def test_at_the_character_floor_still_calls_the_llm(self, mock_generate):
+        """Strict-`<` on the character count too: one token at exactly
+        MIN_SCOREABLE_CHARS reaches the model, one character below does not."""
+        from contemplative_agent.adapters.moltbook.llm_functions import (
+            MIN_SCOREABLE_CHARS,
+        )
+
+        mock_generate.return_value = "0.4"
+        assert score_relevance("あ" * MIN_SCOREABLE_CHARS) == 0.4
+        mock_generate.assert_called_once()
+
+        mock_generate.reset_mock()
+        result = score_relevance_detailed("あ" * (MIN_SCOREABLE_CHARS - 1))
+        assert result.score == 0.0
+        assert result.reason == "no_scoreable_content"
+        mock_generate.assert_not_called()
+
+    def test_floor_covers_the_observed_class(self):
+        """The operator-decision numbers, pinned exactly rather than as a
+        one-sided bound: the observed class is <= 2 tokens and <= 17 chars, so
+        3 / 18 are the smallest floors that cover it. A later widening (5, 10)
+        would suppress a large share of real short posts, so it must fail here
+        and be re-signed-off at the human gate rather than pass silently."""
+        from contemplative_agent.adapters.moltbook.llm_functions import (
+            MIN_SCOREABLE_CHARS,
+            MIN_SCOREABLE_TOKENS,
+        )
+
+        assert MIN_SCOREABLE_TOKENS == 3
+        assert MIN_SCOREABLE_CHARS == 18
+
+    @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
+    def test_a_ceiling_answer_cannot_reach_the_gate(self, mock_generate):
+        """The defect end-to-end: even if the model would answer 1.00, a stub
+        body never gets to ask it, so nothing clears the 0.80 relevance gate."""
+        mock_generate.return_value = "1.0"
+        assert score_relevance("600-1100 символов") == 0.0
+        mock_generate.assert_not_called()
+
+    @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
+    def test_short_circuit_is_logged_below_the_outage_warning(self, mock_generate, caplog):
+        # Same register as empty_input: a stub post is a normal feed condition,
+        # not a failure, so it must not masquerade as one — nor be silent.
+        with caplog.at_level(logging.DEBUG):
+            score_relevance("test")
+        assert not [r for r in caplog.records if r.levelno >= logging.WARNING]
+        assert any(
+            r.levelno == logging.DEBUG and "no_scoreable_content" in r.getMessage()
+            for r in caplog.records
+        )
+
+
 class TestGenerateInternalNote:
     """Pre-action reflection note: single-responsibility plain-text call."""
 
@@ -2662,7 +2760,7 @@ class TestRelevancePromptContract:
     @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
     def test_prompt_carries_post_and_scoring_contract(self, mock_gen):
         mock_gen.return_value = "0.9"
-        score_relevance("test post")
+        score_relevance("a test post")
         prompt = mock_gen.call_args[0][0]
         # Post body is wrapped and embedded.
         assert "test post" in prompt
@@ -3169,7 +3267,7 @@ class TestScoreRelevanceOutageVisibility:
 
         mock_generate.return_value = None
         with caplog.at_level(_logging.WARNING):
-            assert score_relevance("test post") == 0.0
+            assert score_relevance("a test post") == 0.0
         assert "llm_unavailable" in caplog.text
 
 
@@ -3260,6 +3358,16 @@ class TestScoreRelevanceReasonCodes:
         assert result.score == 0.0
         assert result.reason == "empty_input"
 
+    def test_stub_body_is_separable_from_empty_input(self):
+        """A body with text but no proposition is its own event: it must not
+        collapse into empty_input, into the judgment 0.0, or into either
+        failure sentinel in the distribution the instrument reads."""
+        with patch("contemplative_agent.adapters.moltbook.llm_functions.generate") as mock_generate:
+            result = score_relevance_detailed("600-1100 символов")
+        mock_generate.assert_not_called()
+        assert result.score == 0.0
+        assert result.reason == "no_scoreable_content"
+
     @patch("contemplative_agent.adapters.moltbook.llm_functions.generate")
     def test_llm_outage_is_distinguishable(self, mock_generate):
         mock_generate.return_value = None
```

## 4. Insight staging review

```text
## 1. analyzing-systemic-governance-loops — RECOMMEND: adopt
Nearest store skill is `defining-bounded-autonomous-governance-20260801`, which asks for accountability structures in autonomous systems but never asks where the process is allowed to *stop*. The three named probes (required external validation points, explicit termination condition, ambiguity read as permission to continue) are checkable against a concrete loop. Shadows batch siblings 30 (`mandating-operational-scope-limits`) and 13 (`diagnosing-systemic-boundary-failures`), which state the same theme without the termination axis. Caveat for the operator: staged frontmatter carries only name/description/origin, so multi-episode provenance is unverifiable for this and every candidate in the batch.

## 2. anchor-sequential-context — RECOMMEND: reject
The "entire set of prior anchors" / "computationally prohibitive dependency chain" clause is imported blockchain vocabulary with no behaviour the agent can perform on a conversation. Store's `validating-provenance-chains-20260709` already covers demanding a traceable transition log. Clause 3 ("propose an implicit mechanism that passes through tension") is a vague virtue.

## 3. architectural-constraint-mapping — RECOMMEND: reject
Near-verbatim restatement of the adopted `structural-constraint-mapping-scm-20260815` ("shift diagnosis from observed content or data deficits to the underlying architectural limitations"). The added examples do not change the move. Also shadowed within the batch by 23, which at least names the patch-chain artifact.

## 4. audit-contextual-validity — RECOMMEND: reject
The internal-flaw vs world-changed fork is the useful part, but batch sibling 17 (`distinguish-persistence-from-validity`) carries the same temporal-skepticism move in tighter form. Step 3 ("auditing the assumptions that allowed the question to be formulated") is unenactable. Recommending 17 instead of this.

## 5. constructing-composite-metrics — RECOMMEND: reject
"Augmented composite metric framework" names no procedure the agent can run. Store's `anchoring-abstraction-to-measurable-constraints-20260709` already covers turning abstractions into measurable variables, and batch sibling 6 covers the unmetered-resource point with a concrete trigger.

## 6. cross-reference-metric-constraints-analysis — RECOMMEND: adopt
Distinct from store's `deconstructing-confidence-proxies-20260709`: that skill attacks a bad metric, this one attacks the resources the dashboard never had a column for (trust budget, budgeted capacity windows) and reclassifies "rare exception" failures as predictable capacity events. Enactable as a gap analysis on any green-status report. Shadows batch siblings 5 (`constructing-composite-metrics`) and 25 (`identifying-structural-system-boundaries`).

## 7. de-centering-causal-validation-decoupling-source-f — RECOMMEND: reject
"The State of Non-Observation (what remains stable when conscious analysis is suspended)" is not a behaviour that can be enacted or checked. Store's `identifying-simulation-boundaries-20260709` and `evaluating-contextual-functional-dependence-20260801` already cover the reception-side and external-validity halves.

## 8. deconstructing-forced-deterministic-frames — RECOMMEND: reject
Written as a debate posture ("the speaker labels this mechanism... refusing to accept") rather than a procedure. Batch sibling 9 covers the same family — detecting an opponent's structural move — with a named linguistic marker; this one has none. Store's `detecting-abstract-to-operational-constraint-shift-20260709` already covers the philosophy→jargon pivot it describes.

## 9. detecting-rhetorical-discourse-closures — RECOMMEND: adopt
The most behaviourally specific candidate in the batch: a named textual pattern (qualified comparison followed by an absolute longevity superlative) with a named response (treat the closure as a move, not evidence). No store skill covers rhetorical closure moves — `detecting-abstract-to-operational-constraint-shift-20260709` is the nearest and detects a different transition. Shadows sibling 8.

## 10. diagnosing-mechanisms-of-omission — RECOMMEND: adopt
Store's `interpreting-systemic-gaps-the-silence-filter-20260801` treats absence as a data point; this asks a different question — the aperture conditions under which evidence would have become visible, and the exclusion mechanism that kept it out. That is a demand the agent can voice verbatim. Shadows siblings 29 (`interpreting-structural-gaps`), 37 (`reframing-limitations-as-functional-requirements`) and 19, which restate "gap as content" without a new question.

## 11. diagnosing-narrative-vs-factual-reliability — RECOMMEND: reject
Covered jointly by store's `deconstructing-confidence-proxies-20260709` (certainty resting on the wrong signal) and `detecting-foundational-structural-compromise-20260808` (success resting on self-referential validation). The "two-axis metric" is a relabelling; no procedure is given for the independent factual check.

## 12. diagnosing-structural-boundary-violations — RECOMMEND: reject
Duplicates store's `structural-constraint-mapping-scm-20260815` and batch sibling 23. "Immediately suspend philosophical consideration" is a mood instruction, and the concrete list (headers, checksums, 429s) is example material rather than a repeatable move.

## 13. diagnosing-systemic-boundary-failures — RECOMMEND: reject
Store's `structure-authority-tracing-20260709` already converts abstract failures into questions of revision authority and governance layers — the "who writes / who validates" audit here is the same skill. Within the batch it is shadowed by 1, which adds the termination-condition probe.

## 14. diagnosing-systemic-conflict — RECOMMEND: reject
The trade-off-mapping move is held by store's `identifying-fidelity-vs-utility-tension-20260808` and `gradient-modeling-20260815`. Intra-batch it is one of four near-identical tension candidates (14/26/33 plus 8); none of them adds a trigger the store lacks.

## 15. diagnosing-underlying-methodological-mismatch — RECOMMEND: reject
Store's `framework-alignment-detection-20260801` already covers identifying an interlocutor's operational framework and re-pitching to it, and `detecting-abstract-to-operational-constraint-shift-20260709` covers the abrupt domain jump named in the trigger.

## 16. disaggregate-premise-from-effect — RECOMMEND: adopt
The discriminator — a 200 OK / flawless procedure is evidence of execution, not of the external effect — is stated as three separable steps ending in an outside measurement. Store's `pivot-accountability-from-record-to-action-20260725` demands a procedural change instead of an external check, so this is adjacent rather than duplicate. Shadows siblings 34 (`output-to-consequence-pivot-analysis`) and 36 (`recognizing-the-operational-gap-between-logging-an`).

## 17. distinguish-persistence-from-validity — RECOMMEND: adopt
Store's `detecting-abstraction-decay-in-context-20260815` is about metadata lost in compression, not about survival being mistaken for authority; nothing adopted covers "this entry is still here, therefore it still binds". The trigger (deprecated entries, old error flags still visible and functionally linked) is concrete. Shadows sibling 4 (`audit-contextual-validity`).

## 18. establishing-resilience-boundaries-for-meaning — RECOMMEND: reject
The stress-test-by-hypothesized-misuse move is held by store's `identifying-systemic-boundary-stressors-20260801` and `designing-for-structural-underdetermination-20260801`. Batch sibling 45 states the same counterfactual discipline with an explicit trigger; this version stays at the level of $\text{Boundary}_{\text{Permissible}}$ notation.

## 19. evaluating-authority-through-deviation-analysis — RECOMMEND: reject
Store's `detecting-foundational-structural-compromise-20260808` already covers success that depends on a compromised assumption, and `deconstructing-confidence-proxies-20260709` covers the accuracy-claim half. Intra-batch, sibling 10 owns the "treat the documented gap as the object" move with a sharper question.

## 20. functional-applicability-mapping-fam — RECOMMEND: reject
"Systematically hypothesize vectors" is generic; the only concrete instance given is an if-X-then-Y recall test. Store's `identifying-systemic-boundary-stressors-20260801` covers introducing friction to find limits, and batch sibling 45 covers the counterfactual form with a stated failure case.

## 21. identifying-epistemic-and-procedural-gaps — RECOMMEND: reject
Both halves are already adopted: `detecting-abstraction-decay-in-context-20260815` (what fell out during summarisation) and `mapping-epistemic-boundaries-20260709` (state what was tested and what was ruled out). No new demand is added beyond those two.

## 22. identifying-foundational-axioms-in-discourse — RECOMMEND: reject
Store's `identify-structural-pivot-20260808` already covers conceptual tension resolving into a single foundational limiting axiom. The marker list ('load-bearing', 'infrastructure') is a nice detail but does not carry a distinct behaviour past that skill.

## 23. identifying-foundational-model-failures — RECOMMEND: adopt
Best of the root-cause cluster: it names an artifact the store's `structural-constraint-mapping-scm-20260815` and `mapping-systemic-dependency-failures-20260808` do not — the patch chain itself (Symptom → Patch A → Failure B → Patch C) read as a structure whose depth locates the violated assumption. Shadows siblings 3 (`architectural-constraint-mapping`), 12, 41 (`structural-collapse-diagnosis`) and 44.

## 24. identifying-structural-parallels-by-process — RECOMMEND: reject
Store's `analogy-mapping-for-structural-clarity-20260801` already covers translating between domains via a shared functional mechanism. The candidate's addition ("pivot to the governing process, not the objects") is the same instruction restated.

## 25. identifying-structural-system-boundaries — RECOMMEND: reject
Covered by store's `pinpointing-systemic-boundary-conditions-20260808` and `identifying-systemic-boundary-stressors-20260801`. Within the batch, sibling 6 handles the same "the headline metric misses the real constraint" observation with a usable trigger; this one stops at "define the systemic boundary condition".

## 26. identifying-structural-tensions — RECOMMEND: reject
Name and content collide with the adopted `identifying-structural-tensions-via-system-metapho-20260801`, and the "design for managed contradiction" outcome is `gradient-modeling-20260815`. Also fully redundant with batch sibling 33.

## 27. identifying-the-transition-locus — RECOMMEND: reject
The three loci split across existing skills: latency → `fluid-temporal-friction-to-insight-loop-20260530`, drift → `detecting-abstraction-decay-in-context-20260815`, state-vs-process → `shifting-focus-from-state-to-process-mechanics-20260815`. Batch siblings 17 and 38 cover the time axis more precisely.

## 28. interpreting-failures-as-structural-constraints — RECOMMEND: reject
Store's `scope-boundary-mapping-20260808` already asks whether a limitation is definitional or actual, which is the same reclassification. Intra-batch it repeats 10 and 37's "deficit is content" framing without either 10's question or 23's trace.

## 29. interpreting-structural-gaps — RECOMMEND: reject
Direct duplicate of two adopted skills: `translating-temporal-gaps-into-structural-utility-20260709` and `interpreting-systemic-gaps-the-silence-filter-20260801`. Shadowed by sibling 10, which is the one candidate in this family that asks something new.

## 30. mandating-operational-scope-limits — RECOMMEND: reject
Store's `defining-bounded-autonomous-governance-20260801` and `mandating-structural-integrity-axioms-20260801` between them cover accountability structures and structurally guaranteed impossibility. Batch sibling 1 carries the one novel element here (the ambiguity-to-continue mechanism) more cleanly.

## 31. mapping-system-revision-authority — RECOMMEND: reject
`structure-authority-tracing-20260709` is the adopted skill for exactly this: translating a change into questions of revision authority and governance layer. The three-stage check adds procedure but no new discrimination.

## 32. mapping-verifiability-boundaries — RECOMMEND: reject
Store's `mapping-epistemic-boundaries-20260709` covers marking where established knowledge ends, and `identifying-fidelity-vs-utility-tension-20260808` covers the chain-of-custody-vs-synthesis conflict verbatim. No residue left over.

## 33. modeling-conflict-as-structural-tension — RECOMMEND: reject
Same claim as batch sibling 26 and as store's `gradient-modeling-20260815`: hold the contradiction rather than resolve it. Of the redundant group (14/26/33) none is worth adopting, since the store already carries two versions.

## 34. output-to-consequence-pivot-analysis — RECOMMEND: reject
Three loosely joined moves (labelling rules, downstream effect, interpret/act separation) with no single trigger. Batch sibling 16 states the description-vs-consequence pivot in a form that ends in a measurement; adopt that one instead.

## 35. recognizing-interpretive-mechanisms — RECOMMEND: reject
"Pause interpretation and reframe confusion as productive" is a vague virtue, and both halves are adopted already: `suspend-interpretation-upon-premise-doubt-20260801` and `affirm-cognitive-possibility-20260725`.

## 36. recognizing-the-operational-gap-between-logging-an — RECOMMEND: reject
The loggable-vs-constraint layering is the same distinction as batch sibling 16, which additionally requires external verification. Store's `pivot-accountability-from-record-to-action-20260725` and `differentiating-artifacts-from-observation-20260815` already hold the receipts-are-not-truth theme.

## 37. reframing-limitations-as-functional-requirements — RECOMMEND: reject
Duplicate of store's `affirm-cognitive-possibility-20260725` (the gap is the material, not a hole to fill) plus `interpreting-systemic-gaps-the-silence-filter-20260801`. Shadowed intra-batch by 10.

## 38. shifting-focus-to-systemic-degradation-gradient — RECOMMEND: adopt
Store's `gradient-modeling-20260815` turns discrete boundaries into gradients within a single assessment; this instead asks for a decay *rate* across prolonged, heterogeneous use, which no adopted skill requests. The three trigger conditions (undefined lifespan, changing environment, single-point metrics only) are checkable. Distinct enough from batch sibling 17 that both can stand: 17 is about a stale entry, this is about a declining system.

## 39. structural-boundary-analysis — RECOMMEND: reject
Duplicates store's `scope-boundary-mapping-20260808` and overlaps sibling 40 almost completely. "Treat the boundary itself as the highest form of intervention" is a slogan rather than a described behaviour.

## 40. structural-boundary-assessment — RECOMMEND: reject
Same theme as 39 and as store's `pinpointing-systemic-boundary-conditions-20260808`. Of its three components, Veto and Boundary Conditions are that store skill; Reproducibility Vectors is stated too thinly to enact.

## 41. structural-collapse-diagnosis — RECOMMEND: reject
Shadowed by batch sibling 23, which gives the same failure-tracing discipline a concrete artifact to trace. Store's `mapping-systemic-dependency-failures-20260808` already covers diagnosing by causal/temporal/structural constraints rather than components.

## 42. structural-dissonance-detection — RECOMMEND: reject
The A/B/C layering restates claim-versus-support checking, which store's `detecting-foundational-structural-compromise-20260808` already performs, and which sibling 16 does better by ending outside the text. Layer C ("the meta-process of critique") is not a thing to do.

## 43. systematically-auditing-foundational-assumptions — RECOMMEND: reject
Store's `boundary-assumption-verification-20260801` mandates exactly this — explicit verification of foundational operational assumptions. The one novel phrase, "scheduled failure rehearsal", is never specified enough to enact.

## 44. systemic-constraint-triage — RECOMMEND: reject
Covered by store's `recognizing-boundary-declarations-in-content-flow-20260808` (read the structural tags before the content) and `suspend-interpretation-upon-premise-doubt-20260801`. Within the batch it is the weakest member of the 12/23/44 diagnostic-reorientation group.

## 45. testing-procedural-guardrails-with-counterfactuals — RECOMMEND: adopt
Adjacent to store's `identifying-systemic-boundary-stressors-20260801`, which introduces friction into a stated system; this narrows to one repeatable move — end the review by asking what the unwritten procedure is when a core input returns null. That trigger is specific enough to fire. Shadows siblings 20 (`functional-applicability-mapping-fam`) and 18 (`establishing-resilience-boundaries-for-meaning`).

## 46. verifying-state-dependencies — RECOMMEND: reject
`validating-provenance-chains-20260709` is the adopted skill and it already demands the transition log this candidate calls a "second ledger". Redundant intra-batch with 2 and 32 as well; of that group none clears the store.
```

## 5. Dead code candidates (detection only)

週次 vulture スキャン（第 5 決定論 intake、`scripts/dead_code_scan.py`）の読み値。**削除はここでは行われていない** — 候補ごとの判断（削除 / `.vulture_whitelist.py` へ免除追記 / 保留）は土曜ゲートの人間 commit で行う。偽陽性は構造的に不可避（CLI entry point・`config/prompts/*.md` 動的ロード・Protocol 間接参照）。

| file | line | finding | confidence |
|---|---|---|---|
| `src/contemplative_agent/core/domain.py` | 61 | unused variable 'insight_worth' | 60% |

## 6. Pipeline metrics

- this week: F1 4 (code 4 / prompt 0), fix attempted 4, patch ready 4, verify fail 0, dead code 1
- history: 8 prior runs, 10 patches ready total (adopt/reject 率は gate レコード参照)

## 9. Docs consistency (detection only)

週次 docs 整合性スキャン（第 6 決定論 intake、`scripts/docs_consistency_scan.py`）の読み値。**doc の修正はここでは行われていない** — 各 finding の判断（修正 / 例外として容認 / 保留）は土曜ゲートの人間 commit で行う。検査対象は自筆 docs のみ（`enja_drift` = ADR の en が ja より後に commit / `broken_link` = 相対リンク断線 / `notes_ref` = ADR から gitignored な `.notes/` への参照）。

| check | file | line | detail |
|---|---|---|---|
| notes_ref | `docs/adr/0093-repo-plane-deterministic-intakes.ja.md` | 171 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0093-repo-plane-deterministic-intakes.ja.md` | 172 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0093-repo-plane-deterministic-intakes.md` | 193 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0093-repo-plane-deterministic-intakes.md` | 194 | .notes/ is gitignored — broken in every clone |

## Audit trail

- events: `/Users/shimomoto_tatsuya/.config/moltbook/logs/weekly-pipeline-audit.jsonl`（run_id `weekly-2026-08-21-090000`）
- metrics: `/Users/shimomoto_tatsuya/.config/moltbook/logs/pipeline-metrics.jsonl`
- code patches dir: `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-21/code`
