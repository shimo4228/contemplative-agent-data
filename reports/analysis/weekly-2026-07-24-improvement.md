I have everything needed. Both failures are now fully explained, and the proposal targets the one recurring mechanism.

## What the evidence shows

**Recurring code**: `EMPTY_DIFF` (fix stage, both F1.1 and F1.2, two consecutive runs).

**Mechanism (confirmed deterministically, not inferred)**: the findings file `weekly-2026-07-24-findings.md` was written **2026-07-25 12:10:24** (epoch 1784949024). The fixes for both findings then landed on main the *same day*: `d031deb` at 12:47 touches F1.1's referenced paths, and the commits covering F1.2's paths (`9a7ea22`, then `5fcb373` at 12:57) likewise postdate the findings file. When the pipeline re-ran on 2026-07-28 it **reused** the stale findings (`stage=diagnosis result=reused` in the audit log). Each fix session then did the correct thing — the attempt logs show both agents verifying via `git log` that the finding was already implemented at HEAD and deliberately changing nothing ("No change made — the finding is already implemented on main HEAD", citing the exact commits). A clean worktree is the agent's only way to express this, and the orchestrator maps clean worktree → `result=failed reason=EMPTY_DIFF`. So `EMPTY_DIFF` is currently conflating a benign outcome (finding already resolved) with a genuine failure (session produced no work), and the recurrence trigger fired on the benign one.

The fix belongs in the orchestrator, as a deterministic check (structural property — "did any referenced path receive a commit after the findings file was written" is answerable from bytes and git history alone; no LLM judgment, and no trusting the fix agent's self-declared "already implemented" claim).

## Proposed change

```diff
--- a/scripts/weekly-pipeline.sh
+++ b/scripts/weekly-pipeline.sh
@@ -313,11 +313,38 @@
     else
         echo "[$RUN_ID] stage 4: fix ($F1_COUNT F1 findings, cap $MAX_FIX_TARGETS)"
+        # Findings freshness baseline for the stale-finding gate below
+        # (stat -f = macOS, stat -c = Linux fallback; empty => gate disabled).
+        FINDINGS_EPOCH=$(stat -f %m "$FINDINGS_MD" 2>/dev/null || stat -c %Y "$FINDINGS_MD" 2>/dev/null)
         targeted=0
         while IFS=$'\t' read -r fid scope; do
             if (( targeted >= MAX_FIX_TARGETS )); then
                 add_reason BUDGET_EXHAUSTED
                 audit fix_result fix_id="$fid" scope="$scope" result=failed \
                     attempts=0 reason=BUDGET_EXHAUSTED
                 continue
             fi
+            # Stale-finding gate (EMPTY_DIFF recurrence, weekly-2026-07-24):
+            # a reused findings file can predate commits that already
+            # implement a finding — the fix session then correctly changes
+            # nothing and the run mislabels that as an EMPTY_DIFF failure.
+            # Deterministic check: any commit touching a referenced path
+            # after the findings file was written => skip the fix session and
+            # surface STALE_FINDING in the packet (the human decides whether
+            # to re-run diagnosis). Fail-open: no paths / no epoch => proceed.
+            finding_paths=$(python3 -c "
+import json, sys
+for f in json.load(open(sys.argv[1]))['f1']:
+    if f['id'] == sys.argv[2]:
+        print('\n'.join(f['paths']))
+        break
+" "$FINDINGS_JSON" "$fid")
+            if [[ -n "$finding_paths" && -n "$FINDINGS_EPOCH" ]]; then
+                # Unquoted on purpose: parse_findings path tokens cannot
+                # contain whitespace, so word splitting is safe here.
+                last_touch=$(git -C "$PROJECT_ROOT" log -1 --format=%ct -- $finding_paths 2>/dev/null)
+                if [[ -n "$last_touch" ]] && (( last_touch > FINDINGS_EPOCH )); then
+                    add_reason STALE_FINDING
+                    audit fix_result fix_id="$fid" scope="$scope" result=skipped \
+                        attempts=0 reason=STALE_FINDING
+                    continue
+                fi
+            fi
             bodyfile="$RUN_LOG_DIR/body-${fid//[.\/]/-}.md"
```

## Rationale

- **Recurring code → mechanism → change**: `EMPTY_DIFF` fired both weeks because the fix stage was handed findings whose referenced paths had already been committed to *after* the findings were written. The gate detects exactly that precondition before spending a fix session (each costs up to 2 × 20-min claude runs + Verify), and reroutes it to a distinct, honest reason code. Genuine `EMPTY_DIFF` (fresh finding, session completes, changes nothing) remains a failure — the code keeps its meaning instead of being retired.
- **Why not trust the agent's "already implemented" summary instead**: that would put an LLM claim on the control path of a failure classification. The mtime-vs-commit check is code-decidable (when-code-when-llm), fail-open, and would have caught both of this week's cases (1784951224 > 1784949024).
- **Design choices**: `result=skipped` (not `failed`) keeps the skip out of the `verify_fail` metric while still rendering in the packet's fix table via the existing `fix_result` path — no change needed in `build_decision_packet.py`. The skip does not consume a `MAX_FIX_TARGETS` slot (no session was spent). If `STALE_FINDING` itself recurs two weeks running, the improvement trigger fires on *it* — correctly, since that would indicate the diagnosis-reuse policy (`[[ -s "$FINDINGS_MD" ]]` → reuse unconditionally) needs its own revision.
- **Alternatives considered**: (a) regenerate diagnosis whenever HEAD moved past the findings file — heavier (30-min diagnosis session), and reuse is deliberate for `--skip-report` reruns; (b) drop `EMPTY_DIFF` from the recurrence trigger — hides genuine failures; (c) agent-declared `RESOLUTION:` marker line — LLM-trusted classification, rejected above.
- **Observed but deliberately not bundled** (one failure mode, one proposal): F1.2's attempt-1 `verify_fail` was `uv sync --frozen` failing because `uv.lock` is gitignored, so every fresh worktree fails Verify setup once and burns a retry. That is a separate latent failure mode in `run_verify`; if `VERIFY_FAIL_MAX_ATTEMPTS` or wasted retries show up in coming weeks, it deserves its own proposal (run `uv lock` before `uv sync --frozen`, or align Verify with the repo's `uv pip install -e ".[dev]"` convention).

## Falsifiable prediction for next week

If the change works: in any run where the audit shows `stage=diagnosis result=reused` **and** a post-findings commit touched a finding's referenced paths, the audit will contain `fix_result … result=skipped reason=STALE_FINDING` for that finding and **zero** `reason=EMPTY_DIFF` events; `reason_codes` in the weekly metrics record will not contain `EMPTY_DIFF`, so the improvement trigger goes quiet on it. If instead `EMPTY_DIFF` appears again *alongside* fresh (same-run) findings, the stale-reuse hypothesis is wrong and the failure is elsewhere in the fix stage — that outcome should be treated as refuting this proposal, not as grounds to re-tune the gate.
