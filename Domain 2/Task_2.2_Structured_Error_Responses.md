# Task 2.2 — Structured Error Responses for MCP Tools
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **An error is information the agent needs to decide what to do next. A generic "Operation failed" tells it nothing, so it either gives up or retries blindly. A *structured* error — category, retryable-or-not, a human-readable reason — lets the agent make the right recovery move: retry the transient thing, stop retrying the permanent thing, fix the bad input, or explain the policy to the user.**

Everything here is the difference between an error that *informs a decision* and one that *blocks a decision*. The MCP mechanism that carries this is the `isError` flag plus structured metadata in the result.

---

## 1. The `isError` flag (how MCP reports a tool failure)

In MCP, a tool failure is **not** a protocol/transport error. The call succeeds at the protocol level, and the *result* carries `isError: true` with a descriptive message in `content`. This is deliberate: the failure is fed back to the **model** so it can see what went wrong and self-correct.

```json
{ "jsonrpc": "2.0", "id": 4, "result": {
    "content": [{ "type": "text",
      "text": "Refund denied: amount $750 exceeds the $500 auto-approval limit." }],
    "isError": true } }
```

Contrast with a **protocol error** (JSON-RPC `error` object, e.g. `-32602` "Unknown tool") — that's a malformed *request*, which the model usually can't fix. Tool *execution* errors go in the result with `isError`; the model can act on those.

---

## 2. The four error categories (why classification matters)

The agent's correct next move depends entirely on *what kind* of failure happened:

| Category | Examples | Right agent response |
|---|---|---|
| **Transient** | Timeout, service unavailable, rate limit | Retry (often after a wait) |
| **Validation** | Invalid input, malformed argument | Fix the input and retry |
| **Business** | Policy violation (refund over limit, account locked) | Don't retry; explain/escalate |
| **Permission** | Not authorized, missing scope | Don't retry as-is; escalate/auth |

A generic "failed" collapses all four into one, so the agent can't tell "wait and retry" from "stop, this will never work." That's why uniform errors are harmful.

---

## 3. Retryable vs non-retryable (stop wasting retries)

The single most useful piece of metadata is **`isRetryable`** (a.k.a. `retriable`). Transient errors are retryable; validation/business/permission errors usually are not. Returning `isRetryable: false` for a policy violation stops the agent from hammering a call that can *never* succeed — saving latency, tokens, and money. Returning `isRetryable: true` for a timeout tells it a retry is worth attempting.

---

## 4. The structured error payload (the "Skills in" shape)

Return metadata the agent can branch on, not just prose:

```json
{ "isError": true,
  "content": [{ "type": "text", "text":
    "We can't refund this order because it's outside the 30-day window." }],
  "structuredContent": {
    "errorCategory": "business",      // transient | validation | business | permission
    "isRetryable": false,             // stops pointless retries
    "userMessage": "Refunds are only available within 30 days of purchase."
  } }
```

- **`errorCategory`** drives the branch (retry / fix input / explain / escalate).
- **`isRetryable`** prevents wasted attempts.
- **Human-readable reason** — for business violations, a *customer-friendly* explanation the agent can relay verbatim, so it communicates appropriately instead of leaking internals.

(`errorCategory`/`isRetryable` aren't MCP-native field names — they're a best-practice convention you put in the content/`structuredContent`. The native flag is `isError`.)

---

## 5. Two subtleties the exam tests

**Local recovery vs propagation (in multi-agent setups).** A subagent should handle **transient** failures itself (retry locally) and only propagate to the coordinator what it **can't** resolve — and when it does propagate, include **partial results** and **what was attempted**, so the coordinator isn't blind. Don't bubble every blip up to the top.

**Access failure vs valid empty result.** "Query returned 0 rows" is a **success**, not an error — the question was answered, the answer is "none." Marking an empty result as `isError` makes the agent retry a query that's working fine. Reserve `isError` for *access/execution* failures (couldn't reach the service); return empty results as normal successful output.

---

## 6. Anti-patterns (with *why*)
- **Generic "Operation failed."** Erases the category the agent needs; it can't choose a recovery move.
- **No `isRetryable`.** Agent retries permanent failures forever (or never retries a transient one).
- **Raw stack traces / SQL errors as the message.** Confuses the model and leaks internals; return a sanitized, descriptive reason.
- **Empty result flagged as error.** Triggers pointless retries of a working query.
- **Propagating every subagent blip to the coordinator.** Recover transient failures locally; only escalate the unrecoverable, with partial results + what was tried.
- **Using a JSON-RPC protocol error for a business failure.** The model can't self-correct on protocol errors; put recoverable failures in the result with `isError`.

---

## 7. Self-check (core mechanics)
1. How does MCP report a tool failure? → In the result with `isError: true` (not a protocol error), so the model can self-correct.
2. The four categories? → Transient, validation, business, permission.
3. Most useful metadata field, and why? → `isRetryable` — stops wasted retries on permanent failures.
4. What goes in a business-violation error? → A customer-friendly explanation the agent can relay.
5. Empty query result — error or success? → **Success** (valid empty result); don't flag `isError`.
6. Subagent transient failure — handle where? → **Locally**; propagate only the unrecoverable, with partial results + attempts.
7. (Ties to §0) One sentence? → *Make errors carry the decision: category + retryable + a human reason.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through four backend tools (see Domain-1 §7).

**The key link to everything above:** **Core fit — the home scenario.** Every error category appears: `lookup_order` timing out (transient → retry), a bad order ID (validation → ask the customer), a refund over policy (business → `isRetryable:false` + customer-friendly message), an unauthorized action (permission → escalate). Structured errors are what let the agent resolve more cases without dead-ending.

### 8a. Category-specific recovery
Map each tool failure to a category so the agent retries the timeout but immediately explains the over-limit refund instead of retrying it.

### 8b. Customer-friendly business messages
`process_refund` policy denial returns `userMessage: "Refunds are available within 30 days..."` — the agent relays it directly, no internal jargon.

### 8c. Empty vs failed
`lookup_order` finding no orders for a new customer is a **valid empty result**, not an error — the agent says "no orders found," doesn't retry.

### 8d. Scenario-1 self-check
1. Refund-over-limit error fields? → `errorCategory:"business"`, `isRetryable:false`, customer-friendly `userMessage`.
2. `lookup_order` timeout? → Transient, retryable; agent retries.
3. No orders found? → Valid empty result (success), not `isError`.
4. (Goal tie) How do structured errors raise FCR? → The agent recovers correctly per category instead of dead-ending, so more cases close first time.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Strong fit.** Test/build tools fail in distinct ways: a flaky test runner timing out (transient → retry) vs a real compile error (validation-like → fix the code, don't blindly retry). Structured errors tell the agent which, so it doesn't "retry" its way past a genuine bug.

### 9a. Distinguish flaky from real failures
A transient runner crash returns `isRetryable:true`; a compile error returns a descriptive message the agent fixes, `isRetryable:false`.

### 9b. Scenario-2 self-check
1. Flaky test infra vs real compile error — same handling? → No: transient-retry vs fix-the-code.
2. What prevents "retrying" past a real bug? → `isRetryable:false` + a descriptive reason.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research, then synthesizing (see Domain-1 §9).

**The key link to everything above:** **Core fit for §5.** This is the textbook **local-recovery-vs-propagation** case. A search subagent hitting a rate limit should retry locally; only if it truly can't get results does it propagate to the coordinator — and it sends **partial results** (what it did find) plus **what it attempted**, so synthesis can proceed with caveats instead of failing wholesale. Also the **empty vs failed** distinction: "no sources found" is a valid answer, not an error.

### 10a. Recover transient locally, propagate the rest
Subagent retries the rate-limited search itself; escalates only unrecoverable failures with partial findings attached.

### 10b. Empty result is signal, not failure
A subagent finding nothing returns an empty success; the coordinator notes the gap rather than retrying a working search.

### 10c. Scenario-3 self-check
1. Where is a transient search failure handled? → Locally in the subagent.
2. What accompanies an escalated failure? → Partial results + what was attempted.
3. "No sources found" — error? → No; valid empty result.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores codebases, handles legacy, generates boilerplate, automates chores.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Strong fit.** MCP tools wrapping real services (DB, CI, issue tracker) need structured errors: a DB connection blip (transient → retry) vs a read-only permission denial (permission → don't retry, surface it) vs a query returning zero rows (valid empty result). Clear categories keep the dev agent from thrashing.

### 11a. Permission vs transient on infra tools
A `db_query` auth failure returns `errorCategory:"permission", isRetryable:false`; a connection reset returns transient/retryable.

### 11b. Scenario-4 self-check
1. DB auth denial fields? → permission, not retryable, surfaced to the user.
2. Connection reset? → Transient, retryable.
3. Zero rows from a query? → Valid empty result, not an error.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Strong fit.** In headless CI there's no human to interpret a vague failure, so the gating script must branch on structured errors: a transient API hiccup should retry (don't fail the build over flakiness — that's a false failure); a genuine permission/validation error should fail clearly. `isRetryable` is what separates "retry the build step" from "stop and report."

### 12a. Don't fail a build on transient errors
The script reads `isRetryable:true` and retries the review step rather than blocking the PR on a network blip.

### 12b. Scenario-5 self-check
1. Why structured errors in headless CI? → No human to interpret; the script must branch automatically.
2. Transient API error during review? → Retry the step (`isRetryable:true`), don't fail the build.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Core fit.** "Handle edge cases gracefully" *is* structured error handling. A schema-validation failure is a **validation** error (fix/re-extract, often retryable once); an unreadable/corrupt document is a different failure (don't retry forever); a document that legitimately contains none of the target fields is a **valid empty result**, not an error. Distinguishing these is what keeps accuracy high and the downstream system clean.

### 13a. Validation failure → targeted retry
A record failing the schema returns `errorCategory:"validation"` with the specific field; the agent re-extracts that field rather than discarding the whole doc.

### 13b. Empty vs failed, again
"Document has no invoice number because it isn't an invoice" → valid empty/typed result, not `isError`. Flagging it as error would trigger pointless retries and pollute metrics.

### 13c. Scenario-6 self-check
1. Schema-validation failure category? → Validation (fix/re-extract).
2. Target field genuinely absent? → Valid empty result, not an error.
3. (Goal tie) How does this protect "high accuracy" + clean downstream? → Only true failures retry/flag; valid-empty and recoverable cases are handled correctly, so bad data never flows on.

---

### Sources verified against current Anthropic / MCP docs (June 2026)
- *MCP specification — Tools* (modelcontextprotocol.io) — tool execution errors are returned **inside the result with `isError: true`** (not as JSON-RPC protocol errors) so the model can self-correct; `content` carries a descriptive (sanitized) message; `structuredContent` carries structured data; output-schema validation is skipped for `isError` responses.
- *MCP error-handling guides* (community + SDK docs) — error messages should be model-readable and sanitized (no raw stack traces / SQL); `CallToolResult(..., isError=True)` pattern in the Python/TS SDKs.
- `errorCategory` / `isRetryable` / `userMessage` are a **best-practice metadata convention** placed in content/`structuredContent`, not native MCP field names — confirm your server's exact schema. The four-category taxonomy and local-recovery/propagation guidance reflect the task statement.
