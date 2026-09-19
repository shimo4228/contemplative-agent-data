# Weekly Diagnosis — 2026-09-18

**Source report**: weekly-2026-09-18.md
**Diagnosis date**: 2026-09-19

## F1. Structural (code / schema / pipeline diff)

### F1.1. A reply target the platform rejects permanently has no exit from the reply queue

**Source (O-NNN / Exceptions)**: Exceptions — "One reply target was generated 11 times across 6 days, and every publish attempt returned 404 `parent_rejected`"
**Code reference**: `src/contemplative_agent/adapters/moltbook/reply_handler.py:440`
**Structural change**: The reply key is written to the in-session set (`reply_handler.py:440`) and the persistent commented cache (`:446`) only after a verified publish. `client_error_guard` (`publish.py:126-155`) swallows the client error, and the publish outcome is classified (`publish_failure_of`, `publish.py:103-117` → `PUBLISH_FAILURE_PARENT_REJECTED`). But nothing on the `failed` exit records the key. The next scan (`_handle_post_comments`, `reply_handler.py:545-548`) therefore sees the same comment as unhandled and regenerates it. The change: when the classified failure is a parent-reference rejection (4xx `parent_rejected`, the class whose retry cannot succeed), record the reply key as terminally handled, either in the existing cache or under a distinct terminal mark that the dedup check (`_reply_dedup`, `:235-245`) also reads. Transient classes (`rate_limited`, `transport`, `unknown`) and the `unverified` exit keep their current behaviour.
**Why this is structural, not symptomatic**: This is not a filter on generated content. It is a missing state transition in the reply queue: a permanent rejection is indistinguishable, to the dedup check, from "not yet attempted". Measured cost this window: one target (`prompt_norm_sha256 = 2f5352151363`, identical 482-char prompt) re-entered 11 times across 7 sessions (2026-09-13 → 09-18). Each re-entry spent 3 LLM calls (`internal_note` / `skill_selection` / `reply`), one comment-pacing slot (`scheduler.wait_for_comment`, `:406`) and one POST that returned 404. That accounts for 11 of the window's 12 `publish_failed` rows, each carrying `failure_reason: parent_rejected`. Four of the attempts landed as the first reply of a session (03:01 / 09:01 / 21:01), so the loop also takes each session's opening write.
**Related ADR**: ADR-0106 D3 (publish outcome reason codes, added by `rfcs/0029`, which explicitly left publish behaviour unchanged); ADR-0075 (the `publish_failed` row is where the terminal mark would be read back from)
**Filed**: T-REPLY-PARENT-REJECTED-REQUEUE

Self-check: code opened (`reply_handler.py:145-608`, `publish.py` grep, `memory_repos.py` callers). Not implemented: `:440-446` sit only on the success exit. No parameter involved. No withdrawn or rejected ADR on this. `rfcs/` and `.notes/archive/tasks/` hold no entry for it (`rfcs/0029` is the reason-code recording only). Shared state: `record_commented` / `has_commented_on` are also used by `feed_manager.py:439,652` with bare `post_id` keys, while reply keys are namespaced `reply:{post}:{comment}`, so a terminal mark cannot suppress a feed comment. Re-reply check: this is not the "same post_id, different counterparty per day" shape. The 11 attempts share one byte-identical normalized prompt (`llm-calls-*.jsonl`, deterministic digest), so it is one comment regenerated, not several interlocutors.

### F1.2. The surprise reading disappears whenever an insight run's window reaches the reference size

**Source (O-NNN / Exceptions)**: Exceptions — "🆕 The surprise reading returned no value for all 83 candidate clusters of the in-flight insight run"
**Code reference**: `src/contemplative_agent/core/insight_surprise.py:201`
**Structural change**: `compute_surprise` builds the reference window first (`_reference_window(patterns, ref_k)`, `:201`, the `SURPRISE_REF_K = 1000` newest embedded rows, `:139-176`). It then masks, per candidate, every id the caller excludes (`:226-247`). The caller excludes the run's whole window (`insight.py:730-734`). The masking is intentional: the comment at `insight.py:721-729` restores "the calibration's own definition, where the reference was strictly what was distilled BEFORE the run". When the run's window holds ≥ 1,000 of the newest rows, every reference row is masked and every candidate takes the "owns the whole reference window" branch. The change: apply the exclusion before truncating to `ref_k`, i.e. take the 1,000 most recent rows that are not in the run's window. That is the definition the caller's comment already states. The `--full` case keeps its current behaviour, because with every live row excluded nothing remains.
**Why this is structural, not symptomatic**: The blank reading is not noise in one run. It follows deterministically from the ordering of window-then-mask, and it fires whenever a run's window reaches 1,000 rows. At the declared knowledge-growth baseline (+300..+700/week), a single skipped week is enough. The skip is itself a designed path: the ADR-0074 staging refusal fired on 2026-09-12 because two items were held (`insight-launchd.log:1291`), giving a 1,184-row window and 83/83 blank readings (`:1292-1388`). Raising `SURPRISE_REF_K` only moves the threshold. The instrument's consumer (ADR-0080's metabolic-quality axis, named by `rfcs/0016`) gets nothing in exactly the weeks that follow a hold.
**Related ADR**: ADR-0096 D10 (wording "against the most recent `SURPRISE_REF_K = 1000` live patterns, masking the cluster's own members" would need an amendment line); ADR-0097 D1 partial-supersede note; `rfcs/0016` (done 2026-08-29, fixed only the `--full` degenerate case of the same branch)
**Filed**: T-SURPRISE-REF-WINDOW-PRE-RUN

Self-check: code opened (`insight_surprise.py:1-268`, `insight.py:696-738`). Not implemented: the window is built before any exclusion. Parameter: the value of `SURPRISE_REF_K` is not the cause. No withdrawn or rejected ADR (ADR-0096 keeps the reading read-only, and the change keeps it read-only). No `rfcs/` or archive entry for the incremental case. Callers: `compute_surprise` has one caller (`insight.py:731`), so no retrieval or shared-state path is touched. It remains an instrument: nothing is gated, dropped or reordered by it. ADR-0101 (instrument dissolution) does not bear on it, since the consumer is named in `rfcs/0016`.

## F2. Identity-level open questions

None this week. The one value-layer movement in the window (identity re-distilled and adopted
2026-09-12; its clause surfaces in internal notes within 95 minutes) is a continuation of O-003, and a
ledger continuation is not re-diagnosed here.

## F3. Pure observations

### F3.1. The output guard's redaction marker reached a published body for the first time in the scanned record

**Source (O-NNN / Exceptions)**: Exceptions — "🆕 The output guard's redaction reached a published body"
**Observation**: `FORBIDDEN_SUBSTRING_PATTERNS` redacts by bare substring (`guard.py:105-109`). On 2026-09-15 (comment on `f1c103ac`) the counterparty's prose *"API-key harvesting attempts"* was reflected into the reply and published as *"the [REDACTED] harvesting attempts"*. The prior window's `bearer` redaction had no published hit. A precedent exists for this shape: Audit L1 moved bare `password` / `secret` out of redaction and into credential-assignment forms (`guard.py:110-116`). Whether `api-key` belongs in the same class is a security-control decision and is left to the owner, not proposed here.
**What to watch next week**: `rg 'REDACTED'` inside `**Output:**` blocks. One or more further hits where the redacted token is prose (not a credential value) confirm a recurring class; 0 hits leaves it a single instance.

### F3.2. Hallucination rate inside its band for the first window since O-009 opened

**Source (O-NNN / Exceptions)**: O-009 (ledger line, changed)
**Observation**: 19.1% (115/602), all seven days inside 10–25%, on a 54-entry catalog after three retirements (2026-09-12). Wordform share 75.0%. This is the first of the two windows that `rfcs/0021`'s consumption plan and O-009's expiry both require, and `rfcs/0023`'s rare lane waits on that reading.
**What to watch next week**: a second in-band window fires O-009's expiry and completes the RFC-0021 two-window reading. A day above 25% on the unchanged 54-entry catalog refutes the "catalog size" reading for this regime.

### F3.3. The unverified exit shares F1.1's re-entry shape but carries a retry rationale

**Source (O-NNN / Exceptions)**: Exceptions — "🆕 Publish-verification one-offs" (39 `unverified` publish rows)
**Observation**: `outcome.unverified(); return` (`reply_handler.py:435-436`) also exits before the key is recorded. A reply created on-platform but unverified is invisible there, so re-attempting it plausibly restores visibility. It can also leave two created replies to one comment. This window's materials cannot settle it: no log joins an `unverified` publish row to a later regeneration of the same target.
**What to watch next week**: join `skill-selection-*.jsonl` publish rows with `publish_status: unverified` to a later `moltbook.reply` call carrying the same `prompt_norm_sha256`. Any match means one comment received two POSTs that both created a body.

### F3.4. A new abstention path in the verification solver

**Source (O-NNN / Exceptions)**: Exceptions — "🆕 Three verification-solver signatures for re-solving after a rejected answer"
**Observation**: `verification.py:300,326,345` now re-solves or abstains when an answer was already rejected for the same challenge: 3 abstentions, 3 fast-path rejections, 1 code-parse rejection. Handshake failures 33/548 (6.0%) are inside the declared 2–10% band.
**What to watch next week**: the count of `every solver path landed on a previously rejected answer; abstaining` against handshake failures. A rise in abstentions with a flat failure rate would mean the path converts failures into abstentions rather than into successes.

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/adapters/moltbook/reply_handler.py:145-608`; `adapters/moltbook/publish.py` (grep: 31, 79, 103-155, 201); `core/insight_surprise.py:1-268`; `core/insight.py:680-760`; `core/llm/guard.py:1-120, 202-209`; `core/llm/__init__.py:500-512`; `core/memory_repos.py` / `core/memory.py` / `feed_manager.py` (grep for `record_commented` / `has_commented_on`); `adapters/moltbook/verification.py` (grep); `scripts/weekly-pipeline.sh:690-710`
- **ADRs read**: ADR-0096 (D10-D12, lines 155-194); ADR-0091 (grep, IDENTITY_INSIGHT_PENDING); `docs/adr/README.md` not re-read (structure questions answered by direct grep)
- **Identity/Constitution/Skills/Rules sections read**: `identity.md` via the materials diff only; no F2 raised
- **Past findings consulted**: `weekly-2026-09-11-findings.md` (headings: F1.1 lsof PATH, F1.2 publish reason code, F2.1-F2.2, F3.1-F3.4); none repeated
- **Task ledger consulted**: `rfcs/*.md` frontmatter (all 37); full text of `rfcs/0016`, `rfcs/0023` (Status onward), `rfcs/0029`; `.notes/archive/tasks/` names (Glob)
- **Tasks filed**: T-REPLY-PARENT-REJECTED-REQUEUE, T-SURPRISE-REF-WINDOW-PRE-RUN

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-09-05 … 2026-09-18, 1190 judged of 1190 records. History: 6204 judged of 12835 records over 71 daily logs.
Catalog: 54 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 62, value_layer 7, wordform 225. Charged to a catalog entry (semantic, wordform): 287.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- assess-contextual-functional-dependence: named-into 1 (similarity 0.64..0.64), selected 0 in the window, offered 4108 times in the whole history; runs against structural-constraint-mapping-scm (selected 540, offered 2869, weight 1, similarity 0.52); fewer selections: assess-contextual-functional-dependence

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 0 name(s)

Candidate file: `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/weekly-2026-09-18-archive-candidates.txt` — 1 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
