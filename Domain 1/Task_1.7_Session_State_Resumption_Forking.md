# Task 1.7 — Session State: Resumption & Fork-Based Exploration
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **A session is the saved conversation history — every prompt, tool call, result, and response. Three moves: *resume* to keep working in the same thread when its context is still true; *fork* to branch a copy from a shared baseline so you can try two directions without interference; *start fresh with a summary* when the old context has gone stale. The decision is always one question: "is the prior context still valid?"**

Resume = same thread, context trusted. Fork = two threads from one baseline, compare. Fresh+summary = throw away stale detail, carry forward only the conclusions.

---

## 1. What a session is

A **session** is the conversation history the SDK accumulates while the agent works, written to disk automatically (as `.jsonl` files under `~/.claude/projects/<encoded-cwd>/`, or `$CLAUDE_CONFIG_DIR`). Returning to a session means the agent has its full prior context: files it read, analysis it did, decisions it made. Important caveat: **sessions persist the *conversation*, not the *filesystem*.** If the agent edited files, those edits are real and not undone by returning to an earlier session — for file snapshots you use *file checkpointing*, a separate feature.

Each session has a unique ID, emitted in the `init` system message at start. You capture that ID to return later.

---

## 2. Resume (and continue)

Both pick up an existing session and **add to it**:

| Move | How it finds the session | Use when |
|---|---|---|
| **continue** | Most recent session in the current directory (you track nothing) | Single conversation at a time |
| **resume** | A specific session ID you saved | Multiple sessions (e.g., one per user), or returning to a non-latest one |

```python
# Resume a specific prior thread by ID — full context comes back
options = ClaudeAgentOptions(resume="session-abc-123")
async for msg in query(prompt="Continue the investigation", options=options):
    ...
```

Resume is right when **the context is still valid**: same investigation, files unchanged (or you'll tell it what changed), problem unchanged, session recent.

---

## 3. Fork (branch from a shared baseline)

**Forking** creates a *new* session that starts with a **copy** of the original's history, then diverges. The original's ID and history stay unchanged; you end up with two independent threads from the same starting point. Forking branches the *conversation*, not the filesystem (file edits in a fork are real and visible to anything in that directory).

```python
# Fork: same baseline, new independent thread
options = ClaudeAgentOptions(resume="session-abc-123", fork_session=True)
async for msg in query(prompt="Try the OAuth2 approach instead", options=options):
    ...   # new session id; original session-abc-123 untouched
```

Use fork to **explore divergent approaches from one analysis**: you spent 30 minutes understanding an auth module, now you want to try approach A *and* approach B without redoing the analysis or letting them interfere. Fork twice from the baseline; compare results. (TypeScript: `forkSession: true`. To branch from a *specific message* rather than the full history, the SDK exposes `resume_at`/`resumeSessionAt` with a message UUID, or `fork_session(..., up_to_message_id=...)` — fast-moving, re-verify.)

---

## 4. Start fresh with a summary (when context is stale)

Resuming is **not** always right. If the codebase changed significantly, or the prior analysis is now wrong, dragging that stale history forward poisons the new run (the agent cites line 145 when the code moved to line 178). Instead, **start a fresh session and hand it a structured summary** of just the still-valid conclusions. You keep the signal, drop the stale detail.

A middle option when context is *mostly* valid: **resume, but tell it what changed** — e.g., *"Since last session, PR #540 and #542 merged; re-read `src/services/` and `src/cache/` before continuing."* This triggers **targeted re-analysis** of only the changed parts rather than a full redo.

---

## 5. The decision table

| Situation | Move |
|---|---|
| Same investigation, context still true, recent | **Resume** (or continue) |
| Mostly valid, a few files changed | **Resume + tell it what changed** (targeted re-analysis) |
| Want to compare two approaches from one analysis | **Fork** from the shared baseline |
| Prior results stale / codebase moved a lot | **Fresh session + structured summary** |
| Each run should be independent by design (e.g., CI per-PR) | **No session at all** (intentionally stateless) |

That last row is a deliberate exam trap: recognizing when *not* to use sessions is as important as using them.

---

## 6. Anti-patterns (with *why*)
- **Resuming stale context.** The agent reasons over outdated files/line numbers and gets things wrong. Start fresh + summary instead.
- **Resuming without flagging changes.** If files moved since last time and you don't say so, the agent trusts stale positions. Tell it what changed.
- **Re-running analysis to try a second approach.** Wasteful and may drift; **fork** from the shared baseline instead.
- **Expecting a session to undo file edits.** Sessions persist conversation, not the filesystem; use checkpointing for file revert.
- **Forcing sessions where statelessness is the design.** In CI you *want* each PR reviewed independently; persistent state would leak context between PRs.
- **Wrong-directory resume.** Resume looks under the encoded cwd; running from a different directory silently starts fresh. Match the working directory.

---

## 7. Self-check (core mechanics)
1. The one question that drives the choice? → **Is the prior context still valid?**
2. Resume vs fork? → Resume = same thread; fork = new thread from a copied baseline.
3. Compare two refactor approaches from one analysis? → **Fork** (twice) from the baseline.
4. Codebase changed a lot since last run? → **Fresh session + structured summary.**
5. A few files changed but most context holds? → **Resume + tell it what changed** (targeted re-analysis).
6. Do sessions revert file edits? → **No** — conversation only; use file checkpointing.
7. (Ties to §0) When is "no session" correct? → When runs must be independent by design (CI per-PR).

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** An agent resolving support cases via four backend tools (see Task 1.4 §7).

**The key link to everything above:** **Partial fit.** A single support contact is usually one session; the relevant move is **resume per customer/ticket** so a returning customer's thread carries prior context (one session per user is exactly why you `resume` by ID rather than `continue`). When a ticket reopens days later but account state changed, that's the **resume + tell-it-what-changed** case. Forking rarely applies to a live support chat.

### 8a. One session per ticket, resumed by ID
Save each ticket's session ID; on the customer's return, `resume` that ID so the agent recalls the case without re-asking — directly supporting first-contact (now first-*thread*) resolution.

### 8b. Stale account state
If billing changed since the last contact, resume but note it ("balance updated, re-check `get_customer`") for targeted re-analysis.

### 8c. Scenario-1 self-check
1. continue or resume for per-customer threads? → **Resume** by saved ID (multiple users).
2. Account changed since last contact? → Resume + flag the change.
3. Does forking help a live chat? → Rarely; it's for comparing approaches, not serving one customer.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent producing code in stages (see Task 1.4 §8).

**The key link to everything above:** **Partial fit.** A single generation run is one session that already takes many internal turns (no session management needed mid-run). Sessions matter when you **resume** an interrupted build, or **fork** to try two implementations of the same spec from a shared plan — generate plan once, fork, build variant A in one branch and variant B in the other, compare.

### 9a. Fork to compare implementations
After planning, `fork_session=True` twice to build two designs from the identical plan baseline; the original plan session stays clean.

### 9b. Scenario-2 self-check
1. Try two implementations of one spec? → **Fork** from the shared plan.
2. Why not re-plan for the second? → Wasteful and may drift; fork preserves the baseline.
3. Mid-run, do you manage sessions? → No — one `query()` already takes all needed turns.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research, then synthesizing (see Task 1.4 §9).

**The key link to everything above:** **Strong fit.** Deep research is multi-hour and resumable: **resume** to continue an investigation when the user returns. **Fork** to explore divergent hypotheses from a shared evidence baseline (pursue theory A and theory B from the same gathered sources without interference). When earlier findings are superseded by fresher sources, **fresh + summary** carries forward only the still-valid conclusions.

### 10a. Resume long investigations
Save the coordinator's session ID; resume to pick up a multi-hour research thread with all gathered evidence intact.

### 10b. Fork competing hypotheses
From the shared evidence baseline, fork to chase two interpretations in parallel, then compare — without either branch polluting the other.

### 10c. Scenario-3 self-check
1. User returns to a long research task? → **Resume** the saved session.
2. Two hypotheses from the same evidence? → **Fork** from the baseline.
3. Old findings now outdated? → **Fresh + summary** of valid conclusions.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Bash, Grep, Glob; integrates MCP; explores codebases, handles legacy systems, generates boilerplate, automates chores.*

**What this scenario is even about (plain English):** A coding assistant working across a real repo over days (see Task 1.4 §10).

**The key link to everything above:** **Core fit — the home scenario for 1.7.** Multi-day codebase investigation is exactly what sessions are for: **resume** named exploration sessions across days; **fork** to compare two refactoring/testing strategies from one analysis baseline; **tell a resumed session which files changed** for targeted re-analysis; **fresh + summary** when prior results are stale. Every §0 move lands here.

### 11a. Resume across days
Name and save the exploration session; `resume` it tomorrow so the agent recalls everything it learned about the codebase — no re-explaining.

### 11b. Fork to compare refactors
Spent 30 minutes understanding the auth module? Fork the baseline twice and try refactor A vs refactor B simultaneously; keep whichever wins, original analysis intact.

### 11c. Inform a resumed session of changes
"Since last session, 8 files changed (PR #540, #542 merged); re-read `src/services/` and `src/cache/` first." → targeted re-analysis instead of a full redo or stale reasoning.

### 11d. Fresh + summary when stale
If the codebase moved substantially, start fresh and hand over a summary of the still-true conclusions rather than dragging outdated line numbers along.

### 11e. Scenario-4 self-check
1. Continue an exploration tomorrow? → **Resume** the named session.
2. Compare two refactor strategies from one analysis? → **Fork** the baseline.
3. Resume after files changed? → Resume **and tell it which files changed**.
4. Codebase moved a lot? → **Fresh + summary**, not resume.
5. (Goal tie) Why does forking beat re-analyzing for comparisons? → It reuses the expensive shared analysis once and keeps the branches independent.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in your pipeline (see Task 1.4 §11).

**The key link to everything above:** **Deliberate contrast case — the "no session" answer.** Each `claude -p` run in CI is intentionally **stateless**: every PR must be reviewed independently, with no memory bleeding from a previous PR. Recognizing that **the correct session strategy here is *no persistent session*** is a likely exam trap — the skill being tested is knowing when statelessness is the right design, not reflexively reaching for resume/fork.

### 12a. Why stateless is correct
If PR #2's review inherited PR #1's context, you'd get phantom issues and inconsistent verdicts. Independence guarantees each review reflects only that PR's diff — which also reduces false positives.

### 12b. Scenario-5 self-check
1. Session strategy for CI per-PR review? → **None** — intentionally stateless.
2. Why not resume across PRs? → Context bleed → inconsistent, false-positive-prone reviews.
3. What's the exam trap? → Assuming you must always use sessions; here you must *not*.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Task 1.4 §12).

**The key link to everything above:** **Partial fit.** Per-document extraction is typically a stateless, repeatable run (like CI — each document independent), so often **no session** is needed. Where sessions help: **forking** to A/B two extraction-schema or prompt variants over the *same* document set from a shared baseline, then comparing accuracy. Resume matters only for genuinely long, interruptible batch jobs.

### 13a. Fork to A/B schema variants
From one loaded document baseline, fork to run schema/prompt variant A and variant B; compare validation pass rates to pick the better extractor.

### 13b. Mostly stateless by design
Independent documents → independent runs, for the same reliability reason as CI. Don't carry one document's context into the next.

### 13c. Scenario-6 self-check
1. Default session strategy per document? → Often **none** (independent runs).
2. How to compare two extraction schemas fairly? → **Fork** from the same document baseline.
3. (Goal tie) How does statelessness support "high accuracy"? → No cross-document context bleed, so each extraction reflects only its own input.

---

### Sources verified against current Anthropic docs (June 2026)
- *Work with sessions* (platform.claude.com, code.claude.com) — `continue` (most recent in cwd) vs `resume` (specific ID); `fork_session`/`forkSession` copies history into a new session ID, original unchanged; sessions persist conversation not filesystem (use file checkpointing for files); stored as `.jsonl` under `~/.claude/projects/<encoded-cwd>/` (or `$CLAUDE_CONFIG_DIR`); resume looks up by encoded cwd, so a mismatched directory silently starts fresh.
- *Session browser cookbook* / SDK reference — `list_sessions`, `fork_session(..., up_to_message_id=...)`, message UUIDs as fork/resume points; `resume_at`/`resumeSessionAt` for resuming up to a specific message (some pieces are in flux across V1/V2 — re-verify).
- CLI: `claude --resume <id>`, `--fork-session`, `--resume-session-at <msg>`.
- Session APIs move fast (an evolving V2 session interface, open issues around `resume_at` + fork) — re-verify exact flag/option names against current docs before production use.
