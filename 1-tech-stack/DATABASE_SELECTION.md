# Database Selection

## Overview
This document recommends database technologies for each programming language and framework combination evaluated in `FRAMEWORK_SELECTION.md`. Given the financial transaction requirements (ACID compliance, consistency, no double-spending), this analysis focuses primarily on relational databases.

---

## Database Requirements for Financial Systems

### Critical Requirements
1. **ACID Compliance**: Atomicity, Consistency, Isolation, Durability
2. **Transaction Isolation**: Support for serializable or repeatable read isolation
3. **Row-Level Locking**: Pessimistic locking for concurrent balance updates
4. **Data Integrity**: Foreign keys, constraints, triggers
5. **Durability**: Write-ahead logging, crash recovery
6. **Backup & Recovery**: Point-in-time recovery, automated backups

### Performance Requirements
1. **Concurrent Writes**: Handle multiple users updating balances simultaneously
2. **Read Performance**: Fast balance lookups and transaction history queries
3. **Indexing**: Efficient indexes on user IDs, transaction IDs, timestamps
4. **Connection Pooling**: Efficient connection management

### Operational Requirements
1. **Monitoring**: Query performance metrics, slow query logs
2. **Scaling**: Vertical scaling initially, horizontal read replicas for growth
3. **Maintenance**: Online schema migrations, vacuum/analyze
4. **Cloud Support**: Managed services available on major cloud providers

---

## Database Options Analysis

### Option 1: PostgreSQL

**Version**: PostgreSQL 16+

#### Strengths for Financial Systems
- **ACID Guarantees**: Full ACID compliance with MVCC (Multi-Version Concurrency Control)
- **Isolation Levels**: Supports all standard levels including Serializable
- **Row-Level Locking**: `SELECT FOR UPDATE` for pessimistic locking
- **Data Types**: Native `NUMERIC` type for precise decimal calculations
- **Constraints**: Rich constraint system (CHECK, UNIQUE, FK, exclusion constraints)
- **JSON Support**: JSONB for flexible webhook storage
- **Extensions**: pgcrypto for encryption, pg_stat_statements for monitoring

#### Performance
- Excellent for read-heavy workloads with proper indexing
- Good concurrent write performance with MVCC
- Efficient connection pooling with PgBouncer
- Partial indexes and expression indexes for optimization

#### Operational Excellence
- **Monitoring**: pg_stat_statements, pg_stat_activity
- **Backups**: pg_dump, pg_basebackup, WAL archiving
- **Replication**: Streaming replication for read replicas
- **Managed Services**: AWS RDS/Aurora, Google Cloud SQL, Azure Database, DigitalOcean, Supabase
- **Cloud-Native**: Crunchy Data, Aiven, Timescale Cloud

#### Ecosystem Support
- Excellent support across all languages/frameworks
- Mature ORMs (Django ORM, Entity Framework, Hibernate, GORM)
- Large community and extensive documentation

**Score: 5/5** - Best choice for financial systems

---

### Option 2: MySQL

**Version**: MySQL 8.0+

#### Strengths for Financial Systems
- **ACID Compliance**: Full ACID with InnoDB storage engine
- **Isolation Levels**: Supports serializable transactions
- **Row-Level Locking**: `SELECT FOR UPDATE` support
- **Decimal Type**: `DECIMAL` type for monetary values
- **Foreign Keys**: InnoDB supports referential integrity
- **JSON Support**: Native JSON type (less feature-rich than PostgreSQL)

#### Performance
- Very fast for read-heavy workloads
- Good write performance with InnoDB
- Excellent replication for horizontal scaling
- Less sophisticated query optimizer than PostgreSQL

#### Operational Excellence
- **Monitoring**: Performance Schema, sys schema
- **Backups**: mysqldump, binary log backups
- **Replication**: Primary-replica replication, group replication
- **Managed Services**: AWS RDS/Aurora, Google Cloud SQL, Azure Database, PlanetScale
- **Compatibility**: MariaDB as alternative fork

#### Ecosystem Support
- Excellent support across all languages
- Mature ORMs and drivers
- Very large community

**Score: 4/5** - Solid choice, but PostgreSQL edges it out for features

---

### Option 3: Microsoft SQL Server

**Version**: SQL Server 2022

#### Strengths for Financial Systems
- **ACID Compliance**: Full ACID guarantees
- **Isolation Levels**: All standard levels including Snapshot Isolation
- **Row-Level Locking**: Comprehensive locking mechanisms
- **Decimal Type**: `DECIMAL` and `MONEY` types
- **Advanced Features**: Computed columns, indexed views, full-text search
- **Integration**: Excellent .NET/C# integration

#### Performance
- Excellent performance with proper tuning
- Advanced query optimizer
- In-memory OLTP for high-performance scenarios
- Columnstore indexes for analytics

#### Operational Excellence
- **Monitoring**: Dynamic Management Views, Query Store
- **Backups**: Full, differential, transaction log backups
- **High Availability**: Always On Availability Groups
- **Managed Services**: Azure SQL Database, AWS RDS for SQL Server
- **Tooling**: SQL Server Management Studio, Azure Data Studio

#### Ecosystem Support
- Best-in-class for C#/.NET applications
- Entity Framework has most features with SQL Server
- Good support for other languages (less idiomatic)

**Score: 4.5/5** - Excellent for .NET, but less popular for other stacks

---

### Option 4: CockroachDB

**Version**: CockroachDB 23+

#### Strengths for Financial Systems
- **Distributed SQL**: Horizontally scalable from day one
- **ACID Compliance**: Serializable isolation by default
- **Consistency**: Strong consistency across nodes
- **PostgreSQL Compatible**: Uses PostgreSQL wire protocol
- **Geo-Distribution**: Built-in multi-region support
- **Resilience**: Automatic failover and recovery

#### Performance
- Good for distributed workloads
- Higher latency than single-node PostgreSQL (due to consensus)
- Excellent for global distribution
- Automatic load balancing

#### Operational Excellence
- **Monitoring**: Built-in admin UI with metrics
- **Backups**: Distributed backups and restores
- **Scaling**: Automatic rebalancing when adding nodes
- **Managed Service**: CockroachDB Serverless/Dedicated
- **Cloud**: Runs on Kubernetes, any cloud

#### Ecosystem Support
- PostgreSQL-compatible (most PostgreSQL tools work)
- Some PostgreSQL features not supported
- Growing community

**Score: 4/5** - Overkill for initial scale, great for global expansion

---

### Option 5: Amazon Aurora (PostgreSQL/MySQL)

**AWS-Specific Option**

#### Strengths for Financial Systems
- **ACID Compliance**: Full ACID (PostgreSQL or MySQL compatible)
- **Performance**: Up to 5x faster than standard PostgreSQL/MySQL
- **High Availability**: 6-way replication across 3 AZs
- **Auto-Scaling**: Storage scales automatically
- **Backups**: Continuous backup to S3
- **Serverless**: Aurora Serverless v2 for variable workloads

#### Performance
- Excellent read performance with up to 15 read replicas
- Fast recovery from crashes
- Storage-level replication reduces latency
- Better write performance than RDS

#### Operational Excellence
- **Monitoring**: CloudWatch integration, Performance Insights
- **Backups**: Automated with 35-day retention
- **Scaling**: Read replicas, Aurora Serverless
- **Managed**: Fully managed by AWS
- **Global**: Aurora Global Database for multi-region

#### Ecosystem Support
- Compatible with PostgreSQL or MySQL drivers/ORMs
- Works with all frameworks that support PostgreSQL/MySQL

**Score: 4.5/5** - Excellent if committed to AWS

---

## Language-Specific Recommendations

### 1. Python (Django) → **PostgreSQL 16**

**Rationale:**
- Django ORM was originally built for PostgreSQL
- Excellent `django.contrib.postgres` module with PostgreSQL-specific features
- Native support for `JSONField`, `ArrayField`, full-text search
- `psycopg3` driver is mature and performant
- Best documentation and community support

**Configuration Example:**
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'octoco_db',
        'USER': 'octoco_user',
        'PASSWORD': os.environ['DB_PASSWORD'],
        'HOST': 'localhost',
        'PORT': '5432',
        'ATOMIC_REQUESTS': True,  # Wrap each request in a transaction
        'CONN_MAX_AGE': 600,  # Connection pooling
        'OPTIONS': {
            'isolation_level': psycopg2.extensions.ISOLATION_LEVEL_SERIALIZABLE,
        }
    }
}
```

**Alternative:** MySQL 8.0+ (if team has MySQL expertise)

---

### 2. Node.js/TypeScript (NestJS) → **PostgreSQL 16**

**Rationale:**
- Best TypeORM support (TypeORM was designed with PostgreSQL in mind)
- Prisma has excellent PostgreSQL support with type generation
- `pg` driver is battle-tested
- JSON/JSONB support perfect for JavaScript/TypeScript
- Modern features like generated columns, exclusion constraints

**Configuration Example (Prisma):**
```typescript
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  balance   Decimal  @db.Decimal(19, 4)
  createdAt DateTime @default(now())

  transactions Transaction[]

  @@index([email])
}
```

**Alternative:** MySQL 8.0+ or CockroachDB (for PostgreSQL compatibility with scale)

---

### 3. Go (Chi + GORM) → **PostgreSQL 16**

**Rationale:**
- `pgx` driver is one of the best PostgreSQL drivers in any language
- Excellent performance and low-level control
- GORM has solid PostgreSQL support
- `database/sql` standard library works perfectly
- PostgreSQL's explicit error handling matches Go's philosophy

**Configuration Example:**
```go
import (
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

dsn := "host=localhost user=octoco password=secret dbname=octoco_db port=5432 sslmode=require"
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
    PrepareStmt: true,
    TransactionTimeout: 5 * time.Second,
})

sqlDB, err := db.DB()
sqlDB.SetMaxIdleConns(10)
sqlDB.SetMaxOpenConns(100)
sqlDB.SetConnMaxLifetime(time.Hour)
```

**Alternative:** MySQL 8.0+ or CockroachDB (excellent Go support)

---

### 4. Java (Spring Boot) → **PostgreSQL 16**

**Rationale:**
- Hibernate has excellent PostgreSQL support
- Spring Data JPA works seamlessly
- `postgresql` JDBC driver is mature and reliable
- Good support for advanced PostgreSQL features
- Extensive Spring Boot autoconfiguration

**Configuration Example (application.yml):**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/octoco_db
    username: octoco_user
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        default_schema: public
        jdbc:
          batch_size: 20
        order_inserts: true
        order_updates: true
```

**Alternative:** MySQL 8.0+ or SQL Server 2022 (if enterprise standard)

---

### 5. C# (ASP.NET Core) → **PostgreSQL 16** or **SQL Server 2022**

#### Option A: PostgreSQL 16 (Recommended for New Projects)

**Rationale:**
- Npgsql is an excellent PostgreSQL driver for .NET
- Entity Framework Core has first-class PostgreSQL support
- Cost-effective (open source)
- Cross-platform compatibility
- Good cloud options on all providers

**Configuration Example:**
```csharp
services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(
        Configuration.GetConnectionString("DefaultConnection"),
        npgsqlOptions => {
            npgsqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorCodesToAdd: null);
            npgsqlOptions.MigrationsHistoryTable("__EFMigrationsHistory", "public");
        })
    .UseSnakeCaseNamingConvention()
    .EnableSensitiveDataLogging(isDevelopment)
);

// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=octoco_db;Username=octoco_user;Password=secret"
  }
}
```

#### Option B: SQL Server 2022 (Best .NET Integration)

**Rationale:**
- Entity Framework Core was built for SQL Server first
- Most features and best performance with EF Core
- Excellent tooling (SSMS, Azure Data Studio)
- Built-in decimal type support
- Best Azure integration

**Configuration Example:**
```csharp
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        Configuration.GetConnectionString("DefaultConnection"),
        sqlServerOptions => {
            sqlServerOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorNumbersToAdd: null);
            sqlServerOptions.CommandTimeout(30);
        })
);

// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=octoco_db;User Id=octoco_user;Password=secret;TrustServerCertificate=true"
  }
}
```

**Recommendation:** PostgreSQL 16 for cost and flexibility, SQL Server 2022 if deep Azure integration is required.

---

## NoSQL Considerations

### When NoSQL Makes Sense
- **Audit Logs**: Use MongoDB or Elasticsearch for immutable audit trails
- **Webhook Logs**: Store raw webhook payloads in document store
- **Caching**: Use Redis for session state, rate limiting
- **Time-Series**: Use TimescaleDB (PostgreSQL extension) for metrics

### When NoSQL Does NOT Make Sense
- **User Balances**: NEVER use NoSQL for financial balances (lacks ACID)
- **Transaction Records**: Must use relational database for consistency
- **Account Data**: Requires strong consistency and relationships

**Pattern**: Use PostgreSQL for transactional data, supplement with NoSQL for specific use cases.

---

## Database Comparison Summary

| Database | ACID | Isolation | Decimal | JSON | Scalability | Ops Maturity | Cost | Best For |
|----------|------|-----------|---------|------|-------------|--------------|------|----------|
| **PostgreSQL** | ✅ | Excellent | ✅ | Excellent | Good | Excellent | Low | Most projects |
| **MySQL** | ✅ | Good | ✅ | Good | Excellent | Excellent | Low | Read-heavy workloads |
| **SQL Server** | ✅ | Excellent | ✅ | Good | Good | Excellent | High | .NET projects |
| **CockroachDB** | ✅ | Excellent | ✅ | Good | Excellent | Good | Medium | Global scale |
| **Aurora** | ✅ | Excellent | ✅ | Good/Excellent | Excellent | Excellent | Medium | AWS-committed |

---

## Recommended Database: **PostgreSQL 16**

### Why PostgreSQL for Octoco Quiz Backend?

#### 1. Financial Data Safety
- **NUMERIC Type**: Arbitrary precision for monetary values (no rounding errors)
- **SERIALIZABLE Isolation**: Strongest consistency guarantees
- **Row-Level Locking**: `SELECT FOR UPDATE` prevents double-spending
- **Constraints**: CHECK constraints for balance >= 0

#### 2. Feature Richness
- **JSONB**: Store webhook payloads efficiently with indexing
- **Generated Columns**: Computed values stored and indexed
- **Exclusion Constraints**: Prevent overlapping transactions
- **Foreign Keys**: Referential integrity for user-transaction relationships
- **Triggers**: Audit logging at database level

#### 3. Performance
- **MVCC**: Readers don't block writers
- **Partial Indexes**: Index only active records
- **Connection Pooling**: PgBouncer for efficient connection management
- **Read Replicas**: Scale reads horizontally

#### 4. Operational Excellence
- **Monitoring**: pg_stat_statements, pg_stat_activity
- **Backups**: WAL archiving, point-in-time recovery
- **Migrations**: Flyway, Liquibase, or framework-native tools
- **Extensions**: pgcrypto, pg_trgm, pg_stat_statements

#### 5. Cloud Agnostic
- **AWS**: RDS PostgreSQL, Aurora PostgreSQL
- **Azure**: Azure Database for PostgreSQL
- **GCP**: Cloud SQL for PostgreSQL
- **DigitalOcean**: Managed PostgreSQL
- **Self-Hosted**: Docker, Kubernetes, bare metal

#### 6. Ecosystem Support
- Excellent support in all frameworks:
  - Django ORM (best-in-class)
  - Entity Framework Core
  - Hibernate
  - TypeORM/Prisma
  - GORM
- Large community and extensive documentation
- Decades of production battle-testing

---

## Schema Design Preview

### Users Table
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    balance NUMERIC(19, 4) NOT NULL DEFAULT 0.00 CHECK (balance >= 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT positive_balance CHECK (balance >= 0)
);

CREATE INDEX idx_users_email ON users(email);
```

### Transactions Table
```sql
CREATE TYPE transaction_type AS ENUM ('deposit', 'withdrawal');
CREATE TYPE transaction_status AS ENUM ('pending', 'completed', 'failed', 'cancelled');

CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    type transaction_type NOT NULL,
    amount NUMERIC(19, 4) NOT NULL CHECK (amount > 0),
    status transaction_status NOT NULL DEFAULT 'pending',
    revio_transaction_id VARCHAR(255) UNIQUE,
    metadata JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ,

    CONSTRAINT positive_amount CHECK (amount > 0)
);

CREATE INDEX idx_transactions_user_id ON transactions(user_id);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_created_at ON transactions(created_at DESC);
CREATE INDEX idx_transactions_revio_id ON transactions(revio_transaction_id);
```

### Webhook Events Table
```sql
CREATE TABLE webhook_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id VARCHAR(255) UNIQUE NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    signature VARCHAR(500) NOT NULL,
    processed BOOLEAN NOT NULL DEFAULT FALSE,
    processed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT unique_event_id UNIQUE (event_id)
);

CREATE INDEX idx_webhook_events_event_id ON webhook_events(event_id);
CREATE INDEX idx_webhook_events_processed ON webhook_events(processed) WHERE NOT processed;
CREATE INDEX idx_webhook_events_created_at ON webhook_events(created_at DESC);
```

---

## Deployment Recommendations

### Development Environment
- **Local**: Docker Compose with PostgreSQL 16 container
- **Seed Data**: Use migrations to create test users and transactions
- **Tools**: pgAdmin, DBeaver, or Azure Data Studio

### Staging Environment
- **Managed Service**: AWS RDS PostgreSQL or equivalent
- **Instance Size**: db.t3.medium (2 vCPU, 4 GB RAM)
- **Storage**: 100 GB SSD with autoscaling
- **Backups**: Automated daily backups with 7-day retention
- **Encryption**: At rest and in transit

### Production Environment
- **Managed Service**: AWS RDS PostgreSQL (or Aurora PostgreSQL for scale)
- **Instance Size**: db.r6g.xlarge (4 vCPU, 32 GB RAM) - adjust based on load
- **Storage**: 500 GB SSD with autoscaling
- **Backups**: Automated daily backups with 30-day retention
- **High Availability**: Multi-AZ deployment
- **Read Replicas**: 1-2 read replicas for scaling
- **Monitoring**: CloudWatch, pgBadger, pganalyze
- **Connection Pooling**: PgBouncer or RDS Proxy

---

## Cost Estimates (Monthly)

### PostgreSQL Options

| Option | Environment | Cost |
|--------|-------------|------|
| **Local Docker** | Development | $0 |
| **DigitalOcean Managed** | Staging | ~$15 (Basic) |
| **AWS RDS (t3.medium)** | Staging | ~$70 |
| **AWS RDS (r6g.xlarge)** | Production | ~$350 |
| **Azure Database** | Production | ~$300-400 |
| **Aurora PostgreSQL** | Production (scalable) | ~$500+ |
| **CockroachDB Serverless** | Production | Pay-per-use (~$0-200) |

---

## Next Steps

With database selected (PostgreSQL 16), we can now proceed to:

1. **Complete Section 1**: Hosting/Infrastructure selection
2. **Section 2**: System Architecture Design
3. **Section 3**: Detailed Database Schema Design with migrations
4. **Section 4**: API and flow design with database interactions
