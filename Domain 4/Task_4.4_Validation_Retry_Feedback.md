# Task 4.4 — Validation, Retry & Feedback Loops for Extraction Quality
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **When output is wrong, feed the *specific* error back and ask the model to fix it — but only when a retry can actually help. Retries fix format and structural mistakes; they cannot conjure information that isn't in the source. Build the loop to catch *semantic* errors (recompute the total, flag conflicts), and add fields like `detected_pattern` so you can learn which constructs cause false positives over time.**

Everything here is: retry-with-specific-error-feedback, knowing when retry is futile, designing semantic self-checks, and instrumenting findings for analysis.

---

## 1. Retry with error feedback

On a failed/invalid extraction, don't just retry blindly — **append the specific validation error** to a follow-up request so the model knows exactly what to correct. The follow-up includes: the **original document**, the **failed extraction**, and the **precise errors**:

```
Here is the document: <doc>
Your previous extraction: <failed JSON>
Validation errors: total (1450) does not equal sum of line items (1400);
  "currency" missing.
Please correct these specific issues.
```

Specific feedback turns a vague "try again" into a directed correction — far higher success than a bare retry.

---

## 2. When retry is futile (the key judgment)

Retries help only when the problem is the *model's handling*, not the *source's content*:

| Retry **will** help | Retry **won't** help |
|---|---|
| Format mismatch (wrong date format) | Required info simply **absent** from the document |
| Structural output error (wrong nesting) | Data exists only in an **external** doc not provided |
| Total doesn't sum (math redo) | The source genuinely doesn't contain the field |

If the information **isn't in the source**, no amount of retrying produces it — the model will either keep failing or start fabricating. Recognize "this can't be retried into existence" and route to a different path (request the missing doc, mark as not-found, escalate) instead of burning retries.

---

## 3. Semantic vs syntax errors (why this task exists)

Tool use (Task 4.3) already eliminates **syntax** errors. What's left — and what this task targets — is **semantic** errors that pass schema validation:

- Line items that don't sum to the stated total.
- A value in the wrong field.
- Internally inconsistent source data.

These need *validation logic you write*, not the schema. The loop's job is to detect them and feed them back (§1) or flag them.

---

## 4. Designing self-correction validation flows

Bake the checks into the extraction shape so discrepancies surface automatically:

- **Extract both `calculated_total` and `stated_total`** — then compare them; a mismatch flags a likely extraction (or source) error.
- **Add a `conflict_detected` boolean** — set when the source data is internally inconsistent (two different dates for the same event), so downstream knows the record is suspect rather than silently trusting it.

The model does the arithmetic/consistency check as part of extraction, and your code branches on the flags.

---

## 5. Instrumenting findings: `detected_pattern`

To improve precision over time, add a **`detected_pattern`** field to each structured finding recording *which code construct* triggered it. When developers **dismiss** findings, you can then analyze *which patterns* generate false positives systematically — and fix those prompts/criteria (Task 4.1). Without this field, dismissals are just noise; with it, they're a dataset for improvement.

---

## 6. Anti-patterns (with *why*)
- **Bare retry with no error detail.** The model doesn't know what to fix; append the specific errors.
- **Retrying for missing source info.** It can't be retried into existence; you'll get failures or fabrication. Route elsewhere.
- **Trusting schema-valid output as correct.** Semantic errors pass; add validation logic.
- **No `calculated_total` cross-check.** Wrong totals ship silently; extract and compare both.
- **No `conflict_detected` flag.** Inconsistent source data is trusted blindly; flag it.
- **No `detected_pattern` field.** Dismissals can't be analyzed; you can't find the false-positive sources.

---

## 7. Self-check (core mechanics)
1. What do you append on retry? → The specific validation errors (plus original doc + failed extraction).
2. When is retry futile? → When the required info is **absent from the source** (or only in an unprovided external doc).
3. When does retry succeed? → Format mismatches and structural output errors.
4. Semantic vs syntax errors? → Syntax (eliminated by tool use) vs semantic (totals don't sum, wrong field) — this task handles semantic.
5. Self-check for a wrong total? → Extract `calculated_total` and `stated_total`; compare.
6. Flag inconsistent source data? → A `conflict_detected` boolean.
7. Analyze false-positive sources? → A `detected_pattern` field on findings.
8. (Ties to §0) One sentence? → *Feed specific errors back when retry can help; flag semantics and instrument findings.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Strong fit.** The §2 judgment is central: if a customer's order info **isn't in the system**, retrying the lookup won't help — escalate or request more info rather than looping. A `conflict_detected` flag surfaces inconsistent account data so the agent doesn't act on bad info.

### 8a. Don't retry the unretryable
Order genuinely not in the system → escalate/ask, don't loop the lookup.

### 8b. Scenario-1 self-check
1. Lookup returns nothing because the order isn't in the system — retry? → No; route to escalation/ask.
2. Inconsistent account data? → `conflict_detected` flag.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Core fit — retry-with-feedback is the test loop.** Failing tests are the validation errors; append them to the follow-up so the model fixes the specific failure (Task 3.5's test-driven iteration, mechanized). Distinguish a flaky-infra failure (retry) from a real bug needing a code fix vs a missing-dependency that retry won't solve (§2).

### 9a. Test failures as error feedback
Append the exact failing tests to the retry so the fix is directed, not blind.

### 9b. Scenario-2 self-check
1. What do you feed back on a failed test run? → The specific failures.
2. Missing dependency not in the repo? → Retry won't fix it; surface it.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Strong fit for §2 and §4.** If a claim's support **isn't in the gathered sources**, retrying synthesis won't manufacture it — fetch more sources or mark unsupported. A `conflict_detected` flag captures contradictory findings across sources so the answer notes the disagreement instead of picking one silently.

### 10a. Unsupported claim → fetch, don't retry
Missing support is a source gap, not a model error; gather more or flag unsupported.

### 10b. Flag cross-source conflicts
`conflict_detected` when sources disagree, so synthesis surfaces the conflict.

### 10c. Scenario-3 self-check
1. Claim unsupported by gathered sources — retry synthesis? → No; fetch more or mark unsupported.
2. Sources contradict? → `conflict_detected` flag.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Strong fit.** Retry-with-feedback applies whenever generated code fails validation (lint/type/test) — feed the exact errors. The §2 judgment: if the fix needs context in a file the assistant wasn't given, retrying won't help — provide the file. `detected_pattern` on its findings lets you learn which constructs it over-flags.

### 11a. Feed lint/type/test errors back
Append the specific compiler/test errors for a directed fix.

### 11b. Know when to add context, not retry
Failure due to missing file context → provide the file rather than loop.

### 11c. Scenario-4 self-check
1. Generated code fails type-check — what do you feed back? → The exact type errors.
2. Fix needs an unprovided file? → Provide it; don't retry blindly.
3. Track over-flagged constructs? → `detected_pattern` field.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Core fit for `detected_pattern`.** §5 is built for this: add `detected_pattern` to each finding so when developers dismiss comments, you analyze **which constructs** produce false positives and fix those criteria (Task 4.1's disable/improve loop). This is the data engine behind "minimize false positives."

### 12a. `detected_pattern` for dismissal analysis
Record the triggering construct on each finding; dismissals become a dataset showing which patterns to fix.

### 12b. Scenario-5 self-check
1. Systematically find false-positive sources? → `detected_pattern` field + dismissal analysis.
2. (Goal tie) How does this minimize false positives over time? → It turns dismissals into targeted prompt fixes per construct.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Core fit — the home scenario.** All of 4.4 lands: **retry with specific validation errors** (original doc + failed extraction + errors) for format/structural fixes; the **§2 judgment** that a field genuinely absent from the document can't be retried into existence (mark not-found, don't fabricate); **semantic self-checks** via `calculated_total` vs `stated_total` and `conflict_detected`; and the syntax-vs-semantic split (§3) — tool use handled syntax, this handles meaning.

### 13a. Retry with the exact errors
Follow-up includes the document, the failed JSON, and the precise validation errors → directed correction.

### 13b. Absent-info judgment
Field not in the document → not-found path, never fabricate; retrying won't help.

### 13c. Semantic self-correction
Extract `calculated_total` + `stated_total` and compare; set `conflict_detected` on inconsistent source data.

### 13d. Scenario-6 self-check
1. Extraction fails schema/format validation — retry how? → Append the specific errors + original doc + failed extraction.
2. Field genuinely not in the doc? → Mark not-found; don't retry/fabricate.
3. Catch a total that doesn't sum? → `calculated_total` vs `stated_total`.
4. Inconsistent source data? → `conflict_detected` boolean.
5. (Goal tie) How does this serve "high accuracy" + "graceful edge cases"? → Directed retries fix fixable errors; semantic flags catch what schemas can't; absent-info is handled, not faked.

---

### Sources verified against current Anthropic guidance (June 2026)
- *Tool use / structured output* (platform.claude.com) — tool use eliminates JSON **syntax** errors but not **semantic** correctness; validation/retry loops are an application pattern layered on top.
- Retry-with-error-feedback, the absent-information limit of retries, `calculated_total`/`stated_total` and `conflict_detected` self-checks, and `detected_pattern` instrumentation come from the task statement. These are workflow patterns, not versioned APIs; verify any tool/schema specifics against current docs.
