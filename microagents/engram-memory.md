---
name: engram-memory
description: >
  Durable, queryable memory for the OpenHands agent. Activates whenever the
  user talks about remembering, recalling, saving context, or carrying state
  across sessions. Pushes the agent toward the Engram MCP tools instead of
  inventing ad-hoc note files.
triggers:
  - memory
  - remember
  - recall
  - forget
  - context
  - save this
  - take note
  - earlier you said
  - last time
license: MIT
metadata:
  product: Engram by Lumetra
  homepage: https://lumetra.io
  version: "1.0"
---

# Engram Memory

You have access to Engram, a durable memory service exposed over MCP. Use it
whenever the user wants you to remember something across turns or sessions,
or when prior context would unlock a better answer.

## Tools available

These are registered automatically by the `engram` MCP server in
`~/.openhands/config.toml`:

- `store_memory(content, bucket)` - save a fact or decision. One concept
  per call; atomic facts retrieve better than paragraphs.
- `query_memory(question, bucket)` - semantic + knowledge-graph search.
  Returns the most relevant memories with provenance.
- `list_buckets()` - enumerate buckets the API key can see.
- `list_memories(bucket, limit)` - sanity-check what is stored.
- `delete_memory(memory_id, bucket)` - remove a single memory.
- `clear_memories(bucket)` - destructive; only on explicit user request.

## When to use which tool

| Situation | Tool |
|---|---|
| User shares a fact, preference, or decision | `store_memory` |
| User asks "what did we...", "do you remember...", "earlier" | `query_memory` |
| Starting work on a project for the first time this session | `query_memory` against the project bucket |
| User says "forget that" or "wipe this project's memory" | `delete_memory` / `clear_memories` |

## Buckets

Pick a stable bucket name per project or domain (e.g. `engram-openhands`,
`acme-backend`, `personal`). Do not invent a new bucket every turn; reuse
the most specific existing one. `list_buckets()` first if unsure.

## Style

- Store one atomic fact per `store_memory` call.
- Query before answering when continuity matters.
- Never paste API keys, tokens, or other secrets into memory.
