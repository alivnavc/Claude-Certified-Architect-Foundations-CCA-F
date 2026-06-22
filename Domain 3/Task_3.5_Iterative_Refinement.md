# Task 3.5 — Iterative Refinement Techniques for Progressive Improvement
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **When prose instructions produce inconsistent results, stop describing and start *showing*: concrete input→output examples, a test suite Claude iterates against, or an interview where Claude asks the questions. And batch your feedback by dependency — fix interacting problems together in one message, fix independent problems one at a time.**

Everything here is about communicating intent precisely (examples, tests, interview) and structuring the feedback loop (batched vs sequential by whether the issues interact).

---

## 1. Concrete examples beat prose

Natural-language descriptions get interpreted inconsistently — "normalize the dates" can mean five things. The most effective fix is **2–3 concrete input→output examples** that pin down the exact transformation:

```
Input:  "2026-1-5"      → Output: "2026-01-05"
Input:  "Jan 5, 2026"   → Output: "2026-01-05"
Input:  "" (empty)      → Output: null
```

Three examples show the happy path, a reformat, and an edge case — far less ambiguous than a paragraph. Reach for examples whenever the same prose keeps yielding different results.

---

## 2. Test-driven iteration

Write the **test suite first** — covering expected behavior, edge cases, and performance requirements — then have Claude implement and iterate by **sharing the failures**. Each failing test is precise, unambiguous feedback ("expected X, got Y on input Z"), so improvement is directed rather than vague. The loop: tests → implement → run → paste failures → fix → repeat until green. The tests are the spec.

---

## 3. The interview pattern

In an unfamiliar domain, you don't know what you don't know. Have **Claude ask you questions** before implementing — it surfaces considerations you hadn't thought of (cache-invalidation strategy, failure modes, concurrency, edge inputs). You answer; the design accounts for them up front instead of discovering gaps after building. It flips the usual direction: the model interviews the developer to extract the spec.

---

## 4. Specific test cases for edge-case fixes

When a particular edge case is mishandled (e.g., **null values in a migration script**), give a **specific input + expected output** for that case rather than describing it. "Here's a row with a null `email`; the script should write `''`, not crash" fixes it precisely — a targeted instance of §1 aimed at the exact failure.

---

## 5. Batch vs sequential feedback (by dependency)

The structure of your feedback should match how the problems relate:

| Problems | How to give feedback | Why |
|---|---|---|
| **Interacting** (a fix changes the others) | **One detailed message** listing all | Claude sees the whole picture and reconciles them together; sequential fixes would thrash as each undoes the last |
| **Independent** (orthogonal) | **Sequentially**, one at a time | Cleaner iterations; each fix is isolated and verifiable |

Misjudging this is a real failure mode: fixing interacting issues one-by-one causes the model to fix A, break B, fix B, break A. If they interact, say everything at once.

---

## 6. Anti-patterns (with *why*)
- **More prose for an ambiguous transform.** Same ambiguity, same inconsistency; give examples.
- **Implement first, test later.** You lose the precise feedback loop; write tests first and iterate on failures.
- **Diving into an unfamiliar domain without the interview.** You miss considerations (invalidation, failure modes); let Claude ask first.
- **Describing an edge-case bug vaguely.** Give the exact input + expected output.
- **Fixing interacting issues sequentially.** Causes thrashing; batch them in one message.
- **Dumping independent issues all at once.** Muddies iterations; handle orthogonal ones sequentially.

---

## 7. Self-check (core mechanics)
1. Prose gives inconsistent results — best fix? → 2–3 concrete input→output examples.
2. How does test-driven iteration work? → Write tests first, implement, iterate by sharing failures.
3. What's the interview pattern for? → Having Claude ask questions to surface unanticipated considerations in unfamiliar domains.
4. Fix a specific edge case (null in a migration)? → Give the exact input + expected output.
5. Interacting vs independent issues? → Interacting → one detailed message; independent → sequential.
6. (Ties to §0) One sentence? → *Show, don't describe — and batch feedback by whether the issues interact.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Strong fit.** Tuning the agent's behavior is iterative: when "handle ambiguous requests well" is too vague, give **concrete example conversations → desired resolutions** (§1). Use the **interview pattern** to surface edge cases (partial refunds, locked accounts) before coding the policy. Build an **eval/test suite** of representative tickets and iterate on failures (§2).

### 8a. Example dialogues pin behavior
2–3 example tickets with the ideal handling beat a prose description of "good support."

### 8b. Interview to surface policy edge cases
Let Claude ask about refund limits, multi-issue tickets, fraud flags before you finalize the gate logic.

### 8c. Scenario-1 self-check
1. "Handle ambiguity well" is too vague — fix? → Concrete example ticket → resolution pairs.
2. Surface policy edge cases first? → Interview pattern.
3. Drive improvement systematically? → Eval suite of tickets; iterate on failures.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Core fit — test-driven iteration is the spine.** This pipeline already has a test stage; §2 says make tests the spec — write them first (behavior, edge cases, performance), generate, run, and iterate on failures. For ambiguous transforms, add input→output examples (§1).

### 9a. Tests as the spec
Write the suite first; the generator iterates against failures until green — precise, directed improvement.

### 9b. Batch interacting fixes
If two failing tests stem from interacting logic, report both in one message so the fix reconciles them (§5).

### 9c. Scenario-2 self-check
1. Communicate expected behavior to the generator? → Tests first, iterate on failures.
2. Two failures from interacting logic? → One message with both.
3. Ambiguous transform spec? → Add input→output examples.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Strong fit for the interview pattern.** Research design in an unfamiliar domain is exactly where the **interview pattern** shines — have Claude ask what sources count as authoritative, how to handle conflicting findings, what citation format downstream needs, before building. Examples (§1) pin the desired output (a sample question → ideal cited answer).

### 10a. Interview to define research quality
Let Claude ask about source authority, conflict handling, citation format up front.

### 10b. Example of a gold answer
One sample query → ideal cited answer pins the output shape better than prose.

### 10c. Scenario-3 self-check
1. Design research behavior in a new domain? → Interview pattern.
2. Pin the answer format? → A concrete example gold answer.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Core fit — every technique applies.** Ambiguous refactor intent → input→output **examples** (before/after code). New library/domain → **interview pattern** (cache invalidation, failure modes — the exam's example). Behavior changes → **test-driven iteration**. Multiple review comments → **batch interacting** ones, **sequence independent** ones (§5).

### 11a. Before/after examples for refactors
Show 2–3 before→after snippets so the refactor is unambiguous.

### 11b. Interview in an unfamiliar domain
"Add caching to this service" → let Claude ask about invalidation strategy and failure modes before implementing.

### 11c. Batch vs sequence the fixes
Three interacting bugs in one subsystem → one detailed message. Three unrelated cleanups → one at a time.

### 11d. Scenario-4 self-check
1. Refactor intent keeps coming out wrong? → Before/after examples.
2. Adding caching to an unfamiliar service? → Interview pattern (invalidation, failure modes).
3. Three interacting bugs? → One detailed message.
4. Three unrelated cleanups? → Sequential.
5. (Goal tie) How does this aid productivity? → Precise communication and well-structured feedback cut the number of iterations to correct output.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Strong fit.** "Minimize false positives" is an iterative tuning goal: give **examples** of a true issue vs a false positive to calibrate the reviewer (§1), and build a **test suite** of sample PRs with known-correct verdicts, iterating until the reviewer matches (§2). Calibration is refinement.

### 12a. Examples to calibrate severity
Show a real-issue example and a false-positive example so the reviewer learns the line — fewer noisy flags.

### 12b. PR test suite
Sample PRs with expected verdicts; iterate the prompt until the reviewer's output matches.

### 12c. Scenario-5 self-check
1. Reduce false positives via refinement? → Examples of true-issue vs false-positive; iterate.
2. Systematic calibration? → A suite of sample PRs with known verdicts.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Core fit.** Extraction is the textbook §1/§4 case: when "extract the fields" is interpreted inconsistently, give **2–3 input→output examples** (raw doc snippet → exact record). For edge-case bugs (the exam's **null values in migration scripts**), give the **specific input + expected output** (§4). Test-driven iteration (§2) over a labeled document set drives accuracy up.

### 13a. Input→output examples pin extraction
Sample raw text → exact target record (including how nulls/missing fields appear) removes ambiguity better than prose.

### 13b. Specific case for the null bug
"This row has a null `email` → write `''`" fixes the exact edge case precisely.

### 13c. Test-driven accuracy
A labeled set of documents with expected records; iterate on mismatches until accuracy hits target.

### 13d. Scenario-6 self-check
1. "Extract the fields" interpreted inconsistently? → 2–3 input→output examples.
2. Fix null-handling in a migration? → Specific input + expected output.
3. Drive accuracy systematically? → Labeled doc set; iterate on failures.
4. (Goal tie) How does this serve "high accuracy" + "graceful edge cases"? → Examples and labeled tests make the exact expected behavior unambiguous, including edges.

---

### Sources verified against current Claude Code / Anthropic guidance (June 2026)
- *Prompt engineering / Claude Code best practices* (docs.claude.com, code.claude.com) — concrete input/output examples for ambiguous transformations; test-driven iteration (tests first, iterate on failures); the interview pattern (Claude asks clarifying questions before implementing).
- These are workflow techniques, not versioned APIs; the examples (null-in-migrations, cache-invalidation interview, batch-vs-sequential by dependency) come from the task statement. No fast-moving API surface here, but verify any tool/command references against current docs.
