# SynapCores skills for Claude Code

A [Claude Code](https://code.claude.com) plugin marketplace from **SynapCores**.
Install the plugin and Claude Code gains a deep, current skill for building on
[SynapCores AIDB](https://synapcores.com) — the AI-native database that unifies
SQL, vector search, a Cypher graph, LLM completion, an in-database agentic loop
(`AGENT_RUN`), durable agents, native image description, and MCP on one engine.

## Install

```
/plugin marketplace add SynapCores/synapcores-claude-plugins
/plugin install synapcores@synapcores
```

That's it — the next time you ask Claude Code to "build an app on SynapCores",
"call `AGENT_RUN` from Python", "query the graph", or "connect a MySQL driver to
SynapCores", it uses the bundled skill.

## What's inside

| Skill | What it gives Claude Code |
|---|---|
| **build-with-synapcores** | The production playbook for SynapCores AIDB — what the Python/Node SDKs actually expose, the REST/MySQL-wire/MCP surface, and copy-paste patterns for SQL, vectors (`EMBED`/`COSINE_SIMILARITY`), graph (Cypher), NL→SQL, transactions, recipes, filesystem RAG, agent memory (`MEMORY_*`), `AGENT_RUN`, durable agents (`CREATE AGENT`), streaming chat, AutoML, immutable/encrypted tables, and **native in-process vision** (`/v1/multimodal/describe`). **Verified against engine v1.14.0-ce.** |

This is the first skill in the marketplace — more may follow.

## Requirements

- Claude Code with plugin support.
- A SynapCores gateway to build against — self-host the Community Edition
  (`curl -fsSL https://get.synapcores.com | sh`, or Docker
  `synapcores/community:v1.14.0-ce`).

## Links

- SynapCores: https://synapcores.com
- Docs: https://docs.synapcores.com
- Recipes: https://synapcores.com/recipes/
- Engine source: https://github.com/mataluis2k/aidb

## License

Apache-2.0.
