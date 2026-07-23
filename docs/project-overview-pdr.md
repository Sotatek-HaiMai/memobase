# Memobase Project Overview & Product Development Requirements

## Executive Summary

Memobase is a user profile-based long-term memory system for LLM applications. It addresses the limitations of raw RAG systems by maintaining structured, evolving user profiles and time-aware event timelines, optimized for production requirements: <100ms latency (SQL-based), 40-50% LLM cost reduction, and SOTA search performance.

**License**: Apache 2.0

## Product Vision

Enable LLM applications (virtual companions, tutors, assistants) to remember and understand users over time with minimal overhead. Unlike RAG-centric approaches, Memobase treats memory as a structured, user-centric data model that evolves with interaction.

## Functional Requirements

### Core Data Model
- **User Profiles**: Hierarchical, structured attribute storage (topic/sub_topic/content) for user characteristics
- **Event Timeline**: Time-stamped, searchable events with embeddings and fine-grained gists
- **Buffer Zone**: Per-user pending-data staging for batch processing
- **Blobs**: Polymorphic data ingestion (ChatBlob, SummaryBlob, DocBlob, ImageBlob, CodeBlob, TranscriptBlob)
- **Profile Configuration**: Per-project customizable schema for profile attributes

### Core APIs
- **Project-level**: Billing, configuration, usage analytics
- **User CRUD**: Add, get, update, delete users with arbitrary metadata
- **Blob ingestion**: Insert/retrieve/delete typed data (chat, summary, doc, etc.)
- **Profile management**: Get/import/delete profiles, bulk operations
- **Event search**: Query by content, gist, tags with temporal filters
- **Context assembly**: Retrieve formatted memory string for LLM prompting (customizable prompt template, topic filters, token budgeting)
- **Buffer flush**: Manual or auto-triggered (size/idle threshold)
- **Roleplay features**: Proactive topic detection, user interest prediction (experimental)

### Non-Functional Requirements

| Requirement | Target | Mechanism |
|-------------|--------|-----------|
| **Latency** | <100ms | Direct SQL queries, no agent chains |
| **LLM Cost** | 40-50% reduction | Fixed 3 calls/cycle (extract→merge→organize), batch buffer |
| **Search Performance** | SOTA | pgvector embeddings, fine-grained event gists |
| **Multi-tenancy** | Full support | Project ID + bearer token auth, composite PKs (id, project_id) |
| **Scalability** | Horizontal | PostgreSQL, Redis async, stateless FastAPI servers |
| **Availability** | Production-ready | Docker, health checks, error handling, telemetry, structured logging |

## Architecture Overview

### Deployment Model
- **Backend**: FastAPI + Uvicorn, PostgreSQL + pgvector, Redis async
- **Clients**: Python (PyPI), TypeScript (npm/JSR), Go (module), HTTP API
- **MCP Server**: FastMCP wrapper exposing memory tools for MCP agents (Claude Desktop, Cursor, etc.)

### Data Model (PostgreSQL)
Multi-tenant, composite primary key `(id, project_id)`, FK cascade:
- `Project` (read-only except profile_config)
- `User` (with additional_fields JSONB)
- `GeneralBlob` (polymorphic: blob_type, blob_data JSONB)
- `BufferZone` (idle/processing/done/failed states)
- `UserProfile` (content + attributes JSONB)
- `UserEvent` (event_data JSONB + pgvector embedding)
- `UserEventGist` (per-event summary + embedding)
- `UserStatus` (operational metadata)
- `Billing` / `ProjectBilling` (usage tracking)

### Processing Pipeline

**Buffer Flush Trigger**: Size >1024 tokens OR idle >1 hour OR manual

**LLM Pipeline** (3 calls):
1. **Extract**: Parse blobs → extract facts, events, topics
2. **Merge**: Merge new facts into existing profile (or YOLO: single-pass merge, 30% token savings)
3. **Organize**: Restructure profile into canonical form

**Search**: pgvector similarity on embeddings, event gist filtering, temporal queries

## Acceptance Criteria

### Functional
- [ ] All core APIs functional and tested
- [ ] Multi-tenant isolation verified
- [ ] Profile import/export working
- [ ] Event search with temporal filters
- [ ] Context assembly with prompt templates
- [ ] MCP tools exposed and callable from agents

### Performance
- [ ] 95th percentile latency <100ms (GET context, profile)
- [ ] Search <500-1000ms (embedding API dependent)
- [ ] LLM cost <= 3 calls per cycle
- [ ] LOCOMO benchmark >= mem0/langmem/zep

### Reliability
- [ ] Docker deployment reproducible
- [ ] Health checks all critical paths
- [ ] Error logging + telemetry in place
- [ ] Async buffer processing robust
- [ ] Redis/DB connection resilience tested

## Key Features & Differentiators

1. **Profile-First, Not RAG**: Structured attributes instead of embedding similarity search alone
2. **Time-Aware**: Event timeline with temporal queries; not just unordered chat history
3. **Batch Processing**: Buffer zone keeps LLM calls fixed and cost-efficient
4. **Customizable Schema**: Per-project profile configuration, prompt templates
5. **Multi-Language**: Python, TypeScript, Go SDKs; MCP for agent integration
6. **Production-Ready**: Telemetry, structured logging, health checks, Docker

## Development Roadmap (2025)

### Q3 2025 (Current)
- [x] Core profile & event APIs
- [x] Buffer + async flush
- [x] Event gist + fine-grained search
- [x] YOLO merge (cost reduction)
- [ ] Complete MCP server tools
- [ ] Benchmark validation

### Q4 2025 (Planned)
- Multi-profile schema in one project
- Social graph for user entities
- Typed profile slots (number/bool/date)
- Further cost optimization

## Technical Constraints

- **PostgreSQL 17+**: pgvector extension required
- **Redis 7.4+**: Async pool, password auth
- **Python 3.11+**: Backend and client SDKs
- **Node 18+**: TypeScript SDK (tsc → dist/)
- **Go 1.22.3+**: Go client SDK

## Dependencies

### External APIs
- **Embedding**: OpenAI, Jina, Ollama, LMStudio, OpenRouter (configurable)
- **LLM**: OpenAI, Volcengine/Doubao, OpenRouter (OpenAI-compatible)
- **Observability**: OpenTelemetry, structured logging (structlog)

### Key Libraries
- FastAPI, Pydantic, SQLAlchemy 2.0, pgvector
- httpx (Python client), zod (TS schema validation)
- FastMCP (MCP server)

## Metrics & Success KPIs

| KPI | Target | Current |
|-----|--------|---------|
| Search SOTA rank | Top 1 vs mem0/zep/langmem | On par (v0.0.37+) |
| Token cost per cycle | 3 calls | Achieved (v0.0.40) |
| Online latency (p95) | <100ms | SQL-driven, confirmed |
| Community adoption | Growing | Discord + GitHub stars |

## Open Questions & Risks

**Risks**:
- Event gist parity across Go SDK (TypeScript/Python verified)
- Scalability of profile merge at very large event counts (>10k)
- LLM API costs if embedding model changes

**Next Steps**:
- Complete MCP tool coverage
- Full benchmark suite (LOCOMO public results)
- Social graph prototype (Q4)
