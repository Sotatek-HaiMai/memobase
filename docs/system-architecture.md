# Memobase System Architecture

## High-Level Overview

Memobase is a distributed system for managing long-term user memory in LLM applications. It separates the system into three layers:

1. **API Layer** (FastAPI backend, HTTP/MCP interfaces)
2. **Processing Layer** (LLM extraction pipeline, buffer management)
3. **Storage Layer** (PostgreSQL + pgvector, Redis)

User-facing SDKs (Python, TypeScript, Go) communicate with the API layer; the API layer orchestrates processing and storage.

## System Components

```
┌─────────────────────────────────────────────────────────────────┐
│                      Client Applications                         │
│  (LLM Apps, Agents, Chatbots, Virtual Companions)               │
└──┬──────────────────────────────────────────────────────────────┘
   │
   ├──────────────────────────┬────────────────────┬──────────────┐
   │                          │                    │              │
   ▼                          ▼                    ▼              ▼
┌──────────────┐      ┌──────────────┐    ┌──────────────┐  ┌─────────────┐
│ Python SDK   │      │ TypeScript   │    │    Go SDK    │  │ HTTP API    │
│ (memobase)   │      │ SDK (@memo)  │    │  (memobase)  │  │ /api/v1     │
└──────┬───────┘      └──────┬───────┘    └──────┬───────┘  └──────┬──────┘
       │                     │                    │                │
       └─────────────────────┴────────────────────┴────────────────┘
                            │
                   ┌────────▼────────┐
                   │   HTTP/Bearer   │
                   │   Auth Layer    │
                   └────────┬────────┘
                            │
         ┌──────────────────▼──────────────────┐
         │      FastAPI Application            │
         │  (api.py, lifespan hook, routes)   │
         └──────────────────┬──────────────────┘
                            │
         ┌──────────────────▼──────────────────┐
         │        API Layer                    │
         │  ├─ blob endpoints                 │
         │  ├─ user endpoints                 │
         │  ├─ profile endpoints              │
         │  ├─ event endpoints                │
         │  ├─ context assembly               │
         │  ├─ buffer flush                   │
         │  └─ roleplay (experimental)        │
         └──────────────────┬──────────────────┘
                            │
         ┌──────────────────▼──────────────────┐
         │   Controllers (Business Logic)      │
         │  ├─ user_controller                │
         │  ├─ blob_controller                │
         │  ├─ profile_controller             │
         │  ├─ event_controller               │
         │  ├─ context_controller             │
         │  ├─ buffer_controller              │
         │  └─ modal/                         │
         │     ├─ chat (extract→merge→org)   │
         │     └─ roleplay                    │
         └──────────────────┬──────────────────┘
                            │
    ┌───────────────────────┼───────────────────────┐
    │                       │                       │
    ▼                       ▼                       ▼
┌──────────────┐      ┌──────────────┐      ┌─────────────┐
│  PostgreSQL  │      │    Redis     │      │   LLM/      │
│  (pgvector)  │      │ (async pool) │      │ Embeddings  │
│              │      │              │      │   APIs      │
│ ├─ User      │      │ Auth tokens  │      │             │
│ ├─ Event     │      │ config cache │      │ ├─ OpenAI   │
│ ├─ Profile   │      │              │      │ ├─ Jina     │
│ ├─ Blob      │      │              │      │ ├─ Ollama   │
│ └─ Buffer    │      │              │      │ └─ LMStudio │
└──────────────┘      └──────────────┘      └─────────────┘
```

## Detailed Layer Architecture

### API Layer (api_layer/)

**Responsibility**: HTTP request handling, input validation, response formatting

**Components**:
- **Handlers**: Thin HTTP wrappers that parse requests, call controllers, format responses
- **Middleware**: AuthMiddleware (token validation), error handling, CORS
- **Schemas**: Pydantic models for request/response validation

**Pattern**: Handler → Controller → Model → Storage

### Controller Layer (controllers/)

**Responsibility**: Business logic orchestration, validation, state transitions

**Key Controllers**:

| Controller | Responsibility |
|------------|-----------------|
| `user_controller` | User CRUD, metadata management |
| `blob_controller` | Blob ingestion, type validation |
| `profile_controller` | Profile CRUD, import/export |
| `event_controller` | Event CRUD, search, gist management |
| `context_controller` | Memory assembly for prompting |
| `buffer_controller` | Buffer lifecycle, flush orchestration |
| `project_controller` | Project config, billing, usage |
| `chat_modal` | LLM extraction pipeline (extract→merge→organize) |
| `roleplay_modal` | Interest detection, topic prediction |

**Pattern**: Each controller encapsulates a logical domain; controllers call models and connectors.

### Model Layer (models/)

**Responsibility**: Data representation, database ORM, utilities

**Components**:

| Module | Purpose |
|--------|---------|
| `database.py` | SQLAlchemy ORM (User, Event, Profile, Blob, etc.) |
| `blob.py` | Pydantic request models (InsertBlobRequest, etc.) |
| `response.py` | BaseResponse envelope, response models |
| `utils.py` | Promise[T] error handling, helpers |

### Connector Layer (connectors.py)

**Responsibility**: Database engine, Redis pool initialization, connection management

**Provides**:
- SQLAlchemy engine (connection pooling)
- Redis async pool
- Config object

## Data Model & Relationships

### Core Tables (Multi-Tenant)

```sql
-- Projects (meta-information)
CREATE TABLE projects (
    id UUID PRIMARY KEY,
    name VARCHAR,
    profile_config JSONB,
    created_at TIMESTAMP
);

-- Users (per-project)
CREATE TABLE users (
    id UUID,
    project_id UUID,
    name VARCHAR,
    additional_fields JSONB,
    created_at TIMESTAMP,
    PRIMARY KEY (id, project_id),
    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE
);

-- Blobs (raw ingested data)
CREATE TABLE general_blobs (
    id UUID,
    project_id UUID,
    user_id UUID,
    blob_type VARCHAR,
    blob_data JSONB,
    created_at TIMESTAMP,
    PRIMARY KEY (id, project_id),
    FOREIGN KEY (project_id) REFERENCES projects(id),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Buffer Zone (pending async processing)
CREATE TABLE buffer_zones (
    id UUID,
    project_id UUID,
    user_id UUID,
    buffer_type VARCHAR,
    status VARCHAR (idle|processing|done|failed),
    token_count INT,
    created_at TIMESTAMP,
    PRIMARY KEY (id, project_id),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- User Profiles (hierarchical)
CREATE TABLE user_profiles (
    id UUID,
    project_id UUID,
    user_id UUID,
    content VARCHAR,
    attributes JSONB,
    created_at TIMESTAMP,
    PRIMARY KEY (id, project_id),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Events (timeline with embeddings)
CREATE TABLE user_events (
    id UUID,
    project_id UUID,
    user_id UUID,
    event_data JSONB,
    embedding vector(1536),  -- pgvector
    created_at TIMESTAMP,
    PRIMARY KEY (id, project_id),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Event Gists (fine-grained summaries)
CREATE TABLE user_event_gists (
    id UUID,
    project_id UUID,
    event_id UUID,
    gist_text VARCHAR,
    embedding vector(1536),
    created_at TIMESTAMP,
    PRIMARY KEY (id, project_id),
    FOREIGN KEY (event_id) REFERENCES user_events(id)
);

-- Status tracking
CREATE TABLE user_statuses (
    id UUID,
    project_id UUID,
    user_id UUID,
    status_type VARCHAR,
    value VARCHAR,
    PRIMARY KEY (id, project_id),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Billing & Usage
CREATE TABLE billing (
    id UUID,
    project_id UUID,
    amount DECIMAL,
    timestamp TIMESTAMP,
    PRIMARY KEY (id, project_id),
    FOREIGN KEY (project_id) REFERENCES projects(id)
);
```

## Processing Pipeline

### Buffer Flush Lifecycle

```
Blob Inserted
    │
    └─→ GeneralBlob row created
            │
            └─→ BufferZone entry (idle)
                    │
                    └─→ Accumulated until:
                        ├─ token_count > 1024 OR
                        ├─ idle time > 1 hour OR
                        └─ manual flush() call
                            │
                            ▼
                        [EXTRACTION PHASE]
                        Extract → structured facts
                            │
                            ▼
                        [MERGE PHASE]
                        Merge facts into UserProfile
                        (YOLO: single-pass, 30% token savings)
                            │
                            ▼
                        [ORGANIZE PHASE]
                        Restructure profile to canonical form
                            │
                            ▼
                        BufferZone status → done
                        GeneralBlobs deleted (unless config.persist_blobs)
```

### LLM Integration Points

| Call | Purpose | Input | Output | Cached |
|------|---------|-------|--------|--------|
| Extract | Parse blob → facts | blob_data | List[Fact] | No |
| Merge | Integrate facts into profile | facts + existing_profile | Updated profile | No |
| Organize | Restructure profile | profile | Canonical form | No |
| Event Gist | Summarize event | event_data | gist_text | No |
| Search embedding | Vectorize query | query_text | embedding (1536D) | Per-text hash |
| Preference embedding | Vectorize for similarity | text | embedding (1536D) | Per-text hash |

**Cost Optimization** (v0.0.40):
- **Before**: 3-10 calls per cycle → **After**: Fixed 3 calls
- **YOLO merge**: Single-pass merge replaces iterative refinement (30% token savings)

### Context Assembly (GET /users/context/{user_id})

```
Request: max_token_size, prefer_topics, topic_limits, profile_ratio

    │
    ├─→ Fetch UserProfile (latest version)
    │
    ├─→ Fetch UserEvents (with similarity threshold)
    │       └─→ Rank by recency + relevance
    │
    ├─→ Format output:
    │   ├─ Memory introduction
    │   ├─ User Background (profile)
    │   ├─ Latest Events (timeline)
    │   └─ Optional: custom prompt template
    │
    └─→ Enforce token budget (truncate if needed)

Response: assembled_memory_string (suitable for LLM prompt)
```

## Authentication & Authorization

### Bearer Token Flow

```
Client
    │
    ├─→ Authorization: Bearer sk-<project_id>-<secret>
    │
    ▼
AuthMiddleware
    │
    ├─→ Extract token from header
    │
    ├─→ Check Redis cache (TTL-based)
    │       ├─ Hit: return cached project_id
    │       └─ Miss: validate token (lookup in DB)
    │
    ├─→ On validation:
    │       ├─ Success: cache in Redis, allow request
    │       └─ Failure: return 401 Unauthorized
    │
    └─→ Attach project_id to request context

Controller
    │
    └─→ Uses project_id to filter all queries (WHERE project_id = ?)
        (Ensures multi-tenant isolation)
```

### Multi-Tenancy Enforcement

- **Composite PKs**: Every user-facing table has `(id, project_id)` PK
- **Foreign Keys**: All FKs reference both `id` and `project_id`
- **Query filters**: Controllers always include `WHERE project_id = ?`
- **Connection isolation**: Redis caches per-project tokens separately

## Async Processing & Resilience

### Buffer Flush Retry Logic

```
Flush trigger
    │
    ▼
Update BufferZone status → processing

    │
    ├─→ Try extract()
    │   ├─ Success: proceed to merge
    │   └─ Failure (retryable): exponential backoff, retry up to 3 times
    │       └─ Failure (max retries): status → failed, log error
    │
    ├─→ Try merge()
    │   ├─ Success: proceed to organize
    │   └─ Failure: same retry logic
    │
    ├─→ Try organize()
    │   ├─ Success: status → done
    │   └─ Failure: status → failed
    │
    └─→ On success: Delete GeneralBlobs, update timestamps
        On failure: status → failed (manual intervention or retry)
```

### Redis Connection Pooling

- Min connections: 5
- Max connections: 20
- Timeout: 30s per operation
- Auto-retry on transient failures

### PostgreSQL Connection Pooling

- Min connections: 5
- Max connections: 20
- Timeout: 60s per query
- Connection recycling every 3600s

## Deployment Architecture

### Docker Compose (Development/Local)

```yaml
version: '3.8'
services:
  memobase-server-db:
    image: pgvector/pgvector:pg17
    environment:
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  memobase-server-redis:
    image: redis:7.4
    command: redis-server --requirepass password
    ports:
      - "6379:6379"

  memobase-server-api:
    build: .
    environment:
      DATABASE_URL: postgresql://postgres:password@memobase-server-db:5432/memobase
      REDIS_URL: redis://:password@memobase-server-redis:6379
      API_EXPORT_PORT: 8000
    ports:
      - "8000:8000"
    depends_on:
      - memobase-server-db
      - memobase-server-redis
```

### Production Deployment

- **API**: Stateless FastAPI servers (horizontal scaling)
- **Database**: Managed PostgreSQL (cloud provider, backup/HA)
- **Cache**: Managed Redis (cloud provider, HA)
- **Monitoring**: OpenTelemetry + structured logging
- **Health checks**: `/healthcheck`, `/admin/status_check`

## Client Architecture

### Python SDK

```
Application
    │
    ├─→ MemoBaseClient (project-level ops)
    │   ├─ ping(), get_config(), update_config()
    │   ├─ add_user(), get_user(), get_all_users()
    │   └─ get_usage(), get_daily_usage()
    │
    └─→ User (user-level ops)
        ├─ insert(blob), get(blob_id), delete(blob_id)
        ├─ flush(sync=False)
        ├─ profile(), import_profile()
        ├─ event(), search_events()
        ├─ context(max_token_size, prefer_topics)
        └─ buffer()

httpx.Client (HTTP layer)
    │
    └─→ FastAPI Backend (/api/v1)
```

### TypeScript SDK

```
Application
    │
    ├─→ MemoBaseClient
    │   └─ project methods (getUser, addUser, etc.)
    │
    └─→ User
        └─ user methods (insert, profile, event, context, etc.)

fetch() (HTTP layer, native)
    │
    └─→ FastAPI Backend (/api/v1)
```

### Go SDK

```
Application
    │
    ├─→ MemoBaseClient
    │   └─ project methods (GetUser, AddUser, etc.)
    │
    └─→ User
        └─ user methods (Insert, Profile, Event, Context, etc.)

net/http.Client (HTTP layer)
    │
    └─→ FastAPI Backend (/api/v1)
```

## Integration with External Services

### LLM Providers

| Provider | Use Case | Config |
|----------|----------|--------|
| OpenAI | Chat, extraction, embeddings | API key, model name |
| Volcengine/Doubao | Chat extraction (alternative) | API key, model name |
| OpenRouter | Chat + embeddings (multi-model router, OpenAI-compatible) | API key, model slug (`{provider}/{model}`) |
| Jina | Embeddings (alternative) | API key |
| Ollama | Local inference (dev/test) | Base URL |
| LMStudio | Local inference (dev/test) | Base URL |

**Failover**: If primary LLM fails, fallback to secondary (if configured).

### Embedding Model Selection

- **Default**: OpenAI ada-002 (1536D)
- **Alternatives**: Jina, Ollama (configurable dimension)
- **Per-project config**: Profile configuration can specify embedding model

## Observability & Monitoring

### Telemetry Collection

**Structured logging** (structlog):
```json
{
  "event": "profile_merged",
  "user_id": "uuid",
  "profile_version": 5,
  "fact_count": 42,
  "duration_ms": 1250,
  "timestamp": "2025-07-23T10:15:30Z"
}
```

**OpenTelemetry instrumentation**:
- Trace extraction/merge/organize calls
- Track database query performance
- Monitor Redis latency

**Health checks**:
- `/healthcheck` — basic liveness (200 OK)
- `/admin/status_check` — deep health (DB, Redis, LLM connectivity)

### Metrics

| Metric | Purpose |
|--------|---------|
| `buffer_flush_duration_ms` | Time to complete full LLM cycle |
| `extract_tokens_used` | Tokens consumed in extract phase |
| `merge_tokens_used` | Tokens consumed in merge phase |
| `organize_tokens_used` | Tokens consumed in organize phase |
| `embedding_latency_ms` | Time for embedding API calls |
| `context_assembly_latency_ms` | Time to assemble memory for prompting |
| `db_query_latency_ms` | Database query performance |
| `redis_operation_latency_ms` | Cache performance |

## Security Considerations

- **Data encryption**: HTTPS in production, TLS for DB/Redis
- **Token validation**: All requests validated via AuthMiddleware
- **SQL injection prevention**: Prepared statements (SQLAlchemy)
- **Rate limiting**: Per-project quotas (future: implement)
- **Audit logging**: All state-changing operations logged
- **Data residency**: Deployable in region-specific cloud (future)

## Scalability & Performance

### Horizontal Scaling

**Stateless API servers**: Add/remove FastAPI instances behind load balancer; all state in DB/Redis.

**Database sharding** (future): Partition by project_id or user_id for very large deployments.

**Search optimization** (current):
- pgvector indexes on embeddings
- Event gist filtering (reduces vector search result set)
- Temporal indexes on created_at

### Performance Targets

| Operation | Target | Mechanism |
|-----------|--------|-----------|
| GET user | <20ms | Direct lookup + cache |
| GET profile | <50ms | SQL join + in-process cache |
| GET context | <100ms | Profile fetch + event query + formatting |
| Search events | 500-1000ms | Embedding similarity + gist filter |
| Flush cycle | <3s | Parallel LLM calls (future: enable) |

## Evolution & Future Architecture

### Planned Enhancements (Q4 2025+)

**Social graph**: Link user entities (e.g., "user is related to X"), shared memory modeling

**Multi-profile schema**: Allow multiple profiles per user (e.g., work profile, personal profile)

**Typed profile slots**: Number, boolean, date fields with validation

**Streaming context**: Real-time memory updates (WebSocket)

**Regional deployments**: Data residency + geo-replication
