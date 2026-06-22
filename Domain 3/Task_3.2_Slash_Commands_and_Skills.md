# Task 3.2 — Custom Slash Commands & Skills
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **A skill is a reusable playbook (`SKILL.md`) you invoke on demand with `/name` — versus CLAUDE.md, which is always loaded. Where it lives sets who gets it (project = committed/team, user = private). YAML frontmatter tunes it: `context: fork` runs it in an isolated subagent so its verbose output doesn't pollute your conversation; `allowed-tools` restricts what it can do; `argument-hint` prompts for parameters.**

Everything here is: skill vs CLAUDE.md (on-demand vs always-on), scope (project vs user), and three frontmatter levers (fork, allowed-tools, argument-hint).

---

## 1. Skills vs CLAUDE.md (when to use which)

| | CLAUDE.md | Skill |
|---|---|---|
| Loaded | **Always**, every session | **On demand**, when you type `/name` (or Claude auto-invokes) |
| For | Universal standards Claude should always hold | Task-specific workflows you run sometimes |
| Cost | Permanent context | Zero until invoked |

Rule: **always-true universal standard → CLAUDE.md; sometimes-run procedure → skill.** A multi-step deploy or a codebase audit is a skill; "use 2-space indent" is CLAUDE.md. (Commands and skills are unified now — a file in `.claude/commands/` and a `SKILL.md` both create `/name`; skills are the recommended format because they support frontmatter and supporting files.)

---

## 2. Scope: project vs user

| Scope | Path | Who gets it |
|---|---|---|
| **Project** | `.claude/commands/` or `.claude/skills/<name>/SKILL.md` | Committed → whole team |
| **User** | `~/.claude/commands/` or `~/.claude/skills/<name>/SKILL.md` | Private to your machine |

Commit project skills to share team-wide. For a **personal variant** of a team skill, create it in `~/.claude/skills/` **under a different name** so it doesn't shadow or affect teammates' version.

---

## 3. `context: fork` — isolate verbose/exploratory skills

By default a skill runs **inline** (in your main conversation, with full history). Add `context: fork` to run it in an **isolated subagent**: it does the work, then returns only a **summary** to your main thread — its verbose output never pollutes your context. Use it for skills that produce a lot of output (codebase analysis, file census) or exploratory context (brainstorming alternatives).

```yaml
---
name: codebase-audit
description: Audit the codebase for common issues
context: fork
---
Scan every file and report issues...   # all this noise stays in the subagent
```

Caveats: a forked skill has **no conversation history** (write it self-contained), and it only makes sense for skills with an explicit **task** (a pure reference/guidelines skill forked into a subagent has nothing to act on). You can pick the subagent type with `agent:` (e.g., `Explore` for read-only).

---

## 4. `allowed-tools` — restrict tools during the skill

`allowed-tools` lists the tools the skill may use while it runs, pre-approved without a permission prompt. It's a least-privilege lever: a read-only research skill gets `Read, Grep, Glob`; a safe writer gets only file-write, never destructive `Bash`. This prevents the skill from taking actions outside its job.

```yaml
allowed-tools: Read, Grep, Glob          # research skill: cannot modify anything
# or
allowed-tools: write_file                 # can write files, not run shell
```

---

## 5. `argument-hint` — prompt for parameters

`argument-hint` shows the expected arguments in the autocomplete menu when someone types `/name`, so they know what to pass (and are prompted when they invoke it without args). Pure UX, no behavior change, but it makes parameterized skills usable. Convention: `<required>` / `[optional]`.

```yaml
argument-hint: <environment>      # /deploy <environment>
```

(Inside the skill, reference inputs via `$ARGUMENTS` or positional `$1`, `$2`; live data via `` !`command` ``.)

---

## 6. Anti-patterns (with *why*)
- **Putting a sometimes-run procedure in CLAUDE.md.** It burns context every session; make it an on-demand skill.
- **Forking a pure-reference skill.** The subagent gets guidelines but no task → returns nothing useful. Fork only task skills.
- **No `allowed-tools` on a research skill.** It can wander into writes/Bash; restrict it.
- **Verbose skill without `context: fork`.** Floods your conversation, shrinking room to think; fork it.
- **Personal variant with the same name in `~/.claude/skills/`.** Risks confusion/shadowing; use a different name.
- **Parameterized skill without `argument-hint`.** Users don't know what to pass; add the hint.

---

## 7. Self-check (core mechanics)
1. Skill vs CLAUDE.md? → On-demand workflow vs always-loaded universal standard.
2. Project vs user skill location? → `.claude/skills/` (committed, team) vs `~/.claude/skills/` (private).
3. What does `context: fork` do? → Runs the skill in an isolated subagent; only a summary returns, keeping main context clean.
4. When does fork *not* make sense? → For pure-reference skills with no actionable task (and remember: no conversation history).
5. Restrict a skill to read-only? → `allowed-tools: Read, Grep, Glob`.
6. Prompt for a required parameter? → `argument-hint: <param>`.
7. (Ties to §0) One sentence? → *On-demand playbooks, scoped to who needs them, with fork/allowed-tools/argument-hint to isolate, restrict, and prompt.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7). Slash commands/skills are Claude Code constructs, so they serve the **team building** this agent.

**The key link to everything above:** **Indirect fit.** A project skill like `/add-support-tool` or `/run-agent-evals` packages a repetitive build/test workflow for the team; `argument-hint` prompts for which tool/scenario. Universal "always" rules stay in CLAUDE.md (3.1), procedures become skills.

### 8a. Project skill for a repeated workflow
`/run-agent-evals <scenario>` packages the eval run once for everyone; `argument-hint` makes the parameter obvious.

### 8b. Scenario-1 self-check
1. Repeated build/test workflow → skill or CLAUDE.md? → Skill (on-demand).
2. Share it team-wide? → Project scope (`.claude/skills/`, committed).

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Strong fit.** The pipeline *is* a procedure → encode it as a project skill `/generate-feature <spec>` with `argument-hint`, `allowed-tools` scoped to file writes + the test runner (no destructive Bash), and the plan→write→test steps in the body. Reusable, parameterized, tool-restricted.

### 9a. The pipeline as a skill
Body: plan → write → run tests → report. `allowed-tools: Read, Write, Edit, Bash(npm:test)` keeps it from destructive actions. `argument-hint: <feature-spec>`.

### 9b. Scenario-2 self-check
1. Package the pipeline? → Project skill with the steps in the body.
2. Stop it doing destructive Bash? → `allowed-tools` limited to writes + the test command.
3. Prompt for the spec? → `argument-hint: <feature-spec>`.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Core fit for `context: fork`.** Research/exploration is exactly the verbose, exploratory output `context: fork` is designed to isolate — a `/research <topic>` skill with `context: fork` (and `agent: Explore`, `allowed-tools: Read, Grep, Glob`) does the heavy digging in a subagent and returns only a summary, keeping the main conversation clean.

### 10a. Forked research skill
`context: fork` + `agent: Explore` + read-only `allowed-tools` → all the noisy searching stays in the subagent; you get the conclusion. Write it self-contained (no history in a fork).

### 10b. Scenario-3 self-check
1. Keep verbose research out of the main thread? → `context: fork`.
2. Make the forked researcher read-only? → `allowed-tools: Read, Grep, Glob` (and `agent: Explore`).
3. Gotcha with fork? → No conversation history; write the skill self-contained.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Core fit — the home scenario.** Every 3.2 idea applies: project skills for team workflows (`/scaffold-module`), user skills for personal helpers, `context: fork` for a verbose `/codebase-audit` (isolate the noise), `allowed-tools` to make an exploration skill read-only, `argument-hint` for parameters, and a **personal variant in `~/.claude/skills/` under a different name** so your tweaks don't affect teammates.

### 11a. Fork a verbose audit
`/codebase-audit` with `context: fork` scans everything in a subagent and returns a summary — your context stays free for actual work.

### 11b. Read-only exploration skill
`/explore <area>` with `allowed-tools: Read, Grep, Glob` can investigate but never modify — safe to run anywhere.

### 11c. Personal variant, different name
Like the team `/commit` but want your own flavor? Create `~/.claude/skills/my-commit/SKILL.md` — different name, private, no impact on teammates' `/commit`.

### 11d. Scenario-4 self-check
1. A verbose audit floods your context? → `context: fork`.
2. Make a skill investigate-only? → `allowed-tools: Read, Grep, Glob`.
3. Personal tweak to a team skill? → New skill in `~/.claude/skills/` with a **different name**.
4. (Goal tie) How do skills aid productivity? → Repeatable workflows run on demand with the right tools and isolation, not re-typed each time.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Strong fit.** Package the review as a project skill `/review` (committed, so CI and humans run the identical workflow) with `allowed-tools` scoped to read-only (the reviewer must not modify code — Task 2.3 least privilege). `argument-hint` documents inputs. The same `/review` a developer runs locally is what CI invokes.

### 12a. One shared `/review` skill
Project-scoped, read-only `allowed-tools`, used both locally and in CI — consistent behavior, no merge/write capability.

### 12b. Scenario-5 self-check
1. Share the review workflow with CI + team? → A committed project skill.
2. Stop the reviewer modifying code? → Read-only `allowed-tools`.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Strong fit.** An `/extract <document>` project skill encapsulates the extract→validate workflow with `argument-hint: <document-path>` and `allowed-tools` scoped to read + write-output (no destructive actions). If a batch run is verbose, `context: fork` isolates per-document noise and returns a summary.

### 13a. Extraction skill with scoped tools + hint
Body: load doc → extract → validate against schema → write record. `argument-hint: <document-path>`, `allowed-tools: Read, write_file`.

### 13b. Scenario-6 self-check
1. Package the extract→validate flow? → Project skill `/extract`.
2. Prompt for the document? → `argument-hint: <document-path>`.
3. Verbose batch output? → `context: fork` to isolate and summarize.

---

### Sources verified against current Claude Code docs (June 2026)
- *Extend Claude with skills* / *Slash commands* (code.claude.com) — `SKILL.md` frontmatter: `name`, `description`, `argument-hint`, `allowed-tools`, `context: fork`, `agent`, `model`, `disable-model-invocation`, `user-invocable`, `hooks`; `context: fork` runs in an isolated subagent with **no conversation history**, returns a summary, and only makes sense for skills with an explicit task; `allowed-tools` pre-approves/limits tools during execution; `argument-hint` is autocomplete UX; commands (`.claude/commands/`) and skills (`.claude/skills/<name>/SKILL.md`) are unified into `/slash-commands`; project (committed) vs user (`~/.claude/...`, private) scope; personal variants use a different name; `$ARGUMENTS`/`$1` and `` !`cmd` `` for args and live data.
- Skills ship fast (frontmatter fields are added over time) — re-verify field names against current docs.
