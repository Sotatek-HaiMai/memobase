# Memobase Code Standards & Implementation Guidelines

## Principles

**YAGNI** — You Aren't Gonna Need It  
**KISS** — Keep It Simple, Stupid  
**DRY** — Don't Repeat Yourself

Prioritize functionality and readability over strict style enforcement. Ensure no syntax errors, code compiles/runs, and follows reasonable quality standards.

## File Organization & Naming

### Python (Backend & Client SDKs)

**Convention**: snake_case for file names with descriptive purpose

```python
src/server/api/
├── api.py                          # FastAPI app builder
├── auth/
│   ├── token.py                    # Bearer token validation
│   └── admin_api.py                # Admin endpoints
├── api_layer/
│   ├── blob.py                     # Blob endpoints
│   ├── buffer.py                   # Buffer/flush endpoints
│   ├── profile.py                  # Profile endpoints
│   ├── event.py                    # Event endpoints
│   ├── context.py                  # Context assembly
│   └── middleware.py               # Auth, error handling
├── controllers/
│   ├── user.py, blob.py, profile.py, event.py, etc.
│   └── modal/
│       ├── chat/                   # LLM extraction pipeline
│       └── roleplay/               # Roleplay features
├── models/
│   ├── database.py                 # SQLAlchemy ORM
│   ├── blob.py                     # Pydantic request models
│   ├── response.py                 # Response envelopes
│   └── utils.py                    # Promise[D] type
├── llms/
│   ├── openai_model_llm.py         # OpenAI integration (also serves OpenRouter, OpenAI-compatible)
│   ├── doubao_cache_llm.py         # Volcengine integration
│   └── embeddings/                 # Embedding providers (openai/jina/ollama/lmstudio/openrouter)
├── prompts/
│   ├── extract.py
│   ├── merge.py
│   ├── organize.py
│   └── ...                         # Templates + Chinese variants
└── telemetry/
    ├── usage.py                    # Metrics tracking
    └── otel_setup.py               # OpenTelemetry config
```

**File size limit**: Keep files under 200 lines for optimal context. Split large modules into focused components.

**Imports**: Group by standard library, third-party, local (in that order, separated by blank lines).

### TypeScript (Client SDK)

```typescript
src/client/memobase-ts/
├── src/
│   ├── index.ts                    # Barrel export
│   ├── client.ts                   # MemoBaseClient
│   ├── user.ts                     # User class
│   ├── types.ts                    # Zod schemas + types
│   ├── network.ts                  # HTTP utilities
│   └── error.ts                    # Error classes
├── tests/
│   ├── client.test.ts
│   ├── user.test.ts
│   └── env.ts                      # Test config
└── dist/                           # Compiled output (tsc)
```

**Convention**: camelCase for identifiers, PascalCase for classes/types

**Build**: `tsc` outputs to dist/, distributed via npm + JSR

### Go (Client SDK)

```go
src/client/memobase-go/
├── core/
│   ├── client.go                   # MemoBaseClient
│   ├── user.go                     # User struct
│   ├── types.go                    # Domain types
│   └── *_test.go                   # Tests
├── blob/
│   ├── blob.go                     # BlobType, implementations
│   └── blob_test.go
├── network/
│   ├── network.go                  # BaseResponse, HTTP utils
│   └── network_test.go
├── error/
│   ├── error.go                    # ServerError
│   └── error_test.go
├── examples/
│   └── main.go                     # Standalone example
├── go.mod                          # Module definition
└── go.sum
```

**Convention**: snake_case for file names, CamelCase for exported types/functions

**Testing**: `*_test.go` pairs, testify for assertions

## Code Style & Conventions

### Python Backend

**Style**: PEP 8, but not pedantic. Readability > strict compliance.

```python
# Type hints (required for new code)
def extract_facts(blob: Dict[str, Any], config: Config) -> Promise[List[Fact]]:
    """Extract facts from blob using LLM.
    
    Args:
        blob: Raw ingested data
        config: Extraction configuration
        
    Returns:
        Promise[List[Fact]]: Facts or error
    """
    # Implementation
    
# Error handling via Promise[T]
result = extract_facts(blob, config)
if result.is_error():
    return result  # Propagate error
facts = result.data

# Imports grouped
import json
from typing import Any, Dict, List

import httpx
from pydantic import BaseModel

from models.utils import Promise
```

**Naming**:
- Functions: `verb_noun` (e.g., `extract_facts`, `merge_profiles`)
- Classes: `PascalCase` (e.g., `UserProfile`, `BufferZone`)
- Constants: `UPPER_SNAKE_CASE` (e.g., `DEFAULT_TIMEOUT`)
- Private: `_leading_underscore`

**Comments**: For *why*, not *what*. Code should be self-documenting.

```python
# Why: Embedding models have different max lengths; exceed and degrade quality
if len(embedding_text) > MAX_TOKENS_FOR_EMBEDDING:
    embedding_text = embedding_text[:MAX_TOKENS_FOR_EMBEDDING]
```

### TypeScript Client

**Style**: ESLint + Prettier configured. Strict mode enforced.

```typescript
// Type safety required
interface User {
  id: string;
  projectId: string;
  metadata?: Record<string, unknown>;
}

class MemoBaseClient {
  private baseUrl: string;
  
  constructor(projectUrl: string, apiKey: string) {
    this.baseUrl = `${projectUrl}/api/v1`;
  }
  
  async getUser(userId: string): Promise<User> {
    // Implementation
  }
}

// Zod schemas for validation
const userSchema = z.object({
  id: z.string().uuid(),
  projectId: z.string(),
  metadata: z.record(z.unknown()).optional(),
});

export type User = z.infer<typeof userSchema>;
```

**Naming**:
- Classes: `PascalCase` (e.g., `MemoBaseClient`)
- Functions: `camelCase` (e.g., `getUser`, `formatContext`)
- Constants: `UPPER_SNAKE_CASE` (e.g., `DEFAULT_TIMEOUT`)
- Private: `_leading_underscore`

### Go Client

**Style**: Go conventions (gofmt, goimports).

```go
// Error handling: explicit returns
func (c *MemoBaseClient) GetUser(ctx context.Context, userID string) (*User, error) {
    if userID == "" {
        return nil, errors.New("userID cannot be empty")
    }
    // Implementation
    return user, nil
}

// Naming: CamelCase for exported, camelCase for private
type MemoBaseClient struct {
    baseURL   string
    apiKey    string
    httpClient *http.Client
}

// Constants
const defaultTimeout = 60 * time.Second

// Functions
func NewMemoBaseClient(projectURL, apiKey string) *MemoBaseClient {
    // ...
}
```

## Database Standards

### SQLAlchemy ORM (models/database.py)

**Composite PKs** for multi-tenancy:

```python
class User(Base):
    __tablename__ = "users"
    
    id: Mapped[UUID] = mapped_column(Uuid, primary_key=True)
    project_id: Mapped[UUID] = mapped_column(Uuid, primary_key=True)
    
    # Foreign keys
    project: Mapped[Project] = relationship(
        "Project",
        foreign_keys=[project_id],
        cascade="all, delete"
    )
    
    # Data
    name: Mapped[str] = mapped_column(String, nullable=True)
    additional_fields: Mapped[Dict[str, Any]] = mapped_column(JSON)
    
    # Indexes
    __table_args__ = (
        Index("idx_project_user", project_id, id),
    )
```

**Rules**:
- All user-facing tables: composite PK `(id, project_id)`
- Foreign keys: Cascade delete
- JSON columns: Use JSONB (PostgreSQL)
- Vectors: Use pgvector for embeddings
- Timestamps: Always include `created_at`, `updated_at`

## API Design

### Request/Response Envelopes

**Pydantic models** (models/blob.py, response.py):

```python
# Request
class InsertBlobRequest(BaseModel):
    blob_type: BlobType
    blob_data: Dict[str, Any]
    metadata: Optional[Dict[str, Any]] = None

# Response
class BaseResponse(BaseModel):
    data: Optional[T] = None
    errmsg: Optional[str] = None
    errno: int = 0  # 0 = success, >0 = error code

# Usage
return BaseResponse(data=blob, errno=0)
```

**HTTP Status Codes**:
- `200 OK`: Success
- `400 Bad Request`: Invalid input
- `401 Unauthorized`: Auth failure
- `404 Not Found`: Resource not found
- `500 Internal Server Error`: Server error

**Bearer auth**: `Authorization: Bearer sk-<project_id>-<secret>`

## Error Handling

### Promise[T] Pattern

**Replaces exceptions** for cleaner error propagation:

```python
class Promise(Generic[T]):
    def __init__(self, data: Optional[T] = None, error: Optional[str] = None):
        self.data = data
        self.error = error
    
    def is_error(self) -> bool:
        return self.error is not None

# Usage
def process():
    result = extract_facts(blob, config)
    if result.is_error():
        logger.error(f"Extraction failed: {result.error}")
        return result  # Propagate
    
    merged = merge_profiles(result.data, existing_profile)
    if merged.is_error():
        return merged
    
    return organize(merged.data)
```

### Logging

**Structured logging** (structlog + JSON):

```python
import structlog

logger = structlog.get_logger()

# Good: Structured context
logger.info(
    "profile_merged",
    user_id=user_id,
    profile_version=version,
    fact_count=len(facts),
    duration_ms=duration
)

# Not: Unstructured strings
logger.info(f"Merged {len(facts)} facts for user {user_id}")
```

**Log levels**:
- `debug`: Developer-facing detail
- `info`: Milestones, state changes
- `warning`: Recoverable issues
- `error`: Failures (still recovered)
- `critical`: Unrecoverable

## Testing Standards

### Backend (pytest)

**Location**: `src/server/api/tests/`

```python
import pytest
from httpx import TestClient

@pytest.fixture
def client(app):
    return TestClient(app)

def test_insert_blob_success(client, user_id):
    """Test blob insertion succeeds with valid data."""
    response = client.post(
        f"/api/v1/blobs/insert/{user_id}",
        json={"blob_type": "chat", "blob_data": {...}},
        headers={"Authorization": f"Bearer {MOCK_TOKEN}"}
    )
    assert response.status_code == 200
    assert response.json()["errno"] == 0

def test_insert_blob_missing_auth(client, user_id):
    """Test blob insertion fails without auth."""
    response = client.post(f"/api/v1/blobs/insert/{user_id}")
    assert response.status_code == 401
```

**Rules**:
- One test per behavior
- Descriptive names: `test_<scenario>_<expected>`
- Fixtures for setup (client, user_id, auth tokens)
- Mock external APIs (OpenAI, embeddings) in conftest.py
- Don't skip tests; fix underlying issues

### Python Client (pytest)

**Location**: `src/client/tests/`

```python
def test_user_insert_chat_blob(mocked_backend):
    """Test inserting chat blob returns blob ID."""
    client = MemoBaseClient("http://test", "test-key")
    user = client.get_user("user-123")
    blob_id = user.insert(ChatBlob(messages=[...]))
    assert isinstance(blob_id, str)
```

### TypeScript Client (Jest)

**Location**: `src/client/memobase-ts/tests/`

```typescript
describe('MemoBaseClient', () => {
  it('should fetch user by ID', async () => {
    const client = new MemoBaseClient('http://test', 'test-key');
    const user = await client.getUser('user-123');
    expect(user.id).toBe('user-123');
  });
  
  it('should throw on missing auth', async () => {
    const client = new MemoBaseClient('http://test', '');
    await expect(client.getUser('user-123')).rejects.toThrow();
  });
});
```

### Go Client (testify)

**Location**: `src/client/memobase-go/core/*_test.go`

```go
func TestGetUser(t *testing.T) {
    client := NewMemoBaseClient("http://test", "test-key")
    user, err := client.GetUser(context.Background(), "user-123")
    
    require.NoError(t, err)
    require.NotNil(t, user)
    assert.Equal(t, "user-123", user.ID)
}
```

## Security Standards

### Authentication
- Bearer tokens: `sk-<project_id>-<secret>` format
- Validated in AuthMiddleware
- Cached in Redis (with expiry)
- Never log full tokens

### Data Protection
- All user data is multi-tenant isolated (project_id foreign key)
- HTTPS in production
- Database connection pooling with TLS
- JSON fields use JSONB (indexed, secure)

### Error Messages
- Never expose stack traces to clients
- Log full errors internally; return generic messages to HTTP
- Sensitive info (tokens, keys) never in logs

## Performance Guidelines

### Database
- Composite indexes on (project_id, id) for multi-tenant queries
- pgvector indexes for similarity search
- Connection pooling (min=5, max=20 typical)
- Prepared statements to prevent SQL injection

### Caching
- Redis for auth token validation (TTL per token)
- In-process caches for config (invalidate on change)
- Embedding caches: per-content hash, configurable TTL

### Async Processing
- Buffer flush: async background task, retries on failure
- Long-running LLM calls: non-blocking, timeout enforcement
- Batch processing: consolidate multiple small requests

## Deployment Standards

### Docker
- Multi-stage builds for slim images
- Health checks configured
- Environment-driven config (no hardcoding)
- Volume mounts for data persistence (dev)

### Environment Variables
- `.env.example` checked in; `.env` ignored
- Centralized in Config dataclass (env.py)
- Never commit secrets

### Migrations
- Alembic setup; schema generated from ORM (build_init_sql.py)
- Migrations in version control, tested
- Rollback capability preserved

## Code Review Checklist

Before submitting PR:
- [ ] No syntax errors, code compiles/runs
- [ ] Tests pass locally (pytest, npm test, go test)
- [ ] New code has type hints (Python/TypeScript/Go)
- [ ] Docstrings for public functions/classes
- [ ] Error handling via Promise[T] or explicit returns
- [ ] No hardcoded secrets or credentials
- [ ] Follows naming conventions (snake_case/camelCase/CamelCase as appropriate)
- [ ] Comments explain *why*, not *what*
- [ ] File sizes reasonable (<200 lines preferred)
- [ ] Related files in same module not duplicating logic

## Git Workflow

### Branch Naming
```
feature/short-feature-name
fix/bug-description
docs/what-was-added
refactor/what-changed
test/what-tests-added
```

### Commit Messages (Conventional Commits)
```
feat: add event gist search
fix: handle missing profile gracefully
docs: document buffer flush API
refactor: extract LLM pipeline into service
test: add integration tests for context API
chore: update dependencies
```

### Before Push
1. `git status` — ensure no secrets or large files staged
2. `git diff` — review changes
3. Run linting/format: `black .`, `npm run lint`, `gofmt ./...`
4. Run tests: must pass before push
5. Rebase onto `dev` branch, clean history

## Documentation Standards

### Docstrings (Python)

```python
def extract_facts(blob: Dict[str, Any], config: Config) -> Promise[List[Fact]]:
    """Extract facts from blob using LLM.
    
    Uses the configured LLM to parse the blob and extract structured facts.
    Handles errors gracefully, returning a Promise with error details if parsing fails.
    
    Args:
        blob: Raw ingested data (blob_type, blob_data from request)
        config: Extraction configuration (model, prompt template, etc.)
        
    Returns:
        Promise[List[Fact]]: Success case: list of extracted facts.
                            Error case: error message describing parse failure.
    """
```

### Comments (Inline)

```python
# Why: Embedding models have different max token limits; exceeding degrades quality
if len(text) > MAX_TOKENS_FOR_EMBEDDING:
    text = text[:MAX_TOKENS_FOR_EMBEDDING]
```

### README Files
- Clear setup/installation instructions
- Example usage
- Link to public docs for details
- Troubleshooting common issues

## Continuous Improvement

- Measure test coverage and maintain >80%
- Use linting tools (pylint, ESLint, golangci-lint) to catch issues
- Peer review all PRs before merge
- Collect performance metrics (latency, error rates)
- Refactor incrementally; don't wait for perfect code
