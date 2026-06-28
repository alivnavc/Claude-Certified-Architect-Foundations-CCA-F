# Task 2.3 — Distributing Tools Across Agents & Configuring Tool Choice
### TRUE zero-to-mastery guide — every term defined, every concept shown in code

This version assumes you know **nothing** about tools or the API. We build from the absolute ground up, and every idea is shown as real code you could run. Read top to bottom.

---

# PART A — THE GROUND FLOOR (what is even happening)

## A1. What is a "tool"?

A large language model (LLM) like Claude can only do one thing on its own: produce text. It cannot look up an order, search the web, or read a database. A **tool** is a function *you* write (in your own code) that Claude is *allowed to ask you to run*.

The flow is a conversation:
1. You tell Claude "here are the tools you can use" (you describe them).
2. Claude reads a user's question and says "please run `lookup_order` with order_id 48217."
3. **Your** code actually runs `lookup_order(48217)`, gets the answer from your database, and hands the result back to Claude.
4. Claude uses that result to write its final reply.

Claude never runs the tool itself. It only *requests* the call; you execute it and return the result. That back-and-forth is called the **agentic loop**.

## A2. What is the "API" and a "request"?

You talk to Claude by sending an **HTTP request** to Anthropic's **Messages API** — think of it as sending a structured form to a web address and getting a structured form back. The form is written in **JSON** (JavaScript Object Notation): just text made of `key: value` pairs inside `{ }`.

Here is the simplest possible request (no tools yet):

```python
import anthropic
client = anthropic.Anthropic()   # reads your API key from the environment

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "What is the capital of France?"}
    ],
)
print(response.content[0].text)   # -> "The capital of France is Paris."
```

- `model` — which Claude to use.
- `max_tokens` — the longest reply you'll allow.
- `messages` — the conversation so far, a list of `{"role": ..., "content": ...}`. Roles are `"user"` (the human) and `"assistant"` (Claude).

That's the whole foundation. Tools and `tool_choice` are just **extra fields** you add to this same request.

## A3. What does a tool look like in the request? (the schema)

To give Claude tools, you add a `tools` field. Each tool is a JSON description with three parts:

```python
tools = [
    {
        "name": "lookup_order",
        "description": "Look up an order by its order ID. Returns status, items, and charges.",
        "input_schema": {                     # the SHAPE of the arguments
            "type": "object",
            "properties": {
                "order_id": {
                    "type": "string",
                    "description": "The order ID, e.g. '48217'"
                }
            },
            "required": ["order_id"]           # order_id MUST be provided
        }
    }
]
```

- **`name`** — what the tool is called.
- **`description`** — plain-English explanation (this is what Claude reads to decide *whether* to use it — that's Task 2.1).
- **`input_schema`** — a **JSON Schema**, which is just a rulebook describing what arguments the tool takes: here, one string called `order_id`, and it's required. "Schema" simply means "the expected shape of the data."

## A4. What does Claude send back when it wants a tool?

When Claude decides to use a tool, the response's `stop_reason` is `"tool_use"`, and the content contains a **tool_use block** — Claude's *request* for you to run the tool:

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "What's the status of order 48217?"}],
)

print(response.stop_reason)   # -> "tool_use"
print(response.content)
# [ ToolUseBlock(
#     type="tool_use",
#     id="toolu_01A2b3...",            # a unique ID for THIS call
#     name="lookup_order",
#     input={"order_id": "48217"}      # the arguments Claude chose
#   ) ]
```

You then run *your* `lookup_order("48217")`, and send the result back as a `tool_result` in the next message (we'll show the full loop in C3). **Hold onto this picture — `tool_choice` is entirely about controlling this moment: will Claude produce a tool_use block, and which one?**

---

# PART B — `tool_choice`: CONTROLLING WHETHER & WHICH TOOL FIRES

`tool_choice` is **one extra field** in the same `messages.create(...)` call. It tells Claude how it's *allowed* to decide about tools on this turn. There are exactly four settings.

## B1. `auto` — Claude decides (the default)

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "auto"},     # <-- Claude may use a tool OR just talk
    messages=[{"role": "user", "content": "What's the status of order 48217?"}],
)
```

With `auto`, Claude looks at the message and **decides for itself** whether to call a tool or just answer in text. This is the default whenever you provide tools.

- "What's the status of order 48217?" → Claude calls `lookup_order`.
- "Thanks, bye!" → Claude just replies in text, no tool.

**The risk:** with `auto`, Claude *might* answer in prose when you actually needed a tool call. If your program is counting on getting structured tool output, `auto` doesn't guarantee it.

## B2. `any` — Claude MUST call some tool (its choice which)

```python
tool_choice={"type": "any"}     # <-- MUST call a tool; Claude picks which one
```

`any` forces Claude to call **one of the tools** — it can't reply with plain conversational text. It still chooses *which* tool. Use this when an action is mandatory and you can't risk a chatty non-answer.

Example: a data-extraction service where every input must produce a structured record. You never want "Sure, here's the info!" as prose — you want a tool call every time. `any` guarantees that.

## B3. `tool` — Claude MUST call ONE specific tool you name

```python
tool_choice={"type": "tool", "name": "lookup_order"}   # <-- MUST call lookup_order
```

This forces a **specific** tool. Claude has no choice — it will produce a `lookup_order` call. Use it when you know exactly what must happen first (e.g., "always verify identity before anything else").

## B4. `none` — Claude may NOT call any tool

```python
tool_choice={"type": "none"}     # <-- tools are visible but forbidden this turn
```

`none` lets Claude *see* the tools but forbids calling them — it must answer in text. Useful when you want a purely conversational turn even though tools exist. (This is the default when you provide **no** tools at all.)

## B5. Summary table

| `tool_choice` | Claude's options this turn | Use when |
|---|---|---|
| `{"type": "auto"}` | Call a tool **or** answer in text (its call) | Normal turns; default |
| `{"type": "any"}` | **Must** call a tool; picks which | An action is mandatory; never want prose |
| `{"type": "tool", "name": "X"}` | **Must** call tool **X** | You know exactly what must run |
| `{"type": "none"}` | **No** tool; text only | Force a conversational turn |

## B6. TWO BEHAVIORS THAT TRIP PEOPLE UP (advanced but important)

**(1) `any` and `tool` suppress Claude's preamble text.**
When you force a tool with `any` or `tool`, the API *prefills* Claude's turn so it jumps straight to the tool call — meaning **Claude will NOT write any explanation first**, even if your prompt says "explain your reasoning." If you need Claude to think out loud *and* call a tool, you must use `auto` and ask for the reasoning in the prompt.

```python
# This will NOT produce reasoning text — just the bare tool call:
tool_choice={"type": "any"}
# This CAN produce reasoning + a tool call:
tool_choice={"type": "auto"}   # + prompt: "Explain briefly, then look up the order."
```

**(2) Forcing is PER-TURN. You cannot force "A then B" in one request.**
`tool_choice` controls a single turn. To force tool A first and *then* run tool B, you force A this turn, read its result, and continue in the **next** turn (see C4). One forced tool per turn — that's the whole rule.

**(3) Limit to one tool call per turn with `disable_parallel_tool_use`.**
By default Claude can request several tools at once. To force **at most one** call per turn, add:

```python
tool_choice={"type": "any", "disable_parallel_tool_use": True}   # exactly ONE tool call
```

**(4) `any`/`tool` don't work with extended thinking.** If you've turned on extended thinking, only `auto` and `none` are allowed.

---

# PART C — DISTRIBUTING TOOLS ACROSS AGENTS (the other half of this task)

`tool_choice` controls choice *among the tools an agent has*. The first question is: **which tools should each agent even have?** That's tool distribution.

## C1. The core problem: too many tools = worse choices

Imagine one agent with 18 tools. Every time it must pick, it weighs 18 options — and the more similar-looking options there are, the more often it picks wrong. **Selection reliability drops as the tool count climbs.** An agent scoped to the 4–5 tools it actually needs picks far more reliably.

```python
# BAD: one mega-agent holding everything
mega_tools = [search_web, read_file, write_file, run_sql, send_email,
              create_ticket, refund, lookup_order, get_customer, ...]  # 18 tools

# GOOD: scope each agent to its job
support_tools  = [get_customer, lookup_order, process_refund, escalate_to_human]  # 4
research_tools = [search_web, read_document, extract_findings]                    # 3
```

## C2. Out-of-specialty tools get MISUSED

If you give an agent a tool outside its role, it tends to use it when it shouldn't. The classic example: a **synthesis agent** (whose job is to combine findings into an answer) is given a **web-search** tool. It starts running searches mid-synthesis — going off-task, redoing the search agent's work, and producing worse output.

**Rule:** don't grant capabilities "just in case." Capability you grant is capability that gets (mis)used.

```python
# BAD: synthesis agent can search the web -> it wanders off-task
synthesis_tools = [combine_findings, search_web]    # <-- search_web does NOT belong here

# GOOD: synthesis agent only synthesizes
synthesis_tools = [combine_findings]
```

## C3. The full agentic loop (so "distribution" is concrete)

Here's a complete, runnable support-agent loop showing the tools actually being used. This is the picture every distribution decision is about:

```python
import anthropic
client = anthropic.Anthropic()

support_tools = [
    {"name": "lookup_order",
     "description": "Look up an order by ID. Returns status, items, charges.",
     "input_schema": {"type": "object",
        "properties": {"order_id": {"type": "string"}}, "required": ["order_id"]}},
]

def run_lookup_order(order_id):                  # YOUR real implementation
    return {"order_id": order_id, "status": "shipped", "total": 129.99}

messages = [{"role": "user", "content": "What's the status of order 48217?"}]

# Turn 1: Claude asks to use the tool
resp = client.messages.create(
    model="claude-sonnet-4-6", max_tokens=1024,
    tools=support_tools, tool_choice={"type": "auto"}, messages=messages,
)

if resp.stop_reason == "tool_use":
    tool_call = next(b for b in resp.content if b.type == "tool_use")
    result = run_lookup_order(**tool_call.input)   # YOU run the tool

    # Append Claude's request AND your result to the conversation
    messages.append({"role": "assistant", "content": resp.content})
    messages.append({"role": "user", "content": [
        {"type": "tool_result",
         "tool_use_id": tool_call.id,               # must match the call's id
         "content": str(result)}
    ]})

    # Turn 2: Claude reads the result and writes the final answer
    final = client.messages.create(
        model="claude-sonnet-4-6", max_tokens=1024,
        tools=support_tools, tool_choice={"type": "auto"}, messages=messages,
    )
    print(final.content[0].text)   # -> "Order 48217 has shipped; total $129.99."
```

Notice `tool_result.tool_use_id` must equal the `id` from the tool_use block — that's how Claude knows which call this result answers.

## C4. ADVANCED: forcing a specific tool FIRST, then continuing

A very common requirement: "always extract metadata *before* running enrichment." Because forcing is per-turn (B6), you do it in two turns:

```python
# TURN 1 — force the first tool no matter what
resp1 = client.messages.create(
    model="claude-sonnet-4-6", max_tokens=1024,
    tools=[extract_metadata, enrich_record],
    tool_choice={"type": "tool", "name": "extract_metadata"},   # FORCED
    messages=[{"role": "user", "content": document_text}],
)
meta_call = next(b for b in resp1.content if b.type == "tool_use")
meta = run_extract_metadata(**meta_call.input)     # you run it

# Feed the result back
messages = [
    {"role": "user", "content": document_text},
    {"role": "assistant", "content": resp1.content},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": meta_call.id, "content": str(meta)}]},
]

# TURN 2 — now let Claude proceed (auto) to enrichment using the metadata
resp2 = client.messages.create(
    model="claude-sonnet-4-6", max_tokens=1024,
    tools=[extract_metadata, enrich_record],
    tool_choice={"type": "auto"},                  # now free to enrich
    messages=messages,
)
```

Turn 1 *guarantees* `extract_metadata` ran first. Turn 2 lets enrichment follow. That's the "force a tool first, process subsequent steps in follow-up turns" pattern from the exam.

## C5. ADVANCED: constrained tools instead of generic ones

A generic tool lets the agent do too much. Replace it with a **constrained** version that bakes the limit into the tool itself, so the agent *can't* misbehave — it's not relying on the prompt to stay in bounds.

```python
# BAD: a generic fetcher — the agent can pull ANY url
{"name": "fetch_url",
 "description": "Fetch the contents of any URL.",
 "input_schema": {"type": "object",
    "properties": {"url": {"type": "string"}}, "required": ["url"]}}

# GOOD: a constrained loader — validates it's an allowed document first
{"name": "load_document",
 "description": "Load an approved document by URL. Rejects non-document or "
                "disallowed URLs.",
 "input_schema": {"type": "object",
    "properties": {"document_url": {"type": "string"}}, "required": ["document_url"]}}
```

And your implementation enforces it:

```python
def run_load_document(document_url):
    if not is_allowed_document(document_url):      # your validation
        return {"error": "URL is not an approved document."}
    return fetch(document_url)
```

Now even if the agent *wants* to fetch something random, the tool refuses. The boundary lives in code, not in a hopeful prompt instruction (this connects to least-privilege, Task 1.4).

## C6. ADVANCED: a scoped cross-role tool for a frequent need

Sometimes a specialist genuinely needs one capability from another role — *often*. Don't hand it the other role's full toolset; give it **one narrow tool** for that frequent need, and route the rare/complex cases back through the coordinator.

```python
# The synthesis agent often needs to confirm a single fact.
# DON'T give it full web search (it would wander). DO give it a narrow verifier:
{"name": "verify_fact",
 "description": "Verify a single factual claim. Input: one claim string. "
                "Output: {verified: bool, source_url: string}.",
 "input_schema": {"type": "object",
    "properties": {"claim": {"type": "string"}}, "required": ["claim"]}}

synthesis_tools = [combine_findings, verify_fact]   # narrow cross-role tool, not search_web
```

Frequent need met; no general-search misuse; deep research still goes to the search subagent via the coordinator.

---

# PART D — PUTTING IT TOGETHER: a worked multi-agent design

Imagine a research system with three agents. Here's the full distribution, with `tool_choice` choices annotated:

```python
# 1) SEARCH agent — its whole job is finding sources
search_tools = [search_web, open_result]
search_tool_choice = {"type": "auto"}        # decides when to search vs report done

# 2) ANALYSIS agent — reads documents, extracts findings
analysis_tools = [load_document, extract_findings]
analysis_tool_choice = {"type": "auto"}

# 3) SYNTHESIS agent — combines findings into the final cited answer
synthesis_tools = [combine_findings, verify_fact]   # NO search_web (C2/C6)
# Force structured output so the coordinator always gets a parseable result:
synthesis_tool_choice = {"type": "any"}      # must call a tool, never ramble in prose
```

Why each decision:
- **Search agent** has only search tools (scoped, C1). `auto` because it decides when it's gathered enough.
- **Analysis agent** has only document tools — note `load_document`, the *constrained* loader, not `fetch_url` (C5).
- **Synthesis agent** is the star of C2: it deliberately lacks `search_web` (so it can't wander), but gets the narrow `verify_fact` for its frequent need (C6). It uses `any` so it always emits a structured action the coordinator can consume, never a chatty non-answer (B2).

That single example exercises every concept in this task.

---

# PART E — THE SIX EXAM SCENARIOS (now that you understand the mechanics)

Each scenario states the situation, then the 2.3-specific move with code.

## E1. Customer Support Resolution Agent
> *Support agent; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; 80%+ first-contact resolution.*

**Fit: core.** Four tools is already the right size (C1) — the lesson is *resisting sprawl*. Use forced `tool_choice` to make verification run first:
```python
# On a refund ticket, force identity verification before anything else:
tool_choice={"type": "tool", "name": "get_customer"}   # turn 1
# then continue with auto on later turns (C4)
```
Use `any` when the agent must take an action rather than chat. **Goal tie:** a small, sharp tool set selects reliably under ambiguity → more cases resolved first time.

## E2. Code Generation Pipeline
> *plan → write → test → complete.*

**Fit: strong — the forced-sequence example.** Force the plan tool first, then `auto` for write/test:
```python
tool_choice={"type": "tool", "name": "plan"}   # turn 1: guarantee planning happens
# turns 2+: tool_choice={"type": "auto"}        # write, then test (C4)
```
You can't force "plan then write" in one turn — sequence across turns (B6).

## E3. Multi-Source Research Coordinator
> *coordinator + search/analysis/synthesis subagents.*

**Fit: core — the marquee distribution example.** This is Part D verbatim. The headline: **the synthesis agent must NOT have `search_web`** (C2); give it `verify_fact` instead (C6). Scope every subagent to its role.

## E4. Developer Productivity with Claude
> *real repos; built-ins Read/Write/Edit/Bash/Grep/Glob + MCP.*

**Fit: core.** Built-ins + many MCP tools is the 18-tool trap (C1). Scope by sub-task and constrain generics:
```python
explore_tools = [Read, Grep, Glob]          # read-only explorer, NO Write/Bash
edit_tools    = [Read, Edit, Write]          # editor
# replace a generic fetcher with load_document (C5)
```

## E5. CI/CD Code Review Agent
> *headless `claude -p`; emits JSON; script gates the merge; minimize false positives.*

**Fit: strong.** Two levers: scope the reviewer to read-only tools (no merge/write — least privilege), and **guarantee structured output** so the gating script can parse it:
```python
tool_choice={"type": "any"}    # must emit a tool call (the JSON), never prose
```

## E6. Structured Data Extraction
> *extract from docs, validate against JSON schemas, high accuracy.*

**Fit: core — the canonical forced-tool example.** Force the extraction tool so you ALWAYS get a structured record:
```python
tool_choice={"type": "tool", "name": "extract_data_points"}   # guaranteed structured output
```
Combine with **strict tool use** so the arguments strictly follow your schema. Then enrich in follow-up turns (C4). **Goal tie:** forcing removes the chance of a prose answer or wrong tool → every call yields schema-shaped output.

---

# PART F — ONE-SCREEN RECAP

- A **tool** is a function you describe; Claude *requests* it, you *run* it, you return the result (the agentic loop).
- **`tool_choice`** controls a single turn: `auto` (Claude decides), `any` (must call some tool), `tool` (must call a named tool), `none` (no tool, text only).
- `any`/`tool` **suppress preamble text** and are **per-turn** — force one tool, then continue next turn. `disable_parallel_tool_use:true` caps it at one call. `any`/`tool` are incompatible with extended thinking.
- **Distribution:** give each agent only its role's ~4–5 tools; too many degrades selection; out-of-role tools get misused (synthesis agent ≠ web search).
- **Constrain** generic tools (`fetch_url` → `load_document`); give **one narrow cross-role tool** (`verify_fact`) for frequent needs, route complex cases via the coordinator.

---

### Sources verified against current Anthropic docs (June 2026)
- *Implement tool use / Define tools* (platform.claude.com) — `tools` with `name`/`description`/`input_schema`; response `stop_reason:"tool_use"` and the `tool_use`/`tool_result` block shapes; `tool_choice` modes `auto`/`any`/`{"type":"tool","name":...}`/`none`; `any` and `tool` prefill the turn so no preamble text precedes the call; `disable_parallel_tool_use:true` forces a single call; `any`/`tool` are incompatible with extended thinking; combine `any` with **strict tool use** to guarantee schema-conforming inputs.
- The "18 vs 4–5 tools" reliability point, out-of-specialty misuse (synthesis agent web-searching), the `verify_fact` scoped cross-role tool, and `fetch_url`→`load_document` come from the task statement. The Messages-API code shapes are stable, but tool-use options ship fast — re-verify field names against current docs before production.
