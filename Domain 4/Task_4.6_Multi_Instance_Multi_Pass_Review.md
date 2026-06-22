# Task 4.6 — Multi-Instance & Multi-Pass Review Architectures
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–4 are the mechanics. Section 5 = anti-patterns, Section 6 = self-check. Then one section per scenario (7–12) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **A model that just generated something is a poor reviewer of it — it carries the reasoning that produced it and won't question its own assumptions. A *fresh, independent* instance with no prior context catches subtle issues that self-review (and even extended thinking) miss. And for large reviews, split the work into per-file local passes plus a separate cross-file integration pass so attention isn't diluted.**

Everything here is: separate the reviewer from the generator, and separate local review from cross-cutting review.

---

## 1. Why self-review is weak

When the same session reviews what it just generated, it **retains the reasoning context** from generation. That context is exactly the blind spot: the model already "decided" the code is right, so it's **less likely to question its own decisions**. Asking "now review your work" in the same session mostly produces confirmation, not scrutiny. The flaw is structural, not a matter of trying harder.

---

## 2. Independent instances beat self-review (and extended thinking)

A **second, independent Claude instance** — one that never saw the generation reasoning, only the artifact — reviews with fresh eyes and catches subtle issues the generator's session would wave through. This is **more effective than self-review instructions** ("now critique yourself") **and more effective than extended thinking** in the same session, because the problem is the inherited context, not the amount of thinking. Fresh context = genuine second opinion. (This is the inverse of Task 1.7's resume: here you deliberately *don't* carry context; cf. Task 3.6's CI session isolation.)

---

## 3. Multi-pass review for large reviews

A single pass over a big multi-file change suffers **attention dilution** — spread across 40 files, the model loses detail and can produce **contradictory findings**. Split it:

| Pass | Scope | Catches |
|---|---|---|
| **Per-file local passes** | One file at a time, full focus | Local bugs, style, correctness within the file |
| **Cross-file integration pass** | The whole change together | Data-flow issues spanning files: a function signature changed in A breaking callers in C |

Per-file passes get full attention on local issues; the integration pass catches what only appears *between* files. (This is Task 1.6's per-file→cross-file decomposition applied to review.)

---

## 4. Confidence-calibrated verification passes

A verification pass can have the model **self-report a confidence level alongside each finding**, enabling **calibrated routing**: high-confidence findings post automatically, low-confidence ones route to a human or a second pass. This doesn't replace independent review (confidence is self-assessed — cf. Task 4.1's caution), but as a *routing* signal on top of independent review it helps triage where to spend human attention.

---

## 5. Anti-patterns (with *why*)
- **"Now review your own work" in the same session.** Retained reasoning → confirmation, not scrutiny; use a fresh instance.
- **Relying on extended thinking to self-catch.** More thinking in the same context doesn't escape the blind spot; use an independent instance.
- **One pass over a 40-file change.** Attention dilution → missed detail + contradictory findings; split per-file + integration.
- **Only per-file passes.** Misses cross-file data-flow breakages; add the integration pass.
- **Only an integration pass.** Misses local detail; you need the per-file passes too.
- **Treating self-reported confidence as truth.** It's a routing hint, not a guarantee (Task 4.1).

---

## 6. Self-check (core mechanics)
1. Why is self-review weak? → The generator retains its reasoning context and won't question its own decisions.
2. What beats self-review and extended thinking? → A fresh **independent** instance with no prior reasoning context.
3. Why split a large review? → A single pass dilutes attention and yields contradictory findings.
4. The two pass types? → Per-file **local** passes + a **cross-file integration** pass.
5. What does the integration pass catch? → Cross-file data-flow issues (changed signature breaking callers elsewhere).
6. Use of self-reported confidence? → A **routing** signal (auto-post high, escalate low) — not a correctness guarantee.
7. (Ties to §0) One sentence? → *Review with fresh eyes, and review local and cross-cutting separately.*

---

## 7. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Partial fit.** Review architecture applies to evaluating the agent: a **second independent instance** judges whether a resolution was correct better than asking the resolving session to grade itself (which would defend its own decision). Confidence-calibrated routing can send borderline resolutions to human QA.

### 7a. Independent eval of resolutions
A fresh instance reviews transcripts without the resolving session's reasoning — fewer rubber-stamps.

### 7b. Scenario-1 self-check
1. Grade resolutions with the resolving session? → No; use an independent reviewer.
2. Route borderline cases? → Confidence-calibrated routing to human QA.

---

## 8. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Core fit.** The generator should **not** review its own output (§1) — spin up a **second independent instance** to review the generated code with no access to the generator's reasoning (cf. Task 3.6 §5). For a large generated change, apply per-file + integration passes (§3).

### 8a. Independent reviewer, not the generator
A fresh instance reviews the generated code; it questions assumptions the generator baked in.

### 8b. Scenario-2 self-check
1. Who reviews generated code? → A fresh independent instance, not the generator.
2. Large multi-file generation? → Per-file passes + integration pass.

---

## 9. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Strong fit.** A **fresh fact-checking instance** that didn't do the synthesis verifies claims/citations better than asking the synthesizer to check itself (it would defend its narrative). Confidence-calibrated findings route weak claims for another look.

### 9a. Independent fact-check pass
A separate instance verifies the synthesized answer's claims against sources, free of the synthesizer's reasoning.

### 9b. Scenario-3 self-check
1. Verify the synthesized answer with the synthesizing session? → No; use an independent checker.
2. Route weakly-supported claims? → Confidence-calibrated routing.

---

## 10. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Strong fit.** After the assistant generates code, a **second independent instance** reviews it (don't let the author self-approve, §1). Large refactors get per-file local passes plus a cross-file integration pass (§3) to catch signature/data-flow breakages across the repo.

### 10a. Independent review + multi-pass
Fresh reviewer for generated changes; split big refactors into per-file + integration passes.

### 10b. Scenario-4 self-check
1. Self-review a big refactor? → No; independent instance + multi-pass.
2. Catch a changed signature breaking far-away callers? → The cross-file integration pass.
3. (Goal tie) How does this aid productivity? → Catches subtle and cross-file bugs the author's session would miss, before they ship.

---

## 11. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Core fit — the home scenario.** Every 4.6 idea lands: the CI reviewer is a **separate instance** from whatever generated the change (§2, Task 3.6 §5); large PRs get **per-file local passes + a cross-file integration pass** (§3) to avoid attention dilution and contradictory findings; and the reviewer can **self-report confidence per finding** so high-confidence issues post automatically while low-confidence ones route to a human (§4) — which directly helps "minimize false positives."

### 11a. Independent reviewer
The reviewer instance is distinct from the generator; fresh eyes catch what self-review wouldn't.

### 11b. Per-file + integration passes
Review each changed file locally, then one pass over the whole diff for cross-file issues — fewer missed and fewer contradictory findings.

### 11c. Confidence-calibrated routing
Each finding carries a confidence; auto-post high-confidence, route low-confidence to a human — cuts noisy auto-comments.

### 11d. Scenario-5 self-check
1. Should the reviewer be the generator? → No; an independent instance.
2. Large PR review structure? → Per-file local + cross-file integration passes.
3. Reduce noisy auto-comments? → Confidence-calibrated routing (auto-post high, escalate low).
4. (Goal tie) How does this minimize false positives? → Independent review + integration pass + confidence routing keep only solid findings automatic.

---

## 12. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Strong fit.** A **second independent instance** can verify extracted records against the source rather than the extracting session checking itself (it would trust its own reading). For very large documents, per-section passes plus an integration pass mirror the multi-pass idea. Confidence per field routes low-confidence extractions to human review.

### 12a. Independent verification of records
A fresh instance checks each record against the source document — catches misreads the extractor wouldn't doubt.

### 12b. Confidence-routed fields
Low-confidence fields route to human review; high-confidence flow downstream — supports accuracy without blocking everything.

### 12c. Scenario-6 self-check
1. Verify extractions with the extracting session? → No; independent instance.
2. Route uncertain fields? → Confidence-calibrated routing to human review.
3. (Goal tie) How does this serve "high accuracy"? → Fresh-eyes verification + confidence routing catch and triage misreads before downstream.

---

### Sources verified against current Anthropic guidance (June 2026)
- *Claude Code / agent design — review patterns* (code.claude.com, platform.claude.com) — an independent review instance (no generation context) outperforms self-review and extended thinking for catching subtle issues; this matches Task 3.6's session-isolation guidance (a fresh reviewer ≠ the generating session).
- Multi-pass review (per-file local + cross-file integration) is Task 1.6's decomposition applied to review; confidence-calibrated routing builds on the self-reported-confidence pattern (with Task 4.1's caution that confidence is a routing hint, not truth). These are architecture patterns, not versioned APIs; verify any tool/SDK specifics against current docs.
