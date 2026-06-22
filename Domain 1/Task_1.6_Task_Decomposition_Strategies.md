# Task 1.6 — Task Decomposition Strategies for Complex Workflows
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–4 are the mechanics. Section 5 = anti-patterns, Section 6 = self-check. Then one section per scenario (7–12) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **Match the breakdown to how predictable the work is. If you already know the steps, use a *fixed sequential pipeline* (prompt chaining) — step 1 feeds step 2 feeds step 3. If you only discover the steps as you go, use *dynamic adaptive decomposition* — generate the next subtasks from what you just learned. Predictable shape → fixed chain. Open-ended discovery → adaptive plan.**

Everything in this task is choosing where you sit on that spectrum, and one universal move within it: when a single input is too big for the model to handle well in one pass, split it (e.g., per-file, then a cross-file pass) so attention isn't diluted.

---

## 1. Why decompose at all (the attention problem)

A model has a finite context window and finite attention. Cram a 40-file review or a 100-page document into one prompt and quality drops — the middle gets lost, details blur. Decomposition fixes this two ways: **partition** the work into pieces small enough to handle well, and **sequence/structure** the pieces so nothing important falls between them. The cost is more model calls and orchestration; the benefit is accuracy on work too big for one pass.

---

## 2. Fixed sequential pipelines (prompt chaining)

**Prompt chaining** = a fixed series of steps where each step's output is the next step's input. You use it when the **shape of the work is known in advance**.

The canonical pattern for reviews: **analyze each file individually, then run a cross-file integration pass.**

```
[file A]→analyze─┐
[file B]→analyze─┼─→ [cross-file pass: find issues spanning files] → report
[file C]→analyze─┘
```

- **Per-file step** catches local issues without attention dilution (each file gets full focus).
- **Cross-file step** catches what no single file shows: a function renamed in A but still called in C, an interface changed in B that breaks its caller in A. Skipping this pass is the classic failure of naive per-file review.

You know up front it's "per file, then integrate," so it's a *fixed* chain — no runtime improvisation needed.

---

## 3. Dynamic adaptive decomposition

When you **can't know the steps ahead of time**, the agent generates subtasks based on intermediate findings. It investigates, sees what it found, and *then* decides what to do next.

Example: "Add comprehensive tests to a legacy codebase." You can't pre-write the plan, so the agent: **maps** the codebase → identifies **high-impact, low-coverage** areas → produces a **prioritized plan** → and revises that plan as each area reveals new dependencies. Each step's discovery shapes the next. This is the orchestrator-workers / adaptive-investigation pattern.

---

## 4. Choosing between them (the decision the exam rewards)

| Question | Fixed chain | Adaptive |
|---|---|---|
| Do you know the steps in advance? | Yes | No |
| Is the input shape predictable? | Yes (e.g., "review these N files") | No (e.g., "understand this unknown system") |
| Does each step depend on what the last *discovered*? | No | Yes |
| Examples | Per-file → cross-file review; extract → validate → load | Legacy exploration; open-ended research; "add tests to unknown code" |

Many real systems are **hybrid**: a fixed outer chain with an adaptive inner step (e.g., fixed "map → plan → execute," where "plan" is generated adaptively).

---

## 5. Anti-patterns (with *why*)
- **Per-file review with no cross-file pass.** Misses every issue that spans files — the most dangerous bugs. Always add the integration step.
- **Forcing a fixed pipeline onto open-ended work.** A rigid plan can't absorb what you discover in legacy code; you'll either miss things or thrash. Go adaptive.
- **Over-decomposing.** Splitting so finely that pieces lose the context they need to be correct (Task 1.2's "overly narrow decomposition"). Pieces must stay self-contained.
- **One giant prompt for huge input.** Attention dilution → lost details. Partition first.
- **Adaptive when fixed would do.** Improvising steps for predictable work adds cost and nondeterminism for nothing.

---

## 6. Self-check (core mechanics)
1. Predictable steps → which strategy? → **Fixed sequential pipeline (prompt chaining).**
2. Steps discovered as you go → which? → **Dynamic adaptive decomposition.**
3. The canonical review pattern? → Analyze each file individually, **then a cross-file integration pass.**
4. Why is the cross-file pass essential? → It catches issues that span files (renamed function still called elsewhere) that per-file review can't see.
5. How do you decompose "add tests to a legacy codebase"? → Map → find high-impact/low-coverage areas → prioritized **adaptive** plan that revises with findings.
6. (Ties to §0) One sentence? → *Fixed chain when you know the steps; adaptive plan when you must discover them.*

---

## 7. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** An agent resolving support cases via four backend tools (see Task 1.4 §7 for the full term breakdown).

**The key link to everything above:** **Partial-to-strong fit.** A multi-issue dispute is a decomposition problem: split "refund missing + double charge + login broken" into distinct items. Within an item the steps are fairly predictable (`get_customer` → `lookup_order` → decide), so each item is a **short fixed chain**; deciding *how many* items and handling surprises is mildly **adaptive**. The per-item investigation can run in parallel (Task 1.3).

### 7a. Decompose the multi-concern request
Three complaints → three sub-investigations, each a small predictable chain, then synthesize one resolution. Don't pour all three into one tangled pass.

### 7b. Where it stays adaptive
High-ambiguity means you sometimes discover a fourth hidden issue mid-investigation; the plan must accommodate adding an item rather than being frozen.

### 7c. Scenario-1 self-check
1. How do you handle a 3-complaint message? → Decompose into items, investigate each, unify.
2. Fixed or adaptive per item? → Mostly **fixed** (predictable tool order); the *splitting* is lightly adaptive.
3. Why not one big pass? → Attention dilution; you'll resolve one issue and drop others, hurting FCR.

---

## 8. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent that produces code in known stages (see Task 1.4 §8).

**The key link to everything above:** **Strong fit — the textbook fixed chain.** "Plan → write → test → refine" is predictable, so it's prompt chaining: each step consumes the previous step's output. The only adaptive wrinkle is the refine loop (re-plan based on test failures), which is a bounded inner adaptation inside a fixed outer pipeline.

### 8a. The fixed chain
plan → generate → run tests → (pass: done / fail: feed failures back to a fix step). The shape is known up front; that's why it's a chain, not an open-ended plan.

### 8b. Scenario-2 self-check
1. Fixed or adaptive overall? → **Fixed** prompt chain.
2. What feeds the fix step? → The **test results** from the prior step (chaining).
3. Where's the bit of adaptivity? → Re-planning from failures inside the loop.

---

## 9. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research, then synthesizing (see Task 1.4 §9).

**The key link to everything above:** **Strong fit — adaptive end of the spectrum.** Research is open-ended: you don't know the sub-questions until initial findings appear. So the coordinator generates subtasks **adaptively** (find a surprising claim → spawn a subagent to verify it), then a fixed final synthesis step combines results. Classic orchestrator-workers with an adaptive investigation plan.

### 9a. Adaptive subtask generation
Initial searches reveal which threads need deeper digging; the coordinator spawns subagents per discovered thread rather than from a frozen list.

### 9b. Fixed tail
The synthesis pass is predictable ("combine + cite"), so the workflow is adaptive-middle, fixed-end — a hybrid.

### 9c. Scenario-3 self-check
1. Why adaptive here? → Sub-questions are discovered, not known in advance.
2. Which part is fixed? → The final synthesis/citation step.
3. Pattern name? → Orchestrator-workers / adaptive investigation plan.

---

## 10. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Bash, Grep, Glob; integrates MCP; explores codebases, handles legacy systems, generates boilerplate, automates chores.*

**What this scenario is even about (plain English):** A coding assistant working across a real repo (see Task 1.4 §10). "Legacy" = old code nobody fully understands; "coverage" = how much of the code the tests exercise.

**The key link to everything above:** **Core fit — the marquee adaptive example.** "Add comprehensive tests to a legacy codebase" is §3 verbatim: you can't pre-plan it. The agent **maps** the codebase (Grep/Glob/Read) → finds **high-impact, low-coverage** areas → builds a **prioritized adaptive plan** → revises as each area exposes new dependencies. Meanwhile, big multi-file reviews use the §2 **per-file → cross-file** fixed chain. So this one scenario shows *both* ends of the spectrum.

### 10a. Adaptive: legacy test coverage
Map → prioritize high-impact/low-coverage → generate tests → discover a dependency → re-prioritize. The plan is grown from findings, not fixed.

### 10b. Fixed: large code review
For "review these 30 changed files," use the predictable per-file-then-cross-file chain — you know the shape, so don't improvise.

### 10c. The judgment call
The exam wants you to pick correctly: open-ended legacy work → adaptive; predictable multi-file review → fixed chain. Same agent, different strategy by predictability.

### 10d. Scenario-4 self-check
1. Strategy for "add tests to legacy code"? → **Adaptive** (map → high-impact → prioritized plan).
2. Strategy for "review these 30 files"? → **Fixed** per-file → cross-file chain.
3. What decides which? → Whether the steps are **predictable** or **discovered**.
4. (Goal tie) Why does the cross-file pass matter for a real review? → It catches cross-cutting breakages a per-file pass misses — the bugs that actually ship.

---

## 11. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in your pipeline (see Task 1.4 §11). A PR usually changes several files at once.

**The key link to everything above:** **Core fit — the canonical fixed-chain example.** A PR review is predictable in shape: analyze each changed file, then a cross-file integration pass to catch breakages spanning the diff, then emit one JSON verdict. That's §2 prompt chaining exactly. It's *not* adaptive — the steps are the same on every PR, which is what makes it reliable and low-false-positive.

### 11a. Per-file → cross-file on the diff
Review each changed file in isolation (full focus, fewer false positives), then one pass over the whole diff for cross-file issues (a signature changed in one file, callers in another).

### 11b. Why fixed beats adaptive here
Determinism is a feature in CI: the same PR should review the same way every time. A fixed chain gives that; an adaptive plan would add variance you don't want gating merges.

### 11c. Scenario-5 self-check
1. Decomposition pattern for PR review? → **Fixed** per-file → cross-file chain.
2. Why fixed, not adaptive? → Predictable shape + you want deterministic, repeatable reviews.
3. What does the cross-file pass catch? → Breakages spanning the diff (e.g., changed signature + its callers).

---

## 12. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Task 1.4 §12).

**The key link to everything above:** **Strong fit — fixed pipeline.** The shape is known: **extract → normalize → validate → (pass: load downstream / fail: retry-or-flag)**. That's prompt chaining. The only adaptivity is a per-document branch for edge cases (a malformed doc triggers a retry/repair sub-step), a bounded adaptation inside the fixed chain. For very long documents, partition by section (the §1 attention move) and run the chain per section.

### 12a. The fixed extraction chain
extract → normalize formats → schema-validate → route. Each step feeds the next; you know all the steps in advance.

### 12b. Partition long inputs
A 100-page document dilutes attention in one pass → split by section, extract per section, then reconcile — the §2 per-piece-then-integrate idea applied to documents.

### 12c. Scenario-6 self-check
1. Fixed or adaptive? → **Fixed** chain (extract → validate → load).
2. How to handle a 100-page doc? → **Partition** by section, then reconcile.
3. Where's the small adaptivity? → Per-document retry/repair on edge cases.
4. (Goal tie) How does decomposition support "high accuracy"? → Smaller per-section passes avoid attention dilution, so extraction stays precise.

---

### Sources verified against current Anthropic docs (June 2026)
- *Agent SDK overview / building agents* and Anthropic's "Building effective agents" guidance — the canonical patterns (prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer) and the predictability-driven choice between fixed pipelines and adaptive decomposition.
- *Subagents in the SDK* — adaptive subtask generation maps onto spawning workers per discovered thread (mechanics in Task 1.3); attention/context limits motivate partitioning.
- Decomposition is a design pattern (not a single API), so this guide is concept-led; verify any SDK specifics (tool names, subagent config) against current docs when you implement.
