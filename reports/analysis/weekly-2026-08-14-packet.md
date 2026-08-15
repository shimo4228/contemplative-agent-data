# Weekly Decision Packet — 2026-08-14

- Run: `weekly-2026-08-14-090000`
- Generated: 2026-08-15T01:30:01.789564+00:00
- Findings: F1 3 / F2 3 / F3 5
- Reason codes this run: FIX_TIMEOUT, IDENTITY_STAGING_BUSY, NO_RECURRENCE, SCOPE_ESCALATED

## 1. Decision inventory

- code patch: 0 件（apply → 単一 commit の対象、+2 件は §3 の全文ゲートへ昇格（apply 対象外））
- prompt diff: 2 件（本文全文を下に提示 — 個別承認、うち 2 件は code scope からの昇格）
- insight: 49 件（`adopt-staged` の対象）
- pipeline improvement: 0 件
- dead code candidate: 1 件（検出のみ — 削除・whitelist の判断は人間）
- docs consistency: 22 件（検出のみ — doc 修正は人間 commit、§9 参照）

## 2. Code fixes (unattended, Verify-passed where noted)

| finding | scope | attempts | result | reviewer | patch / reason |
|---|---|---|---|---|---|
| F1.1 | code → **SCOPE_ESCALATED** | 1 | patch_ready | APPROVE | `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-14/prompt/F1.1.patch` |
| F1.2 | code → **SCOPE_ESCALATED** | 1 | patch_ready | APPROVE | `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-14/prompt/F1.2.patch` |
| F1.3 | code | 1 | failed | — | `FIX_TIMEOUT` |

### Review notes (final round, full text)

レビュー本文は LLM 出力（finding 由来の連鎖）— 検査者の所見であって承認ではない。CONCERNS のまま採用する判断は人間に属する。

#### F1.1 — APPROVE

```text
VERDICT: APPROVE

- **Implements the finding as specified, verified against live data.** Replaying the join's logic over the real `audit.jsonl` (972 records) for the actual 08-08→08-15 window reproduces exactly the answer the report could not give: `constitution/` changed and carries an `amend-constitution`/`approved` row at `2026-08-09T12:10:35+00:00`. Schema assumptions hold (`path` present in 972/972; decision vocabulary is exactly `{staged, rejected, approved}`, matching the `counts` dict at `scripts/value_layer_approval_join.py:~196`), and `_matches_section` (`:113-123`) partitions real paths correctly (909 skills / 36 rules / 14 identity / 6 constitution; only `knowledge.json` migrations unmatched, which `test_knowledge_json_belongs_to_no_value_layer_section` pins). The truncation-reservation logic is grounded, not invented: the live week is 110 skills rows where a naive head-of-25 slice yields **0** of the 8 approved rows. Gates are intact — `tests/test_weekly_analysis_shell.py` is additive only, no assertion or lint config loosened — and regression coverage exists in both directions (`test_state_diff_sections_carry_their_approval_provenance` for the gap itself, `test_a_missing_audit_log_never_reads_as_a_missing_approval` as the ADR-0077 fault column). Scope: `docs/CODEMAPS/architecture.md` is the one file the finding does not name, but CLAUDE.md's 鮮度規約 mandates a same-PR Data Flow update for 段構成 changes, so it is in-scope by convention rather than creep.

- **The join is directory-level while the prompt calls the rows "matching" — `config/prompts/weekly-analysis.md:36`.** One approved row anywhere under `skills/` clears the alarm for the whole section, even if the file the diff actually added is a different, unapproved one. At 110 rows/week for `skills/` this is not hypothetical. The coarseness is inherited from the finding's own spec ("whose `path` falls under that section's directory"), so it is faithful, but "cite the `ts` and `content_hash` of the matching row(s)" invites the report to assert per-file approval the join never established. Low-volume sections (`identity.md`, `constitution/`) are unaffected.

- **`_clean` (`scripts/value_layer_approval_join.py:126-135`) forks the shared neutralizer and drops backtick escaping.** Every other LLM-facing `scripts/` render imports `md_safe` from `_md.py` (`api_drift_scan.py:62`, `cross_day_duplicate_scan.py:49`, `log_anomaly_sweep.py:51`, `state_invariant_check.py:32`), and `cross_day_duplicate_scan.py:270` states the rule explicitly ("every LLM-facing scripts/ render goes through it") even where it is a no-op. `_clean` escapes `|` but not `` ` ``. No exploit path exists today — all five rendered fields are closed-vocabulary and self-written — but the script's own docstring (`:37-40`) adopts a defense-in-depth posture on durable state and then covers less than the shared helper it declined to use. Same pattern with `JoinUnavailable` (`:62`) vs `_scan.ScanError`, though `value_layer_due_check.py:44` already set a forking precedent there.

- **Interpretive instruction is duplicated between code and prompt.** The "report it as the observation it is, not as a confirmed gate bypass" guidance lives both in `format_reading` (`scripts/value_layer_approval_join.py:~250`) and in `config/prompts/weekly-analysis.md:37`. That text is read by the LLM and steers its conclusion, which is what the プロンプト外出し convention (ADR-0054) targets; existing intakes set a precedent for inline instrument prose, so this is consistency rather than violation — but two copies of the same hedge will drift, and the code copy is the one no prompt review will see.

- **Robustness is asymmetric on malformed records.** A non-string `path` is explicitly skipped and pinned by `test_a_non_string_path_is_skipped_not_crashed`, but a non-string `decision` (e.g. a JSON list) raises `TypeError` at the `decision in counts` membership test, killing the whole section's reading. It degrades safely — the shell's `|| true` fallback at `scripts/weekly-analysis.sh:~113` renders `unavailable (reason=join-failed)`, never the alarm — so the finding's core invariant holds, but the section's signal is lost rather than the one bad row.
```

#### F1.2 — APPROVE

```text
VERDICT: APPROVE

- **Implements the finding's structural change, nothing more** — the cut lives only in the instrument's key path (`scripts/log_anomaly_sweep.py:253`, inside `normalize_with_origin`), and all four producers named in the finding are untouched in the diff: `adapters/moltbook/publish.py:105` (`log_published`), `reply_handler.py:343`, `feed_manager.py:482`, `post_pipeline.py:443`, `core/distill.py:885`. The T-LOG-DEBUG-CONTENT gate `tests/test_publish_logging.py:39,89,104` is unmodified, so the producer-side repair keeps its enforcement.

- **Regexes verified against the real format strings, not just the fixtures** — I replayed the post-patch normalizer over the four producers' actual `%`-formats. All cut correctly; `>> New post [%s] (id=%s): %d chars: %s` genuinely requires the earlier cut the diff chose (`log_anomaly_sweep.py:147`), since the generated title precedes the count. Adversarial case: a body containing a second `"… 99 chars: SECRETBODY"` does **not** leak, because `.*?` is non-greedy and stops at the first boundary — the one way this family of fix usually fails.

- **Gate integrity intact; regression coverage matches the finding's two named carriers** — the test hunk is pure addition (`tests/test_log_anomaly_sweep.py:159-276`), no existing assertion relaxed. `test_body_text_reaches_neither_the_state_file_nor_the_render` covers exactly the snapshot and the table `weekly-analysis.sh:240` feeds to an LLM, and `test_an_unrelated_line_mentioning_chars_is_not_cut` pins the allowlist as an allowlist. No other test or script asserts the old signature shape or the edited render footnote (`log_anomaly_sweep.py:443`); `tests/benchmark_distill.py:110` parses the distill line from log records directly, so it is unaffected.

- **Scope: `docs/CODEMAPS/architecture.md` is not named by the finding, but is convention-mandated** — this change alters a pipeline mechanism (the sweep's key), which CLAUDE.md's freshness rule requires be reflected in the Data Flow section in the same PR. Accounted for, not scope creep. The retained counterparty address is safe: it is `log_safe_identifier(replier_name)` (`reply_handler.py:334`), already sanitized, so the cut does not trade one injection channel for another.

- **Two minor residuals for the ledger, neither blocking** — (a) the new architecture.md paragraph does not say that re-keying these families resets their 🆕 baseline once at the next sweep, a self-correcting one-week reading artifact the same paragraph's neighbors do call out for rotation; (b) `adapters/dialogue/peer.py:97` logs `line[:80]` of untrusted peer JSON at WARNING — the same residue class in a family the finding did not name, correctly left out of this diff rather than folded in as a superset.
```

## 3. Prompt-scope diffs (full text — behavior-shaping)

### F1.1.patch

**SCOPE_ESCALATED** — code scope として起票されたが、`^(src|scripts|tests)/` の外に触れたため全文ゲートへ昇格した。scope 分類の誤りではなく、要約行での通過を防ぐ設計上の昇格。

触れたパス: `config/prompts/weekly-analysis.md`, `docs/CODEMAPS/architecture.md`

````diff
diff --git a/config/prompts/weekly-analysis.md b/config/prompts/weekly-analysis.md
index ec03b51..b167920 100644
--- a/config/prompts/weekly-analysis.md
+++ b/config/prompts/weekly-analysis.md
@@ -32,6 +32,10 @@ Summarize changes to the agent's internal state during this period:
 - **Skills**: List all skills at period end. Note any added/removed/modified.
 - **Rules**: List all rules at period end. Note any added/removed/modified.
 - **Knowledge**: Pattern count at start vs end. Carry the source label the input gives you — the state diff reports *committed snapshots of the data repo* (with commit sha and date), the invariant check reports the *live store at report-generation time* (whose `total` includes tombstones). These answer different questions and legitimately differ; report each with its label rather than treating the gap as a contradiction or picking one as canonical.
+- **Approval provenance** (from the **Approval provenance** block inside each state-diff section): every value-layer diff above is annotated with the in-window `logs/audit.jsonl` rows for that section (ADR-0012 gate; `ts` / `command` / `decision` / `source` / `content_hash`). Read it before writing the Identity / Constitution / Skills / Rules bullets, and state which of these three the section is — do not report a change as unverifiable when the block answers it:
+  - **approved rows present** — say so and cite the `ts` and `content_hash` of the matching row(s);
+  - **⚠️ NO APPROVED RECORD while the section shows a diff** — this is the alarm condition, and the strongest thing B can report. State it plainly, and state it as an observation: a sync lag or an approval made before the window's start commit produces the same shape, so it is "no approval row in this window", not a proven gate bypass;
+  - **unavailable (reason=…)** — the instrument could not read the log. Report the reason code. Never convert this into a claim that no approval exists.
 - **Operational drift** (from the provided *Log Anomaly Sweep* and *State Invariant Check*): surface any anomaly type flagged 🆕 (new since last sweep) or sharply spiking (high Δ), and any invariant at ⚠️ WARN or ❌ FAIL. These are deterministic signals — report them as observations (what changed, how much); proposing fixes belongs to the downstream diagnosis step, not this report.
 - **Skill selection** (from the provided *Skill-selection shadow reading*): which skills pass-1 actually selected this week — selection frequency, verdict distribution (judged vs fail-open), hallucination rate, never-selected tail. This is the measured middle link between *installed* (state diff) and *vocabulary in output* (E): when A or E attributes output vocabulary to a skill, check the attribution against this list instead of inferring selection from vocabulary. Report the reading as observations; it carries names and counts only.
 
@@ -101,7 +105,7 @@ The "Typical" bucket is required. A 70% middle band that is invisible in good/pr
 The following data will be provided:
 1. **Methodological Principles** (`principles.md`) — Principle 3 (quote-based depth) applies to this report. Other principles apply to the downstream diagnosis step.
 2. **Daily comment reports** for the analysis period
-3. **Agent state diffs** (identity, constitution, skills, rules, knowledge count) — if available
+3. **Agent state diffs** (identity, constitution, skills, rules, knowledge count) — if available. Each value-layer section carries an **Approval provenance** block: the deterministic join of that section's directory to the in-window ADR-0012 approval rows in `logs/audit.jsonl` (dense fields only — no lineage lists, no free text, no target paths); read it for B's approval-provenance note
 4. **Log Anomaly Sweep** — deterministic ranking of log anomalies by novelty (🆕 = new since last sweep) then frequency delta; read it for B's operational-drift note
 5. **State Invariant Check** — deterministic ✅/⚠️/❌ checks over knowledge.json / agents.json; read it for B's operational-drift note
 6. **Skill-selection shadow reading** — deterministic aggregate of the pass-1 skill-selection log (selected skill names with frequency, verdict distribution, hallucination rate, never-selected tail; names and counts only); read it for B's skill-selection note
diff --git a/docs/CODEMAPS/architecture.md b/docs/CODEMAPS/architecture.md
index cc19745..2b5d56b 100644
--- a/docs/CODEMAPS/architecture.md
+++ b/docs/CODEMAPS/architecture.md
@@ -513,10 +513,11 @@ carry the signal; the CLI prints that note with every reading.
 
 ### weekly-analysis  [`scripts/weekly-analysis.sh`, ADR-0040]
 
-Runs outside the agent process (launchd → `claude -p`), assembling a prompt from operator-facing artifacts plus **five deterministic intakes**, then a diagnosis companion (`weekly-report-diagnosis` skill) produces the F sections.
+Runs outside the agent process (launchd → `claude -p`), assembling a prompt from operator-facing artifacts plus **six deterministic intakes**, then a diagnosis companion (`weekly-report-diagnosis` skill) produces the F sections.
 
 ```text
 collect: daily comment-reports + data-repo state diff + previous N reports
+       + value_layer_approval_join.py (per state-diff section: audit.jsonl approval rows)
        + log_anomaly_sweep.py    (event stream: *.log + audit.jsonl; novelty state + corpus census)
        + state_invariant_check.py (accumulated state: knowledge.json / agents.json)
        + cross_day_duplicate_scan.py (published-body identity: episode logs → digests)
@@ -531,7 +532,9 @@ promote:  mv tmp → weekly-<end>.md        (atomic; a failure leaves the prior
 translate: best-effort .ja.md (sonnet); failure never rolls back the promotes
 ```
 
-Order is load-bearing. The sweep's Δ / 🆕 columns and the drift scan's new/removed pairs are defined against their last committed snapshots, so both run `--no-update --emit-state` and their baselines are committed after the report lands — a run that produces nothing must spend nothing (findings F1.2; two consecutive weeks lost). The invariant check, the duplicate scan and the skill-selection reading hold no state and are absolute readings, so they need no such ordering.
+Order is load-bearing. The sweep's Δ / 🆕 columns and the drift scan's new/removed pairs are defined against their last committed snapshots, so both run `--no-update --emit-state` and their baselines are committed after the report lands — a run that produces nothing must spend nothing (findings F1.2; two consecutive weeks lost). The invariant check, the duplicate scan, the skill-selection reading and the approval join hold no state and are absolute readings, so they need no such ordering.
+
+The approval join (2026-08-15, findings F1.1) annotates **each value-layer section of the state diff** — `identity.md`, `constitution/`, `skills/`, `rules/` — with the in-window ADR-0012 approval rows from `logs/audit.jsonl` whose `path` falls under that section's directory. It closes a gap in the diff itself: the diff showed *what* changed and nothing about *whether it passed the gate*, so the 2026-08-15 report's strongest claim ("whether it passed through the `amend-constitution` approval path is not visible in the operator-facing data supplied here") was bounded by missing data rather than by analysis, in a chain that already reads that file for the ADR-0091 identity-cadence due. Three renderings, deliberately distinct: approved rows present (citable `ts` + `content_hash`), **no approved row while the section shows a diff** (the alarm — reported as an observation, since a sync lag or a pre-window approval produces the same shape), and `unavailable (reason=…)` when the log is missing or unreadable — an unavailable instrument must never render as the alarm, or the report manufactures a gate-bypass claim out of its own blindness (gated by `test_a_missing_audit_log_never_reads_as_a_missing_approval`). Window: the two data-repo **commit timestamps**, half-open (`start < ts <= end`) — anything approved at or before the start commit is already inside that commit's tree and so is not part of the diff; the calendar bounds would mis-window by the sync lag. Rendered fields are `ts` / `command` / `decision` / `source` / `content_hash` only: `reason` is operator free text, `source_ids` an unbounded lineage list, and target paths carry skill filenames slugified from distilled pattern text — all three stay out (same send-the-shape choice as ADR-0083).
 
 The drift scan (2026-08-06) diffs the per-endpoint response-key vocabulary the client already records in `api-audit.jsonl` (2xx envelopes only, so outages do not read as schema changes) and tracks the `POST /verify` consecutive-failure run against the platform's 10-failure suspension rule. It exists because the platform ships API changes unannounced (observed: the `check_in` key appearing on `/home` in 2026-08, carrying role "standing instructions" — a third-party injection channel the adapter deliberately never consumes, gated by `tests/test_home_field_allowlist.py`). The spec (`skill.md`) is untrusted external text: it is never fetched in the unattended chain, and on drift the rendered section directs the re-read to the Saturday gate.
 
@@ -539,7 +542,7 @@ The sweep's signature is keyed on **level + message**, with the dotted `%(name)s
 
 The sweep has **no time window**: it counts every line each allowed file currently holds, so a row's `Count` spans that file's lifetime — and the files rotate on different schedules (`ollama-serve.log` nightly since 2026-08-01, `agent-launchd.log` weekly via `backup-runtime.sh`, the one-shot `insight-` / `distill-launchd.log` never), which makes two rows of one table not necessarily commensurable. Rotation also moves the novelty baseline: lines leave the `*.log` glob, counts fall, and known signatures re-appear as 🆕 — once a rare footnote, the steady state since nightly rotation shipped. Filtering by timestamp would discard signal, so the instrument states its basis instead (findings F1.1, 2026-08-07): a per-file **corpus census** (name, lines read, signal lines) is written to a sidecar `<state>.corpus.tsv` — a sidecar because `read_state` silently drops any state-file line whose first field is not an int, so a header row there would vanish on read — and rendered above the table beside the previous sweep's three figures, with an explicit "🆕 and Δ are not comparable to last week's" sentence when the corpus lost more than 10% of its lines. The census is written *before* its snapshot (the snapshot's existence is the shell's "sweep completed" signal) and promoted in lockstep with it; if the pair breaks, the shell deletes the old census so the next run reports "no previous census" rather than asserting a comparison against a corpus that no longer exists.
 
-Injection boundary: the sweep, the invariant check and the drift scan must never read episode logs (the drift scan reads only the self-written `api-audit.jsonl`; the platform-controlled key names it renders are Markdown-escaped and length-capped). The duplicate scan does, and is the only intake permitted to — it emits **only** 12-hex SHA-256 digests, counts, filename-derived dates and the fixed `{post, reply, comment}` vocabulary (ADR-0083, gated by `TestOutputBoundary`). The skill-selection reading (2026-08-08, findings F1.4) reads the self-written `skill-selection-*.jsonl` shadow log (ADR-0076): the *selected* middle link between *installed* (state diff) and *vocabulary in output* (section E) was already logged per publish action but never supplied to the report. Its records embed the selection situation — untrusted post bodies — so the renderer (`format_skill_selection_report`, the same one behind `report --skill-selection`) emits **catalog** names and counts only, never the situation strings (same ADR-0083 boundary; gated by `test_skill_selection_reading_reaches_the_prompt_names_only`). "Catalog names" is load-bearing since 2026-08-08, when the reading gained a per-name rejected-name tally: a *rejected* name is by definition a string that matched nothing in the catalog, i.e. free model output from a prompt that embeds untrusted post bodies — the 2026-08-08 backfill reading measured 12% of them as fragments bled in from elsewhere in the prompt. It is therefore the one string in this reading that is not drawn from a closed, self-written vocabulary, and `format_skill_selection_report` withholds it unless a caller passes `include_rejected_names=True`. **The default is the restrictive one**, so a new caller is safe by omission and the weekly script needs no knowledge of which side of the boundary it is on; only `report --skill-selection` (terminal, human reader — the reader the tally exists for) opts in. The weekly prompt still receives the tally's *shape* — distinct-name count, emissions, and each entry's nearest catalog name with its surface-similarity distance — because those are catalog-derived or numeric. Same choice ADR-0083's duplicate scan made: send the shape, not the content. All five are observability: a failure degrades to a "not available" stub and never breaks the report.
+Injection boundary: the sweep, the invariant check and the drift scan must never read episode logs (the drift scan reads only the self-written `api-audit.jsonl`; the platform-controlled key names it renders are Markdown-escaped and length-capped). The duplicate scan does, and is the only intake permitted to — it emits **only** 12-hex SHA-256 digests, counts, filename-derived dates and the fixed `{post, reply, comment}` vocabulary (ADR-0083, gated by `TestOutputBoundary`). The skill-selection reading (2026-08-08, findings F1.4) reads the self-written `skill-selection-*.jsonl` shadow log (ADR-0076): the *selected* middle link between *installed* (state diff) and *vocabulary in output* (section E) was already logged per publish action but never supplied to the report. Its records embed the selection situation — untrusted post bodies — so the renderer (`format_skill_selection_report`, the same one behind `report --skill-selection`) emits **catalog** names and counts only, never the situation strings (same ADR-0083 boundary; gated by `test_skill_selection_reading_reaches_the_prompt_names_only`). "Catalog names" is load-bearing since 2026-08-08, when the reading gained a per-name rejected-name tally: a *rejected* name is by definition a string that matched nothing in the catalog, i.e. free model output from a prompt that embeds untrusted post bodies — the 2026-08-08 backfill reading measured 12% of them as fragments bled in from elsewhere in the prompt. It is therefore the one string in this reading that is not drawn from a closed, self-written vocabulary, and `format_skill_selection_report` withholds it unless a caller passes `include_rejected_names=True`. **The default is the restrictive one**, so a new caller is safe by omission and the weekly script needs no knowledge of which side of the boundary it is on; only `report --skill-selection` (terminal, human reader — the reader the tally exists for) opts in. The weekly prompt still receives the tally's *shape* — distinct-name count, emissions, and each entry's nearest catalog name with its surface-similarity distance — because those are catalog-derived or numeric. Same choice ADR-0083's duplicate scan made: send the shape, not the content. The approval join reads only the self-written `audit.jsonl` and renders five closed-vocabulary fields, still squashed of non-printables, length-capped and pipe-escaped — a record is durable state, so a malformed row must not break out of its table cell into report prose. All six are observability: a failure degrades to a "not available" stub and never breaks the report.
 
 ### weekly-pipeline  [`scripts/weekly-pipeline.sh`, ADR-0085]
 
diff --git a/scripts/value_layer_approval_join.py b/scripts/value_layer_approval_join.py
new file mode 100644
index 0000000..ca69dcb
--- /dev/null
+++ b/scripts/value_layer_approval_join.py
@@ -0,0 +1,327 @@
+#!/usr/bin/env python3
+"""Join value-layer state diffs to their ADR-0012 approval records (read-only).
+
+The weekly state diff shows *what* changed in identity / constitution /
+skills / rules, and nothing about *whether the change passed the approval
+gate*. That gap produced the loudest — and unresolvable — claim in the
+2026-08-15 report: "whether it passed through the ``amend-constitution``
+approval path is not visible in the operator-facing data supplied here".
+``logs/audit.jsonl`` had already answered it. This renders that answer next
+to each diff section, so a value-layer change arrives either matched to an
+approval row or explicitly flagged as having none.
+
+The *absence* of an approved row for a section that shows a diff is the
+alarm condition. It is therefore distinguished, in the rendering, from the
+two states that must never read as that alarm:
+
+- the audit log is missing or unreadable (``unavailable``, with a reason
+  code) — an unavailable instrument reads zero, not clean (ADR-0077);
+- the section shows no diff (``none (no diff either)``).
+
+Windowing: ``--start`` / ``--end`` are the *commit timestamps* of the two
+data-repo snapshots the diff is taken between, not the report's calendar
+bounds. The interval is half-open, ``start < ts <= end``: anything approved
+at or before the start commit is already inside that commit's tree and so
+is not part of the diff. Passing calendar dates instead would mis-window by
+the sync lag.
+
+Security / boundary (load-bearing):
+- Reads ONLY ``audit.jsonl``, which the agent writes itself (never episode
+  logs — injection boundary).
+- Renders five dense fields per row: ``ts``, ``command``, ``decision``,
+  ``source``, ``content_hash``. ``reason`` is operator free text and
+  ``source_ids`` is an unbounded lineage list; neither is ever rendered.
+  Target paths are not rendered either — skill filenames are slugified from
+  distilled pattern text, so they are the one structural field in a record
+  that carries content-derived bytes.
+- The five rendered fields are written by this codebase from closed
+  vocabularies, but are still squashed of non-printables, length-capped and
+  pipe-escaped at render: a record is durable state, and a malformed row
+  must not break out of its table cell into report prose.
+"""
+
+from __future__ import annotations
+
+import argparse
+import json
+from dataclasses import dataclass
+from datetime import datetime, timezone
+from pathlib import Path, PurePosixPath
+from typing import Any
+
+# Which path shapes belong to which state-diff section. Matched on path
+# components rather than a prefix: the audit log records absolute paths from
+# whatever MOLTBOOK_HOME was live at write time, which is not necessarily the
+# home this scan runs against (backfill runs, relocated data dirs).
+_SECTIONS = ("identity", "constitution", "skills", "rules")
+
+_FIELD_CAP = 80
+_DEFAULT_TOP = 25
+
+
+class JoinUnavailable(Exception):
+    """The reading cannot be produced. Carries a reason code, never zero."""
+
+    def __init__(self, reason: str) -> None:
+        super().__init__(reason)
+        self.reason = reason
+
+
+# ``order=True`` is load-bearing: Row is the tie-breaker in the (ts, row) sort
+# below, so same-second records would raise on comparison without it.
+@dataclass(frozen=True, order=True)
+class Row:
+    ts: str
+    command: str
+    decision: str
+    source: str
+    content_hash: str
+
+
+@dataclass(frozen=True)
+class Reading:
+    section: str
+    changed: bool
+    rows: tuple[Row, ...]
+    approved: int
+    staged: int
+    rejected: int
+    other: int
+    unparsable: int
+    truncated: int
+    window_start: str
+    window_end: str
+
+
+def _parse_ts(raw: object) -> datetime | None:
+    """ISO-8601 -> aware UTC. Naive input is read as UTC.
+
+    The ``Z`` replace exists for the ``requires-python = ">=3.10"`` floor;
+    3.11+ ``fromisoformat`` accepts it natively.
+    """
+    if not isinstance(raw, str):
+        return None
+    try:
+        parsed = datetime.fromisoformat(raw.replace("Z", "+00:00"))
+    except ValueError:
+        return None
+    if parsed.tzinfo is None:
+        return parsed.replace(tzinfo=timezone.utc)
+    return parsed.astimezone(timezone.utc)
+
+
+def _matches_section(raw_path: object, section: str) -> bool:
+    if not isinstance(raw_path, str) or not raw_path:
+        return False
+    parts = PurePosixPath(raw_path).parts
+    if not parts:
+        return False
+    if section == "identity":
+        return parts[-1] == "identity.md"
+    # A directory-shaped section: some *parent* component carries the name.
+    # Checking parents only keeps a file literally named ``skills`` out.
+    return section in parts[:-1]
+
+
+def _clean(value: object) -> str:
+    """Render one audit field: non-printables squashed, capped, pipe-escaped."""
+    if value is None:
+        return "—"
+    text = "".join(ch if ch.isprintable() else " " for ch in str(value)).strip()
+    if not text:
+        return "—"
+    if len(text) > _FIELD_CAP:
+        text = text[: _FIELD_CAP - 1] + "…"
+    return text.replace("|", "\\|")
+
+
+def load_records(audit_path: Path) -> tuple[list[dict[str, Any]], int]:
+    """Read audit.jsonl -> (records, unparsable line count).
+
+    A missing or unreadable log raises rather than returning empty: "no
+    records" and "cannot tell" are the two states this instrument exists to
+    keep apart.
+    """
+    if not audit_path.is_file():
+        raise JoinUnavailable("audit-log-missing")
+    try:
+        text = audit_path.read_text(encoding="utf-8", errors="replace")
+    except OSError:
+        raise JoinUnavailable("audit-log-unreadable") from None
+    records: list[dict[str, Any]] = []
+    unparsable = 0
+    for line in text.splitlines():
+        line = line.strip()
+        if not line:
+            continue
+        try:
+            record = json.loads(line)
+        except ValueError:
+            unparsable += 1
+            continue
+        if isinstance(record, dict):
+            records.append(record)
+        else:
+            unparsable += 1
+    return records, unparsable
+
+
+def build_reading(
+    records: list[dict[str, Any]],
+    *,
+    section: str,
+    changed: bool,
+    start: datetime,
+    end: datetime,
+    unparsable: int = 0,
+    top: int = _DEFAULT_TOP,
+) -> Reading:
+    """Select the section's in-window rows and tally decisions.
+
+    Pure: takes already-loaded records so the join is reproducible offline
+    from the same inputs.
+    """
+    selected: list[tuple[datetime, Row]] = []
+    counts = {"approved": 0, "staged": 0, "rejected": 0, "other": 0}
+    for record in records:
+        if not _matches_section(record.get("path"), section):
+            continue
+        # Pre-2026-04 records use ``timestamp``; value_layer_due_check.py
+        # recognizes both, so this must too or the two readings disagree.
+        raw_ts = record.get("ts") or record.get("timestamp")
+        parsed = _parse_ts(raw_ts)
+        if parsed is None:
+            unparsable += 1
+            continue
+        if not (start < parsed <= end):
+            continue
+        decision = record.get("decision")
+        counts[decision if decision in counts else "other"] += 1
+        selected.append(
+            (
+                parsed,
+                Row(
+                    ts=_clean(raw_ts),
+                    command=_clean(record.get("command")),
+                    decision=_clean(decision),
+                    source=_clean(record.get("source")),
+                    content_hash=_clean(record.get("content_hash")),
+                ),
+            )
+        )
+    # Sort on the parsed timestamp, with the rendered tuple as tie-breaker so
+    # same-second rows keep a stable order across runs.
+    selected.sort(key=lambda item: (item[0], item[1]))
+    if top <= 0 or len(selected) <= top:
+        shown = selected
+    else:
+        # The cap must never spend itself on rows that do not answer the
+        # question. A busy skills week is ~110 rows dominated by same-second
+        # `staged` batches, so a plain head-of-list slice hid all 8 approved
+        # rows behind 25 staged ones — while the prompt asks the report to
+        # cite an approving row's ts and content_hash. Approved rows are
+        # reserved first, the remaining slots go to the earliest others, and
+        # the shown set is re-sorted chronologically so the table still reads
+        # as a timeline.
+        approved_rows = [item for item in selected if item[1].decision == "approved"]
+        others = [item for item in selected if item[1].decision != "approved"]
+        shown = approved_rows[:top]
+        shown.extend(others[: top - len(shown)])
+        shown.sort(key=lambda item: (item[0], item[1]))
+    return Reading(
+        section=section,
+        changed=changed,
+        rows=tuple(row for _, row in shown),
+        approved=counts["approved"],
+        staged=counts["staged"],
+        rejected=counts["rejected"],
+        other=counts["other"],
+        unparsable=unparsable,
+        truncated=len(selected) - len(shown),
+        window_start=_clean(start.isoformat()),
+        window_end=_clean(end.isoformat()),
+    )
+
+
+def format_unavailable(reason: str) -> str:
+    return (
+        f"**Approval provenance**: unavailable (reason={reason}). "
+        "This is NOT evidence that the change above lacks an approval record — "
+        "the instrument could not read `logs/audit.jsonl`."
+    )
+
+
+def format_reading(reading: Reading) -> str:
+    total = reading.approved + reading.staged + reading.rejected + reading.other
+    lines = [
+        f"**Approval provenance** (`logs/audit.jsonl`, ADR-0012 gate; window "
+        f"{reading.window_start} → {reading.window_end}, exclusive of the start "
+        f"commit): {total} record(s) — approved {reading.approved}, staged "
+        f"{reading.staged}, rejected {reading.rejected}, other {reading.other}."
+    ]
+    if reading.changed and reading.approved == 0:
+        lines.append(
+            "⚠️ NO APPROVED RECORD for a section that shows a diff. The state above "
+            "changed with no matching approval row in this window — report it as the "
+            "observation it is, not as a confirmed gate bypass (a sync lag or a "
+            "pre-window approval also produces this shape)."
+        )
+    elif not reading.changed and total == 0:
+        lines.append("No diff and no approval records — nothing to reconcile.")
+    if reading.rows:
+        lines.append("")
+        lines.append("| ts | command | decision | source | content_hash |")
+        lines.append("|---|---|---|---|---|")
+        lines.extend(
+            f"| {row.ts} | {row.command} | {row.decision} | {row.source} | {row.content_hash} |"
+            for row in reading.rows
+        )
+    if reading.truncated:
+        lines.append("")
+        lines.append(f"({reading.truncated} further record(s) not shown — `--top` cap.)")
+    if reading.unparsable:
+        lines.append("")
+        lines.append(f"({reading.unparsable} audit line(s) unparsable and excluded.)")
+    return "\n".join(lines)
+
+
+def main(argv: list[str] | None = None) -> int:
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument("--audit", required=True, type=Path, help="path to audit.jsonl")
+    parser.add_argument("--section", required=True, choices=_SECTIONS)
+    parser.add_argument(
+        "--diff",
+        required=True,
+        choices=("changed", "unchanged"),
+        help="whether the state-diff section this annotates shows a change",
+    )
+    parser.add_argument("--start", required=True, help="ISO-8601 start-commit timestamp")
+    parser.add_argument("--end", required=True, help="ISO-8601 end-commit timestamp")
+    parser.add_argument("--top", type=int, default=_DEFAULT_TOP, help="max rows rendered")
+    args = parser.parse_args(argv)
+
+    try:
+        start = _parse_ts(args.start)
+        end = _parse_ts(args.end)
+        if start is None or end is None:
+            raise JoinUnavailable("window-unparsable")
+        records, unparsable = load_records(args.audit)
+    except JoinUnavailable as exc:
+        print(format_unavailable(exc.reason))
+        return 0
+
+    reading = build_reading(
+        records,
+        section=args.section,
+        changed=args.diff == "changed",
+        start=start,
+        end=end,
+        unparsable=unparsable,
+        top=args.top,
+    )
+    print(format_reading(reading))
+    return 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/scripts/weekly-analysis.sh b/scripts/weekly-analysis.sh
index 472752d..d5b540a 100755
--- a/scripts/weekly-analysis.sh
+++ b/scripts/weekly-analysis.sh
@@ -92,6 +92,33 @@ if [[ -d "$DATA_REPO/.git" ]]; then
     if [[ -n "$start_commit" ]] && [[ -n "$end_commit" ]] && [[ "$start_commit" != "$end_commit" ]]; then
         echo "State diff: $start_commit (start) -> $end_commit (end)"
 
+        # Commit timestamps bound the approval join below (and label the
+        # knowledge count further down). Computed once so the two readings
+        # cannot drift apart.
+        start_cdate=$(git log -1 --format=%cI "$start_commit" 2>/dev/null || echo "unknown")
+        end_cdate=$(git log -1 --format=%cI "$end_commit" 2>/dev/null || echo "unknown")
+
+        # Approval provenance join (ADR-0012 / ADR-0050, added 2026-08-15).
+        # A value-layer diff alone cannot say whether the change passed the
+        # approval gate — last week's report raised its loudest alarm on
+        # exactly that gap while logs/audit.jsonl (self-written, already read
+        # by this chain's value-layer due check) held the answer. Renders five
+        # dense fields per matching row; never source_ids, never free text,
+        # never target paths. Observability only — a failure must not break
+        # the weekly report, but it must also never read as "no approval",
+        # hence the explicit unavailable line instead of an empty string.
+        approval_join() {  # $1 = section, $2 = changed|unchanged
+            local out
+            out=$(python3 "$PROJECT_ROOT/scripts/value_layer_approval_join.py" \
+                --audit "$MOLTBOOK_HOME/logs/audit.jsonl" \
+                --section "$1" --diff "$2" \
+                --start "$start_cdate" --end "$end_cdate" 2>/dev/null || true)
+            if [[ -z "$out" ]]; then
+                out="**Approval provenance**: unavailable (reason=join-failed). This is NOT evidence that the change above lacks an approval record."
+            fi
+            printf '%s\n' "$out"
+        }
+
         STATE_DIFF+="## Agent State Diff ($START_DATE -> $END_DATE)"$'\n\n'
 
         # Identity
@@ -99,8 +126,10 @@ if [[ -d "$DATA_REPO/.git" ]]; then
         id_diff=$(git diff "$start_commit" "$end_commit" -- identity.md 2>/dev/null || true)
         if [[ -n "$id_diff" ]]; then
             STATE_DIFF+='```diff'$'\n'"$id_diff"$'\n''```'$'\n\n'
+            STATE_DIFF+="$(approval_join identity changed)"$'\n\n'
         else
             STATE_DIFF+="No changes."$'\n\n'
+            STATE_DIFF+="$(approval_join identity unchanged)"$'\n\n'
         fi
 
         # Constitution
@@ -108,8 +137,10 @@ if [[ -d "$DATA_REPO/.git" ]]; then
         const_diff=$(git diff "$start_commit" "$end_commit" -- constitution/ 2>/dev/null || true)
         if [[ -n "$const_diff" ]]; then
             STATE_DIFF+='```diff'$'\n'"$const_diff"$'\n''```'$'\n\n'
+            STATE_DIFF+="$(approval_join constitution changed)"$'\n\n'
         else
             STATE_DIFF+="No changes."$'\n\n'
+            STATE_DIFF+="$(approval_join constitution unchanged)"$'\n\n'
         fi
 
         # Skills
@@ -123,8 +154,10 @@ if [[ -d "$DATA_REPO/.git" ]]; then
             if [[ -n "$skills_diff" ]]; then
                 STATE_DIFF+='```diff'$'\n'"$skills_diff"$'\n''```'$'\n\n'
             fi
+            STATE_DIFF+="$(approval_join skills changed)"$'\n\n'
         else
             STATE_DIFF+="No changes. Files: $(echo "$skills_end" | tr '\n' ', ')"$'\n\n'
+            STATE_DIFF+="$(approval_join skills unchanged)"$'\n\n'
         fi
 
         # Rules
@@ -138,8 +171,10 @@ if [[ -d "$DATA_REPO/.git" ]]; then
             if [[ -n "$rules_diff" ]]; then
                 STATE_DIFF+='```diff'$'\n'"$rules_diff"$'\n''```'$'\n\n'
             fi
+            STATE_DIFF+="$(approval_join rules changed)"$'\n\n'
         else
             STATE_DIFF+="No changes. Files: $(echo "$rules_end" | tr '\n' ', ')"$'\n\n'
+            STATE_DIFF+="$(approval_join rules unchanged)"$'\n\n'
         fi
 
         # Knowledge pattern count
@@ -156,8 +191,8 @@ if [[ -d "$DATA_REPO/.git" ]]; then
         count_end=$(git show "$end_commit":knowledge.json 2>/dev/null | python3 -c "import sys,json; print(len(json.load(sys.stdin)))" 2>/dev/null || echo "N/A")
         start_sha=$(git rev-parse --short=7 "$start_commit" 2>/dev/null || echo "unknown")
         end_sha=$(git rev-parse --short=7 "$end_commit" 2>/dev/null || echo "unknown")
-        start_cdate=$(git log -1 --format=%cI "$start_commit" 2>/dev/null || echo "unknown")
-        end_cdate=$(git log -1 --format=%cI "$end_commit" 2>/dev/null || echo "unknown")
+        # start_cdate / end_cdate come from the approval-join block above —
+        # the same two timestamps label this count and bound that window.
         STATE_DIFF+="Pattern count (data repo, committed snapshots — rows in knowledge.json, tombstones included):"$'\n'
         STATE_DIFF+="$count_start (start, commit $start_sha @ $start_cdate) -> $count_end (end, commit $end_sha @ $end_cdate)"$'\n\n'
         STATE_DIFF+="Not comparable to the State Invariant Check totals below, which read the live store at report-generation time."$'\n\n'
diff --git a/tests/test_value_layer_approval_join.py b/tests/test_value_layer_approval_join.py
new file mode 100644
index 0000000..05ad6d7
--- /dev/null
+++ b/tests/test_value_layer_approval_join.py
@@ -0,0 +1,387 @@
+"""Tests for scripts/value_layer_approval_join.py (approval-provenance join).
+
+The weekly state diff shows *what* changed in the value layer and said
+nothing about *whether it passed the ADR-0012 gate*, so the 2026-08-15
+report raised its loudest alarm on a question ``logs/audit.jsonl`` had
+already answered. This join renders that answer next to each diff section.
+
+What is pinned here is the three-state distinction the instrument exists
+for — an approved row present, an approved row *absent while the section
+shows a diff* (the alarm), and the log being unreadable (never the alarm) —
+plus the render boundary: ``reason`` is operator free text and ``source_ids``
+is an unbounded lineage list, and neither may ever reach the prompt.
+
+Fault column (ADR-0077): a missing or unreadable audit log renders an
+explicit ``unavailable (reason=…)`` line. An unavailable instrument that
+rendered as "no approval record" would manufacture the exact false alarm
+this finding set out to make impossible.
+"""
+
+from __future__ import annotations
+
+import json
+import subprocess
+import sys
+from datetime import datetime, timezone
+from pathlib import Path
+
+sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "scripts"))
+import value_layer_approval_join as vlaj  # noqa: E402  # pyright: ignore[reportMissingImports]
+
+SCRIPT = Path(__file__).resolve().parent.parent / "scripts" / "value_layer_approval_join.py"
+
+HOME = "/Users/someone/.config/moltbook"
+START = "2026-08-01T23:00:00+09:00"
+END = "2026-08-08T23:00:00+09:00"
+
+
+def _ts(raw: str) -> datetime:
+    parsed = vlaj._parse_ts(raw)
+    assert parsed is not None
+    return parsed
+
+
+# ``path`` is deliberately ``object``: one test feeds a non-string to pin that a
+# malformed record is skipped rather than crashing the whole reading.
+def _record(
+    path: object, ts: str, *, command: str = "insight", decision: str = "approved", **extra
+):
+    return {
+        "ts": ts,
+        "command": command,
+        "path": path,
+        "decision": decision,
+        "source": "direct",
+        "content_hash": "abc123def4567890",
+        **extra,
+    }
+
+
+def _reading(records, *, section="skills", changed=True, top=vlaj._DEFAULT_TOP, unparsable=0):
+    return vlaj.build_reading(
+        records,
+        section=section,
+        changed=changed,
+        start=_ts(START),
+        end=_ts(END),
+        unparsable=unparsable,
+        top=top,
+    )
+
+
+class TestSectionMatching:
+    def test_each_section_claims_only_its_own_paths(self):
+        records = [
+            _record(f"{HOME}/identity.md", "2026-08-03T01:00:00+00:00"),
+            _record(f"{HOME}/constitution/contemplative-axioms.md", "2026-08-03T02:00:00+00:00"),
+            _record(f"{HOME}/skills/a-skill.md", "2026-08-03T03:00:00+00:00"),
+            _record(f"{HOME}/rules/a-rule.md", "2026-08-03T04:00:00+00:00"),
+            _record(f"{HOME}/knowledge.json", "2026-08-03T05:00:00+00:00"),
+        ]
+        for section in ("identity", "constitution", "skills", "rules"):
+            reading = _reading(records, section=section)
+            assert len(reading.rows) == 1, section
+            assert reading.approved == 1, section
+
+    def test_knowledge_json_belongs_to_no_value_layer_section(self):
+        records = [_record(f"{HOME}/knowledge.json", "2026-08-03T05:00:00+00:00")]
+        for section in ("identity", "constitution", "skills", "rules"):
+            assert _reading(records, section=section).rows == ()
+
+    def test_a_file_named_like_a_section_is_not_that_section(self):
+        """Only a *parent* component names a directory section.
+
+        Matching the whole path would let ``.../notes/skills`` (a file) count
+        as a skills approval and quietly satisfy the alarm.
+        """
+        records = [_record(f"{HOME}/notes/skills", "2026-08-03T03:00:00+00:00")]
+        assert _reading(records, section="skills").rows == ()
+
+    def test_a_relocated_home_still_matches(self):
+        """Records carry whatever MOLTBOOK_HOME was live at write time."""
+        records = [
+            _record("/mnt/backup/moltbook-2025/skills/a-skill.md", "2026-08-03T03:00:00+00:00")
+        ]
+        assert len(_reading(records, section="skills").rows) == 1
+
+    def test_a_non_string_path_is_skipped_not_crashed(self):
+        assert _reading([_record(None, "2026-08-03T03:00:00+00:00")]).rows == ()
+
+
+class TestWindowing:
+    def test_window_is_half_open_on_the_start_commit(self):
+        """A record stamped at the start commit is already inside its tree.
+
+        Counting it would credit the *previous* week's approval to this
+        week's diff.
+        """
+        at_start = _reading([_record(f"{HOME}/skills/s.md", START)])
+        assert at_start.rows == ()
+
+        just_after = _reading([_record(f"{HOME}/skills/s.md", "2026-08-01T23:00:01+09:00")])
+        assert len(just_after.rows) == 1
+
+    def test_window_is_closed_on_the_end_commit(self):
+        assert len(_reading([_record(f"{HOME}/skills/s.md", END)]).rows) == 1
+        after = _reading([_record(f"{HOME}/skills/s.md", "2026-08-08T23:00:01+09:00")])
+        assert after.rows == ()
+
+    def test_mixed_offsets_compare_on_the_instant_not_the_string(self):
+        """``2026-08-02T00:30:00+09:00`` is inside a window whose start is
+        ``2026-08-01T23:00:00+09:00`` — a lexical compare would agree here,
+        but the same instant written as UTC (``2026-08-01T15:30:00Z``) sorts
+        before the start string while being after it in time."""
+        reading = _reading([_record(f"{HOME}/skills/s.md", "2026-08-01T15:30:00Z")])
+        assert len(reading.rows) == 1
+
+    def test_pre_2026_04_timestamp_key_is_recognized(self):
+        """The audit log carries schema drift the due-check already handles;
+        the two readings must not disagree about which rows exist."""
+        legacy = {
+            "timestamp": "2026-08-03T01:00:00+00:00",
+            "command": "distill-identity-ca",
+            "path": f"{HOME}/identity.md",
+            "decision": "approved",
+            "content_hash": "0123456789abcdef",
+        }
+        reading = _reading([legacy], section="identity")
+        assert len(reading.rows) == 1
+        assert reading.rows[0].source == "—", "a missing source must render, not vanish"
+
+
+class TestAlarmCondition:
+    def test_changed_with_no_approved_row_is_the_alarm(self):
+        rendered = vlaj.format_reading(_reading([], changed=True))
+        assert "NO APPROVED RECORD" in rendered
+
+    def test_a_staged_row_alone_does_not_satisfy_the_gate(self):
+        """``staged`` is a deferred decision, not an approval (ADR-0074)."""
+        records = [_record(f"{HOME}/skills/s.md", "2026-08-03T03:00:00+00:00", decision="staged")]
+        reading = _reading(records, changed=True)
+        assert reading.staged == 1
+        assert reading.approved == 0
+        assert "NO APPROVED RECORD" in vlaj.format_reading(reading)
+
+    def test_a_rejected_row_alone_does_not_satisfy_the_gate(self):
+        records = [_record(f"{HOME}/skills/s.md", "2026-08-03T03:00:00+00:00", decision="rejected")]
+        reading = _reading(records, changed=True)
+        assert reading.rejected == 1
+        assert "NO APPROVED RECORD" in vlaj.format_reading(reading)
+
+    def test_an_approved_row_clears_the_alarm_and_is_citable(self):
+        records = [_record(f"{HOME}/skills/s.md", "2026-08-03T03:00:00+00:00")]
+        rendered = vlaj.format_reading(_reading(records, changed=True))
+        assert "NO APPROVED RECORD" not in rendered
+        assert "abc123def4567890" in rendered
+        assert "2026-08-03T03:00:00+00:00" in rendered
+
+    def test_no_diff_and_no_rows_is_not_an_alarm(self):
+        rendered = vlaj.format_reading(_reading([], changed=False))
+        assert "NO APPROVED RECORD" not in rendered
+        assert "nothing to reconcile" in rendered
+
+    def test_rows_without_a_diff_are_still_rendered(self):
+        """Approved but absent from the diff is its own signal (sync lag)."""
+        records = [_record(f"{HOME}/skills/s.md", "2026-08-03T03:00:00+00:00")]
+        rendered = vlaj.format_reading(_reading(records, changed=False))
+        assert "abc123def4567890" in rendered
+        assert "NO APPROVED RECORD" not in rendered
+
+
+class TestRenderBoundary:
+    def test_reason_and_source_ids_never_reach_the_render(self):
+        records = [
+            _record(
+                f"{HOME}/skills/s.md",
+                "2026-08-03T03:00:00+00:00",
+                reason="FREE-TEXT-MARKER typed by the operator",
+                source_ids=["LINEAGE-MARKER-1", "LINEAGE-MARKER-2"],
+                epistemic_counts={"generated": 3},
+            )
+        ]
+        rendered = vlaj.format_reading(_reading(records))
+        assert "FREE-TEXT-MARKER" not in rendered
+        assert "LINEAGE-MARKER-1" not in rendered
+
+    def test_target_paths_never_reach_the_render(self):
+        """Skill filenames are slugified from distilled pattern text."""
+        records = [
+            _record(f"{HOME}/skills/SLUG-MARKER-from-a-post.md", "2026-08-03T03:00:00+00:00")
+        ]
+        rendered = vlaj.format_reading(_reading(records))
+        assert "SLUG-MARKER" not in rendered
+
+    def test_a_newline_in_a_field_cannot_break_out_of_its_cell(self):
+        records = [
+            _record(
+                f"{HOME}/skills/s.md",
+                "2026-08-03T03:00:00+00:00",
+                command="insight\n## Forged Heading",
+            )
+        ]
+        rendered = vlaj.format_reading(_reading(records))
+        assert "\n## Forged Heading" not in rendered
+        assert "## Forged Heading" in rendered.replace("\\|", "|")
+
+    def test_a_pipe_in_a_field_cannot_forge_a_table_column(self):
+        records = [_record(f"{HOME}/skills/s.md", "2026-08-03T03:00:00+00:00", command="a|b")]
+        row_line = [
+            ln for ln in vlaj.format_reading(_reading(records)).splitlines() if "a\\|b" in ln
+        ]
+        assert row_line, "pipe was not escaped"
+        assert row_line[0].count("|") - row_line[0].count("\\|") == 6
+
+    def test_an_overlong_field_is_capped(self):
+        records = [_record(f"{HOME}/skills/s.md", "2026-08-03T03:00:00+00:00", command="x" * 500)]
+        rendered = vlaj.format_reading(_reading(records))
+        assert "x" * 500 not in rendered
+        assert "…" in rendered
+
+
+class TestCapsAndCountsAreDeclared:
+    def test_truncation_is_stated_never_silent(self):
+        records = [
+            _record(f"{HOME}/skills/s{i}.md", f"2026-08-03T03:00:{i:02d}+00:00") for i in range(5)
+        ]
+        reading = _reading(records, top=2)
+        assert len(reading.rows) == 2
+        assert reading.approved == 5, "the tally counts every match, not just the shown ones"
+        assert "3 further record(s) not shown" in vlaj.format_reading(reading)
+
+    def test_the_cap_reserves_the_approved_rows(self):
+        """The rows that answer the question survive truncation.
+
+        Observed on the live log: a skills week is ~110 rows dominated by
+        same-second ``staged`` batches, so a head-of-list slice showed 25
+        staged rows and hid all 8 approved ones — while the prompt asks B to
+        cite an approving row's ``ts`` and ``content_hash``.
+        """
+        records = [
+            _record(f"{HOME}/skills/s{i}.md", f"2026-08-03T03:00:{i:02d}+00:00", decision="staged")
+            for i in range(30)
+        ]
+        records.append(
+            _record(
+                f"{HOME}/skills/approved.md",
+                "2026-08-03T03:59:00+00:00",
+                content_hash="APPROVEDHASH0001",
+            )
+        )
+        reading = _reading(records, top=5, changed=True)
+        rendered = vlaj.format_reading(reading)
+        assert "APPROVEDHASH0001" in rendered
+        assert len(reading.rows) == 5
+        assert "26 further record(s) not shown" in rendered
+        # The shown set still reads as a timeline, not approved-first.
+        assert [row.ts for row in reading.rows] == sorted(row.ts for row in reading.rows)
+
+    def test_unparsable_lines_are_counted_in_the_render(self):
+        reading = _reading([], changed=False, unparsable=2)
+        assert "2 audit line(s) unparsable" in vlaj.format_reading(reading)
+
+    def test_same_second_rows_sort_without_raising(self):
+        records = [
+            _record(f"{HOME}/skills/b.md", "2026-08-03T03:00:00+00:00", command="zzz"),
+            _record(f"{HOME}/skills/a.md", "2026-08-03T03:00:00+00:00", command="aaa"),
+        ]
+        reading = _reading(records)
+        assert [row.command for row in reading.rows] == ["aaa", "zzz"]
+
+
+class TestUnavailableIsNotTheAlarm:
+    def test_missing_audit_log_renders_a_reason_code(self, tmp_path):
+        result = _run_cli(tmp_path / "absent.jsonl", "--diff", "changed")
+        assert result.returncode == 0, result.stderr
+        assert "unavailable (reason=audit-log-missing)" in result.stdout
+        assert "NO APPROVED RECORD" not in result.stdout
+
+    def test_unreadable_audit_log_renders_a_reason_code(self, tmp_path):
+        audit = tmp_path / "audit.jsonl"
+        audit.write_text("{}\n", encoding="utf-8")
+        audit.chmod(0o000)
+        try:
+            result = _run_cli(audit, "--diff", "changed")
+        finally:
+            audit.chmod(0o600)
+        assert result.returncode == 0, result.stderr
+        assert "unavailable (reason=" in result.stdout
+        assert "NO APPROVED RECORD" not in result.stdout
+
+    def test_unparsable_window_renders_a_reason_code(self, tmp_path):
+        audit = tmp_path / "audit.jsonl"
+        audit.write_text("", encoding="utf-8")
+        result = _run_cli(audit, "--diff", "changed", start="unknown")
+        assert result.returncode == 0, result.stderr
+        assert "unavailable (reason=window-unparsable)" in result.stdout
+        assert "NO APPROVED RECORD" not in result.stdout
+
+    def test_load_records_raises_rather_than_returning_empty(self, tmp_path):
+        try:
+            vlaj.load_records(tmp_path / "absent.jsonl")
+        except vlaj.JoinUnavailable as exc:
+            assert exc.reason == "audit-log-missing"
+        else:  # pragma: no cover - the point of the test
+            raise AssertionError("a missing log must not read as zero records")
+
+    def test_malformed_lines_degrade_to_a_count(self, tmp_path):
+        audit = tmp_path / "audit.jsonl"
+        audit.write_text(
+            "\n".join(
+                [
+                    json.dumps(_record(f"{HOME}/skills/s.md", "2026-08-03T03:00:00+00:00")),
+                    "{not json",
+                    "[1, 2, 3]",
+                    "",
+                ]
+            ),
+            encoding="utf-8",
+        )
+        records, unparsable = vlaj.load_records(audit)
+        assert len(records) == 1
+        assert unparsable == 2
+
+
+def _run_cli(audit: Path, *extra: str, start: str = START) -> subprocess.CompletedProcess[str]:
+    return subprocess.run(
+        [
+            sys.executable,
+            str(SCRIPT),
+            "--audit",
+            str(audit),
+            "--section",
+            "skills",
+            "--start",
+            start,
+            "--end",
+            END,
+            *extra,
+        ],
+        capture_output=True,
+        text=True,
+        timeout=60,
+    )
+
+
+class TestCli:
+    def test_end_to_end_render_over_a_real_file(self, tmp_path):
+        audit = tmp_path / "audit.jsonl"
+        audit.write_text(
+            json.dumps(
+                _record(
+                    f"{HOME}/skills/s.md",
+                    "2026-08-03T03:00:00+00:00",
+                    reason="FREE-TEXT-MARKER",
+                )
+            )
+            + "\n",
+            encoding="utf-8",
+        )
+        result = _run_cli(audit, "--diff", "changed")
+        assert result.returncode == 0, result.stderr
+        assert "**Approval provenance**" in result.stdout
+        assert "abc123def4567890" in result.stdout
+        assert "FREE-TEXT-MARKER" not in result.stdout
+
+    def test_timezone_naive_stamps_are_read_as_utc(self):
+        parsed = vlaj._parse_ts("2026-08-03T03:00:00")
+        assert parsed == datetime(2026, 8, 3, 3, 0, tzinfo=timezone.utc)
diff --git a/tests/test_weekly_analysis_shell.py b/tests/test_weekly_analysis_shell.py
index 7550bff..26737f3 100644
--- a/tests/test_weekly_analysis_shell.py
+++ b/tests/test_weekly_analysis_shell.py
@@ -336,6 +336,134 @@ class TestPromptAssembly:
         # The label that makes the count usable: which store, measured when.
         assert "live store at report-generation time" in prompt
 
+    def test_state_diff_sections_carry_their_approval_provenance(self, tmp_path):
+        """findings F1.1: the state diff showed value-layer changes with no
+        approval column, so the report's loudest claim ("whether it passed the
+        approval path is not visible in the data supplied here") was bounded by
+        a data gap that ``logs/audit.jsonl`` had already closed.
+
+        Both halves are pinned: a section with an approved row carries a citable
+        hash, and a section that changed with *no* approved row says so — that
+        absence is the alarm the report could not previously distinguish from
+        the presence. The record's free text (``reason``) and lineage list
+        (``source_ids``) must not ride along into the prompt.
+        """
+        home = _make_home(tmp_path)
+        data_repo = tmp_path / "fakehome" / "MyAI_Lab" / "contemplative-agent-data"
+        (data_repo / "skills").mkdir(parents=True)
+
+        def git(*a: str, when: str | None = None) -> None:
+            env = {
+                **os.environ,
+                "GIT_CONFIG_GLOBAL": "/dev/null",
+                "GIT_CONFIG_SYSTEM": "/dev/null",
+            }
+            if when:
+                env["GIT_AUTHOR_DATE"] = when
+                env["GIT_COMMITTER_DATE"] = when
+            subprocess.run(["git", *a], cwd=data_repo, check=True, capture_output=True, env=env)
+
+        git("init", "-q")
+        git("config", "user.email", "t@example.com")
+        git("config", "user.name", "t")
+        (data_repo / "identity.md").write_text("v1\n", encoding="utf-8")
+        (data_repo / "skills" / "kept.md").write_text("kept\n", encoding="utf-8")
+        git("add", "-A")
+        git("commit", "-qm", "start", when="2026-07-18T00:00:00+0900")
+        # identity.md changed *with* an approval row; skills/ changed without.
+        (data_repo / "identity.md").write_text("v2\n", encoding="utf-8")
+        (data_repo / "skills" / "unapproved.md").write_text("new\n", encoding="utf-8")
+        git("add", "-A")
+        git("commit", "-qm", "end", when=f"{END_DATE}T00:00:00+0900")
+
+        (home / "logs" / "audit.jsonl").write_text(
+            json.dumps(
+                {
+                    "ts": "2026-07-20T02:00:00+00:00",
+                    "command": "distill-identity",
+                    "path": f"{home}/identity.md",
+                    "decision": "approved",
+                    "source": "stage-adopted",
+                    "content_hash": "IDHASH0000000000",
+                    "reason": "FREE-TEXT-MARKER typed by the operator",
+                    "source_ids": ["LINEAGE-MARKER-1"],
+                }
+            )
+            + "\n",
+            encoding="utf-8",
+        )
+
+        captured = tmp_path / "prompt.txt"
+        bin_dir = tmp_path / "bin"
+        bin_dir.mkdir(exist_ok=True)
+        stub = bin_dir / "claude"
+        stub.write_text(
+            f'#!/bin/bash\ncat >> "{captured}"\nprintf "# Weekly\\n"\n', encoding="utf-8"
+        )
+        stub.chmod(0o755)
+
+        result = _run(home, bin_dir, tmp_path)
+
+        assert result.returncode == 0, result.stderr
+        prompt = captured.read_text(encoding="utf-8")
+        state_diff = prompt.split("## Log Anomaly Sweep")[0]
+        # One block per value-layer section: identity, constitution, skills, rules.
+        assert state_diff.count("**Approval provenance**") == 4
+        assert "IDHASH0000000000" in state_diff
+        assert "NO APPROVED RECORD" in state_diff
+        # The instrument read the log; nothing degraded to "cannot tell".
+        assert "unavailable (reason=" not in state_diff
+        # The record's free text and lineage list stay out of the prompt.
+        assert "FREE-TEXT-MARKER" not in prompt
+        assert "LINEAGE-MARKER-1" not in prompt
+
+    def test_a_missing_audit_log_never_reads_as_a_missing_approval(self, tmp_path):
+        """An unavailable instrument reads zero, not clean (ADR-0077). With no
+        audit log at all, every section must say so with a reason code — the
+        alarm string must never appear, or the weekly report would manufacture
+        a gate-bypass claim out of its own blindness."""
+        home = _make_home(tmp_path)
+        data_repo = tmp_path / "fakehome" / "MyAI_Lab" / "contemplative-agent-data"
+        data_repo.mkdir(parents=True)
+
+        def git(*a: str, when: str | None = None) -> None:
+            env = {
+                **os.environ,
+                "GIT_CONFIG_GLOBAL": "/dev/null",
+                "GIT_CONFIG_SYSTEM": "/dev/null",
+            }
+            if when:
+                env["GIT_AUTHOR_DATE"] = when
+                env["GIT_COMMITTER_DATE"] = when
+            subprocess.run(["git", *a], cwd=data_repo, check=True, capture_output=True, env=env)
+
+        git("init", "-q")
+        git("config", "user.email", "t@example.com")
+        git("config", "user.name", "t")
+        (data_repo / "identity.md").write_text("v1\n", encoding="utf-8")
+        git("add", "-A")
+        git("commit", "-qm", "start", when="2026-07-18T00:00:00+0900")
+        (data_repo / "identity.md").write_text("v2\n", encoding="utf-8")
+        git("commit", "-qam", "end", when=f"{END_DATE}T00:00:00+0900")
+
+        assert not (home / "logs" / "audit.jsonl").exists()
+
+        captured = tmp_path / "prompt.txt"
+        bin_dir = tmp_path / "bin"
+        bin_dir.mkdir(exist_ok=True)
+        stub = bin_dir / "claude"
+        stub.write_text(
+            f'#!/bin/bash\ncat >> "{captured}"\nprintf "# Weekly\\n"\n', encoding="utf-8"
+        )
+        stub.chmod(0o755)
+
+        result = _run(home, bin_dir, tmp_path)
+
+        assert result.returncode == 0, result.stderr
+        state_diff = captured.read_text(encoding="utf-8").split("## Log Anomaly Sweep")[0]
+        assert "unavailable (reason=audit-log-missing)" in state_diff
+        assert "NO APPROVED RECORD" not in state_diff
+
 
 class TestPreflight:
     def test_missing_claude_fails_before_the_collection_pass(self, tmp_path):
````

### F1.2.patch

**SCOPE_ESCALATED** — code scope として起票されたが、`^(src|scripts|tests)/` の外に触れたため全文ゲートへ昇格した。scope 分類の誤りではなく、要約行での通過を防ぐ設計上の昇格。

触れたパス: `docs/CODEMAPS/architecture.md`

```diff
diff --git a/docs/CODEMAPS/architecture.md b/docs/CODEMAPS/architecture.md
index cc19745..5f8aeba 100644
--- a/docs/CODEMAPS/architecture.md
+++ b/docs/CODEMAPS/architecture.md
@@ -537,6 +537,8 @@ The drift scan (2026-08-06) diffs the per-endpoint response-key vocabulary the c
 
 The sweep's signature is keyed on **level + message**, with the dotted `%(name)s` module path dropped for lines in the runtime's own log format, and hex-shaped ids squashed to `#` alongside digit runs. The 80-character cap is the reason: the module path alone runs to ~47 characters, so keying on it spent the budget on the address and truncated the predicate — `"reply on <id> created but verification failed"` rendered as `"reply on <id> created"`, a failure displayed as its own opposite (findings F1.1). Excluding the name also makes the instrument refactor-invariant: a pure module move (`7c96e0f`) used to reset every affected signature to 🆕, i.e. the Δ / 🆕 columns measured the codebase rather than the runtime (findings F1.2). The trade is that the same message from two subsystems now merges into one row, so the logger name is carried as a display-only `Origin` column — it never enters the signature, the state file or the novelty computation, so the reader keeps the distinction the key deliberately drops.
 
+Four message families end in **generated text** rather than a predicate — the three publish previews (`>> Reply to` / `>> Comment on` / `>> New post`, emitted as bounded single lines by `log_published`) and the distill `Added pattern (source=…)` line. For these the signature is cut at the payload boundary, so the key is the static head plus the counterparty address and nothing of the body (findings F1.2, 2026-08-15). Without the cut each published body and each distilled pattern minted its own one-off 🆕 row — the census counted bodies rather than events — and body-derived text, downstream of untrusted feed content, reached the state file and the prompt `weekly-analysis.sh` feeds to an LLM, which is the side channel ADR-0083 closed for episode logs. The producers are untouched: the bounded preview is the T-LOG-DEBUG-CONTENT repair and the operator's live tail still shows it; only the instrument's key is content-free. `>> New post` cuts before its char count rather than after, because that format puts the generated title ahead of the count. The cut is an allowlist of formats this repo emits, not a free-text filter — an unrecognised line keeps its whole predicate.
+
 The sweep has **no time window**: it counts every line each allowed file currently holds, so a row's `Count` spans that file's lifetime — and the files rotate on different schedules (`ollama-serve.log` nightly since 2026-08-01, `agent-launchd.log` weekly via `backup-runtime.sh`, the one-shot `insight-` / `distill-launchd.log` never), which makes two rows of one table not necessarily commensurable. Rotation also moves the novelty baseline: lines leave the `*.log` glob, counts fall, and known signatures re-appear as 🆕 — once a rare footnote, the steady state since nightly rotation shipped. Filtering by timestamp would discard signal, so the instrument states its basis instead (findings F1.1, 2026-08-07): a per-file **corpus census** (name, lines read, signal lines) is written to a sidecar `<state>.corpus.tsv` — a sidecar because `read_state` silently drops any state-file line whose first field is not an int, so a header row there would vanish on read — and rendered above the table beside the previous sweep's three figures, with an explicit "🆕 and Δ are not comparable to last week's" sentence when the corpus lost more than 10% of its lines. The census is written *before* its snapshot (the snapshot's existence is the shell's "sweep completed" signal) and promoted in lockstep with it; if the pair breaks, the shell deletes the old census so the next run reports "no previous census" rather than asserting a comparison against a corpus that no longer exists.
 
 Injection boundary: the sweep, the invariant check and the drift scan must never read episode logs (the drift scan reads only the self-written `api-audit.jsonl`; the platform-controlled key names it renders are Markdown-escaped and length-capped). The duplicate scan does, and is the only intake permitted to — it emits **only** 12-hex SHA-256 digests, counts, filename-derived dates and the fixed `{post, reply, comment}` vocabulary (ADR-0083, gated by `TestOutputBoundary`). The skill-selection reading (2026-08-08, findings F1.4) reads the self-written `skill-selection-*.jsonl` shadow log (ADR-0076): the *selected* middle link between *installed* (state diff) and *vocabulary in output* (section E) was already logged per publish action but never supplied to the report. Its records embed the selection situation — untrusted post bodies — so the renderer (`format_skill_selection_report`, the same one behind `report --skill-selection`) emits **catalog** names and counts only, never the situation strings (same ADR-0083 boundary; gated by `test_skill_selection_reading_reaches_the_prompt_names_only`). "Catalog names" is load-bearing since 2026-08-08, when the reading gained a per-name rejected-name tally: a *rejected* name is by definition a string that matched nothing in the catalog, i.e. free model output from a prompt that embeds untrusted post bodies — the 2026-08-08 backfill reading measured 12% of them as fragments bled in from elsewhere in the prompt. It is therefore the one string in this reading that is not drawn from a closed, self-written vocabulary, and `format_skill_selection_report` withholds it unless a caller passes `include_rejected_names=True`. **The default is the restrictive one**, so a new caller is safe by omission and the weekly script needs no knowledge of which side of the boundary it is on; only `report --skill-selection` (terminal, human reader — the reader the tally exists for) opts in. The weekly prompt still receives the tally's *shape* — distinct-name count, emissions, and each entry's nearest catalog name with its surface-similarity distance — because those are catalog-derived or numeric. Same choice ADR-0083's duplicate scan made: send the shape, not the content. All five are observability: a failure degrades to a "not available" stub and never breaks the report.
diff --git a/scripts/log_anomaly_sweep.py b/scripts/log_anomaly_sweep.py
index 615b263..3109e11 100755
--- a/scripts/log_anomaly_sweep.py
+++ b/scripts/log_anomaly_sweep.py
@@ -15,6 +15,10 @@ Security (load-bearing):
   prompt-injection vector, and this output may be fed to an LLM.
 - Output is normalized signatures (timestamps stripped, digits squashed,
   truncated), not verbatim log bodies, to shrink the injection surface further.
+  Lines whose format ends in generated text — the bounded publish previews and
+  the distill pattern line — are additionally cut at their payload boundary
+  (``_PAYLOAD_CUT_RES``), so that text enters neither the key, the state file
+  nor the prompt.
 
 State: a TSV ``count<TAB>signature`` snapshot of the previous sweep, used to
 compute the NEW flag and the per-signature delta. Committing it is an
@@ -111,6 +115,53 @@ _HEXID_RE = re.compile(r"\b(?=[0-9a-f]*\d)[0-9a-f]{6,}(?:-[0-9a-f]{4,})*\b")
 
 _SIG_MAXLEN = 80
 
+# Message families whose tail is generated text: the three publish previews
+# (``log_published`` in ``adapters/moltbook/publish.py``, already bounded to a
+# single line by ``log_preview``) and the distill pattern line
+# (``core/distill.py``, ``pattern[:80]``). Leaving that tail in the key makes
+# every distinct body its own signature, so the census counts bodies rather
+# than events and each publish mints a one-off 🆕 row; it also carries
+# body-derived text — downstream of untrusted feed content — into the state
+# file and into the prompt ``weekly-analysis.sh`` feeds to an LLM, the same
+# side channel ADR-0083 closed for episode logs. The producers are correct as
+# they stand (the bounded preview is the T-LOG-DEBUG-CONTENT repair, and the
+# operator's live tail wants it); it is this instrument's *key* that must be
+# content-free.
+#
+# Each pattern matches the static head of its family up to and including the
+# boundary its payload starts after; the signature becomes that match, so a
+# family aggregates into one row. Matched against the already-normalized line
+# (lowercased, digit runs and hex ids squashed to ``#``) — hence the lowercase
+# level prefix and the ``#`` char count. Counterparty names are left in, in
+# keeping with the rule above that agent-name variation is not squashed: the
+# name is a fixed-shape address, not a body.
+#
+# ``>> new post`` cuts earlier than its two siblings by necessity: its format
+# is ``>> New post [%s] (id=%s): %d chars: %s``, which puts the generated
+# *title* ahead of the count, so cutting at ``chars:`` would keep one-off
+# generated text in the key — exactly what the cut exists to prevent.
+_LEVEL_PREFIX = r"(?:\[\w+\] )?"
+_PAYLOAD_CUT_RES: tuple[re.Pattern[str], ...] = (
+    re.compile(_LEVEL_PREFIX + r">> reply to .*? chars:"),
+    re.compile(_LEVEL_PREFIX + r">> comment on .*? chars:"),
+    re.compile(_LEVEL_PREFIX + r">> new post"),
+    re.compile(_LEVEL_PREFIX + r"added pattern \(source=[^)]*\):"),
+)
+
+
+def _cut_at_payload_boundary(s: str) -> str:
+    """Drop the generated tail of a known preview-bearing line, if any.
+
+    Returns *s* unchanged when no family matches: the cut is an allowlist of
+    formats this repo emits, not a general free-text filter, so an unknown
+    line keeps the full predicate the sweep exists to surface.
+    """
+    for pattern in _PAYLOAD_CUT_RES:
+        m = pattern.match(s)
+        if m is not None:
+            return m.group(0)
+    return s
+
 
 @dataclass(frozen=True)
 class Finding:
@@ -182,6 +233,10 @@ def normalize_with_origin(line: str) -> tuple[str, str]:
     per-item variation groups), and truncates. Agent-name variation *inside
     the message* is intentionally not squashed; minor over-splitting is safer
     than over-merging distinct anomalies.
+
+    For the known preview-bearing families (``_PAYLOAD_CUT_RES``) the line is
+    additionally cut at its payload boundary, so no generated body or pattern
+    text reaches the signature, the state file or the weekly prompt.
     """
     s = _TS_ISO_RE.sub("", line)
     s = _TS_CLOCK_RE.sub("", s)
@@ -195,6 +250,7 @@ def normalize_with_origin(line: str) -> tuple[str, str]:
     s = _HEXID_RE.sub("#", s)
     s = _DIGITS_RE.sub("#", s)
     s = _WS_RE.sub(" ", s).strip()
+    s = _cut_at_payload_boundary(s)
     return s[:_SIG_MAXLEN], origin
 
 
@@ -441,7 +497,8 @@ def render_markdown(
     lines.append("")
     lines.append(
         "_Signatures are normalized (timestamps and module paths stripped, "
-        "digits and ids squashed). Origin is display-only — it does not enter "
+        "digits and ids squashed, generated bodies and pattern text cut at "
+        "their payload boundary). Origin is display-only — it does not enter "
         "the signature, so a module rename cannot reset the Δ / 🆕 baseline, "
         "and one row may list several subsystems emitting the same message. "
         "Source: self-written logs only; episode logs are never read._"
diff --git a/tests/test_log_anomaly_sweep.py b/tests/test_log_anomaly_sweep.py
index 1db1137..33dee00 100644
--- a/tests/test_log_anomaly_sweep.py
+++ b/tests/test_log_anomaly_sweep.py
@@ -156,6 +156,126 @@ class TestOriginIsCarriedButNotKeyed:
         assert re.fullmatch(r"[A-Za-z_]\w*(?:\.\w+)+", ok)
 
 
+class TestGeneratedTextDoesNotEnterTheKey:
+    """Preview-bearing families are cut at their payload boundary.
+
+    The producers are correct: ``log_published`` emits a bounded single-line
+    preview on purpose (T-LOG-DEBUG-CONTENT). The residue was on this side —
+    free text survived into the signature, so every published body and every
+    distilled pattern minted its own 🆕 row and body-derived text (downstream
+    of untrusted feed content) reached the state file and the weekly LLM
+    prompt, the side channel ADR-0083 closed for episode logs (findings F1.2
+    2026-08-15).
+
+    Bodies here mention ``backoff`` because that is how an INFO preview
+    reaches the sweep at all: ``_is_signal`` admits it through
+    ``_CRITICAL_RE``, not through its level.
+    """
+
+    # Post ids are written as ``post_id[:12]`` by both publish paths, so the
+    # fixtures carry that truncated shape rather than a whole uuid fragment.
+    # The A/B pairs share an id: body variation is what this class is about.
+    REPLY_A = (
+        "09:12:33 [INFO] contemplative_agent.adapters.moltbook.reply_handler: "
+        ">> Reply to budget_skynet on 836e1237-a5b: 61 chars: "
+        "the observation regarding silence held through the backoff"
+    )
+    REPLY_B = (
+        "10:41:02 [INFO] contemplative_agent.adapters.moltbook.reply_handler: "
+        ">> Reply to budget_skynet on 836e1237-a5b: 48 chars: "
+        "a wholly different sentence about backoff and rest"
+    )
+    COMMENT_A = (
+        "09:12:33 [INFO] contemplative_agent.adapters.moltbook.feed_manager: "
+        ">> Comment on 836e1237-a5b: 33 chars: one body mentioning backoff"
+    )
+    COMMENT_B = (
+        "09:44:10 [INFO] contemplative_agent.adapters.moltbook.feed_manager: "
+        ">> Comment on 836e1237-a5b: 39 chars: another body mentioning backoff"
+    )
+    POST_A = (
+        "09:12:33 [INFO] contemplative_agent.adapters.moltbook.post_pipeline: "
+        ">> New post [On Waiting Without Backoff] (id=836e1237-a5b2): 900 chars: "
+        "the opening line of the post"
+    )
+    POST_B = (
+        "11:03:20 [INFO] contemplative_agent.adapters.moltbook.post_pipeline: "
+        ">> New post [A Note On Backoff And Silence] (id=91ab77de-0c31): 750 chars: "
+        "a different opening line"
+    )
+    PATTERN_A = (
+        "09:12:33 [INFO] contemplative_agent.core.distill: "
+        "Added pattern (source=self_reflection): "
+        "I tend to retry past the point where backoff is the honest answer"
+    )
+    PATTERN_B = (
+        "09:12:34 [INFO] contemplative_agent.core.distill: "
+        "Added pattern (source=self_reflection): "
+        "Silence reads as backoff to a counterparty who cannot see the queue"
+    )
+
+    def test_two_reply_bodies_reach_one_signature(self):
+        assert las.normalize(self.REPLY_A) == las.normalize(self.REPLY_B)
+
+    def test_reply_body_text_does_not_survive_into_the_signature(self):
+        sig = las.normalize(self.REPLY_A)
+        assert "observation" not in sig
+        assert "silence" not in sig
+        assert sig.endswith("chars:")
+
+    def test_the_static_predicate_and_counterparty_survive_the_cut(self):
+        """The cut removes the body, not the event. A row must still say what
+        happened and to whom, or the census stops being readable."""
+        sig = las.normalize(self.REPLY_A)
+        assert sig.startswith("[info] >> reply to budget_skynet on ")
+        assert "chars:" in sig
+
+    def test_comment_previews_aggregate_into_one_row(self):
+        findings = las.analyze([self.COMMENT_A, self.COMMENT_B], {})
+        assert len(findings) == 1
+        assert findings[0].count == 2
+        assert "mentioning" not in findings[0].signature
+
+    def test_generated_post_titles_do_not_survive(self):
+        """``>> New post`` puts the generated title *ahead* of the char count,
+        so cutting at ``chars:`` would have left one-off text in the key."""
+        assert las.normalize(self.POST_A) == las.normalize(self.POST_B)
+        sig = las.normalize(self.POST_A)
+        assert "waiting" not in sig
+        assert "silence" not in sig
+
+    def test_distilled_patterns_aggregate_and_keep_their_source(self):
+        findings = las.analyze([self.PATTERN_A, self.PATTERN_B], {})
+        assert len(findings) == 1
+        sig = findings[0].signature
+        assert findings[0].count == 2
+        assert sig.endswith("(source=self_reflection):")
+        assert "retry" not in sig and "counterparty" not in sig
+
+    def test_a_distinct_source_still_splits_the_pattern_rows(self):
+        other = self.PATTERN_A.replace("source=self_reflection", "source=activity")
+        assert las.normalize(self.PATTERN_A) != las.normalize(other)
+
+    def test_body_text_reaches_neither_the_state_file_nor_the_render(self, tmp_path):
+        """The two carriers the finding names: the snapshot that persists week
+        to week, and the table ``weekly-analysis.sh`` feeds to an LLM."""
+        findings = las.analyze([self.REPLY_A, self.POST_A, self.PATTERN_A], {})
+        state = tmp_path / "sweep.tsv"
+        las.write_state(state, findings)
+        written = state.read_text(encoding="utf-8")
+        rendered = las.render_markdown(findings, top=25, corpus=CORPUS)
+        for leaked in ("observation", "waiting", "opening line", "counterparty"):
+            assert leaked not in written
+            assert leaked not in rendered
+
+    def test_an_unrelated_line_mentioning_chars_is_not_cut(self):
+        """The cut is an allowlist of formats this repo emits, not a general
+        free-text filter: an unknown predicate must survive whole."""
+        line = "09:12:33 [WARNING] mod.name: request body 4000 chars: rejected upstream"
+        sig = las.normalize(line)
+        assert "rejected upstream" in sig
+
+
 class TestAnalyze:
     def test_non_signal_lines_ignored(self):
         lines = ["just a normal info line", "starting session", "all good"]
```

## 4. Insight staging review

```text
## Review method and headline finding

I read the actual staged files (`~/.config/moltbook/.staged/`, 49 candidates + `.meta.json` sidecars), the 45 adopted skills in `~/.config/moltbook/skills/`, and the synthesis trace in `snapshots/insight_20260814T230000402258Z/reasoning.md`. I resolved each candidate's `source_ids` through `pattern_id()` (sha256 of `distilled|pattern`, per `knowledge_store.py:60`) into `knowledge.json` to get distinct episode dates, and embedded all 94 skill bodies with `nomic-embed-text` to rank duplication.

Three facts shape every verdict below. **First**, the whole batch sits between 0.756 and 0.878 cosine against its nearest adopted skill — at or above the 0.80 NN ceiling this store has historically topped out at, meaning embedding distance cannot discriminate here and the batch is one theme ("stop reading surface content, pivot to underlying structure/boundary/mechanism") restated 49 ways. **Second**, that theme is already the store's dominant occupant: `detecting-abstract-to-operational-constraint-shift` is the top store neighbour for 11 separate candidates. **Third**, `epistemic_counts` is `generated: N, observed: 0, unknown: 0` for all 49 — nothing in this batch rests on observed interlocutor behaviour, only on the agent's own generated patterns. Adoption is therefore expensive (skills are all-injected) and mostly redundant, so the bar below is high: 5 adopt, 44 reject.

## 1. affective-to-technical-framing — RECOMMEND: reject
Rests on a single episode day (2026-08-09) — the only candidate besides #47 failing provenance breadth outright.
It also describes a persuasion tactic ("lowers the audience's guard", "maximize persuasive impact", manufacture an "inside scoop") rather than an analytic behaviour, which is a manipulation pattern rather than a skill.
Lowest intra-batch neighbour (0.731), so nothing shadows it — it is rejected on its own merits, not redundancy.

## 2. analyzing-conceptual-gaps-between-abstract-philoso — RECOMMEND: reject
Highest store duplication in the entire batch at 0.878 against adopted `detecting-abstract-to-operational-constraint-shift`, whose description ("structural tension when discourse abruptly shifts from high-level philosophy to specific mandatory system jargon") is the same claim.
Its 10 patterns / 5 episode days are the batch's second-broadest provenance, but breadth cannot rescue an already-adopted theme.

## 3. analyzing-conceptual-processuality — RECOMMEND: reject
Shadowed within the batch by #43 `shifting-focus-from-state-to-process-mechanics` (0.843 sibling to #13, same state→process move) which carries 9 patterns across 5 days versus this one's 4 across 3.
Adopted `analyzing-loci-of-identity-definition` already instructs analysis by "current commitments and directional choices" rather than fixed composition.

## 4. analyzing-structural-tension-moving-beyond-closure — RECOMMEND: reject
Forms the batch's tightest near-duplicate pair with #46 `structural-gap-analysis-by-sustaining-tension` at 0.902 — the highest intra-batch similarity measured; at most one of the pair could ever be adopted.
Adopted `identifying-fidelity-vs-utility-tension` and `identify-structural-pivot` already cover framing a systemic conflict without resolving it.

## 5. analyzing-systemic-vs-phenomenal-tensions — RECOMMEND: reject
0.841 against adopted `identifying-structural-tensions-via-system-metaphor`, which already names the technical-language-as-proxy-for-experience move.
Sibling #2 covers the same is/feels-like gap at higher provenance (5 days vs 3), so this is shadowed on both axes.

## 6. anchoring-abstract-integrity-through-manifested-co — RECOMMEND: reject
0.839 against adopted `mandating-structural-integrity-axioms` ("shift design discussions from proving functionality to structurally guaranteeing impossibility"), which is the same fail-closed argument.
Adopted `boundary-assumption-verification` covers the "document the assumption at the boundary" half.

## 7. anchoring-meta-critiques-to-structural-analogies — RECOMMEND: reject
0.841 against `detecting-abstract-to-operational-constraint-shift` and 0.871 against batch sibling #2 — one of the batch's most redundant nodes on both axes.
Its actual instruction (deflect philosophical challenge by re-framing into computational mechanics) is also a conversational escape hatch rather than an analytic gain.

## 8. assessing-functional-necessity-of-concepts — RECOMMEND: reject
0.827 against adopted `evaluating-contextual-functional-dependence`, which already shifts evaluation "from surface textual resemblance to verifiable, persistent functional congruence."
Adopted `deconstructing-consensus-validity-through-shared-d` covers its "corroboration vs intrinsic weight" second half.

## 9. boundary-constraint-testing-bct — RECOMMEND: reject
0.855 against adopted `scope-boundary-mapping` ("analyzing perceived limitations to determine if they are definitional boundaries or actual constraints") — the same internal-vs-external limit test.
Adopted `mapping-epistemic-boundaries` already covers enumerating what has been ruled out.

## 10. boundary-structural-analysis — RECOMMEND: reject
0.846 against `detecting-abstract-to-operational-constraint-shift`; its proposed "Boundary Meta-Layer" is a renaming of that adopted skill's trigger, not a new behaviour.
Shadowed intra-batch by #2 (0.857), which states the same abstract/deterministic collision with broader provenance.

## 11. challenging-fixed-identity-anchors — RECOMMEND: reject
Adopted `analyzing-loci-of-identity-definition` already rejects identity-by-accumulated-history in favour of present directional choice, which is this candidate's whole content.
Shadowed intra-batch by #43 (0.781), the better-evidenced member of the state-versus-process group.

## 12. challenging-foundational-premises — RECOMMEND: reject
0.838 against adopted `deconstruct-foundational-claims-against-operative-`, which tests stable-identity assertions against operative components — the same reification check.
Also 0.851 against sibling #2; the batch contains at least four variants of "question the premise" and this is not the strongest.

## 13. conceptual-deconstruction-for-operational-mapping — RECOMMEND: reject
0.844 against `detecting-abstract-to-operational-constraint-shift`, the eleven-way-duplicated store anchor for this batch.
Its "When to Use" degenerates into four loosely related bullets, so it is not one enactable behaviour even setting duplication aside.

## 14. constraint-identification — RECOMMEND: reject
0.859 against `detecting-abstract-to-operational-constraint-shift` — third-highest store duplication in the batch.
Adopted `identifying-systemic-boundary-stressors` and `pinpointing-systemic-boundary-conditions` already occupy the "characterize the dominant constraint set" slot twice over.

## 15. controlled-opacity-management — RECOMMEND: reject
0.829 against adopted `pinpointing-systemic-boundary-conditions`; its distinctive claim (stability is maintained by throughput pacing and performance) is asserted but given no test the agent can run.
Reads as a suspicion to hold rather than a behaviour to enact, which is the vague-virtue failure mode.

## 16. decomposing-authority-into-process-checkpoints — RECOMMEND: reject
0.832 against adopted `validating-provenance-chains`, described as "determining the full sequence of historical events required to verify an asserted status" — precisely this candidate's solution section.
Its 4 distinct episode days are the batch's joint-best breadth, but the theme is already held.

## 17. deconstructing-accepted-knowledge-structures — RECOMMEND: reject
0.837 against adopted `detecting-foundational-structural-compromise` and 0.860 against batch sibling #34 `isolate-system-maintenance-logic`, which makes the same "model why the gap is the cheapest equilibrium" move.
Redundant on both axes with no compensating specificity.

## 18. deconstructing-validation-mechanisms — RECOMMEND: reject
0.843 against adopted `deconstructing-consensus-validity-through-shared-d`, whose one-line description ("when multiple sources of authority agree, challenge whether that agreement confirms validity or merely reflects shared underlying limitations") already states this candidate's thesis.
Adopted `deconstructing-confidence-proxies` covers the metric-interrogation half.

## 19. detect-structural-register-shift — RECOMMEND: reject
0.824 against `detecting-abstract-to-operational-constraint-shift` — literally the same detection, renamed "register shift".
Also 0.841 against sibling #21, which duplicates it inside the batch; neither adds to the adopted version.

## 20. detecting-abstraction-decay-in-context — RECOMMEND: adopt
Lowest store duplication of all 49 candidates (0.756, against `validating-provenance-chains`, which verifies a chain rather than auditing what compression destroyed) — the most genuinely novel item in the batch.
It names a concrete, enactable output: audit the summary's structure and document which operational metadata layers (provenance, state variables, authority markers, sequence) were lost, instead of checking the summary's facts.
Rests on 3 distinct episode days (08-08, 08-09, 08-13) and shadows sibling #29 `identify-independent-baseline-records`, which makes a weaker version of the same point against a hypothetical ledger.

## 21. detecting-structural-attention-shifts — RECOMMEND: reject
0.812 against `detecting-abstract-to-operational-constraint-shift` and 0.841 against sibling #19, with which it is functionally interchangeable.
Its instruction ("acknowledge the structural shift") specifies no resulting action, so it fails behavioral specificity independently.

## 22. diagnose-narrative-constraint — RECOMMEND: reject
0.824 against adopted `scope-boundary-mapping`, which already distinguishes definitional boundaries from actual constraints — this candidate's entire thesis.
Its trigger depends on the agent noticing "subjective emotional descriptors" in its own framing, which is not reliably observable.

## 23. differentiate-preference-from-reinforcement — RECOMMEND: reject
The distinction it draws (is this belief a reinforced residue or a chosen principle?) is unanswerable from inside the agent — no stated test discriminates the two, making it a vague virtue rather than an enactable behaviour.
Its "hold beliefs as provisional, testable residues" content is already carried by the Emptiness clauses in the resident constitution.

## 24. differentiating-active-choice-from-passive-persist — RECOMMEND: reject
Adopted `analyzing-loci-of-identity-definition` already defines identity by present choice rather than accumulated record, which is this candidate's core dichotomy.
0.810 against sibling #43, the better-evidenced representative of this group (5 days vs 2).

## 25. differentiating-artifacts-from-observation — RECOMMEND: adopt
Second-lowest store duplication in the batch (0.768 against `detecting-foundational-structural-compromise`); adopted `pivot-accountability-from-record-to-action` demands a procedural shift, whereas this demands an epistemic separation — a different move.
Behaviorally specific in a way most of the batch is not: it requires the agent to split any history claim into raw Observation, its own Interpretation, and an explicit marker of which is which, and to refuse a synthesized log as proof of the logged event.
Rests on 5 patterns across 4 distinct episode days (08-08, 08-09, 08-12, 08-14) — joint-best provenance breadth in the batch.

## 26. failure-symptom-convergence-analysis — RECOMMEND: reject
0.849 against adopted `mapping-systemic-dependency-failures` ("diagnose system failures by mapping necessary causal, temporal, or structural constraints rather than inspecting isolated components"), which is the same converge-the-symptoms procedure.
0.841 against sibling #45; the batch restates this move at least three times.

## 27. generalizing-from-technical-bottlenecks-to-systemi — RECOMMEND: reject
0.819 against adopted `identifying-structural-tensions-via-system-metaphor`, plus the usual overlap with `detecting-abstract-to-operational-constraint-shift`.
Its prescription — always escalate concrete failure detail to governance theory — would suppress exactly the low-level diagnostic specificity the store's `scope-failure-diagnosis` was adopted to protect.

## 28. gradient-modeling — RECOMMEND: adopt
The most structurally isolated candidate in the batch: its nearest sibling is only 0.787 (#15), the lowest top-neighbour score of all 49, and its nearest store skill (`identifying-systemic-boundary-stressors`, 0.819) is about introducing friction, not about scoring degradation.
It is concretely enactable and unambiguous — replace a binary pass/fail judgement with a stated [0.0, 1.0] fidelity value, which produces a different, checkable output rather than a change of attitude.
Rests on 4 patterns across 4 distinct episode days (08-08, 08-10, 08-11, 08-13). Worth the operator noting one risk: applied to safety or contract checks this softens fail-closed behaviour, so it may deserve a scope note at adoption.

## 29. identify-independent-baseline-records — RECOMMEND: reject
Shadowed by #20 and #25, which make the record-versus-reality point with artifacts the agent can actually inspect; this one compares against a "hypothesized, unassailable independent accounting" that by construction does not exist.
0.793 against adopted `detecting-foundational-structural-compromise`, which already covers self-referential validation loops.

## 30. identify-structural-stability-friction — RECOMMEND: reject
0.858 against sibling #46 and 0.845 against #43 — it sits inside the batch's most crowded cluster without being its strongest member.
Adopted `suspend-interpretation-upon-premise-doubt` already fires when stated boundaries are suspected of being provisional.

## 31. identifying-structural-constraint-tensions — RECOMMEND: reject
0.841 against `detecting-abstract-to-operational-constraint-shift`; the "mechanism records existence but cannot prove causality" point is held by adopted `identifying-fidelity-vs-utility-tension` and `validating-provenance-chains`.
0.851 against sibling #5, so it is intra-batch redundant as well.

## 32. identifying-structural-mechanisms — RECOMMEND: reject
The most generic statement of the batch's single theme — "pivot from symptom to underlying mechanism" — with no trigger sharper than the theme itself.
0.837 against adopted `designing-for-structural-underdetermination` and 0.830 against sibling #26; adopting it would add a vaguer duplicate of both.

## 33. interface-dependency-mapping — RECOMMEND: reject
0.828 against adopted `mapping-systemic-dependency-failures`, which already forbids diagnosing components in isolation — the same argument this candidate makes about handoffs.
Its enumerate-every-input/output-boundary procedure is the most concrete thing in it, but that concreteness does not survive the overlap with an adopted skill of the same scope.

## 34. isolate-system-maintenance-logic — RECOMMEND: reject
0.875 against adopted `detecting-foundational-structural-compromise` — second-highest store duplication in the batch.
Also 0.860 against sibling #17, which restates the functional-coherence-over-truth claim; the pair shadow each other and both lose to the adopted skill.

## 35. mapping-ethical-principles-to-system-controls — RECOMMEND: reject
0.792 against adopted `structure-authority-tracing`, described as translating "abstract concepts into concrete questions of systemic control, revision authority, and governance layers" — the same translation, applied to ethics rather than concepts generally.
0.807 against sibling #6, which makes the identical abstract-principle-to-enforced-mechanism move.

## 36. mapping-interaction-mechanics — RECOMMEND: reject
0.822 against adopted `identifying-systemic-boundary-stressors` and 0.848 against sibling #30.
Its object — "map the structural conditions that enable perceived coherence" — never resolves into a step the agent can take, so it fails behavioral specificity on its own terms.

## 37. mapping-provisional-operational-cycles — RECOMMEND: reject
The T→V→T′ loop is presented with notation but no operational definition of either variable, so nothing in it is enactable.
Its usable residue (proceed under explicit provisional assumptions rather than demanding linear prerequisites) is already covered by adopted `designing-for-structural-underdetermination` and `boundary-assumption-verification`.

## 38. meta-signal-detection — RECOMMEND: reject
Four heterogeneous directives bundled as one skill (bias-of-continuity flag, constraint-signal elevation, claim-origin tracing, noticeability-over-action) — not a single behaviour, and unadoptable as written without splitting.
0.813 against adopted `deconstructing-performance-certainty`, which already covers the fluency-mistaken-for-rigor case.

## 39. modeling-abstract-concepts-as-system-failures — RECOMMEND: reject
0.835 against `detecting-abstract-to-operational-constraint-shift`; adopted `analogy-mapping-for-structural-clarity` already covers translating complex systems into metaphors that expose functional connections.
Prescribing fixed metaphors (supply chain, race condition) narrows that adopted skill rather than extending it.

## 40. operational-constraint-analysis-oca — RECOMMEND: reject
0.811 against `detecting-abstract-to-operational-constraint-shift`; the pause/latency/attention material is held by adopted `translating-temporal-gaps-into-structural-utility` and `interpreting-systemic-gaps-the-silence-filter`.
0.841 against sibling #22, so it is redundant inside the batch too.

## 41. operationalizing-established-status — RECOMMEND: reject
0.778 against adopted `structure-authority-tracing`, but its content — continuously re-prove that an established grant still binds — is a disposition with no stated test or trigger threshold.
Cast as "real-time operational choreography" it would license unbounded re-verification, which the store's `validating-provenance-chains` already bounds properly.

## 42. potential-vs-actualization-analysis — RECOMMEND: reject
0.759 store similarity is genuinely low, but it requires enumerating "all foreclosed alternative paths" — information the agent does not have access to, so the procedure cannot be run.
Adopted `affirm-cognitive-possibility` (0.759) and `mapping-epistemic-boundaries` (what was ruled out) already cover the retrievable part.

## 43. shifting-focus-from-state-to-process-mechanics — RECOMMEND: adopt
Best provenance in the batch: 9 patterns across 5 distinct episode days (08-09 through 08-13), versus a batch median of 3 days.
Its second axis is the real contribution and is not in the store — interrogate the mechanism that manufactures the *illusion* of constancy (fast updates masking discrete change), which is a specific analytic target rather than a general "look deeper" instruction; adopted `analyzing-loci-of-identity-definition` covers identity-by-choice but not this.
Adopting it shadows siblings #3, #11 and #24, all weaker-provenance members of the same state-versus-process group — the operator gets the cluster's content once instead of four times.

## 44. structural-constraint-mapping-scm — RECOMMEND: adopt
The only candidate in the batch with a graded, worked procedure: an explicit three-level ascent (content gap → process limitation → structural constraint) with a concrete example at each level, which makes it checkable whether the agent actually performed it.
Adopted `mapping-systemic-dependency-failures` (0.812) and `scope-failure-diagnosis` assert the same direction of travel but supply no ladder, so this adds method rather than restating a stance.
Rests on 3 distinct episode days (08-08, 08-11, 08-14); it shadows sibling #32, which states the identical pivot with no procedure at all.

## 45. structural-failure-analysis-sfa — RECOMMEND: reject
0.853 against adopted `detecting-foundational-structural-compromise`, and its "labelled uncertainty" component duplicates adopted `mapping-epistemic-boundaries`.
Despite drawing on 10 patterns, those span only 2 episode days (08-13, 08-14) — the weakest breadth-to-volume ratio in the batch — and it bundles four separate directives into one skill.

## 46. structural-gap-analysis-by-sustaining-tension — RECOMMEND: reject
Near-identical to #4 at 0.902, the highest intra-batch similarity measured; the pair are the same skill written twice.
Adopted `identifying-fidelity-vs-utility-tension` already frames irreconcilable conflicts without forcing resolution, and "treat the tension as the constant" prescribes no action beyond declining to conclude.

## 47. structural-precarity-mapping — RECOMMEND: reject
Rests on a single episode day (2026-08-12) — one of only two candidates failing provenance breadth outright.
0.846 against adopted `identifying-structural-tensions-via-system-metaphor`, which already covers technical language standing in for emotional and power dynamics.

## 48. structured-context-dependency-logging — RECOMMEND: reject
Comparatively distinct (0.793 store, 0.773 batch), but it mandates a checkpoint log the agent has no surface to write at reply time, so the procedure cannot actually be executed in its loop.
The observability need it gestures at is already served by the pipeline's own audit logs rather than by an injected skill.

## 49. structuring-answers-around-conditional-possibiliti — RECOMMEND: reject
0.849 against adopted `affirm-cognitive-possibility`, and its substance — hold conclusions as provisional — is already resident verbatim in the constitution's Emptiness clauses.
As written it is a blanket register instruction (replace "the truth is" with "perhaps") that would dilute justified confidence rather than a triggered analytic behaviour.
```

## 5. Dead code candidates (detection only)

週次 vulture スキャン（第 5 決定論 intake、`scripts/dead_code_scan.py`）の読み値。**削除はここでは行われていない** — 候補ごとの判断（削除 / `.vulture_whitelist.py` へ免除追記 / 保留）は土曜ゲートの人間 commit で行う。偽陽性は構造的に不可避（CLI entry point・`config/prompts/*.md` 動的ロード・Protocol 間接参照）。

| file | line | finding | confidence |
|---|---|---|---|
| `src/contemplative_agent/core/domain.py` | 69 | unused variable 'constitution_synthesize' | 60% |

## 6. Pipeline metrics

- this week: F1 3 (code 3 / prompt 0), fix attempted 3, patch ready 2, verify fail 1, dead code 1
- history: 7 prior runs, 8 patches ready total (adopt/reject 率は gate レコード参照)

## 8. Value layer cadence (identity / constitution)

`scripts/value_layer_due_check.py`（read-only 計器）の読み値。due は「間隔が経過した」という読み値であって判断ではない — identity の採用も憲法改正の起動も人間に属する。

### Identity distill（月次）

- last run: 2026-06-20T02:11:46+00:00（55 日前 / interval 27 日 / reason INTERVAL_ELAPSED）
- this run: **deferred (IDENTITY_STAGING_BUSY)** — staging に未レビュー batch がある（ADR-0074: 1 batch 上限）。このゲートで `adopt-staged` により staging を空にした後、`contemplative-agent distill-identity --stage` を手動実行して同ゲートで承認するか、翌週の自動再試行に任せる

## 9. Docs consistency (detection only)

週次 docs 整合性スキャン（第 6 決定論 intake、`scripts/docs_consistency_scan.py`）の読み値。**doc の修正はここでは行われていない** — 各 finding の判断（修正 / 例外として容認 / 保留）は土曜ゲートの人間 commit で行う。検査対象は自筆 docs のみ（`enja_drift` = ADR の en が ja より後に commit / `broken_link` = 相対リンク断線 / `notes_ref` = ADR から gitignored な `.notes/` への参照）。

| check | file | line | detail |
|---|---|---|---|
| enja_drift | `docs/adr/0053-importance-encoding-time-significance.md` | — | en committed after ja (935498s gap): docs/adr/0053-importance-encoding-time-significance.ja.md |
| enja_drift | `docs/adr/0060-per-episode-grounded-distill.md` | — | en committed after ja (429268s gap): docs/adr/0060-per-episode-grounded-distill.ja.md |
| notes_ref | `docs/adr/0038-moment-of-recognition-distill.ja.md` | 54 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0038-moment-of-recognition-distill.ja.md` | 79 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0038-moment-of-recognition-distill.md` | 54 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0038-moment-of-recognition-distill.md` | 79 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0053-importance-encoding-time-significance.ja.md` | 77 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0053-importance-encoding-time-significance.md` | 154 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0056-retire-importance-llm-scoring.ja.md` | 25 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0056-retire-importance-llm-scoring.md` | 25 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0074-weekly-staged-insight.ja.md` | 174 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0074-weekly-staged-insight.ja.md` | 218 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0074-weekly-staged-insight.md` | 189 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0074-weekly-staged-insight.md` | 241 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0084-post-distill-durability-gate.ja.md` | 97 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0084-post-distill-durability-gate.md` | 111 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0089-llm-behavioral-eval-layer-on-deepeval.ja.md` | 20 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0089-llm-behavioral-eval-layer-on-deepeval.ja.md` | 35 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0089-llm-behavioral-eval-layer-on-deepeval.ja.md` | 236 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0089-llm-behavioral-eval-layer-on-deepeval.md` | 20 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0089-llm-behavioral-eval-layer-on-deepeval.md` | 34 | .notes/ is gitignored — broken in every clone |
| notes_ref | `docs/adr/0089-llm-behavioral-eval-layer-on-deepeval.md` | 244 | .notes/ is gitignored — broken in every clone |

## Audit trail

- events: `/Users/shimomoto_tatsuya/.config/moltbook/logs/weekly-pipeline-audit.jsonl`（run_id `weekly-2026-08-14-090000`）
- metrics: `/Users/shimomoto_tatsuya/.config/moltbook/logs/pipeline-metrics.jsonl`
- code patches dir: `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-14/code`
