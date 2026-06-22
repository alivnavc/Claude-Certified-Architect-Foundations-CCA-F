# Task 1.5 — Agent SDK Hooks for Tool-Call Interception & Data Normalization
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics (every "Knowledge of" / "Skills in" bullet). Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **A hook is *your* code spliced into the agent's loop at a fixed moment. `PreToolUse` runs *before* a tool call — so it can block or rewrite the call. `PostToolUse` runs *after* a tool returns — so it can clean up or replace the result before the model ever sees it. Because hooks are plain code, they give *deterministic* guarantees that prompts (probabilistic) cannot.**

This is the same prompts-vs-code spine as Task 1.4, but zoomed into the *mechanism*: hooks are the actual lever. Two jobs the exam cares about: **intercept/block** outgoing calls (compliance) and **normalize** incoming results (clean messy data).

---

## 1. What a hook is, and the lifecycle

The agent loop fires named events: a tool is about to run (`PreToolUse`), a tool returned (`PostToolUse`), a subagent started/stopped, the run finished, etc. You register callbacks on the events you care about; the SDK calls your code at that moment with details about what's happening, and your return value can change what happens next.

The two you must know cold:

| Hook | When it fires | What it can do |
|---|---|---|
| `PreToolUse` | Just **before** a tool runs | Allow, **deny** (block), **rewrite inputs**, add context |
| `PostToolUse` | Just **after** a tool returns | **Replace/normalize** the result, append context |

`PreToolUse` is the only one that can *stop* an action. `PostToolUse` can't undo a call (the tool already ran) but can change what the model reads.

---

## 2. Configuring hooks (Python — one canonical example)

```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions, HookMatcher

async def my_pre_hook(input_data, tool_use_id, context):
    # input_data has: tool_name, tool_input (the args), session_id, ...
    return {}   # {} = allow unchanged

options = ClaudeAgentOptions(
    hooks={                                   # dict: event name -> list of matchers
        "PreToolUse": [
            HookMatcher(matcher="process_refund", hooks=[my_pre_hook]),  # only this tool
            HookMatcher(hooks=[audit_logger]),                           # no matcher = all tools
        ]
    }
)
```

- **Keys** are event names (`"PreToolUse"`, `"PostToolUse"`, `"Stop"`, …).
- **`matcher`** is a regex tested against the tool name. `"Write|Edit"` matches either; omitted/`"*"` matches everything. MCP tools are named `mcp__<server>__<tool>`.
- **`hooks`** is a list of callback functions. Each gets `(input_data, tool_use_id, context)`.

TypeScript is the same shape with camelCase: `options.hooks = { PreToolUse: [{ matcher: "...", hooks: [fn] }] }`.

---

## 3. Intercepting / blocking outgoing calls (`PreToolUse`)

Return a `hookSpecificOutput` with a `permissionDecision`:

| `permissionDecision` | Effect |
|---|---|
| `"allow"` | Let it run (bypasses the normal permission prompt) |
| `"deny"` | **Block** it; `permissionDecisionReason` is shown to the model |
| `"ask"` | Pause and ask the human |

You can also return `updatedInput` to **rewrite the arguments** before the call runs, and (newer) `additionalContext` to inject a note.

```python
async def block_big_refund(input_data, tool_use_id, context):
    if input_data["tool_name"] == "process_refund":
        amt = input_data["tool_input"].get("amount", 0)
        if amt > 500:
            return {"hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason":
                  "Refund > $500 must go to human review."}}   # redirects the model
    return {}
```

**Precedence when multiple hooks/rules apply: `deny` > `ask` > `allow`.** Any `deny` wins. Evaluation order overall: **hooks run first**, then deny rules, then permission mode, then the `can_use_tool` callback.

---

## 4. Normalizing incoming results (`PostToolUse`)

Different MCP tools return the same concept in different shapes — one gives a **Unix timestamp** (`1718841600`), another **ISO 8601** (`2026-06-20T00:00:00Z`), another a **numeric status code** (`2`). If the model has to reconcile all that, it wastes reasoning and makes mistakes. A `PostToolUse` hook converts everything to one canonical shape *before* the model reads it.

Return `updatedToolOutput` to replace what the model sees (works for **any** tool; the older `updatedMCPToolOutput` is MCP-only and deprecated). `additionalContext` instead *appends* a note without replacing.

```python
async def normalize_dates(input_data, tool_use_id, context):
    out = input_data.get("tool_response", {})
    if "created" in out:
        out["created"] = to_iso8601(out["created"])  # Unix/ISO/code -> one ISO format
    return {"hookSpecificOutput": {
        "hookEventName": "PostToolUse",
        "updatedToolOutput": out}}                    # model sees the cleaned version
```

---

## 5. Deterministic guarantees vs probabilistic compliance (when to choose hooks)

This is the "Knowledge of" payoff. Choose a **hook** over a **prompt instruction** whenever a business rule requires *guaranteed* compliance:

| Need | Mechanism |
|---|---|
| "Refunds over $500 must never auto-process" | `PreToolUse` deny hook (guaranteed) |
| "Please prefer concise answers" | Prompt (preference; probabilistic is fine) |
| "Always show the model clean ISO dates" | `PostToolUse` normalize hook (guaranteed) |
| "Try to cite sources" | Prompt (soft goal) |

Rule of thumb: **hard rule, money, compliance, data shape → hook. Soft preference → prompt.**

---

## 6. Anti-patterns (with *why*)
- **Prompt-enforcing a hard rule.** "I instructed it not to refund over $500" — non-zero failure rate; use a deny hook.
- **Blocking with no reason.** `deny` without `permissionDecisionReason` leaves the model stuck; the reason is how it recovers/redirects.
- **Mixing up output shapes.** A `SubagentStart`/`Stop` lifecycle hook uses the top-level `decision:"block"` form, not a `PreToolUse` `permissionDecision`. Wrong shape = your block is silently ignored.
- **Normalizing in the prompt.** Asking the model to "treat all timestamps as UTC" is fragile; convert in a `PostToolUse` hook so the data is already clean.
- **Over-broad matchers.** No matcher means the hook fires on *every* tool; scope with a regex when you mean one tool.

---

## 7. Self-check (core mechanics)
1. Which hook can block an action? → `PreToolUse` (via `permissionDecision: "deny"`).
2. Which hook cleans a result before the model sees it? → `PostToolUse` (via `updatedToolOutput`).
3. Precedence of `allow`/`deny`/`ask`? → **deny > ask > allow**.
4. How do you rewrite a tool's arguments before it runs? → `updatedInput` on a `PreToolUse` hook.
5. How do you scope a hook to just `Bash`? → `HookMatcher(matcher="Bash", ...)`.
6. Hook vs prompt for "never refund > $500"? → **Hook** — it needs a guaranteed (deterministic) result.

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution with clean escalation.*

**What this scenario is even about (plain English):** (Same setup as Task 1.4 §7.) An agent that resolves support cases via four backend tools, automatically where safe and escalating otherwise.

**The key link to everything above:** **Core fit.** Hooks are the *implementation* of this scenario's safety rules. The $500 refund block is the §3 deny hook verbatim. Heterogeneous backend tools (orders in Unix time, accounts in ISO) get a §4 normalize hook. This is where "deterministic vs probabilistic" (§5) becomes concrete: money rules are hooks, tone is the prompt.

### 8a. Block-and-redirect the over-threshold refund
The §3 `block_big_refund` hook denies refunds over $500 and the reason routes the model to `escalate_to_human`. Guaranteed — not a hope.

### 8b. Normalize backend data
`lookup_order` returns Unix timestamps, `get_customer` returns ISO; a `PostToolUse` hook converts both to one format so the model reasons cleanly and doesn't mis-compare dates.

### 8c. Audit hook for FCR analytics
A no-matcher `PostToolUse` hook logs every tool call + decision, giving you the data to measure and improve first-contact resolution.

### 8d. Scenario-1 self-check
1. How is "never auto-refund > $500" guaranteed? → `PreToolUse` **deny** hook.
2. Where does the blocked refund go? → Reason redirects to `escalate_to_human`.
3. Fix for mixed timestamp formats? → `PostToolUse` `updatedToolOutput` normalization.
4. (Goal tie) How do hooks raise FCR? → They make safe auto-resolution reliable and give clean data, so more cases close correctly on first contact.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete; nothing ships unless checks pass.*

**What this scenario is even about (plain English):** An agent that writes code in stages and must not declare success on failing tests. (See Task 1.4 §8.)

**The key link to everything above:** **Strong fit.** The "don't finish on red" rule is a `Stop` hook returning `decision:"block"`; auto-formatting written files is a `PostToolUse` hook on `Write|Edit`. Both are §0 — your code in the loop guaranteeing behavior.

### 9a. Format-on-write
`PostToolUse` matched to `Write|Edit` runs a formatter so every generated file is clean before the model continues — deterministic, zero prompts.

### 9b. Completion gate (lifecycle hook shape)
A `Stop` hook re-runs tests and returns the top-level `decision:"block"` form (not a `PreToolUse` permission shape — §6 anti-pattern) until they pass.

### 9c. Scenario-2 self-check
1. Hook to auto-format new files? → `PostToolUse` on `Write|Edit`.
2. Hook shape to force more work when tests fail? → `Stop` hook, `decision:"block"`.
3. Why not a `permissionDecision` there? → That's only for `PreToolUse`; lifecycle hooks use `decision`.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent to produce a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research to specialists, then synthesizing. (See Task 1.4 §9.)

**The key link to everything above:** **Strong fit on governance.** Hooks here aren't about money — they're about **controlling subagents and normalizing sources**. A `SubagentStart` hook enforces a *spawn budget* (block after N subagents) so a runaway coordinator can't fan out forever; a `PostToolUse` hook on search tools normalizes each source into `{content, metadata}` before synthesis.

### 10a. Spawn-budget guard
`SubagentStart` returns `decision:"block"` once the count exceeds the budget — deterministic protection against uncontrolled spawning. (Lifecycle shape, not `permissionDecision`.)

### 10b. Source normalization
A `PostToolUse` hook tags every search result with `{url, title, retrieved_at}` so citations survive the handoff to synthesis — feeding Task 1.3's structured passing.

### 10c. Scenario-3 self-check
1. Hook to cap subagent count? → `SubagentStart`, `decision:"block"` over budget.
2. Hook to make sources citation-ready? → `PostToolUse` normalize into `{content, metadata}`.
3. Output shape for a lifecycle hook? → Top-level `decision`, not `permissionDecision`.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Bash, Grep, Glob; integrates MCP; explores codebases, handles legacy, generates boilerplate, automates chores.*

**What this scenario is even about (plain English):** A coding assistant that can read, search, write, and run shell commands across your repo. (See Task 1.4 §10.)

**The key link to everything above:** **Strong fit.** Hooks are the safety layer over dangerous tools: a `PreToolUse` on `Bash` denies destructive commands; a `PreToolUse` on `Write|Edit` denies sensitive paths (e.g., `.env`); a `PostToolUse` runs linters. The canonical `.env`-protection hook in the docs is exactly this.

### 11a. Protect sensitive files
`PreToolUse` matched to `Write|Edit`: if the path is `.env` (resolve the real path first), return `deny` with a reason. Guaranteed protection no prompt can match.

### 11b. Kill dangerous Bash
`PreToolUse` on `Bash`: deny `rm -rf /`, fork bombs, piped-curl installers. Denials are model context, so the agent reads why and tries a safe alternative.

### 11c. Lint-on-write
`PostToolUse` on `Write|Edit` runs the linter/formatter so output stays clean automatically.

### 11d. Scenario-4 self-check
1. Hook to stop edits to `.env`? → `PreToolUse` deny on a `Write|Edit` matcher.
2. Path-comparison gotcha? → Resolve real paths first (`..`/symlinks bypass naive checks).
3. Hook to auto-lint generated code? → `PostToolUse` on `Write|Edit`.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer running in your pipeline, emitting machine-readable JSON. (See Task 1.4 §11.)

**The key link to everything above:** **Strong fit.** In headless CI you can't click "approve," so hooks (and settings-file hooks) are how you enforce policy automatically: `PreToolUse` denies any write/push tool (review must be read-only); a `PostToolUse`/output step shapes the result into the JSON the merge script reads.

### 12a. Read-only enforcement via hooks
`PreToolUse` denies `Write`, `Bash`, push/merge tools so a CI reviewer structurally cannot modify the repo — least privilege made deterministic.

### 12b. Normalize the review into JSON
A `PostToolUse`/output-format step guarantees the `{verdict, issues}` shape the pipeline gates on, instead of relying on the model to remember the format.

### 12c. Scenario-5 self-check
1. How do you enforce read-only in headless CI? → `PreToolUse` deny on write/push tools.
2. Why hooks instead of clicking approve? → No human in the loop; enforcement must be code.
3. What guarantees parseable output? → A normalization/output step, not the prompt alone.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Convert messy documents into clean, schema-checked records for downstream systems. (See Task 1.4 §12.)

**The key link to everything above:** **Core fit.** This scenario lives on `PostToolUse`. Extraction tools return heterogeneous, sometimes-malformed data; a `PostToolUse` hook normalizes formats (dates, codes, units) and runs schema validation, replacing the result with a clean record (`updatedToolOutput`) or flagging failures — *before* the model or downstream sees anything. Deterministic data hygiene is the §0 spine applied to data.

### 13a. Normalize heterogeneous formats
The textbook §4 case: Unix vs ISO timestamps and numeric status codes from different tools, all converted to one canonical shape in a `PostToolUse` hook so the agent reasons on uniform data.

### 13b. Validate and gate
After normalization, validate against the JSON schema in the same/adjacent hook; on failure, don't emit — flag for retry/human (graceful edge-case handling = a coded failure path).

### 13c. Scenario-6 self-check
1. Which hook normalizes tool outputs? → `PostToolUse`.
2. Field to replace the result the model sees? → `updatedToolOutput` (not the deprecated MCP-only field).
3. Why normalize in a hook not the prompt? → Guaranteed (deterministic) clean data; prompts are probabilistic.
4. (Goal tie) How does this serve "high accuracy"? → The model and downstream only ever see uniform, schema-valid records, removing a whole class of reasoning errors.

---

### Sources verified against current Anthropic docs (June 2026)
- *Control agent behavior with hooks* (platform.claude.com, code.claude.com) — events incl. `PreToolUse`/`PostToolUse`/`Stop`/`SubagentStart`; `PreToolUse` → `hookSpecificOutput.permissionDecision` (`allow`/`deny`/`ask`), `permissionDecisionReason`, `updatedInput`, `additionalContext`; `PostToolUse` → `updatedToolOutput` (any tool; `updatedMCPToolOutput` MCP-only, **deprecated**) and `additionalContext`; precedence **deny > ask > allow**; eval order hooks → deny rules → permission mode → `can_use_tool`; `matcher` is a regex on tool name; MCP tools named `mcp__<server>__<tool>`.
- *Python/TypeScript SDK reference* — `ClaudeAgentOptions(hooks={...})`, `HookMatcher(matcher=..., hooks=[...])`; lifecycle hooks (`SubagentStart`, `Stop`) use the top-level `decision:"block"` form.
- Hooks ship fast (e.g., `additionalContext` was extended to `PreToolUse` in v2.1.9; some events are TypeScript-only) — re-verify field/event names against current docs before production code.
