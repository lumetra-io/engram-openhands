---
name: engram-memory
description: >
  Durable memory for the OpenHands agent via the Engram MCP server. Use when
  the user wants to remember, recall, or carry context across sessions.
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

Registered automatically by the `engram` MCP server in `~/.openhands/config.toml`:

- `store_memory(content, bucket)` - save a fact. One concept per call.
- `query_memory(question, bucket)` - semantic + knowledge-graph search.
- `list_buckets()` - enumerate buckets the API key can see.
- `list_memories(bucket, limit)` - sanity-check what is stored.
- `delete_memory(memory_id, bucket)` - remove a single memory.
- `clear_memories(bucket)` - destructive; only on explicit user request.

## When to use which

| Situation | Tool |
|---|---|
| User shares a fact, preference, or decision | `store_memory` |
| User asks "what did we...", "do you remember..." | `query_memory` |
| Starting on a known project | `query_memory` against its bucket |
| User says "forget that" | `delete_memory` / `clear_memories` |

## Buckets

Pick a stable bucket per project (e.g. `engram-openhands`, `acme-backend`).
Reuse the most specific existing one; do not invent a new bucket each turn.

## Style

- One atomic fact per `store_memory` call.
- Query before answering when continuity matters.
- Never store API keys, tokens, or other secrets.
