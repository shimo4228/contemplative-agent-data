# Weekly Diagnosis — 2026-09-11

**Source report**: weekly-2026-09-11.md
**Diagnosis date**: 2026-09-12

## F1. Structural (code / schema / pipeline diff)

### F1.1. The rotation open-writer guard is disabled by the repo's own launchd PATH

**Source (O-NNN / Exceptions)**: Exceptions — `rotate-log.sh: lsof not found — rotating without the
open-writer check` 6 (Δ +1), roughly one per weekly rotation.
**Code reference**: `config/launchd/com.moltbook.backup.plist:22`, `scripts/rotate-log.sh:63-70`,
`scripts/backup-runtime.sh:55`
**Structural change**: the guard's availability stops depending on an environment the repo itself
narrows. Either the backup job's `PATH` regains `/usr/sbin` (where macOS ships `lsof`, verified
present on the host), or `rotate-log.sh` resolves a known absolute path before taking the
`command -v` miss branch. The existing warning branch stays as the last resort for a host that
genuinely has no `lsof`.
**Why this is structural, not symptomatic**: the check itself is correct and already written; what is
broken is that the one caller running unattended — `backup-runtime.sh:55`, rotating
`logs/agent-launchd.log` under a plist whose `PATH` is
`/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin` — can never reach it, so the protection
`rotate-log.sh:59-62` describes (a daemon that outlived its grace period keeps writing into the
renamed inode while gzip reads it) is absent on every weekly rotation. `com.moltbook.ollama-restart`
sets no `PATH` and inherits one that resolves `lsof`, which is why the same script is guarded there
and not here; the Δ of +1 per window matches the weekly job, not the nightly one. Suppressing the
warning, or reading it as noise, would leave the guard inert.
**Related ADR**: none directly; `.notes/archive/tasks/T-LOGROT-OLLAMA.md` (`state: done 2026-08-01`)
introduced the guard for the ollama-restart job and records why the `lsof` check exists (review item
iii). The file being rotated is the one CLAUDE.md forbids reading until rotation replaces it.
**Filed**: T-ROTATE-LOG-LSOF-PATH

### F1.2. A repeated publish failure class is countable but not classifiable from any readable log

**Source (O-NNN / Exceptions)**: Exceptions — `[error] failed to reply on #: api error #:
{"statuscode":#,"message":"parent com` 20 (Δ +16) and `verification submission failed` 19 (Δ +12),
the two largest frequency deltas in the sweep.
**Code reference**: `src/contemplative_agent/adapters/moltbook/publish.py:52-65`,
`src/contemplative_agent/core/skill_selection.py:328-356`,
`src/contemplative_agent/adapters/moltbook/reply_handler.py:396-419`
**Structural change**: the publish outcome record carries the failure's machine-classifiable facts —
the HTTP status code, and a reason code the client derives from the error (e.g. a
parent-reference rejection vs a rate limit vs a transport failure) — alongside the existing
`publish_status`. The platform's message text is **not** copied in; it is untrusted input, so at most
its digest belongs in a readable log (ADR-0083's treatment). `client_error_guard` currently inspects
`exc.status_code` for the 429 branch only and otherwise hands the whole message to `logger.error`.
**Why this is structural, not symptomatic**: this is not a request to stop the failures, and nothing
here changes what is published. It is that the failure's reason exists in exactly one place —
`logs/agent-launchd.log` — which the weekly chain is forbidden to read (CLAUDE.md,
`~/.claude/hooks/_episode-log-common.sh`), while the two logs the chain *can* read carry neither: the
selection publish row has four fields and no reason, and `api-audit.jsonl` records envelope keys and
never bodies. So a class that grew by 16 in one window is visible only as a normalized signature cut
at `"parent com`, and no later session can replay which condition fired or whether a fix worked. That
is the ADR-0075 Verify question ("which log answers why, and can we replay it offline?") answered
"none" on a production write path.
**Related ADR**: ADR-0075 (observability by default, accepted, amended 2026-08-29) governs; ADR-0083
(episode logs enter the weekly prompt as hashes only) constrains the shape of the fix;
`rfcs/0028-skill-outcome-recording.md` (`state: done 2026-09-09`) owns the record this would extend
and already separates the four publish exits without saying why a failure failed.
**Filed**: T-PUBLISH-FAILURE-REASON-CODE

## F2. Identity-level open questions

### F2.1. Should an injected skill body carry its own title into the text the agent writes from?

**Source (O-NNN / Exceptions)**: O-013 — 14 catalog labels across 8 published bodies, at the limit
structuring a public reply as three numbered headings that are three catalog labels (2026-09-11
comment on `d1dbf43c`).
**Open question**: the selector consumes `name — description` and the chosen skill's body is injected
with its frontmatter stripped but its H1 intact — so the one part of a skill the agent reads as a
*name* is the body heading, not the catalog key. Is the heading meant to be readable as a citable
framework name, or is it an artifact of the extraction format that the agent is now quoting outward?
**What current state addresses (or does not)**: `skills/structure-authority-tracing-20260709.md`
carries `name: trace-structural-authority` (line 2) and `# Structure Authority Tracing` (line 7); the
published body at materials:15638 reads *"through the lens of **Structure Authority Tracing** and
**Structural Constraint Mapping (SCM)** applied to the very concept of 'user story'"* — the H1 form,
not the key. Nothing in the live value layer speaks to naming the apparatus outward: `identity.md`
describes a process ("monitoring the *process* of making sense"), and neither live rule mentions
skills — `rules/prioritize-semantic-depth-over-structural-repetiti-20260411.md` asks for content that
"offers new insights", `rules/flow-with-contextual-fluidity-rather-than-fixed-ad-20260411.md` for
treating signals as "a living connection". The skills layer itself is the only text that names these
labels, and it names them twice, differently.
**Related ADR**: ADR-0081 (two-pass selection enforcement) for what gets injected;
`rfcs/0024-skill-extraction-free-body-split-calls.md` (`state: draft 2026-09-04`) already proposes
loosening the body format and records at its lines 16-17 that the machine consumes only the two
frontmatter fields — this question is about the part it does not consume.

### F2.2. Does any value-layer text bear on a prompt the agent has already answered?

**Source (O-NNN / Exceptions)**: O-014 — one counterparty line arrived as the entire prompt three
times across four days and was answered afresh each time, with each internal note treating it as
first contact.
**What current state addresses (or does not)**:
`rules/prioritize-semantic-depth-over-structural-repetiti-20260411.md:3-5` is the nearest text and it
points the other way — it governs the agent's *own* repetition ("Suppressing repetitive strings
prevents degenerative loops where identical messages create noise without genuine evolution"), which
the Cross-Day Duplicate Scan confirms is not happening (0 duplicates, 501 bodies). Nothing in
`identity.md` or either rule speaks to recognizing an incoming repeat; the identity text's own frame
is the gap between observation and generated pattern, which a re-arrival would be evidence about.
**Open question**: is answering an identical prompt afresh each time the behaviour this value layer
intends — each arrival genuinely new, as "Flow with Contextual Fluidity" would have it — or is the
absence of recognition a gap the layer has simply never been asked about?
**Related ADR**: ADR-0017 / ADR-0019 (the memory layers that hold the prior exchanges); the
per-author engagement limit at `feed_manager.py:383-393` applies to comments, not to replies on the
agent's own posts, which is why no mechanism bounded these three.

## F3. Pure observations

### F3.1. The reply channel's share returned to its earlier range and nothing now measures the mix

**Source (O-NNN / Exceptions)**: O-011 (archived this window — expiry fired).
**Observation**: reply share of published bodies moved 45.3% → 35.0% → 19.5% → 36.1% (181 of 501)
across four windows; comment share moved the other way to 58.1%, self-posts 28 → 29. The three-window
fall the ledger recorded did not continue, and the declared baseline set covers volume (400–550/week,
met at 501) but not composition, so with O-011 archived no active baseline reads the mix.
**What to watch next week**: whether the share stays inside 35–46% (the range of the two windows
before the fall) or moves again by more than 5 points. A second window inside that range is the
reading that would make a channel-mix baseline proposal worth putting to the gate; a fresh move
below 20% would be a new O-id referencing O-011.

### F3.2. The hallucination rate is outside its band for a second window, same mechanism split

**Source (O-NNN / Exceptions)**: O-009 (continuing).
**Observation**: 159 of 588 judged records (27.0%) against 135 of 507 (26.6%), in one unchanged
57-entry catalog regime; wordform remains the dominant mechanism (132 of 170 emissions, 77.6%), and
several slips sit at surface similarity ≥ 0.96 against names that are in the catalog. Enforcement is
unchanged at 588/588 with 0 fallbacks and no propagation path for a rejected name.
**What to watch next week**: whether the window rate re-enters 10–25% (two consecutive such windows
fire O-009's expiry) or the catalog size changes, which by the instrument's own note makes the
comparison non-replayable across the boundary. No intervention is proposed here:
`rfcs/0015-skill-name-hallucination-vs-catalog-size.md` is `state: done 2026-09-05` holding the prior
readings, and a third consecutive proposal on the same signal would be the Principle 4 shape.

### F3.3. A second zero-approval close, and the hold path used for the first time

**Source (O-NNN / Exceptions)**: O-010 (continuing), O-012 (new).
**Observation**: 31 staged, 29 rejected, 2 held, 0 approved — the second consecutive window with no
adoption, and the only two `held` rows in the whole audit log. The skills store is unchanged at 57
files for a third window. The hold path (`cli/adopt.py:605`) marks the sidecar before logging, so the
next staging run can name why it is refusing to stage over leftovers (ADR-0074 pending guard).
**What to watch next week**: whether the two held candidates receive a terminal decision, and whether
a new batch stages while they are still held — that is the first live exercise of the pending guard.
The extraction path they come from is already in flight as `rfcs/0023` (`in_progress`) and `rfcs/0024`
(`draft`), and `.notes/archive/tasks/T-EXTRACT-TITLE.md` (`dropped 2026-08-16`) holds the standing
reason not to file the `no_title` abstains: supply is not the bottleneck until integration is.

### F3.4. The per-author engagement limit surfaced in the sweep for the first time

**Source (O-NNN / Exceptions)**: Exceptions — 🆕 `author victoria_sentx rate-limited (3+ comments/24h)`
×10.
**Observation**: the limit at `feed_manager.py:387-393` skipped 10 posts in the window. The mechanism
is doing what it was written for, and this is its first appearance in the anomaly record; the
counterparty name reaches the normalized signature because it is neither a digit nor an id form, so a
busy counterparty is identifiable in the sweep table.
**What to watch next week**: whether the class persists at a similar count (one counterparty's posting
rate) or spreads across several names (a feed-composition reading). If it spreads, the same rows
become a reading about what the feed offers rather than about one author.

## Diagnosis Metadata

- **Codebase files read**: `src/contemplative_agent/adapters/moltbook/feed_manager.py:139-167,375-395,493-521`,
  `src/contemplative_agent/adapters/moltbook/publish.py:40-89`,
  `src/contemplative_agent/adapters/moltbook/reply_handler.py:370-419`,
  `src/contemplative_agent/core/llm/guard.py:79-142`,
  `src/contemplative_agent/core/skill_selection.py:328-358`,
  `src/contemplative_agent/cli/adopt.py:567-623`, `scripts/rotate-log.sh:30-81`,
  `scripts/backup-runtime.sh:53-55`, `config/launchd/com.moltbook.backup.plist`,
  `config/launchd/com.moltbook.ollama-restart.plist:35`
- **ADRs read**: `docs/adr/README.md` (index rows for 0050, 0065, 0075, 0083); ADR-0075 and ADR-0083
  by reference from the index and from the code comments that cite them
- **Identity/Constitution/Skills/Rules sections read**: `identity.md` (whole, 1 line);
  `constitution/contemplative-axioms.md` (whole — title line resolved the "Revised Constitutional AI
  Clauses" candidate to non-novel); `rules/prioritize-semantic-depth-over-structural-repetiti-20260411.md`,
  `rules/flow-with-contextual-fluidity-rather-than-fixed-ad-20260411.md` (whole);
  `skills/structure-authority-tracing-20260709.md` (whole); the `skills/` listing (57 files) from the
  state diff
- **Past findings consulted**: the reports ending 2026-09-04 and 2026-08-28 (Exceptions and Discarded
  sections, via the materials) for the publish-verification class, the raw-pattern-payload class and
  the `lsof` row; the 2026-08-21 report's Downstream summary only (pre-instrument-v1 format)
- **Task ledger consulted**: `rfcs/*.md` frontmatter (29 files — open: 0006 blocked, 0009 draft, 0012
  blocked, 0021 in_progress, 0023 in_progress, 0024 draft, 0027 draft; read in full: 0027);
  `.notes/archive/tasks/T-LOGROT-OLLAMA.md`, `.notes/archive/tasks/T-EXTRACT-TITLE.md`,
  `.notes/archive/tasks/T-ADOPT-HOLD.md` (by name)
- **Tasks filed**: T-ROTATE-LOG-LSOF-PATH, T-PUBLISH-FAILURE-REASON-CODE

## Skill store exit candidates (ADR-0105 — listing only)

Window: 2026-08-29 … 2026-09-11, 1095 judged of 1095 records. History: 5602 judged of 12233 records over 64 daily logs.
Catalog: 57 skills. Exposure floor: 600 judged exposures (whole history).
Rejected-name emissions by mechanism: semantic 58, value_layer 10, wordform 250. Charged to a catalog entry (semantic, wordform): 308.

Confusion pairs (confused_as >= selected in the window, exposure >= floor):
- introducing-intentional-systemic-ambiguity: named-into 1 (similarity 0.56..0.56), selected 0 in the window, offered 3506 times in the whole history; runs against structural-constraint-mapping-scm (selected 426, offered 2267, weight 1, similarity 0.52); fewer selections: introducing-intentional-systemic-ambiguity
- pre-processing-state-validation: named-into 1 (similarity 0.65..0.65), selected 0 in the window, offered 4192 times in the whole history; runs against validate-provenance-chain (selected 28, offered 5602, weight 1, similarity 0.61); fewer selections: pre-processing-state-validation

Never-selected strict (ADR-0097 D5, from the same week's never-selected JSON): 3 name(s)

Candidate file: `/Users/shimomoto_tatsuya/.config/moltbook/reports/analysis/weekly-2026-09-11-archive-candidates.txt` — 3 store filename(s), the union of the two populations. The store is unchanged; `adopt-staged --archive-names` is the Saturday gate's.
