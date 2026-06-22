# Task 1.2 — Orchestrate Multi-Agent Systems with Coordinator–Subagent Patterns
### Study Guide (beginner-friendly; every term defined)

**Reading order:** Section 0 is the one big idea. Then "What this task is even about" defines the vocabulary. Then each **Knowledge of** and **Skills in** bullet from the task is unfolded in order, followed by anti-patterns and a self-check.

> This builds on Task 1.1. The one thing to carry over: an **agent** runs a loop — it calls a tool, gets a result, and repeats until it's done (`stop_reason: end_turn`). Here, some of those "tools" are *other agents*.

---

## 0. The one idea everything hangs on

> **The coordinator is a hub. Every decision, every message between agents, every error, and every piece of routing flows through it. Subagents are isolated specialists the coordinator picks for the job, gets results from, and re-tasks until the answer is good enough.**

If you remember only one shape, remember the **hub and spokes**: the coordinator in the middle, each subagent on a spoke, and *no spoke connects to another spoke directly* — they only connect through the hub. Every "Knowledge of" and "Skills in" bullet below is a consequence of that.

---

## What this task is even about (plain English)

- **Agent:** an AI (Claude) that can use tools and keeps working step-by-step until a task is done — not a one-shot answer.
- **Multi-agent system:** instead of one agent doing everything, you use *several*, each focused on one kind of work, with one in charge.
- **Coordinator (a.k.a. orchestrator / lead):** the agent in charge. It does **not** do the detailed work itself; it breaks the job up, hands pieces out, collects answers, and assembles the final result. Think *project manager*.
- **Subagent:** a specialist worker agent the coordinator hands a single, focused task to. Examples here: one searches the web, one reads documents, one combines findings, one writes the report.
- **Context (context window):** an agent's working memory — everything it can currently "see." It's limited, like desk space. A key fact you'll see repeated: **each subagent has its own separate desk; it does not automatically see what's on the coordinator's desk.**
- **Hub-and-spoke architecture:** the coordinator is the hub; subagents are on spokes. All communication goes hub→spoke and spoke→hub, never spoke→spoke. The hub is the single place that sees everything.
- **Task decomposition:** breaking a big request ("research topic X") into smaller pieces a subagent can each handle.
- **Delegation:** handing a piece to a subagent.
- **Result aggregation:** gathering the subagents' answers back together.
- **Routing:** deciding which information goes to which agent, and when.

Why bother with all this instead of one big agent? Because one agent trying to web-search, read ten documents, *and* write a report all on one "desk" runs out of room and gets sloppy. Splitting the work keeps each agent focused and lets independent pieces run at the same time.

---

## Knowledge area 1 — Hub-and-spoke: the coordinator manages *all* communication, errors, and routing

In hub-and-spoke, **subagents never talk to each other.** The web-search agent does not hand its results straight to the synthesis agent. Instead:

```
            ┌────────────────────────────────┐
            │          COORDINATOR           │  <- the HUB: sees everything,
            │  (routing, error handling,     │     routes everything,
            │   all inter-agent messages)    │     handles every error
            └──┬─────────┬─────────┬─────────┘
   delegates   │         │         │   delegates
        ┌───────▼──┐ ┌────▼─────┐ ┌─▼──────────┐
        │ web      │ │ document │ │ synthesis  │   <- SPOKES: isolated specialists
        │ search   │ │ analysis │ │ + report   │      (no spoke talks to another spoke)
        └──────────┘ └──────────┘ └────────────┘
```

**Why route everything through the hub?** Three concrete payoffs, all named in the task:

1. **Observability** — because every message passes through the coordinator, there is *one* place to watch what's happening. If subagents whispered to each other directly, you'd have no single log of the conversation.
2. **Consistent error handling** — when a subagent fails (e.g., a web search returns nothing), the coordinator catches it in one place and decides what to do (retry, try a different subagent, give up gracefully). You don't scatter error logic across every agent.
3. **Controlled information flow** — the coordinator decides exactly what each subagent receives. Nothing leaks sideways; sensitive or irrelevant data isn't dumped on agents that don't need it.

---

## Knowledge area 2 — Subagents have isolated context (they don't inherit the coordinator's history)

This is the single most important fact in the whole task, so slow down here.

When the coordinator spawns a subagent, that subagent **starts with a fresh, mostly-blank memory.** It does **not** automatically see:
- the coordinator's conversation history,
- what other subagents found,
- decisions already made.

The *only* thing the subagent receives is **the prompt the coordinator writes for it.** (You'll see the mechanism in Task 1.3: the prompt string is the one and only channel in.)

**Why is it designed this way?** Isolation is the entire benefit. If every subagent inherited the full history, their desks would fill up with each other's clutter, and you'd lose the focus and parallelism that made multi-agent worth doing. The cost of isolation is that *you* (via the coordinator's prompt) must hand a subagent everything it needs — there's no shared memory to fall back on.

**Practical consequence:** if the synthesis agent needs the web-search results, the coordinator must *put those results into the synthesis agent's prompt.* It can't assume the synthesis agent "already knows."

---

## Knowledge area 3 — The coordinator's four jobs (and choosing subagents by complexity)

The coordinator has four responsibilities. Memorize them as a cycle:

1. **Task decomposition** — split the request into pieces.
2. **Delegation** — hand each piece to the right subagent.
3. **Result aggregation** — collect the answers back.
4. **Deciding which subagents to invoke** — and this is the subtle one: *based on the query's complexity.*

That last job means the coordinator does **not** always run the full pipeline. A simple question ("what's the capital of France, with a source?") might need only the web-search agent, then a short synthesis — not document analysis. A deep question needs all four. The coordinator reads the request and **dynamically selects** which subagents are actually needed (this is a Skill below).

---

## Knowledge area 4 — The risk: overly narrow decomposition → incomplete coverage

Here's a failure mode to watch for. If the coordinator chops a broad topic into pieces that are *too narrow*, the subagents will each do their tiny slice perfectly — and the overall research will still **miss whole areas** nobody was assigned.

Example: research "the impact of remote work." If the coordinator only spawns "impact on commute times" and "impact on office rents," it gets two solid answers but completely misses mental health, productivity, team culture, hiring — huge parts of the topic. Each subagent succeeded; the *coverage* failed.

The lesson: decomposition must be broad enough to **cover the topic**, not just deep on a few corners. This is exactly why the iterative-refinement skill (below) exists — the coordinator checks for gaps and fills them.

---

## Skill 1 — Dynamic subagent selection (don't always run the full pipeline)

**What to design:** a coordinator that first *analyzes what the query actually needs*, then invokes only the relevant subagents — rather than blindly sending everything through search → analyze → synthesize → report every time.

**Why it matters:** running the full pipeline on a simple question wastes time and money and can even add noise. Matching the agents invoked to the question's real requirements is faster, cheaper, and cleaner.

This is the **model-driven decision** idea from Task 1.1 applied at the team level: you describe each subagent well, and let the coordinator *choose* which to call based on context — not a hardcoded fixed sequence.

---

## Skill 2 — Partition research scope to minimize duplication

**What to design:** when you do fan work out, give each subagent a **distinct slice** so two agents don't research the same thing.

Two common ways to partition:
- **By subtopic** — agent A covers health effects, agent B covers economic effects, agent C covers policy.
- **By source type** — agent A reads academic papers, agent B reads news, agent C reads internal company docs.

**Why it matters:** without clear partitions, three agents might all google the same query and return the same three articles — wasted effort and a repetitive final report. Distinct assignments = broad coverage with no overlap.

Note the balance with Knowledge area 4: partitions must be **distinct (no overlap)** *and* **collectively complete (no gaps)**. Distinct-but-incomplete is the narrow-decomposition trap.

---

## Skill 3 — Iterative refinement loops (check synthesis for gaps, re-delegate, repeat)

This is the most powerful skill in the task and the cure for narrow decomposition.

Instead of "search once → synthesize once → done," the coordinator runs a **loop**:

```
1. Delegate search + analysis.
2. Collect results; have the synthesis subagent draft findings.
3. COORDINATOR EVALUATES the draft: "Are there gaps? Missing subtopics?
   Unsupported claims? Thin coverage anywhere?"
4. If gaps:  re-delegate to search/analysis with TARGETED queries
            aimed exactly at the gaps  ->  re-invoke synthesis.
5. Repeat 3–4 until coverage is sufficient.
6. Generate the final report.
```

**Why it matters:** the first pass almost never covers a broad topic completely. The refinement loop lets the coordinator notice "we said nothing about X" and send a precise follow-up ("search specifically for X") rather than shipping an incomplete report. The stopping condition is *coverage is good enough* — not a fixed number of rounds.

This mirrors Task 1.1's loop philosophy: keep going while there's more to do (here: gaps remain), stop when the goal is met (coverage sufficient).

---

## Skill 4 — Route all subagent communication through the coordinator

**What to design:** make the coordinator the *only* path between subagents. Subagent results come back to the coordinator; if agent B needs agent A's output, the coordinator passes it along — A and B never connect directly.

**Why it matters (same three as Knowledge area 1):** observability (one place to watch), consistent error handling (one place to catch failures), and controlled information flow (the coordinator decides what each agent sees). This is hub-and-spoke restated as a *design rule you implement*, not just an architecture you describe.

---

## Anti-patterns to avoid

- **Spoke-to-spoke shortcuts** — letting subagents pass results directly to each other. Breaks observability and error handling; the hub loses its single-point-of-control. Always route through the coordinator.
- **Always running the full pipeline** — invoking every subagent regardless of the query. Wasteful and noisy; select dynamically by complexity.
- **Over-narrow decomposition** — slicing a broad topic so finely that whole areas go uncovered. Each agent "succeeds" while the research fails. Decompose for *coverage*, and use the refinement loop to catch gaps.
- **Overlapping assignments** — two agents researching the same slice. Wasted work, repetitive output. Partition into distinct slices.
- **One-and-done synthesis** — synthesizing once and shipping without checking for gaps. Add the iterative-refinement loop.
- **Assuming subagents share memory** — expecting a subagent to "already know" what the coordinator or another agent found. They don't; the coordinator must pass context explicitly (Task 1.3).

---

## Worked example — research "How is AI changing software engineering jobs?"

| Step | Coordinator action | Why |
|---|---|---|
| 1 | Decompose into distinct slices: *hiring trends*, *day-to-day workflow changes*, *required skills*, *wage/role impact* | Broad coverage, no overlap |
| 2 | Decide complexity is high → invoke web-search **and** document-analysis (skip neither) | Dynamic selection |
| 3 | Assign each slice to a search agent; assign uploaded PDFs to the analysis agent | Partition scope |
| 4 | Collect results (all returned to the coordinator) | Hub routing |
| 5 | Synthesis agent drafts; coordinator reviews → notices nothing on *junior vs senior impact* | Gap detection |
| 6 | Re-delegate a targeted search "AI impact on junior vs senior engineers" → re-synthesize | Iterative refinement |
| 7 | Coverage now sufficient → report agent writes the cited report | Aggregation + finish |

Every arrow passes through the coordinator; no subagent ever talked to another.

---

## Self-check

1. What shape is the architecture, and what's the one rule? → **Hub-and-spoke**; subagents never talk directly — everything routes through the coordinator.
2. Why route everything through the coordinator? → **Observability, consistent error handling, controlled information flow.**
3. Do subagents inherit the coordinator's history? → **No** — isolated context; they only get what's in their prompt.
4. What are the coordinator's four jobs? → **Decompose, delegate, aggregate, and decide which subagents to invoke (by complexity).**
5. What's the risk of slicing a topic too finely? → **Incomplete coverage** — every agent succeeds but whole areas go unresearched.
6. How do you avoid duplicated work? → **Partition** into distinct subtopics or source types.
7. How does the coordinator fix gaps? → **Iterative refinement** — evaluate the synthesis, re-delegate targeted queries to fill gaps, re-synthesize until coverage is sufficient.
8. Dynamic selection vs full pipeline? → Analyze the query and **invoke only the needed subagents**, not always all of them.

---

## Deep dive — Task 1.2 in each of the six scenarios

Task 1.2 is about the **coordinator–subagent (hub-and-spoke)** pattern. Below, each of the six Domain-1 scenarios is unfolded the way Task 1.1 unfolds its scenarios: the verbatim description, a plain-English explanation, the *theory* of how the coordinator pattern actually operates there (step by step), code or a diagram where it clarifies, and a takeaway. A relevance tag says how central this task is to that scenario.

### Scenario 1 — Customer Support Resolution Agent · *Partial fit*
> *You are building a customer support resolution agent using the Claude Agent SDK. The agent handles high-ambiguity requests like returns, billing disputes, and account issues. It has access to your backend systems through custom MCP tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`). Your target is 80%+ first-contact resolution while knowing when to escalate.*

**What's happening here (plain English):** a chat agent for a store's support line. A customer writes something messy; the agent looks them up, checks orders, and either fixes the problem or hands it to a human. "High-ambiguity" means the request isn't clean — often several problems bundled in one message.

**How the coordinator pattern works here (theory):** most single-issue tickets are handled by *one* agent in a simple loop — no coordinator needed. The coordinator pattern earns its place when a ticket carries **multiple concerns**, e.g. "my order arrived broken AND I was charged twice." Then the support agent acts as a **hub**:
1. **Decompose** the ticket into independent concerns: {broken item, duplicate charge}.
2. **Delegate** each concern to a focused investigation (check the order; check billing) — these can run in parallel because they don't depend on each other.
3. **Aggregate** the findings back at the hub.
4. **Decide** per concern: resolve automatically, or `escalate_to_human`.

The hub-and-spoke rule matters because there must be **one place** that handles a failed tool call (a billing lookup times out) and **one place** that owns the escalation decision and the unified reply. If sub-investigations talked to each other directly, you'd lose that single point of control and could send the customer two contradictory messages.

```
            Support agent = HUB
   "broken + double-charged"  ──┬──> investigate order   (spoke)
                                └──> investigate billing (spoke)
        hub aggregates → resolves duplicate refund + arranges replacement
        → if anything is outside policy → escalate_to_human (one place)
```

**Takeaway:** coordinator pattern here = decompose a *bundled* ticket, investigate concerns in parallel, and keep one hub owning errors, escalation, and the single unified reply. Skip it for simple one-issue tickets.

### Scenario 2 — Code Generation with Claude Code · *Partial fit*
> *You are using Claude Code to accelerate software development. Your team uses it for code generation, refactoring, debugging, and documentation. You need to integrate it into your development workflow with custom slash commands, CLAUDE.md configurations, and understand when to use plan mode vs direct execution.*

**What's happening here (plain English):** Claude Code is Claude running in a developer's terminal, able to read and edit the project's real files. "Refactoring" = restructuring code without changing behavior; a big refactor can touch many files across the codebase.

**How the coordinator pattern works here (theory):** the **main Claude Code session is the coordinator.** For a refactor that spans several modules (say, renaming a core API used across `auth`, `billing`, and `ui`), the main session can **delegate per-module work to subagents** — one subagent per module — and then aggregate. The hub-and-spoke benefit is **context isolation**: the subagent doing the deep read of `billing` fills *its* context with billing details, which never clutter the `auth` subagent's context. The main session holds only the high-level plan plus each subagent's clean summary, so it doesn't drown in all three modules at once (the "single-agent ceiling" from Task 1.1's multi-agent scenario).

**Takeaway:** for cross-module work, let the main session coordinate per-module subagents so each module's detail stays on its own desk and the hub keeps the big picture.

### Scenario 3 — Multi-Agent Research System · *Core fit (the canonical scenario)*
> *You are building a multi-agent research system using the Claude Agent SDK. A coordinator agent delegates to specialized subagents: one searches the web, one analyzes documents, one synthesizes findings, and one generates reports. The system researches topics and produces comprehensive, cited reports.*

**What's happening here (plain English):** an AI research team. A manager (coordinator) splits a research question among specialists — a web searcher, a document reader, a synthesizer, a report writer — and assembles a final cited report.

**How the coordinator pattern works here (theory):** this is Task 1.2 *end to end*; every Knowledge/Skill bullet appears:
1. **Task decomposition** — the coordinator breaks the topic into distinct slices (subtopics or source types).
2. **Dynamic selection** — it invokes only the subagents the query needs (a simple factual question may need only search + a short synthesis, not document analysis).
3. **Scope partitioning** — it assigns *distinct* slices so two search agents don't fetch the same articles.
4. **Iterative refinement loop** — after the synthesis agent drafts, the coordinator **evaluates for gaps** ("we covered economic effects but nothing on health"), **re-delegates targeted queries** to fill them, and **re-synthesizes** — repeating until coverage is sufficient.
5. **Routing through the hub** — all results return to the coordinator, giving one place for observability, error handling, and (critically) **preserving each claim's source** so the final report can cite.

The central danger named in the task lives here: **over-narrow decomposition** → each subagent nails its tiny slice while whole subtopics go uncovered. The refinement loop is the cure.

```
GOAL → COORDINATOR ──parallel──> [web-search] [doc-analysis]   (distinct slices)
                       ↓ aggregate (sources preserved)
                    [synthesis] → coordinator checks gaps?
                       ↑ if gaps: re-delegate targeted query, re-synthesize
                       ↓ coverage sufficient
                    [report] → comprehensive, cited report
```

**Takeaway:** the research system *is* the coordinator pattern — decompose broadly, select dynamically, partition to avoid overlap, and loop to close coverage gaps, with everything routed through the hub so citations survive.

### Scenario 4 — Developer Productivity with Claude · *Strong fit*
> *You are building developer productivity tools using the Claude Agent SDK. The agent helps engineers explore unfamiliar codebases, understand legacy systems, generate boilerplate code, and automate repetitive tasks. It uses the built-in tools (Read, Write, Bash, Grep, Glob) and integrates with MCP servers.*

**What's happening here (plain English):** an AI helper for developers facing a big, unfamiliar pile of code ("codebase") — often old, poorly documented "legacy" software. Just mapping how it's organized takes days.

**How the coordinator pattern works here (theory):** mapping a sprawling system is exactly the **fan-out** the coordinator pattern is built for. The main agent acts as coordinator and **delegates exploration per module or per service** to subagents — one reads the `payments` package, one reads `auth`, one checks test coverage — all in parallel, each in its own isolated context. Each returns a clean structural summary; the coordinator **aggregates** them into one map of the system. **Dynamic selection** is the judgment call: a one-line "where is `authenticate()` defined?" should *not* spin up a research team (just Grep it directly); mapping a 200-file legacy service *should*. Spawning subagents for trivial lookups wastes time and money (Task 1.2's "always running the full pipeline" anti-pattern).

**Takeaway:** use a coordinator to fan exploration across modules so each subagent's deep read stays isolated and the hub assembles the map — but only when the task is big enough to justify it.

### Scenario 5 — Claude Code for Continuous Integration · *Partial fit*
> *You are integrating Claude Code into your CI/CD pipeline. The system runs automated code reviews, generates test cases, and provides feedback on pull requests. You need to design prompts that provide actionable feedback and minimize false positives.*

**What's happening here (plain English):** every time a developer proposes a code change (a "pull request"), an automated step reviews it and leaves comments — like a tireless reviewer on every change. (See Task 1.1's CI guide for the full from-scratch CI/CD primer.)

**How the coordinator pattern works here (theory):** for a large pull request touching many files, a **review coordinator** can fan out **per-file review subagents** (each reviews one file in its own clean context — avoiding "attention dilution"), then **aggregate** their findings into **one consolidated PR comment**. Routing through the hub is what prevents six scattered comments and lets the coordinator de-duplicate overlapping findings and add a cross-file integration check before posting. A light **iterative-refinement** pass can re-examine the highest-severity flags to cut false positives before the comment goes out.

**Takeaway:** a review coordinator fans out per-file reviewers and aggregates one clean comment — the hub is what turns many isolated reviews into a single, de-duplicated, trustworthy result.

### Scenario 6 — Structured Data Extraction · *Partial fit*
> *You are building a structured data extraction system using Claude. The system extracts information from unstructured documents, validates the output using JSON schemas, and maintains high accuracy. It must handle edge cases gracefully and integrate with downstream systems.*

**What's happening here (plain English):** turning human-written documents (invoices, resumes, emails) into tidy computer-readable data (JSON) that downstream systems can use. (Task 1.1's extraction guide covers the mechanics.)

**How the coordinator pattern works here (theory):** for a *single* document, no coordinator is needed — one agent extracts and you're done. The pattern appears at **batch scale**: a coordinator **partitions** a large set of documents across extraction subagents (by document, or by section of a very long document), runs them in parallel, and **aggregates** the records. The **refinement loop** applies too: after validation, the coordinator can spot records with **missing required fields** and **re-delegate** just those for a focused re-extraction, rather than redoing the whole batch. Hub routing gives one place to collect validation failures and decide which records need a retry vs human review.

**Takeaway:** for batches, a coordinator partitions extraction across subagents and loops to re-extract only the records that failed validation — overkill for one document, valuable at volume.

**One-line cross-scenario takeaway:** the coordinator pattern is *core* to research (S3) and developer exploration (S4), and a *targeted tool* for bundled support tickets (S1), cross-module refactors (S2), batched reviews (S5), and large extraction jobs (S6) — but skip it whenever the work is genuinely single-threaded.

---

### Sources verified against current Anthropic docs (June 2026)
- *Subagents in the SDK* / *Agent SDK overview* — context isolation (subagent context starts fresh; only the prompt crosses), coordinator delegates and receives final results, the coordinator decides when to invoke.
- *Multiagent sessions / Managed Agents multi-agent* — coordinator roster, parallel fan-out, specialization, per-agent isolation.
- The orchestration patterns (hub-and-spoke routing, iterative refinement, scope partitioning) reflect Anthropic's documented multi-agent guidance as of June 2026; this is a fast-moving area, so re-check specifics against current docs.
