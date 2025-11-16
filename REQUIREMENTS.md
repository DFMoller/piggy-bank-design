# Backend Design Requirements - Octoco Quiz

## Project Overview
Design a backend system that manages user accounts with deposit and withdrawal functionality using Revio (third-party payment provider). The system must be secure, consistent, and resilient to concurrency issues.

## Deliverable Format
- Document-based design (not code implementation)
- Assume a senior frontend developer will handle all UI/client-side logic
- Focus on backend system design

---

## 1. Tech Stack Requirements ✅

### Required Information
- [x] Programming language choice with justification
- [x] Framework selection with justification
- [x] Database technology with justification
- [x] Hosting platform/infrastructure with justification
- [x] Any additional technologies needed

**Decision**: Python + Django + PostgreSQL + Railway
**Documentation**: See `1-tech-stack/TECH_STACK_SUMMARY.md`

---

## 2. System Architecture Requirements ✅

### Architecture Design
- [x] Define overall architecture approach (monolith vs microservices)
- [x] Provide rationale for architectural choice
- [x] Create diagram showing component interactions
- [x] Show data flow between components

### Authentication & Authorization
- [x] Design user authentication mechanism
- [x] Design user authorization system
- [x] Explain how users are authenticated
- [x] Explain how access control is enforced

**Decision**: Modular Monolith with Event-Driven elements
**Authentication**: JWT (JSON Web Tokens) with access/refresh tokens
**Authorization**: Role-Based Access Control (RBAC)
**Documentation**:
- `2-system-architecture/SYSTEM_ARCHITECTURE.md` - Architecture options and recommendation
- `2-system-architecture/ARCHITECTURE_DIAGRAM.md` - High-level component diagram and data flows ⭐
- `2-system-architecture/COMPONENT_DIAGRAM.md` - Detailed reference (supplementary)

---

## 3. Database Design Requirements ✅

### Schema Definition
- [x] Design `users` table/entity
- [x] Design `transactions` table/entity
- [x] Design `balances` table/entity
- [x] Design any other relevant entities

### Database Optimization
- [x] Define indexes for performance optimization
- [x] Define constraints (primary keys, foreign keys, unique, not null, etc.)
- [x] Document any special database features used

**Tables Designed**: users, balances, transactions, webhook_logs, sessions
**Key Features**: SERIALIZABLE isolation, pessimistic locking, partial unique indexes, JSONB
**Documentation**: `3-database-design/DATABASE_SCHEMA.md`

---

## 4. Deposit & Withdrawal Flow Requirements ✅

### Deposit Flow
- [x] Document user-facing deposit flow
- [x] Document backend deposit processing flow
- [x] Explain how deposits are initiated
- [x] Explain how Revio webhooks are handled for deposits
- [x] Create flowchart or sequence diagram for deposit process

### Withdrawal Flow
- [x] Document user-facing withdrawal flow
- [x] Document backend withdrawal processing flow
- [x] Explain how withdrawals are initiated
- [x] Explain how Revio webhooks are handled for withdrawals
- [x] Create flowchart or sequence diagram for withdrawal process

**Documentation**:
- `4-deposit-withdrawal-flows/DEPOSIT_FLOW.md` - Complete deposit flow with sequence diagram, webhook handling, idempotency, and failure scenarios
- `4-deposit-withdrawal-flows/WITHDRAWAL_FLOW.md` - Complete withdrawal flow with sequence diagrams, balance reservation, concurrency safety, and reconciliation

---

## 5. Webhook Handling Requirements ✅

### Webhook Verification
- [x] Design webhook signature verification mechanism
- [x] Implement trust validation for incoming webhooks
- [x] Protect against spoofed webhooks

### Webhook Logging & Auditing
- [x] Design webhook delivery log storage
- [x] Create audit trail for webhook events
- [x] Track webhook processing status

**Key Features**:
- RSA-SHA256 signature verification with Revio's public key
- Multi-layer trust validation (signature, timestamp, event ID deduplication, optional IP allowlist)
- Protection against replay attacks, spoofed webhooks, and MITM attacks
- Comprehensive audit logging using `webhook_logs` table (from Section 3)
- Asynchronous processing with retry strategy and dead letter queue

**Documentation**: `5-webhook-handling/WEBHOOK_HANDLING.md`

---

## 6. Security Requirements ✅

### Authentication & Authorization Security
- [x] Implement secure authentication mechanism
- [x] Implement role-based or policy-based authorization
- [x] Protect user credentials

### Attack Prevention
- [x] Prevent replay attacks on webhooks
- [x] Prevent spoofed webhooks
- [x] Prevent SQL injection
- [x] Prevent privilege escalation

### Sensitive Data Protection
- [x] Secure storage of API keys
- [x] Secure storage of authentication tokens
- [x] Encrypt sensitive data at rest
- [x] Encrypt sensitive data in transit

**Key Mechanisms**:
- **Authentication**: JWT (15-min access + 7-day refresh tokens), bcrypt password hashing
- **Authorization**: RBAC (user/admin roles), resource ownership validation
- **Attack Prevention**: Signature verification (webhooks), Django ORM (SQL injection), rate limiting, security headers
- **Data Protection**: Environment variables (secrets), HTTPS/TLS 1.3, database encryption (AES-256), HTTP-only cookies

**Documentation**: `6-security/SECURITY_DESIGN.md`

---

## 7. Deployment & Hosting Requirements ✅

### Infrastructure
- [x] Select cloud provider (or hosting solution)
- [x] Design containerization strategy (if applicable)
- [x] Set up CI/CD pipeline

### Environment Management
- [x] Configure staging environment
- [x] Configure production environment
- [x] Manage environment-specific configuration

### Observability
- [x] Implement logging system
- [x] Implement monitoring system
- [x] Implement alerting mechanism
- [x] Define key metrics to track

**Cloud Provider**: Railway (Docker-native PaaS with managed PostgreSQL + Redis)

**Containerization**:
- Multi-container Docker architecture (web, worker, beat)
- Docker Compose for local development
- Dockerfile with Gunicorn + Django
- Railway deployment with 2 web replicas + 2 worker replicas

**CI/CD**:
- GitHub Actions (PR checks: lint, test, security scan, Docker build)
- Auto-deploy to staging on merge to main
- Manual approval gate for production deployment
- One-click rollback via Railway

**Environments**: Development (local), Staging (auto-deploy), Production (manual approval)

**Observability**:
- Structured JSON logging to stdout (Railway logs)
- Sentry for error tracking + APM
- Railway metrics for infrastructure monitoring
- Health check endpoint + alerts (Slack, email)
- Key metrics: error rate, response time, transaction throughput, resource usage

**Documentation**: `7-deployment-hosting/DEPLOYMENT_DESIGN.md`

---

## 8. Failure Handling Requirements ✅

### Edge Cases & Risks
- [x] Handle duplicate transaction scenarios
- [x] Handle Revio downtime
- [x] Handle network failures
- [x] Handle race conditions/concurrency issues
- [x] Handle partial failures

### Recovery Mechanisms
- [x] Implement retry logic with backoff
- [x] Implement dead letter queue for failed operations
- [x] Design failure detection system
- [x] Design recovery procedures

**Edge Cases**:
- Duplicate transactions: Idempotency keys + pessimistic locking
- Revio downtime: Refund balance, retry with exponential backoff, DLQ after 4 attempts
- Network failures: Timeouts, connection pooling, health checks
- Race conditions: SELECT FOR UPDATE, event ID deduplication
- Partial failures: Two-phase pattern (critical ops succeed, best-effort async)

**Recovery**:
- Retry logic: 4 attempts (immediate, 1min, 5min, 15min) with exponential backoff
- Dead letter queue: Manual admin review after max retries
- Detection: Health checks, error rate monitoring, stuck transaction cron jobs
- Reconciliation: Every 15 min check for missed webhooks, query Revio API

**Documentation**: `8-failure-handling/FAILURE_HANDLING.md`

---

## 9. Bonus Requirements (Optional) ✅

### Transaction Dashboard
- [x] Design API for transaction history retrieval
- [x] Define data structure for transaction history
- [x] Explain pagination/filtering strategy

### Scalability
- [x] Describe approach to scale to 100x usage
- [x] Identify bottlenecks
- [x] Propose scaling strategies (horizontal/vertical)
- [x] Discuss caching strategies
- [x] Discuss database scaling approach

**Documentation**:
- `9-bonus/TRANSACTION_DASHBOARD.md` - REST API design with pagination (offset & cursor), filtering, sorting, search, and export functionality
- `9-bonus/SCALABILITY.md` - 3-phase scaling strategy (1x → 10x → 50x → 100x) with bottleneck analysis and cost projections

---

## Design Guidelines

### Production-Ready Thinking
- Think like an engineer building a production system, not a prototype
- Consider operational requirements from day one

### Documentation Standards
- Use diagrams extensively
- Use proper technical terminology:
  - Serializable isolation
  - Pessimistic locking
  - JWT
  - Event sourcing
  - Idempotency
  - etc.

### Design Philosophy
- Show tradeoffs in technical decisions
- Start with solid base design
- Explain evolution path for future growth
- Don't over-engineer initial solution

---

## Key System Properties

### Consistency
- System must maintain accurate user balances
- No double-spending
- No lost transactions

### Security
- Protect against unauthorized access
- Protect against common web vulnerabilities
- Secure integration with Revio

### Resilience
- Handle concurrent operations safely
- Recover from failures gracefully
- Maintain data integrity during failures
