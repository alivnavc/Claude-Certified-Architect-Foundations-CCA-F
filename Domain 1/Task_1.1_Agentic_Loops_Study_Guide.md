# Task 1.1 — Agentic Loops for Autonomous Task Execution
### Study Guide (grounded in the Customer Support Resolution Agent scenario)

**Reading order:** each section builds on the one before it. 1–5 are the *mechanics* (what's happening in code). 6–10 are the *concepts and exam material*. Read top to bottom the first time.

---

## 0. The one idea everything hangs on

> **The model decides *what* to do. Your code does it. The loop's only on/off switch is `stop_reason`.**

Tool use is a *contract*: you declare which operations exist and what their inputs/outputs look like; Claude decides *when* and *how* to call them. The model never runs anything itself — it emits a structured request, your code runs the operation, and the result flows back into the conversation. Then the model looks at that result and decides the next move.

Every knowledge bullet, skill, and anti-pattern below is a direct consequence of this. Keep coming back to it.

---

## 1. Anatomy of one response (read this before the loop)

When you ask Claude something, you get back **one response object** — just data. Two fields matter:

- **`response.stop_reason`** — a single text label answering *"why did Claude stop talking just now?"* It is **exactly one** word per response (like a traffic light showing one color), never a list. The two you care about: `"tool_use"` (not done) and `"end_turn"` (done).
- **`response.content`** — a **list of blocks**. Each block has a `.type`:

| `block.type` | What it is | Fields on it |
|---|---|---|
| `"text"` | Claude chatting / reasoning out loud | `block.text` |
| `"tool_use"` | Claude's actual request to run a tool | `block.id`, `block.name`, `block.input` |

For a `tool_use` block:
- `block.name` → which tool, e.g. `"lookup_order"`
- `block.input` → a **dictionary** of arguments Claude chose, e.g. `{"order_id": "4471"}`
- `block.id` → a unique id like `"toolu_01XYZ"` you must echo back later so Claude knows which result answers which call

**Response when Claude wants a tool** — note `content` holds **both** a text block and a tool_use block at the same time:

```json
{
  "id": "msg_01AbC...",
  "role": "assistant",
  "stop_reason": "tool_use",          // THE LABEL -> loop must CONTINUE
  "content": [
    { "type": "text", "text": "Sure, let me pull up that order for you." },
    { "type": "tool_use", "id": "toolu_01XYZ",
      "name": "lookup_order", "input": { "order_id": "4471" } }
  ]
}
```

**Response when Claude is finished** — same structure, different label:

```json
{
  "id": "msg_01DeF...",
  "role": "assistant",
  "stop_reason": "end_turn",          // now end_turn -> loop STOPS
  "content": [
    { "type": "text", "text": "Done! I've refunded $89.99 for order #4471." }
  ]
}
```

Text and tool_use living in the **same** response is exactly why "is there text?" can never be your stop signal — only `stop_reason` can.

---

## 2. The `stop_reason` values

| Value | Meaning | What your loop does |
|---|---|---|
| `"tool_use"` | Claude emitted one or more `tool_use` blocks; it needs a tool run. Turn is **incomplete**. | Execute tools, append results, **continue**. |
| `"end_turn"` | Claude finished its turn on its own. | **Terminate** the loop; return the final message. |
| `"max_tokens"` | Output was cut off by the `max_tokens` limit. | Not a clean finish — raise the limit or continue generation. |
| `"stop_sequence"` | A custom stop sequence you set was hit. | Handle per your design. |
| `"pause_turn"` | A **server-tool** internal loop (e.g. web_search) hit its iteration cap mid-work. | Send the response back to let Claude continue. Mostly for server tools, not your MCP tools. |
| `"refusal"` | Claude declined to generate. On some models this is a normal 200 response, not an error. | Surface gracefully / rephrase; don't treat as a tool turn. |

**For this task the two that matter are `tool_use` (continue) and `end_turn` (stop).** The others exist so you don't accidentally treat them as either.

---

## 3. The agentic loop

A single user message can require many round-trips before the agent is "done." Each round-trip is one iteration:

1. **Send** the conversation (system prompt + messages + `tools`) to Claude.
2. **Inspect `stop_reason`.** `"tool_use"` → not finished; `"end_turn"` → finished, exit.
3. **Execute** the requested tool(s) in your code (or, for MCP, via the connected server).
4. **Append** the tool result(s) back into the conversation history.
5. **Repeat** from step 1 with the now-larger conversation.

The canonical shape is a `while` loop keyed on `stop_reason`. This single annotated version shows the control flow **and** what each variable holds at each line:

```python
messages = [{"role": "user", "content": "I want a refund on order #4471"}]
# messages -> [ {"role":"user", "content":"I want a refund on order #4471"} ]

while True:
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        tools=tools,            # get_customer, lookup_order, process_refund, escalate_to_human
        messages=messages,
    )
    # response.stop_reason -> "tool_use"
    # response.content     -> [ TextBlock, ToolUseBlock ]   (a LIST of blocks)

    if response.stop_reason == "tool_use":

        # 1. record what Claude said/asked for
        messages.append({"role": "assistant", "content": response.content})

        # 2. run every tool Claude requested this turn
        tool_results = []
        # tool_results -> []   (empty, we fill it below)

        for block in response.content:
            # pass 1: block.type -> "text"
            #         block.text -> "Let me pull up that order."
            # pass 2: block.type  -> "tool_use"
            #         block.id    -> "toolu_01XYZ"
            #         block.name  -> "lookup_order"
            #         block.input -> {"order_id": "4471"}   (a dict)

            if block.type == "tool_use":                  # skip text, act on tool requests
                result = run_tool(block.name, block.input)   # YOUR code executes
                # result -> "Order 4471: $89.99, delivered 2026-06-10, refund-eligible"

                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,   # "toolu_01XYZ" -> MUST match block.id above
                    "content": result,
                })

        # 3. feed results back as the next user turn
        messages.append({"role": "user", "content": tool_results})
        # loop repeats -> model now reasons WITH the new data
        # (next response will likely be tool_use -> process_refund)

    else:
        # response.stop_reason -> "end_turn"  ->  we're done
        break

final_text = response.content
# final_text -> [ {"type":"text", "text":"Done! I've refunded $89.99 for order #4471."} ]
```

**Read the loop condition out loud:** `stop_reason == "tool_use"`. No counting, no keyword-matching, no reading Claude's prose. That is the entire point of the task.

---

## 4. How the conversation grows (the `messages` list)

Conversation history is the model's **only memory**. For Claude to "reason about the next action," the new information must physically be in `messages` before the next `create()` call. The list **only grows** — each iteration adds one assistant turn and one tool-result user turn.

Watch `messages` build across one tool call:

```python
# START — just the customer's request
messages = [
  {"role": "user", "content": "I want a refund on order #4471"}
]

# AFTER appending the assistant turn (text + the tool request)
messages = [
  {"role": "user", "content": "I want a refund on order #4471"},
  {"role": "assistant", "content": [
      {"type": "text", "text": "Let me pull up that order."},
      {"type": "tool_use", "id": "toolu_01XYZ",
       "name": "lookup_order", "input": {"order_id": "4471"}}
  ]}
]

# AFTER running the tool and appending the paired result
messages = [
  {"role": "user", "content": "I want a refund on order #4471"},
  {"role": "assistant", "content": [
      {"type": "text", "text": "Let me pull up that order."},
      {"type": "tool_use", "id": "toolu_01XYZ",
       "name": "lookup_order", "input": {"order_id": "4471"}}
  ]},
  {"role": "user", "content": [
      {"type": "tool_result", "tool_use_id": "toolu_01XYZ",   # matches the id above
       "content": "Order 4471: $89.99, delivered 2026-06-10, refund-eligible"}
  ]}
]
```

The loop sends this **whole list** back. Claude reads the `tool_result`, sees "refund-eligible," and its next response is a *new* `tool_use` block for `process_refund` — repeating until `stop_reason` is `"end_turn"`.

**Pairing rules that trip people up:**
- The **assistant** turn holds the `tool_use` block (with an `id`); the **next user** turn holds a `tool_result` block whose `tool_use_id` equals that `id`. **The two ids must be the same string** — that pairing is how Claude matches a result to the request.
- **Every `tool_use` must get a matching `tool_result`** in the immediately following user turn. Unpaired = 400 error.
- If Claude calls **multiple tools in one turn** (parallel tool use), return **all** their results in one user turn, each with its own `tool_use_id`.
- Append the assistant turn *and* the tool-result turn — dropping the assistant turn breaks the alternation.

---

## 5. The three "levels" people all call "the SDK"

Same `stop_reason`, same place — what changes is **who reads it and who runs the loop**.

| Level | What it is | Who runs the loop? | How you read `stop_reason` |
|---|---|---|---|
| **1. Raw Messages API (HTTP)** | You POST JSON, get JSON back. No library. | **You** | Dig into JSON: `response["stop_reason"]` |
| **2. Anthropic SDK** (`anthropic` library) | Thin helper. `client.messages.create(...)` does the HTTP call, returns a tidy **object**. | **You** | `response.stop_reason` (a dot instead of JSON digging) |
| **3. Claude Agent SDK** | High-level. You give it tools + a request; it runs the *whole* loop internally. | **The SDK** | Usually you **don't** — it checks every internal turn, returns only when done |

**Level 2 is where this certification's loop code lives.** The SDK returns **one response per call**; it does **not** loop for you. You read `response.stop_reason`, run the tool, append the result, and call `create()` again yourself (Section 3's loop).

**Level 3 (Agent SDK) is the one that confuses newcomers.** It runs the loop for you — internally doing exactly what you'd do by hand:

> send → sees `tool_use` → runs the tool (e.g. your MCP tool) → appends the result → sends again → … → sees `end_turn` → stops → hands you the final answer.

So with the Agent SDK you often **never hand-inspect `stop_reason`**. The certification still tests the Level-2 mechanics because the Agent SDK is doing precisely this under the hood — you must understand it to design and debug agents.

---

## 6. Model-driven decisions vs. pre-configured decision trees

This distinction is heavily tested. Know both sides.

**Pre-configured / hardcoded (the old way):**
```python
if "refund" in user_message:
    order = lookup_order(...)
    if order.eligible:
        process_refund(...)
    else:
        escalate_to_human(...)
```
You wrote the branching. The model is barely involved. This breaks the moment a request is ambiguous ("my order's wrong AND I was double-charged") because reality doesn't match your `if`-tree.

**Model-driven (the agentic way):**
You expose the four tools, describe them well, and let Claude choose the sequence from context. With the default `tool_choice: {"type": "auto"}`, Claude decides **on each turn** whether to call a tool — and which one — or to respond directly.

```
User: "I was charged twice and never got my order."
  Claude -> get_customer            (figure out who this is)
  Claude -> lookup_order            (check the order status)
  Claude reasons: double charge + non-delivery -> beyond auto-refund policy
  Claude -> escalate_to_human       (knows when to escalate)
```

You never wrote that path. Claude assembled it from the tool descriptions and the conversation. **This is what lets the agent handle high-ambiguity requests and hit 80%+ first-contact resolution while still escalating correctly.** The architect's job shifts from *writing branches* to *writing good tool descriptions and a good system prompt*.

---

## 7. Skills (what you must be able to *do*)

**Skill A — Loop control flow keyed on `stop_reason`.** Continue while `stop_reason == "tool_use"`, terminate on `"end_turn"` (Section 3). The control flow reads `stop_reason` and nothing else to decide continue/stop.

**Skill B — Add tool results between iterations.** Append the assistant turn, run the tools, append paired `tool_result` blocks as the next user turn, then re-send (Section 4). The model incorporates the new info because it's now in the context window.

---

## 8. Anti-patterns — and *why* each is wrong

The three most likely "spot the mistake" questions. Memorize the reasoning, not just the label.

**Anti-pattern 1: Parsing natural-language signals to decide termination**
```python
if "let me know if you need anything else" in response_text:
    break   # WRONG
```
Phrasing is non-deterministic. Claude might say that mid-task, or never say it when truly done. You're inferring a control signal from prose when the API hands you a precise structured one: `stop_reason`.

**Anti-pattern 2: Arbitrary iteration cap as the *primary* stopping mechanism**
```python
for i in range(5):   # stop after 5 no matter what -- WRONG as primary control
    ...
```
A cap doesn't reflect completion. It cuts off legitimate long resolutions and masks bugs (an infinite loop "looks fine" stopping at 5). The **primary** stop must be `end_turn`. A cap is acceptable only as a *safety backstop*, never as the thing that defines "done."

**Anti-pattern 3: Checking for assistant text content as a completion indicator**
```python
if response_has_text_block(response):
    break   # WRONG
```
Claude routinely emits text *alongside* a `tool_use` block. Text present ≠ finished. A turn with both still has `stop_reason: "tool_use"` and must continue. Only `stop_reason` tells you the turn is complete.

**The through-line:** all three substitute an unreliable proxy for the one authoritative signal. Correct mental model: *"`stop_reason` is the source of truth; everything else is a guess."*

---

## 9. Full worked example — Customer Support Resolution Agent

> **Scenario 1 (as given):** *You are building a customer support resolution agent using the Claude Agent SDK. The agent handles high-ambiguity requests like returns, billing disputes, and account issues. It has access to your backend systems through custom MCP tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`). Your target is 80%+ first-contact resolution while knowing when to escalate.*

**What this scenario is even about (plain English):** Imagine the chat agent on a store's website. A customer types something messy like "my order's wrong and I think I got charged twice." A human rep would look up the customer, check the order, and either fix it or pass it to a specialist. This scenario builds an AI that does that. The four **MCP tools** are just doors into the company's systems — `get_customer` looks someone up, `lookup_order` checks an order, `process_refund` issues money back, `escalate_to_human` hands off to a person. "**First-contact resolution**" means solving it in one conversation without bouncing the customer around; "**knowing when to escalate**" means recognizing when it's too tricky and handing off. This is the running example used throughout Sections 1–8.

Tools (MCP): `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`.

**Trace for "I want a refund on order #4471":**

| Iter | `stop_reason` | Claude's action | Your code | Appended to history |
|---|---|---|---|---|
| 1 | `tool_use` | calls `get_customer` | run it | customer profile |
| 2 | `tool_use` | calls `lookup_order(4471)` | run it | "eligible, $89.99" |
| 3 | `tool_use` | calls `process_refund(4471)` | run it | "refund confirmed" |
| 4 | `end_turn` | writes confirmation to customer | **break** | — final answer |

**Contrast — "double-charged and never delivered":**

| Iter | `stop_reason` | Claude's action |
|---|---|---|
| 1 | `tool_use` | `get_customer` |
| 2 | `tool_use` | `lookup_order` → sees anomaly |
| 3 | `tool_use` | `escalate_to_human` (outside auto-refund policy) |
| 4 | `end_turn` | tells customer a specialist will follow up |

Same loop, same code, **completely different path** — chosen by the model, not your `if`-statements. That is model-driven orchestration delivering high first-contact resolution *and* correct escalation.

**Where escalation logic lives:** not in your loop — in the **tool descriptions and system prompt**. e.g. `escalate_to_human`'s description: *"Use when the request involves billing disputes, suspected fraud, or anything outside standard refund policy."* Claude reads that and routes accordingly. Tune resolution-vs-escalation by editing those descriptions, not the loop.

---

## 10. 60-second self-check (exam recall)

1. What field decides whether the loop continues? → **`stop_reason`** (`tool_use` = continue, `end_turn` = stop).
2. Where do tool results go so the model can use them? → Appended to **conversation history** as `tool_result` blocks paired by `tool_use_id`, as the next user turn.
3. Why not stop when the response contains text? → Claude emits **text alongside tool_use**; only `stop_reason` is authoritative.
4. Why is `for i in range(5)` a bad primary stop? → A cap reflects **neither completion nor correctness**; it's a backstop, not the control.
5. Model-driven vs decision tree? → Model **chooses the tool sequence from context**; a decision tree hardcodes branches and fails on ambiguity.
6. Where does "when to escalate" get decided? → In **tool descriptions + system prompt**, surfaced through the model's tool choice — not in loop code.

---

## 11. Scenario 2 — Code Generation with Claude Code

> **Scenario 2 (as given):** *You are using Claude Code to accelerate software development. Your team uses it for code generation, refactoring, debugging, and documentation. You need to integrate it into your development workflow with custom slash commands, CLAUDE.md configurations, and understand when to use plan mode vs direct execution.*

**What this scenario is even about (plain English):** **Claude Code** is a version of Claude that runs in a developer's *terminal* (the text-command window programmers use) and can actually read and change the files in their project. Instead of copy-pasting code into a chat, the developer says "add a login screen" and Claude edits the real files. The jargon: **code generation** = writing new code; **refactoring** = cleaning up existing code without changing what it does; **debugging** = finding and fixing errors; **documentation** = writing the explanations that go with code. The three things you're configuring are just *controls* for this assistant — **slash commands** are saved shortcuts (type `/review` instead of re-typing a long instruction), **CLAUDE.md** is a notes file the assistant always reads first (your project's rules), and **plan mode vs direct execution** is the choice between "tell me your plan before you touch anything" and "just do it."

**The key link to everything above:** Claude Code *is* an agentic loop — it's the **Level-3 Agent SDK** from Section 5 running autonomously against your repo. It sends to Claude, sees `tool_use` (Read, Edit, Bash, Grep…), runs the tool on your filesystem, feeds the result back, and repeats until `end_turn`. You don't write that loop. Instead you **steer** it with three controls: `CLAUDE.md` (standing context), custom slash commands (reusable prompts), and plan mode vs direct execution (how much autonomy you grant per task).

### 11a. `CLAUDE.md` — standing context for every session

`CLAUDE.md` is your project's instruction file. **Claude Code reads it at the start of every session** and it persists across sessions, so you stop re-explaining the same things. Think of it as a permanent prefix to the system prompt for this repo.

- **Generate it:** run `/init` in a new repo — it scans the project and drafts directory structure, common commands, and conventions.
- **Edit it:** `/memory`, or just type `# always use single quotes in JS` in chat to append a quick memory.
- **Scope/hierarchy:** project file at `./CLAUDE.md` (commit it, shared via git) and a personal one at `~/.claude/CLAUDE.md` (applies across all your projects).
- **Golden rule:** *shorter is better.* A 30-line CLAUDE.md that's actually read beats a 300-line one that gets ignored.

```markdown
# CLAUDE.md
## Commands
- Build: `npm run build`   - Test: `npm test`   - Lint: `npm run lint`
## Conventions
- TypeScript strict mode; no `any`.
- Functional React components only; co-locate tests as `*.test.tsx`.
- Conventional Commits for messages.
## Architecture
- `src/api` = backend calls, `src/ui` = components. Never import api into ui directly; go through hooks.
```

**Why it matters:** without it, Claude re-discovers "is there a lint command?" every session and may violate conventions. With it, generated and refactored code matches your project on the first try — fewer correction cycles.

### 11b. Custom slash commands — reusable prompts

A slash command is a **prompt template** you invoke as `/<name>`. It's a markdown file; the body is the prompt, optional YAML frontmatter sets metadata.

- **Location:** `.claude/commands/<name>.md` (project, shared via git) or `~/.claude/commands/<name>.md` (personal).
- **Arguments:** `$ARGUMENTS` captures everything; `$1`, `$2` are positional.
- **Frontmatter keys:** `description`, `allowed-tools`, `model`, `argument-hint`.
- **Handy prefixes inside the body:** `@path/to/file` includes a file's contents; `!` runs a shell command and embeds its output.

```markdown
---
description: Review staged changes for bugs and security issues
allowed-tools: Read, Grep, Glob, Bash(git diff:*)
argument-hint: [path]
---
## Context
- Staged diff: !`git diff --staged`
Review the staged changes$ARGUMENTS for: logic errors, security issues
(injection, XSS, auth bypass), and violations of our CLAUDE.md conventions.
Report findings as a punch list.
```

Invoke with `/review` or `/review src/auth`. **Current note:** as of recent versions, custom commands have been **merged into Skills** — the modern form is `.claude/skills/<name>/SKILL.md`, which can be invoked as `/<name>` *and* triggered autonomously by Claude. Legacy `.claude/commands/` files still work; if a skill and command share a name, the skill wins.

**Why it matters:** team-shared commands turn ad-hoc prompts into repeatable, version-controlled workflows (`/review`, `/test`, `/changelog`), so everyone gets consistent results.

### 11c. Plan mode vs. direct execution — how much autonomy per task

This is the most testable judgment call in the scenario.

| | **Plan mode** | **Direct execution** |
|---|---|---|
| What Claude does | Investigates the codebase and proposes a step-by-step plan, makes **no file changes**, waits for your approval | Edits/creates files **immediately** as it works |
| You review… | The plan, *before* anything changes | The resulting diff, *after* changes land |
| Best for | Large/multi-file refactors, ambiguous tasks, unfamiliar code, anything risky | Small, well-scoped, low-risk changes |
| Risk profile | Safer — catch wrong direction before edits exist | Faster — but you can undo only after the fact |

- **Toggle modes** by cycling with `Shift+Tab` (plan mode → auto-accept → normal), or start a task with `/plan <task>`.
- **`opusplan`**: Opus writes the plan, Sonnet executes it — cost-efficient for complex refactors (heavy reasoning only where it's needed).

**Decision rule for the scenario:** *"Refactor this one function for clarity"* → direct execution. *"Migrate our auth from callbacks to async/await across the codebase"* → plan mode first (review the approach, then let it execute), ideally `opusplan`.

### 11d. How the four controls work together (a workflow)

```
1. /init                          -> generate CLAUDE.md (one-time per repo)
2. Open a task in PLAN MODE       -> "migrate auth to async/await"
3. Claude proposes a plan         -> you review, adjust, approve  (no edits yet)
4. Claude executes the loop       -> Read/Edit/Bash tools, conventions from CLAUDE.md
5. /review                        -> custom command audits the diff
6. /test                          -> custom command runs the suite
```

CLAUDE.md keeps every step on-convention; slash commands standardize the recurring steps; plan mode gates the risky one; the agentic loop does the work.

### 11e. Scenario-2 self-check

1. When is `CLAUDE.md` read, and what's the golden rule? → At the **start of every session**; keep it **short**.
2. Where do custom slash commands live and how do you pass args? → `.claude/commands/<name>.md` (or the newer `.claude/skills/<name>/SKILL.md`); `$ARGUMENTS` / `$1`, `$2`.
3. Plan mode vs direct execution — pick by what? → **Risk and ambiguity.** High → plan mode (review before edits); low/scoped → direct execution.
4. What does `opusplan` do? → **Opus plans, Sonnet executes** — reasoning where it counts, cheaper execution.
5. How does this connect to Task 1.1? → Claude Code is a **Level-3 agentic loop**; these controls steer it, they don't replace the `tool_use`/`end_turn` mechanic underneath.

---

## 12. Scenario 3 — Multi-Agent Research System

> **Scenario 3 (as given):** *You are building a multi-agent research system using the Claude Agent SDK. A coordinator agent delegates to specialized subagents: one searches the web, one analyzes documents, one synthesizes findings, and one generates reports. The system researches topics and produces comprehensive, cited reports.*

**What this scenario is even about (plain English):** Think of how a research team works: a **manager** breaks a big question into pieces and hands each piece to a **specialist** — one person googles, one reads the source documents, one combines everyone's notes into a story, one writes the final report. This scenario builds exactly that, but with AI. The manager is the **coordinator agent**; the specialists are **subagents**. Why bother splitting it up instead of one AI doing everything? Because each AI has a limited "working memory" (its **context window** — like desk space). One AI trying to hold a web search, ten documents, and the final report all at once runs out of room and gets sloppy. Giving each subagent its own desk keeps everyone focused. "**Cited reports**" means every claim in the final report points back to where it came from — which, as you'll see, requires deliberately carrying the sources through each hand-off.

**The key link to everything above:** multi-agent is **agentic loops nested inside agentic loops.** The coordinator runs the exact same `tool_use → execute → end_turn` loop from Sections 1–5 — except some of its "tools" are *other agents*. When the coordinator delegates, that's a tool call; the subagent runs its **own** full loop (with its own tools and its own `stop_reason` cycle) and returns its final output back to the coordinator as a single result. So nothing new replaces the loop; it's the loop, recursively.

### 12a. The coordinator–subagent (orchestrator-worker) pattern

One **coordinator** agent doesn't do the bulk of the work — its job is **decompose, delegate, monitor, synthesize**:

1. Receive the high-level goal ("produce a cited report on X").
2. Break it into discrete subtasks.
3. Delegate each to a **specialized subagent** with a focused system prompt and only the tools it needs.
4. Collect each subagent's returned result.
5. Sequence dependencies and synthesize into the final deliverable.

For this scenario the roster is four specialists: a **web-search** agent, a **document-analysis** agent, a **synthesis** agent, and a **report-generation** agent.

### 12b. Why context isolation is the whole point

Each subagent runs in its **own isolated context window** (its own session thread with its own history). All of its intermediate work — tool calls, partial reasoning, raw search results — **stays inside its window**; the coordinator only receives the **final output**.

That's what makes the system scale: the web-search agent can burn 50k tokens reading pages, but the coordinator's context only grows by the agent's clean summary. Without isolation, one coordinator would drown in every subagent's raw working history and hit the single-agent ceiling (context overload + no specialization).

Practical limits to know: a coordinator can run **many** threads (the Managed Agents API caps concurrent threads — currently 25), but for most research tasks **3–5 parallel subagents** is the sweet spot, and you keep delegation nesting **shallow (2–3 levels)** to avoid runaway token use.

### 12c. The three delegation patterns (and when NOT to delegate)

| Pattern | What it means | In this scenario |
|---|---|---|
| **Parallelization** | Fan out independent subtasks at once, coordinator merges | Web search + document analysis run **simultaneously** |
| **Specialization** | Route to domain-focused agents instead of one do-everything agent | Each agent has its own prompt + minimal toolset |
| **Escalation** | Consult a more capable agent/model for a hard subset | Send a thorny synthesis step to an Opus-class agent |

**Don't use subagents when:** the task is a single quick lookup, or subtasks depend heavily on each other's *intermediate* steps, or one task's output is needed to frame the next. In those cases sequential work in the main context is simpler. Subagents pay off when subtasks are **substantive, independent, and would otherwise bloat context** — which describes research fan-out exactly.

### 12d. The research pipeline (fan-out, then sequential checkpoints)

This scenario is **not** fully parallel — there are real dependencies. Synthesis needs the search and analysis results first; the report needs the synthesis. So the coordinator runs a parallel batch, hits a checkpoint, then runs the dependent stages:

```
                 ┌─────────────────────────────────────────┐
   GOAL ───────▶ │              COORDINATOR                 │
 "cited report   │  decompose → delegate → synthesize       │
   on topic X"   └───┬───────────────┬──────────┬───────────┘
                     │ (parallel)    │          │ (sequential, after results)
            ┌────────▼─────┐ ┌───────▼───────┐  │
            │ web-search   │ │ doc-analysis  │  │
            │ agent        │ │ agent         │  │
            │ (own loop +  │ │ (own loop +   │  │
            │  web tools)  │ │  file tools)  │  │
            └────────┬─────┘ └───────┬───────┘  │
                     │ findings+srcs │ findings+srcs
                     └───────┬───────┘          │
                     ┌───────▼───────┐          │
                     │ synthesis     │◀─────────┘
                     │ agent         │
                     └───────┬───────┘
                     ┌───────▼───────┐
                     │ report-gen    │──▶ comprehensive, cited report
                     │ agent         │
                     └───────────────┘
```

Each box is a full agentic loop. The arrows are tool calls (delegations) and their returned results.

### 12e. How citations survive the pipeline

"Cited" is a design constraint, not magic. The rule: **every stage must pass sources forward, never just conclusions.**

- The **web-search** and **doc-analysis** agents return findings **paired with source metadata** (URL/title/doc id + the specific claim each supports), not bare prose.
- The **coordinator passes those sources downstream** as context to the synthesis agent (it controls what each subagent sees — subagents don't share memory automatically).
- The **synthesis** agent keeps each claim tied to its source id; the **report** agent renders them as citations.

If any stage summarizes away the sources, the final report can't cite — so the system prompts for the search/analysis agents must *require* structured `{claim, source}` output.

### 12f. Minimal shape in the Agent SDK

You define each specialist as an agent (own `system` prompt + own `tools`/MCP servers), then declare a coordinator with a **roster** of who it may delegate to. Tools and credentials are scoped per agent — e.g. only the web-search agent gets web tools; the coordinator itself has none.

```yaml
# coordinator (delegates; does no research itself)
name: Research Coordinator
model: claude-opus-4-8
system: >
  You coordinate a research team. Decompose the topic, delegate web search and
  document analysis in parallel, then synthesis, then report generation. Require
  every finding to carry its source. Do not do the research yourself.
tools:
  - type: agent_toolset_20260401      # the "delegate to a subagent" tool
multiagent:
  type: coordinator
  agents:
    - { type: agent, id: $WEB_SEARCH_AGENT_ID }
    - { type: agent, id: $DOC_ANALYSIS_AGENT_ID }
    - { type: agent, id: $SYNTHESIS_AGENT_ID }
    - { type: agent, id: $REPORT_AGENT_ID }
```

The SDK drives the whole thing and sets the multi-agent beta header for you. On the **primary thread** you see a condensed view — each subagent's start/end and any blocking events (like a tool-permission request) — and you can drill into a specific agent's **session thread** to inspect its full reasoning and tool calls. (This is a beta surface that ships fast — confirm exact fields/IDs against current docs.)

### 12g. What the coordinator's history looks like (and what crosses the boundary)

This is the multi-agent version of Section 4. There are now **two separate `messages` lists**: the coordinator's, and one per subagent. They never merge — only **final outputs** cross between them.

**The coordinator's own `messages`** (its isolated history). Delegation shows up as a `tool_use`; the subagent's finished result comes back as a `tool_result` — exactly like any other tool:

```python
coordinator_messages = [
  {"role": "user", "content": "Produce a cited report on <topic X>"},

  # Coordinator decides to fan out -> stop_reason: "tool_use" (two delegations at once)
  {"role": "assistant", "content": [
     {"type": "text", "text": "I'll run web search and document analysis in parallel."},
     {"type": "tool_use", "id": "toolu_DEL1", "name": "delegate",
      "input": {"agent": "web_search_agent", "task": "Find recent sources on X"}},
     {"type": "tool_use", "id": "toolu_DEL2", "name": "delegate",
      "input": {"agent": "doc_analysis_agent", "task": "Analyze the PDFs on X"}}
  ]},

  # What comes back = ONLY each subagent's FINAL output (not its internal steps)
  {"role": "user", "content": [
     {"type": "tool_result", "tool_use_id": "toolu_DEL1",
      "content": "3 findings: [{claim, url}, {claim, url}, {claim, url}]"},
     {"type": "tool_result", "tool_use_id": "toolu_DEL2",
      "content": "2 findings: [{claim, doc_id}, {claim, doc_id}]"}
  ]},

  # Checkpoint reached -> coordinator delegates synthesis, passing the sources forward
  {"role": "assistant", "content": [
     {"type": "tool_use", "id": "toolu_DEL3", "name": "delegate",
      "input": {"agent": "synthesis_agent", "task": "Synthesize these",
                "findings": "[...all 5 claims WITH their sources...]"}}
  ]},
  {"role": "user", "content": [
     {"type": "tool_result", "tool_use_id": "toolu_DEL3",
      "content": "Synthesized narrative + source map"}
  ]},

  # Then report generation
  {"role": "assistant", "content": [
     {"type": "tool_use", "id": "toolu_DEL4", "name": "delegate",
      "input": {"agent": "report_agent", "task": "Generate the cited report"}}
  ]},
  {"role": "user", "content": [
     {"type": "tool_result", "tool_use_id": "toolu_DEL4",
      "content": "<final cited report markdown>"}
  ]},

  # Coordinator wraps up -> stop_reason: "end_turn" -> its loop ends
  {"role": "assistant", "content": [
     {"type": "text", "text": "Here is the completed cited report: ..."}
  ]}
]
```

**The web-search subagent's OWN `messages`** (a totally separate, isolated list). It starts with *only* the task it was handed — it cannot see the coordinator's history — and runs its own full loop:

```python
web_search_agent_messages = [
  {"role": "user", "content": "Find recent sources on X"},   # only the delegated task

  {"role": "assistant", "content": [                          # stop_reason: "tool_use"
     {"type": "tool_use", "id": "t1", "name": "web_search", "input": {"q": "X latest"}}]},
  {"role": "user", "content": [
     {"type": "tool_result", "tool_use_id": "t1", "content": "10 raw search hits..."}]},

  {"role": "assistant", "content": [                          # stop_reason: "tool_use"
     {"type": "tool_use", "id": "t2", "name": "web_fetch", "input": {"url": "..."}}]},
  {"role": "user", "content": [
     {"type": "tool_result", "tool_use_id": "t2", "content": "full page text..."}]},

  {"role": "assistant", "content": [                          # stop_reason: "end_turn"
     {"type": "text", "text": "3 findings: [{claim, url}, {claim, url}, {claim, url}]"}]}
  # ^ THIS final message is the ONLY thing the coordinator receives (as toolu_DEL1's result)
]
```

**The boundary — read this carefully:**

| Subagent side | → crosses as → | Coordinator side |
|---|---|---|
| The delegated **task** string | maps from | the coordinator's `tool_use.input` |
| 10 raw search hits, fetched pages, intermediate reasoning | **never crosses** | (stays inside the subagent only) |
| The subagent's **final `end_turn` message** | becomes | the coordinator's `tool_result.content` |

So the coordinator's context grows by a few clean summaries, **not** by every page the search agent read. That single fact — *full history stays in the subagent, only the final output returns* — **is** the "context isolation" from 12b, made concrete. In the Managed Agents API each subagent's list is a separate **session thread**; on the coordinator's **primary thread** you see only the start/end of each delegation (plus blocking events like permission prompts), and you drill into a session thread when you want the subagent's full reasoning.

**Citations tie back here:** because only the final output crosses, the search/analysis agents must put `{claim, source}` *in that final message*. If sources live only in the raw hits (which never cross), they're lost — which is why 12e insists every stage pass sources forward.

### 12h. Scenario-3 self-check

1. How does this connect to Task 1.1? → Each agent runs the **same agentic loop**; delegation is a **tool call**, the subagent's final output is the **tool result**.
2. What is the coordinator's actual job? → **Decompose, delegate, sequence, synthesize** — not do the work itself.
3. Why isolated context per subagent? → Keeps the coordinator's context clean and enables **specialization + parallelism**; the parent sees only final outputs.
4. Is the pipeline fully parallel? → No. Search + analysis are parallel; synthesis and report are **sequential checkpoints** because they depend on prior results.
5. When should you NOT split into subagents? → Single lookups, or tightly **interdependent** steps where one output frames the next.
6. How do reports stay cited? → Every stage passes **`{claim, source}` forward**; the coordinator routes sources downstream; nothing summarizes them away.

---

## 13. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *You are building developer productivity tools using the Claude Agent SDK. The agent helps engineers explore unfamiliar codebases, understand legacy systems, generate boilerplate code, and automate repetitive tasks. It uses the built-in tools (Read, Write, Bash, Grep, Glob) and integrates with MCP servers.*

**What this scenario is even about (plain English):** When a developer joins a project or inherits old software, they face a giant pile of files nobody fully remembers — a "**codebase**" they're "**unfamiliar**" with, often a "**legacy system**" (old code still running the business that people are scared to touch). Just figuring out *where* things are and *how* it works eats days. This scenario builds an AI helper that does that grunt work: it can search the files, read them, run commands, and explain what it finds. "**Boilerplate**" is the repetitive starter code every new file needs (the same setup typed over and over) — tedious and perfect for automation. The **built-in tools** are the assistant's hands: **Read** (open a file), **Glob** (find files by name pattern), **Grep** (search inside files for text), **Write** (create/edit files), **Bash** (run commands like tests). **MCP servers** are how you plug in *your own company's* tools (your database, your Jira) so the assistant can reach beyond just files.

**The key link to everything above:** the Claude Agent SDK is *Claude Code's engine as a library* — the **same agent loop** from Sections 1–5, except the **executor is built in**. In Scenario 1 you hand-wrote `run_tool(block.name, block.input)`. Here, when Claude emits a `tool_use` for `Read`, the SDK reads the file; for `Bash`, the SDK runs the command. You don't implement a single tool executor. So the mechanic is unchanged — `tool_use → execute → tool_result → repeat until end_turn` — only *who runs the tool* changed (the SDK, not your code). Mental model: **Claude Code is the GUI; the Agent SDK is the API. Same engine.**

### 13a. The built-in toolbelt

These ship with the SDK; you "turn them on" by listing them in `allowed_tools` — no wiring required.

| Tool | What it does | Used in this scenario for |
|---|---|---|
| **Read** | Read a file's contents | Reading source, configs, legacy modules |
| **Glob** | Find files by name pattern (`**/*.py`) | Mapping an unfamiliar repo's shape |
| **Grep** | Regex-search file *contents* | Finding where a symbol/route/config is defined |
| **Write / Edit** | Create files / precise in-place edits | Generating boilerplate, scaffolding |
| **Bash** | Run terminal commands, scripts, git | Running tests, builds, automating repetitive tasks |

`Glob` + `Grep` + `Read` together are the **agentic-search backbone**: that trio alone lets the agent explore and explain any codebase while **changing nothing** — exactly what "explore unfamiliar codebases / understand legacy systems" needs. (Other built-ins exist too: `WebSearch`, `WebFetch`, `AskUserQuestion`, `Agent` for subagents, `NotebookEdit`.)

### 13b. A built-in tool flows through the loop identically to Scenario 1

```python
# Claude's response  ->  stop_reason: "tool_use"
{"role": "assistant", "content": [
   {"type": "text", "text": "Let me find where auth is handled."},
   {"type": "tool_use", "id": "t1", "name": "Grep",
    "input": {"pattern": "def authenticate", "glob": "**/*.py"}}
]}

# The SDK runs Grep itself (you wrote no executor) and feeds the result back:
{"role": "user", "content": [
   {"type": "tool_result", "tool_use_id": "t1",
    "content": "src/auth/login.py:42: def authenticate(user, pw):"}
]}
# loop continues -> Claude now Reads that file, etc., until stop_reason: "end_turn"
```

Same `tool_use`/`tool_result` pairing from Section 4 — the only difference from Scenario 1 is that the line where you'd call `run_tool` is now inside the SDK.

### 13c. Two flavors of MCP integration

Built-in tools cover the filesystem and web. **Your domain** (Jira, an internal deploy API, a database) lives behind MCP tools. There are two kinds:

- **In-process SDK MCP servers** — custom tools defined as plain functions in your own program (Python `@tool` decorator + `create_sdk_mcp_server`). They run **inside your process** — no subprocess, no network hop.
- **External MCP servers** — separate servers you connect to: **stdio** (`command`/`args`) or **HTTP/SSE** (`url`). This is how you plug into the broader ecosystem (GitHub, Playwright, internal services).

Both are configured via `mcp_servers={...}`. **Naming convention that trips people up:** an MCP tool is fully qualified as `mcp__<server>__<tool>`, and that *full* string is what goes in `allowed_tools` (e.g. `mcp__github__create_issue`).

```python
from claude_agent_sdk import tool, create_sdk_mcp_server

@tool("create_ticket", "Open a Jira ticket", {"title": str, "body": str})
async def create_ticket(args):
    ticket_id = jira.create(args["title"], args["body"])     # your real code
    return {"content": [{"type": "text", "text": f"Created {ticket_id}"}]}

tickets = create_sdk_mcp_server(name="jira", version="1.0", tools=[create_ticket])
# Claude calls it as the tool  mcp__jira__create_ticket
```

### 13d. Permissions & least privilege — the real architect decision

Because this agent can **write files and run shell commands**, the central design question is *what is it allowed to do without asking?* The SDK gives you layered control:

- **`allowed_tools`** — pre-approve specific tools (so they run without a prompt). **`disallowed_tools`** — hard-deny (a deny rule wins even under `bypassPermissions`).
- **Permission modes** (set per query or changed mid-session):

| Mode | Behavior |
|---|---|
| `default` | Unmatched tools route to your `canUseTool` callback |
| `plan` | Explore and plan, **no edits** |
| `acceptEdits` | Auto-approve file edits; other side-effects still gated |
| `dontAsk` | Approve only the pre-approved list; deny everything else outright |
| `bypassPermissions` | Approve everything — **trusted environments only** |
| `auto` (TS) | A model classifier approves each call |

- **`canUseTool` callback** — decide per call at runtime (interactive approval flows).
- **Hooks** (`PreToolUse`, `PostToolUse`, …) — deterministic code that can block a dangerous call (e.g. veto any `Bash` containing `rm -rf`) before it runs.

**Golden rule: start from least privilege.** A read-only agent with `["Read", "Glob", "Grep"]` *can analyze anything and damage nothing* — the safe default for exploration. Add write/bash only where the task requires it.

### 13e. The four dev tasks → four privilege profiles

| Task | Tools | Permission mode | Why |
|---|---|---|---|
| Explore unfamiliar codebase | Read, Glob, Grep | `plan` (read-only) | Investigate + propose, change nothing |
| Understand a legacy system | Read, Glob, Grep (+ a docs MCP) | `plan` | Same — analysis must be non-destructive |
| Generate boilerplate | + Write, Edit | `acceptEdits` | Let it scaffold files without a prompt per file |
| Automate repetitive tasks | + Bash (+ domain MCP) | `default` + `canUseTool`/hooks | Side-effects need guardrails, not blanket trust |

This is just **model-driven decisions (Section 6) plus a safety envelope**: you let Claude choose *which* tool, while permissions bound *what it may do unattended.* And recall the anti-pattern from Section 8 — `maxTurns` is the SDK's only built-in bound, so treat it as a **backstop**, never as your real "done" signal (`end_turn` still is).

### 13f. A minimal read-only codebase scout (Python)

```python
from claude_agent_sdk import query, ClaudeAgentOptions

async for message in query(
    prompt="Explore this repo with Glob/Grep/Read and summarize its architecture.",
    options=ClaudeAgentOptions(
        cwd="/workspace/project",
        allowed_tools=["Read", "Glob", "Grep"],   # least privilege
        disallowed_tools=["Write", "Edit", "Bash"],
        permission_mode="plan",                    # cannot modify anything
        model="claude-opus-4-8",
        max_turns=30,                              # safety backstop, not the stop signal
    ),
):
    if getattr(message, "type", None) == "result":
        print(message.result)
```

The SDK runs the whole loop, executes each Read/Glob/Grep itself, and streams back messages — you only read the final result.

### 13g. Scenario-4 self-check

1. How does the SDK relate to Scenario 1's loop? → Same loop; the **executor is built in** (the SDK runs Read/Bash/etc., so you don't write `run_tool`).
2. What's the read-only "search backbone"? → **Glob + Grep + Read** — explore and explain while changing nothing.
3. Two kinds of MCP servers? → **In-process SDK servers** (functions in your process) and **external servers** (stdio / HTTP-SSE); tools are named `mcp__<server>__<tool>`.
4. What's the central design decision for a write/bash-capable agent? → **Least privilege** — `allowed_tools` + permission mode + `canUseTool`/hooks; start read-only, add power only where needed.
5. Which mode lets it explore but not edit? → **`plan`** (read-only). Which auto-approves edits? → **`acceptEdits`**.
6. Is `maxTurns` the stopping mechanism? → No — it's a **backstop**; `end_turn` is the real stop (Section 8 anti-pattern).

---

## 14. Scenario 5 — Claude Code for Continuous Integration

> **Scenario 5 (as given):** *You are integrating Claude Code into your Continuous Integration/Continuous Deployment (CI/CD) pipeline. The system runs automated code reviews, generates test cases, and provides feedback on pull requests. You need to design prompts that provide actionable feedback and minimize false positives.*

**What this scenario is even about (plain English) — start here if you're not a DevOps person.**

Let's build up the words first, because the rest of this section assumes them.

- **A codebase shared by a team.** Many developers work on the same software. Everyone keeps their own copy and makes changes, then needs to combine those changes back into the one official version. The tool that tracks all this is **Git**, and the website most teams use to host it is **GitHub**.
- **A pull request (PR).** When a developer finishes a change, they don't just shove it into the official code. They open a **pull request**: "here's my proposed change — please review it before we merge it in." Other people look at it, comment, and approve. A PR is basically *a change waiting for review*.
- **Why this is risky.** Every change can introduce a bug, a security hole, or break something else. With dozens of PRs a week, humans can't carefully check everything, and tired reviewers miss things.
- **CI/CD = automation that runs on every change.** So teams set up a robot that automatically *does stuff* every time a PR is opened or updated. **CI (Continuous Integration)** = automatically check and test each change as it comes in. **CD (Continuous Deployment/Delivery)** = automatically release the code once it passes. The sequence of automated steps is called a **pipeline**. Think of it as a conveyor belt: a change comes in, and it automatically gets checked, tested, and (if all's well) shipped — no human pressing buttons.
- **What's a "GitHub Action" / "runner"?** GitHub lets you define those automated steps in a small config file. When a PR arrives, GitHub spins up a temporary computer (a **runner**) and executes your steps on it. A **GitHub Action** is a pre-packaged step you can drop in — and Anthropic ships one that runs Claude Code.
- **Where Claude fits in this scenario.** You're adding Claude as *one automated step on that conveyor belt*: every time someone opens a PR, Claude automatically reads the change, **reviews the code**, **suggests test cases**, and **leaves comments on the PR** — like a tireless reviewer that looks at every single change, instantly, at 3am.
- **What "false positives" means here, and why the scenario obsesses over it.** A **false positive** is when the reviewer flags something that *isn't actually a problem* ("you should rename this variable", "this might be slow" — when it's fine). If your automated reviewer cries wolf constantly, developers start ignoring *all* its comments, and it becomes useless. So the goal is **actionable feedback** (clear, specific, worth acting on) with **few false positives** (don't nag about non-issues). Most of this section is about how to get that.
- **"Headless" / `claude -p`.** Normally you *chat* with Claude Code by typing in a terminal. But on the conveyor belt there's no human to chat — it needs to run *unattended*: receive a task, do it, output text, exit. That non-interactive mode is called **headless mode**, and you turn it on with the `-p` flag. This is the single most important practical fact in the whole scenario.

With those words in hand, the rest is straightforward.

**The key link to everything above:** CI is **headless Claude Code** — the same Level-3 agentic loop (Section 5, Scenario 4) — triggered by a *pipeline event* (a PR open/update) instead of a human at a terminal, with its output forced into **JSON** so a script can parse it and post a comment. Nothing about the loop changes; you just run it unattended and read the structured result. The two genuinely new dimensions are **(1) running non-interactively** and **(2) prompt design for signal quality** (the scenario's explicit goal: actionable feedback, few false positives).

### 14a. Headless ("print") mode — the CI entry point

`claude -p "..."` (short for `--print`) runs Claude Code **non-interactively**: it reads the prompt (as an argument or piped on stdin), runs the loop to completion, prints the result to stdout, and exits with a status code your pipeline can branch on.

> **Common failure (and a likely exam question):** a CI job runs plain `claude "review this PR"` and **hangs forever** waiting for interactive input. The fix is one flag: **`-p`**. (`CLAUDE_HEADLESS`, stdin redirects, and `--batch` are not real fixes.)

Key flags that make a run controllable and parseable:

| Flag | Purpose |
|---|---|
| `-p` / `--print` | Non-interactive: one prompt in, one result out, exit |
| `--output-format json` (or `stream-json`) | Machine-readable result (parse with `jq -r '.result'`). Default text is for humans |
| `--allowedTools "Read,Grep,Glob"` | Restrict tools — read-only for review |
| `--max-turns N` | Cap the loop (safety backstop) |
| `--model` | opus / sonnet / haiku |

You can also pipe data in: `git diff origin/main...HEAD | claude -p "Review this diff. Be concise." --output-format json`.

### 14b. Two ways to run it

- **Official GitHub Action** — `anthropics/claude-code-action@v1`. It runs the **full Claude Code runtime inside your GitHub runner** (not a thin API call), reads the diff/git history, and posts comments. You pass `prompt`, CLI flags via `claude_args`, and a credential via `anthropic_api_key` (or `use_bedrock`/`use_vertex`). Triggers on `pull_request: [opened, synchronize]` or on `@claude` mentions. `/install-github-app` sets it up.
- **Raw `claude -p`** — for GitLab CI, Jenkins, CircleCI, or any runner: install the CLI, inject the key as a secret env var, invoke `claude -p ... --output-format json`, parse the JSON. The shape is identical; only the scheduler differs.

### 14c. Every invocation is a fresh, stateless session

Headless mode **does not persist** between runs. So if one step generates code and a later step reviews it, **the review step cannot see the generation step's reasoning** — only whatever is on disk / in the diff. Implication: each `claude -p` call must be given *all* the context it needs in its prompt and the repo state. Don't assume continuity across steps.

### 14d. CLAUDE.md is your CI config (and your false-positive control)

Claude Code reads `CLAUDE.md` even in CI. Put your **review criteria, code style, and project rules** there once, and every pipeline run enforces the *same* standard. This is also the single biggest lever on **false positives**: if Claude judges the diff against your *actual* documented rules (named exports only, Zod on every endpoint, max function length 30) instead of guessing your conventions, it stops flagging things that aren't real violations.

```markdown
# CLAUDE.md  (excerpt used by CI)
## Review Criteria
- Every API endpoint validates input with Zod.
- No `any` in TypeScript. Named exports only.
## CI Rules
- Never modify .github/workflows/. Always open a PR; never push to main.
```

### 14e. Prompt design — actionable feedback, minimal false positives (the core)

This is the heart of the scenario. A vague prompt ("review this PR") yields a noisy wall of nitpicks. The fixes are concrete:

1. **Scope to the diff, not the repo.** Feed only changed files / the diff. Less surface = higher signal, lower cost.
2. **Force a structured output format** with **severity + exact location + a suggested fix**, grouped by category. Structure makes it actionable *and* lets you filter (e.g. post only high/medium).
3. **Tell it to stay silent when there's nothing to say** — the strongest single anti-false-positive instruction: *"If no issues in a category, write 'No issues found.'"*
4. **Demand a rationale and confidence**, and instruct it to **only report high-confidence issues** — low-confidence hunches get dropped or downgraded, not posted.
5. **Ask for fixes, not complaints** — a suggested diff/line is actionable; "this could be better" is not.
6. **For test generation, pass the existing tests in context** so it doesn't re-suggest tests you already have.

A reusable review prompt (works as a `claude -p` string or a `ci-review` skill):

```text
Review ONLY the changed files in this diff: $FILES
Report issues you are HIGH-CONFIDENCE about. Judge against CLAUDE.md conventions.
Output exactly this structure (if a category is clean, write "No issues found."):

## Security
- [HIGH|MED|LOW] <finding> (file:line) — <why> — Suggested fix: <concrete change>
## Correctness
- ...
## Performance
- ...

Do not report style preferences already satisfied by the linter. Be concise.
```

**Why each rule cuts false positives:** scoping removes unrelated-code noise; "high-confidence only" + "no issues → say so" suppresses speculative nitpicks; judging against CLAUDE.md removes convention-guessing; "not style the linter covers" removes duplication with existing tooling. The result is a review a developer will actually trust — the goal being *high-signal*, not *maximal* output.

### 14f. Least privilege & safety in CI

The Scenario-4 least-privilege idea, applied to an unattended pipeline:

- **Read-only for review:** `--allowedTools "Read,Grep,Glob"` — a reviewer should never edit. Add `Write`/`Edit`/`Bash` only for fix/test-generation jobs.
- **Secrets, never hardcode:** `ANTHROPIC_API_KEY` from repository/org secrets; never log it.
- **Minimal GitHub token permissions:** e.g. `contents: read`, `pull-requests: write` — drop `contents: write` and `issues:` if the job only comments.
- **Fork safety:** fork PRs don't get your secrets by default (so untrusted contributors can't burn your credits or exfiltrate keys); `pull_request_target` removes that protection — use it with extreme care.
- **Backstops:** `timeout-minutes` on the job, `--max-turns` on the run (the loop's real stop is still `end_turn`/exit — Section 8), token caps for cost, and a concurrency lock so a slow scheduled run doesn't race the next tick.
- **Human-in-the-loop:** always review suggestions before merge. `--dangerously-skip-permissions` belongs only in a sandboxed container with no prod creds and no network egress.

### 14g. A minimal PR-review workflow

```yaml
name: Claude PR Review
on:
  pull_request:
    types: [opened, synchronize]
permissions:
  contents: read          # least privilege
  pull-requests: write    # only what's needed to comment
jobs:
  review:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          claude_args: '--allowedTools "Read,Grep,Glob" --max-turns 5 --model sonnet'
          prompt: >
            Review ONLY this PR's changed files for high-confidence security and
            correctness issues, judged against CLAUDE.md. Use severity labels and
            file:line, suggest a fix per finding, and write "No issues found." for
            any clean category. Ignore style the linter already enforces.
```

### 14h. Scenario-5 self-check

1. What makes a CI run non-interactive (and why does the job hang without it)? → **`claude -p`** (`--print`); without it Claude waits for terminal input forever.
2. Why `--output-format json`? → A **machine-readable** result the pipeline can parse (`jq '.result'`) and post; text is for humans.
3. Do steps share context? → No — **each `claude -p` is a fresh, stateless session**; give each its full context.
4. Biggest lever on false positives? → **Codify standards in CLAUDE.md** + prompt for **high-confidence only** + **"no issues → say so"** + scope to the diff.
5. What makes feedback *actionable*? → **Severity + file:line + a suggested fix**, grouped by category.
6. Least-privilege defaults for a reviewer? → Read-only `--allowedTools "Read,Grep,Glob"`, minimal token perms, secrets not hardcoded, fork safety, timeouts.
7. Is `--max-turns` the stop signal? → No — a **backstop**; the loop still ends on `end_turn`/exit (Section 8).

---

## 15. Scenario 6 — Structured Data Extraction

> **Scenario 6 (as given):** *You are building a structured data extraction system using Claude. The system extracts information from unstructured documents, validates the output using JSON schemas, and maintains high accuracy. It must handle edge cases gracefully and integrate with downstream systems.*

**What this scenario is even about (plain English):** An "**unstructured document**" is text written for *humans* — a PDF invoice, a resume, an email. A person can read it and pick out "the total is $89.99, due June 30," but a *computer* can't easily use free-flowing text. "**Structured data**" is the same information rearranged into tidy labeled fields a computer can use directly, like `{ "total": 89.99, "due_date": "2026-06-30" }`. This scenario builds a system that reads the messy human document and spits out the tidy computer version — "**extraction**." **JSON** is just the standard text format for that tidy data (labels and values). A "**JSON schema**" is a blueprint that says exactly which fields must be present and what type each is (total must be a number, due_date must be a date-or-empty); "**validate against the schema**" means automatically checking the output matches that blueprint. "**Downstream systems**" are whatever uses the data next — a database, an accounting app — which only work if the data is reliably the right shape. "**Handle edge cases gracefully**" means: don't crash or make things up when a field is missing, the document is weird, or the model is unsure.

**The key link to everything above:** the cleanest way to extract structured data reuses the **exact tool-use machinery** from Sections 1–5. You define an `input_schema` (the same JSON schema from "define tools"), Claude emits a `tool_use` block with `stop_reason: "tool_use"`, and **`tool_use.input` *is* your extracted record** — you just read it instead of executing a tool. And the "edge cases" the scenario asks you to handle are mostly `stop_reason` values from Section 2 (`refusal`, `max_tokens`). So extraction is a one-iteration agentic loop.

### 15a. Two native approaches (both "Structured Outputs")

Anthropic's Structured Outputs feature gives format guarantees two ways. (Legacy "JSON mode" is deprecated; don't use it.)

| | **JSON outputs** | **Strict tool use** |
|---|---|---|
| How | `output_format: {type:"json_schema", schema:{...}}` | A tool with `strict: true` + its `input_schema` |
| What's constrained | The **final text** (`response.content[0].text`) | The **tool call** (`tool_use.input`) |
| Best for | **Data extraction** — invoices, emails, resumes → JSON | Agentic workflows needing **valid tool arguments** |
| You read | `response.content[0].text` (valid JSON string) | `tool_use.input` (already a dict/object) |

For *this* scenario (documents → JSON), **JSON outputs** is the natural fit, but the strict-tool path is interchangeable and connects directly to the loop. They can also be **combined** in one request (constrain a tool's args *and* the final JSON).

### 15b. How the guarantee actually works

Structured Outputs uses **constrained decoding**: your JSON schema is compiled into a grammar, and token generation is restricted during inference so the model **literally cannot emit a token that would violate the schema**. This is the difference between *asking* for valid JSON and *enforcing* it — no more fragile regex or retry loops. The compiled grammar is cached (~24h), so **stabilize your schema early** to benefit from the cache. It's a public beta, enabled with a beta header (e.g. `anthropic-beta: structured-outputs-2025-11-13`).

> Parameter naming is in flux across versions (`output_format` vs `output_config.format`) and the beta header may change — confirm both against current docs before shipping.

### 15c. The caveat that defines "high accuracy"

**Structured Outputs guarantees the *shape*, not the *truth*.** The model can still hallucinate — you can get a **perfectly-formatted wrong answer**. So "maintains high accuracy" is *not* solved by the schema; it's solved by everything around it: tight extraction instructions ("extract only what is present; use null if absent"), low temperature (0–0.2 for factual extraction), confidence fields, and **downstream validation/business rules**. Treat schema-validity and correctness as two separate problems.

### 15d. Designing the schema

Standard JSON Schema, with the knobs that matter for extraction:

- `required` vs optional — mark only what must always exist.
- `additionalProperties: false` — forbid stray fields (no hallucinated keys).
- `enum` — constrain to a fixed set (`urgency: ["low","medium","high"]`).
- `format` / typed fields — `"email"`, `integer`, `boolean`, `number`.
- **Nullable** types for fields that may be missing: `{"type": ["string", "null"]}`.
- Nested objects/arrays for line items, education history, etc.

Generate the schema from **Pydantic** (`model_json_schema()`) or **Zod** (`z.toJSONSchema()`) rather than hand-writing it — you get one source of truth and type-safe parsing on the way out.

```python
# JSON outputs — extract an invoice (illustrative; confirm param/header vs current docs)
schema = {
  "type": "object",
  "properties": {
    "vendor":   {"type": "string"},
    "total":    {"type": "number"},
    "currency": {"type": "string", "enum": ["USD", "EUR", "GBP"]},
    "due_date": {"type": ["string", "null"]},        # nullable: may be absent
    "line_items": {"type": "array", "items": {
        "type": "object",
        "properties": {"desc": {"type": "string"}, "amount": {"type": "number"}},
        "required": ["desc", "amount"], "additionalProperties": False}},
  },
  "required": ["vendor", "total", "currency"],
  "additionalProperties": False,
}

resp = client.messages.create(
    model="claude-opus-4-8", max_tokens=1024, temperature=0,
    messages=[{"role": "user",
               "content": f"Extract invoice fields. Use null if a field is absent.\n\n{doc}"}],
    output_format={"type": "json_schema", "schema": schema},
)
record = json.loads(resp.content[0].text)   # guaranteed valid JSON, no retry logic
```

### 15e. Handling edge cases gracefully

The scenario's "edge cases" are concrete and most map to `stop_reason`:

- **Field genuinely absent** → make it nullable and instruct "use null if not present." Don't force the model to invent a value to satisfy `required`.
- **Ambiguity / "not found"** → add a `confidence` (0–1) or `found` boolean, or an `enum` value like `"unknown"`, so downstream can route low-confidence records to human review.
- **Truncation** → if the JSON is cut off, `stop_reason` is **`max_tokens`** — raise `max_tokens` and re-check. (Watch this on long line-item tables.)
- **Refusal** → `stop_reason` is **`refusal`** — handle as a clean branch, don't crash the parser.
- **Document too large** → chunk it, extract per chunk, merge — don't blow the context window.
- **Wrong-but-valid values** → schema won't catch these; catch them with **post-extraction validation** (e.g. totals must equal the sum of line items).

### 15f. Downstream integration

Because the output is guaranteed-valid JSON, integration is trivial and robust: `json.loads(...)` → a Pydantic/typed object → hand to the next system. No defensive parsing, no "did it add a markdown fence?" checks. The point, in Anthropic's framing, is to treat Claude as a **deterministic software component**: text in, schema-shaped data out. Keep a validation layer in front of downstream writes for the accuracy concerns from 15c.

### 15g. The strict-tool-use path = a one-iteration agentic loop

If you prefer the tool path (or need it on a model without JSON outputs), it's literally Section 1's loop run once — but you **read `tool_use.input` instead of executing**:

```python
tool = {"name": "save_invoice", "strict": True, "input_schema": schema}

resp = client.messages.create(
    model="claude-opus-4-8", max_tokens=1024,
    tools=[tool],
    tool_choice={"type": "tool", "name": "save_invoice"},   # force the tool
    messages=[{"role": "user", "content": f"Extract invoice fields:\n\n{doc}"}],
)
# stop_reason == "tool_use"; the structured record is the tool's input:
record = next(b.input for b in resp.content if b.type == "tool_use")
```

Same `tool_use`/`input_schema`/`stop_reason` mechanics as the customer-support agent — only here the "action" is just *handing you the data*. (In the **Agent SDK**, pass `output_format`/`outputFormat` to `query()` and read the result message's `structured_output` field — handy when the agent must Grep/Read first, then return structured data.)

### 15h. Scenario-6 self-check

1. Two native ways to get structured output? → **JSON outputs** (`output_format`, constrains final text) and **strict tool use** (`strict: true`, constrains `tool_use.input`).
2. What does the schema guarantee — and not? → **Format/shape yes; accuracy no.** You can get well-formed wrong answers (15c).
3. How is the guarantee enforced? → **Constrained decoding** — schema compiled to a grammar; invalid tokens can't be generated.
4. How do you keep accuracy high then? → "extract only what's present / null if absent," **low temperature**, confidence fields, and **downstream validation**.
5. Which `stop_reason` values are the edge cases? → **`max_tokens`** (truncated JSON) and **`refusal`** — handle both gracefully.
6. How does this connect to Task 1.1? → Strict tool use is the **same tool-use loop**; `tool_use.input` is the extracted record — a one-iteration agentic loop.
7. Field that might be missing — how to model it? → **Nullable** (`["string","null"]`) + instruct null-if-absent, not a forced `required` value.

---

### Sources verified against current Anthropic docs (June 2026)
- *How tool use works* / *Tool use overview* — the agentic loop, client vs server tools, `tool_choice: auto`.
- *Handling stop reasons* — `tool_use`, `end_turn`, `pause_turn`, `refusal`, `max_tokens`, `stop_sequence`.
- *Define tools / Tool Runner (SDK)* — `input_schema`, paired `tool_use`/`tool_result`, SDK auto-loop.
- *Claude Code docs* (slash commands, Skills, CLAUDE.md/memory, plan mode, `opusplan`) — verified June 2026; confirm specifics with `/help` and official docs.
- *Multiagent sessions / Managed Agents multi-agent* — coordinator roster, per-agent context isolation (session threads), concurrency cap, primary-vs-thread event streams — beta surface, re-check fields/IDs against current docs.
- *Claude Agent SDK* (built-in tools Read/Write/Edit/Bash/Glob/Grep/WebSearch/WebFetch, in-process vs external MCP servers, `mcp__server__tool` naming, permission modes, `canUseTool`, hooks, `allowed_tools`/`disallowed_tools`, `maxTurns`) — verified June 2026; SDK ships fast (note the `claude-code-sdk` → `claude-agent-sdk` rename), so re-verify imports/options against current versions.
- *Claude Code GitHub Actions / headless mode* (`claude -p`/`--print`, `--output-format json`, `--allowedTools`, `--max-turns`, `anthropics/claude-code-action@v1`, `prompt`/`claude_args`, least-privilege token permissions, fork safety, stateless per-invocation sessions) — verified June 2026; the Action and CLI ship fast, so re-check inputs/flags against current docs.
- *Structured outputs* (JSON outputs via `output_format`/`output_config.format`, strict tool use via `strict: true` + `input_schema`, constrained decoding/grammar, `tool_choice` forcing, schema caching, format-not-accuracy guarantee, citations/prefill incompatibilities, Agent SDK `structured_output`) — verified June 2026; public beta with header `anthropic-beta: structured-outputs-2025-11-13`, parameter names in flux, so confirm exact param/header against current docs.
