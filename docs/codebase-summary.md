# Memobase Codebase Structure & Summary

## Repository Layout

```
memobase/
├── src/
│   ├── server/
│   │   ├── api/memobase_server/    # FastAPI backend service
│   │   ├── docker-compose.yml      # Local deployment (DB, Redis, API)
│   │   └── readme.md               # Server setup instructions
│   ├── client/
│   │   ├── memobase/               # Python SDK (PyPI: memobase)
│   │   ├── memobase-ts/            # TypeScript SDK (npm: @memobase/memobase)
│   │   └── memobase-go/            # Go SDK (github.com/memodb-io/memobase/src/client/memobase-go)
│   └── mcp/
│       └── src/                    # MCP server (FastMCP) for Claude Desktop, Cursor
├── docs/
│   ├── site/                       # Public Mintlify/MkDocs (API reference, guides)
│   ├── experiments/                # Benchmarks, evals (LOCOMO, 900-chat)
│   └── *.md                        # Internal engineering docs (this folder)
├── plans/                          # Development plans and reports
├── CONTRIBUTING.md                 # Contribution guidelines
├── Changelog.md                    # Version history and features
└── readme.md                       # Project overview (concise, <300 lines)
```

## Backend Service (src/server/api/)

### Technology Stack
- **Framework**: FastAPI + Uvicorn
- **Database**: PostgreSQL 17+ with pgvector extension
- **Cache**: Redis 7.4+
- **ORM**: SQLAlchemy 2.0
- **Schemas**: Pydantic
- **Migrations**: Alembic (schema from ORM models via build_init_sql.py)
- **Telemetry**: OpenTelemetry, structlog (JSON logging)
- **Token counting**: tiktoken

### Entry Point & Initialization
**File**: `src/server/api/api.py`
- FastAPI app builder
- Lifespan hook: Redis pool init, embedding/LLM sanity checks
- Custom OpenAPI: Bearer auth scheme, `/api/v1/healthcheck` exempt
- Single `APIRouter(prefix="/api/v1")`
- `AuthMiddleware`, OpenTelemetry instrumentation

### Architecture: Layered Design

```
HTTP Request
    ↓
api_layer/
  ├── handlers (thin HTTP wrappers)
  ├── middleware (auth, error handling)
  └── schemas (Pydantic models)
    ↓
controllers/
  ├── user, blob, buffer, profile, event, context, project, roleplay
  ├── modal/chat (extract→merge→organize pipeline)
  └── modal/roleplay (interest detection, topic prediction)
    ↓
models/
  ├── database.py (SQLAlchemy ORM)
  ├── blob.py (request models)
  ├── response.py (response envelopes)
  └── utils.py (Promise[D] Result type)
    ↓
connectors.py
  └── DB engine, Redis pool
```

### Core Modules

| Module | Purpose |
|--------|---------|
| `auth/token.py` | Validates `sk-<project_id>-<secret>` bearer tokens (Redis-cached) |
| `auth/admin_api.py` | Admin API for billing, status checks |
| `llms/` | OpenAI, Doubao, OpenRouter, embeddings (OpenAI/Jina/Ollama/LMStudio/OpenRouter), token counting |
| `prompts/` | LLM prompts (extraction, merge, organize, event, context, roleplay); English + Chinese variants |
| `env.py` | Centralized Config dataclass from config.yaml/env |
| `telemetry/` | Usage metrics, OTel setup, structured logging |

### Database Schema

Multi-tenant with composite PK `(id, project_id)`, FK cascade:

| Table | Purpose |
|-------|---------|
| `Project` | Project metadata (read-only via ORM listeners except profile_config) |
| `User` | User records + arbitrary JSONB metadata (additional_fields) |
| `GeneralBlob` | Raw ingested blobs (blob_type + blob_data JSONB) |
| `BufferZone` | Pending async processing (idle/processing/done/failed) |
| `UserProfile` | Hierarchical profile (content + attributes JSONB) |
| `UserEvent` | Time-stamped events (event_data JSONB + pgvector embedding) |
| `UserEventGist` | Per-event summary (content + embedding) |
| `UserStatus` | Operational flags and metadata |
| `Billing` / `ProjectBilling` | Usage tracking and billing |

### API Endpoints

All under `/api/v1` with bearer auth (except `/healthcheck`):

| Endpoint | Purpose |
|----------|---------|
| `GET /healthcheck` | Liveness probe |
| `GET /admin/status_check` | Detailed service status |
| `GET\|POST /project/profile_config` | Get/set customizable schema |
| `GET /project/billing` | Billing info |
| `GET /project/users` | List users |
| `GET /project/usage` | Usage analytics |
| `POST /users`, `GET\|PUT\|DELETE /users/{id}` | User CRUD |
| `GET /users/blobs/{id}/{blob_type}` | Retrieve blob |
| `POST /blobs/insert/{user_id}` | Insert blob |
| `GET\|DELETE /blobs/{user_id}/{blob_id}` | Blob retrieval/deletion |
| `GET\|POST /users/profile/{user_id}` | Profile CRUD |
| `PUT\|DELETE /users/profile/{profile_id}` | Update/delete profile |
| `POST /users/buffer/{user_id}/{buffer_type}` | Buffer flush |
| `GET .../capacity/...` | Buffer capacity check |
| `GET /users/event/{user_id}` | List events |
| `PUT\|DELETE .../event/{event_id}` | Event update/deletion |
| `GET .../event/search` | Search events (content/gist/tags) |
| `GET /users/context/{user_id}` | Assembled memory context for prompting |
| `POST /users/roleplay/proactive/{user_id}` | Roleplay features (experimental) |

### LLM Processing Pipeline

**Trigger**: Buffer size >1024 tokens OR idle >1 hour OR manual flush

**Three-call cycle**:
1. **Extract** (extract.py): Parse blobs → structured facts
2. **Merge** (merge.py / merge_yolo.py): Integrate into profile (YOLO = single-pass, 30% cost savings)
3. **Organize** (organize.py): Canonical profile restructuring

**Event gist**: Per-event fine-grained summary + embedding for detailed search

**Context assembly**: Format profile + events into prompt string (topic filters, token budgeting, customizable template)

### Testing
**Location**: `src/server/api/tests/`
- `test_api.py` — HTTP endpoint tests
- `test_controller.py` — Business logic tests
- `test_db.py` — Database operations
- `test_chat_modal.py` — LLM extraction pipeline
- `test_summary_modal.py` — Summary features
- `conftest.py` — Pytest fixtures, test config override

## Python Client SDK (src/client/memobase/)

### Packaging & Distribution
- **Package**: `memobase` (PyPI)
- **Min Python**: 3.11
- **Dependencies**: httpx, pydantic
- **Version**: Check `memobase/__init__.py`

### Public API

| Class | Purpose |
|-------|---------|
| `MemoBaseClient` | Sync HTTP client (project-level ops) |
| `AsyncMemoBaseClient` | Async client via httpx.AsyncClient |
| `User` / `AsyncUser` | User-level ops (blob, profile, event, context) |
| `ChatBlob` | Chat message data type |
| `SummaryBlob`, `DocBlob`, `CodeBlob`, `ImageBlob`, `TranscriptBlob` | Other blob types |

### Core Modules

| Module | Purpose |
|--------|---------|
| `core/entry.py` | Sync client, User class, project/user methods |
| `core/async_entry.py` | Async mirror |
| `core/blob.py` | Blob hierarchy + types |
| `core/user.py` | UserProfile, UserEvent data classes |
| `core/type.py` | BaseResponse envelope, ServerError |
| `network.py` | unpack_response() (HTTP + app-level errno handling) |
| `error.py` | ServerError exception |
| `utils.py` | string_to_uuid (deterministic) |
| `patch/openai.py` | OpenAI integration (auto-inject context, background flush) |

### Communication
- httpx Client/AsyncClient
- Base URL: `{project_url}/api/v1` (default: https://api.memobase.dev)
- Bearer auth: function arg or `MEMOBASE_API_KEY` env var
- Timeout: 60s

### Testing
**Location**: `src/client/tests/`
- Fixtures, sync/async tests against real backend
- Blob, user, profile, event tests

## TypeScript Client SDK (src/client/memobase-ts/)

### Packaging & Distribution
- **npm**: `@memobase/memobase`
- **JSR**: `@memobase/memobase`
- **Min Node**: 18
- **Dev**: Jest + ts-jest, ESLint + Prettier
- **Build**: tsc → dist/

### Public API

| Class | Purpose |
|-------|---------|
| `MemoBaseClient` | Sync HTTP client (native fetch) |
| `User` | User-level ops |
| `BlobType` enum | chat, summary, doc, code, image, transcript |
| Blob classes | ChatBlob, DocBlob, etc. |

### Core Modules

| Module | Purpose |
|--------|---------|
| `src/client.ts` | MemoBaseClient, project methods |
| `src/user.ts` | User class, blob/profile/event/context methods |
| `src/types.ts` | Zod schemas, TypeScript types |
| `src/network.ts` | unpackResponse() error handling |
| `src/error.ts` | ServerError exception |

### Testing
**Location**: `src/client/tests/`
- Jest tests (client, user)
- env.ts for test config

## Go Client SDK (src/client/memobase-go/)

### Packaging & Distribution
- **Module**: `github.com/memodb-io/memobase/src/client/memobase-go`
- **Go**: 1.22.3+
- **Dependencies**: google/uuid, stretchr/testify (test)

### Public API

| Type | Purpose |
|------|---------|
| `MemoBaseClient` | HTTP client (net/http with custom transport) |
| `User` | User-level ops |
| `BlobType` | Enumeration |
| Blob types | ChatBlob, DocBlob, CodeBlob, ImageBlob, TranscriptBlob |

### Core Modules

| Module | Purpose |
|--------|---------|
| `core/client.go` | MemoBaseClient initialization, project methods |
| `core/user.go` | User struct and methods (all blob types supported) |
| `core/types.go` | Response/domain types |
| `blob/blob.go` | BlobType, blob implementations, JSONTime parsing |
| `network/network.go` | BaseResponse, UnpackResponse |
| `error/error.go` | ServerError |
| `examples/main.go` | Standalone example |
| `*_test.go` | Unit tests (testify-based) |

### Configuration
- ProjectURL: Must be passed explicitly (no default like Python)
- API key: Function arg or `MEMOBASE_API_KEY` env
- Bearer auth transport layer
- 60s timeout

**Note**: Event gist/tag search present in Python but parity not confirmed in Go — flag as potential gap.

## MCP Server (src/mcp/src/)

### Purpose
FastMCP wrapper exposing Memobase memory tools for MCP agents (Claude Desktop, Cursor, Windsurf, n8n).

### Technology
- **Python 3.11+**
- **pyproject.toml** (uv-managed)
- **Dependencies**: httpx, mcp[cli]>=1.3.0, memobase (Python client)
- **Transport**: SSE (default) or stdio (env-configurable)

### Main Components

| File | Purpose |
|------|---------|
| `src/main.py` | FastMCP server ("memobase-mcp"), lifespan, tool definitions |
| `src/utils.py` | get_memobase_client() helper |

### Configuration (via .env)
- `TRANSPORT`: sse or stdio
- `HOST`, `PORT`: Server address
- `MEMOBASE_API_KEY`: Auth token
- `MEMOBASE_BASE_URL`: API endpoint

### Exposed Tools
- `save_memory(text)` — Insert ChatBlob + flush
- `get_user_profiles()` — Formatted profile list
- `search_memories(query, max_length=1000)` — Call user.context()

**Note**: No MCP resources/prompts defined yet; single DEFAULT_USER_ID (deterministic UUID).

## Development Workflow

### Branching
- `main` — production-ready
- `dev` — integration branch
- `feature/your-feature-name` — feature branches

### Commits
- Conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`
- Rebase before merge, clean history

### Testing
- Run tests before push (don't skip failures)
- Backend: pytest src/server/api/tests/
- Python client: pytest src/client/tests/
- TypeScript client: npm test
- Go client: go test ./...

### Deployment
- Docker Compose for local (see src/server/docker-compose.yml)
- Cloud: Memobase Cloud (free tier)
- Health checks: `/healthcheck`, `/admin/status_check`

## Key Design Patterns

### Error Handling
**Promise[D] Result type** (src/server/api/models/utils.py): Functions return Promise[T] instead of raising exceptions, enabling cleaner error propagation and logging.

### Telemetry
- Structured logging (structlog, JSON output)
- OpenTelemetry instrumentation (optional)
- Usage metrics and billing tracking

### Multi-tenancy
- Composite PK `(id, project_id)` on all user-facing tables
- Bearer token validation in AuthMiddleware
- Redis caching of auth tokens

### Async Processing
- BufferZone tracks state (idle/processing/done/failed)
- Background task for async flush
- Connection pooling (Redis, DB)

### Schema Customization
- Profile configuration stored per project
- Prompt templates configurable per context() call
- Blob types extensible

## Open Gaps & TODOs

- Event gist search parity in Go SDK (confirm or implement)
- MCP resources/prompts (future expansion)
- Social graph implementation (roadmap Q4)
- Typed profile slots prototype (roadmap Q4)
- Full LOCOMO benchmark suite public reporting
