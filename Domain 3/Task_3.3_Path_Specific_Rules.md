# Task 3.3 — Path-Specific Rules for Conditional Convention Loading
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–4 are the mechanics. Section 5 = anti-patterns, Section 6 = self-check. Then one section per scenario (7–12) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **A rule file in `.claude/rules/` can carry a `paths:` frontmatter field — a list of glob patterns. The rule then loads *only when Claude is working on a file that matches a pattern*. So conventions live where they're relevant: Terraform rules load only on Terraform files, test rules only on test files — and because globs match by pattern, not folder, they catch files of a type wherever they're scattered.**

Everything here is: conditional loading by glob, the token/relevance savings it buys, and why glob-pattern rules beat directory-level CLAUDE.md for cross-cutting conventions.

---

## 1. The mechanism: `paths:` frontmatter

A file in `.claude/rules/` normally loads as project memory **unconditionally**. Add a `paths:` field (a YAML list of glob patterns) and it becomes **conditional** — it loads only when Claude edits/reads a matching file:

```markdown
---
paths:
  - "terraform/**/*"
---
# Terraform conventions
- Always pin provider versions
- Use remote state, never local
```

A rule **without** a `paths` field loads everywhere (so a rule "firing everywhere" usually just lacks `paths`). Patterns match against absolute file paths with standard glob syntax; brace expansion (`src/**/*.{ts,tsx}`) and symlinks are supported.

---

## 2. Why conditional loading helps (relevance + tokens)

Loading every convention into every session does two harms: it **wastes tokens** (permanent context cost) and it **dilutes relevance** (Terraform rules competing for attention while you edit React). Path-scoped rules fix both — the React conventions are present only when you touch `.tsx`, the Terraform conventions only when you touch `.tf`. This is the one mechanism that actually *trims* context (recall `@import` does not — Task 3.1 §3).

---

## 3. Glob rules vs directory-level CLAUDE.md (the key choice)

Both can scope conventions, but they differ in *how* they target:

| | Directory-level CLAUDE.md | Path-scoped rule (`paths:` glob) |
|---|---|---|
| Targets by | **Location** (a subfolder) | **Pattern** (file type/name, any folder) |
| Best for | Conventions for one cohesive directory (a package) | Conventions for a file *type* spread across many directories |
| Example | `backend/CLAUDE.md` for the backend package | `**/*.test.tsx` for all tests, wherever they live |

**The decisive case:** test files (or `*.stories.tsx`, or migration scripts) are scattered all over the tree. A directory CLAUDE.md can't cover them without copies in every folder. A single path-scoped rule `**/*.test.tsx` covers them all at once. **Cross-cutting by type → glob rule; cohesive by location → directory CLAUDE.md.**

---

## 4. Writing the patterns

- Type anywhere: `**/*.test.tsx`, `**/*.tf`
- Directory subtree: `terraform/**/*`, `src/api/**/*`
- Multiple extensions (brace expansion): `**/*.{ts,tsx}`, `{src,lib}/**/*.ts`
- Test simple patterns first (`src/**/*.ts`) before complex ones; if a rule isn't applying, the glob is the usual culprit.

User-level path rules can live in `~/.claude/rules/` too (load before project rules, so project rules win conflicts).

---

## 5. Anti-patterns (with *why*)
- **One big always-on CLAUDE.md for everything.** Every convention loads always → wasted tokens + diluted relevance. Path-scope the type-specific ones.
- **Directory CLAUDE.md for a scattered file type.** You'd need copies in every folder; use one glob rule instead.
- **Forgetting `paths:`.** The rule loads everywhere (the usual "why is this firing on every file?" cause).
- **Over-broad glob.** `**/*` scopes to everything (no savings); target the actual type/subtree.
- **Assuming `@import` trims context.** It doesn't (loads at launch); path-scoped rules are the tool that does.

---

## 6. Self-check (core mechanics)
1. What makes a rule conditional? → A `paths:` glob list in its YAML frontmatter.
2. Rule with no `paths` field? → Loads unconditionally (everywhere).
3. Convention for all test files scattered across the repo? → A path-scoped rule `**/*.test.tsx`.
4. Convention for one cohesive package? → Directory-level CLAUDE.md (or a subtree glob).
5. Why path rules over a monolithic CLAUDE.md? → They load only when relevant — saving tokens and reducing irrelevant context.
6. (Ties to §0) One sentence? → *Scope conventions by glob so they load exactly when the matching file is in play.*

---

## 7. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7). Path rules govern Claude Code building the agent.

**The key link to everything above:** **Indirect fit.** In the agent's repo, scope conventions by type: a rule for the MCP tool-definition files (`**/tools/*.py`) loads only when editing tools; a test rule (`**/*.test.py`) only when editing tests. Keeps each editing context focused.

### 7a. Type-scoped conventions
`paths: ["**/tools/*.py"]` for tool-contract rules; they load only while editing tool definitions.

### 7b. Scenario-1 self-check
1. Tool-definition conventions everywhere or scoped? → Scoped via `paths:` to the tool files.

---

## 8. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Strong fit.** Generated code spans file types — source, tests, configs. Path-scoped rules apply the right convention per type: source style on `**/*.ts`, test conventions on `**/*.test.ts`, so the generator follows test rules exactly when writing tests and not otherwise.

### 8a. Per-type generation rules
`**/*.test.ts` → test conventions; `**/*.ts` (non-test) → source style. Each loads only for its file type.

### 8b. Scenario-2 self-check
1. Apply test conventions only when generating tests? → Path-scoped rule `**/*.test.ts`.
2. Why not put it all in CLAUDE.md? → It would load always, diluting relevance and wasting tokens.

---

## 9. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Minimal fit.** Path rules are about editing matching files, which is tangential to a research workflow. The applicable angle is the same as S1: in the system's codebase, scope per-file-type conventions so each editing context stays lean.

### 9a. Scenario-3 self-check
1. Do path rules drive runtime research behavior? → No — they scope conventions when editing matching code files.

---

## 10. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Core fit — the home scenario.** The two marquee examples live here: `paths: ["terraform/**/*"]` so infra rules load only on Terraform work, and `**/*.test.tsx` so test conventions cover tests scattered throughout the codebase. The §3 choice is the exam's centerpiece: tests are spread everywhere → a glob rule beats per-folder directory CLAUDE.md files.

### 10a. Terraform subtree rule
`paths: ["terraform/**/*"]` — pin-versions/remote-state rules load only when editing infra, invisible during app work.

### 10b. Scattered tests → glob rule
`paths: ["**/*.test.tsx"]` — one rule for all test files regardless of folder; no need to copy a directory CLAUDE.md into every package.

### 10c. Choosing the right tool
Cohesive package (`backend/`)? Directory CLAUDE.md. File type spread across the tree (tests, stories, migrations)? Path-scoped glob rule.

### 10d. Scenario-4 self-check
1. Infra rules only on Terraform? → `paths: ["terraform/**/*"]`.
2. Conventions for tests scattered everywhere? → `paths: ["**/*.test.tsx"]` (one rule).
3. Glob rule vs directory CLAUDE.md — when each? → Cross-cutting type → glob; cohesive location → directory CLAUDE.md.
4. (Goal tie) How does this aid productivity? → Each editing context loads only relevant conventions — less noise, fewer tokens, better adherence.

---

## 11. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Strong fit.** A PR touches files of several types; path-scoped rules mean the reviewer applies test conventions only to changed test files, API conventions only to changed API files, etc. Type-appropriate criteria reduce false positives (it stops judging a config file by source-code rules).

### 11a. Type-appropriate review criteria
`**/*.test.*` → test-quality rules; `src/api/**/*` → API conventions. The reviewer applies the right standard per changed file.

### 11b. Scenario-5 self-check
1. Apply API rules only to API files in review? → Path-scoped rule on the API glob.
2. How does that cut false positives? → Files are judged by the right type-specific criteria.

---

## 12. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Partial fit.** In the extractor's codebase, scope conventions by type — schema-definition rules on `**/*.schema.json`, migration rules on `migrations/**/*` (the exam's null-handling-in-migrations theme). The migration example is a good cross-cutting case if scripts live in multiple places.

### 12a. Scope schema/migration conventions
`paths: ["**/*.schema.json"]` for schema rules; `paths: ["**/migrations/**/*"]` for migration null-handling rules — each loads only on matching files.

### 12b. Scenario-6 self-check
1. Schema conventions only on schema files? → `paths: ["**/*.schema.json"]`.
2. Migration null-handling rules scattered across dirs? → A glob rule, not per-folder CLAUDE.md.

---

### Sources verified against current Claude Code docs (June 2026)
- *Memory / Rules directory* (code.claude.com) — `.claude/rules/*.md` load as project memory by default; a `paths:` YAML frontmatter field (list of glob patterns) scopes a rule to load **only when editing matching files**; rules without `paths` load unconditionally; patterns match absolute paths with standard glob syntax; brace expansion (`src/**/*.{ts,tsx}`) and symlinks supported; user-level `~/.claude/rules/` loads before project rules. Path-scoped rules trim context, unlike `@import`.
- Examples (`terraform/**/*`, `**/*.test.tsx`) and the glob-vs-directory-CLAUDE.md choice come from the task statement; re-verify frontmatter field names against current docs.
