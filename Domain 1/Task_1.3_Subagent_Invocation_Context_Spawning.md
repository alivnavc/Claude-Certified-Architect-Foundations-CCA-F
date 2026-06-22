# Task 1.3 — Configure Subagent Invocation, Context Passing, and Spawning
### Study Guide (beginner-friendly; every term defined)

**Reading order:** Section 0 (the one idea) → "What this task is even about" (vocabulary) → each **Knowledge of** and **Skills in** bullet unfolded → anti-patterns → self-check.

> Task 1.2 told you *why* multi-agent works (hub-and-spoke, isolation). Task 1.3 is the *how*: the exact mechanism for spawning a subagent, getting context into it, and running several at once.

---

## 0. The one idea everything hangs on

> **Spawning a subagent is a tool call (the Task/Agent tool). The subagent starts blank, so whatever it needs you must write into its prompt. To run several in parallel, emit several spawn calls in one response.**

Three facts, one mechanism. If you internalize "a subagent is spawned by a tool call, fed only by its prompt, and parallelized by emitting multiple calls at once," every bullet below follows.

---

## What this task is even about (plain English)

- **Spawn:** to start up a new subagent. (Like opening a new worker process.)
- **Invocation:** the act of calling/triggering a subagent to run.
- **The Task tool (now also called the Agent tool):** the built-in tool an agent uses to *create and run a subagent*. When the coordinator wants to delegate, it "calls the Task tool" the same way it would call any other tool — except the tool's effect is "run a whole subagent and give me back its answer." *(Naming note: the tool was historically `Task`; in current Claude Code it's also referred to as `Agent`. The cert uses "Task." Detection code should match both names.)*
- **`allowedTools`:** the list of tools an agent is permitted to use. If `Task`/`Agent` isn't on the coordinator's list, **it physically cannot spawn subagents.**
- **Context passing:** getting information *into* a subagent. Since subagents start blank, you "pass context" by writing it into the prompt you hand them.
- **AgentDefinition:** the configuration that *defines* a subagent type — its description, its system prompt (instructions), and which tools it's allowed to use.
- **System prompt:** the standing instructions that tell an agent who it is and how to behave ("You are a security reviewer. Be specific.").
- **Parallel:** running multiple subagents at the same time instead of one after another.
- **Fork (fork_session):** making a copy of a session so you can explore two different directions from the same starting point without them interfering.

---

## Knowledge area 1 — The Task tool spawns subagents; `allowedTools` must include "Task"

A coordinator spawns a subagent by **emitting a `Task` (Agent) tool call** — structurally identical to any `tool_use` from Task 1.1. The difference: executing this tool means "spin up the named subagent, run its whole loop, and return its final message as the tool result."

**The gotcha the cert is testing:** the coordinator can only do this if **`Task` (or `Agent`) is in its `allowedTools`.** Leave it out and a very confusing thing happens — the coordinator just *does the work itself* instead of delegating, with no error. (Other causes of silent non-delegation: a vague subagent description, or not naming the subagent in the prompt.)

```python
# Coordinator MUST list Agent/Task or it can't delegate
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep", "Agent"],   # <-- without this, no subagents
    agents={ ... },                                     # subagent definitions (below)
)
```

**Related rule:** a subagent **cannot spawn its own subagents.** Don't put `Task`/`Agent` in a subagent's tool list — only the coordinator gets it. (Keeps the hub-and-spoke flat: one hub, not nested hubs.)

---

## Knowledge area 2 — Subagent context must be explicit in the prompt (no inheritance, no shared memory)

Restating Task 1.2's key fact, now as a hard rule you build around:

> **The only channel from coordinator to subagent is the prompt string in the Task call. A subagent's context starts fresh — it does not inherit the parent's history, and it does not share memory between invocations.**

Two implications:
- **No inheritance:** the subagent can't see the coordinator's conversation. If it needs a file path, an error message, or a prior finding, that text must be *inside the prompt you pass.*
- **No shared memory between invocations:** call the same subagent twice and the second call doesn't remember the first. Each invocation is blank again. (You re-supply what it needs each time.)

On the way back, **only the subagent's final message returns** to the coordinator (as the Task tool result) — not its internal steps.

---

## Knowledge area 3 — AgentDefinition: description, system prompt, tool restrictions

You define each subagent type with an **AgentDefinition**, which has these parts:

| Field | What it does | Why it matters |
|---|---|---|
| `description` | A short note telling the coordinator **when to use this subagent** | This is how the coordinator *chooses* the right subagent. Vague description → coordinator ignores it. |
| `prompt` (system prompt) | The subagent's standing instructions / expertise | Defines its behavior and specialty |
| `tools` | The restricted set of tools this subagent may use | Least privilege — a read-only analyst gets `Read, Grep, Glob`; a test-runner also gets `Bash` |
| `model` (optional) | Override the model for this subagent | Use a cheaper/stronger model per role |

```python
agents = {
  "web-search": AgentDefinition(
     description="Searches the web for recent sources. Use for current-events research.",
     prompt="You are a web research specialist. Return findings each paired with its source URL.",
     tools=["WebSearch", "WebFetch"],
  ),
  "doc-analysis": AgentDefinition(
     description="Analyzes uploaded documents. Use when the user provided files.",
     prompt="You are a document analyst. Extract claims and cite document name + page.",
     tools=["Read", "Grep", "Glob"],
  ),
}
```

The `description` is doing the same job the tool description did in Task 1.1: it's how the model decides *whether and when* to use this capability.

---

## Knowledge area 4 — Fork-based session management for divergent approaches

A **session** is a saved, ongoing conversation/agent state. **Forking** a session makes an independent *copy* of it so you can branch off in two directions from the **same shared baseline** without the branches affecting each other.

Why you'd want this: suppose a subagent has just spent effort analyzing a codebase. Now you want to try **two different refactoring strategies** on top of that analysis. Rather than re-doing the analysis twice, you **fork** the session after the analysis, and each fork explores one strategy from the identical starting point. (Covered more in Task 1.7; here it's listed as a way to spawn divergent explorations from a common base.)

---

## Skill 1 — Pass complete prior findings *into* the subagent's prompt

**What to do:** when a later subagent depends on earlier ones, include the earlier agents' **complete results directly in the later subagent's prompt.**

Concretely: the synthesis subagent needs the web-search results and the document-analysis results. Since it starts blank, the coordinator builds its prompt like:

```
Synthesize the following into a coherent answer.

WEB FINDINGS:
- "Remote work cut average commute by 54 min/day" [source: bls.gov/...]
- ...
DOCUMENT FINDINGS:
- "Productivity rose 13% in the study group" [doc: WFH_study.pdf, p.4]
- ...
```

**Why it matters:** there's no shared memory — if you don't paste the findings in, the synthesis agent literally has nothing to synthesize.

---

## Skill 2 — Use structured formats to separate content from metadata (preserve attribution)

**What to do:** when passing findings between agents, don't mash everything into loose prose. Keep the **content** (the claim) separate from its **metadata** (where it came from: source URL, document name, page number) using a structured format.

```json
{ "claim": "Productivity rose 13% in the study group",
  "source": { "doc": "WFH_study.pdf", "page": 4 } }
```

**Why it matters:** the final report must be **cited**. If a finding loses its source as it passes from the search agent → coordinator → synthesis → report, you can't attribute it, and the citation is gone. Keeping `{claim, source}` structured at every hand-off **preserves attribution** end-to-end. (This is the same "pass sources forward" rule from the Task 1.1 multi-agent scenario, now stated as a skill.)

---

## Skill 3 — Spawn parallel subagents by emitting multiple Task calls in ONE response

**What to do:** to run subagents *at the same time*, the coordinator emits **several `Task` tool calls in a single response** — not one call, wait, then another call on the next turn.

```
Coordinator's single assistant turn  ->  stop_reason: "tool_use"
  tool_use: Task(agent="web-search",   task="slice A")
  tool_use: Task(agent="web-search",   task="slice B")
  tool_use: Task(agent="doc-analysis", task="the PDFs")
# All three run in parallel; results come back together.
```

**Why it matters:** emitting them across separate turns forces them to run **one after another** (serial), which is slower. Multiple calls in one turn lets the independent work happen **simultaneously.** (You define the capability; Claude decides the actual scheduling.) Reserve parallelism for genuinely *independent* slices — if B needs A's output, they must be sequenced.

---

## Skill 4 — Coordinator prompts specify goals + quality criteria, not step-by-step procedures

**What to do:** write the coordinator's prompt in terms of **what good looks like** (goals and quality bars), not a rigid script of exact steps.

- **Procedural (avoid):** "First call web-search with query X. Then call doc-analysis on file Y. Then call synthesis. Then stop."
- **Goal + criteria (prefer):** "Produce a comprehensive, well-cited answer on <topic>. Cover the major subtopics, use multiple source types, and ensure every claim has a source. Delegate to subagents as needed."

**Why it matters:** rigid procedures make the subagents (and coordinator) brittle — they can't adapt when reality differs from your script. Goals + quality criteria let the agents **adapt** their approach to what they actually find, which is the whole point of using agents instead of a hardcoded program. (Same lesson as Task 1.1's model-driven vs decision-tree distinction.)

---

## Anti-patterns to avoid

- **Forgetting `Task`/`Agent` in `allowedTools`** — the coordinator silently does the work itself and never delegates. Always include it (and name the subagent in the prompt, with a clear description).
- **Putting `Task`/`Agent` in a subagent's tools** — lets subagents spawn subagents (nested chains, breakage). Only the coordinator delegates.
- **Assuming inheritance** — expecting a subagent to "already know" prior context. It starts blank; pass everything in the prompt.
- **Losing metadata** — passing findings as loose prose so source URLs/pages drop out. Keep `{content, metadata}` structured to preserve citations.
- **Serial-by-accident** — emitting Task calls on separate turns when the work is independent. Emit them in one turn for parallelism.
- **Over-scripting the coordinator** — step-by-step procedural prompts that remove the agents' ability to adapt. Specify goals + quality criteria.

---

## Worked example — the synthesis hand-off

```python
# 1. Parallel fan-out: ONE coordinator turn, multiple Task calls
#    Task(web-search, "AI hiring trends")  +  Task(doc-analysis, "the uploaded PDFs")

# 2. Results return to the coordinator (final messages only).

# 3. Coordinator builds the SYNTHESIS subagent's prompt by pasting in the findings,
#    keeping each claim tied to its source (structured):
synthesis_prompt = """
Synthesize a cited answer on "AI and hiring".
FINDINGS (each with its source — preserve attribution):
[
  {"claim": "...", "source": {"url": "..."}},
  {"claim": "...", "source": {"doc": "report.pdf", "page": 7}}
]
Goal: comprehensive coverage; every sentence in the output must trace to a source.
"""
# 4. Task(synthesis, synthesis_prompt)  -> draft returns to coordinator.
```

Every requirement of the task is visible here: Task tool to spawn, explicit context in the prompt, structured `{claim, source}` for attribution, parallel fan-out in one turn, and a goal-oriented prompt.

---

## Self-check

1. How does a coordinator spawn a subagent? → By emitting a **`Task`/`Agent` tool call** (a `tool_use`); the result is the subagent's final message.
2. What must be in `allowedTools` or delegation silently fails? → **`Task` (or `Agent`)**.
3. Can a subagent spawn subagents? → **No** — don't give it the Task tool.
4. How does context get into a subagent? → **Explicitly in its prompt** — no inheritance, no shared memory between invocations.
5. What three things does an AgentDefinition set? → **description** (when to use it), **prompt/system prompt** (behavior), **tools** (restricted set) — plus optional model.
6. How do you preserve citations across hand-offs? → Pass findings as **structured `{content, metadata}`** so source URLs/doc/page survive.
7. How do you run subagents in parallel? → Emit **multiple Task calls in one response**, not across turns.
8. How should the coordinator prompt be written? → **Goals + quality criteria**, not step-by-step procedures, so subagents can adapt.
9. What is forking for? → Branching from a **shared baseline** to explore divergent approaches without interference.

---

## Deep dive — Task 1.3 in each of the six scenarios

Task 1.3 is about the **mechanics of spawning**: the Task/Agent tool (and `allowedTools`), putting context into the prompt, `AgentDefinition`, parallel calls, and forking. Each scenario below is unfolded with a plain-English explanation, the theory of how these mechanics play out, code where it clarifies, and a takeaway.

### Scenario 1 — Customer Support Resolution Agent · *Partial fit*
> *You are building a customer support resolution agent using the Claude Agent SDK. The agent handles high-ambiguity requests like returns, billing disputes, and account issues. It has access to your backend systems through custom MCP tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`). Your target is 80%+ first-contact resolution while knowing when to escalate.*

**What's happening here (plain English):** a support chat agent that looks customers up and resolves or escalates their issues using backend tools.

**How the spawning mechanics work here (theory):** when a ticket has multiple concerns, the agent spawns sub-investigators. Two Task-1.3 rules bite immediately:
1. **`Task`/`Agent` must be in `allowedTools`** or the agent silently does everything itself instead of delegating.
2. **Subagents start blank** — so the verified customer context (the ID from `get_customer`) must be **written into each sub-investigator's prompt**; it is not inherited.

To investigate concerns simultaneously, the agent emits **multiple Task calls in one response** (parallel), not one per turn.

```python
# One coordinator turn → two parallel sub-investigations, each handed the customer context
Task(agent="order-investigator",
     task="Customer CUST-48213 (verified). Investigate the broken-item claim on order #4471.")
Task(agent="billing-investigator",
     task="Customer CUST-48213 (verified). Investigate the duplicate charge on 2026-06-10.")
```

**Takeaway:** delegation only works with `Task`/`Agent` allowed, the verified context must be pasted into each subagent's prompt, and parallel concerns go out as multiple calls in one turn.

### Scenario 2 — Code Generation with Claude Code · *Strong fit*
> *You are using Claude Code to accelerate software development. Your team uses it for code generation, refactoring, debugging, and documentation. You need to integrate it into your development workflow with custom slash commands, CLAUDE.md configurations, and understand when to use plan mode vs direct execution.*

**What's happening here (plain English):** Claude Code editing real project files for a dev team; for a multi-file job it can delegate to specialist subagents.

**How the spawning mechanics work here (theory):** you define each specialist with an **`AgentDefinition`** — its `description` (when the main agent should use it), its `prompt` (how it behaves), and its restricted `tools`. A read-only "explore" agent gets `Read/Grep/Glob`; a "test-writer" also gets `Write`. The main session needs `Agent`/`Task` in `allowedTools` to invoke them, and because subagents don't inherit history, the **file paths and decisions already made must be written into each subagent's prompt.**

```python
options = ClaudeAgentOptions(
  allowed_tools=["Read","Grep","Glob","Agent"],          # Agent required to delegate
  agents={
    "explore":     AgentDefinition(description="Map a module. Use before changes.",
                                   prompt="Read-only. Summarize structure + key functions.",
                                   tools=["Read","Grep","Glob"]),
    "test-writer": AgentDefinition(description="Write tests for a given file.",
                                   prompt="Given a file path, write thorough tests.",
                                   tools=["Read","Write"]),
  })
```

**Takeaway:** `AgentDefinition` is how you create specialists with least-privilege tools; remember the `Agent` tool in `allowedTools` and pass paths explicitly.

### Scenario 3 — Multi-Agent Research System · *Core fit (canonical)*
> *You are building a multi-agent research system using the Claude Agent SDK. A coordinator agent delegates to specialized subagents: one searches the web, one analyzes documents, one synthesizes findings, and one generates reports. The system researches topics and produces comprehensive, cited reports.*

**What's happening here (plain English):** the AI research team — coordinator delegating to search/analysis/synthesis/report specialists.

**How the spawning mechanics work here (theory):** every Task-1.3 skill appears:
- **Spawn** search and analysis via the Task tool, **in parallel in one turn** (they're independent).
- **Pass complete prior findings into the synthesis subagent's prompt** — since it starts blank, the coordinator pastes the search and analysis results in.
- **Keep `{claim, source}` structured** at every hand-off so attribution survives into the cited report.
- **Write the coordinator's prompt as goals + quality criteria** ("comprehensive, every claim sourced"), not a rigid script, so the subagents adapt.

```python
synthesis_prompt = """Synthesize a cited answer on <topic>.
FINDINGS (preserve each source):
[ {"claim":"...","source":{"url":"..."}},
  {"claim":"...","source":{"doc":"x.pdf","page":4}} ]
Goal: comprehensive coverage; every sentence must trace to a source."""
Task(agent="synthesis", task=synthesis_prompt)
```

**Takeaway:** research is the textbook case — parallel spawn, explicit findings-in-prompt, structured sources for citations, goal-oriented coordinator prompt.

### Scenario 4 — Developer Productivity with Claude · *Strong fit*
> *You are building developer productivity tools using the Claude Agent SDK. The agent helps engineers explore unfamiliar codebases, understand legacy systems, generate boilerplate code, and automate repetitive tasks. It uses the built-in tools (Read, Write, Bash, Grep, Glob) and integrates with MCP servers.*

**What's happening here (plain English):** an AI helper that maps unfamiliar/legacy code and writes repetitive starter code.

**How the spawning mechanics work here (theory):** define a **read-only explorer** `AgentDefinition` (`Read/Grep/Glob`, deliberately *no* `Write` — it can analyze anything and damage nothing) and a separate **boilerplate-writer** (`Write`). To map a big system, **fan out parallel per-module exploration** (multiple Task calls in one turn), each subagent handed its target path explicitly. When you then want to try two implementation directions from the same analysis, **fork the session** so both branches start from the identical baseline (this bridges to Task 1.7).

**Takeaway:** least-privilege `AgentDefinition`s, parallel per-module spawning with explicit paths, and `fork_session` to branch from a shared analysis.

### Scenario 5 — Claude Code for Continuous Integration · *Partial fit*
> *You are integrating Claude Code into your CI/CD pipeline. The system runs automated code reviews, generates test cases, and provides feedback on pull requests. You need to design prompts that provide actionable feedback and minimize false positives.*

**What's happening here (plain English):** an automated reviewer on every proposed code change (pull request).

**How the spawning mechanics work here (theory):** for a large PR, spawn **per-file review subagents as parallel Task calls in one turn**, each handed **its file's diff explicitly in the prompt** (no shared memory — the diff must be in the prompt). Instruct each to return **structured findings with `file:line` metadata** so, when the coordinator aggregates, the consolidated PR comment keeps exact locations. Goal-oriented prompts ("report only high-confidence issues") keep false positives down.

**Takeaway:** parallel per-file spawns, each fed its diff, returning structured `file:line` findings the hub can merge into one located comment.

### Scenario 6 — Structured Data Extraction · *Partial fit*
> *You are building a structured data extraction system using Claude. The system extracts information from unstructured documents, validates the output using JSON schemas, and maintains high accuracy. It must handle edge cases gracefully and integrate with downstream systems.*

**What's happening here (plain English):** turning documents into tidy JSON records for downstream systems.

**How the spawning mechanics work here (theory):** at batch scale, spawn extraction subagents and **pass each document into the prompt** (blank start = the document text must be in the prompt). Have each return a **structured record with source metadata** (document name, page) so attribution and validation are possible downstream. **Fork** the session to A/B two schema variants on the same document set without redoing setup. (Note the overlap with Task 1.1's extraction guide: the structured output itself is best produced via strict tool use / `output_format`.)

**Takeaway:** feed each document explicitly into its subagent, return structured records with source metadata, and fork to compare schema variants.

**One-line cross-scenario takeaway:** the spawning mechanics are *core* to research (S3) and used heavily wherever you delegate — but the rule that bites everywhere is the same: **`Task`/`Agent` in `allowedTools`, and every bit of context the subagent needs must be in its prompt.**

---

### Sources verified against current Anthropic docs (June 2026)
- *Subagents in the SDK* / *Agent SDK overview* (platform.claude.com, code.claude.com) — `Agent`/`Task` tool required in `allowedTools`; `AgentDefinition` (`description`, `prompt`, `tools`, optional `model`); subagent context starts fresh, prompt is the only inbound channel; only the final message returns; subagents can't spawn subagents; parallel via multiple calls.
- *Sessions guide* — `fork_session`/`forkSession` for history-copying branches (see Task 1.7).
- Tool-naming note (`Task` → `Agent`, Claude Code v2.1.63) verified June 2026; SDK still emits `Task` in some places, so match both. This API moves fast — re-verify field names against current docs.
