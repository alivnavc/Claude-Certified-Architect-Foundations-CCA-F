# Task 2.4 — Integrating MCP Servers into Claude Code & Agent Workflows
### Zero-to-mastery study guide (every term defined; all 6 scenarios unfolded)

**Reading order:** Section 0 is the one idea. Sections 1–5 are the mechanics. Section 6 = anti-patterns, Section 7 = self-check. Then one section per scenario (8–13) in the fixed card shape.

---

## 0. The one idea everything hangs on

> **An MCP server is a plugin that hands Claude a bundle of tools (and optionally read-only "resources"). You wire it in at one of two scopes — *project* (`.mcp.json`, committed, shared with the team) or *user* (`~/.claude.json`, private, all your projects) — keep secrets out of the file with `${VAR}` expansion, and describe the tools well enough that Claude prefers them over weaker built-ins. All configured servers' tools are discovered at connect time and available together.**

Everything here is: where the config lives (scope), how credentials stay safe (env expansion), what's exposed (tools + resources), and making the agent actually use them (descriptions).

---

## 1. What MCP is, in one paragraph

**MCP** (Model Context Protocol) is an open standard for connecting AI agents to external tools and data. An **MCP server** exposes capabilities; Claude Code (the client) connects and the server's **tools** become callable. Servers also expose **resources** (read-only, URI-addressed data the model can read as context) and **prompts** (templates). Transports: **stdio** (Claude spawns a local process) or **HTTP** (remote URL; SSE is the deprecated predecessor).

---

## 2. Scopes: where the config lives

| Scope | File | Shared? | Use for |
|---|---|---|---|
| **Project** | `.mcp.json` in the repo root | Committed to git → whole team | Shared team tooling (the team's Jira, DB) |
| **User** | `~/.claude.json` | Private to you, across all your projects | Personal/experimental servers |
| (Local) | `~/.claude.json`, keyed by project path | Private to you, this project only | Sensitive personal creds for one project |

Add via CLI: `claude mcp add <name> --scope project ...` or `--scope user ...`. When the same server is defined in multiple scopes, Claude Code connects once using the highest-precedence definition (entries aren't merged).

```json
// .mcp.json (project scope, committed)
{ "mcpServers": {
    "github": { "type": "http", "url": "https://api.githubcopilot.com/mcp/",
                "headers": { "Authorization": "Bearer ${GITHUB_TOKEN}" } } } }
```

---

## 3. Environment-variable expansion (secrets without committing secrets)

`.mcp.json` supports `${VAR}` and `${VAR:-default}` in `command`, `args`, `env`, `url`, and `headers`. The value is read from your **shell environment at connection time**, so the committed file holds the *structure* while the actual token stays in your environment (or `.env`, gitignored). This is how a team shares one config safely — if someone commits a real token by mistake, rotate it immediately.

```json
"env": { "DB_DSN": "${DATABASE_URL}" },
"url": "${API_BASE_URL:-https://api.company.com}/mcp"
```

---

## 4. Discovery and resources

**Simultaneous discovery:** tools from *all* configured servers are discovered when Claude connects (session start) and are available together. (With Tool Search on by default, definitions are deferred to save context, but the tools are still discoverable on demand.) Add a server mid-session → start a new session (for stdio) to pick up its tools.

**Resources reduce exploratory calls.** A resource is read-only content the server exposes by URI — an issue-summary catalog, a documentation hierarchy, a database schema. Exposing these as resources gives the agent *visibility into what's available* up front, so it doesn't burn tool calls blindly probing ("what tables exist? what issues are open?"). The catalog is right there to read.

---

## 5. Making the agent actually use MCP tools (+ build-vs-buy)

**Beat the built-ins with descriptions.** Claude defaults to familiar built-ins (like `Grep`). If your MCP tool is more capable but thinly described, the agent ignores it. **Enhance the MCP tool's description** to spell out its capability and output in detail so the agent prefers it when appropriate. (This is Task 2.1's discipline applied to MCP tools.)

**Buy before build.** For standard integrations (Jira, GitHub, Sentry), prefer an existing community/official MCP server over a custom one. Reserve custom servers for genuinely team-specific workflows. Less code to maintain, faster to adopt.

---

## 6. Anti-patterns (with *why*)
- **Hardcoding tokens in `.mcp.json`.** It's committed; you've leaked a secret. Use `${VAR}` expansion.
- **Personal/experimental server in project scope.** Pollutes everyone's config; put it in user scope (`~/.claude.json`).
- **Thinly described MCP tools.** The agent keeps using built-in `Grep` instead; enrich the description.
- **Building a custom Jira server.** Wasteful when a community server exists; build only team-specific tooling.
- **No resources for a known catalog.** The agent wastes calls probing for schema/issues you could have exposed as a resource.
- **Expecting mid-session stdio tools without reconnecting.** stdio tools load at session start; restart to pick them up.

---

## 7. Self-check (core mechanics)
1. Project vs user scope files? → `.mcp.json` (committed, team) vs `~/.claude.json` (private, all your projects).
2. How do you keep a token out of a committed config? → `${VAR}` expansion (read from the shell at connect time).
3. When are tools from configured servers available? → All discovered at connection time, available together.
4. What's an MCP resource for? → Exposing read-only catalogs (schemas, issue lists) to cut exploratory tool calls.
5. Why does the agent prefer `Grep` over a better MCP tool? → The MCP tool's description is too thin; enrich it.
6. Build or buy for Jira? → Buy (community server); custom only for team-specific workflows.
7. (Ties to §0) One sentence? → *Wire the right scope, hide secrets with `${VAR}`, expose resources, and describe tools so the agent uses them.*

---

## 8. Scenario 1 — Customer Support Resolution Agent

> **Scenario 1 (as given):** *Customer support resolution agent on the Agent SDK; high-ambiguity returns/billing/account requests; MCP tools `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`; target 80%+ first-contact resolution.*

**What this scenario is even about (plain English):** A support agent acting through four backend MCP tools (see Domain-1 §7).

**The key link to everything above:** **Strong fit.** Those four tools come from an MCP server you configure. Team-shared → **project scope** `.mcp.json`; the backend auth token → `${SUPPORT_API_TOKEN}` expansion (never committed). A **resource** exposing the refund-policy catalog or order-status schema lets the agent answer policy questions without extra probing calls.

### 8a. Project scope + env expansion
Commit the server definition; keep the token in the environment via `${SUPPORT_API_TOKEN}` so the whole support team shares config safely.

### 8b. Policy catalog as a resource
Expose the returns/refund policy as a resource so the agent reads eligibility rules directly instead of guessing or calling extra tools.

### 8c. Scenario-1 self-check
1. Scope for the shared support server? → Project (`.mcp.json`).
2. Where does the API token live? → Shell env via `${VAR}`; not in the file.
3. (Goal tie) How does a policy resource help FCR? → The agent resolves policy questions immediately, no exploratory calls.

---

## 9. Scenario 2 — Code Generation Pipeline

> **Scenario 2 (paraphrased — confirm against your exam copy):** *Multi-step code generation: plan → write → test → complete.*

**What this scenario is even about (plain English):** An agent writing code in stages (see Domain-1 §8).

**The key link to everything above:** **Partial-to-strong fit.** The pipeline pulls in MCP servers for test runners, package registries, or a build service — team-shared, so **project scope**. Exposing the project's test/build config or dependency manifest as a **resource** saves the agent from probing for how to run things.

### 9a. Shared toolchain in project scope
The CI/test MCP server lives in `.mcp.json` so every contributor's agent uses the same toolchain.

### 9b. Scenario-2 self-check
1. Scope for the shared test-runner server? → Project.
2. What cuts "how do I run tests?" probing? → A resource exposing the test/build config.

---

## 10. Scenario 3 — Multi-Source Research Coordinator

> **Scenario 3 (paraphrased — confirm against your exam copy):** *Coordinator spawns search/analysis subagents and a synthesis subagent for a cited answer.*

**What this scenario is even about (plain English):** A manager agent delegating research, then synthesizing (see Domain-1 §9).

**The key link to everything above:** **Strong fit.** Research integrates several MCP servers (web search, a docs server, a database). All their tools are **discovered together at connect time** (§4), so the coordinator sees the full toolset. A **documentation-hierarchy resource** or **database-schema resource** gives the agent a map up front, sharply cutting exploratory calls before it knows what's there.

### 10a. Multiple servers, one toolset
Configure search + docs + DB servers; their tools are available simultaneously to the coordinator.

### 10b. Catalogs as resources
Expose the doc hierarchy and DB schema as resources so subagents target the right source immediately instead of probing.

### 10c. Scenario-3 self-check
1. When are multi-server tools available? → All at connection time, together.
2. How to cut exploratory calls across sources? → Expose catalogs (doc tree, schema) as resources.

---

## 11. Scenario 4 — Developer Productivity with Claude

> **Scenario 4 (as given):** *Agent SDK over real repos; built-in tools Read, Write, Edit, Bash, Grep, Glob; integrates MCP; explores codebases, handles legacy, generates boilerplate, automates chores.*

**What this scenario is even about (plain English):** A coding assistant across a real repo (see Domain-1 §10).

**The key link to everything above:** **Core fit — the home scenario.** Every 2.4 idea lands: a shared GitHub/Jira server in **project** `.mcp.json` with `${GITHUB_TOKEN}` expansion; a personal experimental server in **user** `~/.claude.json`; **buy** the community Jira/GitHub server rather than building one; and crucially, **enrich descriptions so a capable MCP code-search tool isn't ignored in favor of built-in `Grep`** (§5) — the canonical example.

### 11a. Project vs user placement
Team's GitHub/Jira → project scope (committed, shared). Your experimental scratch server → user scope (private).

### 11b. Beat `Grep` with description quality
A semantic/cross-repo MCP search tool must advertise what it does beyond `Grep`, or the agent defaults to `Grep` and never uses it.

### 11c. Buy the standard integration
Use the official GitHub MCP server; reserve custom servers for your team's bespoke workflows.

### 11d. Scenario-4 self-check
1. Team GitHub server scope? → Project (`.mcp.json`), token via `${GITHUB_TOKEN}`.
2. Personal experiment scope? → User (`~/.claude.json`).
3. Why is the MCP search tool ignored? → Thin description; the agent prefers built-in `Grep`. Fix: enrich it.
4. Build a Jira server? → No — buy the community one; build only team-specific tools.

---

## 12. Scenario 5 — CI/CD Code Review Agent

> **Scenario 5 (paraphrased — confirm against your exam copy):** *Claude Code in CI reviews each PR headlessly (`claude -p`), emits JSON, a script gates the merge; minimize false positives.*

**What this scenario is even about (plain English):** An automated PR reviewer in the pipeline (see Domain-1 §11).

**The key link to everything above:** **Strong fit.** The CI environment must load the review tooling reproducibly, so the MCP server belongs in **project** `.mcp.json` (committed → every CI run identical), with the CI token supplied via env expansion from the pipeline's secret store. Exposing the repo's lint-rules or coding-standards as a **resource** gives the reviewer the project's standards directly, reducing false positives from guessing.

### 12a. Reproducible config in project scope
`.mcp.json` committed means the CI runner and every developer review the same way — determinism that matters in a gate.

### 12b. Standards as a resource
Expose coding standards/lint config as a resource so the reviewer judges against the real rules, not assumptions (fewer false positives).

### 12c. Scenario-5 self-check
1. Scope for reproducible CI tooling? → Project (`.mcp.json`), committed.
2. Where does the CI token come from? → Pipeline secret → `${VAR}` expansion.
3. How does a standards resource cut false positives? → The reviewer checks real rules instead of guessing.

---

## 13. Scenario 6 — Structured Data Extraction

> **Scenario 6 (paraphrased — confirm against your exam copy):** *Extract from unstructured docs, validate against JSON schemas, high accuracy, graceful edge cases, downstream integration.*

**What this scenario is even about (plain English):** Turn messy documents into clean schema-checked records (see Domain-1 §12).

**The key link to everything above:** **Strong fit.** The extractor connects MCP servers for document sources and the downstream system; shared → **project scope**, credentials via env expansion. The killer 2.4 move: expose the **target JSON schema (and any reference catalogs) as resources** so the agent reads the exact output shape up front — directly supporting accuracy and clean downstream integration — rather than inferring it.

### 13a. Schema as a resource
Publish the output schema as an MCP resource; the extractor reads it and produces conforming records, tightening accuracy.

### 13b. Project scope for shared sources
Document-source and downstream servers live in `.mcp.json` with `${VAR}` credentials so the whole pipeline shares config.

### 13c. Scenario-6 self-check
1. How to give the agent the exact output shape? → Expose the JSON schema as a resource.
2. Scope for shared document/downstream servers? → Project, with env-expanded credentials.
3. (Goal tie) How does a schema resource serve "high accuracy"? → The agent extracts against the real schema instead of guessing the shape.

---

### Sources verified against current Anthropic docs (June 2026)
- *Connect Claude Code to tools via MCP* (code.claude.com) — scopes: **project** (`.mcp.json`, committable/team), **user** (`~/.claude.json`, cross-project private), local (default, `~/.claude.json` keyed by project path); `claude mcp add --scope ...`, `-e`, `--`; duplicate servers resolved by highest-precedence scope (no field merge); `${VAR}` and `${VAR:-default}` expansion in `command`/`args`/`env`/`url`/`headers`, read from the shell at connection time; tools from all servers discovered at connection time; Tool Search defers definitions; SSE deprecated in favor of HTTP.
- *MCP specification — Resources* (modelcontextprotocol.io) — read-only, URI-addressed data (schemas, catalogs, docs) the model consumes as context.
- Build-vs-buy and "enrich descriptions so MCP beats built-in `Grep`" reflect the task statement and tool-description best practices. MCP config UX ships fast — re-verify flag/file specifics against current docs.
