# Task 5.4 — Managing Context in Large Codebase Exploration
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **In a long exploration, context degrades — the model starts giving inconsistent answers and citing "typical patterns" instead of the specific classes it found earlier. Fight this by keeping the heavy reading out of the main context (delegate to subagents that return summaries), persisting findings to a scratchpad file the main agent re-reads, summarizing each phase before the next, and `/compact`-ing when context fills. For crashes, export structured agent state to a known location and reload a manifest on resume.**

Everything here is: detect degradation, offload verbose discovery (subagents), persist findings (scratchpad + summaries), trim (`/compact`), and recover (state manifests).

---

## 1. Context degradation in extended sessions

As a session runs long, quality **degrades**: the model gives **inconsistent answers** to the same question and starts referencing **"typical patterns"** ("usually a service like this would…") rather than the **specific classes it actually discovered** earlier. That's the tell — generic hand-waving replacing concrete, found facts. It means earlier discoveries have effectively fallen out of usable context. Everything below counteracts this.

---

## 2. Subagent delegation isolates verbose output

The main agent should **coordinate high-level understanding**, not drown in raw output. Spawn **subagents** to investigate specific questions — "find all test files," "trace the refund-flow dependencies" — so the **verbose discovery output stays in the subagent's context**, and only a **summary** returns to the main agent. The main agent keeps a clean, high-level view; the noisy file-reading happens elsewhere. (This is Task 3.4's Explore subagent / Task 1.3 delegation, applied to exploration.)

---

## 3. Scratchpad files persist findings

A **scratchpad file** is an external file where the agent **records key findings** as it works ("AuthService lives in `src/auth/`, calls TokenStore; refund flow: Controller→RefundService→PaymentGateway"). The agent **references it for subsequent questions**, so discoveries survive across context boundaries even as the conversation window churns. It's external memory that doesn't degrade with the session — the direct counter to §1.

---

## 4. Phase summaries + `/compact`

- **Summarize before the next phase.** Before spawning subagents for phase 2, summarize phase 1's key findings and **inject that summary into the initial context** of the next phase. Each phase starts grounded in what's known, not the raw history.
- **`/compact`** compresses the conversation history to reduce context usage during extended exploration when it fills with verbose discovery. Use it when the window is bloating; it preserves the thread while reclaiming space. (Project-root CLAUDE.md is re-read after compaction — Task 3.1.)

---

## 5. Crash recovery via structured state manifests

For long, multi-agent explorations that might crash, design **structured state persistence**: each agent **exports its state to a known location** (a manifest — what it found, where it was), and on resume the **coordinator loads the manifest** and **injects it into the agents' prompts**. Recovery becomes "reload the manifest and continue" instead of restarting the whole exploration. The manifest is the durable checkpoint of distributed progress.

---

## 6. Anti-patterns (with *why*)
- **Ignoring degradation signs.** "Typically…" answers mean discoveries fell out of context; persist and re-ground.
- **Doing all discovery in the main agent.** Verbose output crowds the high-level view; delegate to subagents for summaries.
- **Relying on the conversation window as memory.** It churns and degrades; use a scratchpad file.
- **Starting each phase from raw history.** Bloated and ungrounded; summarize the prior phase and inject it.
- **Never compacting a bloated session.** Context exhausts; `/compact` when it fills.
- **No state export for long runs.** A crash loses everything; export manifests, reload on resume.

---

## 7. Self-check (core mechanics)
1. Signs of context degradation? → Inconsistent answers; "typical patterns" instead of specific discovered classes.
2. How to isolate verbose discovery? → Subagents that investigate and return summaries; main agent coordinates.
3. How to persist findings across context boundaries? → A scratchpad file the agent records to and re-reads.
4. How to start each exploration phase grounded? → Summarize the prior phase; inject the summary into the next phase's context.
5. Command to reclaim context mid-session? → `/compact`.
6. Crash recovery design? → Each agent exports structured state (manifest) to a known location; coordinator reloads on resume.
7. (Ties to §0) One sentence? → *Offload verbose discovery, persist findings externally, compact when full, and checkpoint state for recovery.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Minimal fit.** This task is about *codebase* exploration, which is the developers' workflow, not the support agent's runtime. The carry-over is the general degradation lesson: a very long support conversation benefits from persisting facts (Task 5.1's case-facts block is the support analogue of the scratchpad).

### 8a. The support analogue of a scratchpad
Persist case facts (Task 5.1) so a long conversation doesn't degrade into vague answers.

### 8b. Scenario-1 self-check
1. Does the support agent explore a codebase at runtime? → No; 5.4 applies to the dev workflow. The analogue is the case-facts block.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Partial fit.** A generation run over a large codebase can degrade; use a scratchpad to record the structure it discovered while planning, summarize between stages, and `/compact` if the session bloats. Crash-recovery manifests help long multi-module generation jobs.

### 9a. Scratchpad + phase summaries during generation
Record discovered structure; summarize before each stage; `/compact` when full.

### 9b. Scenario-2 self-check
1. Long generation session degrading? → Scratchpad findings + phase summaries + `/compact`.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Strong fit (conceptual mirror).** The §2 pattern *is* this scenario's architecture: the coordinator stays high-level while subagents do verbose investigation and return summaries. Phase summaries (§4) and state manifests (§5) apply to long multi-phase research — export each subagent's findings to a manifest so a long run can resume after a crash.

### 10a. Coordinator-stays-high-level, subagents summarize
Verbose source reading lives in subagents; the coordinator integrates summaries — preventing its context from degrading.

### 10b. Manifests for long research
Each subagent exports findings to a manifest; the coordinator reloads on resume.

### 10c. Scenario-3 self-check
1. Keep the coordinator's context clean? → Subagents do verbose work, return summaries.
2. Resume a long research run after a crash? → Reload subagent state manifests.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Core fit — the home scenario.** Every 5.4 idea lands: exploring an unfamiliar/legacy codebase is exactly where **degradation** appears ("typically a repo like this…" instead of the actual classes); spawn **subagents** for "find all test files" / "trace refund-flow dependencies" so verbose output stays isolated (§2); keep a **scratchpad** of discovered structure and re-read it (§3); **summarize each phase** before the next and inject it (§4); **`/compact`** when the window fills; and use **state manifests** for crash recovery on long explorations (§5).

### 11a. Subagents for specific questions
"Find all test files," "trace the refund flow" → subagents investigate, return summaries; main agent coordinates the mental model.

### 11b. Scratchpad against degradation
Record "RefundService → PaymentGateway, AuthService in `src/auth/`" to a file; re-read it so later answers cite specifics, not "typical patterns."

### 11c. Phase summaries + `/compact`
Summarize phase 1 (architecture map) before phase 2 (refactor planning); `/compact` when discovery output bloats the window.

### 11d. Crash-recovery manifests
Each exploration agent exports state to a known location; on resume the coordinator loads the manifest and injects it.

### 11e. Scenario-4 self-check
1. Agent starts saying "typically…" instead of named classes? → Context degradation; persist findings to a scratchpad and re-ground.
2. Isolate "find all test files" output? → A subagent that returns a summary.
3. Reclaim a bloated exploration window? → `/compact`.
4. Resume a long exploration after a crash? → Reload structured state manifests.
5. (Goal tie) How does this aid productivity? → The agent keeps citing real, discovered facts across a long session instead of degrading into generic guesses.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Partial fit.** A reviewer analyzing a large diff plus surrounding code can hit degradation; delegate "find all callers of this changed function" to a subagent that returns a summary (§2), keeping the reviewer's context focused. Per-PR runs are short, so scratchpads/manifests matter less than in interactive exploration.

### 12a. Subagent for blast-radius queries
Delegate "find callers of the changed function" so verbose results don't crowd the reviewer's context.

### 12b. Scenario-5 self-check
1. Reviewer needs all callers of a changed function? → A subagent returns a summary; reviewer stays focused.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Partial fit.** A very long single document can cause degradation; persist already-extracted records to a scratchpad/state file and process in chunks with phase summaries so earlier fields aren't lost (overlaps Task 5.1). State manifests help resume a long batch extraction after a crash.

### 13a. Persist partial extractions + manifests
Record extracted records to a file; use a manifest so a long batch resumes after a crash.

### 13b. Scenario-6 self-check
1. Very long document degrading extraction? → Chunk + scratchpad of extracted records + phase summaries.
2. Resume a long batch after a crash? → State manifest reload.

---

### Sources verified against current Claude Code docs (June 2026)
- *Subagents / context management* (code.claude.com) — subagents start with clean context, do focused work, and return summaries (isolating verbose output); `/compact` compresses conversation history to reclaim context during long sessions; project-root CLAUDE.md is re-read after compaction.
- Context-degradation symptoms ("typical patterns" vs specific discovered classes), scratchpad files, phase-summary injection, and structured state manifests for crash recovery come from the task statement and Claude Code exploration practice. These are architecture patterns; verify any command/subagent specifics against current docs.
