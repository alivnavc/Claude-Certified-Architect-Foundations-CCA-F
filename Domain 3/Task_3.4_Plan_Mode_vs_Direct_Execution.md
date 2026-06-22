# Task 3.4 — Plan Mode vs Direct Execution
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–4 are the mechanics. Section 5 = anti-patterns, Section 6 = self-check. Then one section per scenario (7–12) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **Plan first when the cost of going the wrong way is high — big multi-file changes, architectural decisions, several valid approaches. Just execute when the change is small and the path is obvious — a one-line fix with a clear stack trace. And when discovery is verbose, push it to the Explore subagent so all that reading doesn't eat your main context.**

Everything here is a single judgment — *how much could going wrong cost?* — plus one context-saving tool (Explore) and one combined workflow (plan to decide, execute to do).

---

## 1. What plan mode is

**Plan mode** lets Claude explore the codebase and design an approach **without making any changes** — it reads, reasons, and proposes a plan you approve before any edit happens. Enter it with `/plan` or toggle with Shift+Tab. It exists to make exploration and design **safe**: you commit to an approach only after seeing it, preventing costly rework from charging ahead on the wrong design. (`Opusplan` uses the stronger model for planning and a faster one for execution.)

---

## 2. When to plan (high cost-of-wrong)

Choose plan mode when the task has any of:

| Signal | Why plan |
|---|---|
| **Large-scale change** (many files) | A wrong approach means redoing dozens of files |
| **Multiple valid approaches** | You need to weigh trade-offs before committing |
| **Architectural decisions** | Hard to reverse; design first |
| **Multi-file / cross-cutting** | Coordination risk; map it before touching |

Examples: a microservice restructuring; a library migration affecting 45+ files; choosing between integration approaches with different infrastructure requirements. In all of these, exploring and deciding *before* editing is far cheaper than discovering mid-edit that the approach was wrong.

---

## 3. When to execute directly (low cost-of-wrong)

Choose direct execution when the task is **simple and well-scoped** — you already know what to change and where:

- A single-file bug fix with a clear stack trace.
- Adding one validation check (e.g., a date-validation conditional) to one function.
- Any change where the scope is obvious and contained.

Planning these adds overhead with no benefit; just do it.

---

## 4. The Explore subagent (and the combined workflow)

**Explore subagent:** a fast, **read-only** subagent (Glob, Grep, Read, safe bash like `git status`/`diff`) that skips CLAUDE.md/git status to stay lean. Use it for **verbose discovery phases** so the mountain of files it reads stays in *its* context and only a **summary** returns to your main conversation — preventing context-window exhaustion during multi-phase tasks.

**Combined pattern:** use **plan mode (with Explore) to investigate and decide**, then **direct execution to implement** the approved plan. E.g., plan a library migration (explore the 45 files, weigh approaches, produce a step list), then execute the chosen steps directly. Investigation and implementation are different phases with different modes.

---

## 5. Anti-patterns (with *why*)
- **Charging into a 45-file migration without a plan.** A wrong approach means massive rework; plan first.
- **Plan-moding a one-line fix.** Pure overhead; execute directly.
- **Doing verbose discovery in the main thread.** It exhausts your context window; delegate to Explore for a summary.
- **Planning forever, never executing.** Plan to decide, then switch to execution; don't loop.
- **Treating "plan vs execute" as a personality, not a per-task call.** It's a judgment about cost-of-wrong for *this* task.

---

## 6. Self-check (core mechanics)
1. The deciding question? → How costly is going the wrong way for *this* task?
2. Four plan-mode signals? → Large-scale change, multiple valid approaches, architectural decisions, multi-file/cross-cutting.
3. A clear single-file bug fix? → Direct execution.
4. What is plan mode's safety value? → Explore + design with **no changes** until you approve, preventing rework.
5. What does the Explore subagent prevent? → Context-window exhaustion from verbose discovery (returns a summary).
6. Combined workflow? → Plan (with Explore) to decide, then execute directly.
7. (Ties to §0) One sentence? → *Plan when wrong is expensive; execute when the path is obvious; Explore to keep discovery out of your context.*

---

## 7. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7). Plan/execute is a Claude Code workflow for the **team building** the agent.

**The key link to everything above:** **Indirect fit.** Designing the agent's tool architecture or refund-gate logic is an architectural decision → **plan mode**. Adding one field to a tool response is a contained change → **direct execution**. Same cost-of-wrong judgment.

### 7a. Plan the architecture, execute the tweaks
Plan the escalation/gate design (cross-cutting); execute small tool tweaks directly.

### 7b. Scenario-1 self-check
1. Design the refund-gate architecture? → Plan mode.
2. Add one field to a tool's output? → Direct execution.

---

## 8. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Core fit — the name says it.** The pipeline's "plan" stage *is* plan mode: for a complex feature, plan the approach before writing; for a trivial change, skip to writing. The combined workflow (§4) is exactly this pipeline — plan the design, then execute the implementation.

### 8a. Plan stage = plan mode
Complex feature → plan first (multiple approaches, multi-file). Trivial change → direct execution. The pipeline already encodes the judgment.

### 8b. Scenario-2 self-check
1. Complex multi-file feature? → Plan, then execute.
2. One-line change in the pipeline? → Execute directly, skip planning.

---

## 9. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Strong fit via Explore.** Research *is* a verbose discovery phase — the exact thing the **Explore subagent** isolates. The coordinator delegates noisy reading to Explore (read-only, returns a summary), preserving its main context for synthesis. Conceptually, Explore is the built-in version of this scenario's search subagent.

### 9a. Explore for verbose discovery
Delegate heavy reading to Explore; only the summary returns, so the coordinator's context isn't exhausted across many sources.

### 9b. Scenario-3 self-check
1. Keep verbose multi-source reading out of main context? → The **Explore** subagent (read-only, summary back).
2. Why does that matter here? → Prevents context exhaustion across many sources.

---

## 10. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Core fit — the home scenario.** All the exam's examples live here: plan mode for a **library migration affecting 45+ files**, a **microservice restructuring**, or **choosing between integration approaches**; direct execution for a **single-file bug fix with a clear stack trace** or **adding a date-validation conditional**; the **Explore subagent** for the verbose discovery of an unfamiliar/legacy codebase; and the **combined** workflow (plan the migration, then execute it).

### 10a. Plan the big migration
45-file library migration → plan mode: explore (via Explore), weigh approaches, produce a step list, approve, then execute.

### 10b. Execute the obvious fix
Clear stack trace pointing at one function → direct execution; no planning overhead.

### 10c. Explore an unfamiliar codebase
Delegate the read-heavy exploration to Explore so understanding a legacy system doesn't exhaust your main context.

### 10d. Combine the modes
Plan to decide the migration approach; switch to direct execution to implement the approved plan.

### 10e. Scenario-4 self-check
1. 45-file migration? → Plan mode (then execute).
2. Single-file fix with a clear stack trace? → Direct execution.
3. Verbose legacy-code discovery? → Explore subagent.
4. Migration end-to-end? → Plan to decide, execute to do.
5. (Goal tie) How does this aid productivity? → Effort matches stakes — no over-planning small fixes, no costly rework on big changes, no context blown on discovery.

---

## 11. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Partial fit.** CI review is non-interactive, so interactive plan mode doesn't apply at runtime — but the *judgment* informs design: building the review pipeline (multi-component, architectural) warrants planning; tweaking one prompt line is direct execution. And Explore-style read-only discovery is how the reviewer inspects a diff's blast radius cheaply.

### 11a. Plan the pipeline, execute the tweaks
Designing the CI integration → plan; adjusting a single rule → direct execution.

### 11b. Scenario-5 self-check
1. Does interactive plan mode run inside headless CI? → No; it's a design-time judgment.
2. Build the whole review pipeline? → Plan first (architectural, multi-component).

---

## 12. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Partial fit.** Designing the extraction+validation architecture (schemas, edge-case strategy, downstream contract) is a multi-component decision → **plan mode**. Adding one field to the output schema or one null-check is **direct execution**. The cost-of-wrong judgment again.

### 12a. Plan the architecture, execute the small changes
Plan the schema/validation/edge-case design; execute single-field or single-check additions directly.

### 12b. Scenario-6 self-check
1. Design the extraction + validation architecture? → Plan mode.
2. Add one null-check to a migration? → Direct execution.

---

### Sources verified against current Claude Code docs (June 2026)
- *Plan mode / headless* (code.claude.com) — plan mode (`/plan`, Shift+Tab) explores and designs **without making changes** until approved; `Opusplan` uses Opus for planning, Sonnet for execution.
- *Subagents* — the **Explore** subagent is fast, **read-only** (Glob/Grep/Read + safe bash), skips CLAUDE.md/git status to stay lean, and returns summaries to preserve main context; subagents start with clean context.
- The 45-file migration / microservice-restructuring / single-file-fix examples and the combined plan-then-execute workflow come from the task statement; re-verify mode toggles and subagent specifics against current docs.
