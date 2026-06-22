# Task 5.5 — Human Review Workflows & Confidence Calibration
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **A high overall accuracy number can hide a field or document type that's failing badly, so measure accuracy *by segment* before you trust automation. Have the model output per-field confidence, calibrate the review thresholds against a labeled validation set, and route the low-confidence and ambiguous cases to humans. Keep watching with stratified random sampling — even of the "high-confidence" outputs — to catch new error patterns.**

Everything here is: don't trust the aggregate, calibrate confidence against ground truth, route by it, and keep sampling.

---

## 1. Aggregate accuracy hides segment failures

"97% accurate overall" feels safe — but it can **mask poor performance on a specific document type or field**. If invoices are 99% but handwritten receipts are 60%, the blended number looks fine while a whole segment is broken. So before automating, **analyze accuracy by document type and by field segment** and confirm performance is acceptable **across all segments**, not just on average. An average is not a guarantee.

---

## 2. Field-level confidence, calibrated against labeled data

Have the model **output a confidence score per field** (not one number for the whole record), then **calibrate the review thresholds using a labeled validation set**. Calibration means: check whether the model's stated confidence actually matches real accuracy on data where you know the truth — does "0.9 confidence" really mean ~90% correct? Only after calibrating against ground truth can you trust a threshold to route review. (Raw, uncalibrated confidence is unreliable — cf. Task 4.1; calibration is what makes it usable.)

---

## 3. Routing review attention

With calibrated field-level confidence, **route**:

- **Low model confidence** → human review.
- **Ambiguous or contradictory source documents** → human review.
- High-confidence, unambiguous → automate.

This **prioritizes limited reviewer capacity** on the cases most likely wrong, instead of reviewing everything (too slow) or nothing (too risky). Reviewers spend their time where it changes outcomes.

---

## 4. Stratified random sampling (ongoing QA)

Even after automating the high-confidence tier, **keep measuring** — because confidence can drift and new error types appear. Use **stratified random sampling**: randomly sample within each stratum (document type, field, confidence band), including the **high-confidence extractions**, to measure the real error rate and **detect novel error patterns** you didn't anticipate. "Stratified" ensures every segment is checked, not just the common one (which would re-hide the rare-segment problem of §1).

---

## 5. Validate by segment *before* reducing review

The sequence matters: **before reducing human review** for a category, **validate accuracy by document type and field** for that category specifically. Don't automate a segment on the strength of the global number; prove that segment is reliable first. Automation is earned per-segment, not granted wholesale.

---

## 6. Anti-patterns (with *why*)
- **Trusting the aggregate accuracy.** It hides failing segments; measure by document type and field.
- **One confidence number per record.** Too coarse to route; output per-field confidence.
- **Using raw confidence to route.** Uncalibrated confidence doesn't track accuracy; calibrate against labeled data first.
- **Reviewing everything or nothing.** Wastes capacity or takes on risk; route by calibrated confidence + ambiguity.
- **Stopping measurement after automating.** Confidence drifts, new errors appear; keep stratified sampling (incl. high-confidence).
- **Automating a segment on the global number.** Validate that segment specifically before reducing its review.

---

## 7. Self-check (core mechanics)
1. Why distrust "97% overall"? → It can mask poor performance on a specific document type or field.
2. What granularity of confidence, and how trusted? → Per-field, calibrated against a labeled validation set.
3. What gets routed to humans? → Low-confidence and ambiguous/contradictory cases.
4. Why route instead of review all/none? → To prioritize limited reviewer capacity on likely-wrong cases.
5. How do you keep catching errors after automating? → Stratified random sampling, including high-confidence extractions.
6. What must precede reducing review for a segment? → Validating that segment's accuracy by type and field.
7. (Ties to §0) One sentence? → *Measure by segment, calibrate confidence against truth, route the risky cases, and keep sampling.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Partial fit.** The routing idea maps to escalation/QA: route uncertain or ambiguous resolutions to human review, and don't trust an aggregate resolution rate that might hide a failing request category (e.g., billing disputes resolved poorly while returns look great). Stratified sampling of "auto-resolved" tickets catches drift.

### 8a. Segment the resolution metric
Check resolution quality per request type; a high overall rate can hide a weak category.

### 8b. Scenario-1 self-check
1. Is an 85% overall resolution rate safe to trust? → Not without checking per request-type segments.
2. Catch drift in auto-resolved tickets? → Stratified sampling.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Partial fit.** Before auto-merging generated code, validate by segment (which kinds of changes pass tests reliably) and route low-confidence generations to human review. An aggregate "tests pass 95%" can hide a change-type that frequently breaks subtly.

### 9a. Route low-confidence generations
Auto-accept high-confidence, human-review the rest; segment the pass-rate by change type.

### 9b. Scenario-2 self-check
1. Trust a 95% test-pass rate to auto-merge all? → No; segment by change type first.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Partial fit.** Route low-confidence or conflicting findings to human review; segment quality by source type (a method that's accurate on news but poor on technical papers). Stratified sampling of accepted findings catches novel error patterns.

### 10a. Route weak/conflicting findings + segment by source type
Human-review uncertain claims; check accuracy per source type, not just overall.

### 10b. Scenario-3 self-check
1. Trust overall citation accuracy? → Segment by source type first.
2. Route conflicting findings? → Human review.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Partial fit.** Route low-confidence automated changes for human review; validate by task segment (boilerplate generation may be reliable while legacy refactors aren't) before reducing oversight on a category.

### 11a. Validate per task type before reducing review
Prove legacy-refactor reliability separately from boilerplate before automating it.

### 11b. Scenario-4 self-check
1. Reduce review for legacy refactors on the global number? → No; validate that segment first.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Strong fit.** Calibrated per-finding confidence routes review (Task 4.6): auto-post high-confidence findings, send low-confidence to a human — directly serving "minimize false positives." Segment accuracy by issue type before trusting any category to auto-gate; stratified-sample auto-posted findings to catch new false-positive patterns.

### 12a. Calibrated confidence routing
Per-finding confidence (calibrated on labeled PRs) decides auto-post vs human review.

### 12b. Segment + sample
Check accuracy per issue type before auto-gating it; stratified-sample auto-posted findings for drift.

### 12c. Scenario-5 self-check
1. Route findings to reduce false positives? → Calibrated per-finding confidence (auto-post high, escalate low).
2. Trust the reviewer's overall accuracy to auto-gate everything? → No; segment by issue type first.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Core fit — the home scenario.** Every 5.5 idea lands: an overall 97% can hide a failing **document type or field** (§1), so **analyze accuracy by document type and field** before automating (§5); output **field-level confidence** calibrated on a **labeled validation set** (§2); **route** low-confidence and ambiguous/contradictory documents to human review (§3); and run **stratified random sampling** of high-confidence extractions to measure the true error rate and catch **novel patterns** (§4).

### 13a. Don't trust the aggregate
Measure accuracy per document type and per field; confirm every segment, not just the 97% average.

### 13b. Calibrated field-level confidence + routing
Per-field confidence calibrated on labeled data; route low-confidence and contradictory docs to humans, automate the rest.

### 13c. Stratified sampling, including high-confidence
Sample within each stratum (type/field/confidence band) — even high-confidence ones — to find drift and new error types.

### 13d. Validate before automating a segment
Prove a document type is reliable by type and field before reducing its human review.

### 13e. Scenario-6 self-check
1. Is 97% overall safe to automate on? → No; check per document type and field.
2. What granularity of confidence, calibrated how? → Per-field, against a labeled validation set.
3. Which extractions go to humans? → Low-confidence and ambiguous/contradictory ones.
4. How to keep catching errors post-automation? → Stratified sampling, including high-confidence extractions.
5. (Goal tie) How does this serve "high accuracy"? → Per-segment validation + calibrated routing + ongoing sampling ensure no segment silently fails.

---

### Sources verified against current Anthropic guidance (June 2026)
- *Evaluation / human-in-the-loop guidance* (docs.claude.com) — segment-level accuracy measurement (aggregate metrics can mask segment failures), calibrating model-reported confidence against labeled/validation data, and routing low-confidence/ambiguous cases to human review; consistent with Task 4.1's caution that raw confidence is unreliable until calibrated.
- Stratified random sampling (incl. high-confidence outputs), field-level confidence calibration, and validate-by-segment-before-automating come from the task statement. These are evaluation/QA methods, not versioned APIs; verify any tooling references against current docs.
