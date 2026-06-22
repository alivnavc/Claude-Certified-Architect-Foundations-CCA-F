# Task 3.1 — Configuring CLAUDE.md (Hierarchy, Scoping, Modular Organization)
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics (every "Knowledge of" / "Skills in" bullet). Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **CLAUDE.md is a plain-markdown file Claude Code reads at the start of every session and treats as standing instructions. There are layers — *user* (just you, every project), *project* (committed, the whole team), *directory* (a subfolder) — and the rule is: put each instruction at the layer whose audience and scope it matches. Most "Claude ignores my rule" and "my teammate doesn't get it" bugs are an instruction sitting at the wrong layer.**

Everything here is: which layer, how to keep it modular (`@import`, `.claude/rules/`), and how to diagnose loading (`/memory`).

---

## 1. The hierarchy (where files live, who gets them)

| Layer | Path | Shared? | Loads |
|---|---|---|---|
| **User** | `~/.claude/CLAUDE.md` | **No** — your machine only, every project | Always |
| **Project** | `./CLAUDE.md` or `./.claude/CLAUDE.md` (repo root) | **Yes** — committed to git, whole team | Always |
| **Directory** | `subdir/CLAUDE.md` | Yes (committed) | **On demand** — when Claude touches a file in that subdir |
| (Local) | `./CLAUDE.local.md` | No — gitignored, this project | Always |

**Precedence: more specific wins.** Project overrides user; managed-policy (org) is highest and can't be excluded. So if user-level says "4-space indent" and project says "2-space," the project wins in that repo. Parent-directory files load in full at launch; child-directory files load only when Claude reads a file there (keeps a 50-package monorepo from bloating context).

---

## 2. User-level is private (the most-tested gotcha)

`~/.claude/CLAUDE.md` is **not** version-controlled — it lives on your workstation and follows *you* across projects, but **teammates never see it**. So anything your team needs ("all API routes return JSON") must go in **project** CLAUDE.md, not user-level. The classic bug: a lead puts a team convention in their own `~/.claude/CLAUDE.md`, it works for them, and a new hire's Claude behaves differently because that instruction was never shared. Fix: move it to project scope.

---

## 3. `@import` for modularity

CLAUDE.md can pull in other files with `@path/to/file` (relative resolves from the importing file; absolute and `~/` work too). Use it to keep CLAUDE.md a lean *roadmap* that imports the standards each package actually needs:

```markdown
# Backend package conventions
@../docs/api-conventions.md
@../docs/error-handling.md
```

Details: recursive imports up to depth 5; the first import from an external location triggers a one-time approval; import lines inside code blocks aren't evaluated. **Caveat:** imports load at launch, so they help *organization*, not context size — an imported file costs the same tokens as inline.

---

## 4. `.claude/rules/` for topic-specific files

Instead of one monolithic CLAUDE.md, split rules into focused files under `.claude/rules/` — `testing.md`, `api-conventions.md`, `deployment.md`. Every `.md` there loads as project memory automatically (no `@import` needed), is committed (whole team benefits), and is easier to maintain ("new topic → new file"). (Path-scoping these via frontmatter is Task 3.3.) `.claude/rules/*.md` and `.claude/CLAUDE.md` share the same priority.

---

## 5. `/memory` — diagnose what's loaded

The `/memory` command shows **which memory files are currently loaded** (and lets you browse/edit auto-memory). It's the first move when behavior is inconsistent across sessions or differs between teammates: run `/memory` to see whether the instruction you expect is actually loaded and from which layer. Keep the main file lean — files over ~200 lines consume more context and reduce adherence; the project-root CLAUDE.md is re-read after `/compact`, but nested ones reload only on next access.

---

## 6. Anti-patterns (with *why*)
- **Team convention in user-level CLAUDE.md.** Teammates never get it (it's not committed). Use project scope.
- **One giant CLAUDE.md.** Everything competes for attention even when irrelevant; split into `.claude/rules/` and/or `@import`.
- **Expecting `@import` to save context.** Imports load at launch — same token cost; use path-scoped rules (3.3) to actually trim.
- **Secrets in CLAUDE.md.** It's committed plain text; never put credentials there.
- **Bloated parent-directory file in a monorepo.** Loads every session for everyone; push package-specifics to directory-level or path-scoped rules.
- **Debugging inconsistent behavior by guessing.** Run `/memory` to see what's actually loaded.

---

## 7. Self-check (core mechanics)
1. Three main layers + paths? → User (`~/.claude/CLAUDE.md`), project (`./CLAUDE.md` or `./.claude/CLAUDE.md`), directory (`subdir/CLAUDE.md`).
2. User vs project: conflict winner? → Project (more specific wins).
3. Why doesn't a teammate get your user-level rule? → User-level isn't version-controlled; it's private to your machine.
4. Keep CLAUDE.md modular? → `@import` external files and/or `.claude/rules/` topic files.
5. Does `@import` reduce context? → No — imports load at launch (organization only).
6. Diagnose inconsistent behavior? → `/memory` to see which files are loaded.
7. (Ties to §0) One sentence? → *Put each instruction at the layer whose audience and scope match it.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7). CLAUDE.md governs *Claude Code*, so here it shapes the workflow of the **team building** this agent, not the deployed agent's runtime.

**The key link to everything above:** **Indirect fit.** The agent's repo gets a **project** CLAUDE.md documenting the tool contracts, escalation policy, and test conventions so every engineer's Claude Code builds it consistently. Personal editor preferences stay in **user** scope so they aren't forced on teammates (§2).

### 8a. Project CLAUDE.md for the agent's repo
Document the four MCP tools' contracts and the refund-policy rules once, at project scope, so the whole team's Claude Code shares them.

### 8b. Scenario-1 self-check
1. Where do the agent's build conventions live? → Project CLAUDE.md (committed).
2. Your personal indent preference? → User-level (private).

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Strong fit.** A generation pipeline needs tight, consistent conventions, and a sprawling rulebook hurts. Split standards into `.claude/rules/` (`testing.md`, `code-style.md`, `commit-conventions.md`, §4) so generated code follows house style without one bloated file. Project scope means every contributor's pipeline behaves identically.

### 9a. Modular rules for generation standards
`.claude/rules/testing.md` + `code-style.md` keep each concern focused; all load automatically as project memory.

### 9b. Scenario-2 self-check
1. Avoid a monolithic standards file? → Split into `.claude/rules/` topic files.
2. Why project scope? → Identical generation behavior across the team.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Indirect/partial fit.** Like S1, CLAUDE.md governs the team building the system. One useful angle: subagents start with fresh context, so the **project** CLAUDE.md is what gives every spawned Claude Code instance the shared citation/format conventions it needs — a place to encode "always attach source URLs" once for all of them.

### 10a. Shared conventions for fresh-context subagents
Project CLAUDE.md (or `.claude/rules/citations.md`) ensures each freshly-spawned instance knows the citation format without re-explaining.

### 10b. Scenario-3 self-check
1. How do fresh-context subagents get shared conventions? → From project CLAUDE.md / rules, loaded at session start.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Core fit — the home scenario.** Every 3.1 idea lands: a **project** CLAUDE.md for team standards; **user** CLAUDE.md for personal preferences (and the §2 diagnosis when a new teammate's behavior differs — their rule is user-level, not shared); **directory-level** CLAUDE.md per package in a monorepo (loaded on demand); `@import` to pull each package's relevant standards (§3); split into `.claude/rules/` as it grows (§4); `/memory` to debug inconsistency (§5).

### 11a. Diagnose the new-teammate bug
New hire's Claude misbehaves → the missing rule lives in someone's `~/.claude/CLAUDE.md` (private). Move it to project scope so everyone gets it. Confirm with `/memory`.

### 11b. Monorepo directory scoping + imports
Each package gets a `CLAUDE.md` that `@import`s only the standards relevant to its domain (per the maintainer's knowledge); these load on demand when Claude works in that package — no global bloat.

### 11c. Modularize a bloated file
A 600-line CLAUDE.md → split into `.claude/rules/testing.md`, `api-conventions.md`, `deployment.md`. Lean, focused, auto-loaded.

### 11d. Scenario-4 self-check
1. New teammate not getting instructions? → They're in user-level, not project; move to project, verify with `/memory`.
2. Different standards per monorepo package? → Directory-level CLAUDE.md + `@import` the relevant standards.
3. Fix a 600-line CLAUDE.md? → Split into `.claude/rules/` topic files.
4. Confirm what's loaded? → `/memory`.
5. (Goal tie) Why does correct scoping aid productivity? → Each context gets exactly the right, lean instructions — fewer corrections, consistent behavior.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Strong fit.** CLAUDE.md is *how you give CI-invoked Claude its project context* (Task 3.6): review criteria, testing standards, fixture conventions all live in committed CLAUDE.md / `.claude/rules/` so the headless reviewer judges against real house rules — which directly reduces false positives. Because it's project scope and committed, the CI run and every developer review use the same standards.

### 12a. Review criteria in CLAUDE.md
Document what counts as a real issue, severity bands, and known false-positive patterns in CLAUDE.md so the reviewer applies them — fewer noisy flags.

### 12b. Note on `--bare`
A hermetic CI run with `--bare` *skips* CLAUDE.md/auto-memory discovery for reproducibility; if you want the reviewer to use CLAUDE.md, don't use `--bare` (or pass context explicitly). Know the trade-off (detail in Task 3.6).

### 12c. Scenario-5 self-check
1. How does CI Claude get review standards? → Committed project CLAUDE.md / `.claude/rules/`.
2. How does that cut false positives? → It judges against real criteria, not assumptions.
3. What does `--bare` do to CLAUDE.md? → Skips it for reproducibility — opt out if you need that context.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Partial fit.** CLAUDE.md governs the team building the extractor. Useful angle: document the target schemas, edge-case rules, and validation conventions in project CLAUDE.md / `.claude/rules/extraction.md` so every engineer's Claude Code generates extraction code consistently and against the real schema — supporting accuracy at the build layer.

### 13a. Schema + edge-case conventions in rules
`.claude/rules/extraction.md` captures the output schema and how to handle nulls/missing fields, so generated extractors follow them.

### 13b. Scenario-6 self-check
1. Where do schema/edge-case conventions live for the dev team? → Project CLAUDE.md / `.claude/rules/`.
2. Why not user-level? → Team needs them; user-level isn't shared.

---

### Sources verified against current Claude Code docs (June 2026)
- *How Claude remembers your project / Memory* (code.claude.com) — layers: user `~/.claude/CLAUDE.md` (private, all projects), project `./CLAUDE.md` or `./.claude/CLAUDE.md` (committed, team), directory `subdir/CLAUDE.md` (on-demand), `CLAUDE.local.md` (gitignored); precedence "more specific wins," managed policy highest; `@path` imports (relative/absolute, recursive depth 5, first external import prompts approval, load at launch so no context savings); `.claude/rules/*.md` auto-load as project memory and share priority with `.claude/CLAUDE.md`; `/memory` shows loaded files and edits auto-memory; project-root CLAUDE.md re-read after `/compact`; files >200 lines reduce adherence.
- Memory and rules UX ships fast — re-verify exact paths/precedence against current docs.
