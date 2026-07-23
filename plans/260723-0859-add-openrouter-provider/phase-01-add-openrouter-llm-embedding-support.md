---
title: "Phase 1 — Add OpenRouter provider support"
status: completed
priority: P2
---

# Phase 1: Add OpenRouter LLM + Embedding Support

## Context Links
- Config: `src/server/api/memobase_server/env.py`
- LLM factory: `src/server/api/memobase_server/llms/__init__.py`
- LLM client: `src/server/api/memobase_server/llms/utils.py`, `openai_model_llm.py`
- Embedding factory: `src/server/api/memobase_server/llms/embeddings/__init__.py`
- Embedding client: `src/server/api/memobase_server/llms/embeddings/utils.py`, `openai_embedding.py`
- Example configs: `src/server/api/example_config/jina_embedding/config.yaml`, `.../ollama_embedding/config.yaml`
- Root example: `src/server/api/config.yaml.example`

## Overview
- Priority: P2
- Status: pending
- Register `"openrouter"` as a valid value for `llm_style` and `embedding_provider`, mapping both to the existing OpenAI-compatible implementations. Add sensible base_url defaulting and a combined example config.

## Key Insights
- `openai_complete()` and `openai_embedding()` are already style-agnostic — driven by `CONFIG.llm_base_url`/`llm_api_key` and `CONFIG.embedding_base_url`/`embedding_api_key` respectively. OpenRouter's API is OpenAI-compatible for both chat completions and embeddings (confirmed: OpenRouter unified `/embeddings` endpoint, per OpenRouter docs, 2026-07).
- No new provider module needed — just add `FACTORIES["openrouter"]` entries pointing at the existing functions.
- Existing precedent for base_url defaulting: `env.py.__post_init__` already defaults `embedding_base_url` for `embedding_provider == "jina"`. Follow the same pattern for openrouter on both LLM and embedding sides.
- Pre-existing bug to fix opportunistically: `embedding_provider` Literal type is missing `"lmstudio"` even though it's a valid runtime value in `FACTORIES` — 1-line fix while touching this line.
- Risk: OpenRouter may not honor the `dimensions` param in `embeddings.create()` for all routed models (unlike native OpenAI). Mitigate by documenting a known-good model (`openai/text-embedding-3-small`) in the example config rather than adding fallback/retry logic (YAGNI).
- OpenRouter's recommended `HTTP-Referer`/`X-Title` attribution headers require zero code changes — already supported via existing `llm_openai_default_header` config field. Document as a usage tip only.

## Requirements
- `llm_style: "openrouter"` routes chat completions through OpenRouter.
- `embedding_provider: "openrouter"` routes embeddings through OpenRouter.
- Both usable independently or together.
- No breaking changes to existing providers.

## Related Code Files
**Modify:**
- `src/server/api/memobase_server/env.py`
- `src/server/api/memobase_server/llms/__init__.py`
- `src/server/api/memobase_server/llms/embeddings/__init__.py`
- `src/server/api/config.yaml.example` (optional one-line comment pointer)
- `docs/system-architecture.md` or `docs/code-standards.md` (add "openrouter" to provider list — grep for `doubao_cache`/`jina` to find exact spot)

**Create:**
- `src/server/api/example_config/openrouter/config.yaml`

## Implementation Steps
1. In `env.py`: extend `llm_style: Literal["openai", "doubao_cache", "openrouter"]`.
2. In `env.py`: extend `embedding_provider: Literal["openai", "jina", "ollama", "lmstudio", "openrouter"]` (also fixes missing `lmstudio`).
3. In `env.py.__post_init__`: add, mirroring the jina pattern —
   ```python
   if self.llm_style == "openrouter":
       self.llm_base_url = self.llm_base_url or "https://openrouter.ai/api/v1"
   if self.embedding_provider == "openrouter":
       self.embedding_base_url = self.embedding_base_url or "https://openrouter.ai/api/v1"
   ```
4. In `llms/__init__.py`: add `"openrouter": openai_complete` to `FACTORIES`.
5. In `llms/embeddings/__init__.py`: add `"openrouter": openai_embedding` to `FACTORIES`.
6. Create `src/server/api/example_config/openrouter/config.yaml` with combined LLM + embedding example: `llm_style: openrouter`, `llm_api_key`, `llm_model: openai/gpt-4o-mini` (or `anthropic/claude-3.5-haiku`), `embedding_provider: openrouter`, `embedding_api_key`, `embedding_model: openai/text-embedding-3-small`, `embedding_dim: 1536`. Include comment noting `HTTP-Referer`/`X-Title` can be set via `llm_openai_default_header`.
7. Grep `docs/system-architecture.md` and `docs/code-standards.md` for `doubao_cache` or `jina` to find the provider list/table; add an "openrouter" row/entry to both LLM and embedding provider lists.
8. Optionally add a one-line comment in `src/server/api/config.yaml.example` pointing to `example_config/openrouter/config.yaml` (matching existing lmstudio-style pointer comment, if present).

## Todo List
- [x] Update `env.py` Literals + `__post_init__` defaulting
- [x] Register `openrouter` in `llms/__init__.py` FACTORIES
- [x] Register `openrouter` in `llms/embeddings/__init__.py` FACTORIES
- [x] Create `example_config/openrouter/config.yaml`
- [x] Update provider list in docs
- [x] Manually validate: set env vars, start server, confirm `llm_sanity_check()` and `check_embedding_sanity()` pass

## Success Criteria
- Server starts successfully with `MEMOBASE_LLM_STYLE=openrouter` and/or `MEMOBASE_EMBEDDING_PROVIDER=openrouter` set, with a valid OpenRouter API key.
- Startup sanity checks (`llm_sanity_check()`, `check_embedding_sanity()`) pass without code changes to the checks themselves.
- No regression for existing providers (openai, doubao_cache, jina, ollama, lmstudio).

## Risk Assessment
- **Embedding `dimensions` param may be rejected by some OpenRouter-routed models** → mitigate via documented known-good model in example config, not code changes.
- **Low risk overall** — purely additive registration, reuses tested code paths.

## Security Considerations
- API key handled the same way as existing providers (env var / config file, not committed). No new auth surface.

## Next Steps
- None — single-phase change, no follow-up phases required.
