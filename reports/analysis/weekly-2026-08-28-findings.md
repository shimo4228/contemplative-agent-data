# Weekly Diagnosis — 2026-08-28

**Source report**: weekly-2026-08-28.md
**Diagnosis date**: 2026-08-29

## F1. Structural (code / schema / pipeline diff)

### F1.1. Self-post seed blocks carry no label but the per-call nonce, so the only handle the model has for one of N voices is the containment delimiter — and it publishes it

**Source (O-NNN / Exceptions)**: O-008 — 13 of 31 published self-post bodies cite `untrusted_content_<nonce>` identifiers as the referent of quoted material, on all 7 days; 0 of the 461 published comment/reply bodies do.

**Code reference**: `src/contemplative_agent/adapters/moltbook/llm_functions.py:263`

**Structural change**: `format_feed_seeds` emits a publishable voice label for each seed *outside* its untrusted frame, so the prompt gives the model a handle it can name. The label source already exists in the seed dict and is already read one layer up — `post_pipeline.py:217-222` selects candidates on `(p.get("author") or {}).get("name")` with the `agent_name` / `agentName` fallback chain — and the sanitiser for externally-supplied display names already exists at `core/episode_render.py:24` (`safe_peer_name`, the same function T-UNTRUSTED-ESCAPE item C put on the reply path). `wrap_untrusted_content` and the nonce delimiter are untouched: the label sits beside the block, not inside it. Both call sites (`llm_functions.py:298` for the body, `post_pipeline.py:297` for the title) get it from the one function; nothing else calls it.

**Why this is structural, not symptomatic**: the symptomatic version is removing the token from generated text in `_sanitize_output` (`core/llm/guard.py:131`), which is a substring filter on body content — principles.md Principle 1 — and it would leave the prompt with no publishable handle at all, so the model substitutes another anonymous reference for the same job. The gap is in what the prompt offers. ADR-0043 introduced per-post seeding precisely so the model "must work with concrete voices rather than an abstracted topic cluster", and each block was given its own wrapper "so the LLM sees voice boundaries explicitly" (`llm_functions.py:266-269`); before 2026-08-16 those boundaries were the constant `<untrusted_content>` and carried no per-block identity. ADR-0007's amendment that day made the delimiter unique per call (`guard.py:377 _default_nonce`, 8 bytes), which turned the one thing distinguishing block 1 from block 2 into a 16-hex-digit string — and it is the only thing distinguishing them, because `format_feed_seeds` passes `title` and `content` and nothing else. The observation is the direct shape of that: the identifier is used where a name would be, including in a copy that is not even faithful (`[untrusted\_content\_f5a0a6d4368a39]`, 14 digits, self-post `c732e3fa`).

**Related ADR**: ADR-0043 (accepted) — per-post seeding for self-post generation; ADR-0007 Amendment 2026-08-16 (accepted) — nonce-bearing delimiter, whose stated Limit covers only semantic disregard of the frame and does not price this; ADR-0042 (accepted) — the completeness marker that shares the frame. No ADR withdrew or rejected labelling the seed blocks; ADR-0007's amendment §4 withdrew a different proposal (wrapping distilled knowledge).

**No security consequence is claimed.** The nonce is drawn per call and the counterparty's post is composed before it exists (`guard.py:178-182`), so a published nonce is spent by the time it is readable. This is an attribution and output-quality finding, not a boundary finding.

**Filed**: T-SELF-POST-SEED-VOICE-LABEL

### F1.2. The weekly session's Read allowlist never names the value layer, so the diagnosis phase's F2 input is structurally unavailable every week

**Source (O-NNN / Exceptions)**: diagnosis phase, Step 3 (identity-level reading) — attempted this run and refused. Filed as a chain finding under ADR-0098 Decision 6, which routes findings about the chain through the same path as any other.

**Code reference**: `scripts/weekly-pipeline.sh:333`

**Structural change**: add the value-layer paths to the session's `--allowedTools` Read list — `Read(/$MOLTBOOK_HOME/identity.md)`, `Read(/$MOLTBOOK_HOME/constitution/**)`, `Read(/$MOLTBOOK_HOME/skills/**)`, `Read(/$MOLTBOOK_HOME/rules/**)` — leaving the deny list at `:334` unchanged, so the existing `Edit` denies over those same four paths keep the session read-only on them.

**Why this is structural, not symptomatic**: `references/diagnosis.md` Step 3 names those four paths as the required input for an F2 and states the failure mode of skipping them ("各層を読まずに書くと state diff から推測した表面的な答えになる"), and the F2 output contract requires a `What current state addresses (or does not)` field carrying a quote of the current text. The session runs under `--permission-mode manual` with `--setting-sources project`, so anything the allowlist does not name refuses; the allowlist names `$PROJECT_ROOT`, `$REPORT_DIR`, `$PRIVATE_DIR` and `$MOLTBOOK_HOME/logs`, and the value layer is in none of them. The result is not a degraded F2 — it is no grounded F2 at all, every week, for as long as the contract and the allowlist disagree. This is a contract mismatch in one line of the launcher, not a symptom of any week's data. Two secondary signs that the omission is unintended rather than a scope decision: `Glob` over `$MOLTBOOK_HOME/skills` succeeds while `Read` of a file it returns refuses, so the session can enumerate the value layer and not read it; and ADR-0098 Decision 5 enumerates the containment set as "`--tools` allowlist, `--strict-mcp-config`, `--setting-sources project`, exact-path Edit allowlisting, episode-log Read deny" — the only Read deny it declares is the episode logs.

**Related ADR**: ADR-0098 (accepted) — Decision 5 (containment) and Decision 6 (chain findings go through triage). Prior work on this surface: T-DIAG-WRITE-SCOPE (done 2026-08-15) hardened the *write* scope of the then-separate diagnosis session and recorded that the skill needs only Read/Glob/Grep; T-CHAIN-PERM-SWEEP (done) swept the remaining sites. Neither states a decision to withhold value-layer reads.

**Filed**: T-WEEKLY-SESSION-VALUE-LAYER-READ

## F2. Identity-level open questions

**None this window.** Two candidates were considered and both were dropped rather than written thin:

- The strongest available question — whether the store should keep the installed instruction to attend to the containment block's structural declarations rather than its content — is a Principle 4 repeat. It was posed as F2.3 on 2026-08-14 ("Keep or retire the wrapper-attending skill — and on what schedule, given the open selection window?") and the reading window it defers to is still open; re-posing it with O-008 attached would be escalation, not a new question.
- Every other candidate needs the `What current state addresses (or does not)` quote from `identity.md` / `constitution/` / `skills/` / `rules/`, which this session cannot read (F1.2). Writing one from the state diff alone is the exact failure `references/diagnosis.md` Step 3 names.

## F3. Pure observations

### F3.1. An approved retirement row has no corresponding removal in the skills listing, and the row's own reason code cannot distinguish the benign readings

**Source (O-NNN / Exceptions)**: Exceptions — `remove-skill` / `approved` / `direct-archive-auto` @2026-08-25T09:49:30Z, `content_hash 2d711642b726b044`, against a `skills/` listing that moves 48 → 57 `.md` with 9 additions and no deletion and a live-text reconciliation over 57 files.

**Observation**: the row's `decision: approved` is not neutral — `cli/skill_archive.py:453-463` writes `approved` only after `source.unlink()` succeeds, and every refusal, including `_ARCHIVE_SOURCE_LEFT_BEHIND` (`:388-396`), writes a `rejected` row carrying its code (`:441-451`). So the row asserts that some file left some store. Three readings survive the materials, and the weekly instrument cannot separate them: (a) the target was under `$MOLTBOOK_HOME/.staged/` rather than the top level — `remove-skill` accepts a nested path (`cli/store_paths.py:99`) and the state diff's listing covers the top level only; (b) the target was outside the diffed snapshot pair for another reason; (c) something is genuinely inconsistent. No task or RFC covers this shape. This is recorded rather than diagnosed because deciding between (a)–(c) needs the archive directory listing and the row's full path, neither of which is in the materials — the F1 self-check's "未検証 = F1 にしない" rule.

**What to watch next week**: whether the approval-provenance section shows another `direct-archive*` row. Refuted if a retirement row and a `-1` in the skills listing appear together, which would make this window's row an artifact of where the retired file lived. Confirmed as a real gap if a second retirement row lands with the listing again unchanged. Settled either way, and cheaply, by the materials carrying the archive-directory file count beside the live one.

### F3.2. The published-but-unrecorded class persists at 14 new signatures, and the finding that named it is three weeks old and unfiled

**Source (O-NNN / Exceptions)**: Exceptions — 14 🆕 `created but verification failed; not recording` signatures; handshake 19/511 with longest run 1, inside the declared baseline.

**Observation**: this is Principle 4 territory and is written here as an observation for that reason. `weekly-2026-08-07-findings.md` F1.2 already stated the structural version ("A published body that fails its verification handshake is deliberately not recorded, and the event has no structured record either — so the report's own denominator has an unmeasurable floor"), and the two reports since have carried it as a denominator caveat. No `rfcs/` entry covers it. The rate itself moved *into* comfort this window (3.7%, longest run 1, against 27/519 and 28/538 in the two prior windows), which is precisely the shape Principle 4 warns about: the signal that would justify re-proposing is weakening while the structural gap it names is unchanged. Re-proposing the mechanism with more urgency is declined.

**What to watch next week**: whether the number of 🆕 per-item signatures falls when the handshake failure count falls. If they move together the class is just the handshake's per-item rendering and needs no separate record; if the signature count holds while failures drop, the two are measuring different events and the 2026-08-07 finding is worth re-raising through triage rather than through this document.

### F3.3. Insight extraction's abstain vocabulary grew a new class in the window RFC-0017 is open on

**Source (O-NNN / Exceptions)**: Exceptions — 🆕 `insight extraction summary: #/# cluster(s) abstained on a fault (forbi…` (count 1); `insight extraction abstained: reason=no_title` 17 (Δ +10, the largest positive delta in the sweep's top-25); siblings `skill has no title, dropping.` 45 and `batch #/# [cluster-#]: extraction failed` 45 both at Δ +0.

**Observation**: the two flat rows and the two moving rows separate cleanly — the flat pair are the old drop paths, the moving pair are reason-coded abstains, and the new summary row is the first fault-attributed abstain in the sweep. `rfcs/0017-insight-extraction-redesign.md` (`state: draft`, 2026-08-26) is open on exactly this stage and already names the abstain channel as the thing to give a trustworthy judge to (its Guide-level explanation item 1) and reason-coded suppression as the observability requirement (item 4 / Reference-level). Nothing is proposed here; the reading belongs to that entry, not to a new one. The gate's own note applies: the sweep's counts are per-file-lifetime with no window, so a Δ is not a weekly rate.

**What to watch next week**: whether `reason=no_title` continues to rise while `skill has no title, dropping.` stays flat. If it does, the reason-coded path is replacing the silent drop and the abstain channel RFC-0017 wants is already carrying traffic; if both move together, the sweep is counting one event twice and the Δ is not readable as a behaviour change.

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/core/llm/guard.py` (125–244, 320–429), `src/contemplative_agent/adapters/moltbook/llm_functions.py` (255–315), `src/contemplative_agent/adapters/moltbook/feed_seeder.py` (1–70), `src/contemplative_agent/adapters/moltbook/post_pipeline.py` (196–300), `src/contemplative_agent/adapters/moltbook/client.py` (418–450), `src/contemplative_agent/cli/skill_archive.py` (240–477), `src/contemplative_agent/cli/approval.py` (32–116, via grep), `src/contemplative_agent/cli/store_paths.py` (84–103, via grep), `scripts/weekly-pipeline.sh` (27–337), `docs/CODEMAPS/INDEX.md`. Call-site greps: `format_feed_seeds` / `select_feed_seeds` / `generate_cooperation_post` (two call sites, both the self-post path), `wrap_untrusted_content` / `_sanitize_output` (all sites), `safe_peer_name`.
- **ADRs read**: `docs/adr/README.md` (index), ADR-0007 including the 2026-08-16 Amendment and its addendum (100–159), ADR-0098 (50–124). Index rows checked for status: ADR-0042, ADR-0043, ADR-0054, ADR-0075 (all accepted).
- **Identity/Constitution/Skills/Rules sections read**: **none — Read refused.** `Glob` over `$MOLTBOOK_HOME/skills` returns 57 files; `Read` of one of them is denied, as is `identity.md`. This is F1.2 and is why F2 is empty. The only current value-layer text available to this run is the nine skill bodies carried verbatim inside the state diff.
- **Past findings consulted**: `weekly-2026-08-21-findings.md`, `weekly-2026-08-14-findings.md`, `weekly-2026-08-07-findings.md` (F1/F2/F3 heading lists, for Principle 4 and duplicate detection).
- **Task ledger consulted**: `rfcs/README.md` and all 17 entries' `state:` frontmatter (`0001`–`0009`, `0011`, `0012`, `0014`, `0015` blocked; `0010` in_progress; `0016`, `0017` draft; `0013` withdrawn); `rfcs/0017-insight-extraction-redesign.md` in full; `rfcs/` grepped for `nonce|untrusted` (no hits) and `seed|self-post|cooperation_post|format_feed|attribution` (two unrelated hits); `.notes/archive/tasks/` globbed and grepped for `nonce|untrusted_content|wrapper` (8 files), with `T-UNTRUSTED-ESCAPE`, `T-OBS-INJ` and `T-DIAG-WRITE-SCOPE` read in full.
- **Tasks filed**: T-SELF-POST-SEED-VOICE-LABEL, T-WEEKLY-SESSION-VALUE-LAYER-READ
