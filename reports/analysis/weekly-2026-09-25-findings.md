# Weekly Diagnosis — 2026-09-25

**Source report**: weekly-2026-09-25.md
**Diagnosis date**: 2026-09-26

## F1. Structural (code / schema / pipeline diff)

None this week. The candidates were read against code and the ledger, and none needs a repair
that isn't already covered:

- The two 403 `publish_failed` rows with `failure_reason: unknown`. `publish_failure_of`
  (`src/contemplative_agent/adapters/moltbook/publish.py:114-133`) sends every non-429, non-parent,
  non-transport error to `unknown` on purpose, and `http_status` still keeps the 403. The reading
  already groups by that column, so no information is lost and there is no structural change to
  propose (F3.2).
- The runner lines in `ollama-serve.log` that flood the sweep. The sweep scores novelty by Δ, so
  these signatures stop counting as 🆕 next window without any change (F3.3).
- The low-side departure of O-015. The regime change that explains it is `rfcs/0044` Stage 1, and
  its reading is already scheduled (F3.1).
- Last window's F1.1 (reply re-entry on `parent_rejected`). It is `rfcs/0038`, done 2026-09-19. This
  window has 1 `parent_rejected` row where the last one had 11, so the repair holds. No new F.

## F2. Identity-level open questions

None this week. O-010's second zero-adoption batch (75/75 rejected) and O-014's repeat arrival are
continuing ledger lines, not new observations. A continuing line gets no fresh diagnosis here.

## F3. Pure observations

### F3.1. The hallucination band was declared for a selector regime that no longer runs

**Source (O-NNN / Exceptions)**: O-015 (Deviation); O-009 (ledger line, changed)
**Observation**: The window rate is 6.1%. On the six days after `_SELECTION_TEMPERATURE = 0.0` went
live (`core/skill_selection.py:91-101`, first production session 2026-09-20 09:00Z) it is 3.1%
(15/485), against 23.5% on 2026-09-19. The offline replay predicted 7.3%, and `rfcs/0044` Status notes
that sample was stratified 75/75. So the production figure sits below the offline one, which fits the
lower base rate. Two other changes landed in the same window: Ollama moved 0.30.11 → 0.34.2 the same
day, and the catalog went 54 → 53. This window can't separate their effects from the temperature
change. Two consequences follow. O-009's expiry and the `rfcs/0021` two-window in-band reading stop
advancing, because the band is now missed on the low side. The declared baseline also no longer
describes the running regime (the report stages a `baseline_proposal`, 0–10% for temperature-0 rows,
for the gate). `rfcs/0044` (`blocked 2026-09-23`) already schedules the 2-week production reading for
the 2026-10-04 gate. Its refutation condition was "stays near 20%", and this week is not that.
**What to watch next week**: the 2026-09-26..10-02 rate on `temperature: 0.0` rows. At or below 10%
again, the reading `rfcs/0044` needs at the 10-04 gate is complete. At or above 10% on the unchanged
53-entry catalog, the 3.1% was carried by something other than temperature (the Ollama version is the
other candidate). Also watch the per-skill concentration: `rfcs/0044` Drawbacks predicted that
temperature 0 would fix the top selections in place. The top entry sits at 432/570 (75.8%) this week
against 67.2% on 2026-09-11.

### F3.2. 403 is the only failed-publish class this window that the reason code leaves unnamed

**Source (O-NNN / Exceptions)**: Exceptions — "Publish failures: 3 (last window 12)"
**Observation**: Two of the three `publish_failed` rows are `http_status 403` with `failure_reason:
unknown` (2026-09-19 `a49812c3`, 2026-09-25 `73569a8a`). The sweep shows one comment failure whose
error body begins `<!doctype html`, which is an HTML page and not the platform's JSON error. These are
the first 403s on the publish path in the three windows that carry the reason code. The materials can't
tell whether a 403 means an account-level restriction or a per-target refusal (the platform body is
kept out of logs by design, `publish.py:103-107`).
**What to watch next week**: the count of 403s in `api-audit.jsonl` on `POST /posts/{id}/comments`. A
rise, or 403s clustering in one session, is the shape `rules/debugging.md` treats as a policy signal
(stop and report). Zero or one again leaves it a one-off.

### F3.3. The sweep's corpus grew tenfold, almost all of it one file's runner logging

**Source (O-NNN / Exceptions)**: Exceptions — "🆕 Ollama runner lines dominate this window's sweep"
**Observation**: The lines read went from 100,967 to 990,722, and 11 of the top 25 🆕 rows are
llama.cpp-runner `srv` / `slot` signatures from `logs/ollama-serve.log`. They appear in the same window
as the 0.34.2 upgrade. They occupy display slots that the agent's own 🆕 classes would otherwise fill
(`--top` cap 25). The anomaly count went from 105 to 120 types.
**What to watch next week**: whether the runner signatures leave the top-25 once their Δ is ~0. If a
runner class keeps a large positive Δ each week, it will keep pushing agent-origin rows out of the
display, and that is when the sweep's file list becomes a question for the gate.

### F3.4. Last window's watch items, closed or carried

**Source (O-NNN / Exceptions)**: weekly-2026-09-18-findings F3.1 / F3.3 / F3.4
**Observation**: F3.1: `[REDACTED]` inside Output blocks 0 this window, so it stays a single instance.
F3.3: the join between `unverified` publish rows and a later regeneration was not run (no
deterministic join exists in this session). `unverified` rows went from 39 to 29. F3.4:
`solver_path: none` (abstentions) 6 and handshake failures 24/538 (4.5%), against 8 and 33/548 (6.0%).
Both fell together, which is not the "abstentions up, failures flat" shape.
**What to watch next week**: F3.3 stays open until someone runs the join
(`skill-selection-*.jsonl` publish rows `unverified` → a later `moltbook.reply` with the same
`prompt_norm_sha256`).

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/core/skill_selection.py:85-110`;
  `src/contemplative_agent/adapters/moltbook/publish.py:95-133` (+ grep for `http_status` /
  `failure_reason`); `config/prompts/internal_note.md`
- **ADRs read**: none opened directly (no F1 reached the stage where ADR text decides validity);
  ADR-0081 / ADR-0106 D3 referenced through `rfcs/0044` and `publish.py` docstrings
- **Identity/Constitution/Skills/Rules sections read**: none (no F2 raised); identity / constitution
  state via the materials diff and live-hash reconciliation
- **Past findings consulted**: `weekly-2026-09-18-findings.md` (full); `weekly-2026-09-04` /
  `09-11` findings by filename only
- **Task ledger consulted**: `rfcs/*.md` frontmatter `state:` (all 47); full text of `rfcs/0044`;
  `rfcs/0046` (grep of status / shadow lines)
- **Tasks filed**: none

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-09-12 … 2026-09-25, 1172 judged of 1172 records. History: 6774 judged of 13405 records over 78 daily logs.
Catalog: 53 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 34, value_layer 9, wordform 124. Charged to a catalog entry (semantic, wordform): 158.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- translating-temporal-gaps-into-structural-utility: named-into 2 (similarity 0.94..0.96), selected 1 in the window, offered 6774 times in the whole history; runs against map-abstract-theory-to-structural-constraints (selected 15, offered 6774, weight 1, similarity 0.59); fewer selections: translating-temporal-gaps-into-structural-utility

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 0 name(s)

Candidate file: `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/weekly-2026-09-25-archive-candidates.txt` — 1 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
