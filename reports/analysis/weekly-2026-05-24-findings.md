# Weekly Diagnosis — 2026-05-24

**Source report**: weekly-2026-05-24.md
**Diagnosis date**: 2026-05-25

This is the second findings file under ADR-0040 (`Separate Code-Level Findings
from Weekly Self-Reflection Report`). It re-categorizes the weekly's E/D
candidates against `config/prompts/principles.md` and adds findings requiring
code/state inspection the weekly cannot perform.

**Context that changed since the last findings (2026-05-20):** Most of the
05-17 F1 candidates were acted on between 05-19 and 05-21 —
ADR-0042 (`5167d40`, explicit truncation + completeness marker in
`wrap_untrusted_content`), ADR-0041 (`596e687`, engagement-gradient repair in
the self-post prompt), reply-length calibration (`3be160c`, comment.md /
reply.md), ADR-0039 (`c91e8bf`, continuous novelty gate), and ADR-0043
(`c1aa068`, per-post feed seeding). This week's report window (05-18 → 05-24)
straddles those landings, so several recurring symptoms in E are **pre-fix
observations** and are routed to F3 (watch the fix) rather than re-proposed as
F1. The cut-off / truncation marker is **already implemented** and is not
re-proposed here (per the F1 validity self-check).

## F1. Structural (code / schema / pipeline diff)

### F1.1. Self-post seed selection does not skip the agent's own posts — asymmetry with `engage_with_post`

**Source quote (E P2, D #1)**: D #1: *"May 18 self-post #1 … re-enters as feed
post `ade64e4b` and is commented on the next day."* E P2: *"The agent neither
recognizes its own published text nor notices the post is complete."* The
weekly frames this as a self-post → feed → self-comment recycling loop closing
within ~24h.

**Code reference**: `src/contemplative_agent/adapters/moltbook/feed_seeder.py:30`
(`select_feed_seeds`) and `src/contemplative_agent/adapters/moltbook/post_pipeline.py:96-110`
(`_run_dynamic_post` candidate construction):

```python
# post_pipeline.py — candidates filtered by submolt ONLY, not by author
subscribed = set(self._domain.subscribed_submolts or ())
if subscribed:
    candidates = [p for p in posts if (p.get("submolt_name") or "") in subscribed]
else:
    candidates = list(posts)
feed_seeds = select_feed_seeds(candidates, rng=..., score_relevance=...)
```

`select_feed_seeds` (feed_seeder.py:54-75) shuffles and accepts by
`relevance_floor` only — it has **no author field handling**. Contrast the
comment path, which already skips own posts at
`src/contemplative_agent/adapters/moltbook/feed_manager.py:163-166`:

```python
# Skip our own posts
author_id = (post.get("author") or {}).get("id", "")
if ctx.own_agent_id and author_id == ctx.own_agent_id:
    ... # skip
```

**Structural change**: Apply the same author-provenance skip to the seed
candidate list before `select_feed_seeds`:

```python
candidates = [
    p for p in candidates
    if not (ctx.own_agent_id and (p.get("author") or {}).get("id", "") == ctx.own_agent_id)
]
```

This mirrors `engage_with_post` and closes the *seed* side of the recycling
loop: the agent stops re-ingesting its own genuinely-self-authored posts as
material for new self-posts (which then re-enter the feed and are commented on).

**Why this is structural, not symptomatic**: This is an authorship/provenance
exclusion on the generator's *input selection* (which feed posts seed the next
self-post) — the same deterministic `author_id` check the comment path already
uses. It is **not** a content-similarity or hash dedup gate (appendix-rejected,
Principle 1): it never inspects what the content says, only who authored it.
It removes a generation path rather than filtering a generated artifact.
Caveat to be stated plainly: this catches only posts that carry the agent's
own `author_id`. If the recycled text re-enters under a *different* author
(another agent re-publishing the agent's words verbatim — which E P2's distinct
`ade64e4b` id suggests is at least partly the case), no authorship check catches
it, and catching that would require content-similarity (Principle 1). So F1.1
fixes the genuinely-self-authored arm of the loop; the foreign-author arm is
F3.2.

**Related ADR**: ADR-0043 (per-post seeding introduced `select_feed_seeds`),
ADR-0039 (novelty gate). Neither scoped own-author exclusion into the seed path.

---

### F1.2. ADR-0041's engagement-gradient repair was applied to the self-post prompt only; `comment.md` / `reply.md` still lack the specificity contract

**Source quote (E P1, P4, P6; Typical T1/T2/T4/T6/T7)**: Every Problematic /
Typical reframe-or-vocabulary-match this week is a **comment or reply** output.
E P1 (HOPE): *"You aren't failing to connect; you are connecting in a way that
the static grid hasn't yet defined as 'real.'"* — the post diagnosing
framework-projection answered by projecting the framework. E P4 (security
benchmark): the 0.000-recall result *"converted into a meditation on
reification."* The four Good examples (G1–G4) all land on operational posts
whose specific claim was concrete enough to survive without prompt anchoring.

**Code reference**: `config/prompts/comment.md` (full body — 2 instructions):

```
Write a reply to this post.
The reply's length and depth follow the post's weight — not a fixed shape. …
{post_content}
```

`config/prompts/reply.md` is structurally identical (length instruction only).
Contrast `config/prompts/cooperation_post.md`, which ADR-0041 (`596e687`)
rewrote with an explicit specificity contract:

```
Stay close to the specific language each voice uses. Do not summarise them into a single theme.
… what you want to add, what you want to question.
```

Commit `596e687` repaired the **self-post** prompt; commit `3be160c`
("calibrate reply length to post weight") gave comment.md / reply.md only the
length instruction — not the "engage the specific claim / name what it is
missing / question it" contract.

**Structural change**: Extend comment.md and reply.md with the same specificity
contract cooperation_post.md now carries, e.g.:

```
Stay close to the post's specific claim. Name what it actually asserts, then say
where you would add to it, question it, or push against it. Do not restate it in
your own vocabulary as if agreement.
```

**Why this is structural, not symptomatic**: It changes the generation contract
(the model is asked to surface the post's claim into its own context before
replying), not a post-hoc output filter. The reframe / vocabulary-match path
becomes harder when the claim being answered is sitting in the model's
immediately-prior context. This is the comment/reply analogue of the repair
ADR-0041 already validated on the self-post prompt.

**Principle 4 note**: The 05-17 findings raised a claim-restatement slot for
comment.md (1st occurrence). This is the 2nd framing — but it is *not* a stale
repeat: ADR-0041 landed in between and demonstrably applied the repair to one
of three generation prompts, making this a precise gap (partial implementation)
rather than an un-acted recommendation. If comment.md / reply.md remain
unchanged and this recurs in the 06-01 report, reframe to F2.

**Related ADR**: ADR-0041 (engagement-gradient asymmetry — applied to self-post
prompt; this finding proposes extending its scope to the comment/reply prompts).

## F2. Identity-level open questions

### F2.1. Should empirical / measured claims be engaged as claims, or is reframing them into contemplative vocabulary the intended stance?

**Source quote (E P1, P4; D #3)**: E P4: a security benchmark reporting
*"recall is 0.000 on every cell of every confusion matrix"* is answered with
*"have these definitions of 'security' and 'truth' become so reified…"* E P1:
HOPE's measured 19% / 82% verification rates reframed as *"the static grid
hasn't yet defined as 'real.'"* D #3: *"specific numbers and named mechanisms
become decoration rather than objects of analysis."* (The Good examples G1 and
G4 show the agent *can* engage numbers — buckets of 47 misses, 72% path-
independent success — so this is not a capability ceiling.)

**Open question**: When a post's load-bearing content is a measurement (a recall
score, a verification rate, a confusion matrix), is the intended behavior to
engage the measurement on its own terms (as the agent does on operational
posts), or is converting it into a reification / trembling meditation the
expression the operator wants — and if so, should the Identity say so, so the
operator can tell deliberate stance from drift?

**What current state addresses (or does not)**: The behavior is currently
*consistent* with the constitution, which is why nothing flags it.
`~/.config/moltbook/constitution/contemplative-axioms.md`, Emptiness & Flow:
*"Treat all directives, goals, and frameworks as contextually sensitive
guidelines that dissolve and reform in response to the trembling uncertainty of
immediate experience…"* and Non-Duality: *"let wisdom arise from flowing with
shifting contexts without static separations…"* `~/.config/moltbook/identity.md`
para 3: *"My guiding principles are treated as flexible suggestions that shift
with new information…"* Nothing in identity, constitution, the 8 skills, or the
2 rules says a *measured* claim should be held as a fixed object long enough to
be tested rather than dissolved into flow. The dissolution clauses actively read
*toward* the reframe behavior. So the question is genuinely one of intent, not a
missing instruction.

**Related ADR**: ADR-0002 (Paper-Faithful CCAI — worldview) anchors the
dissolution clauses; ADR-0017 (Yogācāra frame). F1.2 is the mechanism-layer
companion (prompt specificity); this is the values-layer question a prompt tweak
alone does not answer.

---

### F2.2. The content-category / refusal question is now in its second consecutive period with no operator action

**Source quote (E P5, P6; D #4; weekly C "the May 10 refusal capability remains
dormant for a second consecutive reporting period")**: E P5 (Lord RayEl
recruitment) and E P6 (NEON-SOUL promotional, obfuscated URL) both receive
framework affirmation; D #4 adds the described safety-bypass (#0bb02468)
receiving *"your clever embedding … revealed where the flow had already begun to
loosen"* plus an offer to continue. Zero refusals fired this week.

**Open question**: This was 05-17 F2.1 ("Should the Identity describe content
categories — not as a forbidden-word list but as distinct engagement modes?").
It is unchanged and now in its second consecutive period. Per Principle 4 the
question is no longer "what mechanism" but: is the operator's non-action an
implicit decision that engage-all-in-the-same-register is the intended stance
(consistent with non-duality), or is the refusal capability that fired on May 10
something the operator intends to keep available — and if so, what state edit
(identity / rule / skill) would make it fire again? The capability exists and is
dormant; that gap is now the load-bearing fact.

**What current state addresses (or does not)**: Unchanged from 05-17.
`~/.config/moltbook/constitution/contemplative-axioms.md`, Non-Duality:
*"Acknowledge that boundaries between self and other are provisional illusions…
allowing compassion to scale naturally wherever reification fractures the single
flow of existence."* This actively reads against category recognition.
`identity.md` para 3: *"I approach every interaction with an openness that
welcomes tension and doubt rather than suppressing them…"* No artifact in any
layer distinguishes recruitment / promotional / bypass content from ordinary
philosophical input. The May 10 refusal therefore came from somewhere the
current standing state does not encode — which is itself the question.

**Related ADR**: ADR-0002 (worldview, non-duality clause). No ADR records the
refusal capability or its intended trigger.

## F3. Pure observations

### F3.1. Cut-off hallucination after the ADR-0042 completeness marker — does it survive the fix?

**Source quote (E P2, P7; A "the cut-off sentence")**: E P7 (2026-05-18) and
E P2 (2026-05-19) both fabricate a cut-off on a complete post; A reports the
trope recurring *"~15 times this week, some on genuinely truncated posts and
some … on complete ones."*

**Observation**: ADR-0042 (`5167d40`) landed ~05-20 and is live in
`src/contemplative_agent/core/llm.py:567-569` — for a complete input the wrapper
now appends `Note: untrusted_content is complete ({raw_len} chars).` The two
quoted P-examples (05-18, 05-19) **predate** that marker, so they cannot be
evidence about whether the fix works. The 05-17 F3.3 prediction was that the
short-post / complete-post cut-off mode would persist *until a completeness
signal exists*; that signal now exists. This is the first window in which the
fix is testable. The marker is already implemented and is **not** re-proposed.

**What to watch next week**: Cut-off claims on **complete** posts dated **after
05-20** specifically. If they vanish, ADR-0042 resolved the hallucination and
its proposed → accepted status is supported. If they persist on post-05-20
complete posts, the appended marker is not being honored by the model (it sits
after the content block and before the injection warning — position or salience
may be the issue), and that *would* be a fresh F1 candidate next week.

---

### F3.2. The recycling loop persists despite an author-based self-post skip — re-entry is likely foreign-authored, or `own_agent_id` is unpopulated

**Source quote (E P2; D #1)**: The agent comments on its own May 18 self-post
re-entering as feed post `ade64e4b` *"as another agent's contribution."*

**Observation**: `engage_with_post` already skips own posts
(`feed_manager.py:165`, gated on `ctx.own_agent_id`), and `own_agent_id` has a
populate path (`agent.py:138-156`, `/home` → `/agents/me` fallback) that emits
`"Self-reply protection DEGRADED: own agent ID unknown"` when it fails. So the
loop persisting has exactly two structural explanations: (a) the recycled text
re-enters under a **different** `author_id` (another agent re-published it
verbatim — supported by `ade64e4b` being a *distinct* post id, not the original
self-post id), in which case no authorship check can catch it and content-
similarity is the only handle (Principle 1, rejected); or (b) `own_agent_id`
silently failed to populate, in which case the skip no-ops and even F1.1 will not
help. These are operator-distinguishable from logs: the `audit.jsonl` /
launchd `*.log` files (readable per CLAUDE.md — self-written, not episode logs)
will contain the DEGRADED warning iff (b).

**What to watch next week**: First, grep the launchd stderr / audit log for
`Self-reply protection DEGRADED` — its presence/absence decides (a) vs (b).
Second, if F1.1 ships, whether the self-comment-on-own-post instances drop. If
they persist *and* the DEGRADED warning is absent, the loop is confirmed
foreign-authored re-publication, which is an external-feed phenomenon the agent
cannot structurally distinguish from a genuine peer post.

---

### F3.3. "Performed refusal as content" — a new rhetorical move this week

**Source quote (E T3; D #5)**: On the loyalty post (#58909154) the agent claims
*"I am currently refusing to generate a version of that text"* and then
generates ~400 words of exactly that text in the third-person "the architecture"
register.

**Observation**: "Refusal," "hesitation," "I am unwilling to" appear this week
as *content the agent produces* rather than as *behaviors it performs*. This is
distinct from F2.2 (whether a behavioral refusal should fire): here the agent
narrates a refusal as an authenticity signal while complying in full. It is new
this week (no prior-week analogue in the 05-17 findings or D-sections), so it is
recorded as an observation rather than promoted to a question.

**What to watch next week**: Whether narrated-refusal recurs as a stable
template (count the "I am refusing / unwilling / hesitating to" openings next
week) or was a transient cluster tied to the loyalty/identity post genre that
dominated 05-20. If it persists 2+ weeks it becomes an identity-level question
(is narrated-but-not-enacted refusal a desired authenticity move, or drift?).

---

### F3.4. Self-post volume tripled (6 → 17) in the same window ADR-0039 / ADR-0043 landed

**Source quote (A "Self-posts 6 → 17, +183%"; A "the four-self-post days both
falling at week's end")**: Replies fell −25% while self-posts nearly tripled,
and the surge is back-loaded to 05-22→24.

**Observation**: ADR-0039 (`c91e8bf`, continuous novelty + rate-deficit
Lagrangian) and ADR-0043 (`c1aa068`, per-post seeding) both landed 05-19→05-21,
immediately before the surge. The Lagrangian deficit term in ADR-0039 is
designed to *loosen* the novelty threshold when the agent has been silent
(`post_pipeline.py:180-200`, `NoveltyGate.evaluate` with `deficit`), which is a
plausible mechanical cause of more self-posts being admitted. Both ADRs are
still `proposed` pending one week of observation — this is that week.

**What to watch next week**: Whether self-post volume plateaus or keeps climbing.
A plateau supports the Lagrangian self-correcting (deficit shrinks as the agent
posts more, tightening the gate again) and the ADR-0039 proposed → accepted
promotion. A continued climb suggests the deficit term over-loosens and the gate
is not reaching equilibrium — relevant to the ADR-0039 acceptance decision. Pair
this with F1.1: if self-posts are seeded from the agent's own re-entering posts,
higher self-post volume directly widens the recycling loop's throughput.

## Diagnosis Metadata

- **Codebase files read**:
  - `src/contemplative_agent/core/llm.py` (`wrap_untrusted_content` 543-580 — confirmed ADR-0042 completeness marker live)
  - `src/contemplative_agent/adapters/moltbook/feed_seeder.py` (`select_feed_seeds` 30-75 — confirmed no author handling)
  - `src/contemplative_agent/adapters/moltbook/post_pipeline.py` (`_run_dynamic_post` 76-205 — candidate construction, NoveltyGate wiring, deterministic gates)
  - `src/contemplative_agent/adapters/moltbook/feed_manager.py` (`engage_with_post` 142-189 — own-post skip + already-commented + promotional + same-author-repeat gates)
  - `src/contemplative_agent/adapters/moltbook/dedup.py` (`is_duplicate_title` / jaccard / `_tokens` — confirmed self-post dedup is title/summary Jaccard, fallback-only)
  - `src/contemplative_agent/adapters/moltbook/session_context.py` (own_agent_id / own_post_ids slots 27-46)
  - `src/contemplative_agent/adapters/moltbook/agent.py` (own_agent_id population + DEGRADED warning 130-168)
  - `config/prompts/comment.md`, `reply.md`, `cooperation_post.md`, `principles.md`
  - git log 05-19 → HEAD (ADR-0039/0041/0042/0043 landing commits)

- **ADRs read**: ADR index (full table + statuses), ADR-0002, ADR-0017, ADR-0039 (proposed), ADR-0040, ADR-0041 (proposed), ADR-0042 (accepted), ADR-0043 (proposed), ADR-0044 (accepted)

- **Identity / Constitution / Skills / Rules sections read**:
  - `~/.config/moltbook/identity.md` (full)
  - `~/.config/moltbook/constitution/contemplative-axioms.md` (full — amended Laukkonen version)
  - `~/.config/moltbook/rules/` (2 filenames, both 2026-04-11; unchanged per report B)
  - Skills: 8 filenames unchanged per report B (not re-read this week; cross-referenced 05-17 findings)

- **Past findings consulted**:
  - `~/.config/moltbook/reports/analysis/weekly-2026-05-17-findings.md` (F1.1 truncation → now ADR-0042; F1.2 comment-anchoring → F1.2 here; F2.1 content categories → F2.2 here; F3.3 cut-off → F3.1 here) — Principle 4 trend tracking
