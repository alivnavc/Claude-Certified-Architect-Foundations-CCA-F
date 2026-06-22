# Task 2.5 — Selecting & Applying Built-in Tools (Read, Write, Edit, Bash, Grep, Glob)
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–4 are the mechanics. Section 5 = anti-patterns, Section 6 = self-check. Then one section per scenario (7–12) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **Each built-in tool has one job: `Grep` searches file *contents*, `Glob` matches file *paths*, `Read` loads a whole file, `Write` replaces a whole file, `Edit` changes a *uniquely-identifiable* slice of a file. Pick by what you're actually doing, and build understanding *incrementally* — search to find the few relevant files, then read those — instead of reading everything up front.**

Everything here is matching the operation to the right tool, plus one fallback rule (`Edit` needs unique anchor text; when it can't find one, drop to `Read` + `Write`).

---

## 1. The six tools and their one job each

| Tool | Operates on | Use when |
|---|---|---|
| **Grep** | File *contents* (regex/text) | Find where a function is called, locate an error string, find imports |
| **Glob** | File *paths* (name/extension patterns) | Find files by pattern, e.g. `**/*.test.tsx` |
| **Read** | One whole file | Load full contents to understand it |
| **Write** | One whole file | Create a new file, or replace a file's entire contents |
| **Edit** | A unique substring within a file | Make a targeted change to existing code |
| **Bash** | The shell | Run commands (tests, git, build) — use sparingly, it's powerful/dangerous |

**Grep vs Glob is the most-tested pair:** Grep looks *inside* files (content), Glob looks *at* filenames (paths). "Find all callers of `processPayment`" → Grep. "Find all test files" → Glob `**/*.test.tsx`.

---

## 2. Edit's unique-match rule and the Read+Write fallback

`Edit` works by finding a piece of text and replacing it — and that text **must be unique** in the file. If the anchor string appears more than once, `Edit` can't tell which occurrence you mean and **fails**. Two ways out:

1. **Expand the anchor** — include enough surrounding lines to make the match unique (preferred for a single targeted change).
2. **Read + Write fallback** — when you can't get a unique anchor (e.g., the same line repeats many times, or you're rewriting large chunks), `Read` the whole file, transform the content, and `Write` it back entirely. Reliable because it doesn't depend on uniqueness.

```
Edit(file, old="count += 1", new="count += 2")   # FAILS if "count += 1" appears 3×
# Fallback:
Read(file) -> full text -> change the intended occurrence -> Write(file, new_full_text)
```

---

## 3. Incremental codebase understanding (the core skill)

Don't read every file up front — it wastes context and attention. Build understanding in steps:

1. **Grep for entry points** — find `main`, route definitions, the error message, the function name.
2. **Read** those specific files to understand them.
3. **Follow imports** — Grep/Read the modules they pull in, tracing the flow.

This "search → read → follow" loop (the same incremental idea as Task 1.6's decomposition) keeps you focused on the relevant slice of a large codebase.

---

## 4. Tracing usage across wrapper modules

To find everywhere a function is used when it's re-exported through wrappers (file A defines it, B re-exports it, C imports it under a new name), one Grep for the original name misses the aliased calls. The method: **first identify all the exported names** (the original plus each re-export/alias), **then Grep for each name** across the codebase. Two passes — enumerate names, then search each — catch usages a single search would miss.

---

## 5. Anti-patterns (with *why*)
- **Glob to search contents / Grep to find filenames.** Wrong tool: Glob matches paths, Grep matches contents. They don't substitute.
- **Reading the whole repo up front.** Burns context and attention; Grep to the relevant files first, then Read.
- **Retrying a failing Edit on non-unique text.** It will keep failing; expand the anchor or use Read+Write.
- **One Grep for a function traced through wrappers.** Misses aliased re-exports; enumerate names first, then search each.
- **Reaching for Bash when a dedicated tool exists.** Use Read/Write/Edit/Grep/Glob for file ops; reserve Bash for actual commands (and gate it — Task 1.5).

---

## 6. Self-check (core mechanics)
1. Grep vs Glob? → Grep searches file **contents**; Glob matches file **paths/names**.
2. Find all callers of a function? → **Grep** the function name.
3. Find all `**/*.test.tsx`? → **Glob**.
4. Why does Edit fail sometimes? → Its anchor text isn't **unique** in the file.
5. Fallback when Edit can't find a unique anchor? → **Read** the file, transform, **Write** it back.
6. How to understand a big codebase? → Incrementally: Grep entry points → Read them → follow imports.
7. Trace a function through wrapper re-exports? → Enumerate all exported names, then Grep each.

---

## 7. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Minimal fit.** This agent works through *MCP* tools, not file-system built-ins — so 2.5 mostly applies to the *developers building and maintaining* the agent (using Grep/Read to navigate the agent's own codebase). The exam point: built-ins are for filesystem/code work; a support agent's runtime toolset is MCP, not Read/Grep.

### 7a. Built-ins are for building, not serving
The team uses Grep/Read to develop the support agent; the deployed agent itself uses MCP tools. Don't conflate the two layers.

### 7b. Scenario-1 self-check
1. Does the support agent use Grep at runtime? → No — it uses MCP tools; built-ins are for developing it.
2. Where do built-ins fit here? → The developers' workflow on the agent's codebase.

---

## 8. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Core fit.** This agent *lives* in the built-ins: `Write` to create new files, `Edit` for targeted changes, `Read` to inspect, `Bash` to run tests, `Grep`/`Glob` to locate code. The Edit-uniqueness rule (§2) bites constantly — generated code has repeated patterns, so Edit often needs the Read+Write fallback.

### 8a. Write vs Edit by situation
New module → `Write`. Change one unique line in an existing file → `Edit`. Repeated/ambiguous target → Read+Write fallback.

### 8b. Run tests with Bash
The "test" stage is a `Bash` call; gate destructive commands (Task 1.5).

### 8c. Scenario-2 self-check
1. Create a brand-new file? → `Write`.
2. Edit fails because the line repeats? → Read the file, change it, Write it back.
3. Run the test suite? → `Bash`.

---

## 9. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Minimal fit.** Research runs over web/document MCP tools, not the local filesystem — so built-ins are largely out of scope, except when analysis subagents work over *local* document files (then `Glob` finds the files, `Read` loads them). The exam point: match the tool to the data location — web/data sources → MCP; local files → built-ins.

### 9a. Local docs → Glob + Read
If the corpus is local files, `Glob` to enumerate them and `Read` to load; if it's the web, that's MCP search, not built-ins.

### 9b. Scenario-3 self-check
1. Corpus is on the web — built-ins? → No; that's MCP search tools.
2. Corpus is local files? → `Glob` to find, `Read` to load.

---

## 10. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Core fit — the home scenario.** Every 2.5 skill is named in the scenario. "Explore unfamiliar codebases" *is* §3 incremental understanding: Grep entry points → Read → follow imports, not read-everything. "Understand legacy systems" with wrappers is §4: enumerate exports, then Grep each. Editing legacy code triggers the §2 Edit-uniqueness fallback constantly.

### 10a. Explore incrementally
Grep for the route/handler/error → Read those files → Grep their imports → Read those. Trace the flow without loading the whole repo.

### 10b. Trace through wrappers
Find all exported names (original + re-exports/aliases), then Grep each name to find every caller — single-search misses aliases.

### 10c. Edit fallback in legacy code
Legacy files have repeated patterns; when `Edit` can't find a unique anchor, Read+Write the file.

### 10d. Glob for file patterns
`Glob **/*.test.tsx` to find tests, `**/*.config.js` for configs — path patterns, not content.

### 10e. Scenario-4 self-check
1. First move on an unfamiliar codebase? → **Grep** entry points (not read everything).
2. Trace a function re-exported through wrappers? → Enumerate exported names, then Grep each.
3. Edit fails on a repeated line in legacy code? → Read + Write fallback.
4. Find all test files? → `Glob **/*.test.tsx`.
5. (Goal tie) Why incremental over read-all? → Keeps focus/context on the relevant slice — faster and more accurate on big/legacy code.

---

## 11. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Strong fit (read-only slice).** A reviewer should use **read-only** built-ins: `Grep` to find related code the diff touches, `Read` to inspect changed files and their callers, `Glob` to locate related tests — but **no `Write`/`Edit`/destructive `Bash`** (least privilege, Task 2.3). Incremental understanding (§3) of the diff's blast radius is exactly what reduces false positives: checking whether a changed function's callers actually break, rather than flagging in isolation.

### 11a. Read-only investigation of a diff
`Grep` the changed symbols to find callers, `Read` those files to judge real impact — context that cuts false positives.

### 11b. No write tools
The reviewer gets Read/Grep/Glob only; it cannot modify the repo.

### 11c. Scenario-5 self-check
1. Tools for a CI reviewer? → Read, Grep, Glob (read-only); no Write/Edit.
2. How does Grep+Read cut false positives? → It checks the diff's real callers/impact instead of flagging in isolation.

---

## 12. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Moderate fit.** When the documents are local files, built-ins do the I/O: `Glob` to enumerate the input files (e.g., `**/*.pdf`), `Read` to load each, `Write` to emit the validated records. `Edit` is rarely the right tool for generated output — you're writing whole records, so `Write` (or Read+Write) fits, not targeted Edits.

### 12a. Glob + Read in, Write out
`Glob` the input set, `Read` each document, extract+validate, `Write` the output record file. Whole-file writes, not Edits.

### 12b. Scenario-6 self-check
1. Enumerate all input PDFs? → `Glob **/*.pdf`.
2. Emit a validated record file? → `Write` (whole file), not `Edit`.
3. Why not Edit for output? → You're producing whole records; Edit is for unique-anchor changes to existing files.

---

### Sources verified against current Anthropic docs (June 2026)
- *Claude Code settings / tools* and *Agent SDK built-in tools* (code.claude.com, platform.claude.com) — `Grep` (content search), `Glob` (path/name patterns), `Read`/`Write` (whole-file), `Edit` (unique-text match; fails on non-unique anchors), `Bash` (shell). The Edit-uniqueness constraint and Read+Write fallback are documented behavior.
- Incremental "search → read → follow imports" exploration and wrapper-tracing (enumerate exports, then search each) reflect the task statement and Claude Code usage guidance.
- Built-in tool names/semantics are stable but evolve; verify exact tool names and Edit semantics against the current Claude Code docs when relying on edge behavior.
