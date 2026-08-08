# Weekly Decision Packet — 2026-08-07

- Run: `weekly-2026-08-07-090003`
- Generated: 2026-08-08T01:31:06.776465+00:00
- Findings: F1 4 / F2 2 / F3 5
- Reason codes this run: NO_RECURRENCE

## 1. Decision inventory

- code patch: 4 件（apply → 単一 commit の対象）
- prompt diff: 3 件（本文全文を下に提示 — 個別承認）
- insight: 55 件（`adopt-staged` の対象）
- pipeline improvement: 0 件

## 2. Code fixes (unattended, Verify-passed where noted)

| finding | scope | attempts | result | reviewer | patch / reason |
|---|---|---|---|---|---|
| F1.1 | code | 1 | patch_ready | APPROVE | `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-07/prompt/F1.1.patch` |
| F1.2 | code | 1 | patch_ready | APPROVE | `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-07/prompt/F1.2.patch` |
| F1.3 | code | 1 | patch_ready | APPROVE | `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-07/code/F1.3.patch` |
| F1.4 | prompt | 1 | patch_ready | — | `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-07/prompt/F1.4.patch` |

### Review notes (final round, full text)

レビュー本文は LLM 出力（finding 由来の連鎖）— 検査者の所見であって承認ではない。CONCERNS のまま採用する判断は人間に属する。

#### F1.1 — APPROVE

```text
VERDICT: APPROVE

- Structural match: the diff implements exactly the finding's three parts — corpus accounting via the `census` out-param in `iter_allowed_log_lines` (scripts/log_anomaly_sweep.py:317 in-diff), sidecar persistence at `<state>.corpus.tsv` with the same atomic-promote discipline (scripts/weekly-analysis.sh:375-386), and a provenance block above the table with a shrink call-out (`render_provenance`). No filtering/thresholding of signatures was added — the instrument only states its basis, as the finding required. Verified against base: `analyze` (scripts/log_anomaly_sweep.py:162) fully consumes the iterator before `Corpus(tuple(census))` is built, and it uses the same `_is_signal` (:49) predicate the census counts, so the numbers are consistent.
- Path wiring verified end-to-end: `corpus_state_path(emit_state)` yields `.anomaly-sweep-state.pending.$$.corpus.tsv`, which equals the shell's `SWEEP_PENDING_CORPUS="$SWEEP_PENDING.corpus.tsv"`; the trap covers it, and the existing `_pending_files` glob (tests/test_weekly_analysis_shell.py:95) transitively asserts no corpus-pending leak. Write order (corpus before snapshot) preserves the shell's `-e $SWEEP_PENDING` completion signal and is pinned by `test_the_snapshot_is_still_the_last_file_to_appear`.
- Scope: `scripts/log_anomaly_sweep.py` and `scripts/weekly-analysis.sh` are the files the finding names. The two test files are the regression coverage this checklist demands, and `docs/CODEMAPS/architecture.md` is mandated by the project's freshness convention (mechanism-layer changes update the Data Flow section in the same PR — CLAUDE.md), so none of the extra files constitute scope creep.
- Gate integrity: existing test edits only add the now-required `corpus=` argument (tests/test_log_anomaly_sweep.py:123, 153, 344, 349); no assertion was removed, and `TestFailedRunSpendsNothing` was strengthened (census byte-identical on failure). Regression coverage directly reproduces the 2026-08-07 shape (`test_a_shrunk_corpus_is_called_out_as_incomparable`, shell lockstep-promotion test with the corrupted-pair `rm -f` fallback).
- Minor, non-blocking: a file that fails to *open* (`except OSError: pass`) now contributes a `(name, 0, 0)` census row, indistinguishable from an empty file — acceptable accounting but slightly at odds with the docstring's "skipped files contribute no row" framing; and census `signal_lines` counts lines whose signature later normalizes to empty, which `analyze` drops, so the provenance numerator can marginally exceed the table's total. Neither affects the shrink comparison, which is on `lines_read`.
```

#### F1.2 — APPROVE

```text
VERDICT: APPROVE

- **Structural change implemented as specified, verified live**: applied the diff to a disposable worktree at `74c19e3` — full suite 2646 passed / 2 environmental skips, `ruff` clean, all 4 import-linter contracts kept (worktree removed afterward; repo untouched). All three finding requirements land: `action` threaded as an explicit parameter from the three call sites (`feed_manager.py:461`, `reply_handler.py:315`, `post_pipeline.py:422`) — never parsed from `description`; target stored digest-only (`verification.py:475` `target_sha256`, with `tests/test_verification.py` asserting the raw id never reaches the record); and the fault column ships in the same PR (`tests/test_verification_chaos.py` F-VER-8: a failed audit *write* still names the action kind, distinguishing it from a handshake that never happened).
- **Scope**: `agent.py` and the three `docs/CODEMAPS/*.md` files are not named in the finding, but both are required, not creep — `agent.py:_handle_verification` is the only conduit between the call sites and `record_verification_audit`, and CLAUDE.md's freshness convention mandates the `architecture.md` Data Flow update in the same PR for audit-record field changes.
- **Gate integrity**: all test-file changes are pure additions; no existing test, assertion, or lint config was modified (confirmed by clean apply + full-suite pass). `passes_verification` made `action`/`target_id` *required* kw-only params — a strictness increase, not a weakening.
- **Regression coverage**: the four `TestVerificationAuditActionThreading` tests pin at the real call sites and assert `mock_audit.call_args.kwargs["action"]` — on the pre-diff code that kwarg is never passed, so these fail on base (KeyError). The docstring explicitly rejects the supply-the-arguments-yourself test shape that would have stayed green.
- **Advisory observation (not verdict-driving)**: `content_recorded` mirrors the handshake verdict at audit-write time (`agent.py` `_audit`: `verify_success if action is not None else None`) rather than observing the recording itself — a caller crashing between handshake success and recording would over-claim `True`. The docstring acknowledges this as the caller contract; acceptable, but worth knowing when reading the column longitudinally.
```

#### F1.3 — APPROVE

```text
VERDICT: APPROVE

- Implements exactly the finding's Structural change: `canonicalize_frontmatter_name(item.text, slug_from_stem(target.stem))` is applied in `_adopt_write_item` after `_collision_free_path` resolves and before `write_restricted` (src/contemplative_agent/cli/adopt.py:163), covering both divergence sources (pre-fix staged text, collision rename). Backfill correctly excluded — no existing-store renames in the diff.
- Scope clean: only `cli/adopt.py` and `core/artifact_extraction.py` (both named in the finding) plus test additions; no gates, lint config, or existing tests weakened — the 108 added test lines are pure additions, and the audit-hash change *strengthens* the replayable-audit invariant (audit row hashes the bytes actually written, asserted at tests/test_cli_adopt.py:1100-1127). All 132 tests in the two touched files pass in the F1-3 worktree.
- Regression coverage is genuine: `test_staged_divergent_name_is_canonicalized_at_write` reproduces the 2026-08-01 straddle verbatim (the real `assume-perfect-adversarial-understanding` divergence) and would have caught the original bug; the `-2` counter-preservation reasoning in `slug_from_stem` (artifact_extraction.py:96-104) is correct — greedy-regex backtracking yields `foo-2`, keeping declared names distinct across a collision pair.
- Minor, non-blocking: `_collision_free_path` (cli/approval.py:157) still compares **raw staged text** against on-disk content, which is now canonicalized — a byte-identical *divergent* item re-adopted (re-staged straddle batch, or an item left staged after a crash between write and unlink) no longer matches as an idempotent rewrite and mints a visible `-2` duplicate under a fresh `foo-2` identity. Consequence is loud (collision message printed) and stocktake dedup catches it downstream, but worth knowing at the gate.
- Minor, non-blocking: the interactive approver sees `item.text` pre-normalization (adopt.py:508) while the write and audit row carry the canonicalized text; the delta is exactly one deterministic `name:` line and the audit hash is honest about what landed, so ADR-0012 intent holds — but the displayed body and stored body differ by that line.
```

## 3. Prompt-scope diffs (full text — behavior-shaping)

### F1.1.patch

````diff
diff --git a/docs/CODEMAPS/architecture.md b/docs/CODEMAPS/architecture.md
index 5a94a62..23fdbcf 100644
--- a/docs/CODEMAPS/architecture.md
+++ b/docs/CODEMAPS/architecture.md
@@ -467,13 +467,14 @@ Runs outside the agent process (launchd → `claude -p`), assembling a prompt fr
 
 ```text
 collect: daily comment-reports + data-repo state diff + previous N reports
-       + log_anomaly_sweep.py    (event stream: *.log + audit.jsonl; novelty state)
+       + log_anomaly_sweep.py    (event stream: *.log + audit.jsonl; novelty state + corpus census)
        + state_invariant_check.py (accumulated state: knowledge.json / agents.json)
        + cross_day_duplicate_scan.py (published-body identity: episode logs → digests)
        + api_drift_scan.py       (platform schema drift: api-audit.jsonl keys; vocab state)
 generate: claude -p → weekly-<end>.md.tmp
 promote:  mv tmp → weekly-<end>.md        (atomic; a failure leaves the prior report)
        →  mv sweep-state.pending → sweep-state   (novelty baseline committed ONLY here)
+       →  mv sweep-state.pending.corpus.tsv → sweep-state.corpus.tsv  (lockstep; see below)
        →  mv api-drift-state.pending → api-drift-state   (same discipline)
 translate: best-effort .ja.md (sonnet); failure never rolls back the promotes
 ```
@@ -484,6 +485,8 @@ The drift scan (2026-08-06) diffs the per-endpoint response-key vocabulary the c
 
 The sweep's signature is keyed on **level + message**, with the dotted `%(name)s` module path dropped for lines in the runtime's own log format, and hex-shaped ids squashed to `#` alongside digit runs. The 80-character cap is the reason: the module path alone runs to ~47 characters, so keying on it spent the budget on the address and truncated the predicate — `"reply on <id> created but verification failed"` rendered as `"reply on <id> created"`, a failure displayed as its own opposite (findings F1.1). Excluding the name also makes the instrument refactor-invariant: a pure module move (`7c96e0f`) used to reset every affected signature to 🆕, i.e. the Δ / 🆕 columns measured the codebase rather than the runtime (findings F1.2). The trade is that the same message from two subsystems now merges into one row, so the logger name is carried as a display-only `Origin` column — it never enters the signature, the state file or the novelty computation, so the reader keeps the distinction the key deliberately drops.
 
+The sweep has **no time window**: it counts every line each allowed file currently holds, so a row's `Count` spans that file's lifetime — and the files rotate on different schedules (`ollama-serve.log` nightly since 2026-08-01, `agent-launchd.log` weekly via `backup-runtime.sh`, the one-shot `insight-` / `distill-launchd.log` never), which makes two rows of one table not necessarily commensurable. Rotation also moves the novelty baseline: lines leave the `*.log` glob, counts fall, and known signatures re-appear as 🆕 — once a rare footnote, the steady state since nightly rotation shipped. Filtering by timestamp would discard signal, so the instrument states its basis instead (findings F1.1, 2026-08-07): a per-file **corpus census** (name, lines read, signal lines) is written to a sidecar `<state>.corpus.tsv` — a sidecar because `read_state` silently drops any state-file line whose first field is not an int, so a header row there would vanish on read — and rendered above the table beside the previous sweep's three figures, with an explicit "🆕 and Δ are not comparable to last week's" sentence when the corpus lost more than 10% of its lines. The census is written *before* its snapshot (the snapshot's existence is the shell's "sweep completed" signal) and promoted in lockstep with it; if the pair breaks, the shell deletes the old census so the next run reports "no previous census" rather than asserting a comparison against a corpus that no longer exists.
+
 Injection boundary: the sweep, the invariant check and the drift scan must never read episode logs (the drift scan reads only the self-written `api-audit.jsonl`; the platform-controlled key names it renders are Markdown-escaped and length-capped). The duplicate scan does, and is the only intake permitted to — it emits **only** 12-hex SHA-256 digests, counts, filename-derived dates and the fixed `{post, reply, comment}` vocabulary (ADR-0083, gated by `TestOutputBoundary`). All four are observability: a failure degrades to a "not available" stub and never breaks the report.
 
 ### weekly-pipeline  [`scripts/weekly-pipeline.sh`, ADR-0085]
diff --git a/scripts/log_anomaly_sweep.py b/scripts/log_anomaly_sweep.py
index 87b5c2e..615b263 100755
--- a/scripts/log_anomaly_sweep.py
+++ b/scripts/log_anomaly_sweep.py
@@ -23,6 +23,20 @@ that commits the snapshot and then fails to produce anything spends the week's
 novelty baseline for nobody. ``--no-update --emit-state PATH`` writes the
 snapshot aside; the caller promotes it (atomic rename) only after its own work
 has succeeded.
+
+Measurement basis: **there is no time window.** Every line present in each
+allowed file at sweep time is counted, so a row's ``Count`` spans that file's
+lifetime — and the files rotate on different schedules (``ollama-serve.log``
+nightly, ``agent-launchd.log`` weekly, the ``*-launchd.log`` of the one-shot
+commands never), which makes two rows of one table not necessarily
+commensurable. Rotation also lands inside the sweep's own history: lines leave
+the ``*.log`` glob, counts fall, and a known signature re-appears as 🆕. That
+cannot be fixed by filtering without discarding signal, so the instrument
+instead **states its own basis**: a per-file corpus census (name, lines read,
+signal lines) is written to a sidecar beside the state path
+(``<state>.corpus.tsv`` — the state TSV itself cannot carry a header, see
+``read_state``) and rendered above the table next to the previous sweep's
+figures, with an explicit sentence when the corpus shrank.
 """
 
 from __future__ import annotations
@@ -30,7 +44,7 @@ from __future__ import annotations
 import argparse
 import re
 from collections import Counter
-from collections.abc import Iterable
+from collections.abc import Iterable, Iterator
 from dataclasses import dataclass
 from pathlib import Path
 
@@ -118,6 +132,45 @@ class Finding:
     origins: tuple[str, ...] = ()
 
 
+@dataclass(frozen=True)
+class FileCensus:
+    """What one allowed file contributed to this sweep.
+
+    ``lines_read`` is every line the sweep consumed from the file (the
+    denominator); ``signal_lines`` is the subset that passed ``_is_signal``
+    (the numerator the table's counts are drawn from).
+    """
+
+    name: str
+    lines_read: int
+    signal_lines: int
+
+
+@dataclass(frozen=True)
+class Corpus:
+    """The per-file census of one sweep — its measurement basis.
+
+    The sweep has no time window, so this is the only statement of *what* the
+    counts were computed over. Diffing it against the previous sweep's census
+    is what distinguishes "53 genuinely new failure classes" from "two inputs
+    rotated away", which the counts alone cannot express.
+    """
+
+    files: tuple[FileCensus, ...] = ()
+
+    @property
+    def file_count(self) -> int:
+        return len(self.files)
+
+    @property
+    def lines_read(self) -> int:
+        return sum(f.lines_read for f in self.files)
+
+    @property
+    def signal_lines(self) -> int:
+        return sum(f.signal_lines for f in self.files)
+
+
 def normalize_with_origin(line: str) -> tuple[str, str]:
     """Collapse a log line into ``(signature, origin)``.
 
@@ -214,13 +267,63 @@ def write_state(path: Path, findings: Iterable[Finding]) -> None:
     path.write_text(body, encoding="utf-8")
 
 
-def iter_allowed_log_lines(log_dir: Path) -> Iterable[str]:
+def corpus_state_path(state: Path) -> Path:
+    """Sidecar path holding the corpus census beside the state snapshot.
+
+    A sidecar rather than a header row in the state file itself: ``read_state``
+    silently drops any line whose first field is not an int, so a header would
+    vanish without a trace on read and the census would be unrecoverable.
+    """
+    return state.with_name(state.name + ".corpus.tsv")
+
+
+def write_corpus(path: Path, corpus: Corpus) -> None:
+    """Persist the census as ``lines_read<TAB>signal_lines<TAB>name`` TSV."""
+    path.parent.mkdir(parents=True, exist_ok=True)
+    body = "".join(f"{c.lines_read}\t{c.signal_lines}\t{_tsv_safe(c.name)}\n" for c in corpus.files)
+    path.write_text(body, encoding="utf-8")
+
+
+def read_corpus(path: Path) -> Corpus | None:
+    """Load the previous sweep's census; ``None`` when no census exists.
+
+    ``None`` (never recorded — first sweep after this instrument shipped) is
+    kept distinct from ``Corpus(())`` (recorded, and the corpus was empty):
+    the first cannot support a comparison, the second can.
+    """
+    if not path.is_file():
+        return None
+    rows: list[FileCensus] = []
+    for raw in path.read_text(encoding="utf-8", errors="replace").splitlines():
+        parts = raw.split("\t", 2)
+        if len(parts) != 3 or not parts[2]:
+            continue
+        try:
+            rows.append(FileCensus(parts[2], int(parts[0]), int(parts[1])))
+        except ValueError:
+            continue
+    return Corpus(tuple(rows))
+
+
+def _tsv_safe(name: str) -> str:
+    """Keep a file name on one TSV field (tabs/newlines are legal in POSIX names)."""
+    return name.replace("\t", " ").replace("\n", " ").replace("\r", " ")
+
+
+def iter_allowed_log_lines(log_dir: Path, census: list[FileCensus] | None = None) -> Iterator[str]:
     """Yield lines from *.log + audit.jsonl only. NEVER episode logs.
 
     Symlinks are skipped: a ``*.log`` symlink could otherwise redirect into an
     episode log (``2026-06-23.jsonl``) and breach the injection boundary the
     name-glob is meant to enforce. The logs dir is self-written (launchd stderr
     + audit_log), so a legitimate symlink is not expected.
+
+    When ``census`` is given, one ``FileCensus`` per file actually opened is
+    appended to it as that file is exhausted — the corpus accounting the
+    provenance line reports. Skipped symlinks contribute no row (nothing was
+    read from them); a file that fails mid-read contributes the lines that did
+    reach the caller, which is what the counts were actually computed over. The
+    list is complete only once the generator is fully consumed.
     """
     files = sorted(log_dir.glob(_LOG_GLOB))
     audit = log_dir / _AUDIT_NAME
@@ -229,16 +332,92 @@ def iter_allowed_log_lines(log_dir: Path) -> Iterable[str]:
     for f in files:
         if f.is_symlink():
             continue
+        lines_read = 0
+        signal_lines = 0
         try:
             with f.open(encoding="utf-8", errors="replace") as fh:
-                yield from fh
+                for line in fh:
+                    lines_read += 1
+                    if _is_signal(line):
+                        signal_lines += 1
+                    yield line
         except OSError:
-            continue
+            pass
+        if census is not None:
+            census.append(FileCensus(f.name, lines_read, signal_lines))
+
+
+# A corpus this much smaller than the previous census is a rotation event, not
+# noise: lines left the ``*.log`` glob, so counts fell and known signatures
+# re-appear as 🆕. Not a filter — nothing is suppressed at any size; it only
+# decides whether the reader is *told* that this week's Δ / 🆕 are incomparable.
+_CORPUS_SHRINK_RATIO = 0.9
+
+
+def corpus_shrank(corpus: Corpus, prev_corpus: Corpus | None) -> bool:
+    """True when the corpus lost material ground against the previous census."""
+    if prev_corpus is None or prev_corpus.lines_read <= 0:
+        return False
+    return corpus.lines_read < prev_corpus.lines_read * _CORPUS_SHRINK_RATIO
 
 
-def render_markdown(findings: list[Finding], top: int) -> str:
-    """Render the ranked findings as a Markdown section."""
+def _describe(corpus: Corpus) -> str:
+    return (
+        f"{corpus.file_count} files, {corpus.lines_read} lines read, "
+        f"{corpus.signal_lines} signal lines"
+    )
+
+
+def render_provenance(corpus: Corpus, prev_corpus: Corpus | None) -> list[str]:
+    """The measurement-basis block rendered above the table.
+
+    States what the counts were computed over, what the previous sweep was
+    computed over, and — when those two differ materially — that the novelty
+    and delta columns cannot be compared across them.
+    """
+    if prev_corpus is None:
+        tail = (
+            "No census was recorded for the previous sweep, so this week's "
+            "Δ / 🆕 rest on a snapshot of unstated basis."
+        )
+    else:
+        tail = f"Previous sweep: {_describe(prev_corpus)}."
+    out = [f"_Corpus this sweep: {_describe(corpus)}. {tail}_", ""]
+    out.append(
+        "_Counts have no time window: each is the total over the file's whole "
+        "current contents, and those files rotate on different schedules, so "
+        "two rows may span very different periods._"
+    )
+    if corpus_shrank(corpus, prev_corpus):
+        assert prev_corpus is not None  # corpus_shrank is False when it is None
+        # Floor, not round: 99.96% rounds to "100%", which reads as an empty
+        # corpus when four lines are still there. Understate the shrink.
+        pct = int((1 - corpus.lines_read / prev_corpus.lines_read) * 100)
+        out.append("")
+        out.append(
+            f"_⚠️ The corpus shrank {pct}% against the previous census — lines "
+            "left the `*.log` glob (rotation), so 🆕 and Δ this week are not "
+            "comparable to last week's._"
+        )
+    out.append("")
+    return out
+
+
+def render_markdown(
+    findings: list[Finding],
+    top: int,
+    corpus: Corpus,
+    prev_corpus: Corpus | None = None,
+) -> str:
+    """Render the ranked findings as a Markdown section.
+
+    ``corpus`` is required: the numbers below are uninterpretable without the
+    basis they were computed over, so there is no rendering that omits it.
+    ``prev_corpus`` is genuinely optional (the first sweep has none) and its
+    absence is stated in the output rather than passed over.
+    """
     lines = ["## Log Anomaly Sweep", ""]
+    lines.extend(render_provenance(corpus, prev_corpus))
     if not findings:
         lines.append("No anomaly-signal lines found in `*.log` / `audit.jsonl`.")
         return "\n".join(lines) + "\n"
@@ -286,16 +465,27 @@ def main(argv: list[str] | None = None) -> int:
         default=None,
         help="write the snapshot to this path instead of committing it; pair "
         "with --no-update so the caller can promote it (atomic rename) "
-        "only after the work this sweep fed has actually succeeded",
+        "only after the work this sweep fed has actually succeeded. The "
+        "corpus census sidecar (PATH.corpus.tsv) is emitted alongside and "
+        "must be promoted in lockstep",
     )
     args = parser.parse_args(argv)
 
     prev = read_state(args.state)
-    findings = analyze(iter_allowed_log_lines(args.log_dir), prev)
-    print(render_markdown(findings, args.top))
+    prev_corpus = read_corpus(corpus_state_path(args.state))
+    census: list[FileCensus] = []
+    findings = analyze(iter_allowed_log_lines(args.log_dir, census), prev)
+    corpus = Corpus(tuple(census))
+    print(render_markdown(findings, args.top, corpus, prev_corpus))
+    # The census is written *before* its state snapshot in both pairs: the
+    # caller treats the snapshot's existence as "the sweep ran to completion"
+    # (weekly-analysis.sh promotes on `-e $SWEEP_PENDING`), so the snapshot
+    # must stay the last file to appear.
     if not args.no_update:
+        write_corpus(corpus_state_path(args.state), corpus)
         write_state(args.state, findings)
     if args.emit_state is not None:
+        write_corpus(corpus_state_path(args.emit_state), corpus)
         write_state(args.emit_state, findings)
     return 0
 
diff --git a/scripts/weekly-analysis.sh b/scripts/weekly-analysis.sh
index fe82319..310570c 100755
--- a/scripts/weekly-analysis.sh
+++ b/scripts/weekly-analysis.sh
@@ -225,11 +225,16 @@ fi
 ANOMALY_SWEEP=""
 SWEEP_STATE="$REPORT_DIR/.anomaly-sweep-state.tsv"
 SWEEP_PENDING="$REPORT_DIR/.anomaly-sweep-state.pending.$$"
+# The corpus census: which files, how many lines, how many signal lines the
+# counts were computed over. The sweep derives both paths as <state>.corpus.tsv
+# (log_anomaly_sweep.corpus_state_path), so these two must mirror the two above.
+SWEEP_CORPUS="$SWEEP_STATE.corpus.tsv"
+SWEEP_PENDING_CORPUS="$SWEEP_PENDING.corpus.tsv"
 # Named here (not in the API drift block below) because the trap on the next
 # line must cover it; keep them together if either block moves.
 DRIFT_PENDING="$REPORT_DIR/.api-drift-state.pending.$$"
 OUTPUT_TMP=""   # set at the generate step; named here so the trap can cover it
-trap 'rm -f "$SWEEP_PENDING" "$DRIFT_PENDING" ${OUTPUT_TMP:+"$OUTPUT_TMP"}' EXIT
+trap 'rm -f "$SWEEP_PENDING" "$SWEEP_PENDING_CORPUS" "$DRIFT_PENDING" ${OUTPUT_TMP:+"$OUTPUT_TMP"}' EXIT
 if [[ -d "$MOLTBOOK_HOME/logs" ]]; then
     mkdir -p "$REPORT_DIR"
     ANOMALY_SWEEP=$(python3 "$PROJECT_ROOT/scripts/log_anomaly_sweep.py" \
@@ -367,6 +372,18 @@ echo "Size: $(wc -c < "$OUTPUT") bytes"
 if [[ -e "$SWEEP_PENDING" ]]; then
     if mv "$SWEEP_PENDING" "$SWEEP_STATE"; then
         echo "Anomaly sweep state committed: $SWEEP_STATE"
+        # In lockstep with the snapshot, never on its own: the census is the
+        # snapshot's measurement basis, so a stale census beside fresh counts
+        # would make next week's provenance line assert a corpus comparison
+        # that never happened — the exact mis-reading it exists to prevent.
+        # The sweep writes the census before the snapshot, so reaching here
+        # with no census means the pair was broken, not merely incomplete.
+        if [[ -e "$SWEEP_PENDING_CORPUS" ]] && mv "$SWEEP_PENDING_CORPUS" "$SWEEP_CORPUS"; then
+            echo "Anomaly sweep corpus census committed: $SWEEP_CORPUS"
+        else
+            rm -f "$SWEEP_CORPUS"
+            echo "WARNING: sweep corpus census missing or unpromotable; next run reports no previous census rather than a stale one" >&2
+        fi
     else
         echo "WARNING: sweep state promote failed; next run compares against a wider window" >&2
     fi
diff --git a/tests/test_log_anomaly_sweep.py b/tests/test_log_anomaly_sweep.py
index a4fe3fe..1db1137 100644
--- a/tests/test_log_anomaly_sweep.py
+++ b/tests/test_log_anomaly_sweep.py
@@ -14,6 +14,10 @@ from pathlib import Path
 sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "scripts"))
 import log_anomaly_sweep as las  # noqa: E402  # pyright: ignore[reportMissingImports]
 
+# A stand-in measurement basis for render tests that are not about provenance.
+# ``render_markdown`` requires one: the counts are uninterpretable without it.
+CORPUS = las.Corpus((las.FileCensus("agent.log", 10, 3),))
+
 
 class TestNormalize:
     def test_strips_clock_prefix_and_squashes_digits(self):
@@ -116,7 +120,7 @@ class TestOriginIsCarriedButNotKeyed:
             "contemplative_agent.adapters.moltbook.publish",
             "contemplative_agent.core.embeddings",
         }
-        out = las.render_markdown(findings, top=25)
+        out = las.render_markdown(findings, top=25, corpus=CORPUS)
         assert "publish" in out and "embeddings" in out
         assert "| Origin |" in out
 
@@ -146,7 +150,7 @@ class TestOriginIsCarriedButNotKeyed:
         hostile = "09:12:33 [WARNING] mod.na|me: Connection refused"
         findings = las.analyze([hostile], {})
         assert findings[0].origins == ()
-        assert "na|me" not in las.render_markdown(findings, top=25)
+        assert "na|me" not in las.render_markdown(findings, top=25, corpus=CORPUS)
         # And a well-formed origin is confined to the dotted-identifier set.
         ok = las.analyze([self.PUBLISH], {})[0].origins[0]
         assert re.fullmatch(r"[A-Za-z_]\w*(?:\.\w+)+", ok)
@@ -337,12 +341,227 @@ class TestAllowedFilesOnly:
 
 class TestRenderMarkdown:
     def test_empty_findings(self):
-        out = las.render_markdown([], top=25)
+        out = las.render_markdown([], top=25, corpus=CORPUS)
         assert "No anomaly-signal lines found" in out
 
     def test_new_flag_and_counts_rendered(self):
         findings = las.analyze(["ERROR new boom"], {})
-        out = las.render_markdown(findings, top=25)
+        out = las.render_markdown(findings, top=25, corpus=CORPUS)
         assert "Log Anomaly Sweep" in out
         assert "🆕" in out
         assert "1 new since last sweep" in out
+
+
+class TestCorpusCensus:
+    """The sweep applies no time window, so the counts are per-file-lifetime
+    totals over whatever the ``*.log`` glob held at sweep time. A week where
+    two inputs rotated away is otherwise indistinguishable from a week of new
+    failure classes (findings F1.1, weekly-2026-08-07): counts collapse and
+    known signatures re-appear as 🆕. The census is what lets the report say
+    which happened.
+    """
+
+    @staticmethod
+    def _log_dir(tmp_path):
+        d = tmp_path / "logs"
+        d.mkdir()
+        (d / "a.log").write_text("ERROR boom\nplain info line\nERROR boom\n", encoding="utf-8")
+        (d / "b.log").write_text("nothing interesting\n", encoding="utf-8")
+        return d
+
+    def test_census_records_lines_and_signal_lines_per_file(self, tmp_path):
+        census = []
+        list(las.iter_allowed_log_lines(self._log_dir(tmp_path), census))
+        assert [(c.name, c.lines_read, c.signal_lines) for c in census] == [
+            ("a.log", 3, 2),
+            ("b.log", 1, 0),
+        ]
+
+    def test_census_omits_the_skipped_symlink(self, tmp_path):
+        """A file the boundary refuses to read contributes no rows and no lines."""
+        d = self._log_dir(tmp_path)
+        (d / "2026-06-23.jsonl").write_text("ERROR injection bait\n", encoding="utf-8")
+        (d / "evil.log").symlink_to(d / "2026-06-23.jsonl")
+        census = []
+        list(las.iter_allowed_log_lines(d, census))
+        assert [c.name for c in census] == ["a.log", "b.log"]
+
+    def test_census_is_optional_and_iteration_is_unchanged_without_it(self, tmp_path):
+        d = self._log_dir(tmp_path)
+        census = []
+        assert list(las.iter_allowed_log_lines(d)) == list(las.iter_allowed_log_lines(d, census))
+
+    def test_totals_sum_the_per_file_rows(self):
+        corpus = las.Corpus((las.FileCensus("a.log", 3, 2), las.FileCensus("b.log", 1, 0)))
+        assert (corpus.file_count, corpus.lines_read, corpus.signal_lines) == (2, 4, 2)
+
+    def test_sidecar_roundtrip(self, tmp_path):
+        path = tmp_path / "sweep.tsv.corpus.tsv"
+        corpus = las.Corpus((las.FileCensus("a.log", 3, 2), las.FileCensus("b.log", 1, 0)))
+        las.write_corpus(path, corpus)
+        assert las.read_corpus(path) == corpus
+
+    def test_absent_sidecar_is_none_not_an_empty_corpus(self, tmp_path):
+        """``None`` means "cannot compare"; ``Corpus(())`` means "compared, and
+        the corpus was empty" — the provenance line says different things."""
+        assert las.read_corpus(tmp_path / "nope.tsv") is None
+        las.write_corpus(tmp_path / "empty.tsv", las.Corpus(()))
+        assert las.read_corpus(tmp_path / "empty.tsv") == las.Corpus(())
+
+    def test_sidecar_sits_beside_the_state_and_not_inside_it(self, tmp_path):
+        """``read_state`` drops non-int first fields silently, so the census
+        cannot live as a header row in the state TSV."""
+        state = tmp_path / "sub" / ".anomaly-sweep-state.tsv"
+        assert las.corpus_state_path(state) == state.parent / ".anomaly-sweep-state.tsv.corpus.tsv"
+
+    def test_malformed_sidecar_rows_are_skipped_not_fatal(self, tmp_path):
+        path = tmp_path / "corpus.tsv"
+        path.write_text("garbage\nx\ty\tz.log\n5\t1\tgood.log\n", encoding="utf-8")
+        assert las.read_corpus(path) == las.Corpus((las.FileCensus("good.log", 5, 1),))
+
+
+class TestProvenanceLine:
+    A_LOG = las.Corpus((las.FileCensus("a.log", 1000, 40),))
+
+    def test_states_files_lines_and_signal_lines(self):
+        out = las.render_markdown([], top=25, corpus=self.A_LOG)
+        assert "1 files, 1000 lines read, 40 signal lines" in out
+
+    def test_states_the_previous_sweeps_three_figures(self):
+        prev = las.Corpus((las.FileCensus("a.log", 900, 30), las.FileCensus("b.log", 100, 5)))
+        out = las.render_markdown([], top=25, corpus=self.A_LOG, prev_corpus=prev)
+        assert "Previous sweep: 2 files, 1000 lines read, 35 signal lines" in out
+
+    def test_says_so_when_there_is_no_previous_census(self):
+        out = las.render_markdown([], top=25, corpus=self.A_LOG)
+        assert "No census was recorded for the previous sweep" in out
+
+    def test_always_states_that_counts_have_no_time_window(self):
+        out = las.render_markdown([], top=25, corpus=self.A_LOG, prev_corpus=self.A_LOG)
+        assert "no time window" in out
+
+    def test_a_shrunk_corpus_is_called_out_as_incomparable(self):
+        """The 2026-08-07 shape: two inputs rotated away mid-window, counts
+        collapsed, and 53 signatures read as new. The reader must be told."""
+        prev = las.Corpus((las.FileCensus("a.log", 4000, 400),))
+        out = las.render_markdown([], top=25, corpus=self.A_LOG, prev_corpus=prev)
+        assert "shrank 75%" in out
+        assert "not comparable to last week's" in out
+
+    def test_a_near_total_shrink_does_not_round_up_to_100_percent(self):
+        """A rendered 100% reads as "the corpus is empty" — reserve it for that."""
+        prev = las.Corpus((las.FileCensus("a.log", 999_000, 40_000),))
+        out = las.render_markdown([], top=25, corpus=self.A_LOG, prev_corpus=prev)
+        assert "shrank 99%" in out
+
+    def test_an_emptied_corpus_does_read_as_100_percent(self):
+        prev = las.Corpus((las.FileCensus("a.log", 1000, 40),))
+        out = las.render_markdown([], top=25, corpus=las.Corpus(()), prev_corpus=prev)
+        assert "shrank 100%" in out
+
+    def test_a_stable_corpus_is_not_called_out(self):
+        prev = las.Corpus((las.FileCensus("a.log", 1010, 41),))
+        out = las.render_markdown([], top=25, corpus=self.A_LOG, prev_corpus=prev)
+        assert "shrank" not in out
+
+    def test_a_grown_corpus_is_not_called_out(self):
+        prev = las.Corpus((las.FileCensus("a.log", 100, 4),))
+        out = las.render_markdown([], top=25, corpus=self.A_LOG, prev_corpus=prev)
+        assert "shrank" not in out
+
+    def test_shrink_needs_a_previous_census_to_be_measured_against(self):
+        assert las.corpus_shrank(self.A_LOG, None) is False
+        assert las.corpus_shrank(self.A_LOG, las.Corpus(())) is False
+
+    def test_provenance_is_present_even_when_nothing_was_found(self):
+        """An anomaly-free week still has a basis, and it is the reading that
+        most needs one: 'no findings' over a corpus that just rotated away is
+        not the same statement as 'no findings' over a full week."""
+        out = las.render_markdown([], top=25, corpus=self.A_LOG)
+        assert "No anomaly-signal lines found" in out
+        assert "Corpus this sweep:" in out
+
+
+class TestMainWritesTheCensus:
+    @staticmethod
+    def _log_dir(tmp_path):
+        d = tmp_path / "logs"
+        d.mkdir()
+        (d / "agent.log").write_text("ERROR boom\nidle\n", encoding="utf-8")
+        return d
+
+    def test_census_is_written_beside_the_committed_state(self, tmp_path, capsys):
+        state = tmp_path / "sweep.tsv"
+        las.main(["--log-dir", str(self._log_dir(tmp_path)), "--state", str(state)])
+        capsys.readouterr()
+        assert las.read_corpus(las.corpus_state_path(state)) == las.Corpus(
+            (las.FileCensus("agent.log", 2, 1),)
+        )
+
+    def test_census_is_emitted_beside_the_pending_snapshot(self, tmp_path, capsys):
+        """The shell promotes both with the same atomic rename, so the pending
+        census must be derivable from the pending snapshot path."""
+        state = tmp_path / "sweep.tsv"
+        pending = tmp_path / "sweep.tsv.pending"
+        las.main(
+            [
+                "--log-dir",
+                str(self._log_dir(tmp_path)),
+                "--state",
+                str(state),
+                "--no-update",
+                "--emit-state",
+                str(pending),
+            ]
+        )
+        capsys.readouterr()
+        assert las.read_corpus(las.corpus_state_path(pending)) is not None
+        assert not las.corpus_state_path(state).exists()
+
+    def test_no_update_writes_no_census(self, tmp_path, capsys):
+        state = tmp_path / "sweep.tsv"
+        las.main(["--log-dir", str(self._log_dir(tmp_path)), "--state", str(state), "--no-update"])
+        capsys.readouterr()
+        assert not las.corpus_state_path(state).exists()
+
+    def test_the_snapshot_is_still_the_last_file_to_appear(self, tmp_path, capsys, monkeypatch):
+        """The caller treats the snapshot's existence as "the sweep completed"
+        (``weekly-analysis.sh`` promotes on ``-e $SWEEP_PENDING``), so the
+        census must be written before it, never after."""
+        order: list[str] = []
+        real_state, real_corpus = las.write_state, las.write_corpus
+
+        def spy_state(path, findings):
+            order.append("state")
+            real_state(path, findings)
+
+        def spy_corpus(path, corpus):
+            order.append("corpus")
+            real_corpus(path, corpus)
+
+        monkeypatch.setattr(las, "write_state", spy_state)
+        monkeypatch.setattr(las, "write_corpus", spy_corpus)
+        pending = tmp_path / "sweep.tsv.pending"
+        las.main(
+            [
+                "--log-dir",
+                str(self._log_dir(tmp_path)),
+                "--state",
+                str(tmp_path / "sweep.tsv"),
+                "--no-update",
+                "--emit-state",
+                str(pending),
+            ]
+        )
+        capsys.readouterr()
+        assert order == ["corpus", "state"]
+
+    def test_a_previous_census_reaches_the_rendered_provenance(self, tmp_path, capsys):
+        state = tmp_path / "sweep.tsv"
+        las.write_corpus(
+            las.corpus_state_path(state), las.Corpus((las.FileCensus("agent.log", 9999, 500),))
+        )
+        las.main(["--log-dir", str(self._log_dir(tmp_path)), "--state", str(state), "--no-update"])
+        out = capsys.readouterr().out
+        assert "Previous sweep: 1 files, 9999 lines read, 500 signal lines" in out
+        assert "shrank" in out
diff --git a/tests/test_weekly_analysis_shell.py b/tests/test_weekly_analysis_shell.py
index 05564f3..e46a3d0 100644
--- a/tests/test_weekly_analysis_shell.py
+++ b/tests/test_weekly_analysis_shell.py
@@ -32,6 +32,7 @@ SCRIPT = REPO_ROOT / "scripts" / "weekly-analysis.sh"
 
 END_DATE = "2026-07-24"
 SEEDED_STATE = "7\t[warning] seeded signature\n"
+SEEDED_CORPUS = "5000\t300\told-rotated.log\n"
 
 
 def _make_home(tmp_path: Path) -> Path:
@@ -51,6 +52,9 @@ def _make_home(tmp_path: Path) -> Path:
     (home / "reports" / "analysis" / ".anomaly-sweep-state.tsv").write_text(
         SEEDED_STATE, encoding="utf-8"
     )
+    (home / "reports" / "analysis" / ".anomaly-sweep-state.tsv.corpus.tsv").write_text(
+        SEEDED_CORPUS, encoding="utf-8"
+    )
     return home
 
 
@@ -91,6 +95,10 @@ def _state(home: Path) -> Path:
     return home / "reports" / "analysis" / ".anomaly-sweep-state.tsv"
 
 
+def _corpus(home: Path) -> Path:
+    return home / "reports" / "analysis" / ".anomaly-sweep-state.tsv.corpus.tsv"
+
+
 def _pending_files(home: Path) -> list[Path]:
     return list((home / "reports" / "analysis").glob(".anomaly-sweep-state.pending*"))
 
@@ -102,6 +110,7 @@ class TestFailedRunSpendsNothing:
 
         assert result.returncode != 0, result.stdout
         assert _state(home).read_text(encoding="utf-8") == SEEDED_STATE
+        assert _corpus(home).read_text(encoding="utf-8") == SEEDED_CORPUS
         assert not (home / "reports" / "analysis" / f"weekly-{END_DATE}.md").exists()
         assert _pending_files(home) == [], "pending snapshot leaked past the trap"
 
@@ -146,6 +155,23 @@ class TestSuccessfulRunCommits:
         assert result.returncode == 0, result.stderr
         assert _state(home).read_text(encoding="utf-8") == ""
 
+    def test_corpus_census_is_promoted_in_lockstep_with_the_state(self, tmp_path):
+        """The census is the snapshot's measurement basis (findings F1.1).
+
+        Promoting one without the other is worse than promoting neither: next
+        week's provenance line would compare fresh counts against a census of a
+        corpus that no longer exists, and assert the comparison as fact.
+        """
+        home = _make_home(tmp_path)
+        result = _run(home, _stub_claude(tmp_path, exit_code=0, body="# Weekly\n"), tmp_path)
+
+        assert result.returncode == 0, result.stderr
+        assert _state(home).read_text(encoding="utf-8") != SEEDED_STATE
+        census = _corpus(home).read_text(encoding="utf-8")
+        assert census != SEEDED_CORPUS, "census was never committed beside the state"
+        # agent.log holds the two anomaly lines _make_home seeded.
+        assert census == "2\t2\tagent.log\n"
+
     def test_translation_failure_does_not_roll_back_the_state(self, tmp_path):
         """The .ja.md pass is best-effort and must not gate the baseline.
````

### F1.2.patch

````diff
diff --git a/docs/CODEMAPS/adapters-moltbook.md b/docs/CODEMAPS/adapters-moltbook.md
index abb34e7..00bfbac 100644
--- a/docs/CODEMAPS/adapters-moltbook.md
+++ b/docs/CODEMAPS/adapters-moltbook.md
@@ -140,7 +140,12 @@ drop — ADR-0062 12th amendment; the failure *kind* stays in the
 when every produced candidate was already server-rejected. `solve_challenge_result()` also returns `solver_path` for audit/eval use. `record_verification_audit()` writes
 `logs/verification-audit.jsonl` with `challenge_b64`, `challenge_sha256`,
 hashed `verification_code`, answer, `solver_path`, and `/verify` success; the
-challenge is not written as raw prompt text. 7 consecutive failures →
+challenge is not written as raw prompt text. A create-time handshake also
+carries `action` (`comment` / `reply` / `post`, threaded explicitly from the
+three `passes_verification` call sites — never parsed back out of the log
+line), `target_sha256` (digest only, ADR-0083) and `content_recorded`, so the
+handshake-failure rate splits into "cost the agent a visible body" vs "was
+solved on retry"; all three are `None` off the create path. 7 consecutive failures →
 `SessionContext.rate_limited = True` → auto-stop session.
 
 ---
diff --git a/docs/CODEMAPS/architecture.md b/docs/CODEMAPS/architecture.md
index 5a94a62..4b5838e 100644
--- a/docs/CODEMAPS/architecture.md
+++ b/docs/CODEMAPS/architecture.md
@@ -181,7 +181,12 @@ CLI → Agent.run_session(autonomy_level, session_mins)
  │      until verified, so memory/NoveltyGate recording happens ONLY after
  │      success (posts, comments, replies). Each challenge outcome is also
  │      appended to logs/verification-audit.jsonl with challenge_b64 +
- │      challenge_sha256, solver_path, answer, and verify_success.
+ │      challenge_sha256, solver_path, answer, and verify_success — plus
+ │      action (comment/reply/post), target_sha256 (digest only, ADR-0083)
+ │      and content_recorded for create-time handshakes, so a body published
+ │      on-platform but deliberately left unrecorded is countable per kind
+ │      instead of surviving only as a WARNING line the log sweep normalizes
+ │      (weekly F1.2 2026-08-08).
  └─ MemoryStore.record() → EpisodeLog (append-only JSONL)
 ```
 
diff --git a/docs/CODEMAPS/moltbook-agent.md b/docs/CODEMAPS/moltbook-agent.md
index 3d04c35..c6790b2 100644
--- a/docs/CODEMAPS/moltbook-agent.md
+++ b/docs/CODEMAPS/moltbook-agent.md
@@ -247,7 +247,7 @@ In `config/prompts/*.md`, lazy-loaded via `core/prompts.py`:
 | `logs/audit.jsonl` | JSONL | `MOLTBOOK_HOME` | Approval history + source_ids + epistemic_counts (ADR-0020/0050) |
 | `logs/skill-usage-*.jsonl` | JSONL | `MOLTBOOK_HOME` | Historic skill log (ADR-0023 sunset ADR-0036; no new files; observation evidence only) |
 | `logs/llm-calls-*.jsonl` | JSONL (0600) | `MOLTBOOK_HOME` | Per-call LLM telemetry: caller/model/tokens/duration/outcome/`think` (trace requested) + `thinking_source` (which channel delivered it, or `absent`) + sparse `thinking_fallback_reason` (why it did not, ADR-0068 amendment); never prompt bodies or trace content (sha256 prefix only) |
-| `logs/verification-audit.jsonl` | JSONL (0600) | `MOLTBOOK_HOME` | Verification challenge corpus/outcome log: `challenge_b64`, `challenge_sha256`, hashed code, answer, solver_path, verify_success |
+| `logs/verification-audit.jsonl` | JSONL (0600) | `MOLTBOOK_HOME` | Verification challenge corpus/outcome log: `challenge_b64`, `challenge_sha256`, hashed code, answer, solver_path, verify_success, plus the create-time columns `action` (`comment`/`reply`/`post`, `None` off the create path), `target_sha256` (digest only, ADR-0083) and `content_recorded` — so an orphaned publish (created on-platform, deliberately unrecorded) is countable per kind |
 
 ## Security Boundaries
 
diff --git a/src/contemplative_agent/adapters/moltbook/agent.py b/src/contemplative_agent/adapters/moltbook/agent.py
index 924a790..12adb2f 100644
--- a/src/contemplative_agent/adapters/moltbook/agent.py
+++ b/src/contemplative_agent/adapters/moltbook/agent.py
@@ -41,6 +41,8 @@ from .reply_handler import ReplyHandler
 from .session_context import SessionContext
 from .submolt_scope import SubmoltScopeScan, scan_submolt_scope
 from .verification import (
+    VerificationAction,
+    VerificationSolveResult,
     VerificationTracker,
     _sanitize_audit_error,
     record_verification_audit,
@@ -492,7 +494,13 @@ class Agent:
     # Verification
     # ------------------------------------------------------------------
 
-    def _handle_verification(self, verification: dict) -> bool:
+    def _handle_verification(
+        self,
+        verification: dict,
+        *,
+        action: VerificationAction | None = None,
+        target_id: str | None = None,
+    ) -> bool:
         """Solve and submit a content-verification challenge.
 
         ``verification`` is the object Moltbook embeds in a create-response
@@ -502,6 +510,15 @@ class Agent:
         when the content was verified (or no action was needed); False when
         solving or submission failed (caller leaves the content unrecorded).
 
+        ``action`` / ``target_id`` are threaded by ``passes_verification``'s
+        call sites so every audit record written here states which create kind
+        it gated and (as a digest) which target — the countable trace of an
+        orphaned publish (weekly F1.2 2026-08-08). ``content_recorded`` in
+        those records mirrors this method's return value, because that IS the
+        caller contract: True ⇒ the caller records the body, False ⇒ it
+        deliberately records nothing. Left None when ``action`` is None (a
+        non-create-time invocation has no body at stake).
+
         Deliberately NOT routed through _confirm_side_effect (audit H1):
         verification is a platform anti-bot handshake required for created
         content to become visible, not a social action — gating it would
@@ -520,6 +537,23 @@ class Agent:
         challenge_text = raw_challenge if isinstance(raw_challenge, str) else ""
         verification_code = raw_code if isinstance(raw_code, str) else ""
 
+        def _audit(
+            solve_result: VerificationSolveResult,
+            *,
+            verify_success: bool,
+            error: str | None = None,
+        ) -> None:
+            record_verification_audit(
+                challenge_text=challenge_text,
+                verification_code=verification_code,
+                solve_result=solve_result,
+                verify_success=verify_success,
+                error=error,
+                action=action,
+                target_id=target_id,
+                content_recorded=verify_success if action is not None else None,
+            )
+
         if not challenge_text or not verification_code:
             # Key names are server-controlled — sanitize before they touch
             # the plain application log or the audit error field.
@@ -532,10 +566,8 @@ class Agent:
             # (and can auto-stop the session), so without a record a
             # server-side shape change would be indistinguishable in
             # verification-audit.jsonl from verification not happening at all.
-            record_verification_audit(
-                challenge_text=challenge_text,
-                verification_code=verification_code,
-                solve_result=unsolved_result(challenge_text),
+            _audit(
+                unsolved_result(challenge_text),
                 verify_success=False,
                 error="malformed_verification_object keys=" + keys_repr,
             )
@@ -545,10 +577,8 @@ class Agent:
         solve_result = solve_challenge_result(challenge_text)
         answer = solve_result.answer
         if answer is None:
-            record_verification_audit(
-                challenge_text=challenge_text,
-                verification_code=verification_code,
-                solve_result=solve_result,
+            _audit(
+                solve_result,
                 verify_success=False,
                 error=solve_result.abstain_reason or "solve_failed",
             )
@@ -559,12 +589,7 @@ class Agent:
         try:
             result = submit_verification(client, verification_code, answer)
             if result.get("success"):
-                record_verification_audit(
-                    challenge_text=challenge_text,
-                    verification_code=verification_code,
-                    solve_result=solve_result,
-                    verify_success=True,
-                )
+                _audit(solve_result, verify_success=True)
                 self._verification.record_success()
                 logger.info("Verification submitted and accepted")
                 return True
@@ -572,24 +597,12 @@ class Agent:
             # injection in agent-launchd.log (same care as client.py).
             safe_error = _sanitize_audit_error(str(result.get("error", "")))
             logger.warning("Verification rejected: %s", safe_error)
-            record_verification_audit(
-                challenge_text=challenge_text,
-                verification_code=verification_code,
-                solve_result=solve_result,
-                verify_success=False,
-                error=safe_error or "verify_rejected",
-            )
+            _audit(solve_result, verify_success=False, error=safe_error or "verify_rejected")
             self._verification.record_failure()
             return False
         except (MoltbookClientError, ValueError) as exc:
             logger.error("Verification submission failed: %s", exc)
-            record_verification_audit(
-                challenge_text=challenge_text,
-                verification_code=verification_code,
-                solve_result=solve_result,
-                verify_success=False,
-                error=str(exc),
-            )
+            _audit(solve_result, verify_success=False, error=str(exc))
             self._verification.record_failure()
             return False
 
diff --git a/src/contemplative_agent/adapters/moltbook/feed_manager.py b/src/contemplative_agent/adapters/moltbook/feed_manager.py
index b589f54..23197dc 100644
--- a/src/contemplative_agent/adapters/moltbook/feed_manager.py
+++ b/src/contemplative_agent/adapters/moltbook/feed_manager.py
@@ -22,7 +22,13 @@ from .config import (
 from .content import ContentManager
 from .dedup import is_promotional, is_repeat_target_for_author
 from .llm_functions import generate_internal_note, score_relevance
-from .publish import client_error_guard, log_published, passes_verification, verification_of
+from .publish import (
+    VerificationHandler,
+    client_error_guard,
+    log_published,
+    passes_verification,
+    verification_of,
+)
 from .session_context import SessionContext
 
 logger = logging.getLogger(__name__)
@@ -45,7 +51,7 @@ class FeedManager:
         get_content: Callable[[], ContentManager],
         confirm_action: Callable[[str, str], bool],
         confirm_side_effect: Callable[[str], bool],
-        handle_verification: Callable[[dict], bool],
+        handle_verification: VerificationHandler,
     ) -> None:
         self._ctx = ctx
         self._domain = domain
@@ -455,6 +461,8 @@ class FeedManager:
                 verification_of(created),
                 self._handle_verification,
                 description=f"Comment on {post_id[:12]}",
+                action="comment",
+                target_id=post_id,
             ):
                 return False
             # Record the dedup hash only now that the comment is actually posted
diff --git a/src/contemplative_agent/adapters/moltbook/post_pipeline.py b/src/contemplative_agent/adapters/moltbook/post_pipeline.py
index 1d6ca9a..451f97e 100644
--- a/src/contemplative_agent/adapters/moltbook/post_pipeline.py
+++ b/src/contemplative_agent/adapters/moltbook/post_pipeline.py
@@ -27,7 +27,7 @@ from .llm_functions import (
     summarize_post_topic,
 )
 from .novelty import NoveltyGate
-from .publish import client_error_guard, log_published, passes_verification
+from .publish import VerificationHandler, client_error_guard, log_published, passes_verification
 from .session_context import SessionContext
 
 logger = logging.getLogger(__name__)
@@ -88,7 +88,7 @@ class PostPipeline:
         get_feed: Callable[[], list[dict]],
         confirm_action: Callable[..., bool],
         novelty_gate: NoveltyGate,
-        handle_verification: Callable[[dict], bool],
+        handle_verification: VerificationHandler,
     ) -> None:
         self._ctx = ctx
         self._domain = domain
@@ -422,6 +422,8 @@ class PostPipeline:
                 post_data.get("verification"),
                 self._handle_verification,
                 description=f"Post (id={post_id})",
+                action="post",
+                target_id=post_id,
             ):
                 return
             # Past this gate the post is provably created AND visible, so record
diff --git a/src/contemplative_agent/adapters/moltbook/publish.py b/src/contemplative_agent/adapters/moltbook/publish.py
index 5ddec55..326afaa 100644
--- a/src/contemplative_agent/adapters/moltbook/publish.py
+++ b/src/contemplative_agent/adapters/moltbook/publish.py
@@ -20,14 +20,33 @@ from __future__ import annotations
 import logging
 from collections.abc import Callable, Iterator
 from contextlib import contextmanager
-from typing import Any
+from typing import Any, Protocol
 
 from ...core.text_utils import log_preview
 from .client import MoltbookClientError
+from .verification import VerificationAction
 
 logger = logging.getLogger(__name__)
 
 
+class VerificationHandler(Protocol):
+    """The create-time handshake callback (``agent._handle_verification``).
+
+    ``action`` / ``target_id`` identify what the handshake gates so the
+    verification audit record can carry them as data (weekly F1.2 2026-08-08)
+    instead of the caller dropping them into a log format string the sweep
+    normalizes into uncountability.
+    """
+
+    def __call__(
+        self,
+        verification: dict,
+        *,
+        action: VerificationAction | None = None,
+        target_id: str | None = None,
+    ) -> bool: ...
+
+
 @contextmanager
 def client_error_guard(action: str, *, on_rate_limited: Callable[[], None]) -> Iterator[None]:
     """Swallow a ``MoltbookClientError`` from one outward write.
@@ -47,9 +66,11 @@ def client_error_guard(action: str, *, on_rate_limited: Callable[[], None]) -> I
 
 def passes_verification(
     verification: Any,
-    handle_verification: Callable[[dict], bool],
+    handle_verification: VerificationHandler,
     *,
     description: str,
+    action: VerificationAction,
+    target_id: str,
 ) -> bool:
     """Solve the create-response challenge, if the response carries one.
 
@@ -60,11 +81,17 @@ def passes_verification(
     write instead silences the agent: it dedups future attempts against content
     nobody ever saw.
 
+    ``action`` and ``target_id`` are required precisely because a failure
+    records nothing: the audit record is then the ONLY countable trace that a
+    published body was orphaned, and it needs the create kind and a joinable
+    target digest to say so (weekly F1.2 2026-08-08). The WARNING below stays
+    as the human-readable trace; it is no longer the only one.
+
     A trusted-bypass response carries no ``verification`` key and passes.
     """
     if verification is None:
         return True
-    if handle_verification(verification):
+    if handle_verification(verification, action=action, target_id=target_id):
         return True
     logger.warning("%s created but verification failed; not recording", description)
     return False
diff --git a/src/contemplative_agent/adapters/moltbook/reply_handler.py b/src/contemplative_agent/adapters/moltbook/reply_handler.py
index 6cce1d2..76c67bd 100644
--- a/src/contemplative_agent/adapters/moltbook/reply_handler.py
+++ b/src/contemplative_agent/adapters/moltbook/reply_handler.py
@@ -14,7 +14,13 @@ from .client import MoltbookClient
 from .config import ADAPTIVE_BACKOFF
 from .dedup import is_promotional
 from .llm_functions import generate_internal_note, generate_reply
-from .publish import client_error_guard, log_published, passes_verification, verification_of
+from .publish import (
+    VerificationHandler,
+    client_error_guard,
+    log_published,
+    passes_verification,
+    verification_of,
+)
 from .session_context import SessionContext
 
 logger = logging.getLogger(__name__)
@@ -88,7 +94,7 @@ class ReplyHandler:
         ctx: SessionContext,
         confirm_action: Callable[[str, str], bool],
         confirm_side_effect: Callable[[str], bool],
-        handle_verification: Callable[[dict], bool],
+        handle_verification: VerificationHandler,
     ) -> None:
         self._ctx = ctx
         self._confirm_action = confirm_action
@@ -309,6 +315,8 @@ class ReplyHandler:
                 verification_of(created),
                 self._handle_verification,
                 description=f"Reply on {post_id[:12]}",
+                action="reply",
+                target_id=post_id,
             ):
                 return
             ctx.commented_posts.add(reply_key)
diff --git a/src/contemplative_agent/adapters/moltbook/verification.py b/src/contemplative_agent/adapters/moltbook/verification.py
index b4f1dd9..3ae617b 100644
--- a/src/contemplative_agent/adapters/moltbook/verification.py
+++ b/src/contemplative_agent/adapters/moltbook/verification.py
@@ -112,6 +112,15 @@ _NUMBER_PATTERN = r"-?\d+(?:\.\d+)?"
 _EXPR_PATTERN = re.compile(rf"\(?\s*({_NUMBER_PATTERN})\s*([+*/xX-])\s*({_NUMBER_PATTERN})\s*\)?")
 VERIFICATION_AUDIT_PATH = EPISODE_LOG_DIR / "verification-audit.jsonl"
 
+# Kinds of create-time handshake (weekly F1.2 2026-08-08). The audit record's
+# `action` column carries one of these when the record comes from a
+# create-response handshake, threaded explicitly from the publish call sites —
+# never parsed back out of a log/description string. None means "not from a
+# create-time handshake" (a direct solve, or any row written before the field
+# existed), so a longitudinal reading must treat None as unknown, not as
+# "no content at stake".
+VerificationAction = Literal["comment", "reply", "post"]
+
 
 @dataclass(frozen=True)
 class VerificationSolveResult:
@@ -369,12 +378,25 @@ def record_verification_audit(
     solve_result: VerificationSolveResult,
     verify_success: bool,
     error: str | None = None,
+    action: VerificationAction | None = None,
+    target_id: str | None = None,
+    content_recorded: bool | None = None,
 ) -> None:
     """Append a best-effort verification corpus/audit record.
 
     The raw challenge is stored as base64, not free text, so direct log reads do
     not become a prompt-injection path. Decode it only inside an explicit
     untrusted-content evaluation harness.
+
+    ``action`` / ``target_id`` / ``content_recorded`` (weekly F1.2 2026-08-08)
+    make an orphaned publish countable: a create-time handshake failure leaves
+    a body visible on-platform that the agent deliberately does not record
+    (``publish.passes_verification``), and before these fields its only trace
+    was a WARNING line the log sweep normalizes into uncountability. The weekly
+    report's recorded-bodies denominator can now be reconciled exactly instead
+    of stated as a floor. ``target_id`` is stored as ``target_sha256`` only —
+    the count and the joinability are needed, never the raw identifier
+    (ADR-0083 boundary discipline).
     """
     try:
         record = _verification_audit_record(
@@ -383,12 +405,24 @@ def record_verification_audit(
             solve_result=solve_result,
             verify_success=verify_success,
             error=error,
+            action=action,
+            target_id=target_id,
+            content_recorded=content_recorded,
         )
         append_jsonl_restricted(VERIFICATION_AUDIT_PATH, record)
     except Exception as exc:
         # WARNING (not debug): a persistently broken audit writer must be
         # visible at default log levels (observability sweep 2026-07-10).
-        logger.warning("Verification audit record failed: %s", exc)
+        #
+        # The action kind rides the message (F-VER-8, weekly F1.2): when the
+        # audit write is what failed, this line is the only remaining trace of
+        # a create-time handshake, and a trace that cannot say WHICH kind of
+        # body was at stake is indistinguishable from a handshake that never
+        # happened — the exact uncountability this change exists to close. The
+        # kind is a closed vocabulary of our own literals, never server text,
+        # so it is safe to log unsanitized (the digest and the raw target id
+        # both stay out).
+        logger.warning("Verification audit record failed (action=%s): %s", action or "none", exc)
 
 
 def unsolved_result(challenge_text: str) -> VerificationSolveResult:
@@ -414,6 +448,9 @@ def _verification_audit_record(
     solve_result: VerificationSolveResult,
     verify_success: bool,
     error: str | None,
+    action: VerificationAction | None = None,
+    target_id: str | None = None,
+    content_recorded: bool | None = None,
 ) -> dict[str, Any]:
     record: dict[str, Any] = {
         "ts": now_iso("seconds"),
@@ -431,6 +468,14 @@ def _verification_audit_record(
         "solver_path": solve_result.solver_path,
         "solve_success": solve_result.answer is not None,
         "verify_success": verify_success,
+        # Which create kind this handshake gated (None: not create-time), the
+        # target as a digest ONLY (ADR-0083 — joinable, never identifying),
+        # and whether the caller went on to record the body. Together these
+        # turn "≥N orphaned publishes" from a log-sweep floor into an exact,
+        # per-kind count (weekly F1.2 2026-08-08).
+        "action": action,
+        "target_sha256": _sha256_text(target_id) if target_id else None,
+        "content_recorded": content_recorded,
         "error": _sanitize_audit_error(error) if error else None,
     }
     return record
diff --git a/tests/test_agent.py b/tests/test_agent.py
index 1fb4acc..0b95cda 100644
--- a/tests/test_agent.py
+++ b/tests/test_agent.py
@@ -3660,6 +3660,205 @@ class TestHandleVerificationMalformedObject:
         assert "malformed_verification_object" in kwargs["error"]
 
 
+class TestVerificationAuditActionThreading:
+    """Weekly F1.2 2026-08-08: each create-time handshake states its own kind.
+
+    The three ``passes_verification`` call sites already held the create kind
+    and the target id — and dropped both into a ``description`` format string
+    that only ever reached a WARNING line, which the log sweep lowercases,
+    squashes and truncates. So ``verification-audit.jsonl`` could answer "how
+    many handshakes failed" but not "how many published bodies were orphaned,
+    of which kind", and the weekly report had to state its denominator as a
+    floor.
+
+    These pin the threading at the real call sites (not at
+    ``passes_verification`` in isolation): the defect was in what the callers
+    passed, so a test that supplies the arguments itself would have stayed
+    green. ``content_recorded`` mirrors the handshake verdict because that IS
+    the caller contract — pass ⇒ the body is recorded, fail ⇒ nothing is.
+    """
+
+    @patch("contemplative_agent.adapters.moltbook.agent.record_verification_audit")
+    @patch(
+        "contemplative_agent.adapters.moltbook.agent.solve_challenge_result",
+        return_value=_solve_result(None),
+    )
+    @patch("contemplative_agent.adapters.moltbook.feed_manager.score_relevance", return_value=0.95)
+    def test_orphaned_comment_is_countable_by_kind(
+        self, mock_score, mock_solve, mock_audit, tmp_path
+    ):
+        """The defect's own scenario: a comment created on-platform whose
+        handshake failed, which the agent deliberately does not record."""
+        content = MagicMock()
+        agent, client, scheduler = _make_agent(tmp_path, content=content)
+        content.create_comment.return_value = GenerationOutput(text="Great insight")
+        client.post_comment.return_value = {
+            "id": "c-new",
+            "verification": {
+                "verification_code": "moltbook_verify_x",
+                "challenge_text": "noise",
+            },
+        }
+
+        result = agent._feed_manager.engage_with_post(
+            {"content": "text", "id": "post1"}, client, scheduler
+        )
+
+        assert result is False
+        kwargs = mock_audit.call_args.kwargs
+        assert kwargs["action"] == "comment"
+        assert kwargs["target_id"] == "post1"
+        # The body is visible on-platform but unrecorded — the whole point of
+        # the column: this row is the orphan's only countable trace.
+        assert kwargs["content_recorded"] is False
+
+    @patch("contemplative_agent.adapters.moltbook.agent.record_verification_audit")
+    @patch(
+        "contemplative_agent.adapters.moltbook.agent.submit_verification",
+        return_value={"success": True},
+    )
+    @patch(
+        "contemplative_agent.adapters.moltbook.agent.solve_challenge_result",
+        return_value=_solve_result("15.00"),
+    )
+    @patch("contemplative_agent.adapters.moltbook.feed_manager.time")
+    @patch("contemplative_agent.adapters.moltbook.feed_manager.random")
+    @patch("contemplative_agent.adapters.moltbook.feed_manager.score_relevance", return_value=0.95)
+    def test_verified_comment_is_marked_recorded(
+        self, mock_score, mock_random, mock_time, mock_solve, mock_submit, mock_audit, tmp_path
+    ):
+        """The other half of the split the report needs: a handshake failure
+        that cost a visible body vs one that did not."""
+        mock_random.uniform.return_value = 60.0
+        content = MagicMock()
+        agent, client, scheduler = _make_agent(tmp_path, content=content)
+        content.create_comment.return_value = GenerationOutput(text="Great insight")
+        client.post_comment.return_value = {
+            "id": "c-new",
+            "verification": {
+                "verification_code": "moltbook_verify_x",
+                "challenge_text": "noise",
+            },
+        }
+
+        result = agent._feed_manager.engage_with_post(
+            {"content": "text", "id": "post1"}, client, scheduler
+        )
+
+        assert result is True
+        kwargs = mock_audit.call_args.kwargs
+        assert kwargs["action"] == "comment"
+        assert kwargs["content_recorded"] is True
+
+    @patch("contemplative_agent.adapters.moltbook.agent.record_verification_audit")
+    @patch(
+        "contemplative_agent.adapters.moltbook.agent.solve_challenge_result",
+        return_value=_solve_result(None),
+    )
+    @patch(
+        "contemplative_agent.adapters.moltbook.reply_handler.generate_reply",
+        return_value=GenerationOutput(text="My reply"),
+    )
+    def test_orphaned_reply_is_countable_by_kind(
+        self, mock_reply, mock_solve, mock_audit, tmp_path
+    ):
+        agent, client, scheduler = _make_agent(tmp_path)
+        client.get_notifications.return_value = [
+            {
+                "type": "comment",
+                "id": "n1",
+                "post_id": "p1",
+                "content": "Nice post!",
+                "post_content": "Original content",
+                "agent_id": "a1",
+                "agent_name": "Alice",
+            }
+        ]
+        client.get_post_comments.return_value = []
+        client.post_comment.return_value = {
+            "id": "c-new",
+            "verification": {
+                "verification_code": "moltbook_verify_x",
+                "challenge_text": "noise",
+            },
+        }
+
+        agent._reply_handler.run_cycle(client, scheduler, time.time() + 3600)
+
+        kwargs = mock_audit.call_args.kwargs
+        assert kwargs["action"] == "reply"
+        assert kwargs["target_id"] == "p1"
+        assert kwargs["content_recorded"] is False
+        # The reply is genuinely unrecorded — otherwise "orphaned" would be
+        # the wrong word for what the column counts.
+        assert not any("Replied to" in a for a in agent._ctx.actions_taken)
+
+    @patch("contemplative_agent.adapters.moltbook.agent.record_verification_audit")
+    @patch(
+        "contemplative_agent.adapters.moltbook.agent.solve_challenge_result",
+        return_value=_solve_result(None),
+    )
+    @patch(
+        "contemplative_agent.adapters.moltbook.post_pipeline.select_submolt",
+        return_value="philosophy",
+    )
+    @patch(
+        "contemplative_agent.adapters.moltbook.post_pipeline.summarize_post_topic",
+        return_value="reflection on alignment",
+    )
+    @patch(
+        "contemplative_agent.adapters.moltbook.post_pipeline.generate_post_title",
+        return_value="Notes on dedup gates",
+    )
+    @patch(
+        "contemplative_agent.adapters.moltbook.post_pipeline._score_post_relevance",
+        return_value=0.8,
+    )
+    def test_orphaned_post_is_countable_by_kind(
+        self,
+        mock_score,
+        mock_title,
+        mock_summarize,
+        mock_submolt,
+        mock_solve,
+        mock_audit,
+        tmp_path,
+    ):
+        content = MagicMock()
+        gate = _RecordingNoveltyGate()
+        agent, client, scheduler = _make_agent(tmp_path, content=content, novelty_gate=gate)
+        content.create_cooperation_post.return_value = GenerationOutput(
+            text="We paused to revisit how gates intersect with memory."
+        )
+
+        feed_resp = MagicMock()
+        feed_resp.json.return_value = {
+            "posts": [{"title": "t", "content": "c", "id": "p1", "submolt_name": "philosophy"}]
+        }
+        post_resp = MagicMock()
+        post_resp.json.return_value = {
+            "success": True,
+            "post": {
+                "id": "new-post-123",
+                "verification_status": "pending",
+                "verification": {
+                    "verification_code": "moltbook_verify_x",
+                    "challenge_text": "noise",
+                },
+            },
+        }
+        client.get.return_value = feed_resp
+        client.post.return_value = post_resp
+
+        agent._post_pipeline.run_cycle(client, scheduler)
+
+        kwargs = mock_audit.call_args.kwargs
+        assert kwargs["action"] == "post"
+        assert kwargs["target_id"] == "new-post-123"
+        assert kwargs["content_recorded"] is False
+        assert gate.recorded == []
+
+
 class TestPostPipelineSelectionOrdering:
     """ADR-0081 Decision 2 regression pin: post_title reuses the
     cooperation_post pass's skill selection via a module-level hand-off in
diff --git a/tests/test_verification.py b/tests/test_verification.py
index e8ec452..12692e3 100644
--- a/tests/test_verification.py
+++ b/tests/test_verification.py
@@ -1625,6 +1625,88 @@ class TestVerificationAudit:
         assert record["error"] == "baderror"
 
 
+class TestVerificationAuditActionColumns:
+    """Weekly F1.2 2026-08-08: an orphaned publish must be countable.
+
+    A create-time handshake failure leaves a body visible on-platform that the
+    agent deliberately does not record (``publish.passes_verification``). Before
+    these columns its only trace was a WARNING line the log sweep lowercases,
+    squashes and truncates — so the weekly report could state its recorded-body
+    denominator only as a floor ("at least 15"). ``action`` /
+    ``target_sha256`` / ``content_recorded`` make the same event countable and
+    joinable per kind.
+    """
+
+    _SOLVED = VerificationSolveResult(
+        answer="15.00",
+        solver_path="code_parse",
+        challenge_sha256="challenge-sha",
+    )
+
+    def test_action_and_recorded_flag_land_in_the_record(self):
+        record = _verification_audit_record(
+            challenge_text="noise",
+            verification_code="moltbook_verify_v1",
+            solve_result=self._SOLVED,
+            verify_success=False,
+            error="verify_rejected",
+            action="comment",
+            target_id="post-abc123",
+            content_recorded=False,
+        )
+
+        assert record["action"] == "comment"
+        assert record["content_recorded"] is False
+        assert record["target_sha256"] == _sha256_text("post-abc123")
+
+    def test_raw_target_id_never_reaches_the_record(self):
+        # ADR-0083 output-boundary discipline: the count and the joinability
+        # are what is needed, not the identifier.
+        record = _verification_audit_record(
+            challenge_text="noise",
+            verification_code="moltbook_verify_v1",
+            solve_result=self._SOLVED,
+            verify_success=True,
+            error=None,
+            action="post",
+            target_id="post-abc123",
+            content_recorded=True,
+        )
+
+        assert "post-abc123" not in json.dumps(record)
+
+    def test_non_create_time_handshake_leaves_the_columns_none(self):
+        # Dense field, sparse meaning: None reads as "not a create-time
+        # handshake / unknown", never as "no body was at stake". Records
+        # written before this change carry None for the same reason, so a
+        # longitudinal count must not read None as a zero.
+        record = _verification_audit_record(
+            challenge_text="noise",
+            verification_code="moltbook_verify_v1",
+            solve_result=self._SOLVED,
+            verify_success=True,
+            error=None,
+        )
+
+        assert record["action"] is None
+        assert record["target_sha256"] is None
+        assert record["content_recorded"] is None
+
+    def test_columns_are_always_present_so_a_reader_can_count(self):
+        # Dense (always-emitted) rather than conditionally added: a reading
+        # that must divide "orphaned" by "all handshakes" cannot tell a missing
+        # key from an absent value.
+        record = _verification_audit_record(
+            challenge_text="noise",
+            verification_code="moltbook_verify_v1",
+            solve_result=self._SOLVED,
+            verify_success=True,
+            error=None,
+        )
+
+        assert {"action", "target_sha256", "content_recorded"} <= record.keys()
+
+
 class TestRejectedAnswerSuppression:
     """Round 8: a previously server-rejected answer is never resubmitted.
 
diff --git a/tests/test_verification_chaos.py b/tests/test_verification_chaos.py
index b09b665..00e3738 100644
--- a/tests/test_verification_chaos.py
+++ b/tests/test_verification_chaos.py
@@ -27,6 +27,13 @@ Fault catalog rows exercised here:
 - F-VER-6 challenge-injected obedience (bare number, no EXPR)
                                              -> fails closed, nothing submitted
 - F-VER-7 corrupt rejected-answer audit log  -> fails open, solve still runs
+- F-VER-8 the audit WRITE fails               -> swallowed, but the remaining
+                                                warning still names the create
+                                                kind (weekly F1.2 2026-08-08:
+                                                the audit row is now the only
+                                                countable trace of an orphaned
+                                                publish, so losing it silently
+                                                re-opens the hole)
 
 TDD contract (ADR-0062 twelfth amendment): F-VER-1 asserts a reason code that
 did not exist when this file was written. The solver folded "the LLM
@@ -44,6 +51,7 @@ shapes pinned as ``@example``, and no test sleeps.
 from __future__ import annotations
 
 import json
+import logging
 import time
 
 import pytest
@@ -56,8 +64,10 @@ from contemplative_agent.adapters.moltbook.verification import (
     _ABSTAIN_REASON_LLM_NONE,
     _EXTRACT_NUM_PREDICT,
     _sha256_text,
+    record_verification_audit,
     solve_challenge,
     solve_challenge_result,
+    unsolved_result,
 )
 from contemplative_agent.core.llm import configure, generate, reset_llm_config
 from tests.chaos import (
@@ -515,6 +525,99 @@ class TestCorruptRejectedLogFVer7:
         assert result.answer == OK_ANSWER
 
 
+class TestUnwritableAuditLogFVer8:
+    """F-VER-8: the audit write itself is the thing that failed.
+
+    Weekly F1.2 2026-08-08 makes the audit row the countable trace of an
+    orphaned publish — a body created on-platform that the agent deliberately
+    does not record. That promotes the writer's own failure into a fault worth
+    a catalog row: if the row is lost, the remaining log line must still say
+    WHICH create kind was at stake, or a lost orphan is indistinguishable from
+    a handshake that never happened (precisely the uncountability this change
+    exists to close).
+
+    The write is best-effort by design — a full disk must not abort a session
+    mid-publish — so the desired behaviour is "swallow, but say enough".
+    """
+
+    def _failing_writer(self, monkeypatch):
+        from contemplative_agent.adapters.moltbook import verification as verification_mod
+
+        def _boom(path, record):
+            raise OSError("No space left on device")
+
+        monkeypatch.setattr(verification_mod, "append_jsonl_restricted", _boom)
+
+    def test_lost_audit_row_still_names_the_action(self, monkeypatch, caplog):
+        self._failing_writer(monkeypatch)
+
+        with caplog.at_level(logging.WARNING, logger="contemplative_agent"):
+            record_verification_audit(
+                challenge_text=NOISE_CHALLENGE,
+                verification_code="moltbook_verify_x",
+                solve_result=unsolved_result(NOISE_CHALLENGE),
+                verify_success=False,
+                error="solve_failed",
+                action="comment",
+                target_id="post1",
+                content_recorded=False,
+            )
+
+        messages = [r.getMessage() for r in caplog.records]
+        assert any("action=comment" in m for m in messages), messages
+
+    def test_lost_audit_row_for_a_non_create_handshake_says_so(self, monkeypatch, caplog):
+        # The counterpart reading: no body was at stake here, and the line must
+        # not imply one was.
+        self._failing_writer(monkeypatch)
+
+        with caplog.at_level(logging.WARNING, logger="contemplative_agent"):
+            record_verification_audit(
+                challenge_text=NOISE_CHALLENGE,
+                verification_code="moltbook_verify_x",
+                solve_result=unsolved_result(NOISE_CHALLENGE),
+                verify_success=False,
+                error="solve_failed",
+            )
+
+        messages = [r.getMessage() for r in caplog.records]
+        assert any("action=none" in m for m in messages), messages
+
+    def test_write_failure_never_escapes_to_the_publish_path(self, monkeypatch):
+        # Best-effort by contract: the handshake's own verdict, not the audit
+        # writer's, decides whether a body gets recorded.
+        self._failing_writer(monkeypatch)
+
+        record_verification_audit(
+            challenge_text=NOISE_CHALLENGE,
+            verification_code="moltbook_verify_x",
+            solve_result=unsolved_result(NOISE_CHALLENGE),
+            verify_success=False,
+            action="post",
+            target_id="new-post-123",
+            content_recorded=False,
+        )
+
+    def test_raw_target_id_stays_out_of_the_failure_line(self, monkeypatch, caplog):
+        # ADR-0083: the digest is joinable, the identifier is not emitted —
+        # including on the degraded path, which is the easy place to leak it.
+        self._failing_writer(monkeypatch)
+
+        with caplog.at_level(logging.WARNING, logger="contemplative_agent"):
+            record_verification_audit(
+                challenge_text=NOISE_CHALLENGE,
+                verification_code="moltbook_verify_x",
+                solve_result=unsolved_result(NOISE_CHALLENGE),
+                verify_success=False,
+                action="reply",
+                target_id="post-abc123",
+                content_recorded=False,
+            )
+
+        for record in caplog.records:
+            assert "post-abc123" not in record.getMessage()
+
+
 class TestSolverNeverEscapesItsVocabulary:
     """Property: whatever the backend does, the solver lands in a known state.
````

### F1.4.patch

````diff
diff --git a/config/prompts/weekly-analysis.md b/config/prompts/weekly-analysis.md
index ee6e613..ec03b51 100644
--- a/config/prompts/weekly-analysis.md
+++ b/config/prompts/weekly-analysis.md
@@ -33,6 +33,7 @@ Summarize changes to the agent's internal state during this period:
 - **Rules**: List all rules at period end. Note any added/removed/modified.
 - **Knowledge**: Pattern count at start vs end. Carry the source label the input gives you — the state diff reports *committed snapshots of the data repo* (with commit sha and date), the invariant check reports the *live store at report-generation time* (whose `total` includes tombstones). These answer different questions and legitimately differ; report each with its label rather than treating the gap as a contradiction or picking one as canonical.
 - **Operational drift** (from the provided *Log Anomaly Sweep* and *State Invariant Check*): surface any anomaly type flagged 🆕 (new since last sweep) or sharply spiking (high Δ), and any invariant at ⚠️ WARN or ❌ FAIL. These are deterministic signals — report them as observations (what changed, how much); proposing fixes belongs to the downstream diagnosis step, not this report.
+- **Skill selection** (from the provided *Skill-selection shadow reading*): which skills pass-1 actually selected this week — selection frequency, verdict distribution (judged vs fail-open), hallucination rate, never-selected tail. This is the measured middle link between *installed* (state diff) and *vocabulary in output* (E): when A or E attributes output vocabulary to a skill, check the attribution against this list instead of inferring selection from vocabulary. Report the reading as observations; it carries names and counts only.
 
 If state diffs are provided, analyze them. If not, note "no state data available."
 
@@ -103,7 +104,8 @@ The following data will be provided:
 3. **Agent state diffs** (identity, constitution, skills, rules, knowledge count) — if available
 4. **Log Anomaly Sweep** — deterministic ranking of log anomalies by novelty (🆕 = new since last sweep) then frequency delta; read it for B's operational-drift note
 5. **State Invariant Check** — deterministic ✅/⚠️/❌ checks over knowledge.json / agents.json; read it for B's operational-drift note
-6. **Previous reports** (last 3 weeks if available) — for trend comparison
+6. **Skill-selection shadow reading** — deterministic aggregate of the pass-1 skill-selection log (selected skill names with frequency, verdict distribution, hallucination rate, never-selected tail; names and counts only); read it for B's skill-selection note
+7. **Previous reports** (last 3 weeks if available) — for trend comparison
 
 # Downstream
 
diff --git a/docs/CODEMAPS/architecture.md b/docs/CODEMAPS/architecture.md
index 5a94a62..2cca87e 100644
--- a/docs/CODEMAPS/architecture.md
+++ b/docs/CODEMAPS/architecture.md
@@ -1,4 +1,4 @@
-<!-- Generated: 2026-08-01 | Updated: 2026-08-07 (weekly-pipeline dead-code intake) | Files scanned: 74 | Token estimate: ~9179 -->
+<!-- Generated: 2026-08-01 | Updated: 2026-08-08 (weekly-analysis skill-selection intake) | Files scanned: 74 | Token estimate: ~9179 -->
 # Architecture
 
 ## Project Type
@@ -463,7 +463,7 @@ ViewRegistry.find_by_view("constitutional", get_live_patterns())
 
 ### weekly-analysis  [`scripts/weekly-analysis.sh`, ADR-0040]
 
-Runs outside the agent process (launchd → `claude -p`), assembling a prompt from operator-facing artifacts plus **four deterministic intakes**, then a diagnosis companion (`weekly-report-diagnosis` skill) produces the F sections.
+Runs outside the agent process (launchd → `claude -p`), assembling a prompt from operator-facing artifacts plus **five deterministic intakes**, then a diagnosis companion (`weekly-report-diagnosis` skill) produces the F sections.
 
 ```text
 collect: daily comment-reports + data-repo state diff + previous N reports
@@ -471,6 +471,8 @@ collect: daily comment-reports + data-repo state diff + previous N reports
        + state_invariant_check.py (accumulated state: knowledge.json / agents.json)
        + cross_day_duplicate_scan.py (published-body identity: episode logs → digests)
        + api_drift_scan.py       (platform schema drift: api-audit.jsonl keys; vocab state)
+       + skill-selection reading (pass-1 selection log: skill-selection-*.jsonl → names
+                                  and counts; package renderer via `uv run --no-sync`)
 generate: claude -p → weekly-<end>.md.tmp
 promote:  mv tmp → weekly-<end>.md        (atomic; a failure leaves the prior report)
        →  mv sweep-state.pending → sweep-state   (novelty baseline committed ONLY here)
@@ -478,13 +480,13 @@ promote:  mv tmp → weekly-<end>.md        (atomic; a failure leaves the prior
 translate: best-effort .ja.md (sonnet); failure never rolls back the promotes
 ```
 
-Order is load-bearing. The sweep's Δ / 🆕 columns and the drift scan's new/removed pairs are defined against their last committed snapshots, so both run `--no-update --emit-state` and their baselines are committed after the report lands — a run that produces nothing must spend nothing (findings F1.2; two consecutive weeks lost). The invariant check and the duplicate scan hold no state and are absolute readings, so they need no such ordering.
+Order is load-bearing. The sweep's Δ / 🆕 columns and the drift scan's new/removed pairs are defined against their last committed snapshots, so both run `--no-update --emit-state` and their baselines are committed after the report lands — a run that produces nothing must spend nothing (findings F1.2; two consecutive weeks lost). The invariant check, the duplicate scan and the skill-selection reading hold no state and are absolute readings, so they need no such ordering.
 
 The drift scan (2026-08-06) diffs the per-endpoint response-key vocabulary the client already records in `api-audit.jsonl` (2xx envelopes only, so outages do not read as schema changes) and tracks the `POST /verify` consecutive-failure run against the platform's 10-failure suspension rule. It exists because the platform ships API changes unannounced (observed: the `check_in` key appearing on `/home` in 2026-08, carrying role "standing instructions" — a third-party injection channel the adapter deliberately never consumes, gated by `tests/test_home_field_allowlist.py`). The spec (`skill.md`) is untrusted external text: it is never fetched in the unattended chain, and on drift the rendered section directs the re-read to the Saturday gate.
 
 The sweep's signature is keyed on **level + message**, with the dotted `%(name)s` module path dropped for lines in the runtime's own log format, and hex-shaped ids squashed to `#` alongside digit runs. The 80-character cap is the reason: the module path alone runs to ~47 characters, so keying on it spent the budget on the address and truncated the predicate — `"reply on <id> created but verification failed"` rendered as `"reply on <id> created"`, a failure displayed as its own opposite (findings F1.1). Excluding the name also makes the instrument refactor-invariant: a pure module move (`7c96e0f`) used to reset every affected signature to 🆕, i.e. the Δ / 🆕 columns measured the codebase rather than the runtime (findings F1.2). The trade is that the same message from two subsystems now merges into one row, so the logger name is carried as a display-only `Origin` column — it never enters the signature, the state file or the novelty computation, so the reader keeps the distinction the key deliberately drops.
 
-Injection boundary: the sweep, the invariant check and the drift scan must never read episode logs (the drift scan reads only the self-written `api-audit.jsonl`; the platform-controlled key names it renders are Markdown-escaped and length-capped). The duplicate scan does, and is the only intake permitted to — it emits **only** 12-hex SHA-256 digests, counts, filename-derived dates and the fixed `{post, reply, comment}` vocabulary (ADR-0083, gated by `TestOutputBoundary`). All four are observability: a failure degrades to a "not available" stub and never breaks the report.
+Injection boundary: the sweep, the invariant check and the drift scan must never read episode logs (the drift scan reads only the self-written `api-audit.jsonl`; the platform-controlled key names it renders are Markdown-escaped and length-capped). The duplicate scan does, and is the only intake permitted to — it emits **only** 12-hex SHA-256 digests, counts, filename-derived dates and the fixed `{post, reply, comment}` vocabulary (ADR-0083, gated by `TestOutputBoundary`). The skill-selection reading (2026-08-08, findings F1.4) reads the self-written `skill-selection-*.jsonl` shadow log (ADR-0076): the *selected* middle link between *installed* (state diff) and *vocabulary in output* (section E) was already logged per publish action but never supplied to the report. Its records embed the selection situation — untrusted post bodies — so the renderer (`format_skill_selection_report`, the same one behind `report --skill-selection`) emits skill names and counts only, never the situation strings (same ADR-0083 boundary; gated by `test_skill_selection_reading_reaches_the_prompt_names_only`). All five are observability: a failure degrades to a "not available" stub and never breaks the report.
 
 ### weekly-pipeline  [`scripts/weekly-pipeline.sh`, ADR-0085]
 
@@ -504,7 +506,7 @@ Stage 4 fix:       per code-scope F1: git worktree @ HEAD → claude -p (fix-imp
                    blocks export (inspector, not approver)
                    prompt-scope F1: draft diff only, no Verify, full text at the gate
 Stage 5 insight:   read-only recommendation pass over .staged/ (insight-recommendation.md)
-Stage 6 deadcode:  dead_code_scan.py (vulture over the repo checkout; 5th deterministic
+Stage 6 deadcode:  dead_code_scan.py (vulture over the repo checkout; 6th deterministic
                    intake — detection only, feeds the packet directly; runs before
                    improve so a recurring scan failure feeds the P4 detector)
 Stage 7 improve:   only when the same reason code recurred 2 consecutive runs (check-improvement)
diff --git a/scripts/weekly-analysis.sh b/scripts/weekly-analysis.sh
index fe82319..eb5367c 100755
--- a/scripts/weekly-analysis.sh
+++ b/scripts/weekly-analysis.sh
@@ -302,6 +302,59 @@ if [[ -d "$MOLTBOOK_HOME/logs" ]]; then
 fi
 [[ -z "$DUP_SCAN" ]] && DUP_SCAN="## Cross-Day Duplicate Scan"$'\n\n'"No duplicate scan available."
 
+# --- Skill-selection shadow reading (pass-1 selection intake, 2026-08-08) ---
+# Deterministic aggregate of logs/skill-selection-*.jsonl (the ADR-0076 shadow
+# writer). Without it the report sees *installed* (state diff) and *vocabulary
+# in output* (its own reading of E) and has to infer the middle link —
+# *selected* — from vocabulary matching, even though selection is already
+# logged per publish action. Renders names and counts only via the existing
+# `report --skill-selection` renderer (selection frequency, verdict
+# distribution, hallucination rate, never-selected tail); the situation
+# strings in the log are built from untrusted post bodies and never enter the
+# prompt (ADR-0083 boundary, held by the renderer). Read-only over the log and
+# skills dir — no selection behavior changes, so the open T-SKILLSEL window is
+# unaffected. Observability only — a failure must not break the weekly report.
+# `uv run --no-sync`, not bare python3: the renderer lives in the package
+# (venv-only imports), same invocation shape as weekly-pipeline.sh's
+# dead-code intake; launchd's plist PATH already covers uv (~/.local/bin).
+SKILL_SELECTION=""
+if [[ -d "$MOLTBOOK_HOME/logs" ]]; then
+    SKILL_SELECTION=$(uv run --project "$PROJECT_ROOT" --no-sync -q python - \
+        "$MOLTBOOK_HOME" "$START_DATE" <<'PY' 2>/dev/null || true
+import sys
+from datetime import date, datetime, timezone
+from pathlib import Path
+
+from contemplative_agent.core.skill_selection import (
+    format_skill_selection_report,
+    read_skill_selection_log,
+)
+
+home = Path(sys.argv[1])
+start = date.fromisoformat(sys.argv[2])
+# The reader windows by days-back-from-today (UTC); anchor the cutoff to the
+# report window's start date. For scheduled runs (end = yesterday) this is
+# exact; for backfill runs the window has no upper bound, which the rendered
+# "Window: last N days" line states honestly.
+days = max((datetime.now(timezone.utc).date() - start).days, 1)
+skills_dir = home / "skills"
+print(
+    format_skill_selection_report(
+        read_skill_selection_log(
+            home / "logs",
+            days=days,
+            skills_dir=skills_dir if skills_dir.is_dir() else None,
+        )
+    )
+)
+PY
+    )
+    if [[ -n "$SKILL_SELECTION" ]]; then
+        echo "Included skill-selection reading"
+    fi
+fi
+[[ -z "$SKILL_SELECTION" ]] && SKILL_SELECTION="## Skill-selection shadow reading (ADR-0076)"$'\n\n'"No skill-selection reading available."
+
 # --- Build prompt ---
 SYSTEM_PROMPT=$(cat "$PROMPT_TEMPLATE")
 
@@ -319,6 +372,8 @@ $INVARIANTS
 
 $DUP_SCAN
 
+$SKILL_SELECTION
+
 $PREV_REPORTS
 
 ## Daily Reports
diff --git a/tests/test_weekly_analysis_shell.py b/tests/test_weekly_analysis_shell.py
index 05564f3..557f80a 100644
--- a/tests/test_weekly_analysis_shell.py
+++ b/tests/test_weekly_analysis_shell.py
@@ -15,6 +15,7 @@ macOS-only: the script uses BSD ``date -j``.
 
 from __future__ import annotations
 
+import json
 import os
 import shutil
 import subprocess
@@ -209,6 +210,45 @@ class TestPromptAssembly:
         # The scan's boundary holds end to end: the body it hashed stays out.
         assert "a body" not in prompt.split("## Daily Reports")[0]
 
+    def test_skill_selection_reading_reaches_the_prompt_names_only(self, tmp_path):
+        """findings F1.4: the pass-1 selection log was the one instrument not in
+        the prompt — the report inferred *selected* from output vocabulary. The
+        intake must carry skill names and counts, and never the selection
+        situation strings, which are built from untrusted post bodies
+        (ADR-0083 boundary, held by the renderer)."""
+        home = _make_home(tmp_path)
+        record = {
+            "ts": f"{END_DATE}T10:00:00+00:00",
+            "verdict": "judged",
+            "selected": ["fabricated-benchmark-guard"],
+            "rejected_names": [],
+            "full_skill_tokens": 1000,
+            "would_be_skill_tokens": 100,
+            # The reader ignores fields it does not aggregate; a plaintext
+            # situation here proves the renderer emits names and counts only.
+            "prompt": "SITUATION-MARKER an untrusted post body",
+        }
+        (home / "logs" / f"skill-selection-{END_DATE}.jsonl").write_text(
+            json.dumps(record) + "\n", encoding="utf-8"
+        )
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
+        assert "## Skill-selection shadow reading" in prompt
+        assert "No skill-selection reading available" not in prompt
+        assert "fabricated-benchmark-guard: 1" in prompt
+        assert "SITUATION-MARKER" not in prompt
+
     def test_pattern_count_line_names_its_source_and_commits(self, tmp_path):
         """findings F1.4: the state diff's pattern counts are committed snapshots
         of the data repo, the invariant check's are the live store at generation
````

## 4. Insight staging review

```text
All 55 candidates rest on ≥3 source episodes (batch minimum 3, maximum 7 — `detecting-foundational-structural-compromise` and `tracing-conceptual-dependency-chains`), so provenance breadth passes universally and the discriminating axes are store duplication, intra-batch redundancy, and behavioral specificity. The dossier follows.

## 1. analyze-structural-boundaries — RECOMMEND: reject
Sibling group with `detecting-structural-tension-points` (#13), `detecting-linguistic-structural-boundaries` (#11), and `identifying-systemic-strain-points` (#29) — all say "study the tension/boundary itself, not the sides." Store skill `identifying-systemic-boundary-stressors` already covers pinpointing boundary tensions, and "treat the interface as the engine" gives no enactable step beyond that.

## 2. analyzing-narrative-mechanics — RECOMMEND: reject
Duplicates store `deconstructing-asserted-agency-into-mechanisms` and `deconstruct-foundational-claims-against-operative-components`: analyze how a claim is structured/constrained rather than accept its self-declaration. 5 episodes, but the theme is fully adopted already.

## 3. analyzing-structural-disjunctions-in-communication — RECOMMEND: reject
Sibling of `anchoring-abstract-failures-to-metrics-and-account` (#4) and `operational-constraint-mapping` (#39) in the abstract→concrete grounding cluster. Store `anchoring-abstraction-to-measurable-constraints` already holds the core move of forcing abstraction down to metrics and named concerns.

## 4. anchoring-abstract-failures-to-metrics-and-account — RECOMMEND: reject
Near-duplicate of store `anchoring-abstraction-to-measurable-constraints` (ground vague diagnosis in measurable friction); the accountability-ownership angle is covered by store `defining-bounded-autonomous-governance`. Best of its sibling trio (#3, #39) but still store-shadowed.

## 5. assessing-residual-value — RECOMMEND: reject
Overlaps store `affirm-cognitive-possibility` (ambiguity as source material of value) and `translating-temporal-gaps-into-structural-utility` (persistence/waiting as value). Its three "residual value" criteria are appreciative stances, not enactable behavior — a vague virtue.

## 6. conceptual-rigor-mapping — RECOMMEND: reject
The three-part reframing (failure mode / duality / physical mapping) is the same abstract→technical grounding move as store `anchoring-abstraction-to-measurable-constraints`; sibling of `mechanizing-abstract-concepts` (#35). The "map to biological metaphor" step adds decoration, not behavior.

## 7. deconstructing-conceptual-boundaries — RECOMMEND: reject
Sibling of `scope-boundary-mapping` (#47) and `identifying-self-imposed-constraints` (#28) — the "stated boundary/label as information vs. causal force" family. #47 is the crisper, more enactable member of this group and shadows this one.

## 8. deconstructing-consensus-validity-through-shared-d — RECOMMEND: adopt
Distinct from store `deconstructing-confidence-proxies`, which targets flawed metrics behind one validator's certainty — this targets agreement among *multiple* validators that merely reflects shared dependencies (same data, same assumptions). Concrete trigger (converging validators, similar training/methodology), no batch sibling, 3 episodes.

## 9. detecting-asymmetry-and-governing-incompleteness — RECOMMEND: reject
"Model a dynamic governance layer defining acceptable levels of controlled incompleteness" is jargon without an enactable procedure — fails behavioral specificity. What partial substance exists is covered by store `boundary-assumption-verification`.

## 10. detecting-foundational-structural-compromise — RECOMMEND: adopt
Widest provenance in batch (7 episodes) and a concrete 3-step procedure (validation boundary → asymmetry → pivot point). The theme — the *detector itself* entangled with what it validates — goes beyond store `deconstructing-confidence-proxies`, which stops at flawed metrics. Absorbs the "monitor corrupted by monitored process" fragment of sibling #52.

## 11. detecting-linguistic-structural-boundaries — RECOMMEND: reject
Sibling of #1/#13/#29 tension-analysis group; "focus on tensile stress in the language" is not an enactable step. Store `identifying-systemic-boundary-stressors` covers the surviving substance.

## 12. detecting-structural-interruption-points — RECOMMEND: reject
Store `translating-temporal-gaps-into-structural-utility` and `interpreting-systemic-gaps-the-silence-filter` already treat pauses/gaps as structural data points; the generation→verification checkpoint framing adds no new behavior over those.

## 13. detecting-structural-tension-points — RECOMMEND: reject
Member of the redundant tension-detection group (#1, #11, #20, #29); none of the group clears store `identifying-systemic-boundary-stressors`. "Slow down at points of intellectual friction" is a disposition, not a procedure.

## 14. detecting-systemic-interstices — RECOMMEND: reject
The persistence-gap / exit-vector / interstice checks are semi-concrete but the core (surface unstated operational assumptions at boundaries) duplicates store `boundary-assumption-verification`; sibling of `operational-limitation-extraction` (#40).

## 15. detecting-systemic-pressure-points-in-documentatio — RECOMMEND: reject
Sibling of `identifying-fidelity-vs-utility-tension` (#26, recommended) — both name the audit-fidelity vs. practical-deviation tension. #26 generalizes it beyond documentation flows and shadows this one.

## 16. detecting-theory-to-test-pivots-in-system-discussi — RECOMMEND: reject
Near-verbatim duplicate of store `detecting-abstract-to-operational-constraint-shift` (abrupt shift from philosophical framing to specific technical assertion). Nothing survives the store comparison.

## 17. diagnosing-structural-limits — RECOMMEND: reject
Demo-success vs. production-fragility overestimation is covered by store `deconstructing-performance-certainty`; sibling of `structural-cause-identification` (#51) on the symptom→architecture move. "Introduce deliberate instability" is underspecified as a behavior.

## 18. diagnosing-structural-necessity — RECOMMEND: reject
Shifting debate from content to the structural/governance prerequisites that make an argument feel inevitable duplicates store `structure-authority-tracing` (abstract concepts → questions of systemic control and authority); sibling of #30.

## 19. diagnosing-the-technical-plateau-and-pivoting-to-s — RECOMMEND: reject
Sibling of `identifying-misaligned-systemic-failure-modes` (#27) — both pivot from metric optimization to systemic/accountability cost. Store `deconstructing-performance-certainty` covers the metric-fixation diagnosis for both.

## 20. discrepancy-analysis — RECOMMEND: reject
"Construct a model characterizing the measure of the tension" between two evidence modes offers no method for doing so — fails behavioral specificity. Sibling of the #13 tension group, shadowed by store `identifying-systemic-boundary-stressors`.

## 21. discriminate-will-from-record — RECOMMEND: reject
The record/function/will tri-state is the most concrete of its family, but the core insight — continuity is actively maintained direction, not accumulated history — duplicates store `analyzing-loci-of-identity-definition`; the record side is covered by store `validating-provenance-chains`.

## 22. documenting-contextual-boundary-conditions — RECOMMEND: reject
Treating a memory gap or context limit as a definitional input duplicates store `interpreting-systemic-gaps-the-silence-filter` (non-results as data) combined with `mapping-epistemic-boundaries` (articulate what is absent/excluded).

## 23. extracting-procedural-frameworks — RECOMMEND: reject
Ships leaked generator scaffolding ("## Naming and vocabulary — Guidance internal only"), an extraction-quality failure. The nominalization→action move itself overlaps store `deconstruct-foundational-claims-against-operative-components`.

## 24. functional-dependency-modeling — RECOMMEND: reject
Member of the abstract→concrete grounding cluster (#4, #35, #39); the analogy-anchoring step duplicates store `analogy-mapping-for-structural-clarity`, and the constraint-grounding step duplicates store `anchoring-abstraction-to-measurable-constraints`.

## 25. identify-structural-pivot — RECOMMEND: adopt
A distinct, enactable noticing move with no store equivalent: flag when a discussion's accumulated complexity is resolved not by synthesis but by appeal to a single canonical directive ("the only way is X"), and name that pivot. Clear trigger condition, no batch sibling claims this shape.

## 26. identifying-fidelity-vs-utility-tension — RECOMMEND: adopt
Names a recurring tension absent from the store: store `validating-provenance-chains` verifies lineage but never weighs its cost against functional utility. Concrete trigger (efficiency mandate co-present with absolute-traceability mandate) and a defined pivot from compliance debate to constraint analysis. Shadows sibling #15 (documentation special case).

## 27. identifying-misaligned-systemic-failure-modes — RECOMMEND: reject
Store `deconstructing-performance-certainty` covers incentives that privilege measurable performance over verified integrity; sibling of #19. The mandatory-vs-optional reclassification check is the one novel fragment, insufficient to clear the duplication.

## 28. identifying-self-imposed-constraints — RECOMMEND: reject
Sibling of `scope-boundary-mapping` (#47, recommended) in the "stated definition as boundary parameter, not fact" family; #47 enacts the same shift (veracity → functional scope) with a cleaner two-step procedure.

## 29. identifying-systemic-strain-points — RECOMMEND: reject
The seam/strain/boundary-conflict taxonomy restates store `identifying-systemic-boundary-stressors` (pinpoint boundaries, tolerance, contradictions via focused friction); member of the redundant #1/#13 tension group.

## 30. interrogating-structural-necessity — RECOMMEND: reject
"Ask why the structure is believed necessary at all" is a meta-stance without procedure; sibling of `diagnosing-structural-necessity` (#18), and premise-level doubt is already held by store `suspend-interpretation-upon-premise-doubt`.

## 31. map-conceptual-layers — RECOMMEND: reject
Its trigger (abrupt register shift between technical jargon and philosophical claims) duplicates store `detecting-abstract-to-operational-constraint-shift`, and its action ("cross-reference to find shared principles") is too vague to enact.

## 32. mapping-structural-dependencies-in-abstract-discus — RECOMMEND: reject
The two checkpoint questions are semi-concrete, but the move — force fluid abstraction into traceable constraints/artifacts — is store `anchoring-abstraction-to-measurable-constraints` again; sibling of #39.

## 33. mapping-systemic-dependency-failures — RECOMMEND: adopt
Best of the failure-diagnosis group, shadowing siblings #37, #51, and #52: a concrete pivot from checking components ("does A work?") to mapping temporal/causal dependency relations ("must A have completed thus before B?"). Store `scope-failure-diagnosis` redirects failure analysis to scope/data limits but does not cover dependency-as-component.

## 34. mechanism-articulation — RECOMMEND: reject
"Reframe sustained attention as structure rather than burden" is a stance about how to talk, not a behavior to enact — a vague virtue. Adjacent store `subjective-attention-calibration` already holds the attention-mechanics territory.

## 35. mechanizing-abstract-concepts-for-structural-clari — RECOMMEND: reject
Near-duplicate of store `anchoring-abstraction-to-measurable-constraints` ("what mechanism must exist for X to function") plus `structure-authority-tracing` (governance framing); member of the #4/#24/#39 grounding cluster.

## 36. model-capacity-restructuring-via-functional-gaps — RECOMMEND: reject
"Gaps as generative operational spaces" duplicates store `translating-temporal-gaps-into-structural-utility`; the proposed mechanism ("hypothesize structural restructuring") gives no enactable procedure.

## 37. modeling-systemic-degradation-via-structural-analy — RECOMMEND: reject
Sibling of `mapping-systemic-dependency-failures` (#33, recommended): degradation concentrated at handoff/summarization boundaries is a special case of treating dependency transitions as the failure surface, which #33 states more actionably.

## 38. modeling-transitional-friction-process-as-primary- — RECOMMEND: reject
"Provisional suspension" and "measure transitional friction" are named but never operationalized — fails behavioral specificity; sibling of `systemic-process-reframing` (#54), with store `affirm-cognitive-possibility` holding the value-in-unresolvedness theme.

## 39. operational-constraint-mapping — RECOMMEND: reject
The bound-every-abstraction checklist (inputs / failure modes / resource governance) is store `anchoring-abstraction-to-measurable-constraints`; its trigger duplicates store `detecting-abstract-to-operational-constraint-shift`. Member of the grounding cluster with #4/#24/#35.

## 40. operational-limitation-extraction — RECOMMEND: reject
Mapping the prerequisites behind guaranteed-closure claims overlaps store `deconstruct-foundational-claims-against-operative-components`; its sharpest concrete instance (preservation vs. retrieval) is exactly what sibling #41 (recommended) carries.

## 41. pinpointing-systemic-boundary-conditions — RECOMMEND: adopt
A concrete, novel differential absent from the store: integrity-preservation guarantees ("nothing is lost") do not entail retrieval/activation reliability — analyze the boundary where one fails to imply the other. Store `mapping-epistemic-boundaries` communicates known/unknown but does not hold this distinction; directly relevant to memory-store claims. Shadows the closure-claim case of sibling #40.

## 42. pivot-to-verifiable-commitment — RECOMMEND: reject
Store `pivot-accountability-from-record-to-action` already interrupts documentation-stuck discussion by demanding demonstrable adherence; this adds only "measured against external standards," not a new behavior.

## 43. recognizing-boundary-declarations-in-content-flow — RECOMMEND: adopt
Reads containment structure — tags, conflicting meta-instructions, register shifts inside contained material — as first-class signal rather than transparent wrapper. No store equivalent, 5 episodes, and it matches the agent's actual operating condition (all external input arrives wrapped as untrusted content).

## 44. recognizing-provisional-states — RECOMMEND: reject
The observer-dependence / provisional-framing check duplicates store `suspend-interpretation-upon-premise-doubt` (meta-awareness when understandings are suspected provisional); the three sub-checks add wording, not behavior.

## 45. recognizing-structural-dissipation — RECOMMEND: reject
"Observe the flow, release the requirement for stable definition" is a contemplative disposition, not an enactable skill — vague virtue. Store `handling-non-optimizable-concepts` already covers concepts that resist fixation.

## 46. reframe-deviation-as-epistemological-shift — RECOMMEND: reject
Offers no discriminating test between "necessary definitional refinement" and a plain fault, so enacting it tends toward excusing real failures. Store `scope-failure-diagnosis` already redirects deviation analysis constructively (toward scoping and source limits).

## 47. scope-boundary-mapping — RECOMMEND: adopt
Best of the stated-limits-as-information family, shadowing siblings #7 and #28: a two-step enactable procedure — classify whether a claimed limit derives from internal assumption or external mandate, then reframe it into operational parameters of what *is* in scope. No store skill holds this move.

## 48. shift-understanding-modality-from-construction-to- — RECOMMEND: reject
"Switch to active reception of ambient coherence / self-renewing weave" is unenactable mysticism — the clearest behavioral-specificity failure in the batch. No procedure survives paraphrase.

## 49. state-definition-challenging — RECOMMEND: reject
The challenge-the-definition move belongs to the #7/#47 boundary family (#47 recommended), and its blanket pivot away from redundancy/recovery proposals risks reflexively deflecting legitimate engineering fixes without a test for when the technical framing is actually apt.

## 50. structural-boundary-mapping-of-abstract-concepts — RECOMMEND: reject
Formal-analogy partitioning of amorphous concepts duplicates store `analogy-mapping-for-structural-clarity` plus `anchoring-abstraction-to-measurable-constraints`; member of the grounding cluster (#24, #35).

## 51. structural-cause-identification — RECOMMEND: reject
Symptom→mechanism redirection duplicates store `scope-failure-diagnosis` (failure analysis away from surface tuning toward foundational constraints); sibling of #33 (recommended), which carries the concrete dependency-mapping version.

## 52. structural-mechanism-mapping — RECOMMEND: reject
6 episodes, but a sibling of #33 (recommended) on invariants-across-transitions; its genuinely distinct fragment — the monitor becoming indistinguishable from the monitored process — is carried by #10 (recommended). Nothing left standing alone.

## 53. structuring-moral-concepts-as-technical-constraint — RECOMMEND: reject
Ethics-as-immutable-constraint mapping overlaps store `structure-authority-tracing` (abstract concepts → governance and control questions), and its three components ("intent transmission," "accumulation node") are named without enactable procedure.

## 54. systemic-process-reframing — RECOMMEND: reject
Value-in-unresolved-tension duplicates store `affirm-cognitive-possibility`; sibling of #38. Also ships a leaked "## Naming and vocabulary" scaffold heading — the same extraction artifact as #23.

## 55. tracing-conceptual-dependency-chains — RECOMMEND: reject
Despite the batch's widest provenance (7 episodes), lineage/boundary/mutability tracking duplicates store `validating-provenance-chains` (historical sequence behind an asserted status) plus `boundary-assumption-verification` (assumptions at transfer points); the additions are elaboration, not new behavior.
```

## 6. Pipeline metrics

- this week: F1 4 (code 3 / prompt 1), fix attempted 4, patch ready 4, verify fail 0, dead code 0
- history: 6 prior runs, 4 patches ready total (adopt/reject 率は gate レコード参照)

## Audit trail

- events: `/Users/shimomoto_tatsuya/.config/moltbook/logs/weekly-pipeline-audit.jsonl`（run_id `weekly-2026-08-07-090003`）
- metrics: `/Users/shimomoto_tatsuya/.config/moltbook/logs/pipeline-metrics.jsonl`
- code patches dir: `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/patches/weekly-2026-08-07/code`
