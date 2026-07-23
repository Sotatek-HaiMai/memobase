# Memobase Deployment Guide

## Overview

This guide covers deploying Memobase in local development, staging, and production environments. Choose the deployment model that fits your needs:

- **Local Docker Compose**: Development & testing
- **Managed Cloud (Memobase Cloud)**: Quickest setup (free tier available)
- **Self-Hosted Docker**: Custom deployments (VPS, Kubernetes)
- **Cloud-Managed Databases**: Scale on cloud providers (AWS RDS, Google Cloud SQL)

## Prerequisites

### Required

- Docker & Docker Compose (for containerized deployments)
- Python 3.11+ (for development/MCP server)
- PostgreSQL 17+ with pgvector extension
- Redis 7.4+
- Python client SDK (`pip install memobase`)

### Optional

- Kubernetes (for enterprise scaling)
- Terraform (for infrastructure-as-code)
- Cloud CLI tools (AWS CLI, gcloud, etc.)

## Quick Start: Local Docker Compose

### 1. Clone Repository

```bash
git clone https://github.com/memodb-io/memobase.git
cd memobase
```

### 2. Start Services

```bash
cd src/server
docker-compose up -d
```

**Expected output**:
```
Creating memobase-server-db ... done
Creating memobase-server-redis ... done
Creating memobase-server-api ... done
```

### 3. Verify Health

```bash
curl http://localhost:8000/api/v1/healthcheck
# Response: {"data": "ok", "errmsg": null, "errno": 0}
```

### 4. Test Connection

```python
from memobase import MemoBaseClient, ChatBlob

client = MemoBaseClient(
    project_url="http://localhost:8000",
    api_key="secret"  # Local default
)
assert client.ping()
print("Connected!")
```

**Database**: PostgreSQL at `localhost:5432` (user: postgres, password: password)  
**Cache**: Redis at `localhost:6379` (password: password)

## Configuration

### Environment Variables

**Backend** (src/server/api/.env or docker-compose.yml):

| Variable | Default | Purpose |
|----------|---------|---------|
| `DATABASE_URL` | `postgresql://postgres:password@localhost:5432/memobase` | PostgreSQL connection |
| `REDIS_URL` | `redis://:password@localhost:6379/0` | Redis connection |
| `API_EXPORT_PORT` | `8000` | FastAPI server port |
| `OPENAI_API_KEY` | (required) | LLM extraction, embeddings |
| `OPENAI_MODEL` | `gpt-4-turbo` | Primary LLM model |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | Embedding model |
| `EMBEDDING_DIMENSION` | `1536` | Embedding vector size |
| `LOG_LEVEL` | `INFO` | Logging level (DEBUG/INFO/WARNING/ERROR) |
| `CORS_ORIGINS` | `http://localhost:3000` | CORS allowed origins |
| `TELEMETRY_ENABLED` | `true` | Enable OpenTelemetry |

**Python Client** (.env):

| Variable | Purpose |
|----------|---------|
| `MEMOBASE_API_KEY` | Bearer token for authentication |
| `MEMOBASE_BASE_URL` | API base URL (default: https://api.memobase.dev) |

### Configuration File (config.yaml)

Alternative to environment variables:

```yaml
database:
  url: postgresql://postgres:password@localhost:5432/memobase
  pool_size: 20
  max_overflow: 10

redis:
  url: redis://:password@localhost:6379
  pool_size: 20

api:
  port: 8000
  cors_origins:
    - "http://localhost:3000"
    - "https://app.memobase.io"

llm:
  openai:
    api_key: "${OPENAI_API_KEY}"
    model: "gpt-4-turbo"
    embedding_model: "text-embedding-3-small"
    embedding_dimension: 1536
    
  fallback:
    - provider: "doubao"
      api_key: "${DOUBAO_API_KEY}"
      model: "claude-3-opus"

embeddings:
  primary: "openai"
  fallback: ["jina", "ollama", "openrouter", "lmstudio"]
  
telemetry:
  enabled: true
  jaeger_endpoint: "http://localhost:14250"
```

## Deployment: Docker Compose (Production-Like)

### Setup

```bash
cd src/server
# Copy example env file
cp .env.example .env

# Edit for production
nano .env
# Set: OPENAI_API_KEY, DATABASE_URL (managed), REDIS_URL (managed)
```

### Updated docker-compose.yml (with persistent volumes)

```yaml
version: '3.8'

services:
  memobase-server-db:
    image: pgvector/pgvector:pg17
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD:-password}
      POSTGRES_DB: memobase
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  memobase-server-redis:
    image: redis:7.4
    command: redis-server --requirepass ${REDIS_PASSWORD:-password}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  memobase-server-api:
    build: .
    environment:
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      API_EXPORT_PORT: 8000
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      LOG_LEVEL: ${LOG_LEVEL:-INFO}
    ports:
      - "8000:8000"
    depends_on:
      memobase-server-db:
        condition: service_healthy
      memobase-server-redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/v1/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  postgres_data:
  redis_data:
```

### Start

```bash
docker-compose up -d
docker-compose logs -f memobase-server-api
```

## Deployment: Kubernetes

### Prerequisites

- kubectl configured
- Helm package manager (optional)

### Basic Deployment

**Dockerfile** (src/server/Dockerfile):

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
CMD ["uvicorn", "api.api:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Kubernetes manifest** (k8s/deployment.yaml):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: memobase-config
data:
  LOG_LEVEL: "INFO"
  API_EXPORT_PORT: "8000"

---
apiVersion: v1
kind: Secret
metadata:
  name: memobase-secrets
type: Opaque
stringData:
  openai_api_key: "sk-..."
  database_url: "postgresql://..."
  redis_url: "redis://..."

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memobase-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: memobase-api
  template:
    metadata:
      labels:
        app: memobase-api
    spec:
      containers:
      - name: api
        image: memodb-io/memobase-api:0.0.40
        ports:
        - containerPort: 8000
        env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: memobase-config
              key: LOG_LEVEL
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: memobase-secrets
              key: openai_api_key
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: memobase-secrets
              key: database_url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: memobase-secrets
              key: redis_url
        livenessProbe:
          httpGet:
            path: /api/v1/healthcheck
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/v1/healthcheck
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"

---
apiVersion: v1
kind: Service
metadata:
  name: memobase-api
spec:
  selector:
    app: memobase-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

### Deploy

```bash
kubectl apply -f k8s/deployment.yaml
kubectl get pods -l app=memobase-api
kubectl port-forward svc/memobase-api 8000:80
```

## Deployment: Cloud Providers

### AWS (ECS + RDS + ElastiCache)

**Steps**:

1. **RDS PostgreSQL**:
   - Engine: PostgreSQL 17
   - Add pgvector extension: `CREATE EXTENSION pgvector;`
   - Security group: Allow port 5432 from ECS

2. **ElastiCache Redis**:
   - Engine: Redis 7.4
   - Auth token enabled
   - Security group: Allow port 6379 from ECS

3. **ECS Task Definition**:
   ```json
   {
     "family": "memobase-api",
     "containerDefinitions": [
       {
         "name": "api",
         "image": "YOUR_ECR_REPO/memobase-api:0.0.40",
         "portMappings": [{"containerPort": 8000}],
         "environment": [
           {"name": "API_EXPORT_PORT", "value": "8000"},
           {"name": "LOG_LEVEL", "value": "INFO"}
         ],
         "secrets": [
           {"name": "DATABASE_URL", "valueFrom": "arn:aws:secretsmanager:..."},
           {"name": "REDIS_URL", "valueFrom": "arn:aws:secretsmanager:..."},
           {"name": "OPENAI_API_KEY", "valueFrom": "arn:aws:secretsmanager:..."}
         ]
       }
     ]
   }
   ```

4. **ECS Service**: 
   - Desired count: 3
   - Load balancer: Application Load Balancer (ALB)
   - Health check: `/api/v1/healthcheck`

### Google Cloud (Cloud Run + Cloud SQL + Memorystore)

**Steps**:

1. **Cloud SQL PostgreSQL**:
   - Version: 17
   - Install pgvector extension
   - Private IP (VPC)

2. **Memorystore Redis**:
   - Version: 7.4
   - Private IP (same VPC)

3. **Cloud Run Deployment**:
   ```bash
   gcloud run deploy memobase-api \
     --image gcr.io/PROJECT/memobase-api:0.0.40 \
     --platform managed \
     --region us-central1 \
     --memory 1Gi \
     --cpu 2 \
     --min-instances 1 \
     --max-instances 10 \
     --set-env-vars "LOG_LEVEL=INFO,API_EXPORT_PORT=8000" \
     --set-secrets "DATABASE_URL=db-url:latest,REDIS_URL=redis-url:latest,OPENAI_API_KEY=openai-key:latest" \
     --vpc-connector memobase-vpc
   ```

### Azure (App Service + Azure Database + Azure Cache)

Similar approach with Azure-specific services (App Service, Azure Database for PostgreSQL, Azure Cache for Redis).

## Health Checks & Monitoring

### Health Check Endpoints

```bash
# Basic liveness
curl http://localhost:8000/api/v1/healthcheck

# Detailed status (requires auth)
curl -H "Authorization: Bearer sk-project-secret" \
  http://localhost:8000/api/v1/admin/status_check
```

**Response** (200 OK):
```json
{
  "data": {
    "status": "healthy",
    "database": "ok",
    "redis": "ok",
    "llm": "ok"
  },
  "errno": 0
}
```

### Logging & Observability

**Structured logs** (JSON):
```json
{"timestamp": "2025-07-23T10:15:30Z", "level": "INFO", "event": "buffer_flushed", "user_id": "...", "duration_ms": 1250}
```

**OpenTelemetry integration**:
```bash
# Export traces to Jaeger (optional)
export OTEL_EXPORTER_JAEGER_ENDPOINT=http://localhost:14250
export OTEL_TRACES_EXPORTER=jaeger
```

### Metrics & Alerts

**Key metrics to monitor**:
- `buffer_flush_duration_ms` (p50, p95, p99)
- `context_latency_ms` (p95 < 100ms target)
- `http_request_count` (by endpoint, status)
- `database_connection_pool_utilization` (should stay <80%)
- `redis_operation_latency_ms`
- `llm_api_latency_ms` (extract, merge, organize)

**Recommended alerts**:
- Health check failures (2+ consecutive)
- Database connection pool exhaustion
- Buffer flush failures
- LLM API timeouts
- High error rates (>1%)

### Using Prometheus + Grafana (Optional)

**Prometheus config** (prometheus.yml):
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'memobase-api'
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/api/v1/metrics'
```

## Troubleshooting

### Connection Issues

**Problem**: `Connection refused to PostgreSQL`

```bash
# Check database is running
docker ps | grep postgres
# Verify connection
psql -U postgres -h localhost -d memobase
```

**Problem**: `Redis timeout`

```bash
# Check Redis
docker exec memobase-server-redis redis-cli ping
# Verify connection string
redis-cli -h localhost -p 6379 -a password ping
```

### Performance Issues

**Problem**: `Buffer flush takes >5s`

- Check LLM API latency: `OPENAI_API_KEY` valid?
- Monitor database query times: Enable query logging
- Check Redis connection pool: `REDIS_URL` correct?

**Problem**: `Context assembly latency >200ms`

- Reduce profile size: Trim old events
- Optimize embedding search: Use smaller embedding dimension
- Add database indexes: Check explain plan

### Authentication Errors

**Problem**: `401 Unauthorized`

```bash
# Verify token format
# Token should be: sk-<project_id>-<secret>

# Check Redis cache
docker exec memobase-server-redis redis-cli -a password keys "*"
```

## Backup & Recovery

### PostgreSQL Backup

```bash
# Full backup
pg_dump -h localhost -U postgres memobase > backup.sql

# Restore
psql -h localhost -U postgres memobase < backup.sql

# Or with Docker
docker exec memobase-server-db pg_dump -U postgres memobase > backup.sql
```

### Redis Backup

```bash
# RDB snapshot (automatic)
docker exec memobase-server-redis redis-cli BGSAVE

# Copy snapshot
docker cp memobase-server-redis:/data/dump.rdb ./
```

### Automated Backups (Production)

- **AWS**: RDS automated snapshots + S3 exports
- **GCP**: Cloud SQL automatic backups
- **Azure**: Built-in Azure Backup

## Scaling

### Horizontal Scaling (Multiple API Instances)

Memobase API is stateless; add instances behind a load balancer.

**Docker Compose example** (scale):
```bash
docker-compose up -d --scale memobase-server-api=3
```

**Kubernetes**:
```bash
kubectl scale deployment memobase-api --replicas=5
```

### Vertical Scaling (Larger Instances)

Increase memory/CPU for high-throughput scenarios.

**Recommended for**:
- >10K concurrent users
- Complex profiles (>1MB each)
- High event volumes (>1M events/user)

### Database Sharding (Future)

For very large deployments, partition by project_id or user_id.

## Managed Cloud: Memobase Cloud

For simplicity, use the official hosted service:

**Free tier**:
- Up to 5 projects
- Up to 1M total users
- Up to 100GB storage
- No credit card required

**Sign up**: [Memobase Cloud](https://www.memobase.io/en/login)

**Benefits**:
- No infrastructure management
- Automatic backups
- Built-in monitoring
- 99.9% uptime SLA (paid)

## Security Best Practices

### Network

- Use TLS/HTTPS in production (e.g., Let's Encrypt)
- Run database/Redis on private networks (no public IP)
- Use VPCs or security groups to restrict access
- Enable database encryption at rest

### Credentials

- Store API keys in secrets manager (AWS Secrets, GCP Secret Manager, Azure Key Vault)
- Rotate credentials regularly
- Never commit `.env` files
- Use service accounts for inter-service auth

### Monitoring

- Enable audit logging for admin operations
- Monitor for suspicious token usage
- Set up alerts for failed auth attempts
- Regularly review access logs

## Upgrade Path

### In-Place Upgrade

```bash
# 1. Backup database
pg_dump -h localhost -U postgres memobase > backup-before-upgrade.sql

# 2. Pull latest image
docker pull memodb-io/memobase-api:0.0.41

# 3. Update docker-compose.yml or K8s manifests

# 4. Restart services
docker-compose up -d
# or
kubectl rollout restart deployment/memobase-api

# 5. Verify health
curl http://localhost:8000/api/v1/healthcheck
```

### Rollback

```bash
# If upgrade fails, revert to previous version
docker-compose down
docker pull memodb-io/memobase-api:0.0.40
docker-compose up -d
```

## Support & Help

- **Documentation**: [docs.memobase.io](https://docs.memobase.io)
- **GitHub Issues**: [github.com/memodb-io/memobase/issues](https://github.com/memodb-io/memobase/issues)
- **Discord**: [Join community](https://discord.gg/YdgwU4d9NB)
- **Email**: contact@memobase.io
