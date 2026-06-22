# Task 5.2 — Escalation & Ambiguity Resolution Patterns
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **Escalate on the right triggers — the customer asks for a human, the policy is silent or ambiguous on their request, or you genuinely can't make progress — not on vibes. Sentiment and self-reported confidence are bad proxies for complexity. Honor an explicit demand for a human immediately; offer to resolve a straightforward issue while acknowledging frustration; and when a lookup returns multiple matches, ask for another identifier rather than guessing.**

Everything here is: escalate on real triggers (not sentiment/confidence), respect explicit human requests, and resolve ambiguity by *asking*, not guessing.

---

## 1. The right escalation triggers

Escalate when one of these is true:

| Trigger | Example |
|---|---|
| **Customer asks for a human** | "I want to speak to a person" |
| **Policy exception or gap** | The request isn't covered by policy (not merely "this is hard") |
| **Can't make meaningful progress** | Repeated dead-ends, missing data the agent can't obtain |

Note the second one: escalate on **policy gaps**, *not just complex cases*. A complex request the policy clearly covers can be resolved; a simple request the policy is *silent* on should escalate. Complexity ≠ escalation; policy coverage does.

---

## 2. Explicit human request → escalate immediately

If a customer **explicitly demands a human**, hand off **immediately** — don't first attempt investigation, don't try to resolve it anyway. Respecting that request is the correct behavior; making them argue past the bot erodes trust. (Contrast §3: mere *frustration* is not an explicit demand.)

---

## 3. Frustration → acknowledge and offer, escalate only if reiterated

When the issue is **within the agent's capability** but the customer is frustrated, the move is: **acknowledge the frustration AND offer to resolve it**. Only escalate if the customer **reiterates** their preference for a human. Frustration alone isn't an escalation trigger — many frustrated customers are happiest with a fast resolution. Don't bounce them to a queue when you can fix it now.

---

## 4. Sentiment and confidence are bad proxies

Two tempting-but-wrong escalation signals:

- **Sentiment-based escalation** ("they seem angry → escalate") — anger doesn't equal complexity; plenty of angry customers have simple, resolvable issues.
- **Self-reported confidence scores** ("model says 60% confident → escalate") — the model's confidence doesn't reliably track real case complexity (cf. Task 4.1).

Both are unreliable proxies. Escalate on the **actual triggers** (§1), not on emotional tone or a confidence number.

---

## 5. Multiple matches → ask, don't guess

When a tool result returns **multiple customer matches** (two people named "John Smith"), the agent must **request an additional identifier** (email, order number, ZIP) — **not** pick one by heuristic ("probably the most recent"). Guessing risks acting on the wrong account (exposing data, refunding the wrong person). Ambiguity in *identity* is resolved by asking, full stop.

**Implementation:** add explicit escalation criteria with **few-shot examples** (Task 4.2) to the system prompt showing when to escalate vs resolve, and instruct the agent to ask for more identifiers on multiple matches.

---

## 6. Anti-patterns (with *why*)
- **Escalating on complexity alone.** A complex but policy-covered case is resolvable; escalate on policy *gaps*, not difficulty.
- **Investigating before honoring an explicit human request.** Disrespects the customer; hand off immediately.
- **Escalating any frustrated customer.** Frustration ≠ wanting a human; acknowledge + offer, escalate only if reiterated.
- **Sentiment- or confidence-triggered escalation.** Unreliable proxies for complexity; use the real triggers.
- **Heuristically picking among multiple matches.** Risks the wrong account; ask for another identifier.
- **No few-shot escalation examples.** Vague criteria → inconsistent escalation; show examples.

---

## 7. Self-check (core mechanics)
1. Three escalation triggers? → Explicit human request, policy exception/gap, can't make meaningful progress.
2. Escalate on complexity? → No — on policy *gaps*, not mere difficulty.
3. Customer explicitly demands a human? → Escalate immediately, no investigation first.
4. Customer is frustrated but the issue is fixable? → Acknowledge + offer to resolve; escalate only if they reiterate.
5. Why not escalate on sentiment/confidence? → Both are unreliable proxies for actual complexity.
6. Tool returns multiple customer matches? → Ask for an additional identifier; never guess.
7. (Ties to §0) One sentence? → *Escalate on real triggers, honor explicit human requests, and resolve ambiguity by asking.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Core fit — the home scenario.** Every 5.2 idea lands directly: honor "I want a human" immediately (§2); acknowledge-and-offer for a frustrated-but-fixable issue (§3); escalate on a **policy gap** like competitor price-matching when policy only covers own-site adjustments (§1); ask for another identifier when `get_customer` returns multiple matches (§5); and add few-shot escalation examples to the prompt.

### 8a. The competitor-price-match policy gap
Policy addresses only own-site price adjustments and is silent on competitor matching → escalate (policy gap), don't improvise a decision.

### 8b. Multiple matches on `get_customer`
Two matching customers → ask for email/order #, never pick the "likely" one (wrong-account risk).

### 8c. Frustrated but fixable
Acknowledge the frustration, offer the fix; escalate only if they insist on a human.

### 8d. Scenario-1 self-check
1. "Connect me to a person" — what do you do? → Escalate immediately, no investigation.
2. Competitor price-match, policy silent? → Escalate (policy gap).
3. `get_customer` returns two matches? → Ask for another identifier.
4. Frustrated customer, simple fixable issue? → Acknowledge + offer; escalate only if reiterated.
5. (Goal tie) How does this serve 80% FCR *and* good escalation? → Resolve the resolvable (incl. frustrated cases), escalate only true triggers — high resolution without mis-escalating.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Partial fit.** "Escalation" maps to handing off to a human developer. The §1 triggers translate: escalate on **can't make progress** (repeated test failures it can't resolve) or **ambiguous requirements** (the spec is silent), not on a confidence number. Ask the developer to clarify ambiguous specs rather than guessing (the §5 ask-don't-guess principle).

### 9a. Clarify ambiguous specs
Underspecified requirement → ask the developer, don't guess an interpretation.

### 9b. Scenario-2 self-check
1. Spec is ambiguous? → Ask for clarification (don't guess).
2. Escalate on a low confidence score? → No; escalate on inability to progress / true gaps.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Partial fit.** Ambiguity resolution maps to a vague research question: ask the user to clarify scope rather than guessing (§5). "Escalate" to the human when sources can't answer (a §1 can't-make-progress analogue). Don't pick one interpretation of an ambiguous query by heuristic.

### 10a. Clarify an ambiguous research question
Vague scope → ask the user to narrow it, don't assume.

### 10b. Scenario-3 self-check
1. Research question is ambiguous? → Ask to clarify scope.
2. Sources can't answer? → Surface the gap (can't-make-progress).

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Partial fit.** The §5 ask-don't-guess principle is the main carry-over: when a request is ambiguous (which `User` class, which environment) or multiple matches exist (three files named `config.py`), ask which one rather than guessing — guessing in code edits causes wrong-file changes, the dev analogue of the wrong-account risk.

### 11a. Ask on ambiguous matches
Three `config.py` files → ask which; don't edit the "likely" one.

### 11b. Scenario-4 self-check
1. Multiple files match the reference? → Ask which (don't guess) — wrong-file edits are costly.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Partial fit.** Headless CI has no human to escalate to mid-run, but the *routing* analogue applies: flag genuinely ambiguous findings for human review rather than auto-deciding (cf. Task 4.6 confidence routing). Don't let a sentiment/confidence proxy gate the merge; use real criteria (Task 4.1).

### 12a. Route ambiguous findings to humans
Uncertain findings → human review queue, not an auto-block; don't guess.

### 12b. Scenario-5 self-check
1. Genuinely ambiguous finding in CI? → Route to human review, don't auto-decide.
2. Gate on a confidence proxy? → No; use explicit criteria (Task 4.1).

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Strong fit for the ask/route principle.** When a document is ambiguous or contradictory (two different totals), the move is to **route to human review** rather than guess a value (§5, and Task 5.5). Don't let the model pick arbitrarily — flag for human resolution, the extraction analogue of "ask for another identifier."

### 13a. Route ambiguous/contradictory docs to humans
Conflicting values or unclear fields → human review, not a silent guess.

### 13b. Scenario-6 self-check
1. Document gives two different totals? → Route to human review (don't pick arbitrarily).
2. (Goal tie) How does this serve accuracy? → Genuine ambiguity goes to a human instead of becoming a confident wrong value.

---

### Sources verified against current Anthropic guidance (June 2026)
- *Agent design / support-agent patterns* — escalate on explicit human requests, policy gaps, and inability to progress (not on complexity, sentiment, or self-reported confidence); resolve identity ambiguity by requesting more identifiers; few-shot escalation examples (Task 4.2) improve consistency. Self-reported confidence is an unreliable complexity proxy (consistent with Task 4.1).
- These are interaction-design patterns from the task statement, not versioned APIs. No fast-moving surface; verify any tool references against current docs.
