# Task 4.3 — Enforcing Structured Output (Tool Use & JSON Schemas)
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **The reliable way to get structured output is to define a tool whose input is your JSON schema, then read the data out of the model's `tool_use` call — the schema guarantees the JSON is syntactically valid. But syntax-valid ≠ correct: tool use eliminates malformed JSON, not *semantic* errors (totals that don't sum, values in the wrong field). Make fields optional when the source may lack them so the model doesn't fabricate, and use `tool_choice` to control whether/which tool fires.**

Everything here is: tool_use + schema for guaranteed-valid structure, the syntax-vs-semantic boundary, optional/enum schema design, and tool_choice control.

---

## 1. Tool use + JSON schema = guaranteed-valid structure

Instead of asking the model to "return JSON" in prose (where it can produce malformed output — trailing commas, unescaped quotes), you **define a tool whose `input_schema` is your target shape** and have the model call it. The API constrains the `tool_use.input` to that schema, so you read your structured data straight from the `tool_use` block with **no JSON syntax errors**. This is the most reliable structured-output approach.

```python
tools = [{
  "name": "extract_invoice",
  "description": "Record the extracted invoice fields",
  "input_schema": {
    "type": "object",
    "properties": {
      "invoice_number": {"type": "string"},
      "total": {"type": "number"},
      "currency": {"type": "string", "enum": ["USD","EUR","GBP","other"]}
    },
    "required": ["invoice_number"]
  }
}]
# read result from the tool_use block's .input
```

(Combining `tool_choice:"any"` with **strict tool use** guarantees both that a tool is called *and* that its inputs strictly follow the schema.)

---

## 2. `tool_choice`: auto vs any vs forced (recap for extraction)

| `tool_choice` | Behavior | Extraction use |
|---|---|---|
| `{"type":"auto"}` | Model may return **text instead** of calling a tool | Risky for guaranteed output |
| `{"type":"any"}` | Must call **some** tool (model picks which) | Unknown doc type, multiple schemas |
| `{"type":"tool","name":"X"}` | Must call **tool X** | Force a specific extraction first |

For guaranteed structured output you want `any` or a forced tool, not `auto` (which can answer in prose). Note `any`/`tool` prefill the turn, so no prose precedes the call (Task 2.3).

---

## 3. The syntax-vs-semantic boundary (critical)

Tool use guarantees the output is **schema-valid JSON** — correct types, required fields present, enums respected. It does **not** guarantee the output is **semantically correct**:

- Line items that don't sum to the stated total.
- A value placed in the wrong field (vendor name in the `customer` field).
- A plausible-but-wrong extracted number.

These pass schema validation because they're well-formed. Catching them needs **semantic validation** (Task 4.4) — e.g., recomputing the total, cross-checking fields. Don't assume "it matched the schema" means "it's right."

---

## 4. Optional fields prevent fabrication

If a field is **required** but the source document doesn't contain it, the model is pressured to **fabricate** a value to satisfy the schema. Make fields **optional/nullable** when the source may legitimately lack them, so the model can return `null`/omit rather than invent. Required should be reserved for fields that are genuinely always present.

```json
"properties": { "tax_id": {"type": ["string","null"]} },
"required": ["invoice_number"]      // only the truly-always-present field
```

---

## 5. Enum design: "unclear" and "other" + detail

Closed enums force a choice that may not fit. Two patterns keep them robust:

- **`"unclear"`** as an enum value for genuinely ambiguous cases — better than forcing a wrong category.
- **`"other"` + a detail string** for extensible categories — the model picks `other` and writes the specifics, so you capture novel categories without breaking the schema.

```json
"category": {"enum": ["invoice","receipt","contract","other"]},
"category_detail": {"type": ["string","null"]}   // filled when category = "other"
```

Also include **format-normalization rules in the prompt** alongside the strict schema (e.g., "dates as ISO 8601, amounts as numbers without currency symbols") so inconsistent source formatting maps cleanly into the schema.

---

## 6. Anti-patterns (with *why*)
- **Asking for JSON in prose.** Produces syntax errors; use a tool with a schema.
- **`tool_choice:auto` for guaranteed output.** The model may answer in prose; use `any` or a forced tool.
- **Assuming schema-valid = correct.** It only guarantees syntax; semantic errors slip through (Task 4.4).
- **All fields required.** Forces fabrication when the source lacks data; make optional/nullable.
- **Rigid closed enums.** No fit for novel/ambiguous cases; add `"unclear"` and `"other"`+detail.
- **Schema without prompt normalization rules.** Inconsistent source formats map badly; state normalization in the prompt.

---

## 7. Self-check (core mechanics)
1. Most reliable structured-output method? → A tool whose `input_schema` is your shape; read `tool_use.input`.
2. What does tool use eliminate? → JSON **syntax** errors.
3. What does it *not* prevent? → **Semantic** errors (totals don't sum, wrong field).
4. Guarantee a tool is called when doc type is unknown? → `tool_choice:"any"`.
5. Force a specific extractor first? → `{"type":"tool","name":"extract_metadata"}`.
6. Stop fabrication of absent fields? → Make them optional/nullable, not required.
7. Handle ambiguous/novel categories? → `"unclear"` enum value; `"other"` + detail string.
8. (Ties to §0) One sentence? → *Tool-use schemas guarantee valid structure; design for missing/ambiguous data and validate semantics separately.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Strong fit.** The handoff summary to a human (Task 1.4) should be structured via a schema'd tool so it's always well-formed for the downstream system. Use `tool_choice:"any"` to guarantee the agent emits a structured action rather than prose, and `"unclear"`/`"other"` enums for ambiguous request categories.

### 8a. Schema'd handoff
Define an `escalate` tool with a schema (customer ID, root cause, amount, recommended action); the structured summary is guaranteed well-formed.

### 8b. Scenario-1 self-check
1. Guarantee a structured action not prose? → `tool_choice:"any"`.
2. Ambiguous request category? → `"unclear"` enum value.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Strong fit.** When a stage must emit structured data (a plan object, a test result summary), a schema'd tool guarantees the orchestrator gets valid JSON. Force the planning tool first (`{"type":"tool","name":"plan"}`) so a structured plan precedes writing.

### 9a. Schema'd plan/result objects
Define tools for the plan and the test-summary so each stage's output is machine-readable for the next.

### 9b. Scenario-2 self-check
1. Guarantee a valid plan object? → A schema'd `plan` tool, forced first.
2. Does valid JSON mean a correct plan? → No — semantics still need checking.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research (see Domain-1 §9).

**The key link to everything above:** **Strong fit.** Findings passed between agents should be schema'd (`{claim, source_url, confidence}`) so synthesis gets valid, parseable records. Optional fields matter: if a source lacks a date, the `date` field is nullable so the model doesn't fabricate one.

### 10a. Schema'd findings
A tool schema for findings guarantees valid hand-off records; nullable fields prevent invented metadata.

### 10b. Scenario-3 self-check
1. Guarantee parseable findings between agents? → A schema'd findings tool.
2. Source lacks a date? → Nullable `date` field (no fabrication).

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Partial fit.** When the assistant must return structured analysis (a dependency map, a list of findings) for a script to consume, use a schema'd tool. Otherwise much of its work is free-form coding. `"other"`+detail enums help when categorizing diverse findings.

### 11a. Schema'd analysis output
Define a tool schema when a script consumes the assistant's analysis; otherwise free-form is fine.

### 11b. Scenario-4 self-check
1. Script needs to parse the assistant's findings? → Emit them via a schema'd tool.
2. Diverse finding categories? → `"other"` + detail.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Strong fit.** The review must be machine-parseable for the gating script — a schema'd findings tool (or the CLI `--json-schema`, Task 3.6) guarantees valid `{file, line, severity, issue, fix}` output. Severity as an enum; `"other"`+detail for issue types outside the main set. Remember §3: valid JSON doesn't mean the *findings* are correct — that's a precision concern (Task 4.1).

### 12a. Schema-locked review output
Guarantee the `{file, line, severity, issue, fix}` shape via a tool schema / `--json-schema` so the gate script parses reliably.

### 12b. Scenario-5 self-check
1. Guarantee parseable review JSON? → Tool-use schema or `--json-schema`.
2. Issue type outside the enum set? → `"other"` + detail.
3. Valid JSON = correct findings? → No; precision is separate (Task 4.1).

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Core fit — the home scenario.** Every 4.3 idea lands: define an **extraction tool with a JSON-schema input**; use `tool_choice:"any"` when the **document type is unknown** (multiple schemas) or force `{"type":"tool","name":"extract_metadata"}` to run a specific extraction **before enrichment**; make fields **optional/nullable** so the model doesn't fabricate; add `"unclear"`/`"other"`+detail enums; and put **format-normalization rules** in the prompt. Then validate semantics separately (§3 → Task 4.4).

### 13a. The extraction tool
Schema as `input_schema`; read the record from `tool_use.input`. No syntax errors.

### 13b. tool_choice by document certainty
Unknown type / multiple schemas → `any`. Need a specific extraction first → force that tool, then enrich in follow-up turns.

### 13c. Optional fields + enums + normalization
Nullable optional fields prevent fabrication; `"unclear"`/`"other"`+detail handle ambiguity/novelty; prompt normalization rules (ISO dates, numeric amounts) map messy sources cleanly.

### 13d. Remember the semantic gap
Schema-valid records can still be wrong (total ≠ sum of line items); validate semantically (Task 4.4).

### 13e. Scenario-6 self-check
1. Most reliable extraction output? → A schema'd extraction tool; read `tool_use.input`.
2. Unknown document type, several schemas? → `tool_choice:"any"`.
3. Run metadata extraction before enrichment? → `{"type":"tool","name":"extract_metadata"}`.
4. Source may lack a field? → Make it optional/nullable (no fabrication).
5. Ambiguous/novel category? → `"unclear"`; `"other"`+detail.
6. (Goal tie) How does this serve "high accuracy"? → Guaranteed-valid structure + no-fabrication design; pair with semantic validation for full correctness.

---

### Sources verified against current Anthropic docs (June 2026)
- *Tool use / Define tools* (platform.claude.com) — tool use with `input_schema` is the reliable path to structured output; `tool_choice` `auto`/`any`/`{"type":"tool","name":...}`/`none`; `any`/`tool` prefill the turn (no preceding prose); combine `tool_choice:"any"` with **strict tool use** to guarantee a schema-conforming call. Tool use eliminates JSON syntax errors but not semantic correctness.
- Optional/nullable fields to prevent fabrication, `"unclear"`/`"other"`+detail enum patterns, and prompt-side format normalization come from the task statement and schema best practices. Strict-tool-use/structured-output beta status ships fast — re-verify against current docs.
