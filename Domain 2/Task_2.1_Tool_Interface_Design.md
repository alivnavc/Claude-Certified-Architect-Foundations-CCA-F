# Task 2.1 — Designing Effective Tool Interfaces (Descriptions & Boundaries)
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–4 are the mechanics (every "Knowledge of" / "Skills in" bullet). Section 5 = anti-patterns, Section 6 = self-check. Then one section per scenario (7–12) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **The model picks a tool by reading its name and description — nothing else. So the description *is* the interface. If two tools read alike, the model can't tell them apart and misroutes. Fix selection problems by making each tool's purpose, inputs, outputs, and "use this vs that" boundary unmistakable in words — and by splitting one vague tool into several sharp ones.**

Every concept in this task is a consequence of that single fact: tool selection is a *reading-comprehension* task the model performs over your descriptions. Vague writing → unreliable selection. Sharp, differentiated writing → reliable selection.

---

## 1. The description is the primary selection mechanism

When the model decides which tool to call, it has the tool **name**, the **description**, and the **input schema** (parameter names + their descriptions). That's the whole basis for the decision. A minimal description like `"Analyzes content"` gives the model almost nothing to discriminate on, so among similar tools it guesses — and guesses wrong at a rate you can't tolerate in production.

A strong description states four things explicitly:

| Element | Why it matters |
|---|---|
| **Purpose** — what the tool does and *when to use it* | The core selection signal |
| **Input format** — exact shape/types, with an example query | Prevents malformed calls and wrong-tool calls |
| **Output** — what comes back | Lets the model chain tools correctly |
| **Boundaries** — when *not* to use it / use the sibling instead | Eliminates overlap with similar tools |

Edge cases and example inputs in the description measurably improve both selection and parameter-filling.

---

## 2. Ambiguous / overlapping descriptions cause misrouting

The classic failure: `analyze_content` and `analyze_document` with near-identical descriptions. The model has no basis to choose, so it routes inconsistently — sometimes one, sometimes the other, for the same input. Symptoms: the "wrong" tool fires, or the model oscillates between two tools across runs. The root cause is always **insufficient differentiation in the text**, not a model defect.

Two cures (Sections 3–4): **rename + re-describe** to remove overlap, or **split** a generic tool into purpose-specific ones.

---

## 3. The system prompt can override good descriptions (keyword sensitivity)

Tool selection is keyword-sensitive. A system-prompt line like *"always analyze the document thoroughly"* can create an unintended association pulling the model toward a tool whose name contains "document," even when a better-matched tool exists. So reviewing tools isn't enough — you must **review the system prompt for keyword-sensitive instructions** that fight your descriptions. The wording in the prompt and the wording in the descriptions have to point the same way.

---

## 4. The three fixes (the "Skills in" toolkit)

| Fix | What you do | Example |
|---|---|---|
| **Differentiate** | Rewrite descriptions so each states a distinct purpose, inputs, outputs, and "use vs sibling" | Make `analyze_content` clearly web-only, `analyze_document` clearly file-only |
| **Rename + re-describe** | Change the name to remove functional overlap and align the description | `analyze_content` → `extract_web_results`, described web-specifically |
| **Split** | Break one generic tool into purpose-specific tools with defined I/O contracts | `analyze_document` → `extract_data_points`, `summarize_content`, `verify_claim_against_source` |

Splitting is the strongest move when a tool is doing several jobs: each resulting tool has one purpose, so the model's choice becomes obvious and each has a clean input/output contract.

---

## 5. Anti-patterns (with *why*)
- **Minimal descriptions ("Analyzes data").** No discrimination signal → unreliable selection among similar tools.
- **Two tools, near-identical descriptions.** Guaranteed misrouting; differentiate, rename, or merge.
- **Generic catch-all tool.** `analyze_document` doing extract+summarize+verify forces the model to infer intent; split it.
- **Ignoring the system prompt.** A keyword in the prompt silently overrides a well-written description; review both together.
- **Describing how it works, not when to use it.** The model needs the *when/when-not* boundary, not implementation trivia.
- **No input example.** The model mis-formats parameters; add an example query and edge cases.

---

## 6. Self-check (core mechanics)
1. What does the model use to choose a tool? → Name + description + input schema (the words).
2. Why do `analyze_content`/`analyze_document` misroute? → Near-identical descriptions give no basis to choose.
3. Four things a good description states? → Purpose/when-to-use, input format + example, output, boundaries vs siblings.
4. How can a system prompt break selection? → Keyword-sensitive instructions create unintended tool associations.
5. Three fixes for overlap? → Differentiate, rename+re-describe, or split into purpose-specific tools.
6. (Ties to §0) One sentence? → *The description is the interface; write it so the right tool is unmistakable.*

---

## 7. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent that acts through four backend tools (full term breakdown in any Domain-1 guide, §7). Here the angle is whether the *descriptions* of those four tools let the model pick correctly under ambiguous requests.

**The key link to everything above:** **Core fit.** "High-ambiguity" is exactly where weak descriptions bite — a vague `lookup_order` vs `get_customer` boundary makes the model fetch the wrong thing. Each tool's description must state its purpose, inputs (order ID vs customer ID), output, and when to use it vs the sibling (§1).

### 7a. Differentiate the four tools
`get_customer` ("look up a customer by email/ID; returns profile + verification status") vs `lookup_order` ("look up an order by order ID; returns status, items, charges") must not blur. Give each a distinct input contract so an order question never calls `get_customer`.

### 7b. Boundaries prevent dangerous misrouting
`process_refund`'s description states when to use it *and* when not to (escalate instead), reinforcing the Task-1.4 gate at the description layer.

### 7c. Scenario-1 self-check
1. Why do clear descriptions matter most here? → High-ambiguity input stresses selection.
2. How to stop order questions hitting `get_customer`? → Distinct input contracts (order ID vs customer ID) in the descriptions.
3. (Goal tie) How does this serve 80% FCR? → Correct tool selection means fewer wrong actions and dead-ends, so more cases resolve first time.

---

## 8. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent that writes code in stages (see Domain-1 §8).

**The key link to everything above:** **Strong fit.** Code tools overlap badly if under-described: `write_file` vs `edit_file` vs `apply_patch` all "change code." The boundary ("Write = whole new file; Edit = targeted change to existing unique text") must live in the descriptions or the model picks wrong and corrupts files. (Built-in Read/Write/Edit selection is Task 2.5; here it's the *description-writing* discipline.)

### 8a. Differentiate write vs edit vs patch
State each tool's exact precondition and effect so "change line 12" routes to edit, "create a new module" routes to write.

### 8b. Scenario-2 self-check
1. Why do code tools misroute? → "Modify code" descriptions overlap.
2. Fix? → Encode the precondition/effect boundary (new file vs targeted edit) in each description.

---

## 9. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research, then synthesizing (see Domain-1 §9).

**The key link to everything above:** **Core fit — the textbook example.** This is where `analyze_content` vs `analyze_document` actually lives: a web-result analyzer and a document analyzer with twin descriptions make the research agent grab the wrong one. The fix is the §4 rename: `analyze_content` → `extract_web_results` with a web-specific description; keep `analyze_document` for files. Now "analyze this URL's results" and "analyze this PDF" route deterministically.

### 9a. Rename to kill overlap
`extract_web_results` (input: search-result set; output: extracted findings + URLs) vs `analyze_document` (input: a file; output: structured doc analysis). Different names, different input contracts, zero overlap.

### 9b. Split the generic analyzer
If one `analyze_document` is doing too much, split into `extract_data_points`, `summarize_content`, `verify_claim_against_source` (§4) so each subagent calls exactly the right one.

### 9c. Scenario-3 self-check
1. The canonical overlap pair? → `analyze_content` vs `analyze_document`.
2. Rename fix? → `analyze_content` → `extract_web_results`, web-specific description.
3. When split instead of rename? → When one tool does several jobs (extract/summarize/verify).

---

## 10. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores codebases, handles legacy, generates boilerplate, automates chores.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Strong fit.** With built-ins *plus* MCP tools, overlap is rampant: a powerful MCP `code_search` competes with built-in `Grep`, and the model defaults to the familiar built-in unless the MCP description sells its added capability (this is also Task 2.4). Sharp, capability-rich descriptions are what make the model reach for the better tool.

### 10a. Describe MCP tools to beat built-ins
If `Grep` and an MCP semantic-search tool both "search code," the model picks `Grep` by default. The MCP tool's description must state what it does that `Grep` can't (semantic/cross-repo) so it wins when appropriate.

### 10b. Scenario-4 self-check
1. Why does the model ignore a better MCP tool? → Its description doesn't differentiate it from the familiar built-in.
2. Fix? → Enrich the description with the distinct capability and when to prefer it.

---

## 11. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Partial-to-strong fit.** A CI reviewer usually has few tools, so overlap is less of a risk — but the *system prompt* angle (§3) is sharp here: a keyword-heavy review prompt can pull the model toward the wrong analysis tool or behavior. Review the prompt and the tool descriptions together so they don't conflict.

### 11a. Audit the prompt for keyword pulls
A line like "thoroughly scan the document for issues" might bias toward a doc-analysis tool over a diff-analysis tool. Align wording with the tool you actually want.

### 11b. Scenario-5 self-check
1. Main 2.1 risk in a small-tool CI agent? → System-prompt keywords overriding intended selection.
2. Fix? → Review prompt + descriptions together for conflicting keywords.

---

## 12. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Core fit — the split example.** A generic `analyze_document` that extracts, summarizes, *and* verifies is the wrong shape for an accuracy-critical pipeline: the model picks an interpretation per call and consistency suffers. Split into `extract_data_points`, `summarize_content`, `verify_claim_against_source` (§4), each with a defined input/output contract. Now the pipeline calls exactly the operation it needs, and accuracy stops depending on the model's guess about what "analyze" means.

### 12a. Split into purpose-specific tools
`extract_data_points` (doc → typed fields), `summarize_content` (doc → summary), `verify_claim_against_source` (claim+source → verdict). Three sharp contracts replace one ambiguous one.

### 12b. Input/output contracts aid validation
A tool whose output contract is "JSON matching schema X" makes the downstream schema check (Task 2.2/1.4) straightforward — the description and the contract reinforce accuracy.

### 12c. Scenario-6 self-check
1. Why split `analyze_document`? → A multi-job tool makes selection ambiguous; accuracy suffers.
2. The three split tools? → `extract_data_points`, `summarize_content`, `verify_claim_against_source`.
3. (Goal tie) How does splitting raise accuracy? → Each call invokes one well-defined contract, removing the model's guess about intent.

---

### Sources verified against current Anthropic docs (June 2026)
- *Define tools* / *Tool use overview* (platform.claude.com) — tool selection is driven by name + description + input schema; the default `tool_choice:auto` boundary is steerable via system-prompt wording (the keyword-sensitivity point); detailed descriptions and input examples improve selection and parameter accuracy.
- *Tool-use best practices* — differentiating, renaming, and splitting tools to remove overlap; including purpose, inputs, outputs, and boundaries in descriptions.
- Tool-description quality is a design discipline, not a versioned API; verify any SDK-specific schema fields against current docs when implementing.
