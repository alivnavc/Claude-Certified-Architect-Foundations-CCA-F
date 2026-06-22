# Task 3.6 — Integrating Claude Code into CI/CD Pipelines
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **To run Claude Code in a pipeline it must be non-interactive (`-p`), emit machine-parseable output (`--output-format json`, optionally schema-locked with `--json-schema`), and get its project context from CLAUDE.md. One subtle rule: a fresh, independent review instance beats having the same session that wrote the code review itself.**

Everything here is: make it scriptable (`-p`), make its output parseable (json/schema), feed it context (CLAUDE.md), and isolate review from generation.

---

## 1. CI/CD in one paragraph

**CI/CD** is automation that runs on every code change (build, test, deploy). A **pull request (PR)** is a proposed change; a **pipeline** is the sequence of automated steps that runs against it. Putting Claude Code in the pipeline means it runs **headlessly** — no human typing — on each PR, and a script acts on its output (e.g., posting comments, gating the merge).

---

## 2. `-p` / `--print` — non-interactive mode

`claude -p "<prompt>"` runs the query and **exits** instead of opening an interactive session. This is essential in CI: without it, Claude waits for interactive input and the pipeline **hangs**. It reads stdin and writes stdout, so it composes like any Unix tool:

```bash
git diff main --name-only | claude -p "Review these changed files" > review.txt
```

For CI you often add `--bare` to make the run **hermetic** — it skips auto-discovery of hooks, skills, MCP servers, auto-memory, and CLAUDE.md so the result is identical on every machine. (Trade-off in §5: `--bare` also skips CLAUDE.md, so if you want project context, don't use it — or pass context explicitly.)

---

## 3. `--output-format json` + `--json-schema` — parseable output

A script can't reliably act on prose. `--output-format json` returns a JSON envelope (run metadata plus the result; with structured output it lands in a `structured_output` field) so a script can parse it. `--json-schema <schema>` goes further: it **constrains the output to your JSON Schema**, so the structured result is guaranteed to match the shape your next step expects:

```bash
claude -p --bare --output-format json --json-schema review.schema.json \
  "Review this diff; be specific about file and line" < <(git diff HEAD~1) \
  | jq -r '.structured_output.annotations[] | "::\(.level) file=\(.file),line=\(.line)::\(.message)"'
```

That `jq` line turns each finding into a GitHub annotation — the LLM review shows up as **inline PR comments**, like any linter. Schema-locked output is what makes the step composable.

---

## 4. CLAUDE.md as the CI context source

CI-invoked Claude has no human to explain the project, so **CLAUDE.md is how you give it context**: testing standards, fixture conventions, review criteria, what counts as a valuable test. Documenting these (Task 3.1) directly improves output — better-targeted reviews (fewer false positives) and better test generation (fewer low-value tests). (Remember the `--bare` trade-off: it skips CLAUDE.md.)

Two CI-specific quality skills:
- **Re-runs after new commits:** include the **prior review findings** in context and instruct Claude to report only **new or still-unaddressed** issues — avoids duplicate comments on every push.
- **Test generation:** provide the **existing test files** in context so Claude doesn't suggest duplicate scenarios already covered.

---

## 5. Session isolation: don't let the author review itself

The same Claude session that **generated** code is a **worse reviewer** of that code than a **fresh, independent** instance — it shares the generator's assumptions and blind spots, so it tends to bless its own work. For review, spin up an **independent review instance** with its own clean context (the inverse of Task 1.7's "resume": here you deliberately *don't* carry context). Separation of author and reviewer is the point.

---

## 6. Anti-patterns (with *why*)
- **Running Claude in CI without `-p`.** It waits for input and the pipeline hangs.
- **Parsing prose output.** Brittle; use `--output-format json` (+ `--json-schema` for a guaranteed shape).
- **Reviewing with the generating session.** It rubber-stamps its own assumptions; use a fresh review instance.
- **No prior-findings context on re-runs.** Duplicate comments on every push; pass prior findings, report only new/unaddressed.
- **No existing tests in context for test-gen.** Duplicate test scenarios; provide the current suite.
- **Forgetting the `--bare` trade-off.** It skips CLAUDE.md/MCP/hooks; great for reproducibility, but you lose project context unless you pass it.

---

## 7. Self-check (core mechanics)
1. Flag for non-interactive CI runs? → `-p` / `--print` (prevents input hangs).
2. Make output machine-parseable + guaranteed-shape? → `--output-format json` with `--json-schema`.
3. How does CI Claude get project context? → CLAUDE.md (testing standards, criteria, fixtures).
4. Why not review with the generating session? → It shares the author's blind spots; use a fresh independent instance.
5. Avoid duplicate comments on re-runs? → Pass prior findings; report only new/unaddressed issues.
6. Avoid duplicate generated tests? → Provide existing test files in context.
7. (Ties to §0) One sentence? → *Headless, JSON/schema output, CLAUDE.md for context, and a fresh instance to review.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Indirect fit.** CI/CD here means the agent's repo pipeline: run agent **evals** headlessly with `-p`, emit `--output-format json` so a script gates the build on the resolution-rate metric, and keep eval criteria in CLAUDE.md. A fresh instance evaluates transcripts the agent itself produced (§5).

### 8a. Headless eval gate
`claude -p --output-format json --json-schema evals.schema.json` over a transcript set; the script fails the build if first-contact-resolution drops below target.

### 8b. Scenario-1 self-check
1. Run agent evals in CI without hanging? → `-p`.
2. Gate the build on a metric? → `--output-format json` (+ schema), parsed by the script.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Strong fit.** A generation pipeline runs headlessly: `-p` to avoid hangs, `--output-format json` so the orchestrating script knows pass/fail per stage. Crucially, §5 applies — a **separate instance** should review the generated code, not the generator, so flaws aren't rubber-stamped.

### 9a. Independent review of generated code
Generate with one instance; review with a fresh one (clean context) so it doesn't inherit the generator's assumptions.

### 9b. Scenario-2 self-check
1. Run the pipeline non-interactively? → `-p`, JSON output for the orchestrator.
2. Who reviews the generated code? → A fresh independent instance, not the generator.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Partial fit.** If research runs as a scheduled/automated job, `-p` makes it non-interactive and `--output-format json --json-schema` produces a structured, citation-bearing report a downstream system can consume. Otherwise CI/CD is tangential to this runtime workflow.

### 10a. Scheduled research as a headless job
`claude -p ... --output-format json --json-schema report.schema.json` → a machine-readable cited report on a cron.

### 10b. Scenario-3 self-check
1. Run research as an unattended job? → `-p` with schema-locked JSON output.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Strong fit.** Automating repetitive dev tasks *is* headless Claude Code: `-p` in scripts (pipe a build log in, get an explanation out), `--output-format json` when another tool consumes the result, CLAUDE.md for project context, and `--bare` when a script must be reproducible across machines (accepting the §4 CLAUDE.md trade-off).

### 11a. Pipe-in / pipe-out automation
`cat build-error.txt | claude -p "explain the root cause" > out.txt` — Unix composition for repetitive chores.

### 11b. Reproducible scripts with `--bare`
Use `--bare` for hermetic runs; pass needed context explicitly since it skips CLAUDE.md/MCP/hooks.

### 11c. Scenario-4 self-check
1. Automate a repetitive task in a script? → `-p`, compose with pipes.
2. Make the script reproducible across machines? → `--bare` (and pass context explicitly).
3. Output consumed by another tool? → `--output-format json`.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Core fit — the home scenario.** Every 3.6 idea lands: `-p` so it doesn't hang; `--output-format json --json-schema` for `{file, line, level, message}` findings posted as **inline PR comments**; **CLAUDE.md** for review criteria (cuts false positives); a **fresh independent instance** (not the author) reviewing; and on re-runs after new commits, **prior findings in context** so it reports only **new/unaddressed** issues — no duplicate comments.

### 12a. Schema-locked findings → inline comments
`--json-schema` guarantees the `{file,line,level,message}` shape; a `jq` line turns each into a GitHub annotation.

### 12b. Criteria in CLAUDE.md
Review standards and known false-positive patterns in CLAUDE.md → fewer noisy flags.

### 12c. No duplicate comments on re-runs
Pass the prior review's findings; instruct Claude to surface only new or still-open issues.

### 12d. Independent reviewer
The reviewer is a fresh instance, separate from whatever generated the change (§5).

### 12e. Scenario-5 self-check
1. Findings as inline PR comments? → `--output-format json --json-schema` + a `jq` formatter.
2. Cut false positives? → Review criteria in CLAUDE.md.
3. Avoid duplicate comments after a new commit? → Prior findings in context; report only new/unaddressed.
4. Who reviews? → A fresh independent instance.
5. (Goal tie) How does the design serve "minimize false positives"? → Real criteria from CLAUDE.md + structured, deduplicated findings keep the signal high.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Core fit.** This is the canonical `--json-schema` use: run extraction headlessly (`-p`), and **schema-lock the output** so every record is guaranteed to match the target shape before downstream integration — the CLI mirror of the schema-validation theme. `--bare` gives reproducible batch runs; CLAUDE.md (when not bare) supplies schema/edge-case conventions.

### 13a. Schema-locked extraction in a pipeline
`claude -p --output-format json --json-schema record.schema.json < doc.txt` → a guaranteed-shape record piped straight to the downstream system.

### 13b. Reproducible batch runs
`--bare` for hermetic batch processing; pass the schema and any needed context explicitly.

### 13c. Scenario-6 self-check
1. Guarantee extracted records match the schema? → `--json-schema` with `--output-format json`.
2. Run the batch reproducibly? → `--bare` (pass context explicitly).
3. (Goal tie) How does this serve "high accuracy" + "downstream integration"? → Schema-locked output means only correctly-shaped records flow downstream, no parsing guesswork.

---

### Sources verified against current Claude Code docs (June 2026)
- *Run Claude Code programmatically / headless* (code.claude.com) — `-p`/`--print` runs non-interactively (reads stdin, writes stdout); `--output-format json` returns an envelope with metadata and a `structured_output`/`result` field (and `total_cost_usd`); `--json-schema <schema>` constrains output to a JSON Schema; `--bare` makes runs hermetic by skipping auto-discovery of hooks, skills, MCP, auto-memory, and CLAUDE.md (reproducibility vs context trade-off); structured findings can be formatted (e.g., via `jq`) into inline PR annotations.
- Session-isolation guidance (independent reviewer ≠ generating session), prior-findings-on-re-run, and existing-tests-in-context come from the task statement; CLAUDE.md as CI context is Task 3.1. CLI flags ship fast — re-verify against current docs before wiring a pipeline.
