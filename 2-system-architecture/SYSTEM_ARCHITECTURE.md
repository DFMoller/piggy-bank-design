# System Architecture - Options and Recommendation

## Overview
This document analyzes different architectural approaches for the Octoco Quiz backend system and provides a recommendation based on the project requirements, team constraints, and scaling considerations.

---

## Context

### System Requirements Recap
- User account management with deposits and withdrawals
- Integration with Revio payment provider
- Webhook handling for asynchronous payment notifications
- Balance tracking with strong consistency guarantees
- Security: authentication, authorization, webhook verification
- Resilience: concurrent operations, failure recovery
- Observability: logging, monitoring, alerting

### Key Constraints
- **Tech Stack**: Python + Django + PostgreSQL + Railway
- **Team Size**: Small team (assumed)
- **Timeline**: Need production-ready system quickly
- **Scale**: Initially small (0-1,000 users), potential to scale to 100x
- **Budget**: Cost-conscious (~$125/month initial budget)

### Critical System Properties
1. **Consistency**: No double-spending, accurate balances
2. **Security**: Secure payment processing, webhook verification
3. **Resilience**: Handle concurrent requests, recover from failures
4. **Performance**: Sub-second response times
5. **Maintainability**: Easy to understand, test, and evolve

---

## Architecture Option 1: Modular Monolith (RECOMMENDED)

### Overview
A single Django application with well-defined internal boundaries, organized as separate Django apps with clear responsibilities.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         Load Balancer                            │
│                      (Railway/Cloudflare)                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Django Application                          │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    API Layer (DRF)                       │   │
│  │  - REST endpoints                                        │   │
│  │  - Request validation                                    │   │
│  │  - Authentication middleware                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                             │                                    │
│  ┌──────────────┬──────────┴────────┬─────────────┬──────────┐ │
│  │              │                   │             │          │ │
│  ▼              ▼                   ▼             ▼          ▼ │
│  ┌────────┐  ┌───────────┐  ┌────────────┐  ┌─────────┐  ┌──┐│
│  │ Auth   │  │Transaction│  │  Webhook   │  │  Core   │  │.││
│  │  App   │  │    App    │  │    App     │  │   App   │  │.││
│  │        │  │           │  │            │  │         │  │.││
│  │- Users │  │- Deposits │  │- Signature │  │- Utils  │  │  ││
│  │- JWT   │  │- Withdraw │  │  Verify    │  │- Base   │  │  ││
│  │- Perms │  │- Balance  │  │- Process   │  │  Models │  │  ││
│  └────┬───┘  └─────┬─────┘  └──────┬─────┘  └────┬────┘  └──┘│
│       │            │               │             │            │
│       └────────────┴───────────────┴─────────────┘            │
│                             │                                  │
│                             ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              Django ORM (Data Access Layer)              │ │
│  └─────────────────────────────────────────────────────────┘ │
└────────────────────────────┬────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌──────────────┐    ┌─────────────────┐    ┌──────────────┐
│  PostgreSQL  │    │  Redis Cache    │    │    Celery    │
│   Database   │    │  - Sessions     │    │  Background  │
│              │    │  - Rate limit   │    │    Worker    │
│  - Users     │    │  - Temp data    │    │              │
│  - Trans     │    └─────────────────┘    │  - Async     │
│  - Balances  │                           │    tasks     │
│  - Webhooks  │                           │  - Retry     │
└──────────────┘                           │    logic     │
                                           └──────────────┘

External Services:
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Revio API    │    │   Sentry     │    │   SendGrid   │
│ - Purchases  │    │  (Errors)    │    │   (Email)    │
│ - Payouts    │    │              │    │              │
│ - Webhooks   │    └──────────────┘    └──────────────┘
└──────────────┘
```

### Internal Structure

```
octoco_quiz/
├── apps/
│   ├── accounts/              # User management & authentication
│   │   ├── models.py         # User, Profile models
│   │   ├── views.py          # Login, logout, register endpoints
│   │   ├── serializers.py    # User data validation
│   │   ├── permissions.py    # Authorization logic
│   │   └── services.py       # Business logic (user operations)
│   │
│   ├── transactions/          # Financial transactions
│   │   ├── models.py         # Transaction, Balance models
│   │   ├── views.py          # Deposit, withdrawal endpoints
│   │   ├── serializers.py    # Transaction validation
│   │   ├── services.py       # Business logic (balance updates)
│   │   └── tasks.py          # Celery tasks (polling, reconciliation)
│   │
│   ├── webhooks/              # Webhook handling
│   │   ├── models.py         # WebhookLog, WebhookEvent models
│   │   ├── views.py          # Webhook receiver endpoint
│   │   ├── handlers.py       # Event-specific handlers
│   │   ├── verification.py   # Signature verification
│   │   └── idempotency.py    # Duplicate detection
│   │
│   ├── notifications/         # User notifications
│   │   ├── models.py         # Notification model
│   │   ├── services.py       # Email, SMS logic
│   │   └── tasks.py          # Async notification sending
│   │
│   └── core/                  # Shared utilities
│       ├── models.py         # Abstract base models
│       ├── exceptions.py     # Custom exceptions
│       ├── middleware.py     # Request/response processing
│       ├── decorators.py     # Reusable decorators
│       └── utils.py          # Helper functions
│
├── octoco/                    # Django project settings
│   ├── settings/
│   │   ├── base.py           # Shared settings
│   │   ├── development.py    # Dev-specific settings
│   │   └── production.py     # Prod-specific settings
│   ├── urls.py               # URL routing
│   └── wsgi.py               # WSGI entry point
│
└── tests/                     # Tests
    ├── integration/
    ├── unit/
    └── e2e/
```

### Communication Pattern

**Internal Communication:**
- Direct function calls between Django apps
- Shared database for data consistency
- Django signals for loose coupling (optional)

**External Communication:**
- REST API calls to Revio (HTTP/HTTPS)
- Webhook receivers for Revio callbacks
- Redis for caching and rate limiting
- Celery for background job processing

### Data Flow Example: Deposit

```
1. User → Frontend: Click "Deposit"
2. Frontend → Backend (API): POST /api/deposits {amount}
3. Backend (Transaction App):
   a. Validate request
   b. Create pending transaction in DB
   c. Call Revio API to create purchase
   d. Store purchase_id
   e. Return checkout_url to frontend
4. Revio → Backend (Webhook App): POST /webhooks/revio {purchase.paid}
5. Webhook App:
   a. Verify signature
   b. Check idempotency
   c. Call Transaction App service to update balance
6. Transaction App:
   a. Begin database transaction (SERIALIZABLE)
   b. Lock user balance row (SELECT FOR UPDATE)
   c. Update balance
   d. Update transaction status
   e. Commit transaction
7. Webhook App: Return 200 OK to Revio
```

### Pros

✅ **Simple to develop and test**
- Single codebase, unified development environment
- Easy to run locally with `python manage.py runserver`
- Simplified debugging (single process to debug)

✅ **Fast performance**
- No network overhead between components
- Single database connection pool
- Minimal serialization/deserialization

✅ **Easy deployment**
- Single deployment artifact
- Simple CI/CD pipeline
- One service to monitor

✅ **Strong consistency**
- ACID transactions across all operations
- No distributed transaction complexity
- Guaranteed data integrity

✅ **Lower operational complexity**
- Fewer moving parts
- Simpler monitoring and logging
- Easier to reason about system behavior

✅ **Cost-effective**
- Single application server
- One database
- Minimal infrastructure overhead (~$110/month)

✅ **Perfect for Django**
- Leverages Django's batteries-included philosophy
- Uses Django ORM, admin, migrations effectively
- Well-documented pattern

### Cons

❌ **Coupled deployment**
- All components deploy together
- Small change requires full deployment
- Higher risk per deployment

❌ **Scaling constraints**
- Scale entire app even if only one part needs it
- Cannot independently scale webhook processing vs API

❌ **Technology constraints**
- All components must use Python/Django
- Harder to adopt new languages for specific use cases

❌ **Potential for tight coupling**
- Risk of creating tangled dependencies
- Requires discipline to maintain boundaries

### Mitigation Strategies

**Prevent tight coupling:**
- Use service layer pattern (business logic in `services.py`)
- Clear interfaces between Django apps
- Avoid direct model access across apps (use services)
- Document module boundaries

**Enable future migration to microservices:**
- Design apps with single responsibility
- Use dependency injection where possible
- Keep external API clients separate from business logic
- Design for horizontal scaling from day one

**Scaling approach:**
- Horizontal scaling: Run multiple instances behind load balancer
- Separate Celery workers for background tasks
- Read replicas for database reads
- Redis caching for frequently accessed data

### When to Use
✅ **Use modular monolith if:**
- Team size < 10 people
- Initial user base < 10,000 users
- Need to ship quickly
- Want simpler operations
- Budget-conscious
- Team experienced with Django

---

## Architecture Option 2: Microservices Architecture

### Overview
Decompose the system into independently deployable services, each responsible for a specific business capability.

### Architecture Diagram

```
                        ┌─────────────────┐
                        │  API Gateway    │
                        │  (Kong/Tyk)     │
                        └────────┬────────┘
                                 │
                 ┌───────────────┼───────────────┐
                 │               │               │
                 ▼               ▼               ▼
        ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
        │   Auth      │  │ Transaction │  │  Webhook    │
        │  Service    │  │   Service   │  │  Service    │
        │             │  │             │  │             │
        │ - Login     │  │ - Deposits  │  │ - Verify    │
        │ - Register  │  │ - Withdraw  │  │ - Process   │
        │ - JWT       │  │ - Balance   │  │ - Log       │
        └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
               │                │                │
               ▼                ▼                ▼
        ┌──────────┐     ┌──────────┐     ┌──────────┐
        │  Users   │     │  Trans   │     │ Webhooks │
        │   DB     │     │   DB     │     │   DB     │
        └──────────┘     └──────────┘     └──────────┘
                 │               │               │
                 └───────────────┼───────────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │   Message Broker       │
                    │   (RabbitMQ/Redis)     │
                    └────────────────────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │  Notification Service  │
                    │  - Email               │
                    │  - SMS                 │
                    └────────────────────────┘
```

### Service Breakdown

**1. Auth Service**
- User registration and login
- JWT token generation and validation
- Password reset
- User profile management
- Database: User accounts

**2. Transaction Service**
- Deposit creation and processing
- Withdrawal creation and processing
- Balance queries and updates
- Transaction history
- Database: Transactions, balances

**3. Webhook Service**
- Receive webhooks from Revio
- Verify signatures
- Process events
- Publish events to message broker
- Database: Webhook logs

**4. Notification Service**
- Send emails (deposit confirmed, withdrawal completed)
- Send SMS (optional)
- Push notifications (optional)
- Database: Notification logs

**5. API Gateway**
- Route requests to appropriate service
- Rate limiting
- Authentication
- Request/response transformation

### Communication Patterns

**Synchronous (REST):**
- Frontend → API Gateway → Services
- Transaction Service → Auth Service (validate user)

**Asynchronous (Message Broker):**
- Webhook Service → Message Broker → Transaction Service
- Transaction Service → Message Broker → Notification Service

**Event Flow:**
```
1. Revio sends webhook to Webhook Service
2. Webhook Service verifies and publishes event to broker
3. Transaction Service consumes event and updates balance
4. Transaction Service publishes balance_updated event
5. Notification Service consumes event and sends email
```

### Pros

✅ **Independent scaling**
- Scale webhook processing separately from API
- Optimize resources per service

✅ **Technology flexibility**
- Use Go for high-throughput webhook service
- Keep Python for business logic
- Choose best tool for each job

✅ **Independent deployment**
- Deploy services independently
- Lower risk per deployment
- Faster iteration on specific features

✅ **Team autonomy**
- Different teams can own different services
- Clear boundaries and contracts

✅ **Failure isolation**
- Webhook service failure doesn't affect API
- Graceful degradation possible

### Cons

❌ **High operational complexity**
- Multiple services to deploy and monitor
- Complex orchestration
- Distributed logging and tracing needed

❌ **Distributed transaction challenges**
- No ACID across services
- Need eventual consistency patterns
- Saga pattern for multi-step transactions

❌ **Network overhead**
- Service-to-service latency
- Serialization/deserialization costs
- More failure points

❌ **Higher infrastructure costs**
- Multiple databases and services
- Message broker infrastructure
- API gateway
- Estimated cost: ~$400-500/month

❌ **Development complexity**
- More boilerplate (API clients, contracts)
- Complex local development setup
- Harder to test end-to-end

❌ **Data consistency challenges**
- No foreign keys across databases
- Eventual consistency required
- Complex reconciliation logic

### When to Use
✅ **Use microservices if:**
- Large team (> 20 people) with multiple squads
- User base > 100,000 users
- Different scaling needs per component
- Need to use multiple technologies
- Have strong DevOps capabilities

---

## Architecture Option 3: Event-Driven Architecture (Hybrid)

### Overview
Monolith for synchronous operations, event-driven for asynchronous operations using message queues.

### Architecture Diagram

```
                    ┌─────────────────────────┐
                    │   Django Monolith       │
                    │                         │
                    │  ┌──────────────────┐   │
                    │  │  REST API        │   │
                    │  └────────┬─────────┘   │
                    │           │             │
                    │  ┌────────▼─────────┐   │
                    │  │  Business Logic  │   │
                    │  │  - Deposits      │   │
                    │  │  - Withdrawals   │   │
                    │  │  - Auth          │   │
                    │  └────────┬─────────┘   │
                    │           │             │
                    │           │             │
                    └───────────┼─────────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
            ┌───────────────┐       ┌──────────────┐
            │  PostgreSQL   │       │    Redis     │
            │   Database    │       │  (Pub/Sub)   │
            └───────────────┘       └──────┬───────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
                    ▼                      ▼                      ▼
          ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
          │ Webhook Processor│   │  Reconciliation  │   │   Notification   │
          │ Celery Worker    │   │  Celery Worker   │   │  Celery Worker   │
          │                  │   │                  │   │                  │
          │ - Process events │   │ - Sync w/ Revio  │   │ - Send emails    │
          │ - Update balance │   │ - Fix orphans    │   │ - Send SMS       │
          └──────────────────┘   └──────────────────┘   └──────────────────┘
```

### Event Flow

**1. Webhook arrives:**
```
Revio → Django (webhook endpoint) → Verify signature → Publish event to Redis
→ Celery worker picks up event → Process balance update → Publish balance_updated
→ Notification worker picks up → Send email
```

**2. Reconciliation (scheduled):**
```
Cron trigger → Celery beat → Reconciliation worker
→ Query Revio API for all transactions in last 24h
→ Compare with internal DB
→ Publish missing_transaction events
→ Transaction worker processes missing events
```

### Event Types

```python
# events.py
class EventType:
    WEBHOOK_RECEIVED = "webhook.received"
    DEPOSIT_COMPLETED = "deposit.completed"
    WITHDRAWAL_COMPLETED = "withdrawal.completed"
    BALANCE_UPDATED = "balance.updated"
    TRANSACTION_FAILED = "transaction.failed"
    RECONCILIATION_NEEDED = "reconciliation.needed"

# Example event
{
    "event_type": "deposit.completed",
    "event_id": "evt_123456",
    "timestamp": "2024-01-15T10:30:00Z",
    "data": {
        "user_id": "usr_abc",
        "transaction_id": "txn_xyz",
        "amount": "100.00",
        "currency": "USD"
    }
}
```

### Pros

✅ **Best of both worlds**
- Simple synchronous operations (API calls)
- Scalable asynchronous processing (webhooks, notifications)

✅ **Decoupled components**
- Workers can be scaled independently
- Easy to add new event subscribers

✅ **Resilient**
- Failed events can be retried
- Dead letter queue for persistent failures
- Audit trail of all events

✅ **Moderate complexity**
- Simpler than full microservices
- More flexible than pure monolith

✅ **Cost-effective**
- Estimated: ~$150-200/month
- Uses Celery + Redis (already in stack)

### Cons

❌ **Eventual consistency**
- Events processed asynchronously
- Small delay between action and effect

❌ **Event schema management**
- Need to version events
- Breaking changes require migration

❌ **Debugging complexity**
- Async flows harder to trace
- Need distributed tracing

### When to Use
✅ **Use event-driven if:**
- Need to process many webhooks
- Want to decouple long-running tasks
- Need audit trail of all events
- Want flexibility to add new consumers

---

## Architecture Option 4: Serverless Architecture

### Overview
Use serverless functions (AWS Lambda, Azure Functions, Google Cloud Functions) for compute, with managed database.

### Architecture Diagram

```
                        ┌─────────────────┐
                        │   API Gateway   │
                        │  (AWS/Azure)    │
                        └────────┬────────┘
                                 │
                 ┌───────────────┼───────────────┐
                 │               │               │
                 ▼               ▼               ▼
        ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
        │  Function   │  │  Function   │  │  Function   │
        │  /login     │  │  /deposits  │  │  /webhooks  │
        └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
               │                │                │
               └────────────────┼────────────────┘
                                │
                                ▼
                        ┌───────────────┐
                        │  PostgreSQL   │
                        │  (RDS/Azure)  │
                        └───────────────┘
```

### Function Breakdown

**API Functions:**
- `POST /auth/login` → Lambda: AuthLogin
- `POST /auth/register` → Lambda: AuthRegister
- `POST /deposits` → Lambda: CreateDeposit
- `POST /withdrawals` → Lambda: CreateWithdrawal
- `GET /balance` → Lambda: GetBalance
- `POST /webhooks/revio` → Lambda: ProcessWebhook

**Background Functions:**
- Reconciliation (scheduled, runs every 15 min)
- Notification sender (triggered by events)

### Pros

✅ **Auto-scaling**
- Scales to zero when idle
- Scales up automatically under load
- No capacity planning needed

✅ **Pay-per-use pricing**
- Only pay for actual execution time
- Potentially very cheap at low volume

✅ **No server management**
- No infrastructure to maintain
- Automatic patching and updates

### Cons

❌ **Cold start latency**
- First request after idle period is slow (1-5 seconds)
- Unacceptable for user-facing API

❌ **Vendor lock-in**
- Hard to migrate between cloud providers
- Proprietary APIs and tooling

❌ **Complexity**
- Distributed tracing required
- Complex local development
- Hard to debug production issues

❌ **Connection limits**
- Database connections are limited
- Each function invocation needs a connection
- Connection pooling is complex

❌ **Not ideal for Django**
- Django not designed for serverless
- Large framework with high cold start time
- Better suited for lightweight frameworks (FastAPI, Flask)

### When to Use
❌ **NOT recommended for this project because:**
- User-facing API needs consistent low latency
- Django is not serverless-optimized
- PostgreSQL connection management is complex
- Higher operational complexity
- Team lacks AWS/Azure serverless expertise

---

## Comparison Matrix

| Criteria | Modular Monolith | Microservices | Event-Driven | Serverless |
|----------|------------------|---------------|--------------|------------|
| **Development Speed** | ⭐⭐⭐⭐⭐ Fast | ⭐⭐ Slow | ⭐⭐⭐⭐ Fast | ⭐⭐⭐ Moderate |
| **Operational Complexity** | ⭐⭐⭐⭐⭐ Low | ⭐ Very High | ⭐⭐⭐ Moderate | ⭐⭐ High |
| **Scalability** | ⭐⭐⭐ Good | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐⭐ Very Good | ⭐⭐⭐⭐⭐ Excellent |
| **Performance** | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐ Good | ⭐⭐⭐⭐ Very Good | ⭐⭐ Fair |
| **Cost (Initial)** | ⭐⭐⭐⭐⭐ Low | ⭐⭐ High | ⭐⭐⭐⭐ Low | ⭐⭐⭐⭐⭐ Very Low |
| **Cost (at Scale)** | ⭐⭐⭐ Moderate | ⭐⭐ High | ⭐⭐⭐⭐ Moderate | ⭐⭐⭐ Low |
| **Data Consistency** | ⭐⭐⭐⭐⭐ Strong | ⭐⭐ Eventual | ⭐⭐⭐⭐ Strong | ⭐⭐⭐ Conditional |
| **Team Size Required** | ⭐⭐⭐⭐⭐ 1-5 | ⭐ 10+ | ⭐⭐⭐⭐ 2-8 | ⭐⭐⭐ 2-10 |
| **Django Fit** | ⭐⭐⭐⭐⭐ Perfect | ⭐⭐⭐ Good | ⭐⭐⭐⭐⭐ Perfect | ⭐⭐ Poor |
| **Testing Ease** | ⭐⭐⭐⭐⭐ Easy | ⭐⭐ Hard | ⭐⭐⭐⭐ Easy | ⭐⭐⭐ Moderate |
| **Debugging Ease** | ⭐⭐⭐⭐⭐ Easy | ⭐⭐ Hard | ⭐⭐⭐ Moderate | ⭐⭐ Hard |
| **Future Flexibility** | ⭐⭐⭐ Moderate | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐⭐ Very Good | ⭐⭐⭐ Moderate |

---

## Recommendation: Modular Monolith with Event-Driven Elements

### Final Architecture

I recommend a **Modular Monolith** as the primary architecture with **Event-Driven patterns** for asynchronous processing. This combines the simplicity of a monolith with the scalability of async task processing.

### Why This Hybrid Approach?

**Rationale:**

1. **Right-sized for the project**
   - Small team, initial user base
   - Need to ship quickly
   - Budget-conscious
   - Aligns with Django's strengths

2. **Balances simplicity and scalability**
   - Simple synchronous API for CRUD operations
   - Async workers for webhooks, notifications, reconciliation
   - Can scale to 100x without major rewrite

3. **Leverages existing tech stack**
   - Django for web framework
   - PostgreSQL for ACID transactions
   - Celery + Redis (already planned) for async tasks
   - No additional infrastructure needed

4. **Clear evolution path**
   - Can extract microservices later if needed
   - Well-defined module boundaries enable future splitting
   - No premature optimization

5. **Team-friendly**
   - Single codebase is easier to understand
   - Simpler onboarding
   - Easier to maintain with small team

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                      Load Balancer / CDN                          │
│                      (Railway + Cloudflare)                       │
└───────────────────────────────┬──────────────────────────────────┘
                                │
                                ▼
        ┌───────────────────────────────────────────────┐
        │        Django Application (Gunicorn)          │
        │                                               │
        │  ┌─────────────────────────────────────────┐ │
        │  │          API Layer (DRF)                │ │
        │  │  - Authentication Middleware            │ │
        │  │  - Rate Limiting                        │ │
        │  │  - Request Validation                   │ │
        │  └──────────────────┬──────────────────────┘ │
        │                     │                         │
        │  ┌──────────────────┴──────────────────────┐ │
        │  │         Service Layer                   │ │
        │  │  (Business Logic)                       │ │
        │  │                                         │ │
        │  │  ┌──────────┐ ┌──────────┐ ┌─────────┐│ │
        │  │  │  Auth    │ │Transaction│ │Webhook │││ │
        │  │  │ Service  │ │  Service  │ │Service │││ │
        │  │  └──────────┘ └──────────┘ └─────────┘│ │
        │  └──────────────────┬──────────────────────┘ │
        │                     │                         │
        │  ┌──────────────────▼──────────────────────┐ │
        │  │         Data Access Layer (ORM)         │ │
        │  └──────────────────┬──────────────────────┘ │
        └─────────────────────┼──────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  PostgreSQL     │  │  Redis          │  │  Celery Worker  │
│                 │  │                 │  │  Pool           │
│  - Users        │  │  - Cache        │  │                 │
│  - Transactions │  │  - Sessions     │  │  ┌───────────┐  │
│  - Balances     │  │  - Rate Limit   │  │  │  Webhook  │  │
│  - Webhooks     │  │  - Task Queue   │  │  │ Processor │  │
│                 │  │                 │  │  └───────────┘  │
│  [ACID]         │  └─────────────────┘  │  ┌───────────┐  │
│  [SERIALIZABLE] │                       │  │Notification│  │
└─────────────────┘                       │  │  Sender   │  │
                                          │  └───────────┘  │
                                          │  ┌───────────┐  │
                                          │  │Reconcile  │  │
                                          │  │  (Cron)   │  │
                                          │  └───────────┘  │
                                          └─────────────────┘

External Services:
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Revio API   │  │   Sentry    │  │  SendGrid   │  │ Prometheus  │
│ (Payments)  │  │  (Errors)   │  │   (Email)   │  │  (Metrics)  │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
```

### Key Architectural Patterns

#### 1. Layered Architecture

**Layers (top to bottom):**
1. **Presentation Layer** (API endpoints)
2. **Business Logic Layer** (Services)
3. **Data Access Layer** (ORM)
4. **Database Layer** (PostgreSQL)

**Rules:**
- Each layer only communicates with the layer directly below it
- No skipping layers (e.g., API cannot directly access database)
- Business logic lives in service layer, not views or models

#### 2. Service Layer Pattern

```python
# apps/transactions/services.py
from decimal import Decimal
from django.db import transaction
from .models import Transaction, Balance

class TransactionService:
    @transaction.atomic
    def create_deposit(self, user_id: int, amount: Decimal) -> Transaction:
        """
        Create a deposit transaction and call Revio API.
        """
        # Business logic here
        txn = Transaction.objects.create(
            user_id=user_id,
            amount=amount,
            type=Transaction.Type.DEPOSIT,
            status=Transaction.Status.PENDING
        )

        # Call Revio API
        revio_response = self.revio_client.create_purchase(amount)
        txn.external_id = revio_response['purchase_id']
        txn.save()

        return txn

    @transaction.atomic(isolation_level='SERIALIZABLE')
    def process_deposit_webhook(self, purchase_id: str) -> None:
        """
        Process webhook and update user balance.
        Uses SERIALIZABLE isolation for strong consistency.
        """
        txn = Transaction.objects.select_for_update().get(
            external_id=purchase_id
        )

        if txn.status == Transaction.Status.COMPLETED:
            # Idempotency: already processed
            return

        # Update balance
        balance = Balance.objects.select_for_update().get(
            user_id=txn.user_id
        )
        balance.amount += txn.amount
        balance.save()

        # Update transaction
        txn.status = Transaction.Status.COMPLETED
        txn.save()
```

#### 3. Repository Pattern (Optional)

For complex queries, abstract database access:

```python
# apps/transactions/repositories.py
class TransactionRepository:
    def get_pending_withdrawals_older_than(self, hours: int):
        """Get all withdrawals stuck in PROCESSING state."""
        cutoff = timezone.now() - timedelta(hours=hours)
        return Transaction.objects.filter(
            type=Transaction.Type.WITHDRAWAL,
            status=Transaction.Status.PROCESSING,
            created_at__lt=cutoff
        )
```

#### 4. Event Publishing Pattern

```python
# apps/core/events.py
from typing import Dict, Any
from redis import Redis
import json

class EventPublisher:
    def __init__(self):
        self.redis = Redis.from_url(settings.REDIS_URL)

    def publish(self, event_type: str, data: Dict[str, Any]) -> None:
        event = {
            'event_type': event_type,
            'timestamp': timezone.now().isoformat(),
            'data': data
        }
        self.redis.publish('events', json.dumps(event))

# Usage in service
publisher = EventPublisher()
publisher.publish('deposit.completed', {
    'user_id': user.id,
    'transaction_id': txn.id,
    'amount': str(txn.amount)
})
```

#### 5. Idempotency Pattern

```python
# apps/webhooks/idempotency.py
from django.core.cache import cache

class IdempotencyChecker:
    def is_processed(self, webhook_id: str) -> bool:
        """Check if webhook already processed."""
        key = f"webhook:{webhook_id}:processed"
        return cache.get(key) is not None

    def mark_processed(self, webhook_id: str) -> None:
        """Mark webhook as processed."""
        key = f"webhook:{webhook_id}:processed"
        # Store for 7 days
        cache.set(key, True, timeout=7*24*60*60)
```

### Scaling Strategy

#### Phase 1: Single Instance (0-1,000 users)
```
1 Web Server + 1 Database + 1 Redis + 1 Celery Worker
Cost: ~$110/month
```

#### Phase 2: Horizontal Scaling (1,000-10,000 users)
```
2-3 Web Servers (load balanced)
1 Database (larger instance)
1 Redis (larger instance)
2-3 Celery Workers
Database read replicas

Cost: ~$250-350/month
```

#### Phase 3: Advanced Scaling (10,000-100,000 users)
```
5-10 Web Servers (auto-scaling)
1 Primary DB + 2 Read Replicas
Redis Cluster (3 nodes)
5-10 Celery Workers
CDN for static assets
Separate webhook processing workers

Cost: ~$600-800/month
```

#### Phase 4: Microservices Migration (100,000+ users)
```
Extract high-load services:
- Webhook Service (Go) - handles 1000s of webhooks/sec
- Notification Service (Go) - sends emails/SMS
- Keep core transaction logic in Django

Use Kafka for event streaming
Use separate databases per service
API Gateway for routing

Cost: ~$1,500-2,000/month
```

### When to Refactor to Microservices

**Triggers for migration:**
- Team grows beyond 15 people
- User base exceeds 100,000 active users
- Webhook processing becomes bottleneck
- Need to use different technology for specific service
- Single deployment becomes too risky

**How to migrate incrementally:**
1. Extract Webhook Service first (highest independent load)
2. Then extract Notification Service
3. Keep Auth and Transaction logic together (core domain)
4. Use API Gateway (Kong) to route between services
5. Migrate database using read replicas and sync

---

## Authentication & Authorization Design

Since auth is part of system architecture, here's the design:

### Authentication: JWT (JSON Web Tokens)

**Why JWT?**
- Stateless (scales horizontally easily)
- Self-contained (no database lookup on every request)
- Industry standard
- Works well with Django REST Framework

**Token Structure:**
```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT"
  },
  "payload": {
    "user_id": 123,
    "email": "user@example.com",
    "exp": 1705334400,
    "iat": 1705248000,
    "jti": "unique-token-id"
  },
  "signature": "..."
}
```

**Token Types:**
1. **Access Token** - Short-lived (15 minutes), used for API calls
2. **Refresh Token** - Long-lived (7 days), used to get new access tokens

**Flow:**
```
1. User → POST /auth/login {email, password}
2. Backend validates credentials
3. Backend → {access_token, refresh_token}
4. User stores tokens (localStorage or httpOnly cookie)
5. User → GET /api/balance {Authorization: Bearer <access_token>}
6. Backend validates token and returns data
7. When access_token expires → POST /auth/refresh {refresh_token}
8. Backend → {new_access_token}
```

**Implementation:**
```python
# settings.py
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ALGORITHM': 'RS256',
    'SIGNING_KEY': PRIVATE_KEY,
    'VERIFYING_KEY': PUBLIC_KEY,
    'AUTH_HEADER_TYPES': ('Bearer',),
}

# views.py
from rest_framework_simplejwt.views import TokenObtainPairView

class LoginView(TokenObtainPairView):
    # Uses username/password to return JWT tokens
    pass
```

### Authorization: Role-Based Access Control (RBAC)

**Roles:**
- `user`: Regular user (can manage own account)
- `admin`: Admin user (can view all accounts, manual reconciliation)

**Permissions:**
```python
# apps/accounts/permissions.py
from rest_framework.permissions import BasePermission

class IsOwnerOrAdmin(BasePermission):
    def has_object_permission(self, request, view, obj):
        # Admin can access any object
        if request.user.is_staff:
            return True
        # Users can only access their own objects
        return obj.user_id == request.user.id

# Usage in views
class BalanceView(APIView):
    permission_classes = [IsAuthenticated, IsOwnerOrAdmin]

    def get(self, request):
        balance = Balance.objects.get(user=request.user)
        return Response({'balance': balance.amount})
```

**Enforcement Points:**
1. **API Gateway/Middleware**: Validate JWT token
2. **View Layer**: Check user permissions
3. **Service Layer**: Validate user owns resource
4. **Database Layer**: Row-level security (optional, PostgreSQL RLS)

---

## Next Steps

### Immediate Actions
1. ✅ Review and approve this architecture
2. Create detailed component diagrams (next deliverable)
3. Design authentication flow diagram
4. Design database schema based on this architecture

### Follow-up Documents
Based on this architecture, I will create:
1. **Component Interaction Diagram** - Detailed view of how components communicate
2. **Data Flow Diagram** - End-to-end data flow for deposit and withdrawal
3. **Deployment Diagram** - Infrastructure and deployment view
4. **API Design Document** - RESTful API specification

---

## References

### Architectural Patterns
- **Modular Monolith**: https://www.kamilgrzybek.com/blog/posts/modular-monolith-primer
- **Service Layer Pattern**: https://martinfowler.com/eaaCatalog/serviceLayer.html
- **Event-Driven Architecture**: https://martinfowler.com/articles/201701-event-driven.html
- **Microservices**: https://microservices.io/patterns/microservices.html

### Django Best Practices
- **Two Scoops of Django**: Best practices for Django development
- **Django Design Patterns**: https://django-design-patterns.readthedocs.io/
- **Celery Best Practices**: https://docs.celeryq.dev/en/stable/userguide/tasks.html#best-practices

### System Design
- **Designing Data-Intensive Applications** by Martin Kleppmann
- **Building Microservices** by Sam Newman
- **System Design Interview** by Alex Xu
