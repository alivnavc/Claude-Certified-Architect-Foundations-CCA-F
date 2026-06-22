# Task 1.4 — Multi-Step Workflows with Enforcement & Handoff Patterns
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–4 are the mechanics (every "Knowledge of" / "Skills in" bullet). Section 5 lists anti-patterns. Section 6 is a self-check. Then one section per scenario (7–12), each in the same shape: verbatim description → plain English → the key link → mechanics → scenario self-check.

---

## 0. The one idea everything hangs on

> **A prompt *requests*; code *guarantees*. When a step MUST happen before another — verify identity before you move money — you do not ask the model nicely in the system prompt. You put a gate in code that the model physically cannot get around. And when you hand a half-finished case to a human, you give them a structured summary, because they cannot see your conversation.**

Everything in this task is one comparison: **prompt-based guidance** (probabilistic — works *most* of the time) versus **programmatic enforcement** (deterministic — works *every* time). You reach for enforcement exactly when "most of the time" is unacceptable.

---

## 1. Prompt guidance vs programmatic enforcement (the core distinction)

| | Prompt-based guidance | Programmatic enforcement |
|---|---|---|
| Where the rule lives | System-prompt text ("always verify before refunding") | Code: a **hook** or a **prerequisite gate** |
| Guarantee | Probabilistic — high but **non-zero failure rate** | Deterministic — the action cannot fire if the gate fails |
| Fails when | Model is distracted, jailbroken, mis-reasons, hits an edge case | Essentially never (it is plain code, not a model decision) |
| Use for | Style, tone, preferences, soft ordering | Money, identity, data deletion, compliance, anything irreversible |

**Why "non-zero failure rate" is the whole argument.** A model following an instruction correctly 99% of the time still violates it 1 in 100 times. On a payment path that is unacceptable, and the failures cluster exactly on the weird inputs you most want handled correctly. So the rule moves out of the prompt and into code, where 99% becomes 100%.

---

## 2. Prerequisite gates (block a tool until a precondition is met)

A **prerequisite gate** is code that refuses to let tool B run until tool A has produced a valid result. Canonical example: *block `process_refund` until `get_customer` has returned a verified customer ID.*

You implement the gate as a `PreToolUse` hook (full hook mechanics live in Task 1.5; here we use just enough). The hook inspects the pending call, checks your own state, and returns `permissionDecision: "deny"` if the precondition is unmet.

```python
verified_ids = set()   # your state, populated when get_customer succeeds

async def refund_gate(input_data, tool_use_id, context):
    if input_data["tool_name"] == "process_refund":
        cust = input_data["tool_input"].get("customer_id")
        if cust not in verified_ids:                       # precondition unmet
            return {"hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",              # block it
                "permissionDecisionReason":
                  "Identity not verified — call get_customer first."}}
    return {}   # empty = allow
```

The denial message is **fed back to the model**, so Claude reads "verify first," calls `get_customer`, and retries the refund legitimately. The gate didn't break the agent — it *redirected* it.

---

## 3. Decomposing multi-concern requests

Real customer messages bundle several problems: *"my refund never arrived AND I was double-charged AND I can't log in."* The pattern: **split into distinct items, investigate each (often in parallel), then synthesize one unified resolution** — rather than fixating on whichever issue you noticed first. (Parallel investigation mechanics = Task 1.3; deciding *how* to split = Task 1.6.) The enforcement angle is that each item still passes through the same gates before any action fires.

---

## 4. Structured handoff protocols

When the agent escalates mid-process, the human picks up **with none of your context** — they can't see the transcript. A structured handoff summary carries everything they need as data, not prose:

| Field | Example |
|---|---|
| Customer ID | `CUST-48217` (verified) |
| Root cause | Duplicate charge from a retried payment webhook |
| Relevant amount | $129.99 |
| Recommended action | Refund one charge; no fraud indicators |
| What's been tried | `get_customer` ✓, `lookup_order` ✓, refund **blocked** (> $100 policy) |

The point: the human reads five fields, not fifty messages. Same idea reappears in subagent→coordinator handoffs (Task 1.3's structured `{content, metadata}`).

---

## 5. Anti-patterns (each with *why* it's wrong)

- **Trusting the prompt for a hard rule.** "I told it to verify first" — non-zero failure rate means it eventually won't. Gate it in code.
- **Hard-blocking with no redirect.** Denying `process_refund` and stopping is a dead end. Always return a reason so the model can recover (verify, then retry; or escalate).
- **Free-text handoffs.** "Customer seems upset about a charge" forces the human to re-investigate. Hand off structured fields.
- **Tunnel vision on one concern.** Acting on the first issue and dropping the other two tanks first-contact resolution. Decompose first.

---

## 6. Self-check (core mechanics)
1. Why not enforce "verify before refund" in the system prompt? → Prompts have a **non-zero failure rate**; money needs a deterministic **gate**.
2. What does a prerequisite gate return to block a call? → `permissionDecision: "deny"` (plus a reason) from a `PreToolUse` hook.
3. Where does the denial reason go? → Back to the model as context, so it can correct course.
4. A message bundles three complaints. First move? → **Decompose** into distinct items, investigate each, synthesize one resolution.
5. What goes in an escalation handoff? → Structured fields: customer ID, root cause, amount, recommended action, what's been tried.
6. (Ties to §0) One sentence for the whole task? → *Prompts request; code guarantees — gate the must-happens and hand off as data.*

---

## 7. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *You are building a customer support resolution agent using the Claude Agent SDK. The agent handles high-ambiguity requests like returns, billing disputes, and account issues. It has access to your backend systems through custom MCP tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`). Your target is 80%+ first-contact resolution while knowing when to escalate.*

**What this scenario is even about (plain English):** A support agent that talks to a customer and can actually *do* things in your backend. **MCP** (Model Context Protocol) is the standard way you expose backend functions to Claude as "tools" — here, four of them: look up a customer, look up an order, issue a refund, and hand off to a human. "High-ambiguity" means messages don't arrive in tidy categories. "First-contact resolution" = solved on the first interaction without bouncing the customer around; 80%+ is the bar. Success = resolve most cases automatically, escalate the rest cleanly.

**The key link to everything above:** This is the home scenario for Task 1.4 — **core fit.** `process_refund` is money, so "verify identity first" cannot live in the prompt; it's a **prerequisite gate** (§2). Disputes bundle several issues, so you **decompose** (§3). `escalate_to_human` needs a **structured handoff** (§4) because the human can't see the chat.

### 7a. The refund gate in this scenario
`get_customer` must return a verified ID before `process_refund` is allowed. Implement exactly the gate in §2, keyed on the customer ID. Policy thresholds (e.g., auto-refund under $100, escalate over) ride in the same hook — over-threshold returns `deny` with a reason that routes to `escalate_to_human` instead of failing.

### 7b. Decompose the dispute
"Refund missing + double charge + can't log in" → three items. Run `lookup_order` / `get_customer` per item, then give one consolidated answer. Don't resolve the login issue and forget the money.

### 7c. Escalation handoff
When `escalate_to_human` fires, attach the §4 table: verified customer ID, root cause, amount, recommended action, tools already run. The human acts in seconds.

### 7d. Scenario-1 self-check
1. Where does "verify before refund" live? → A code **gate**, not the prompt.
2. Refund is $400, policy caps auto at $100. What happens? → Gate **denies**, reason redirects to `escalate_to_human`.
3. Three complaints in one message? → **Decompose**, investigate each, unify.
4. What does the human receive on escalation? → A **structured summary**, not the transcript.
5. (Goal tie) How does enforcement serve 80% FCR? → Gates prevent the unsafe-action failures that would otherwise force escalations, so more cases resolve safely on first contact.

---

## 8. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (as given, paraphrased — confirm against your exam copy):** *You are building a multi-step code-generation system on the Agent SDK: it plans a change, writes the code, runs tests, and only "completes" work that passes. Generated code must never be committed or merged unless tests and checks pass.*

**What this scenario is even about (plain English):** An agent that produces code in stages — plan → write → test → finish — rather than dumping a blob. The hard rule is the "gate at the end": nothing ships unless tests are green.

**The key link to everything above:** **Strong fit.** "Don't mark done while tests fail" is the same non-zero-failure argument (§1): a prompt saying "only finish if tests pass" will occasionally finish anyway. Enforce it with a `Stop`/completion gate (a hook that returns `block` while a real check fails), the §0 spine applied to the *end* of the workflow instead of the start.

### 8a. The completion gate
A `Stop` hook re-runs the test command; if it fails, it returns `decision: "block"` with the failures as the reason, forcing the agent to keep fixing. It returns clean (exit 0 / `{}`) only when tests pass — so the gate is tied to a *real condition*, never an arbitrary turn count.

### 8b. Ordering as prerequisites
"Write before test, test before complete" are prerequisite gates (§2) along the pipeline: block `commit`/`complete` until the test tool returned success.

### 8c. Scenario-2 self-check
1. Why not "only finish if tests pass" in the prompt? → Non-zero failure rate; gate completion in code.
2. What forces another iteration when tests fail? → A `Stop` hook returning **block** with the failure as reason.
3. What must the gate be tied to? → A **real check** (test result), not an iteration cap.

---

## 9. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (as given, paraphrased — confirm against your exam copy):** *A coordinator agent answers complex questions by spawning subagents that search the web and analyze documents, then a synthesis subagent combines their findings into a cited answer.*

**What this scenario is even about (plain English):** One "manager" agent delegates to specialist sub-agents (search, read, synthesize) and assembles a final, sourced answer. Most of the machinery here is Tasks 1.2/1.3 (orchestration and spawning).

**The key link to everything above:** **Partial fit for 1.4** — research has few hard "must-happen-first" gates. Where 1.4 *does* show up: the **handoff** idea (§4). Findings passed to the synthesis subagent must be **structured** (`{claim, source_url, doc, page}`) so citations survive — the same "hand off as data, not prose" rule, applied agent-to-agent instead of agent-to-human.

### 9a. Structured handoff between agents
Don't pass a paragraph of mixed findings; pass a list of `{content, metadata}` records. The synthesis agent can then attribute every claim. (Mechanics: Task 1.3.)

### 9b. Where enforcement is *light* here
There's no money or identity step, so heavy gates are overkill. Recognizing "this workflow doesn't need a prerequisite gate" is itself an architect judgment the exam rewards — enforcement is a tool, not a reflex.

### 9c. Scenario-3 self-check
1. Why is 1.4 only a partial fit? → No irreversible/compliance step needs a hard gate.
2. What 1.4 idea still applies? → **Structured handoff** to preserve citations.
3. Form of the handoff? → `{content, metadata}` records, not prose.

---

## 10. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *You are using the Claude Agent SDK to explore unfamiliar codebases, understand legacy systems, generate boilerplate, and automate repetitive tasks. The agent uses built-in tools (Read, Write, Bash, Grep, Glob) and integrates MCP servers.*

**What this scenario is even about (plain English):** A developer-assistant agent that can read files, search code (Grep/Glob), write files, and run shell commands (Bash) across a real repo. "Legacy" = old code nobody fully remembers. "Boilerplate" = repetitive scaffolding.

**The key link to everything above:** **Strong fit on the safety side.** Bash and Write are dangerous tools, so 1.4's enforcement spine applies: gate destructive actions. "Never `rm -rf` outside the work dir," "never write to `.env`," "no force-push" are **prerequisite/guard gates** (§2), not prompt pleas — because a prompt asking the agent to be careful has a non-zero failure rate, and here a failure deletes files.

### 10a. Guard gates on dangerous tools
A `PreToolUse` hook matched to `Bash` denies commands matching destructive patterns; one matched to `Write|Edit` denies sensitive paths. Resolve real paths before comparing (a naive prefix check is bypassable with `..`).

### 10b. Least privilege as enforcement
A read-only exploration sub-agent simply isn't given `Write`/`Bash`. Removing a capability is the strongest gate of all — there's nothing to block because the tool isn't there.

### 10c. Scenario-4 self-check
1. Why gate Bash instead of trusting the prompt? → A careless command deletes data; non-zero failure rate is unacceptable.
2. Strongest enforcement for a read-only reviewer? → Don't grant Write/Bash at all (least privilege).
3. Path-jail gotcha? → Resolve real paths first; `..`/symlinks bypass naive prefix checks.

---

## 11. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (as given, paraphrased — confirm against your exam copy):** *You integrate Claude Code into your CI/CD pipeline as an automated reviewer. On each pull request it runs headlessly (`claude -p`), outputs structured JSON, and a script gates the merge. You want to minimize false positives.*

**What this scenario is even about (plain English):** **CI/CD** is automation that runs on every code change. A **pull request (PR)** is a proposed change. Here Claude reviews each PR automatically in "headless" mode (`claude -p`, no human chatting), emits JSON, and a script decides pass/fail. "False positive" = flagging a problem that isn't real; too many and developers ignore the bot.

**The key link to everything above:** **Strong fit.** The *merge gate itself* is pure 1.4 enforcement: the decision to block or allow a merge is **deterministic code reading the JSON**, never the model's free-text "looks fine to me." The model advises; the pipeline enforces. That's §0 — code guarantees — at the CI boundary.

### 11a. Enforce on structured output, not prose
Force the review into JSON (`{verdict, issues:[...]}`) and have the CI script gate on `verdict`. Gating on natural-language ("if the reply contains 'approved'") is the classic broken pattern — brittle and spoofable.

### 11b. Least privilege in CI
The reviewer gets read + analysis tools only; it cannot push, merge, or write. The enforcement is structural.

### 11c. Scenario-5 self-check
1. What makes the merge decision? → **Deterministic script** reading JSON, not the model's prose.
2. Why JSON output? → So enforcement is reliable, not string-matching on text.
3. Reviewer's tool set? → Read-only; no merge/push (least privilege).

---

## 12. Scenario 6 — Structured Data Extraction

> **Scenario 6 (as given, paraphrased — confirm against your exam copy):** *A system extracts information from unstructured documents, validates the output against JSON schemas, maintains high accuracy, handles edge cases gracefully, and integrates with downstream systems.*

**What this scenario is even about (plain English):** Turn messy documents (PDFs, emails) into clean structured records, then *check* each record against a **JSON schema** (a rulebook for shape/types) before it flows downstream. "Edge cases" = malformed or surprising inputs you must handle without crashing.

**The key link to everything above:** **Core fit.** Schema **validation + reject/retry** is exactly the 1.4 spine: don't *trust* the model to produce valid data (prompts have a non-zero failure rate) — *enforce* validity with deterministic code, and on failure, block-and-retry rather than passing bad data on. The schema check is a prerequisite gate guarding the downstream handoff.

### 12a. Validate-then-gate
After extraction, run the record through the schema validator. Pass → forward downstream. Fail → reject and retry (or route to human), never emit. This is a `PostToolUse`-style guard plus a downstream gate.

### 12b. Graceful degradation as enforced behavior
"Handle edge cases gracefully" means the failure path is *defined in code* (retry, flag, escalate) — not left to the model to improvise. Defined failure handling is enforcement.

### 12c. Structured handoff downstream
The downstream system is the "human who can't see the transcript" of §4 — it needs clean, well-typed records, which is why validation is non-negotiable before handoff.

### 12d. Scenario-6 self-check
1. Why not trust the model to emit valid JSON? → Non-zero failure rate; **validate in code**.
2. What happens on a schema failure? → **Reject + retry/escalate**, never forward bad data.
3. How is "graceful edge-case handling" enforcement? → The failure path is coded, not improvised.
4. (Goal tie) How does this protect "high accuracy"? → The deterministic gate guarantees only schema-valid records reach downstream, so accuracy isn't left to chance.

---

### Sources verified against current Anthropic docs (June 2026)
- *Control agent behavior with hooks* (platform.claude.com, code.claude.com) — `PreToolUse` returns `permissionDecision` (`allow`/`deny`/`ask`), `permissionDecisionReason`, `updatedInput`; denial reason is shown to the model; precedence **deny > ask > allow**; `Stop` hook returns `decision:"block"` to force continuation. (Hook detail fully unfolded in Task 1.5.)
- *Tool-use / stop reasons* — gates implemented as hooks fit inside the same agentic loop (Task 1.1).
- Enforcement-vs-prompt, prerequisite gates, structured handoff reflect the task statement and Anthropic's documented reliability guidance. Hook field names ship fast — re-verify against current docs before writing production code.
