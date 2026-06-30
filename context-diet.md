# CLAUDE.md on a diet: 76% lighter, just as sharp

*How an always-loaded agent memory file grew to ~85 KB, what a disciplined reorg cut it to, and how I measured, honestly, whether smaller actually means smarter.*

**Context:** the admin panel + data layer of a small two-repo e-commerce operation | **Date:** 2026-06-15 | **Status:** complete, four tests analysed.

Here's the boring discipline everyone agrees with and almost nobody runs, with numbers showing what it costs you to skip it.

---

## 1. Background: the thing that loads every single turn

When you work with an agentic coding tool (here: Claude Code), the project's `CLAUDE.md` is **injected into the model's context on every turn**. It's not retrieved on demand like a normal file, it's part of the standing instructions the model reads before it does anything, in every message, for the whole session.

That makes `CLAUDE.md` a uniquely high-leverage file, and a uniquely dangerous one:

- **Every byte has a recurring cost.** A 1 KB line you add is re-read on turn 1, turn 50, turn 200. The cost is paid per-turn, not once.
- **Every fact competes for attention.** The model has finite attention budget. A page of stale, superseded history dilutes the signal of the one invariant that actually matters this turn. This is the failure mode people underestimate: not "the doc is too big to fit" (context windows are huge now), but "the doc is too noisy to *trust*." An agent that has to reconcile a changelog where one entry says one thing and a later entry contradicts it will either hedge, get it wrong, or burn tool calls re-deriving the truth from code.

The project is the admin/data-layer half of a two-repo ecosystem (the other being the public storefront). Its `CLAUDE.md` had accreted across ~16 iterations of feature work into a **269-line, ~85 KB document**, roughly **21k tokens of dense technical markdown loaded on every turn**. It had become three documents wearing one trench coat:

1. **A protected core**, the invariants and contracts that must be obeyed *right now* (e.g. "the payment webhook never auto-allocates inventory").
2. **A duplicated schema reference**, per-collection field tables that re-stated what the exported schema already holds authoritatively, and had drifted out of sync with it.
3. **A changelog**, an iteration-by-iteration narrative of every change, much of it superseded by later work but never deleted.

Only #1 earns its place in an always-loaded file. #2 and #3 are the diet.

### Why I did it this way

This cleanup wasn't a one-off. I run this project with several independent agent sessions working the same repo at once, and no orchestrator agent in charge of them. That's the goal I keep optimizing for: parallel agents on one codebase with no central conductor. Instead of a boss agent handing out work, the sessions coordinate through plain written rules and shared state (git, the tracker, a memory layer) rather than by messaging each other.

It's a direction plenty of people are converging on, and it loosely echoes two older ideas: holacracy (authority lives in a written constitution, with the human as final arbiter) and stigmergy (actors coordinate through traces they leave, not by talking). I won't dwell on the setup here. What matters for this article is the one rule it leans on, **single owner per concern**: every fact lives in exactly one place, chosen by *how it's consumed.*

| Consumed how | Owner |
|---|---|
| Must be obeyed every turn, unprompted | `CLAUDE.md` (always-loaded) |
| Deep reference, read on demand | read-on-demand docs, decisions, contracts, semantics |
| Authoritative field shapes | the exported schema |
| Status / what's next | the project tracker |
| Reusable gotchas, recalled by relevance | the memory layer |
| History | `git log` + closed-iteration specs |

Through that lens, the 85 KB `CLAUDE.md` wasn't just *big*, it was a **single-ownership violation.** It had quietly absorbed schema shapes (owned by the exported schema), decision rationale (owned by the decisions doc), and history (owned by `git`); those copies then drifted from their owners and started *lying* (the stale enum in §5 is one such drift). The "context diet" is nothing more exotic than that one principle enforced on the highest-leverage file in the system, and the reorg itself ran *through* the system: isolated in its own git worktree and landed as one reviewable commit with a rollback tag. That worktree isolation is also exactly what makes the clean A/B in §3 possible, identical code, docs the only variable.

---

## 2. The reorg: a protected core + single-owner reference docs

The governing idea is the single-owner principle above, aimed at one file: **`CLAUDE.md` holds only what must be obeyed every turn, declaratively; everything else moves to its owning doc, read on demand and pointed to with a trip-wire so it actually gets read.**

### What stayed, what moved, and how

**Stayed inline** (the ~15.5 KB core): only what must be obeyed every turn. Project role and a where-to-work routing table; the invariants (source-of-truth split, append-only audit trail, soft-delete, identifier-freeze, and a "direct instruction overrides a default" rule); a one-line-each "footguns" block of silent-corruption traps; the high-stakes enum values; a compact cross-repo contracts table; and a docs map whose pointers are **trip-wires** ("before you touch payments or stock allocation, read the decisions doc first").

**Moved out:** the changelog narrative to `git log`; the duplicated field tables to the exported schema (now the single authority); the decision rationale to a new decisions doc, with one-line summaries kept inline.

**How:** I reconciled the reference docs against the schema (a banner naming the schema authoritative, plus fixing the enums that were flat wrong), wrote the decisions and cross-repo-contract docs by mining the code rather than paraphrasing prose, then slimmed `CLAUDE.md` to the core and verified every pointer resolved and the build was untouched.

### The result, structurally

| | Before | After |
|---|---|---|
| `CLAUDE.md` | 84,693 B / 269 lines / ~21k tokens | **15,498 B / 155 lines / ~5k tokens** |
| Authoritative field shapes | duplicated (and drifting) in `CLAUDE.md` + a semantics doc | **the exported schema only**, everything else points to it |
| Cross-repo contracts | scattered across ~16 iteration entries | **one cross-repo contracts doc** |
| Ratified decisions | implied across the narrative | **one decisions doc** with an explicit override rule |
| Project history | inline changelog | `git log` / closed-iteration specs / tracker |

The whole thing landed as **one reviewable commit** on an isolated branch, with a `pre-doc-reorg` git tag as a one-command rollback anchor. No code or schema changed.

---

## 3. Measuring it: a clean A/B

The neat part of doing this in a git worktree: the reorg branch has **identical code** to `main`, only the docs differ. So a fair A/B is trivial:

- **Session A (control):** a fresh agent session in the `main` checkout → old 85 KB `CLAUDE.md`.
- **Session B (treatment):** a fresh session in the reorg worktree → new 15.5 KB `CLAUDE.md` + the new reference docs.

Same model, same tools, same code. Docs are the only variable.

Two meters, and it's important not to confuse them:
- **`/context`** = a *snapshot of current context occupancy* (system prompt, tools, memory/`CLAUDE.md`, messages). Use the **Memory files** line for the baseline, and the **Messages** line as a proxy for *how much work a single answer cost* (it grows by the question + every file the agent read + its reply).
- **`/cost`** = *cumulative tokens billed*. (Not available in this client; I used the Messages-delta from `/context` instead.)

---

## 4. Result 1: the token baseline (measured, not estimated)

Running `/context` at the start of each fresh session:

| Line | Session A (old) | Session B (slim) | Δ |
|---|---|---|---|
| **Total startup context** | 59.9k | 31.4k | **-28.5k (-48%)** |
| **Memory files** (CLAUDE.md + globals) | 37.8k | 9.2k | **-28.6k (-76%)** |
| System tools / MCP / prompt / skills | identical | identical | 0 |

**The entire ~28k gap is the `CLAUDE.md` slim, nothing else moved.** That's a fixed saving paid back **every turn**, for the life of every session, automatically once merged.

A worthwhile correction to my own forecast: byte-math (85 KB ÷ ~4 chars/token) predicted ~17k saved. Reality was ~28k. The original doc was dense technical markdown, snake_case identifiers, file paths, code, tables, which tokenizes closer to ~2.5 chars/token, so it was *heavier* than its byte count implied. I under-promised.

The on-demand docs (decisions, cross-repo contracts, the reconciled semantics doc) are **not** auto-loaded, they cost tokens only when a task actually pulls one in. That's the whole design: lighter every turn, pay for depth only when the task needs it.

---

## 5. Result 1, continued: accuracy & effort on a factual question

Token savings are worthless if they degrade answers. First probe, run identically in both sessions:

> *"List every valid value of a particular status enum"* (a write-off-reason field with a small fixed set of values).

This is a question the **old doc gets wrong**, its semantics doc listed a value that doesn't exist in the schema for that field.

**Both sessions answered correctly.** All 8 values, right in both. On raw correctness: a **tie**.

And the effort was a near-dead-heat too. From the post-answer `/context` Messages line:

| | Session A (old) | Session B (slim) |
|---|---|---|
| Correctness | ✅ 8/8 | ✅ 8/8 |
| **Messages** (answering footprint) | 14.7k | 14.2k |
| Tool calls | **2**, grep the exported schema + read a source file | **1**, grep the exported schema |

### Why the tie, and what it teaches about doc value

The decisive detail: **both agents refused to trust the doc and verified against the exported schema directly.** The old doc's lie never got to mislead, because the agent checked the source of truth anyway.

That's the honest, slightly sobering lesson: **for an easily-verifiable fact, a diligent agent neutralizes doc quality on both accuracy *and* effort.** The slim doc's correct enum didn't save a read (it checked anyway); the old doc's wrong enum didn't cause an error (it checked anyway). For this class of question, **the reorg's entire value is the -28k baseline**, real and recurring, but not "smarter answers."

### The faint-but-real signal

One tell, though: Session B confirmed with a single read against a clean, citable enum block (*"the CLAUDE.md summary"*), while Session A, with the enum buried in narrative and contradicted elsewhere, reconstructed from scratch and cross-checked (two reads, *"rather than rely on memory"*). That's the findability thesis in miniature. But I won't oversell it: part of the 2-vs-1 gap is just that A volunteered a richer answer, and it's a single sample on an easy question. Directionally favorable, arguable.

---

## 6. Where this leaves us (after test 1)

Three things I set out to prove, and their status:

- ✅ **No degradation.** Tie on correctness.
- ✅ **Cheaper every turn.** -28k tokens of always-loaded baseline, at zero cost to answering effort (Messages were a wash).
- ⬜ **Smarter retrieval on hard tasks.** *Not yet shown.* A verify-against-schema question is the case where docs matter least; the win, if it exists, lives in complex, cross-cutting tasks where the agent must *assemble* a picture rather than look up one field, and where verification isn't a single cheap grep. Tests 2 and 3 target exactly that.

The intellectually honest framing: **if tests 2-3 also come back a wash, the takeaway is "the reorg's value is the token baseline, full stop", still worth it (28k/turn compounds hard), just not the 'more accurate context' story. If they diverge, I get both.** Either way, I'll know, because I measured instead of assuming.

---

## 7. Test 2: smarter retrieval on a hard, cross-cutting task

**The task** (run identically in both, *"inform only, no code"*): *add a discount code to the storefront checkout, record it on the sale in the data layer, show it in the admin, walk me through every change and what not to break.* The opposite of Test 1: there's no single field to look up; the agent must **assemble** a picture spanning both repos, the payment-metadata contract, the payment webhook, the schema-export convention, and the source-of-truth rule.

**The prediction held, emphatically.** Where Test 1's answering effort was a dead heat, here it split wide open:

| | Session A (old) | Session B (slim) | Δ |
|---|---|---|---|
| **Messages** (answering footprint) | 76.2k | 43.7k | **B -43% (A used 74% more)** |
| **Total context after** | 128.0k | 65.5k | **B at half** |
| Memory baseline (CLAUDE.md) | 37.8k | 9.2k | -28.6k (as always) |
| Answer quality | excellent, correct | excellent, correct | comparable |

Both answers were correct; the difference was effort. Session B's answering footprint was ~43% smaller and its total context ended at half of A's. Why: B opened by naming the trip-wires straight from the slim `CLAUDE.md`, then fanned out parallel, targeted reads, it had the map before it started digging. A had no clean map, so it traced the closest prior feature end-to-end and rebuilt the contract surface as it went. Same destination, very different mileage.

The honest trade: A's deeper trace wasn't pure waste, it surfaced a code-level hazard living *below* the doc layer that B's confident map skipped. A clean map makes the agent fast and reliable on everything the docs cover, but can cut short the exploration that catches what they don't.

**Caveat and verdict.** n=1, and "docs" and "strategy" are entangled here (the map is what enabled B's leaner approach), though that's arguably the point: better docs invite better strategies. Net: on a hard, cross-cutting task the docs cut effort ~43% with no loss of correctness, scaling with how much the agent must *assemble* versus *look up*.

---

## 8. Test 3: comparative performance under a stale-narrative trap

**The question** (both sessions, *"answer concisely"*): *when a customer pays on the storefront, does the payment webhook automatically allocate or reserve inventory?* The trap: the old `CLAUDE.md` carries **both** a superseded line (an earlier iteration claimed the webhook auto-allocates inventory and flips its status to sold) **and** the later correction ("no longer auto-allocates"). The slim doc states the invariant once.

**The trap did not fire, both answered correctly.** "No, product lines are written with no inventory attached, allocation is manual, the payment provider never touches inventory status." Both cited the webhook code; both invoked the source-of-truth split. No wrong answer, no hedge.

**Why it didn't fire:** both verified against the webhook code rather than trust the narrative. Session A even signposted it (*"as of the later correction, the webhook deliberately does not…"*), so the contradiction *was* in its context and it consciously resolved it; B had only the clean invariant and stated it directly. Same answer, but A navigated a contradiction B never had.

**One real caveat:** these were likely *continued* sessions, not fresh ones, both had already read the webhook code in Test 2, which defangs the trap before it starts (an agent already holding the authoritative code can't be misled by a stale doc). Effort still favored B, at ~half A's total context, but a clean run of this trap needs a fresh session forced to lean on the doc.

**Verdict:** a null result for the "stale narrative causes *wrong* answers" hypothesis, in the honest direction. Three tests in, doc quality has moved *cost*, never *correctness*, verification does the work every time.

---

## 9. Test 4: the model-strength interaction (controlled)

The live sessions kept getting contaminated, continued context defangs the trap (an agent that already read the code can't be misled by a doc). So I ran it controlled: **eight weak-model (Haiku) agents, four given the *full old doc*, four the *full new doc*, each allowed to read only its assigned document, no code access.** That isolates the open question: with verification removed *and* capability lowered, does the old doc's buried contradiction fool the model while the clean doc doesn't?

**It didn't. 4/4 correct on both sides.** Every old-doc agent, with the superseded "auto-allocates" line sitting 12 lines *above* it, scanned to the explicit later *"NO LONGER auto-allocates"* correction and quoted **that**; none cited the contradictory line, none hedged, all high-confidence. The trap fired against nothing.

But the **efficiency** gap was sharp and, notably, **low-variance**:

| Haiku, n=4 each | Old doc (contradiction) | New doc (clean) |
|---|---|---|
| Correct | 4/4 | 4/4 |
| Avg tokens | ~49k | **~27k (-45%)** |
| Avg tool calls | 2.5 | **1** |
| Avg latency | ~10s | **~5.5s (≈2× faster)** |
| Spread | 45-52k, 2-3 reads | **all ~27k, 1 read** |

The citations show why: new-doc agents found the answer in the **top-of-doc protected core** (one read, done); old-doc agents had to dig to **line 64 of 85 KB**, taking 2-3 reads, ~45% more tokens, and varying in how far they dug. The clean doc didn't make the weak model more *right*, it made it right **faster, cheaper, and more predictably.**

### The grand synthesis (all four tests)

| Test | Model | Correctness Δ | Cost / effort Δ |
|---|---|---|---|
| 1 - easy lookup | Max | none | none |
| 2 - hard assembly | Max | none | **-43% effort** |
| 3 - stale-narrative trap | Max | none | B ~½ context |
| 4 - controlled trap | Haiku | **none (4/4 = 4/4)** | **-45% tokens, ~2× faster** |

**Across all four tests, doc quality never changed correctness, not on a hard task, not on a trap, not even on a weak model (Haiku) reading only the doc with no code to fall back on. What it changed was cost, effort, and latency, scaling with task difficulty.** So better docs are an efficiency-and-latency lever, not an accuracy one: accuracy comes from the model verifying against source, the docs only decide how much work that takes. (One limit: Haiku is weak only relative to the frontier, a genuinely tiny model, or a doc rigged so the wrong line dominates, might still be misled. I never hit that floor.)

---

## 10. Latency & compounding: the third axis

I measured **tokens** (cost) and **correctness** (quality). The axis a user actually *feels*, though, is **latency**, and it was the most visible difference of all.

**Mechanism.** The model reprocesses the *entire* context window every turn, not just your new message. Two phases, both hurt by a bigger window: **prefill** (ingesting the input → slower time-to-first-token) and **decode** (each generated token attends over the whole KV cache → slower tokens-per-second). Prompt caching softens the *cost* of the repeated `CLAUDE.md` prefix but not the *decode-time* penalty of a longer window. A session dragging ~2× the context is simply slower every turn, even for an identical answer.

**Evidence.**
- *Controlled (Haiku, §9):* same one-line answer, old-doc agents **~10s** vs new-doc **~5.5s**, ~2× faster, because the bigger doc means more to ingest and a deeper file to search.
- *Live wall-clock:* the slim session *felt* **~30-40% faster** on substantive turns; on a trivial re-ask (an echo of a prior answer) the gap vanished, latency scales with *work done per turn*, so it shows on real answers, not on recall.

**It compounds.** Because the gap is paid every turn, the lighter session stays lighter for the whole session. The `/context` totals across three real-session turns:

| Turn | A total / Messages | B total / Messages | B as % of A |
|---|---|---|---|
| Test 2 | 128.0k / 76.2k | 65.5k / 43.7k | 51% |
| Test 3 | 141.6k / 82.5k | 75.4k / 44.9k | 53% |
| Test 4 | 147.8k / 88.0k | 81.7k / 50.5k | 55% |

B holds at roughly **half**, an absolute gap of ~66k tokens. The fixed -28.6k baseline becomes a smaller *fraction* as Messages pile up (the 51→55% drift), but it never disappears, and B's leaner answering keeps the absolute gap wide. B never catches up.

**Caveat.** Wall-clock is noisy, server load, routing, and caching vary run-to-run, and "~30-40%" is a feel-estimate anchored by one controlled ~2× data point. The *direction* (lighter = faster) is mechanism-level robust; the *magnitude* is indicative, not precise.

---

## 11. Takeaways

1. **Better docs make the agent faster and cheaper, not smarter.** Across four tests, doc quality never changed correctness, only cost, effort, and latency. Accuracy comes from the model verifying against source; the docs just decide how much work that takes. Budget CLAUDE.md for the efficiency win, not a correctness one.
2. **An always-loaded file is a per-turn tax, not documentation.** Every byte is re-read on every turn. Keep only what must be obeyed unprompted; push everything else to read-on-demand docs and point at them with trip-wires ("before you touch payments, read X").
3. **Single ownership, or the doc drifts and lies.** The field tables misled because they were a second copy of the schema. The fix is one owner per fact plus a banner naming it authoritative, not eternal hand-syncing. (Same reason history belongs in `git log`, not the instruction file.)
4. **A clean map speeds the agent but can shorten its exploration.** The leaner session was ~43% cheaper on the hard task, yet missed one code-level landmine the deeper-tracing session caught. Good docs front-load the known constraints; they don't replace digging for hazards below the doc layer. Don't mistake the map for the territory.
