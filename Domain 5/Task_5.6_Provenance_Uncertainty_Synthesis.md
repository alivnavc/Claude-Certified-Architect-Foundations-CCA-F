# Task 5.6 — Provenance & Uncertainty in Multi-Source Synthesis
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **Every claim must keep a link to where it came from — and that link is exactly what summarization destroys if you let it. Carry structured claim→source mappings all the way through synthesis. When credible sources conflict, don't pick a winner — annotate the conflict with attribution. Require dates so "different time" isn't mistaken for "contradiction." And render each content type in its natural form rather than flattening everything to one shape.**

Everything here is: preserve provenance through compression, attribute conflicts instead of resolving them silently, date everything, and render appropriately.

---

## 1. Summarization loses attribution

Each summarization step **compresses findings** — and if it compresses *without preserving the claim→source mapping*, the attribution is lost. "Source A says revenue grew 12%, Source B says 8%" becomes "revenue grew" with no idea who said what or which number. Once provenance is gone, you can't cite, can't audit, can't reconcile conflicts. So the rule: **never compress away the source link.**

---

## 2. Structured claim-source mappings carried through synthesis

The fix is **structured claim→source mappings** that the synthesis agent must **preserve and merge** when combining findings. Subagents output records like:

```json
{ "claim": "Revenue grew 12% in 2025",
  "source_url": "https://...", "document": "AcmeAnnualReport2025.pdf",
  "excerpt": "...total revenue increased 12%...", "date": "2026-02-10" }
```

Synthesis combines these *as records*, keeping each claim tied to its source, excerpt, and date — rather than melting them into unattributed prose. The mapping travels intact from extraction through synthesis to the final report.

---

## 3. Conflicting statistics → annotate, don't pick

When credible sources give **conflicting values** (12% vs 8%), the wrong move is to **arbitrarily select one**. The right move is to **annotate the conflict with source attribution**: "Source A (annual report, Feb 2026) reports 12%; Source B (analyst note, Nov 2025) reports 8%." Let the reader (or coordinator) see both and decide. Document analysis should **complete with conflicting values included and explicitly annotated**, letting the **coordinator decide how to reconcile** before synthesis — not silently drop one.

---

## 4. Temporal data prevents false contradictions

Two "different" numbers may not conflict at all — they may be from **different times**. Require **publication or data-collection dates** in structured outputs so a 2024 figure and a 2026 figure aren't misread as a contradiction; they're just a time series. Without dates, temporal differences masquerade as disagreements. Dates are part of provenance.

---

## 5. Well-established vs contested + appropriate rendering

- **Distinguish well-established from contested findings.** Structure reports with explicit sections separating consensus findings from disputed ones, **preserving original source characterizations and methodological context** (how each source measured it). Don't flatten a contested claim into a confident statement.
- **Render content types appropriately.** Financial data → **tables**; news → **prose**; technical findings → **structured lists**. Forcing everything into one uniform format loses meaning; match the rendering to the content.

---

## 6. Anti-patterns (with *why*)
- **Summarizing away the source link.** Destroys provenance; you can't cite or reconcile. Preserve claim→source mappings.
- **Picking one value among conflicting sources.** Hides disagreement and may be wrong; annotate the conflict with attribution.
- **Resolving conflicts before the coordinator sees them.** Removes the decision; pass conflicts up annotated.
- **Omitting dates.** Temporal differences look like contradictions; require publication/collection dates.
- **Stating contested findings as established.** Misleads; separate consensus from disputed, with methodology.
- **One uniform output format.** Tables-as-prose (or vice versa) loses meaning; render per content type.

---

## 7. Self-check (core mechanics)
1. What does summarization destroy if unguarded? → Claim→source attribution.
2. The fix? → Structured claim-source mappings (url, document, excerpt, date) preserved and merged through synthesis.
3. Conflicting statistics from credible sources? → Annotate the conflict with attribution; don't arbitrarily pick one.
4. Who decides how to reconcile a conflict? → The coordinator — so pass conflicts up annotated, not resolved.
5. Why require dates? → So temporal differences aren't misread as contradictions.
6. How to present consensus vs disputed findings? → Separate sections, preserving source characterizations + methodology.
7. How to render mixed content? → Per type — financial as tables, news as prose, technical as structured lists.
8. (Ties to §0) One sentence? → *Keep claims tied to dated sources, attribute conflicts, and render each type naturally.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Minimal fit.** Single-source backend lookups, not multi-source synthesis. The carry-over: when two systems disagree (CRM says shipped, warehouse says pending), **annotate the conflict** and surface it rather than picking one — the provenance principle applied to backend data.

### 8a. Annotate conflicting system data
CRM vs warehouse disagreement → flag both with their source, don't silently pick.

### 8b. Scenario-1 self-check
1. Two backend systems disagree? → Annotate the conflict with source; don't arbitrarily choose.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Minimal fit.** Not a synthesis task. The faint analogue: when generating from multiple spec documents that conflict, flag the conflict rather than silently choosing one interpretation (cf. Task 5.2 ask-don't-guess).

### 9a. Flag conflicting specs
Conflicting requirements across docs → surface the conflict, don't pick silently.

### 9b. Scenario-2 self-check
1. Two spec docs conflict? → Flag it (don't silently choose).

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research, then synthesizing (see Domain-1 §9).

**The key link to everything above:** **Core fit — the home scenario.** Every 5.6 idea lands: subagents output **structured claim-source mappings** (url, document, excerpt, date) that synthesis **preserves and merges** (§2); **conflicting statistics** are **annotated with attribution**, not arbitrarily resolved (§3) — analysis completes with conflicts included for the **coordinator to reconcile**; **dates** prevent temporal differences being read as contradictions (§4); the report **separates well-established from contested** findings with methodology (§5); and content is **rendered per type** — financials as tables, news as prose, technical as lists.

### 10a. Claim-source mappings through synthesis
Each finding stays tied to its source/excerpt/date from subagent to final report — citations survive.

### 10b. Annotate conflicts, dates prevent false ones
12% vs 8% → annotate both with source; include collection dates so a 2024 vs 2026 figure reads as a time series, not a contradiction.

### 10c. Consensus vs contested + per-type rendering
Separate established from disputed findings (with methodology); render financial data as tables, news as prose, technical findings as lists.

### 10d. Scenario-3 self-check
1. Keep citations through synthesis? → Structured claim-source mappings preserved and merged.
2. Two credible sources give different stats? → Annotate with attribution; let the coordinator reconcile.
3. A 2024 and a 2026 figure differ? → Dates show it's temporal, not a contradiction.
4. Mixed content in the report? → Tables for financials, prose for news, lists for technical.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores unfamiliar codebases, understands legacy systems, generates boilerplate, automates repetitive tasks.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Partial fit.** When summarizing findings across many files, keep each finding tied to its **file/line provenance** so claims about the codebase remain verifiable (the code analogue of claim-source mappings). When two sources of truth conflict (a comment vs the code, docs vs implementation), annotate the conflict rather than trusting one.

### 11a. File/line provenance on findings
"AuthService validates tokens (`src/auth/service.ts:42`)" — keep the location so the claim is checkable.

### 11b. Scenario-4 self-check
1. Keep codebase claims verifiable? → Attach file/line provenance to each finding.
2. Comment contradicts code? → Annotate the conflict (don't trust one).

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Partial fit.** Every finding should carry provenance — file and line — so it's a checkable, actionable comment, not a vague assertion (this is also why the JSON schema includes `location`). When two analysis passes disagree about a finding, surface both rather than silently dropping one.

### 12a. Provenance on every finding
`{file, line, issue}` — provenance makes the comment verifiable and reduces "where?" friction.

### 12b. Scenario-5 self-check
1. Make findings checkable? → Attach file/line provenance.
2. Two passes disagree on a finding? → Surface both, don't silently drop.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Strong fit.** Extracted records should carry **provenance** (source document, page/excerpt) so each value is traceable — the extraction analogue of claim-source mappings (§2). When a document gives **conflicting values** (two totals), **annotate the conflict** and include both rather than picking one (§3, cf. Task 4.4 `conflict_detected`). Include **collection/publication dates** so time-stamped figures aren't misread (§4).

### 13a. Provenance + conflict annotation on records
Each field carries its source location/excerpt; conflicting values are flagged with both, not silently resolved.

### 13b. Dates on extracted figures
Capture the document's date so a figure isn't misinterpreted against a differently-dated one downstream.

### 13c. Scenario-6 self-check
1. Make extracted values traceable? → Attach source document + excerpt/page provenance.
2. Document gives two totals? → Annotate both (conflict), don't pick one.
3. Time-stamped figures? → Capture dates to avoid temporal misreads.
4. (Goal tie) How does this serve "high accuracy" + downstream trust? → Every value is sourced, conflicts are explicit, and dates disambiguate — downstream can verify and reconcile.

---

### Sources verified against current Anthropic guidance (June 2026)
- *Citations / multi-agent synthesis guidance* (docs.claude.com) — preserving claim→source attribution through compression; the Citations feature ties generated assertions to source excerpts (the provenance pattern); structured outputs carry source metadata for downstream synthesis.
- Annotating conflicts with attribution (rather than arbitrary selection), requiring publication/collection dates to avoid false temporal contradictions, separating established from contested findings with methodology, and rendering content per type come from the task statement. These are synthesis-architecture patterns; verify any Citations/SDK specifics against current docs.
