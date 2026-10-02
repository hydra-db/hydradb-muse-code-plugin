# HydraDB for Muse Code

Persistent, cross-session memory for [Muse Code](https://dev.meta.ai/docs/muse-code)
(Meta's terminal coding agent), powered by [HydraDB](https://hydradb.com).

Muse Code exposes three extension points - **MCP**, **skills**, and **hooks** -
and this integration uses all three over the same shared engine
(`scripts/plugin.mjs`) as the Claude Code / Codex / Cursor / OpenCode plugins.

## How recall works on Muse Code

Muse Code hooks **cannot inject context back to the model** (per the
[docs](https://dev.meta.ai/docs/muse-code/extending): hooks "enforce checks,
format code, or block actions"). So there is **no per-prompt auto-recall via
hooks** here - recall is agent-invoked, through **MCP** or **skills**:

| Extension point | Role | HydraDB use |
|---|---|---|
| **MCP** (`mcp_servers`) | tools the agent may call | recall (`hydradb_query`) + ingest |
| **Skills** (`.claude/skills/`) | `/`-invoked slash commands | manual recall / remember / sync |
| **Hooks** (`.muse/hooks.json`) | run commands on lifecycle events | **automatic capture + doc sync** |

## Setup

### 1. MCP (recall)

Add HydraDB to your Muse settings `mcp_servers` block (see `settings.snippet.json`):

```json
{
  "mcp_servers": {
    "hydradb": {
      "transport": "streamable_http",
      "url": "https://mcp.hydradb.com",
      "headers": { "Authorization": "Bearer ${HYDRADB_API_KEY}" },
      "enabled": true,
      "mode": "optional"
    }
  }
}
```

### 2. Skills (slash commands)

Muse Code auto-discovers skills from `.claude/skills/` (and `.codex/skills`,
`.agents/skills`). This repo ships the HydraDB skills there, so `/hydradb …`
commands are available once the workspace is trusted. Manage with
`muse skills list` / `muse skills enable <id> --scope project`.

### 3. Hooks (automatic capture + sync)

`.muse/hooks.json` wires lifecycle events to the engine:

| Event | Engine action |
|---|---|
| `SessionStart` | sync workspace docs into HydraDB |
| `PostToolUse` (file writes) | incremental sync |
| `Stop` | capture the completed turn |

Then set credentials (resolved by the shared engine):

```bash
export HYDRADB_API_KEY="your-api-key"
export HYDRADB_TENANT_ID="your-tenant-id"
```

## Running (two flags are required)

Muse gates project skills/hooks behind workspace trust, and the recall skill
needs permission to run `node`. Both are required or recall silently no-ops:

```bash
# --trust-workspace  loads the project skills + hooks (else they are skipped)
# --yolo             allows the auto-recall skill to run Bash(node *) (or use an
#                    approval profile that permits it) so recall can execute
muse exec --trust-workspace --yolo "what did we decide about X?"
```

Verified end to end: the agent invokes the model-invocable `auto-recall` skill,
which queries HydraDB and answers from recalled context. Without `--yolo` (or an
equivalent approval profile) the skill's tool call is blocked in headless mode.

## To verify against your Muse build

- The exact `.muse/hooks.json` schema and whether hook commands run with the
  repo root as cwd (the commands use `./scripts/run-plugin.sh`, which self-resolves
  the plugin root from its own location).
- The auth handshake for `mcp.hydradb.com` (bearer header vs. an MCP auth step).

## License

Apache-2.0 - Copyright (c) 2026 HydraDB
