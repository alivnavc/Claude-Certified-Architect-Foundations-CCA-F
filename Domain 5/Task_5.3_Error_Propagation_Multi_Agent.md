# Task 5.3 — Error Propagation Strategies Across Multi-Agent Systems
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **When a subagent fails, give the coordinator everything it needs to recover: the failure type, what was attempted, any partial results, and possible alternatives. A generic "search unavailable" throws that away. Recover transient failures locally and only propagate what you can't fix (with partial results attached). And never silently swallow an error (returning empty as success) or kill the whole workflow over one subagent's failure.**

Everything here is: structured error context up the chain, local recovery first, access-failure-vs-empty-result, and the two cardinal sins (silent suppression, whole-workflow termination).

---

## 1. Structured error context enables recovery

A coordinator can only make a smart recovery decision if the error tells it enough. Return **structured error context**:

| Field | Why the coordinator needs it |
|---|---|
| **Failure type** | Transient? permission? not-found? → picks the recovery move |
| **What was attempted** | The query/approach tried → avoids repeating it |
| **Partial results** | Anything gathered before failing → synthesis can still use it |
| **Potential alternatives** | Other approaches/sources → coordinator can redirect |

With this, the coordinator can retry, reroute to another source, or proceed with partial data. A bare status gives it nothing to act on.

---

## 2. Generic statuses hide value

A generic "search unavailable" **hides the context** the coordinator needs — was it a timeout (retry), a permission error (don't retry), or no matches (a valid answer)? Collapsing all failures into one opaque string forces the coordinator to guess, usually badly. Specific, structured errors preserve the information that drives good decisions.

---

## 3. Access failure vs valid empty result (the key distinction)

These look similar but mean opposite things:

| | Access failure | Valid empty result |
|---|---|---|
| What happened | Couldn't reach/complete the query (timeout, error) | Query succeeded; there were **no matches** |
| Is it an error? | **Yes** — needs a retry decision | **No** — it's a successful answer ("none found") |
| Coordinator action | Retry / reroute | Accept the empty result; move on |

Conflating them is costly: treating "no matches" as an error triggers pointless retries; treating a timeout as "no matches" silently drops data. Report them distinctly (cf. Task 2.2 §5).

---

## 4. Local recovery, then propagate the rest

Subagents should **recover transient failures locally** — retry a timeout/rate-limit themselves — and only **propagate** errors they **cannot** resolve. When they do propagate, include **what was attempted** and **partial results**, so the coordinator isn't blind. Don't bubble every transient blip to the top (noise, wasted coordinator effort); don't hide an unrecoverable failure either.

---

## 5. The two anti-patterns + coverage annotations

Two opposite failure modes, both wrong:

- **Silent suppression:** returning an empty result *as if it were success* when the query actually failed. The coordinator thinks "no data exists" when really "we couldn't get the data" — a dangerous lie.
- **Whole-workflow termination:** killing the entire run because one subagent failed. One unavailable source shouldn't sink an otherwise good answer.

The constructive middle: **coverage annotations** on synthesis output — explicitly mark which findings are **well-supported** and which **topic areas have gaps** due to unavailable sources. The answer proceeds honestly, flagging where it's thin instead of failing or pretending completeness.

---

## 6. Anti-patterns (with *why*)
- **Generic "X unavailable" status.** Hides failure type/partial results; coordinator can't recover intelligently.
- **Conflating access failure with empty result.** Either pointless retries or silently dropped data; report distinctly.
- **Propagating every transient blip.** Noise; recover transient locally, propagate only the unrecoverable.
- **Silent suppression (empty-as-success).** Coordinator believes false "no data"; never disguise a failure as success.
- **Terminating the whole workflow on one failure.** One bad source sinks a good answer; degrade gracefully.
- **No coverage annotations.** The reader can't tell solid findings from gappy ones; mark them.

---

## 7. Self-check (core mechanics)
1. Four parts of structured error context? → Failure type, what was attempted, partial results, potential alternatives.
2. Why are generic statuses bad? → They hide the context the coordinator needs to recover.
3. Access failure vs valid empty result? → A failed query (retry) vs a successful query with no matches (accept).
4. Where are transient failures handled? → Locally in the subagent; only unrecoverable ones propagate (with partials + attempts).
5. The two cardinal anti-patterns? → Silent suppression (empty-as-success) and whole-workflow termination.
6. How does synthesis stay honest about gaps? → Coverage annotations (well-supported vs gappy areas).
7. (Ties to §0) One sentence? → *Propagate rich error context, recover locally first, distinguish failure from empty, and degrade gracefully.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Strong fit.** The §3 distinction is central: `lookup_order` **timing out** (access failure → retry) vs finding **no orders** (valid empty result → "no orders on file," not an error). A `process_refund` failure should return structured context (failure type, what was attempted) so the agent decides between retry and escalate, rather than a generic "operation failed." (Single-agent here, but the same error-shaping discipline; cf. Task 2.2.)

### 8a. Distinguish timeout from no-orders
Timeout → retry; zero orders → report "none found," don't retry or treat as failure.

### 8b. Scenario-1 self-check
1. `lookup_order` times out vs returns nothing? → Access failure (retry) vs valid empty result (accept).
2. `process_refund` fails — what to return? → Structured context (type, attempt), not "operation failed."

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Partial fit.** If a stage (or sub-task agent) fails, return structured context — which stage, what was attempted, partial output — so the orchestrator can retry just that stage or surface it, rather than terminating the whole pipeline over one failure (§5).

### 9a. Don't kill the pipeline on one stage failure
Return structured per-stage error + partial output; retry/surface that stage, keep the rest.

### 9b. Scenario-2 self-check
1. One stage fails — terminate everything? → No; structured per-stage error, recover that stage.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Core fit — the home scenario.** Every 5.3 idea lands: a search subagent **recovers transient failures locally** and propagates only unrecoverable ones with **what it attempted + partial findings** (§4); it distinguishes a **source timeout** (retry) from **no results found** (valid empty, §3); the coordinator gets **structured error context** to reroute to another source (§1); one failed source does **not** terminate the whole synthesis (§5); and the final answer carries **coverage annotations** marking which topics are well-supported vs gappy (§5).

### 10a. Local recovery + rich propagation
Subagent retries a rate-limited search itself; if it truly can't, it propagates `{type, attempted, partial, alternatives}`.

### 10b. Timeout vs no-results
Source timeout → coordinator retries/reroutes; genuinely no matches → accept as a finding ("no sources on X").

### 10c. Graceful degradation + coverage annotations
One dead source → synthesis proceeds, annotating that topic as under-supported rather than failing the whole answer.

### 10d. Scenario-3 self-check
1. Rate-limited search — handled where? → Locally in the subagent; propagate only if unrecoverable.
2. What does the coordinator receive on an unrecoverable failure? → Structured context: type, attempted, partial results, alternatives.
3. Source timeout vs no results? → Access failure (retry/reroute) vs valid empty result (accept).
4. One source down — kill the answer? → No; degrade gracefully + coverage annotations.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Strong fit.** When exploration subagents fail (a tool errors, a path is inaccessible), return structured context so the main coordinator reroutes — and distinguish "Grep found nothing" (valid empty → that's an answer) from "Grep failed to run" (access failure → retry). Don't abort the whole exploration over one failed sub-query.

### 11a. Empty search vs failed search
"No matches" is a real finding; a tool error is an access failure to retry — report distinctly.

### 11b. Scenario-4 self-check
1. Grep returns nothing vs Grep errors? → Valid empty result vs access failure.
2. One sub-query fails — abort exploration? → No; structured error, reroute.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Partial fit.** If a review sub-step fails (a tool times out), return structured error so the pipeline can retry that step rather than failing the build (a transient blip shouldn't block a merge — a false failure). Don't silently report "no issues" when the review actually failed to run (silent suppression, §5).

### 12a. Don't disguise a failed review as "no issues"
A review that errored must report the failure, not an empty (clean) result — else a broken review silently passes bad code.

### 12b. Scenario-5 self-check
1. Review step times out — fail the build? → Retry the step (transient); don't fail on a blip.
2. Review failed to run — report "no issues"? → No; that's silent suppression (dangerous false pass).

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Strong fit.** In a multi-stage extraction pipeline, distinguish "field genuinely absent" (valid empty → null, a real answer) from "couldn't access the source" (access failure → retry); return structured error context per document so the coordinator reprocesses only the failed ones; and don't terminate a batch because a few documents failed (§5, cf. Task 4.5 resubmit-failed-`custom_id`s).

### 13a. Absent field vs unreadable source
Field not present → null (valid empty); document unreadable → access failure to retry. Distinct reporting.

### 13b. Scenario-6 self-check
1. Field absent vs document unreadable? → Valid empty (null) vs access failure (retry).
2. A few docs fail in a batch — abort all? → No; structured per-doc error, reprocess only those.
3. (Goal tie) How does this serve accuracy + downstream? → Real "not found" isn't faked, real failures are retried, and one bad doc doesn't sink the batch.

---

### Sources verified against current Anthropic / MCP guidance (June 2026)
- *MCP error handling & multi-agent design* (modelcontextprotocol.io, platform.claude.com) — return structured, model-readable error context (not generic statuses); distinguish access/execution failures from valid empty results; recover transient failures locally and propagate only the unrecoverable with partial results + what was attempted (consistent with Task 2.2 §5).
- Structured error context fields, the silent-suppression and whole-workflow-termination anti-patterns, and coverage annotations on synthesis come from the task statement. These are architecture patterns; verify any MCP/SDK field names against current docs.
