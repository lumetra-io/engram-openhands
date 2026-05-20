# Engram for OpenHands

Wire [Engram](https://lumetra.io) - durable, explainable memory - into the
[OpenHands](https://github.com/OpenHands/OpenHands) autonomous coding agent in
under five minutes. Pure GitHub recipe: no PyPI package, no npm install,
nothing to publish. Clone, copy two files, restart OpenHands.

## What you get

1. An MCP server entry that exposes Engram's six memory tools to the
   OpenHands agent (`store_memory`, `query_memory`, `list_buckets`,
   `list_memories`, `delete_memory`, `clear_memories`).
2. A keyword-triggered microagent / skill that nudges the agent to actually
   reach for those tools when the user says things like "remember this" or
   "what did we decide last week."

## Prerequisites

- Docker (the official OpenHands launch path).
- An Engram API key. Grab one at <https://lumetra.io>; it looks like
  `eng_live_...`.
- An LLM key OpenHands can use (Anthropic, OpenAI, etc.).

## Install

> ### Two install paths — pick the one that matches how you use OpenHands
>
> OpenHands 1.x reads MCP settings from **`settings.json`** at runtime, NOT from `config.toml`. The `config.toml` shape still works because OpenHands' **UI** ("Settings → MCP") merges your toml into `settings.json` on first save — but a fresh container with a mounted `config.toml` and no UI step will silently ignore the file (the server logs `config.toml not found`, and `settings.json` stays at defaults). Verified end-to-end in our 2026-05-20 e2e smoke against `docker.openhands.dev/openhands/openhands:1.7`.
>
> - **Path A (UI / interactive)** — mount `config.toml`, then open the UI once so OpenHands writes the merged settings to disk. Easiest if you're going to use the app anyway. The toml below is what to mount.
> - **Path B (headless / CI / scripted)** — skip `config.toml` entirely and `POST /api/v1/settings` with the diff payload below. This is the only path that works without a human click.

```bash
git clone https://github.com/lumetra-io/engram-openhands.git
cd engram-openhands

# Drop the microagent into the project you want Engram-aware.
# Both locations work; the .agents path is the current recommendation,
# microagents/ is the legacy path and still supported.
mkdir -p /path/to/your/project/.agents/skills
cp -r .agents/skills/engram-memory /path/to/your/project/.agents/skills/
# OR
mkdir -p /path/to/your/project/microagents
cp microagents/engram-memory.md /path/to/your/project/microagents/
```

## Path A — UI install via `config.toml`

```bash
mkdir -p ~/.openhands
cp config.example.toml ~/.openhands/config.toml
# Then open it and replace REPLACE_WITH_YOUR_ENGRAM_KEY with your real key.
```

Launch OpenHands (mounts `~/.openhands` into the container):

```bash
docker pull docker.openhands.dev/openhands/openhands:1.7

docker run -it --rm --pull=always \
  -e AGENT_SERVER_IMAGE_REPOSITORY=ghcr.io/openhands/agent-server \
  -e AGENT_SERVER_IMAGE_TAG=1.19.1-python \
  -e LOG_ALL_EVENTS=true \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/.openhands:/.openhands \
  -p 3000:3000 \
  --add-host host.docker.internal:host-gateway \
  --name openhands-app \
  docker.openhands.dev/openhands/openhands:1.7
```

Open <http://localhost:3000> → pick your LLM provider → visit **Settings → MCP** at least once (this triggers the toml→settings.json merge) → start a conversation. Confirm Engram is wired in by asking: *"List the available Engram buckets."* The agent should call `list_buckets`.

## Path B — headless install via the settings API

Skip mounting any config. Boot the container the same way (you can drop the `-v ~/.openhands:/.openhands` flag) and POST the settings diff once OpenHands is healthy:

```bash
curl -X POST http://localhost:3000/api/v1/settings \
  -H 'Content-Type: application/json' \
  -d '{
    "agent_settings_diff": {
      "llm": {
        "model": "anthropic/claude-sonnet-4-5",
        "api_key": "sk-ant-..."
      },
      "mcp_config": {
        "mcpServers": {
          "engram": {
            "url": "https://mcp.lumetra.io/mcp/sse",
            "transport": "sse",
            "auth": "eng_live_..."
          }
        }
      }
    }
  }'
```

A few gotchas the e2e caught:

- The endpoint **requires the diff shape** — `agent_settings_diff`, not `agent_settings`. A naive flat payload is rejected with `"Use *_diff nested settings payloads instead of legacy keys"`.
- `mcp_config.mcpServers.engram.auth` takes the bearer token **as a bare string** (fastmcp's `RemoteMCPServer.auth` field). It is NOT `{"Authorization": "Bearer …"}` like in most other MCP clients. Just `"auth": "eng_live_..."`.
- The legacy `sse_servers = [{ url, api_key }]` toml shape (from older OpenHands releases) uses `api_key`. The new settings.json shape uses `auth`. Both are the same string value, just named differently.

After the POST, kick off a conversation via `POST /api/v1/app-conversations` (see the OpenHands API docs) and watch the event stream for `tool_name: engram_*` calls.

## How the pieces fit together

| File | Purpose |
|---|---|
| `config.example.toml` | The `[mcp]` block OpenHands reads at boot. Mounts at `/.openhands/config.toml` inside the container. |
| `microagents/engram-memory.md` | Legacy-path keyword-triggered microagent. Lives next to your project. |
| `.agents/skills/engram-memory/SKILL.md` | Current-path Skill with the same frontmatter. Pick whichever path your OpenHands version prefers. |

## MCP server details

- Endpoint: `https://mcp.lumetra.io/mcp/sse` (SSE)
- Transport: Server-Sent Events (an SHTTP endpoint is also available;
  uncomment the `shttp_servers` block in `config.example.toml` to use it).
- Auth: Bearer token in the `Authorization` header. OpenHands' SSE client
  sends the `api_key` field as `Authorization: Bearer <key>`.
- Tools registered: `store_memory`, `query_memory`, `list_buckets`,
  `list_memories`, `delete_memory`, `clear_memories`.

## Troubleshooting

**Engram tools don't appear in the agent's tool list.**
Check Settings -> MCP. If the server is listed but tools aren't showing,
restart the conversation - tool discovery happens at conversation start.

**`config.toml` looks ignored.**
Make sure you mounted `~/.openhands:/.openhands` (note the leading slash
inside the container). The OpenHands docker image looks for config at
`/.openhands/config.toml`, not `/openhands/config.toml`.

**401 from `mcp.lumetra.io`.**
Your `api_key` is missing or wrong. The key must start with `eng_live_`.
Regenerate at <https://lumetra.io>.

## License

MIT - see [LICENSE](./LICENSE).
