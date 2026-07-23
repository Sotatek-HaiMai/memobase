<div align="center">
  <a href="https://memobase.io"><img alt="Memobase logo" src="./assets/images/logo.png" width="80%"></a>
  <h1>Memobase</h1>
  <p>
    <a href="https://pypi.org/project/memobase/"><img src="https://img.shields.io/pypi/v/memobase.svg"></a>
    <a href="https://www.npmjs.com/package/@memobase/memobase"><img src="https://img.shields.io/npm/v/@memobase/memobase.svg?logo=npm&logoColor=fff&style=flat&labelColor=2C2C2C&color=28CF8D"></a>
    <a href="https://pkg.go.dev/github.com/memodb-io/memobase/src/client/memobase-go"><img src="https://img.shields.io/badge/go-reference-blue?logo=go&logoColor=fff&style=flat&labelColor=2C2C2C&color=28CF8D" /></a>
    <a href="./src/mcp"><img src="https://img.shields.io/badge/MCP-Memobase-green"></a>
  </p>
</div>

Memobase is a **user profile-based long-term memory system** for LLM applications (virtual companions, tutors, assistants). Instead of raw RAG, it maintains structured user profiles and time-aware event timelines, optimized for search performance, LLM cost, and latency (<100ms via SQL).

## Key Metrics
- **Performance**: Top-tier search in LOCOMO benchmark vs mem0/langmem/zep
- **Cost**: Fixed 3 LLM calls per cycle (40-50% token reduction), batch-processed buffer
- **Latency**: Direct SQL queries, no agent chains—<100ms online reads

## Quick Start

**Prerequisite:** [Start local server](./src/server/readme.md) or use [Memobase Cloud](https://www.memobase.io/en/login) (free tier available).

```bash
pip install memobase
```

```python
from memobase import MemoBaseClient, ChatBlob

client = MemoBaseClient(project_url="...", api_key="...")
uid = client.add_user({"name": "Gus"})
user = client.get_user(uid)

# Insert chat
messages = [
    {"role": "user", "content": "I love Python"},
    {"role": "assistant", "content": "Great!"}
]
user.insert(ChatBlob(messages=messages))
user.flush(sync=True)

# Get memory context
print(user.context(max_token_size=500))
```

Also available: [Python SDK](https://pypi.org/project/memobase/) | [TypeScript SDK](./src/client/memobase-ts/) | [Go SDK](./src/client/memobase-go/) | [HTTP API](https://docs.memobase.io/api-reference/overview) | [MCP Server](./src/mcp)

## Contents

| Item | Purpose |
|------|---------|
| **Public Docs** | [docs.memobase.io](https://docs.memobase.io) — API reference, guides, architecture |
| **Internal Docs** | `docs/` — engineering reference, standards, roadmap |
| **Experiments** | `docs/experiments/` — benchmarks, evals (LOCOMO, 900-chat dataset) |
| **Contributing** | [CONTRIBUTING.md](./CONTRIBUTING.md), [Changelog](./Changelog.md) |

## Support

- [Discord](https://discord.gg/YdgwU4d9NB) | [Twitter](https://x.com/memobase_io) | [Email](mailto:contact@memobase.io)

## License

Apache 2.0 — see [LICENSE](LICENSE)
