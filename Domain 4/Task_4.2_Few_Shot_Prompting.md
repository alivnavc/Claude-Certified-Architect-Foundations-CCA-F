# Task 4.2 — Few-Shot Prompting for Consistency & Quality
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–4 are the mechanics. Section 5 = anti-patterns, Section 6 = self-check. Then one section per scenario (7–12) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **When detailed instructions still produce inconsistent output, *show* 2–4 worked examples. Few-shot examples are the most effective way to lock in format, demonstrate how to handle ambiguous cases (including the reasoning for the choice), and let the model generalize that judgment to new inputs it hasn't seen — rather than matching only the exact cases you listed.**

Everything here is: examples beat more instructions; show the reasoning on hard cases; pick examples that teach generalization, not memorization.

---

## 1. Few-shot is the most effective consistency lever

A **few-shot example** is a worked input→output pair you include in the prompt. When prose alone yields inconsistent results — different formats, different judgment calls across runs — adding a handful of examples is the single most effective fix. The model pattern-matches to your demonstrated format and behavior, so output becomes consistently shaped and actionable. **Instructions describe; examples demonstrate** — and demonstration sticks better.

Use **2–4** targeted examples (not dozens) — enough to establish the pattern without bloating the prompt.

---

## 2. Examples teach ambiguous-case handling (with reasoning)

The highest-value examples are the **ambiguous** ones — the cases where the right move isn't obvious. For these, show **why** one action was chosen over plausible alternatives, not just the answer:

```
Request: "My thing is broken and I want my money back"
Reasoning: Ambiguous between a refund and a technical issue. Resolve the
  technical problem first (it may remove the need for a refund); only refund
  if unresolved. → call lookup_order, then diagnose, before process_refund.
```

Showing the reasoning teaches the *decision procedure*, so the model applies the same logic to new ambiguous inputs — far more useful than an answer with no rationale.

---

## 3. Generalization, not memorization

Good few-shot examples make the model **generalize judgment to novel patterns**, not just reproduce the listed cases. Pick examples that illustrate the *principle* (e.g., "a comment that contradicts code is a bug; a comment that's merely terse is fine") so the model handles unseen variations correctly. Examples that are too narrow teach only those exact cases; examples chosen to show the boundary teach the rule.

---

## 4. Few-shot reduces hallucination in extraction

In extraction, examples are the antidote to **fabrication** and **empty/null** errors. Showing correct handling of messy real inputs — informal measurements ("a couple tablespoons"), varied document structures (inline citations vs a bibliography; a methodology section vs details embedded in prose) — teaches the model what to do with formats that otherwise trip it into guessing or returning empty required fields. Concretely:

- Examples distinguishing **acceptable patterns from genuine issues** → fewer false positives, while still generalizing.
- Examples of **varied document structures** → correct extraction regardless of layout.
- Examples extracting from **odd formats** → fixes empty/null extraction of fields that *are* present but oddly placed.

---

## 5. Anti-patterns (with *why*)
- **Adding more instructions when examples are what's needed.** Prose keeps being interpreted differently; show 2–4 examples.
- **Only easy examples.** They don't teach the hard calls; include the ambiguous boundary cases.
- **Answers without reasoning on ambiguous cases.** The model learns the answer, not the procedure; show the *why*.
- **Over-narrow examples.** The model memorizes those cases and fails on novel ones; pick examples that teach the principle.
- **Too many examples.** Bloats the prompt for little gain; 2–4 targeted ones.
- **No examples of messy formats in extraction.** The model fabricates or returns null; show odd-format handling.

---

## 6. Self-check (core mechanics)
1. Detailed instructions still inconsistent — best fix? → 2–4 targeted few-shot examples.
2. Most valuable examples to include? → Ambiguous cases, **with the reasoning** for the choice.
3. Why show reasoning, not just the answer? → It teaches the decision procedure, which generalizes.
4. Goal of example selection? → Generalization to novel patterns, not memorizing the listed cases.
5. How do examples help extraction? → Reduce hallucination/empty-null by showing handling of informal/varied formats.
6. How many examples? → 2–4 (enough to set the pattern, not bloat the prompt).
7. (Ties to §0) One sentence? → *Show worked examples — especially the hard cases with reasoning — so the model generalizes.*

---

## 7. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Core fit — the ambiguous-tool-selection example.** "High-ambiguity requests" is exactly §2: include 2–4 examples of ambiguous tickets showing **which tool was chosen and why** (resolve technical issue before refunding; verify identity before account changes). The model then generalizes the decision procedure to new ambiguous tickets — directly lifting first-contact resolution.

### 7a. Ambiguous-ticket examples with reasoning
Show the messy request, the reasoning, and the tool sequence chosen over alternatives. Teaches judgment, not just answers.

### 7b. Scenario-1 self-check
1. Best fix for inconsistent tool selection on ambiguous tickets? → Few-shot examples with reasoning.
2. (Goal tie) How does this raise FCR? → The agent makes the right call on novel ambiguous requests, resolving more first time.

---

## 8. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Strong fit.** Few-shot examples of the desired code style and structure lock in consistent generation, and examples of "acceptable pattern vs genuine issue" calibrate any verification step. Format consistency (e.g., a fixed function/test skeleton) is exactly what examples deliver.

### 8a. Style/structure examples
2–3 examples of the house pattern make generated code consistent without re-describing the style each time.

### 8b. Scenario-2 self-check
1. Lock in generation style? → Few-shot examples of the desired structure.

---

## 9. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Core fit — the varied-document-structure example.** §4's "inline citations vs bibliographies, methodology sections vs embedded details" is this scenario. Few-shot examples showing correct extraction/citation from each document layout teach the synthesis agent to handle varied structures instead of failing on unfamiliar ones.

### 9a. Examples per document structure
Show extraction from an inline-citation paper and from a bibliography-style one; the model generalizes to other layouts.

### 9b. Scenario-3 self-check
1. Handle varied document structures reliably? → Few-shot examples of each structure.
2. Why not just describe the structures? → Examples generalize; prose is interpreted inconsistently.

---

## 10. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Strong fit.** Few-shot examples distinguishing **acceptable code patterns from genuine issues** (§4) reduce false positives during exploration while still generalizing to new code. Examples of the desired boilerplate format make generated scaffolding consistent.

### 10a. Acceptable-vs-issue examples
Show a legacy pattern that's fine and a superficially-similar one that's a real bug; the model learns the boundary and generalizes.

### 10b. Scenario-4 self-check
1. Reduce false positives without over-narrowing? → Examples distinguishing acceptable patterns from genuine issues.
2. (Goal tie) Why does this aid productivity? → Fewer noisy flags + consistent boilerplate, generalized to new code.

---

## 11. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Core fit.** Two §-direct uses: include few-shot examples demonstrating the exact **output format** (location, issue, severity, suggested fix) for consistency, and examples distinguishing acceptable patterns from genuine issues to cut false positives while generalizing (pairs with Task 4.1's criteria). Also the "branch-level test coverage gap" ambiguous-case example from §2.

### 11a. Output-format examples
2–3 examples showing a finding as `{location, issue, severity, suggested_fix}` make every review's JSON consistent and actionable.

### 11b. False-positive-reducing examples
Show an acceptable pattern (not flagged) beside a real issue (flagged) so the reviewer generalizes the line.

### 11c. Scenario-5 self-check
1. Consistent finding format? → Few-shot examples of `{location, issue, severity, fix}`.
2. Cut false positives while generalizing? → Examples of acceptable-vs-genuine.
3. Teach a tricky case (branch-coverage gap)? → A few-shot example with reasoning.

---

## 12. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Core fit — the hallucination/empty-null example.** §4 is built for this: few-shot examples showing correct handling of **informal measurements** and **varied formats** reduce fabrication, and examples extracting fields that are present-but-oddly-placed fix the **empty/null** failure on required fields. Examples are the most direct lever on extraction quality.

### 12a. Messy-input examples
Show "a couple tablespoons → 30 ml" and an oddly-placed total being correctly found — teaches non-fabricating handling.

### 12b. Fix empty/null on present fields
Include an example where the field is present in an unusual location and correctly extracted, so the model stops returning null for it.

### 12c. Scenario-6 self-check
1. Reduce fabrication on informal inputs? → Few-shot examples of correct handling.
2. Field present but returned null? → An example extracting it from the odd location.
3. (Goal tie) How does this serve accuracy? → Demonstrated handling of messy/varied formats generalizes to new documents.

---

### Sources verified against current Anthropic guidance (June 2026)
- *Prompt engineering — multishot/examples* (docs.claude.com) — few-shot (multishot) examples are among the most effective techniques for consistent format and behavior; show 2–4 relevant, well-chosen examples; examples improve consistency, reduce misinterpretation, and help generalization.
- The ambiguous-case-with-reasoning, generalization-not-memorization, and extraction (informal measurements, varied document structures, empty/null) applications come from the task statement. These are prompting techniques, not versioned APIs; verify any tool/format references against current docs.
