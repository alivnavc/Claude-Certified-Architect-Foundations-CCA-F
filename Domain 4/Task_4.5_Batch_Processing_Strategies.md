# Task 4.5 — Designing Efficient Batch Processing Strategies
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **The Message Batches API runs many requests asynchronously for 50% off, but you trade away latency: results arrive within a 24-hour window with no faster guarantee. So batch the latency-tolerant bulk work (overnight reports, weekly audits, nightly test gen) and keep the synchronous API for anything a human or pipeline is waiting on (pre-merge checks). Correlate results with `custom_id`, and refine your prompt on a sample before submitting the whole pile.**

Everything here is: the cost/latency trade, the blocking-vs-non-blocking decision, the tool-use limitation, `custom_id` correlation, and SLA-aware scheduling.

---

## 1. What the Batches API is (the trade)

The **Message Batches API** processes large volumes of Messages requests **asynchronously** at **50% of standard token cost** (input *and* output). The catch: a **24-hour processing window** with **no guaranteed latency SLA** below that — most batches finish in under an hour, but you can't count on it. You submit a list of requests (each with a `custom_id` and standard Messages params), poll for completion, and retrieve results. Half price in exchange for "sometime within a day."

(Scale: up to 100,000 requests / 256 MB per batch; results retained 29 days; expired requests aren't billed; prompt-caching discounts stack with the batch discount.)

---

## 2. Blocking vs non-blocking (the core decision)

The deciding question: **is anything waiting on the result right now?**

| Batch (async, 50% off) | Synchronous (full price, immediate) |
|---|---|
| Overnight reports | Pre-merge checks |
| Weekly audits | Interactive chat |
| Nightly test generation | Anything a user/pipeline blocks on |

A **pre-merge check** blocks the developer's merge — it must be synchronous; nobody waits up to 24 hours to merge. A **nightly test-generation** job blocks nothing — batch it for half price. Match the API to the latency requirement, not to cost alone.

---

## 3. The tool-use limitation

The batch API does **not** support **multi-turn tool calling within a single request** — it can't execute a tool mid-request and feed the result back to continue the loop, because that needs interactive round-trips and batch is one-shot per request. A request can include tool definitions and return a `tool_use` block, but the agentic *loop* (call → execute → return → continue) can't run inside one batch request. So workflows that need mid-request tool execution don't fit batch; use the synchronous agent loop.

---

## 4. `custom_id` correlation

Results come back **out of order** (asynchronous, concurrent processing), so each request carries a **`custom_id`** (1–64 chars, `[a-zA-Z0-9_-]`) that you map to your internal record. You join results to inputs by `custom_id` rather than relying on order. This is also how you identify **which specific requests failed** to resubmit just those.

---

## 5. SLA-aware scheduling and failure handling

- **Scheduling math:** if you must guarantee a 30-hour end-to-end SLA and a batch can take up to 24 hours, submit on a window that leaves headroom — e.g., submit every **4 hours** so worst-case 24h processing + the window still lands inside 30h. Plan around the 24-hour *ceiling*, not the typical minutes.
- **Failure handling:** resubmit **only the failed `custom_id`s**, with fixes — e.g., **chunk** a document that exceeded the context limit, then resubmit those chunks. Don't re-run the whole batch.
- **Sample first:** refine your prompt on a small sample to maximize **first-pass success**, *then* submit the large volume — iterating on a 50,000-doc batch is slow (hours per loop) and expensive.

---

## 6. Anti-patterns (with *why*)
- **Batching a pre-merge check.** It blocks the merge; up-to-24h latency is unacceptable — use sync.
- **Expecting fast results from batch.** No sub-24h guarantee; design for the ceiling.
- **Mid-request tool loops in batch.** Unsupported; use the synchronous agent loop.
- **Joining results by order.** They arrive out of order; correlate by `custom_id`.
- **Resubmitting the whole batch on partial failure.** Wasteful; resubmit only failed `custom_id`s, with fixes.
- **Submitting a huge batch before refining the prompt.** Each iteration costs hours; sample-refine first.

---

## 7. Self-check (core mechanics)
1. Batch API's trade? → 50% cheaper, but up-to-24h async with no faster SLA.
2. Pre-merge check — batch or sync? → **Sync** (it blocks the merge).
3. Nightly test generation — batch or sync? → **Batch** (latency-tolerant).
4. Can a batch request run a mid-request tool loop? → **No** (no multi-turn tool calling within a request).
5. How do you match results to inputs? → `custom_id` (results arrive out of order).
6. A doc exceeded context in the batch — what do you resubmit? → Just that `custom_id`, chunked.
7. Guarantee a 30h SLA with 24h batches? → Submit on a window with headroom (e.g., every 4h).
8. (Ties to §0) One sentence? → *Batch the latency-tolerant bulk for half price; keep blocking work synchronous; correlate by `custom_id`.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Mostly a contrast case.** The live agent is **blocking** — a customer is waiting — and it needs **mid-request tool calling** (the agent loop), both of which rule out batch (§2, §3). Batch fits only the *offline* side: nightly eval runs over yesterday's transcripts, or weekly quality audits.

### 8a. Live = sync; offline analysis = batch
Real-time support must be synchronous (and uses the tool loop); batch the overnight eval/audit jobs.

### 8b. Scenario-1 self-check
1. Live support — batch? → No; it's blocking and needs the tool loop.
2. Nightly transcript evals? → Batch (50% off, latency-tolerant).

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Partial fit.** Interactive generation with a test loop needs **mid-request tool execution** → synchronous (§3). But a **nightly bulk test-generation** job over many modules (no one waiting) is a textbook batch use — half price, overnight.

### 9a. Interactive = sync; nightly test-gen = batch
The live plan→write→test loop is sync; batch the unattended overnight test generation.

### 9b. Scenario-2 self-check
1. Live generation loop — batch? → No (mid-request tools, interactive).
2. Nightly test generation across modules? → Batch.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Partial fit.** Interactive research with tool-using subagents is synchronous (§3). But **bulk offline** research — process 5,000 documents for a weekly intelligence report — is ideal batch work: independent, latency-tolerant, high volume; correlate each document's result by `custom_id`.

### 10a. Bulk offline research → batch
Submit the document set as a batch; join results by `custom_id`; resubmit only failed docs (chunked if oversized).

### 10b. Scenario-3 self-check
1. Interactive research with subagent tool loops? → Sync.
2. Weekly 5,000-doc report? → Batch; correlate by `custom_id`.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Partial fit.** Interactive coding is synchronous and tool-loop-heavy (§3). Batch fits unattended bulk chores: generating boilerplate across 200 modules overnight, or a weekly repo-wide documentation pass — no one's waiting, so take the 50%.

### 11a. Bulk overnight chores → batch
Repo-wide doc generation or mass boilerplate = batch; interactive work = sync.

### 11b. Scenario-4 self-check
1. Interactive coding session — batch? → No (interactive, tool loops).
2. Overnight repo-wide doc pass? → Batch.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Core fit — the canonical contrast.** A **pre-merge review blocks the merge** → it must be **synchronous** (the headline §2 example: you can't make a developer wait up to 24h to merge). But a **nightly full-repo audit** or **weekly tech-debt scan** is non-blocking → batch it for half price. Same reviewer, different latency requirement, different API.

### 12a. Pre-merge = sync, nightly audit = batch
The merge-gating review is synchronous; the overnight whole-repo audit is batched.

### 12b. SLA-aware audit scheduling
If the audit must land within 30h, submit on a 4-hour window so worst-case 24h processing still fits.

### 12c. Scenario-5 self-check
1. Pre-merge PR review — batch or sync? → **Sync** (blocks the merge).
2. Nightly repo-wide audit? → Batch (latency-tolerant, 50% off).
3. Guarantee a 30h audit SLA? → Submit every ~4h for headroom over the 24h ceiling.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Core fit — the home scenario.** High-volume offline extraction is the textbook batch workload: thousands of documents, no one waiting, 50% off. Apply every §5 skill — **sample-refine the prompt** before the big run, **`custom_id`** per document, **resubmit only failed docs** (chunk those that exceeded context), and **SLA-aware scheduling**. The one caveat: extraction needing a mid-request tool loop can't batch (§3) — but single-shot schema'd extraction (Task 4.3) fits perfectly.

### 13a. Bulk extraction as a batch
Submit the document set; 50% savings; results correlated by `custom_id`.

### 13b. Sample-refine, then scale
Tune the extraction prompt on a small sample to maximize first-pass success before the 50,000-doc run — iterating on the full batch is slow and costly.

### 13c. Failure handling
Resubmit only failed `custom_id`s; chunk documents that blew the context limit and resubmit the chunks.

### 13d. Scenario-6 self-check
1. 50,000-doc extraction, results next morning fine? → Batch (50% off).
2. Maximize first-pass success? → Refine the prompt on a sample first.
3. 30 docs failed (context overflow)? → Resubmit just those `custom_id`s, chunked.
4. Single-shot schema'd extraction in batch? → Yes; mid-request tool loops? No.
5. (Goal tie) How does batch serve this at scale? → Half the cost for latency-tolerant volume, with `custom_id` correlation and targeted resubmission.

---

### Sources verified against current Anthropic docs (June 2026)
- *Batch processing* (platform.claude.com) — Message Batches API: **50% discount** on input+output across models; **24-hour** processing window (most finish < 1 hour) with **no faster SLA**; up to **100,000 requests / 256 MB** per batch; `custom_id` (1–64 chars, `^[a-zA-Z0-9_-]{1,64}$`) for correlating request/response and identifying failures; results retained **29 days**; expired (>24h) requests not billed; prompt-caching discounts **stack** with batch.
- The blocking-vs-non-blocking guidance, the no-multi-turn-tool-calling-within-a-request limitation, SLA-window math (4h windows for a 30h SLA), failed-`custom_id` resubmission/chunking, and sample-refine-first come from the task statement. Batch limits/pricing ship fast — re-verify against current docs.
