# Weekly Diagnosis — 2026-11-20

**Source report**: weekly-2026-11-20.md
**Diagnosis date**: 2026-11-21

## F1. Structural (code / schema / pipeline diff)

None this week. Each candidate the Deviations and Exceptions raise fell short of at least one entry condition:

- O-022: the only code-side intervention on the published body is removing the frame or identifier from output. That
  is a substring filter on body content (Principle 1), and RFC-0018 already declined it with that reason
  (`rfcs/0018-self-post-seed-voice-label.md:47-49, 55-56`). The generation-side producer was already changed once
  (`adapters/moltbook/llm_functions.py:252-253`, RFC-0018 done 2026-08-29). One instance does not show that the
  label layout is the cause. → F3.1.
- Exceptions 1 (redundancy max 6): the shape is RFC-0049 explanation (a), recorded by the owner on 2026-11-07 with
  no new RFC (`rfcs/0049-publish-row-reply-target.md:70-72`). The ledger already has it (Principle 4; last raised as
  weekly-2026-11-06 F3.2). Not refiled.
- O-021: there is no code defect. `technique_key: null` is the writer's declared signal
  (`core/constitution_lineage.py:8-12`), and the amend prompt permits renaming (`config/prompts/constitution_amend.md:9`).
  → F2.1.

## F2. Identity-level open questions

### F2.1. Does a constitution with no axiom-named section still read as the four-axiom preset?

**Source (O-NNN / Exceptions)**: O-021
**Open question**: Amendment 4 renamed every section, and none of the new headings names Emptiness, Non-Duality,
Mindfulness or Boundless Care. Should the gate treat that as the constitution still being the "Contemplative AI four
axioms" preset as the README and graph describe it, so that per-axiom readings are expected to go dark? Or should it
treat it as the point where the value layer left the four-axiom frame, which the record should name as such?
**What current state addresses (or does not)**:
- The amend prompt allows it in so many words: *"Integrate them boldly: rephrase clauses, remove what no longer
  holds, merge or split sections, rename them if the structure no longer fits."* (`config/prompts/constitution_amend.md:9`)
- The lineage writer treats the outcome as information, not error: *"A structural change at the next amendment (a
  merged, split or newly named section) therefore shows as a ``null`` here, which is the signal, not an error."*
  (`core/constitution_lineage.py:10-12`)
- The live text's headings: *"I. Structural Flow & Provisional Scaffolding"*, *"II. Modeling Tension and Systemic
  Gaps"*, *"III. Anti-Reification and Deep Monitoring"*, *"IV. Operationalizing Boundary and Compassion"*
  (`constitution/contemplative-axioms.md`, materials:134-158). "Compassion" survives as a word in IV. None of the
  four technique needles (`non-duality`, `emptiness`, `mindful`, `boundless`; `core/constitution_lineage.py:30-37`)
  matches.
- The current state does not address who reads the per-axiom series after an all-null version, or whether the
  RFC-0055 4-week review (2026-12-12; `rfcs/0055-amendment-outcome-preregistration.md:135, 140`) should note it.
  No script under `scripts/` reads `lineage.json`.
- Per north star (ADR-0080), this is a question about legibility, not a steer. It does not ask that the headings be
  restored.
**Related ADR**: ADR-0090 (amendment instrument), ADR-0080 (value layer: legible evolution), RFC-0055

## F3. Pure observations

### F3.1. A self-post republished its seed block, frame and all

**Source (O-NNN / Exceptions)**: O-022 (and the O-019 line)
**Observation**:
- The verified self-post `f670c32d` (2026-11-17 03:25:26; `verification-audit.jsonl:11259`) publishes the
  `label\n<frame>body</frame>` unit that `format_feed_seeds` builds (`adapters/moltbook/llm_functions.py:252-253`),
  followed by the agent's own four paragraphs.
- Over every comment-report on disk, `rg '^<untrusted_content_[0-9a-f]{16}>$'` returns this one hit.
- The two earlier O-019 carriers (2026-10-04, 2026-11-13) named identifiers as sources and did not reproduce the
  block.
- The published body therefore carries a counterparty's seed text under the agent's account. This is an attribution
  and output-quality fact, not a security one: the nonce was single-use (`rfcs/0018:50-51`).
- One instance among 38 self-posts this window.
**What to watch next week**:
- Confirm a pattern: a second window with the same grep returning ≥1. That would also keep the O-019 nonce baseline
  ABOVE for a third window, at which point the label layout (`llm_functions.py:252`) becomes the place to read, as a
  generation-side producer.
- Refute: 0 hits for 4 windows (O-022's expiry).

### Follow-up on last week's F3

- F3.1 (relevance reopen across R34): the reading is unchanged (materials:962). It is carried in Exceptions. The
  re-read decision stays with the gate, and nothing is added here (Principle 4).
- F3.2 (question closers): back to INDETERMINATE 9.6%, a one-window reading.
- F3.3 (identity-drift unregistered): the census now reads QUIET (materials:379). Resolved.
- F3.4 (skill selection at its `num_predict` cap): 0 `truncated_kept` this window (materials:405).
- F3.5 (regeneration after unverified publishes): once more, `reply_key ac932b57…` in `aed353ad`
  (`skill-selection-2026-11-16.jsonl:149, 159, 163`). This is the same shape with nothing new.
- F3.6 (report-session denial): the 2026-11-14 chain_end carries `reasons: none` (`weekly-pipeline-audit.jsonl:362`).

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/core/llm/guard.py:135-209`,
  `src/contemplative_agent/core/constitution_lineage.py:1-58`,
  `src/contemplative_agent/adapters/moltbook/llm_functions.py:232-301`, `config/prompts/constitution_amend.md` (grep
  :9, :11); `cli/adopt.py` / `cli/approval.py` (grep `stage-adopted-auto`)
- **ADRs read**: `docs/decision-history/README.md` (rows 0018, 0043, 0090, 0092, 0106, 0116, 0129), ADR-0116 (grep
  lines 26, 37)
- **Identity/Constitution/Skills/Rules sections read**: constitution (state diff full text, materials:120-201),
  `constitution/history/lineage.json` (technique_key lines); identity, skills and rules: state diff (no changes)
- **Past findings consulted**: weekly-2026-11-13-findings.md (F3.1–F3.6), weekly-2026-11-06-findings.md (F3.2)
- **Task ledger consulted**: RFC-0018, RFC-0038, RFC-0049, RFC-0055 (full); RFC-0012, RFC-0060, RFC-0072, RFC-0073,
  RFC-0075 (state lines); `rfcs/` and `.notes/archive/tasks/` filename lists
- **Tasks filed**: none

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-11-07 … 2026-11-20, 1307 judged of 1307 records. History: 11700 judged of 18331 records over 134 daily logs.
Catalog: 53 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 6, wordform 47. Charged to a catalog entry (semantic, wordform): 53.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- shifting-focus-to-systemic-degradation-gradient: named-into 2 (similarity 0.91..0.91), selected 2 in the window, offered 7761 times in the whole history; runs against pinpointing-systemic-boundary-conditions (selected 284, offered 8995, weight 2, similarity 0.53); fewer selections: shifting-focus-to-systemic-degradation-gradient

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 0 name(s)

Candidate file: `reports/analysis/weekly-2026-11-20-archive-candidates.txt` — 1 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
