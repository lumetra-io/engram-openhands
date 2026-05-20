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

```bash
git clone https://github.com/lumetra-io/engram-openhands.git
cd engram-openhands

# 1. Drop the MCP config where OpenHands will read it.
mkdir -p ~/.openhands
cp config.example.toml ~/.openhands/config.toml
# Then open it and replace REPLACE_WITH_YOUR_ENGRAM_KEY with your real key.

# 2. Drop the microagent into the project you want Engram-aware.
#    (Both locations work; the .agents path is the current recommendation,
#    microagents/ is the legacy path and still supported.)
mkdir -p /path/to/your/project/.agents/skills
cp -r .agents/skills/engram-memory /path/to/your/project/.agents/skills/
# OR
mkdir -p /path/to/your/project/microagents
cp microagents/engram-memory.md /path/to/your/project/microagents/
```

## Run OpenHands

The canonical launch command from the OpenHands docs, with our config mount
added:

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

Then open <http://localhost:3000>, pick your LLM provider on first launch,
and start a conversation. Confirm Engram is wired in:

- Open **Settings -> MCP**. You should see `https://mcp.lumetra.io/mcp/sse`
  listed under SSE servers.
- In a new conversation, ask: *"List the available Engram buckets."* The
  agent should call `list_buckets` and return the response.

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
