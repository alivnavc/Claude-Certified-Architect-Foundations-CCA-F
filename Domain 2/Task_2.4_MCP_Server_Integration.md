# Task 2.4 — SUPPLEMENT: Attaching MCP Servers to Agents in CODE
### The two programmatic paths the main guide skipped — Messages API connector vs Claude Agent SDK

The main 2.4 guide covered MCP **inside Claude Code** (via `.mcp.json`). But when you're *building your own agent in code*, there are two other ways to attach MCP servers. This supplement shows both, with runnable code, and exactly how they differ.

---

## 0. The three places MCP can be wired (so you don't mix them up)

| Where | How you attach MCP | Who runs the agent loop |
|---|---|---|
| **Claude Code** (the CLI/app) | `.mcp.json` / `~/.claude.json` (main guide) | Claude Code |
| **Messages API** (raw `/v1/messages`) | `mcp_servers` parameter (the **MCP connector**) | **You** drive turns; Anthropic's API is the MCP client |
| **Claude Agent SDK** (`claude-agent-sdk`) | `mcp_servers` in `ClaudeAgentOptions` | The **SDK** runs the full loop for you |

This supplement is about rows 2 and 3.

---

# PATH 1 — THE MESSAGES API "MCP CONNECTOR"

## 1.1 The idea (this is the surprising part)

Normally *you* are the MCP client: your code connects to the MCP server, calls `tools/list`/`tools/call`, etc. The **MCP connector** flips that — you just name a **remote MCP server URL** in your API request, and **Anthropic's API itself becomes the MCP client**. Anthropic connects to that server during inference, discovers its tools, lets the model call them, executes the calls server-side, and returns the results to you. You write no MCP client code at all.

**Key constraints (memorize these — they're exam-favorite distinctions):**
- **Remote only.** The connector accepts `"type": "url"` servers (HTTP). It cannot launch a **local stdio** server. (Local servers → use the Agent SDK, Path 2.)
- **Tools only.** It uses `tools/list` and `tools/call` — **resources and prompts are not exposed**, even if the server has them.
- **Beta.** It requires a beta header / `betas` flag.
- Not available on Amazon Bedrock or Google Vertex.

## 1.2 The code (current pattern, beta `mcp-client-2025-11-20`)

```python
import anthropic
client = anthropic.Anthropic()

response = client.beta.messages.create(
    model="claude-opus-4-8",
    max_tokens=1000,
    messages=[{"role": "user", "content": "Create a GitHub issue for the login bug."}],

    # 1) Name the REMOTE MCP server(s) — Anthropic connects to these for you:
    mcp_servers=[
        {
            "type": "url",                                   # remote only
            "url": "https://api.githubcopilot.com/mcp/",
            "name": "github",
            "authorization_token": "YOUR_OAUTH_TOKEN",       # optional; pre-acquired
        }
    ],

    # 2) Turn the server's tools on via an MCPToolset in the tools array
    #    (this is the NEW location for tool config — see 1.3):
    tools=[
        {"type": "mcp_toolset", "mcp_server_name": "github"}   # all tools from "github"
    ],

    betas=["mcp-client-2025-11-20"],                          # the beta flag
)
print(response.content)
```

Or as raw HTTP (note the header instead of `betas`):

```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: mcp-client-2025-11-20" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-opus-4-8",
    "max_tokens": 1000,
    "messages": [{"role": "user", "content": "Create a GitHub issue for the login bug."}],
    "mcp_servers": [
      {"type": "url", "url": "https://api.githubcopilot.com/mcp/", "name": "github",
       "authorization_token": "YOUR_OAUTH_TOKEN"}
    ],
    "tools": [{"type": "mcp_toolset", "mcp_server_name": "github"}]
  }'
```

## 1.3 Choosing which tools (allowlist/denylist) — and a version note

The `MCPToolset` in the `tools` array is where you scope which of the server's tools are usable (this is Task 2.3's distribution applied to the connector):

```python
tools=[{
    "type": "mcp_toolset",
    "mcp_server_name": "github",
    "allowed_tools": ["create_issue", "list_pull_requests"],   # only these two
    # (denied_tools and per-tool defer_loading are also supported)
}]
```

**Version note (important):** the *older* connector beta (`mcp-client-2025-04-04`, now deprecated) put this config **inside the server object** as `tool_configuration`, not in a separate `tools` array:

```python
# OLD / deprecated style (mcp-client-2025-04-04):
mcp_servers=[{
    "type": "url", "url": "...", "name": "github", "authorization_token": "...",
    "tool_configuration": {"enabled": True, "allowed_tools": ["create_issue"]}
}]
# extra_headers={"anthropic-beta": "mcp-client-2025-04-04"}
```
If you see `tool_configuration` inside the server object in a tutorial, it's the old pattern. New code puts an `mcp_toolset` in `tools`.

## 1.4 Reading the result

When the model uses a connector tool, the response contains **`mcp_tool_use`** and **`mcp_tool_result`** blocks (note the `mcp_` prefix — distinct from normal `tool_use`):

```python
for block in response.content:
    if block.type == "mcp_tool_use":
        print("Called:", block.name, block.input)
    elif block.type == "mcp_tool_result":
        print("Result error?", block.is_error)
        print("Result:", block.content)
```

Because Anthropic executed the tool for you, you don't run a tool loop here — the result already came back in one response.

---

# PATH 2 — THE CLAUDE AGENT SDK

## 2.1 The idea

The **Claude Agent SDK** (`claude-agent-sdk` in Python, `@anthropic-ai/claude-agent-sdk` in TS) gives you the *entire Claude Code agent loop* callable from your own program — tool execution, context management, permissions, subagents — without an interactive terminal. (It's the renamed successor of the "Claude Code SDK.") Here, **the SDK is the MCP client**, and it supports **all three** server kinds:

| Type in `mcp_servers` | What it is | Config |
|---|---|---|
| **`stdio`** | A **local** process the SDK launches | `command` + `args` + `env` |
| **`http`** / **`sse`** | A **remote** server by URL | `url` + `headers` |
| **`sdk`** | An **in-process** server — your own Python/TS functions, no subprocess | `create_sdk_mcp_server(...)` |

That last one (`sdk`) is unique to the Agent SDK: your tools run *inside your application*, no separate process, no network — fastest and simplest for custom tools.

## 2.2 Local (stdio) and remote (http) servers — code

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

options = ClaudeAgentOptions(
    mcp_servers={
        # LOCAL (stdio): the SDK runs this command on your machine
        "github": {
            "type": "stdio",
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-github"],
            "env": {"GITHUB_TOKEN": "${GITHUB_TOKEN}"},     # ${VAR} expansion works
        },
        # REMOTE (http): the SDK connects to this URL
        "docs": {
            "type": "http",
            "url": "https://code.claude.com/docs/mcp",
            "headers": {"Authorization": "Bearer ${API_TOKEN}"},
        },
    },
    # YOU MUST allowlist the tools (see 2.4) or the agent can't call them:
    allowed_tools=["mcp__github__create_issue", "mcp__docs__*"],
)

async def main():
    async for msg in query(prompt="Open an issue for the login bug.", options=options):
        if isinstance(msg, ResultMessage) and msg.subtype == "success":
            print(msg.result)

asyncio.run(main())
```

Note: unlike the Messages API connector, the SDK **runs the whole loop** — it discovers tools, lets the model call them, executes them, and continues until done. You just read the streamed messages.

## 2.3 In-process (`sdk`) server — define tools as plain functions

No subprocess, no URL — your own functions become MCP tools:

```python
from claude_agent_sdk import tool, create_sdk_mcp_server, ClaudeAgentOptions

# 1) Define tools with the @tool decorator (name, description, input schema)
@tool("add", "Add two numbers", {"a": float, "b": float})
async def add(args):
    return {"content": [{"type": "text", "text": f"Sum: {args['a'] + args['b']}"}]}

@tool("multiply", "Multiply two numbers", {"a": float, "b": float})
async def multiply(args):
    return {"content": [{"type": "text", "text": f"Product: {args['a'] * args['b']}"}]}

# 2) Bundle them into an in-process MCP server
calculator = create_sdk_mcp_server(name="calc", version="1.0.0", tools=[add, multiply])

# 3) Attach it exactly like any other MCP server
options = ClaudeAgentOptions(
    mcp_servers={"calc": calculator},                        # the SDK server object
    allowed_tools=["mcp__calc__add", "mcp__calc__multiply"], # allowlist
)
```

## 2.4 THE GOTCHA: connecting ≠ allowed (the `allowed_tools` rule)

This trips everyone. In the Agent SDK, **merely attaching a server does not let the agent call its tools.** You must list them in `allowed_tools` using the namespaced name `mcp__<server>__<tool>` (wildcards like `mcp__github__*` work):

```python
allowed_tools=["mcp__github__create_issue"]   # auto-approved, runs without a prompt
# Anything not listed falls through to the permission flow / is blocked.
# To explicitly block tools, use disallowed_tools=[...].
```

So `allowed_tools` is the Agent SDK's tool-distribution lever (Task 2.3): scope each agent to exactly the namespaced tools its role needs, and nothing else.

## 2.5 You can mix all three in one agent

```python
options = ClaudeAgentOptions(
    mcp_servers={
        "calc":     calculator,                                       # in-process (sdk)
        "github":   {"type": "stdio", "command": "npx",
                     "args": ["-y", "@modelcontextprotocol/server-github"]},  # local
        "docs":     {"type": "http", "url": "https://code.claude.com/docs/mcp"}, # remote
    },
    allowed_tools=["mcp__calc__*", "mcp__github__create_issue", "mcp__docs__*"],
)
```

All three are discovered at startup and merged into one namespaced tool list (the Part C mechanism from the main guide). The model picks by description; the `mcp__<server>__` prefix routes each call to the right server — local, remote, or in-process alike.

## 2.6 The Agent SDK also reads `.mcp.json`

The SDK can pick up the same `.mcp.json` file Claude Code uses, so project servers are shared between the CLI and your SDK app. You can define servers in code (above) *or* in `.mcp.json` — your choice.

---

# PART 3 — WHICH PATH SHOULD YOU USE? (decision table)

| If you want… | Use |
|---|---|
| A **remote** MCP server, minimal code, you control each turn, no client to run | **Messages API connector** (`mcp_servers` param) |
| **Local stdio** servers, **in-process** custom tools, the **full agent loop** handled for you, subagents/hooks/permissions, **resources** support | **Claude Agent SDK** |
| MCP inside the **interactive CLI / app** for developers | **Claude Code** + `.mcp.json` (main guide) |

Quick rules of thumb:
- **Server gives you a command to run** (`npx ...`) → it's stdio → **Agent SDK** (connector can't do stdio).
- **Server gives you a URL** → either works; pick by whether you want the SDK's full loop or the connector's lightness.
- **Your own custom functions as tools** → Agent SDK **`sdk`** (in-process) server.

---

# PART 4 — ONE-SCREEN RECAP

- **Messages API MCP connector:** add `mcp_servers=[{"type":"url", "url":..., "name":..., "authorization_token":...}]` + an `mcp_toolset` in `tools` + `betas=["mcp-client-2025-11-20"]`. **Remote (URL) only, tools-only, beta.** Anthropic's API acts as the MCP client and executes calls for you; results come back as `mcp_tool_use`/`mcp_tool_result` blocks. (Old `mcp-client-2025-04-04` put config in `tool_configuration` inside the server — deprecated.)
- **Claude Agent SDK:** set `ClaudeAgentOptions(mcp_servers={...}, allowed_tools=[...])`. Three server types: **`stdio`** (local command), **`http`/`sse`** (remote URL), **`sdk`** (in-process via `create_sdk_mcp_server` + `@tool`). The SDK runs the whole loop. **You MUST allowlist tools** as `mcp__<server>__<tool>` — attaching alone isn't enough.
- Both namespace tools `mcp__<server>__<tool>`; the prefix routes calls. Connector = remote-only/lightweight; SDK = all transports + full agent loop + resources.

---

### Sources verified against current Anthropic docs (June 2026)
- *MCP connector* (platform.claude.com) — `mcp_servers` array on the Messages API; **remote `type:"url"` only**; new beta header **`mcp-client-2025-11-20`** with tool config moved to the `tools` array as **`mcp_toolset`** objects (old `mcp-client-2025-04-04` used `tool_configuration` inside the server, now deprecated); connector calls only `tools/list`/`tools/call` (no resources/prompts); responses carry `mcp_tool_use`/`mcp_tool_result` blocks; supports `allowed_tools`/`denied_tools`/per-tool `defer_loading`; not on Bedrock/Vertex.
- *Connect MCP servers / Agent SDK reference (Python & TS)* (platform.claude.com, code.claude.com, github.com/anthropics/claude-agent-sdk-python) — `ClaudeAgentOptions(mcp_servers={...}, allowed_tools=[...])`; server types `stdio` (command/args/env), `http`/`sse` (url/headers), and `sdk` via `create_sdk_mcp_server(name, version, tools)` with `@tool`-decorated functions (in-process, no subprocess); tools namespaced `mcp__<server>__<tool>`; **attaching a server does NOT grant use — you must allowlist via `allowed_tools`** (wildcards allowed; `disallowed_tools` to block); `${VAR}` expansion in `.mcp.json`; the SDK can also read `.mcp.json`.
- Both surfaces are beta/fast-moving (the connector header changed from `2025-04-04`→`2025-11-20`; "Claude Code SDK" was renamed "Claude Agent SDK") — re-verify header/field names against current docs before production.
