# Deployment & Hosting Design

## Overview
This document outlines the Docker-based deployment strategy, CI/CD pipeline, environment management, and observability setup for the piggy bank backend system.

---

## Infrastructure Overview

### Cloud Provider: Railway

**Selected Platform**: Railway.app

**Why Railway?**
- ✅ Native Docker support (deploy from Dockerfile)
- ✅ Managed PostgreSQL and Redis included
- ✅ Simple deployment (git push to deploy)
- ✅ Automatic HTTPS with custom domains
- ✅ Environment variable management
- ✅ Cost-effective (~$110/month for production)
- ✅ Built-in metrics and logs

**Alternative**: Render.com (similar features and pricing)

---

## Containerization Strategy

### Container Architecture

```mermaid
graph TB
    subgraph "Production Deployment"
        LB[Railway Load Balancer<br/>HTTPS/TLS Termination]

        subgraph "Application Tier"
            WEB1[Django Web Container 1<br/>Gunicorn + Django]
            WEB2[Django Web Container 2<br/>Gunicorn + Django]
            WORKER1[Celery Worker Container 1<br/>Background Tasks]
            WORKER2[Celery Worker Container 2<br/>Background Tasks]
        end

        subgraph "Data Tier"
            PG[(PostgreSQL<br/>Managed by Railway)]
            REDIS[(Redis<br/>Managed by Railway)]
        end

        LB --> WEB1
        LB --> WEB2
        WEB1 --> PG
        WEB2 --> PG
        WEB1 --> REDIS
        WEB2 --> REDIS
        WORKER1 --> PG
        WORKER2 --> PG
        WORKER1 --> REDIS
        WORKER2 --> REDIS
    end
```

### Docker Images

**1. Web Container** (Django + Gunicorn)
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Collect static files
RUN python manage.py collectstatic --noinput

# Run migrations and start Gunicorn
CMD python manage.py migrate && \
    gunicorn octoco.wsgi:application \
    --bind 0.0.0.0:$PORT \
    --workers 4 \
    --timeout 30 \
    --access-logfile - \
    --error-logfile -
```

**2. Celery Worker Container** (same base image, different command)
```dockerfile
# Same Dockerfile, different CMD
CMD celery -A octoco worker \
    --loglevel=info \
    --concurrency=4 \
    --max-tasks-per-child=1000
```

**3. Celery Beat Container** (scheduled tasks - optional)
```dockerfile
# Same Dockerfile, different CMD
CMD celery -A octoco beat \
    --loglevel=info \
    --scheduler django_celery_beat.schedulers:DatabaseScheduler
```

### Multi-Service Deployment

**Railway Configuration** (railway.json):
```json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": {
    "builder": "DOCKERFILE",
    "dockerfilePath": "Dockerfile"
  },
  "deploy": {
    "numReplicas": 2,
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 10
  }
}
```

**Services Deployed**:
1. `web` - Django application (2 replicas)
2. `worker` - Celery workers (2 replicas)
3. `postgres` - PostgreSQL database (managed service)
4. `redis` - Redis cache/broker (managed service)

---

## Local Development with Docker Compose

### docker-compose.yml

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: octoco_dev
      POSTGRES_USER: octoco
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U octoco"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Redis Cache & Broker
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  # Django Web Application
  web:
    build:
      context: .
      dockerfile: Dockerfile
    command: >
      sh -c "python manage.py migrate &&
             python manage.py runserver 0.0.0.0:8000"
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    env_file:
      - .env.local
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  # Celery Worker
  worker:
    build:
      context: .
      dockerfile: Dockerfile
    command: celery -A octoco worker --loglevel=info
    volumes:
      - .:/app
    env_file:
      - .env.local
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  # Celery Beat (Scheduled Tasks)
  beat:
    build:
      context: .
      dockerfile: Dockerfile
    command: celery -A octoco beat --loglevel=info
    volumes:
      - .:/app
    env_file:
      - .env.local
    depends_on:
      - redis

volumes:
  postgres_data:
```

**Usage**:
```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f web

# Run migrations
docker-compose exec web python manage.py migrate

# Stop all services
docker-compose down
```

---

## CI/CD Pipeline

### GitHub Actions Workflow

```mermaid
flowchart TB
    START([Push to GitHub]) --> TRIGGER{Branch?}

    TRIGGER -->|feature/*| PR[Pull Request Checks]
    TRIGGER -->|main| DEPLOY[Deployment Pipeline]

    PR --> LINT[Lint & Format<br/>black, flake8, mypy]
    LINT --> TEST[Run Tests<br/>pytest]
    TEST --> SECURITY[Security Scan<br/>pip-audit, bandit]
    SECURITY --> BUILD[Build Docker Image]
    BUILD --> PR_END([PR Ready for Review])

    DEPLOY --> BUILD2[Build Docker Image]
    BUILD2 --> TAG[Tag Image<br/>ghcr.io/org/app:latest]
    TAG --> PUSH_STAGING[Deploy to Staging]
    PUSH_STAGING --> SMOKE[Run Smoke Tests]
    SMOKE --> MANUAL{Manual Approval}
    MANUAL -->|Approved| PUSH_PROD[Deploy to Production]
    MANUAL -->|Rejected| ROLLBACK[Stop - No Deploy]
    PUSH_PROD --> VERIFY[Health Check]
    VERIFY --> DONE([Deployment Complete])
```

### GitHub Actions Configuration

**.github/workflows/ci.yml** (Pull Requests):
```yaml
name: CI Pipeline

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test_password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Lint with flake8
        run: flake8 apps/

      - name: Format check with black
        run: black --check apps/

      - name: Type check with mypy
        run: mypy apps/

      - name: Security audit
        run: |
          pip-audit
          bandit -r apps/ -f json -o bandit-report.json

      - name: Run tests
        env:
          DATABASE_URL: postgresql://postgres:test_password@localhost/test_db
          REDIS_URL: redis://localhost:6379/0
        run: |
          pytest --cov=apps --cov-report=xml --cov-report=html

      - name: Build Docker image
        run: docker build -t octoco-backend:test .
```

**.github/workflows/deploy.yml** (Main branch):
```yaml
name: Deploy Pipeline

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            ghcr.io/org/octoco-backend:latest
            ghcr.io/org/octoco-backend:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to Railway (Staging)
        run: |
          npm install -g @railway/cli
          railway up --service web --environment staging
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN_STAGING }}

      - name: Run smoke tests
        run: |
          sleep 10  # Wait for deployment
          curl -f https://staging.octoco.app/health || exit 1

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval

    steps:
      - name: Deploy to Railway (Production)
        run: |
          npm install -g @railway/cli
          railway up --service web --environment production
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN_PRODUCTION }}

      - name: Health check
        run: |
          sleep 10
          curl -f https://api.octoco.app/health || exit 1

      - name: Notify team
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Production deployment completed!'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Environment Management

### Environment Separation

| Environment | Purpose | URL | Database | Auto-Deploy |
|-------------|---------|-----|----------|-------------|
| **Development** | Local development | localhost:8000 | Local PostgreSQL | N/A |
| **Staging** | Pre-production testing | staging.octoco.app | Railway Staging DB | ✅ Yes (on merge to main) |
| **Production** | Live system | api.octoco.app | Railway Production DB | ⚠️ Manual approval required |

### Environment Variables

**Shared Variables** (all environments):
```bash
# Django
DJANGO_SETTINGS_MODULE=octoco.settings.production
SECRET_KEY=<generated-secret>

# Database (injected by Railway)
DATABASE_URL=postgresql://user:pass@host:5432/db

# Redis (injected by Railway)
REDIS_URL=redis://host:6379/0

# Revio API
REVIO_API_KEY=<environment-specific-key>
REVIO_WEBHOOK_SECRET=<environment-specific-secret>

# External Services
SENDGRID_API_KEY=<api-key>
SENTRY_DSN=<environment-specific-dsn>

# Application
ALLOWED_HOSTS=api.octoco.app
CORS_ALLOWED_ORIGINS=https://octoco.app
```

**Environment-Specific Overrides**:

**Development** (.env.local):
```bash
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1
REVIO_API_KEY=sk_test_abc123
LOG_LEVEL=DEBUG
```

**Staging** (Railway environment):
```bash
DEBUG=False
ALLOWED_HOSTS=staging.octoco.app
REVIO_API_KEY=sk_test_abc123  # Use test keys
SENTRY_ENVIRONMENT=staging
LOG_LEVEL=INFO
```

**Production** (Railway environment):
```bash
DEBUG=False
ALLOWED_HOSTS=api.octoco.app
REVIO_API_KEY=sk_live_xyz789  # Use live keys
SENTRY_ENVIRONMENT=production
LOG_LEVEL=WARNING
```

### Django Settings Structure

```python
# octoco/settings/base.py - Shared settings
# octoco/settings/development.py - Local dev overrides
# octoco/settings/staging.py - Staging overrides
# octoco/settings/production.py - Production overrides

# settings/production.py
from .base import *

DEBUG = False
ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS', '').split(',')

# Security
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# Database (from Railway)
DATABASES = {
    'default': dj_database_url.config(
        conn_max_age=600,
        conn_health_checks=True,
    )
}

# Logging
LOGGING = {
    'version': 1,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'json',
        },
    },
    'formatters': {
        'json': {
            '()': 'pythonjsonlogger.jsonlogger.JsonFormatter',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': os.getenv('LOG_LEVEL', 'INFO'),
    },
}
```

---

## Observability

### Logging System

**Logging Strategy**: Structured JSON logging to stdout, collected by Railway

**Log Levels**:
- `DEBUG` - Development only
- `INFO` - Standard operations (requests, transactions)
- `WARNING` - Recoverable issues (retry attempts, slow queries)
- `ERROR` - Failed operations (exceptions, webhook failures)
- `CRITICAL` - System failures (database down, service unavailable)

**Structured Logging Format**:
```python
import structlog

logger = structlog.get_logger()

logger.info(
    "transaction_created",
    transaction_id="uuid-123",
    user_id="uuid-456",
    type="deposit",
    amount=100.00,
    status="pending",
    request_id="req-abc",
)
```

**Log Output** (JSON):
```json
{
  "event": "transaction_created",
  "transaction_id": "uuid-123",
  "user_id": "uuid-456",
  "type": "deposit",
  "amount": 100.00,
  "status": "pending",
  "request_id": "req-abc",
  "timestamp": "2025-01-15T10:30:00.123Z",
  "level": "info",
  "logger": "apps.transactions"
}
```

**What to Log**:
- All HTTP requests (method, path, status, duration)
- Authentication events (login, logout, failures)
- Transaction operations (create, update, complete)
- Webhook events (received, verified, processed)
- External API calls (Revio, SendGrid)
- Background job execution
- Errors and exceptions (with stack traces)

---

### Monitoring System

**Primary Tool**: Sentry (error tracking + APM)

**Metrics Tracked**:

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **Error Rate** | Exceptions per minute | > 10/min |
| **Response Time** | p95 latency | > 1000ms |
| **Transaction Throughput** | Transactions/hour | < 10/hour (too low) |
| **Database Connections** | Active connections | > 80% of max |
| **Celery Queue Length** | Pending tasks | > 1000 |
| **Memory Usage** | Container memory | > 90% |
| **CPU Usage** | Container CPU | > 80% sustained |

**Railway Built-in Metrics**:
- Container CPU usage
- Container memory usage
- Network I/O
- Deployment history
- Request count and latency

**Sentry Configuration**:
```python
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration
from sentry_sdk.integrations.celery import CeleryIntegration

sentry_sdk.init(
    dsn=os.getenv('SENTRY_DSN'),
    environment=os.getenv('SENTRY_ENVIRONMENT', 'production'),
    integrations=[
        DjangoIntegration(),
        CeleryIntegration(),
    ],
    traces_sample_rate=0.1,  # Sample 10% of transactions for APM
    profiles_sample_rate=0.1,  # Sample 10% for profiling
    send_default_pii=False,  # Don't send PII to Sentry
)
```

---

### Alerting Mechanism

**Alert Channels**:
1. **Sentry** - Automatic alerts on errors
2. **Slack** - Team notifications
3. **PagerDuty** - On-call escalation (optional)
4. **Email** - Critical alerts to admin

**Alert Rules**:

**Critical Alerts** (page immediately):
| Alert | Condition | Action |
|-------|-----------|--------|
| Service Down | Health check fails 3 times | Page on-call engineer |
| Database Unreachable | Connection errors | Page database admin |
| High Error Rate | > 50 errors/min | Page backend team |
| Zero Webhooks | No webhooks for 2 hours | Page on-call engineer |

**Warning Alerts** (investigate within 1 hour):
| Alert | Condition | Action |
|-------|-----------|--------|
| Slow Responses | p95 > 2000ms for 10 min | Slack notification |
| High Memory | > 85% for 15 min | Slack notification |
| Celery Queue Backlog | > 500 tasks for 10 min | Slack notification |
| Failed Webhook Processing | > 5% failure rate | Slack notification |

**Info Alerts** (track, no action):
| Alert | Condition | Action |
|-------|-----------|--------|
| Deployment Complete | After each deploy | Slack notification |
| Daily Summary | Once per day | Email summary |

**Sentry Alert Configuration**:
```python
# Sentry automatically alerts on:
# - New error types
# - Error spike (10x normal rate)
# - Regression (error reappears after being resolved)
```

**Custom Health Check Endpoint**:
```python
# apps/core/views.py
@api_view(['GET'])
@permission_classes([AllowAny])
def health_check(request):
    """
    Health check endpoint for load balancer and monitoring.
    """
    checks = {
        'status': 'healthy',
        'database': check_database(),
        'redis': check_redis(),
        'celery': check_celery_workers(),
        'timestamp': timezone.now().isoformat(),
    }

    if not all([checks['database'], checks['redis'], checks['celery']]):
        checks['status'] = 'unhealthy'
        return JsonResponse(checks, status=503)

    return JsonResponse(checks, status=200)
```

---

### Key Metrics to Track

**Business Metrics**:
- Total deposits (count, volume)
- Total withdrawals (count, volume)
- Active users (daily, weekly, monthly)
- Transaction success rate
- Average transaction value

**Technical Metrics**:
- Request rate (requests/sec)
- Error rate (errors/sec, percentage)
- Response time (p50, p95, p99)
- Database query time (p95, slow queries)
- Celery task duration (p95)
- Webhook processing time (p95)

**Infrastructure Metrics**:
- CPU utilization (per container)
- Memory utilization (per container)
- Database connections (active, idle)
- Redis memory usage
- Disk usage (database)

**Dashboard Example** (using Railway + Sentry):
```
┌─────────────────────────────────────────┐
│ Octoco Backend - Production Dashboard  │
├─────────────────────────────────────────┤
│ Status: ✅ Healthy                      │
│ Uptime: 99.95% (30 days)               │
│                                         │
│ Traffic                                 │
│ • Requests/min: 1,234                  │
│ • Error rate: 0.12%                    │
│ • p95 latency: 234ms                   │
│                                         │
│ Transactions (Last 24h)                │
│ • Deposits: 145 ($12,340)              │
│ • Withdrawals: 89 ($8,920)             │
│ • Success rate: 99.7%                  │
│                                         │
│ Infrastructure                          │
│ • Web containers: 2 (CPU: 45%, Mem: 62%)│
│ • Workers: 2 (Queue: 23 tasks)         │
│ • Database: 23% usage                  │
│ • Redis: 128MB used                    │
└─────────────────────────────────────────┘
```

---

## Deployment Process

### Standard Deployment Flow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GitHub
    participant CI as GitHub Actions
    participant Staging
    participant Team
    participant Prod as Production

    Dev->>GitHub: git push origin feature/new-feature
    GitHub->>CI: Trigger PR checks
    CI->>CI: Lint, test, security scan
    CI-->>GitHub: ✅ All checks passed

    Dev->>GitHub: Merge PR to main
    GitHub->>CI: Trigger deployment pipeline
    CI->>CI: Build Docker image
    CI->>Staging: Deploy to staging
    Staging-->>CI: Deployment successful
    CI->>CI: Run smoke tests
    CI-->>Team: Slack: Staging deployed

    Team->>GitHub: Manual approval (review staging)
    GitHub->>CI: Continue to production
    CI->>Prod: Deploy to production
    Prod-->>CI: Deployment successful
    CI->>Prod: Health check
    Prod-->>CI: ✅ Healthy
    CI-->>Team: Slack: Production deployed
```

### Rollback Strategy

**Automatic Rollback** (Railway feature):
- Keep last 10 deployments
- One-click rollback in Railway dashboard
- DNS/routing stays same (instant switch)

**Manual Rollback Process**:
```bash
# Option 1: Railway CLI
railway rollback --service web

# Option 2: Redeploy previous commit
git revert HEAD
git push origin main
# CI/CD deploys reverted version
```

**Database Rollback**:
- Django migrations are forward-only by design
- For critical issues: Write reverse migration
- Test reverse migration in staging first
- Apply to production only if necessary

---

## Deployment Checklist

### Pre-Deployment
- [ ] All tests passing in CI
- [ ] Security audit clean (no critical vulnerabilities)
- [ ] Database migrations reviewed (no data loss)
- [ ] Environment variables configured
- [ ] Staging deployment successful
- [ ] Smoke tests passing

### During Deployment
- [ ] Monitor error rates in Sentry
- [ ] Watch Railway deployment logs
- [ ] Health check endpoint responding
- [ ] Database migrations applied successfully
- [ ] Celery workers connected

### Post-Deployment
- [ ] Verify key user flows (deposit, withdrawal)
- [ ] Check metrics dashboard (no spikes)
- [ ] Monitor for 15 minutes
- [ ] Update team in Slack
- [ ] Tag release in GitHub

---

## Summary

**Infrastructure**: Railway (Docker-native PaaS), managed PostgreSQL + Redis

**Containerization**:
- Multi-container architecture (web, worker, beat)
- Docker Compose for local development
- Railway for production deployment (2 web replicas, 2 worker replicas)

**CI/CD**:
- GitHub Actions for automated testing and deployment
- PR checks: lint, test, security scan, Docker build
- Main branch: auto-deploy to staging → manual approval → production
- Rollback capability (one-click via Railway)

**Environments**:
- Development (local Docker Compose)
- Staging (Railway, auto-deployed)
- Production (Railway, manual approval)
- Environment-specific configuration via Railway secrets

**Observability**:
- Structured JSON logging (stdout → Railway logs)
- Sentry for error tracking + APM
- Railway metrics for infrastructure monitoring
- Health check endpoint for load balancer
- Slack + email alerts for critical issues

**Key Metrics**: Error rate, response time, transaction throughput, queue length, resource usage
