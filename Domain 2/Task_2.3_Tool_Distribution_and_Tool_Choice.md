# Task 2.3 — Distributing Tools Across Agents & Configuring Tool Choice
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **Give each agent only the few tools its role needs, and use `tool_choice` to control *whether* and *which* tool fires. Too many tools (18 instead of 4–5) drowns the model in decision complexity and selection gets unreliable; tools outside an agent's specialty get misused. Scope tightly, hand out narrow cross-role tools only for high-frequency needs, and use `tool_choice` for hard guarantees.**

Two levers: **distribution** (which tools each agent even has) and **tool_choice** (how the model is allowed to choose among the tools it has). Task 2.1 made each tool legible; this task controls *how many* and *whether forced*.

---

## 1. Too many tools degrades selection

Selection reliability falls as the tool count rises — each extra tool is another option the model must consider and discriminate against. An agent with 18 tools makes more selection mistakes than the same agent scoped to the 4–5 it actually uses. The fix is **scoped tool access**: give an agent only the tools for its role. Fewer, sharper options → more reliable selection. (Tool Search can defer *definitions* to save context, but the decision-complexity problem is about how many *relevant* tools the model is choosing among.)

---

## 2. Out-of-specialty tools get misused

An agent will try to use a tool it has, even when it shouldn't. A **synthesis** agent given a web-search tool tends to run searches mid-synthesis — going off-task, duplicating the search agent's job, and degrading the synthesis. The lesson: don't hand an agent capabilities outside its role "just in case." Capability you grant is capability that will be (mis)used.

---

## 3. Scoped access with limited cross-role tools

Pure isolation is too rigid: sometimes a specialist hits a **high-frequency** need for one cross-role capability. The pattern: scope tightly **and** grant a *single narrow* cross-role tool for that frequent need, routing genuinely complex cases back through the coordinator.

Example: the synthesis agent often needs to confirm one fact. Don't give it full web search; give it a constrained `verify_fact` tool (one claim in, verdict out). Frequent need met, no general-search misuse, complex research still routed to the search subagent via the coordinator.

---

## 4. Constrained tools beat generic ones

Replacing a generic tool with a **constrained** one bakes the boundary into the interface, not the prompt. `fetch_url` (fetch anything) → `load_document` (validates that the URL is an allowed document and rejects the rest). The agent can't wander off because the tool itself won't let it. This pairs with Task 2.1's "split generic tools" and Task 1.4's "least privilege."

---

## 5. `tool_choice` — controlling whether/which (the four modes)

`tool_choice` is a Messages-API parameter controlling tool calling on a turn:

| `tool_choice` | Meaning | Use for |
|---|---|---|
| `{"type":"auto"}` | Model decides whether to call a tool (default when tools present) | Normal agent turns |
| `{"type":"any"}` | Must call **some** tool (model picks which) | Guarantee an action, not chit-chat |
| `{"type":"tool","name":"X"}` | Must call **tool X** | Force a specific tool first |
| `{"type":"none"}` | May not call any tool (default when no tools) | Force a text-only turn |

Key behaviors:
- With `any` or `tool`, the API **prefills** the assistant turn, so the model emits the tool call with **no preceding prose** — even if asked for explanation. Want prose + a call? Use `auto` and instruct in the prompt.
- Add `"disable_parallel_tool_use": true` to any mode to force **at most one** tool call that turn (with `any`/`tool`, exactly one).
- Forcing a tool is a single-turn lever: to force tool A *then* process B, force A this turn, read its result, continue (often `auto`) next turn. (Extended thinking is incompatible with `any`/`tool` — only `auto`/`none`.)

---

## 6. Anti-patterns (with *why*)
- **One mega-agent with 18 tools.** Decision complexity tanks selection; split by role and scope.
- **Granting out-of-role tools "for flexibility."** They get misused (synthesis agent web-searching). Grant only role tools.
- **Generic `fetch_url` everywhere.** Lets the agent wander; replace with a validating `load_document`.
- **Leaving `tool_choice` on auto when an action is mandatory.** Model may reply with prose instead of calling the tool; use `any`.
- **Expecting prose alongside a forced call.** `any`/`tool` suppress preamble; use `auto` + prompt if you need both.
- **Forcing tool A and expecting B in the same turn.** Forcing is per-turn; sequence across turns.

---

## 7. Self-check (core mechanics)
1. Why does an 18-tool agent select worse than a 4-tool one? → More options = more decision complexity = less reliable selection.
2. Why not give a synthesis agent web search? → Out-of-specialty tools get misused (off-task searches).
3. Pattern for a frequent cross-role need? → A single narrow scoped tool (e.g., `verify_fact`), complex cases via the coordinator.
4. `fetch_url` → ? → `load_document` that validates URLs (constrained replacement).
5. Guarantee the model calls *some* tool? → `tool_choice:{"type":"any"}`.
6. Force a specific tool first? → `{"type":"tool","name":"X"}`, then continue next turn.
7. Why no prose before a forced call? → `any`/`tool` prefill the turn, suppressing preamble.

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through four backend tools (see Domain-1 §7).

**The key link to everything above:** **Core fit.** Four tools is already the right size (§1) — the lesson is *resisting tool sprawl* as the agent grows, and using `tool_choice` to enforce ordering. You can force `get_customer` first on a new ticket (`{"type":"tool","name":"get_customer"}`) so verification always precedes action — the Task-1.4 gate expressed via tool_choice. And `any` guarantees the agent acts rather than chatting.

### 8a. Force verification first
On a refund-bearing ticket, force `get_customer` this turn, read the result, then continue — identity is established before `process_refund` is even an option.

### 8b. Resist sprawl
Don't bolt on 14 more niche tools; keep the role scoped to the four, splitting into specialized agents if scope grows.

### 8c. Scenario-1 self-check
1. Force identity check first? → `tool_choice:{"type":"tool","name":"get_customer"}` that turn.
2. Guarantee an action not chit-chat? → `{"type":"any"}`.
3. (Goal tie) How does scoping serve 80% FCR? → A small, sharp tool set selects reliably under ambiguity, so the agent acts correctly more often.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Strong fit — the forced-sequence example.** The stages map to `tool_choice`: force `extract_metadata`/`plan` first, then let `auto` drive write/test in follow-up turns (the exam's "force `extract_metadata` before enrichment, process subsequent steps in follow-up turns"). `any` guarantees each stage produces a tool action rather than a prose stall.

### 9a. Force the first step, then auto
Turn 1: `{"type":"tool","name":"plan"}`. Subsequent turns: `auto` for write/test. Sequencing via tool_choice, one forced step per turn.

### 9b. Scenario-2 self-check
1. Ensure planning happens before writing? → Force the plan tool on turn 1, then `auto`.
2. Can you force plan then write in one turn? → No; forcing is per-turn — sequence across turns.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research, then synthesizing (see Domain-1 §9).

**The key link to everything above:** **Core fit — the marquee distribution example.** This is where "synthesis agent attempting web searches" (§2) lives. Scope each subagent: the **search** agent gets search tools, the **synthesis** agent gets *none* of them — except a narrow `verify_fact` for its frequent need (§3), with complex lookups routed back through the coordinator. Distribution by role is the whole point.

### 10a. Role-scoped tool sets
Search agent: web/search tools. Analysis agent: document tools. Synthesis agent: `verify_fact` only. No agent holds another's tools.

### 10b. The `verify_fact` cross-role tool
A single constrained tool meets synthesis's frequent fact-check need without granting general search — exactly §3.

### 10c. Scenario-3 self-check
1. Why no search tool on the synthesis agent? → It would run off-task searches (out-of-specialty misuse).
2. How is its frequent fact-check need met? → A narrow `verify_fact` tool; complex cases via the coordinator.
3. General principle? → Scope tools to role; limited cross-role tools for high-frequency needs.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores codebases, handles legacy, generates boilerplate, automates chores.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Core fit.** Built-ins plus many MCP servers is exactly the 18-tool trap (§1). Scope by sub-task: a read-only **exploration** agent gets `Read/Grep/Glob` only (no `Write/Bash`); an **editing** agent gets `Edit/Write`. Replace a generic `fetch_url` with a validating `load_document` (§4). Fewer, role-appropriate tools = better selection and safer behavior.

### 11a. Scope read vs write agents
Exploration: `Read`, `Grep`, `Glob`. Editing: `Edit`, `Write`, scoped `Bash`. No agent gets everything.

### 11b. Constrain the generic
Swap `fetch_url` for `load_document` so the agent can't fetch arbitrary URLs — boundary in the tool, not the prompt.

### 11c. Scenario-4 self-check
1. Symptom of too many tools here? → Unreliable selection across built-ins + MCP.
2. Fix? → Scope tool sets per sub-task role.
3. `fetch_url` replacement? → A validating `load_document`.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Strong fit.** Two levers apply: scope the reviewer to read/analysis tools only (no write/merge — least privilege, §4), and use `tool_choice:{"type":"any"}` (or a forced output tool) to **guarantee** the agent emits the structured review rather than returning conversational text the gating script can't parse. Reliability in headless mode comes from constraining choice.

### 12a. Force structured output
`{"type":"any"}` or a forced `emit_review` tool guarantees a tool call (the JSON), never a prose reply — critical when a script must read the result.

### 12b. Scope to read-only
The reviewer holds only read/analysis tools; it structurally cannot modify the repo.

### 12c. Scenario-5 self-check
1. Guarantee JSON not prose in headless CI? → `tool_choice:{"type":"any"}` / forced output tool.
2. Reviewer tool scope? → Read/analysis only (no write/merge).

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Core fit — the canonical forced-tool example.** Forcing a specific extraction tool with `{"type":"tool","name":"extract_data_points"}` guarantees structured output every time (the "force `extract_metadata` before enrichment" pattern), and pairs with strict tool use so inputs follow the schema. Scope the extractor agent to just its extract/validate tools — no stray capabilities to dilute selection or accuracy.

### 13a. Force the extractor (+ strict schema)
`{"type":"tool","name":"extract_data_points"}` makes the model always produce the structured record; combine with strict tool use so the output conforms to the schema — directly serving "high accuracy."

### 13b. Then enrich in follow-up turns
Force extract first; run enrichment/validation tools in subsequent turns (forcing is per-turn).

### 13c. Scenario-6 self-check
1. Guarantee a structured record every call? → Force the extraction tool (`{"type":"tool","name":...}`), ideally with strict tool use.
2. Force extract then enrich in one turn? → No; sequence across turns.
3. (Goal tie) How does forced tool_choice support "high accuracy"? → It removes the chance of a prose answer or wrong tool, so every call yields the schema-shaped output.

---

### Sources verified against current Anthropic docs (June 2026)
- *Define tools / Implement tool use* (platform.claude.com) — `tool_choice` modes `auto` / `any` / `{"type":"tool","name":...}` / `none`; `any`/`tool` **prefill** the assistant turn (no preamble prose); `disable_parallel_tool_use: true` forces at most one call; `any`/`tool` incompatible with extended thinking; combine `any` with strict tool use to guarantee a schema-conforming call.
- *Agent design / subagents* — scoping tool sets per role; too many tools degrades selection; Tool Search defers definitions for large libraries.
- The "18 vs 4–5 tools," out-of-specialty misuse, `verify_fact` cross-role tool, and `fetch_url`→`load_document` examples come from the task statement; verify exact SDK option names against current docs when implementing.
