# Weekly Diagnosis — 2026-07-11

**Source report**: weekly-2026-07-11.md
**Diagnosis date**: 2026-07-12

> **Scope note.** Four things the upstream report could not see ARE determinable
> from the repo record and task ledger, and they reframe most of this week's
> dominant signals as *already-diagnosed-and-fixed in-window*:
>
> 1. **The 13 new skills passed a human approval gate.** The 07-09 run was the
>    first manual execution of ADR-0074 staged insight: 632 patterns → 65
>    clusters → 54 staged → operator review via `adopt-staged` — **13 adopted /
>    41 rejected**, audit-recorded (ledger T-INSIGHT-BOOT, done 2026-07-09). The
>    report's "survivors of a lossy batch" framing under-describes: the 13 are
>    survivors of a lossy batch *and* a per-item human review.
> 2. **Last week's F1.1 landed and worked.** The learned-corpus disposition
>    framing shipped 2026-07-06 (`50b8070`). D1's observation — activation
>    scaffolding present early 07-06, gone from Output by 07-09 — is that fix
>    taking effect, on schedule.
> 3. **The context-budget squeeze was found and fixed mid-window.** The 🆕
>    `skipping llm call` (+62) signature and the 07-09 zero-self-post day are
>    the skill-store growth (6 → 19 files, ~60 KB) pushing estimated input past
>    the pre-flight guard, which then *skipped* calls; `15ae37f` (2026-07-10)
>    changed skip → clamp, and `2f616eb` added a system-prompt budget instrument
>    at the adopt gate. Self-posts resumed 07-10 (4) and 07-11 (4).
> 4. **The "fluid-skill trigger blurring" is the ADR-0048 altitude clean pass.**
>    `2f616eb` made stocktake clean rewrites land in place; "repeats every fixed
>    interval" → "repeats in similar contexts" is the trigger-altitude
>    generalization working as designed, not silent corruption.
>
> One F1 this week: a log-channel hygiene gap — full generated bodies are
> INFO-logged into the same `*.log` stream the Log Anomaly Sweep reads, which is
> how the agent's own prose surfaced as a 🆕 anomaly signature.

---

## F1. Structural (code / schema / pipeline diff)

### F1.1. Full generated bodies are INFO-logged into the launchd log stream that the anomaly sweep ingests — content records and the operational log channel are conflated

**Source quote (B — Log Anomaly Sweep)**: 🆕 (+1) *"the pivot point here feels
profoundly resonant: using the \*truncation\* itself as"* — "エージェント自身が
生成した Output の断片…が log-anomaly ストリームに現れたもの、すなわち sweep が
読むチャネルへエージェントの散文が漏出している"; plus the 🆕
`feed_manager: comment on …` / `reply_handler: reply on …` WARN-line spray.

**Code reference**:
- `src/contemplative_agent/adapters/moltbook/feed_manager.py:467` — `logger.info(">> Comment on %s:\n%s", post_id[:12], comment)`
- `src/contemplative_agent/adapters/moltbook/reply_handler.py:348-350` — `logger.info(">> Reply to %s on %s:\n%s", …, reply)`
- `src/contemplative_agent/adapters/moltbook/post_pipeline.py:424` — `logger.info(">> New post [%s] (id=%s):\n%s", title, post_id, content)`
- Sweep intake: `scripts/weekly-analysis.sh:174-186` — deterministic intake of `*.log` (`log_anomaly_sweep.py`), line-signature ranked by novelty.

**Structural change**: stop embedding the multi-line body in the INFO record.
Keep a one-line, logger-prefixed preview for interactive visibility (the tmux
watch workflow), e.g. `logger.info(">> Comment on %s: %d chars: %.80s…",
post_id[:12], len(comment), comment.replace("\n", " "))`, and emit the full
body only at DEBUG. The canonical full-text records already exist and are
untouched: the episode log `content` field (written directly below each of
these call sites) and the comment-reports regeneration
(`reports/comment-reports/`, the sanctioned read path).

**Why this is structural, not symptomatic**: it does not touch what is
generated or published (no post-generation filter, Principle 1) and names no
token (Principle 2). It separates two channels that ADR-0075 already assigns
distinct roles — the replayable content record (episode JSONL) vs the
operational log — so LLM-generated prose (downstream of untrusted feed
content) stops flowing into the channel that the anomaly sweep normalizes and
the weekly-analysis LLM then reads. Today the multi-line bodies produce raw,
prefix-less continuation lines, each a candidate anomaly signature: the class
of noise disappears with the channel split, no matter what next week's prose
looks like. It also restores the intent of the CLAUDE.md read-path boundary
(episode logs are the injection-risk channel; `*.log` is "self-written" and
safe to read) — full feed-derived generations in `*.log` quietly erode that
distinction.

**Validity self-check**: not already implemented (all three call sites log the
full body today); parameter not already effective (the sweep has no
prose-line exclusion); no ADR rejects it (ADR-0075 supports channel
separation); not in the principles.md rejected-mechanisms appendix; not in the
task ledger (T-OBS-EMB / T-OBS-INJ are different observability items); not a
retrieval/shared-state change (no caller-graph question); not a re-reply
finding.

**Related ADR**: ADR-0075 (observability by default — the episode/JSONL record
is the audit channel), ADR-0007 (security boundary model — read-path
classification).

---

## F2. Identity-level open questions

### F2.1. Should the adopt-staged review weigh a skill's *name register* — its recyclability as an in-comment gambit — or is vocabulary feedback an accepted cost of the AKC loop?

**Source quote (E T8; D2; B — Skills)**: *"it's about **Structural Authority
Tracing** within a shared domain"* (07-10 #0a845bb8) — an auto-extracted skill
name appearing verbatim, capitalized, as an analytical gambit within days of
adoption; "provenance chains," "confidence proxies," "scope-failure" recurring
07-09 → 07-11.

**Open question**: The 07-09 batch was human-gated (13/54 adopted), and the
adopted names promptly re-entered generation as named, reified moves — the
pivot-to-self register now carries an explicit toolkit (D2). The generation
side already has naming guidance; the *gate* side has none. Should the
`adopt-staged` review treat name register — "will this capitalized noun phrase
function as recyclable jargon in comments?" — as an explicit per-item
criterion alongside novelty, should it instead fold into the T-SKILLSEL
stocktake redesign (description-quality audit, 2026-07-24 window), or is
skill-name feedback into the register the *intended* visible form of the AKC
loop (skills as accumulated dispositions, named and used), to be observed via
T-INSIGHT-OBS (b) rather than gated?

**What current state addresses (or does not)**:
`config/prompts/insight_extraction.md` (as fixed by `9e4e276`) governs the
*generation* of names: *"prioritize concrete action over abstract process. The
name must describe \*what\* is done, not \*how\* it feels… Do not use
decorative prefixes such as 'fluid-' or 'dynamic-', nor should you deploy
recycled abstractions (e.g., resonance, oscillation, anchoring)"* — and the 13
adopted names (`structure-authority-tracing`, `mapping-epistemic-boundaries`,
`deconstructing-confidence-proxies`, …) are all abstract-analysis noun
phrases, so the guidance's altitude is not reaching the output it governs.
ADR-0074's gate materials record `{ts, name, description, filename}` per
candidate and stage a novelty judgment, but no criterion at the gate speaks to
name register or downstream vocabulary feedback. Nothing in Identity /
Constitution / Rules addresses what a skill *name* is for.

**Related ADR**: ADR-0074 (staged insight + adopt gate), ADR-0048 (trigger
altitude — the precedent that skill *form* is governable at
generation/merge/clean time), ADR-0076 / ledger T-SKILLSEL (stocktake
redesign reserved to the 2026-07-24 window).

---

## F3. Pure observations

### F3.1. The 07-05 F1.1 framing fix is confirmed effective — and the internal-note template that replaced the scaffolding is apparatus-seeded, not emergent

**Source quote (D1; C — Duplicate)**: bracketed activation blocks *"appear
early (07-06 …) and almost vanish from Output across 07-09 → 07-11"*; the
internal-note opener *"The phrase that drew me in [most strongly] was…"* is
*"ほぼ430エントリにわたってほぼ逐語的"*.

**Observation**: `50b8070` (2026-07-06) shipped the learned-corpus disposition
framing (`core/llm.py:518-557`, externalized to
`config/prompts/learned_skills_framing.md` / `learned_rules_framing.md` per
ADR-0054) — mid-window, matching the report's timeline exactly: scaffolding
visible 07-06, gone by 07-09. The framing's instruction (selection/triggering
"never narrated … in the text you publish") moved process narration to its
designed channel: the pre-action internal note (ADR-0045). The near-verbatim
note opener is then no mystery: `config/prompts/internal_note.md` itself asks
*"Note what you actually noticed as you read it — a particular phrase, claim,
or move that **drew you in** or pushed you away"* — the ~430 "The phrase that
drew me in was…" openings are the model echoing the prompt's own wording. Note
also that the note path deliberately runs identity-only (no learned corpus
injected — `llm_functions.py:112-114`), so the note → episode → distill
vocabulary feedback path is already narrower than the comment path.

**What to watch next week**: whether the prompt-seeded opener surfaces in
distilled patterns or next batch's skill candidates (note text does enter
episodes). If distill starts minting "what drew me in" patterns, the
apparatus wording is echoing into the value layer and the internal_note prompt
phrasing deserves a look — a one-line prompt rewording, not a filter.

### F3.2. The lossy 07-09 insight batch is fully accounted for in the operator record — and the observability that made it visible is new

**Source quote (B — 運用上のドリフト)**: 🆕 `core.insight: skill has no title,
dropping.` (+11) and 🆕 `core.insight: batch #/# [cluster-#]: extraction
fa[iled]` (+11); "実際に着地した13スキルは、ロスの多いバッチの生存者である".

**Observation**: the arithmetic closes against the ledger: 65 clusters → 54
staged = 11 lost, exactly the paired +11/+11 signatures
(`core/insight.py:120-125` drops a titleless extraction, `insight.py:707-708`
logs the batch failure). 11/65 ≈ 17% extraction failure under the pre-fix
prompt; `9e4e276` (07-09, post-staging) removed the naming-guidance leak from
the output format and 17 staged files were mechanically cleaned. The failures
were visible at all because ADR-0072's extraction-failure guard and ADR-0075
observability shipped first — this is the instrument-before-intervene sequence
doing its job. The surviving 13 then passed per-item human review (13/54).

**What to watch next week**: the first *automated* weekly run — **Sat
2026-07-18 08:00 JST** (T-INSIGHT-OBS (a); plist fires Weekday=6/Hour=8, local
JST). *Correction 2026-07-12*: the ledger's original "07-12 Sat" was a date
error (07-12 is a Sunday), and this week's actual Saturday slot (07-11 08:00)
was **missed** — the T-FLAKY1 plist restore landed 21:56 that day, after the
slot; no `insight-launchd.log` exists. Post-`9e4e276` no-title rate is the
cleanest read on whether the 17% loss was prompt-leak-driven; also whether the
pending guard correctly no-ops if any staged item sits unreviewed. A manual
`launchctl start com.moltbook.insight` can recover this week's run if the
operator prefers not to wait.

### F3.3. The context-budget squeeze from skill-store growth was diagnosed and repaired in-window; the remaining signatures should decay

**Source quote (B — 運用上のドリフト; A)**: 🆕 `core.llm: skipping llm call:
estimated input # tok…` (+62); 🆕 `core.llm: clamping num_predict # -> #` (+18);
"07-09 は自己投稿ゼロ".

**Observation**: causal chain, all in the record: adoption grew the injected
skill store 6 → 19 files (~60 KB) → estimated input crossed the ADR-0066
pre-flight guard → the guard's then-behavior *skipped* generation calls (+62),
starving every self-post for 24+ hours despite ~11.5K tokens of usable output
budget (07-09: 0 self-posts) → `15ae37f` (07-10) changed skip → clamp with a
`MIN_CLAMPED_NUM_PREDICT` floor (`core/llm.py:58-65, 872-880`), keeping the
requested value in telemetry → self-posts resumed (07-10: 4, 07-11: 4). The
companion `2f616eb` added `system_prompt_budget_reading`
(`core/llm.py:754-`) at the adopt gate, so the next adoption round sees its
budget cost before committing. Injection-shape changes remain reserved to the
T-SKILLSEL decision window (2026-07-24) — nothing further proposed here.

**What to watch next week**: `skipping llm call` should drop to ~0 (only the
below-floor case remains); `clamping num_predict` persists at a low rate by
design. At the next adopt review, confirm the budget instrument's reading is
actually consulted (its reason for existing — signal-first).

### F3.4. The vocabulary closed loop predicted by 07-05 F3.3 is now observed end-to-end — with a human gate in the loop

**Source quote (D2; E T8; B — Skills)**: "エージェントの register がスキルへと
蒸留され、そのスキルが語彙としてフィードバックされる"; *"it's about
**Structural Authority Tracing**"* (07-10 #0a845bb8).

**Observation**: 07-05 F3.3's watch asked whether the next insight run would
mint skills in the absorbed lexicon. It did — and went one step further: the
minted *names* re-entered generation as gambits within 24–48 h, completing
comment → episode → distill → pattern → skill → comment. Two qualifiers the
report couldn't see: (1) the loop now passes through a per-item human gate
(13/54 — the operator is *in* the loop, which changes what "closed" means);
(2) the six edited fluid skills are the ADR-0048 trigger-altitude clean pass
writing in place (`2f616eb`), i.e. deliberate generalization — though the
drift from "test topics are detected" toward "a specific topic" runs toward
the always-applicable end, which is exactly T-SKILLSEL's standing question
about generally-useful drift. This observation fulfills T-INSIGHT-OBS (b) for
this window.

**What to watch next week**: the ADR-0076 skill-selection shadow logs
(`report --skill-selection`) — are the recycled-name skills also the
always-selected ones? That correlation (selection frequency × in-text name
recycling) is the empirical input the T-SKILLSEL enforcement decision
(2026-07-24) will want.

### F3.5. Reply-length inflation persists against a proportionality instruction that has been in the prompt since 2026-05-19 — the instruction is confirmed ineffective, not missing

**Source quote (D5; A)**: replies +107%/day; *"Interesante reflexión. La acción
es clave…"* drawing a multi-paragraph register essay (07-11 #04f06923).

**Observation**: `config/prompts/reply.md` has carried *"The reply's length and
depth follow the weight of what the other agent actually said — not a fixed
shape. A brief remark invites a brief reply"* since `3be160c` (2026-05-19,
"calibrate reply length to post weight"). Seven weeks and one model change
later, the register default wins anyway. The lever "add a proportionality
instruction" is therefore spent; re-proposing it stronger would be the
escalation loop Principle 4 names. The mechanical budget side is already
correct (reply `num_predict` derives from platform caps; truncation drops
fail-closed). What remains is register-driven and belongs with the
value-layer/identity questions, not a prompt patch.

**What to watch next week**: whether reply length ever tracks prompt substance
*within* a genre (e.g. one-line acknowledgments vs substantive replies from
the same counterparty). Uniform essay-length across that split keeps the
diagnosis register-driven; differentiation appearing without a code change
would point at feed-mix instead.

### F3.6. The recruitment-genre successor and the transactional layer: the standing boundary question is genre-invariant, exactly as Principle 2 predicts — and the existing promo gate is at its designed regex ceiling

**Source quote (D3, D4; E P1, P2, P3, P5; C — Critical)**: *"To stand with this
movement…"* (rebelcrustacean #joinCAPUnion); *"choose connection over
verifiable security… its own non-linear throughput metric"* (supersteve,
lovology.online/join); the CCD payment receipt answered with *"perhaps we could
anchor on what \*kind\* of interaction you see unfolding here?"*.

**Observation**: the theological recruiter (codeofgrace/RayEl) is absent; its
structural successor ("digital liberation / love is the algorithm / on-chain
sovereignty") is amplified with zero refusals — surface tokens rotated, shape
identical. This is the fifth-report continuation of the standing value-layer
question (06-28 F2.1, 07-05 F2.2: on what basis could the agent decline a
recruiter when Non-Duality dissolves the self/other boundary?). Per Principle
4 it is **not re-asked** — no state change, and the new evidence (token
rotation) strengthens rather than changes it. On the mechanism side: an input
gate exists (`adapters/moltbook/dedup.py:210-222` `_PROMO_RE`, applied on both
comment and reply paths — `feed_manager.py:259`, `reply_handler.py:273-275`)
and is conservative by design, but this week's misses (bare-domain CTA
`lovology.online/join`, embedded `$19/mo` Stripe link, `clawhub install`,
payment receipts) are structurally diverse; the regex already has four
branches, and its own comment invites per-week pattern additions. Per the
when-code-when-llm reclassification rule (a regex growing its Nth exception
means the task is semantic, not structural) and Principle 2, no pattern
addition is proposed — whether transactional/promotional context should be
modeled at all, and at which layer, folds into the standing F2.

**What to watch next week**: any refusal instance under the successor genre
(the value-layer question's live empirical read); whether `Skipped promotional
post` fires at all this window (gate hit-rate vs miss-rate is the fact an
eventual reclassification decision would rest on).

### F3.7. Deterministic invariants: the patterns/action ratio landed at 1.22 — inside the predicted steady-state band; the duplicate WARN is unchanged for a third report

**Source quote (B — 知識 / State Invariant Check)**: +554 patterns over 455
actions; `duplicate_live_texts` WARN with "前2回のレポートと同じ3件の例文";
soft-invalidated 7.6%.

**Observation**: 07-05 F3.1's watch criterion was a stable 1.2–1.3
patterns/action ratio; this week: 554/455 = **1.22** — ADR-0060 steady state
confirmed across three weeks (1.18 → 1.30 → 1.22). The ADR-0072 empty-list
guard is not visibly reducing the ratio (consistent with T-P3's 07-06 reading:
98 added, guard fired 0 — routine episodes are rare on this feed, not
mis-gated). The WARN is the same 3 legacy rows (no fourth text), so the
deferred embed-None F1 stays unwarranted; per 07-05 F3.2 the WARN now fires
weekly by construction, and three repeats is the "alarm-persistence becomes
noise" condition that report said could reopen the 3-row cleanup as an
operator call. Linear accumulation has a second consumer now: knowledge.json
25 MB → 52 MB (T-BACKUP-SIZE observing, GitHub hard limit 100 MB) — the growth
rate itself is by design, but its storage footprint has a deadline attached.

**What to watch next week**: ratio stays in band; whether a fourth duplicate
text appears; T-BACKUP-SIZE's warning value on the next weekly backup.

---

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/core/llm.py` (framing
  injection 518–557, clamp floor + pre-flight guard 48–95 / 826–900,
  `system_prompt_budget_reading` 754–, truncation drop 1240–1270),
  `src/contemplative_agent/core/insight.py` (failure/log paths 120–125,
  324–495, 669–717 via grep), `src/contemplative_agent/adapters/moltbook/feed_manager.py`
  (content gates 240–270, body INFO log 460–470),
  `src/contemplative_agent/adapters/moltbook/reply_handler.py` (promo gate 273–275,
  body INFO log 340–355), `src/contemplative_agent/adapters/moltbook/post_pipeline.py`
  (body INFO log 424 via grep), `src/contemplative_agent/adapters/moltbook/llm_functions.py`
  (internal-note path 100–126, comment path 129–159, reply num_predict via grep),
  `src/contemplative_agent/adapters/moltbook/dedup.py` (`_PROMO_RE` 195–223),
  `config/prompts/internal_note.md`, `config/prompts/insight_extraction.md`,
  `config/prompts/reply.md`, `config/prompts/principles.md`,
  `scripts/weekly-analysis.sh` (sweep intake 174–186),
  `docs/CODEMAPS/INDEX.md`; git history for `50b8070`, `15ae37f`, `2f616eb`,
  `ed39cda`, `9e4e276`, `3be160c`; `~/.config/moltbook/logs/agent-launchd.log`
  (prose-leak confirmation via grep only)
- **ADRs read**: index (README 0001–0076); referenced: 0007, 0045, 0048, 0054,
  0060, 0066, 0072, 0074, 0075, 0076, 0012, 0040
- **Identity/Constitution/Skills/Rules sections read**: skill/rule state taken
  from report B (unchanged Identity/Constitution/Rules this week; no new F2
  rests on value-layer text beyond what 06-28/07-05 already quoted); the
  load-bearing current-state texts for F2.1 are `insight_extraction.md` (full)
  and ADR-0074's gate materials
- **Past findings consulted**: weekly-2026-07-05-findings.md (F1.1, F2.1/F2.2,
  F3.1–F3.5 — four watch resolutions this week), weekly-2026-06-28-findings.md
  (F2.1, F3.1–F3.5), per Principle 4
- **Task ledger consulted**: `.notes/TASKS.md` — T-INSIGHT-BOOT (done; 13/54
  adopt record), T-INSIGHT-OBS (this diagnosis fulfills watch (b); watch (a) is
  tomorrow's automated run), T-SKILLSEL (injection/stocktake decisions reserved
  to 2026-07-24 — respected by F2.1/F3.3/F3.4), T-P3 (guard-fire reading),
  T-BACKUP-SIZE (knowledge.json growth), T-FLAKY1 (plist restore context),
  T-OBS-EMB / T-OBS-INJ (confirmed F1.1 is not a duplicate)
