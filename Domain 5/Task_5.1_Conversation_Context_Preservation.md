# Task 5.1 — Managing Conversation Context Across Long Interactions
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics (every "Knowledge of" / "Skills in" bullet). Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **Hard facts — amounts, dates, order numbers, what the customer said — must survive a long conversation verbatim, so pull them out of the summarizable chat history into a separate, always-included "case facts" block. Summaries blur exact values; the middle of a long input gets skipped; and verbose tool results bury the 5 fields you need under 40 you don't. Persist the facts, trim the noise, and put the important stuff where the model actually reads it (start and end).**

Everything here is: protect exact facts from summarization, fight "lost in the middle," and stop tool-output bloat.

---

## 1. Progressive summarization loses exact values

As a conversation grows, you compress old turns into summaries to save tokens. The danger: summarization **blurs precise data** — "$1,247.50 refund requested on March 3rd" becomes "customer wants a refund." Numbers, percentages, dates, statuses, and **customer-stated expectations** are exactly what you can't afford to lose, and they're exactly what vague summaries drop. So summarize *narrative*, but never let the exact facts live only in the summarizable part.

---

## 2. The "lost in the middle" effect

Models reliably attend to the **beginning** and **end** of a long input but may **omit content from the middle**. So a finding buried in the middle of a 30-section aggregated input can be effectively invisible. Mitigations: **put key findings at the beginning** of aggregated inputs, and organize the rest with **explicit section headers** so important items aren't lost in an undifferentiated middle.

---

## 3. Tool-output bloat

Tool results **accumulate in context** and consume tokens **out of proportion to their relevance** — an order lookup might return 40+ fields when only 5 matter for a return. Left unchecked, dozens of fat tool results crowd out the actual conversation. Fix: **trim verbose tool outputs to only the relevant fields** *before* they enter context (keep the return-relevant fields, drop the rest). (Related: context editing can clear stale tool results, but trimming at the source is the first move.)

---

## 4. Persisting facts in a separate layer ("case facts")

The core skill: extract **transactional facts** (amounts, dates, order numbers, statuses) into a persistent **"case facts" block** that is included in **every** prompt, *outside* the summarized history. The summary can compress the chat; the facts block stays verbatim and always present. For **multi-issue** sessions, persist structured issue data (per-issue order IDs, amounts, statuses) in its own **context layer** so the three problems don't bleed together.

```
[CASE FACTS]  (always included verbatim)
order: #48217  | refund requested: $1,247.50 | date: 2026-03-03
status: shipped | customer expectation: full refund by Friday
[/CASE FACTS]
...summarized conversation history below...
```

---

## 5. Full history + structured upstream outputs

- **Pass complete conversation history** on each API request — the API is stateless, so coherence depends on you re-sending the history (your facts block + summarized older turns + recent verbatim turns). Drop it and the model loses the thread.
- **Make upstream agents return structure, not prose.** When a downstream agent has a limited context budget, have upstream agents/subagents return **structured data** (key facts, citations, relevance scores, dates, source locations, methodological context) instead of verbose content and reasoning chains. Structured, trimmed inputs preserve the budget and support accurate synthesis.

---

## 6. Anti-patterns (with *why*)
- **Summarizing exact values away.** "$1,247.50" → "a refund"; you lose what you most need. Keep facts in a verbatim block.
- **Burying key findings in the middle.** Lost-in-the-middle hides them; front-load and use headers.
- **Letting full tool outputs accumulate.** 40 fields × many calls crowds out context; trim to relevant fields.
- **Relying on the summary for facts.** Facts belong in a separate always-included layer, not the compressible summary.
- **Not re-sending history.** The API is stateless; omitting history breaks coherence.
- **Verbose upstream output to a budget-limited downstream.** Reasoning chains blow the budget; return structured key facts.

---

## 7. Self-check (core mechanics)
1. What does progressive summarization risk losing? → Exact numbers, dates, statuses, customer-stated expectations.
2. The "lost in the middle" effect? → Models attend to start/end but may omit the middle; front-load key findings + use headers.
3. Why trim tool outputs? → They accumulate and consume tokens disproportionate to relevance (40 fields vs 5 needed).
4. Where do transactional facts live? → A persistent "case facts" block included in every prompt, outside the summary.
5. Multi-issue sessions? → A separate context layer with per-issue structured data.
6. Why re-send full history? → The API is stateless; coherence depends on it.
7. (Ties to §0) One sentence? → *Persist exact facts, trim the noise, and put the important parts where the model reads them.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Core fit — the home scenario.** Every 5.1 idea lands: extract the refund amount, order number, dates, and the customer's stated expectation into a **case facts block** (§4) so a long dispute doesn't summarize "$1,247.50 by Friday" into "wants a refund"; **trim `lookup_order`** from 40+ fields to the ~5 return-relevant ones (§3); use a **separate issue layer** for multi-complaint tickets (§4).

### 8a. Case facts block
Persist amount, order #, dates, status, and expectation verbatim in every prompt — protected from summarization.

### 8b. Trim `lookup_order`
Keep only return-relevant fields before the result enters context; drop the other 35.

### 8c. Multi-issue layer
Three complaints → three structured issue records in a separate layer so they don't blur.

### 8d. Scenario-1 self-check
1. Stop "$1,247.50 by Friday" becoming "a refund"? → Case facts block (verbatim, always included).
2. `lookup_order` returns 40 fields? → Trim to the ~5 relevant before context.
3. Three complaints? → Separate structured issue layer.
4. (Goal tie) How does this serve 80% FCR? → Exact facts survive the whole conversation, so the agent resolves correctly without re-asking.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Partial fit.** Long generation runs accumulate verbose tool output (test logs, file dumps); trim them to the relevant lines so context isn't exhausted (§3). Persist the spec/requirements in a stable facts block so they don't get summarized away mid-run.

### 9a. Trim logs, persist the spec
Keep only failing-test lines; keep the requirements in a verbatim block.

### 9b. Scenario-2 self-check
1. Verbose test logs filling context? → Trim to relevant lines.
2. Keep requirements from being summarized away? → A persistent spec block.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Core fit.** The synthesis agent has a limited budget, so make **subagents return structured data** — key facts, citations, relevance scores, **dates and source locations, methodological context** — not verbose content and reasoning chains (§5). Front-load the most important findings and use section headers to beat lost-in-the-middle (§2) when aggregating many sources.

### 10a. Structured subagent outputs
Each subagent returns `{claim, source, date, relevance}` records, not prose dumps — preserving the synthesizer's budget.

### 10b. Beat lost-in-the-middle in aggregation
Put top findings first; header each source section so nothing in the middle is skipped.

### 10c. Scenario-3 self-check
1. Synthesis agent has limited budget — what should subagents return? → Structured key facts + citations + metadata, not verbose reasoning.
2. Many sources aggregated — avoid losing the middle? → Front-load key findings; explicit section headers.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Strong fit (overlaps Task 5.4).** Long exploration accumulates verbose file/Grep output; trim to relevant snippets (§3) and persist key discovered facts (class names, entry points) in a facts block so they survive (the scratchpad idea, expanded in Task 5.4). Front-load the architecture summary so it's not lost mid-session.

### 11a. Trim discovery output, persist findings
Keep relevant code snippets; record discovered structure in a facts/scratchpad block.

### 11b. Scenario-4 self-check
1. Verbose Grep/Read output accumulating? → Trim to relevant snippets.
2. Keep discovered class names from fading? → Persist in a facts block (see Task 5.4).

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Partial fit.** A large diff review aggregates many files; front-load the most critical findings and use per-file headers to avoid lost-in-the-middle (§2). Trim verbose diff context to the changed regions plus needed surroundings (§3).

### 12a. Order findings + trim diff context
Critical findings first; header per file; include only relevant diff regions.

### 12b. Scenario-5 self-check
1. Big diff — avoid missing middle files? → Front-load + per-file headers.
2. Reduce diff bloat? → Include only changed regions + needed context.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Strong fit.** For long documents, front-load the extraction target/instructions and section the document so key content isn't lost in the middle (§2). Persist already-extracted facts in a stable block so a long multi-page extraction doesn't summarize earlier fields away, and trim irrelevant document chrome (§3).

### 13a. Front-load instructions, persist extracted facts
Put the schema/target up front; keep extracted records in a verbatim block across pages.

### 13b. Scenario-6 self-check
1. Long doc — key content in the middle missed? → Front-load instructions; section the document.
2. Earlier extracted fields fading over a long run? → Persist them in a facts block.
3. (Goal tie) How does this serve accuracy? → Exact extracted values survive and important content isn't position-lost.

---

### Sources verified against current Anthropic guidance (June 2026)
- *Context management* (platform.claude.com, code.claude.com) — context editing can clear stale tool results/thinking; long-context guidance recommends placing key information at the start/end and using clear structure (the "lost in the middle" mitigation); the Messages API is stateless, so full history must be re-sent for coherence.
- Progressive-summarization risk, the "case facts" block, tool-output trimming, multi-issue context layers, and structured upstream outputs come from the task statement and align with documented long-context best practices. These are architecture patterns; verify any specific context-editing/API fields against current docs.
