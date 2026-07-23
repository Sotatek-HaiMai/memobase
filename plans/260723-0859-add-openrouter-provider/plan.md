---
title: "Add OpenRouter provider for LLM and embeddings"
description: "Register OpenRouter as a config-selectable provider for both chat completions and embeddings, reusing existing OpenAI-compatible code paths"
status: completed
priority: P2
effort: 1.5h
branch: main
tags: [backend, llm, config]
created: 2026-07-23
---

# Add OpenRouter Provider (LLM + Embedding)

## Goal

Let users set `llm_style: openrouter` and `embedding_provider: openrouter` in config to route both chat completions and embeddings through OpenRouter's OpenAI-compatible API — no new provider modules needed, since `openai_complete` / `openai_embedding` already work against any OpenAI-compatible `base_url`.

## Why (root cause / 80-20)

OpenRouter exposes an OpenAI-compatible `/chat/completions` and `/embeddings` API. The existing `openai_complete` and `openai_embedding` functions are style-agnostic (driven purely by `base_url`/`api_key` config). Adding OpenRouter support is pure registration + config plumbing — no new HTTP/client code required (YAGNI).

## Phases

| # | Phase | Status | File |
|---|-------|--------|------|
| 1 | Register OpenRouter in env config + factories, add example config, doc touch-up | completed | [phase-01-add-openrouter-llm-embedding-support.md](./phase-01-add-openrouter-llm-embedding-support.md) |

Single phase — this is a small, single-layer config/registration change (< 3 tasks), no separate testing/docs phases needed; both folded into phase 1.

## Key Dependencies

- None — purely additive, does not touch existing `openai`/`doubao_cache`/`jina`/`ollama`/`lmstudio` code paths.

## Validation

Manual: set `MEMOBASE_LLM_STYLE=openrouter` + `MEMOBASE_EMBEDDING_PROVIDER=openrouter` with a valid OpenRouter API key, start the server, confirm existing startup sanity checks (`llm_sanity_check()`, `check_embedding_sanity()`) pass. No new test file (reuses already-tested code paths).

## Completion Note

✓ All tasks completed successfully (2026-07-23). Config registration + base_url defaulting + example config + provider list docs all in place. Code review passed (9/10, 0 critical). Manual validation confirmed config-level behavior correct; full pytest suite pending local Postgres/Redis infra (acceptable per scope).
