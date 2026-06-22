# Task 4.1 — Designing Prompts with Explicit Criteria (Precision & False Positives)
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–4 are the mechanics (every "Knowledge of" / "Skills in" bullet). Section 5 = anti-patterns, Section 6 = self-check. Then one section per scenario (7–12) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **To make a model precise, tell it *exactly which things count* — not how confident to be. "Flag a comment only when the claimed behavior contradicts the actual code" works; "check that comments are accurate" and "be conservative" do not. Specific categorical criteria beat confidence-based hedging. And one noisy category poisons trust in all the others, so define what to report, what to skip, and the bar for each severity — with concrete examples.**

Everything here is the same move: replace vague instructions and confidence dials with explicit, categorical, example-anchored criteria.

---

## 1. Explicit criteria beat vague instructions

"Check that comments are accurate" leaves the model to guess what "accurate" means, so it flags trivia and misses real problems inconsistently. The fix is a precise rule that names the exact condition: *"flag a comment only when the claimed behavior contradicts the actual code behavior."* Now there's a definite test the model applies the same way every time. **Vague → inconsistent; specific and categorical → consistent.**

---

## 2. Why "be conservative" / "only high-confidence" fail

It's tempting to improve precision by telling the model to hedge: "be conservative," "only report high-confidence findings." These **don't work**, because confidence is a feeling, not a criterion — the model's self-assessed confidence doesn't reliably track real correctness, so the instruction just adds noise without changing *which* things get flagged. What actually moves precision is naming the **categories** to report (bugs, security) versus skip (minor style, local conventions). Define the *what*, not the *how-sure*.

---

## 3. False positives destroy trust across categories

A single high-false-positive category doesn't just waste time on itself — it **undermines confidence in the accurate categories too**. Once developers learn the security warnings are usually noise, they start ignoring *all* the tool's output, including the reliable bug findings. Precision is therefore a trust problem, not just an accuracy metric: one bad category can sink the whole tool's credibility.

**Practical move:** temporarily **disable** the high-false-positive category (restoring trust in what remains) while you improve its prompt offline, then re-enable once it's reliable. Better to ship four trusted categories than five where one is noise.

---

## 4. Severity with concrete examples

"Classify severity as high/medium/low" is itself vague. Define each level with **explicit criteria and a concrete code example**:

| Severity | Criterion | Example |
|---|---|---|
| High | Exploitable / data loss / crash in prod | SQL built by string concatenation from user input |
| Medium | Bug under specific conditions | Off-by-one in a rarely-hit branch |
| Low | Works but suboptimal | Inefficient loop, no correctness impact |

Anchoring each level to an example makes classification consistent across runs and reviewers — the model matches against the example rather than re-inventing the boundary each time.

---

## 5. Anti-patterns (with *why*)
- **"Check that X is accurate."** Undefined test → inconsistent flagging; state the exact condition.
- **"Be conservative" / "only high-confidence."** Confidence isn't a criterion; name categories to report vs skip.
- **Shipping a noisy category.** It poisons trust in the good ones; disable it until fixed.
- **Severity labels with no definitions.** Inconsistent classification; define each level with criteria + a code example.
- **Filtering by self-reported confidence instead of category.** The model's confidence doesn't track correctness; filter by *what*, not *how sure*.

---

## 6. Self-check (core mechanics)
1. Vague vs explicit example? → "Check comments are accurate" (vague) vs "flag only when claimed behavior contradicts the code" (explicit).
2. Why doesn't "be conservative" improve precision? → Confidence is a feeling, not a criterion; it doesn't change which things get flagged.
3. What actually improves precision? → Specific categorical criteria (report bugs/security, skip minor style).
4. Effect of one noisy category? → It undermines trust in the accurate categories too.
5. Quick fix for a noisy category? → Disable it temporarily; improve its prompt; re-enable.
6. Consistent severity? → Define each level with criteria + a concrete code example.
7. (Ties to §0) One sentence? → *Name exactly what counts; don't dial confidence.*

---

## 7. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Strong fit.** "Escalate when unsure" is the §2 trap — vague confidence hedging. Replace it with explicit **escalation criteria**: escalate when the refund exceeds policy, when fraud indicators are present, or when required data is missing — named categories, not a feeling. This makes escalation consistent and keeps auto-resolution high.

### 7a. Categorical escalation criteria
Define the exact conditions that trigger `escalate_to_human` rather than "escalate if not confident." Consistent, auditable.

### 7b. Scenario-1 self-check
1. "Escalate when unsure" — problem? → Confidence-based; vague. Use categorical triggers.
2. (Goal tie) How does this serve 80% FCR? → Clear criteria mean the agent only escalates real edge cases and confidently resolves the rest.

---

## 8. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Partial fit.** When a verification step judges generated code, explicit criteria for "what counts as done/correct" beat "looks good." Define the categorical pass conditions (compiles, tests pass, no security smell from the named list) rather than a vague quality bar.

### 8a. Explicit done-criteria
List the categorical conditions for "complete" instead of relying on the model's judgment of quality.

### 8b. Scenario-2 self-check
1. "Looks correct" vs explicit criteria? → Name the pass conditions (tests pass, no listed smells).

---

## 9. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Partial fit.** "Only include reliable sources" is confidence hedging; replace with explicit criteria for what counts as citable (named source types, recency bounds, requires a direct quote). Categorical inclusion rules reduce false/weak citations.

### 9a. Explicit citation criteria
Define which sources qualify (peer-reviewed, official, dated within N years) instead of "trustworthy ones."

### 9b. Scenario-3 self-check
1. "Use reliable sources" — fix? → Explicit categorical criteria for citability.

---

## 10. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Strong fit.** When the assistant flags issues during exploration, vague "point out problems" yields noise. Explicit criteria — report bugs and security, skip minor style and local conventions — keep its observations trustworthy so the developer keeps listening.

### 10a. Report-vs-skip categories
Tell it exactly which issue classes to surface and which to ignore; don't leave "problem" undefined.

### 10b. Scenario-4 self-check
1. Assistant flags too much noise? → Define report-vs-skip categories explicitly.
2. (Goal tie) Why does this aid productivity? → Trustworthy, low-noise findings the developer actually acts on.

---

## 11. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Core fit — the home scenario.** Every 4.1 idea is here: the comment-accuracy example (§1), the failure of "be conservative" (§2), the trust-poisoning of a noisy category (§3 — disable the noisy "style" category temporarily to restore trust), and severity defined with concrete code examples (§4). "Minimize false positives" *is* this task.

### 11a. Explicit review criteria
"Flag a comment only when it contradicts the code"; report bugs/security, skip minor style — categorical, not "be conservative."

### 11b. Disable a noisy category to restore trust
If the security category is 70% false positives, disable it (devs trust the rest again), improve its prompt offline, re-enable when reliable.

### 11c. Example-anchored severity
Define high/medium/low with a concrete code example each so the JSON `severity` is consistent across PRs.

### 11d. Scenario-5 self-check
1. "Be conservative" reduce false positives? → No; use categorical criteria.
2. One noisy category's effect? → Devs distrust all categories; disable it temporarily.
3. Consistent severity in the JSON? → Define each level with criteria + a code example.
4. (Goal tie) How does this minimize false positives *and* keep trust? → Precise per-category criteria + pulling noisy categories until fixed.

---

## 12. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Strong fit.** Vague "extract the relevant info" yields inconsistent fields; explicit criteria for what to extract vs leave null (and what counts as a match) make extraction precise. "Only extract when confident" is the §2 trap — define the categorical rule instead ("extract the invoice total only from a line labeled Total/Amount Due").

### 12a. Explicit extraction criteria
Name exactly what qualifies for each field rather than "extract if confident."

### 12b. Scenario-6 self-check
1. "Extract if confident" — fix? → Explicit categorical rule for what counts as a valid value.
2. (Goal tie) How does this serve accuracy? → Consistent field-level criteria, not a confidence dial.

---

### Sources verified against current Anthropic guidance (June 2026)
- *Prompt engineering / tool-use precision* (docs.claude.com) — specific, example-anchored criteria outperform vague instructions and confidence-based hedging for precision; this aligns with the broader "be explicit and give examples" prompting guidance.
- The comment-contradiction example, "be conservative" failure mode, trust-poisoning across categories, temporary-disable tactic, and example-anchored severity come from the task statement. These are prompting techniques, not versioned APIs; no fast-moving surface, but verify any tool/flag references against current docs.
