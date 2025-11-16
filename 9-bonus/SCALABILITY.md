# Scalability Design - 100x Growth Strategy

## Overview

This document describes how to scale the piggy bank backend system from initial capacity (1,000 users, 10K transactions/day) to 100x usage (100,000 users, 1M transactions/day). It identifies bottlenecks and proposes phased scaling strategies.

---

## Current Architecture Baseline (1x Scale)

**Capacity**:
- 1,000 active users
- 10,000 transactions/day (~7 TPS average, ~30 TPS peak)
- 10GB database size

**Infrastructure**:
- 2 web containers (Django + Gunicorn)
- 2 Celery worker containers
- 1 PostgreSQL instance (8GB RAM)
- 1 Redis instance (512MB RAM)

**Cost**: ~$110/month (Railway)

---

## Bottleneck Identification

### 1. Application Tier
- **Bottleneck**: Limited Gunicorn workers (8 workers → ~80 TPS max)
- **Symptom at 100x**: Request queue buildup, timeouts
- **Solution**: Horizontal scaling (add more web containers)

### 2. Database Tier
- **Write Bottleneck**: SERIALIZABLE isolation + SELECT FOR UPDATE creates lock contention
- **Read Bottleneck**: Transaction history queries slow with millions of rows
- **Connection Bottleneck**: 100-connection limit exhausted
- **Solution**: Read replicas + connection pooling + caching + partitioning

### 3. Caching Tier
- **Bottleneck**: 512MB Redis → frequent cache evictions
- **Symptom at 100x**: High cache miss rate, increased DB load
- **Solution**: Increase Redis memory, consider Redis cluster

---

## Scaling Strategy: 3-Phase Approach

### Phase 1: Optimize (1x → 10x)

**Goal**: 10,000 users, 100K transactions/day

**Changes**:
- **Application**: 2 → 6 web containers (horizontal scaling)
- **Database**: Vertical scaling (4 vCPU, 16GB RAM)
- **Caching**: Add Redis caching for user balances (5-min TTL)
- **Optimization**: Add covering indexes for transaction queries

**Cost**: ~$300/month

**Result**: 10x throughput capacity

---

### Phase 2: Read/Write Separation (10x → 50x)

**Goal**: 50,000 users, 500K transactions/day

**Changes**:
- **Database**: 1 primary + 2 read replicas
- **Query Routing**: Route reads to replicas, writes to primary
- **Connection Pooling**: Add PgBouncer (1,000 connections → 50 DB connections)

**Tradeoff**:
- ✅ 80% of queries offloaded to replicas
- ❌ Slight replication lag (< 100ms)

**Cost**: ~$800/month

**Result**: 5x additional capacity (50x total)

---

### Phase 3: Sharding & Auto-Scaling (50x → 100x)

**Goal**: 100,000 users, 1M transactions/day

**Changes**:
- **Application**: Auto-scale to 20 web containers based on CPU/memory
- **Database Partitioning**: Partition `transactions` table by date (monthly)
- **Redis Cluster**: 3 master + 3 replica nodes (12GB total)
- **CDN**: Cloudflare for static assets and API response caching

**Cost**: ~$2,500/month

**Result**: 2x additional capacity (100x total)

---

## Horizontal vs Vertical Scaling

| Strategy | When | Pros | Cons |
|----------|------|------|------|
| **Vertical Scaling** | Phase 1 | Simple, no code changes | Hardware limits, single point of failure |
| **Horizontal Scaling** | Phase 2-3 | Unlimited scaling, high availability | Requires load balancing, stateless design |

**Recommendation**: Start vertical, add horizontal when needed

---

## Caching Strategy

### What to Cache

| Data | TTL | Invalidation |
|------|-----|--------------|
| User balance | 5 minutes | On balance update |
| Transaction count | 5 minutes | On new transaction |
| Transaction list (page 1) | 1 minute | On new transaction |
| User session | 7 days | On logout |

### Cache Benefits
- **Balance checks**: 90% cache hit rate → 90% fewer DB queries
- **Transaction counts**: Pagination metadata pre-computed
- **API response time**: 200ms → 50ms (cached responses)

---

## Database Scaling Approach

### Phase 1: Indexing
- Add covering indexes for common queries
- Use partial indexes where appropriate (e.g., `status IN ('pending', 'processing')`)

### Phase 2: Read Replicas
- Asynchronous replication (< 100ms lag)
- 80% of queries routed to replicas
- Primary handles writes only

### Phase 3: Partitioning
- Partition `transactions` by month (range partitioning)
- Queries with date filters only scan relevant partitions
- Easy archival (detach old partitions)

### Future (Beyond 100x): User-Based Sharding
- Shard by `user_id` hash
- Each shard handles subset of users
- Linear scaling (add more shards as needed)

---

## Future Considerations (Beyond 100x)

### Microservices Migration (200,000+ users)

Split into services:
1. **User Service**: Authentication, user management
2. **Transaction Service**: Deposits, withdrawals, balances
3. **Webhook Service**: Revio webhook processing
4. **Reporting Service**: Transaction history, analytics

**Benefits**: Independent scaling, fault isolation
**Tradeoffs**: Increased complexity, distributed transactions

---

### Managed Cloud Services

Migrate from Railway to AWS/GCP:
- **RDS PostgreSQL**: Managed replicas, automated backups
- **ElastiCache Redis**: Managed cluster, auto-failover
- **ECS/Kubernetes**: Container orchestration, auto-scaling

**Benefits**: Better tooling, high availability
**Tradeoffs**: 2-3x higher cost, vendor lock-in

---

## Cost Projection

| Scale | Users | TPS (peak) | Monthly Cost |
|-------|-------|------------|--------------|
| 1x (Current) | 1K | 30 | $110 |
| 10x (Phase 1) | 10K | 300 | $300 |
| 50x (Phase 2) | 50K | 1,500 | $800 |
| 100x (Phase 3) | 100K | 3,000 | $2,500 |
| 200x+ (Microservices) | 200K+ | 6,000+ | $5,000+ |

**Cost per user** improves with scale: $0.11 → $0.025 per user

---

## Monitoring & Capacity Planning

### Key Metrics to Track

**Application**:
- Response time (p95 < 200ms)
- Throughput (requests/second)
- Error rate (< 1%)

**Database**:
- Connection count (< 80% of max)
- Replication lag (< 100ms)
- Slow queries (> 100ms)

**Cache**:
- Hit rate (> 80%)
- Memory usage (< 80%)

### Alerting Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| API p95 response time | > 200ms | > 500ms |
| Database CPU | > 70% | > 90% |
| Cache hit rate | < 70% | < 50% |
| Error rate | > 1% | > 5% |

---

## Summary

**Scaling Path**:
1. **Phase 1**: Vertical scaling + caching (1x → 10x)
2. **Phase 2**: Read replicas + connection pooling (10x → 50x)
3. **Phase 3**: Partitioning + auto-scaling (50x → 100x)

**Key Strategies**:
- Start simple (vertical scaling)
- Add complexity only when needed (horizontal scaling, sharding)
- Cache aggressively (90% cache hit rate)
- Monitor everything (proactive capacity planning)

**Cost Efficiency**: Economies of scale reduce cost per user by 4-5x at 100x scale

**Architecture Evolution**: Modular monolith → Read replicas → Partitioning → Microservices (if needed)
