# ADR-003: Hybrid Database Strategy with Separate Analytics Database

## Status

**Accepted** (2026-08-27)
**Updated** (2026-08-27) - Revised to database-per-domain with strategic grouping

## Context

The Vaullet walleting system uses a polyrepo microservices architecture with 12 services. Each service needs persistent storage for its domain data. We must decide how to structure databases across services.

### Key Requirements

1. **Data isolation**: Critical financial data (ledger) must be completely isolated
2. **Scalability**: High-volume services (transactions) need independent scaling
3. **Operational simplicity**: Minimize overhead for small team
4. **Cost efficiency**: Balance isolation with infrastructure costs
5. **Reporting capability**: Support complex analytics across multiple domains
6. **Technology flexibility**: Allow different databases for different needs

### Reporting Challenge

Financial systems require complex reporting that spans multiple services:
- Transaction reports with user info, bonuses, merchant data
- Fraud analytics across transactions, user behavior, patterns
- Business metrics: daily volume, bonus payouts, refund rates
- User lifetime value calculations

**Key insight**: Even with a shared database, cross-service reporting should not query operational databases directly. This would violate service boundaries and impact operational performance.

### The Trade-off

**Full separation** (database per service):
- ✅ Maximum isolation, independent scaling
- ❌ 12 databases to manage, higher cost, operational overhead

**Shared database** (all services):
- ✅ Simple, low cost, easy JOINs
- ❌ Tight coupling, schema coordination nightmare, no isolation

**Hybrid** (strategic separation):
- ✅ Isolation where critical, simplicity where acceptable
- ⚠️ Need clear criteria for what gets separated

## Decision

We will use a **hybrid database strategy** with clear separation criteria, plus a **dedicated analytics database** for reporting.

### Operational Databases (4 Databases)

#### 1. Vaullet Database (Isolated) 🔐

**Service**: Vaullet (Ledger Service) only

**Technology**: PostgreSQL with append-only constraints

**Schema**:
```sql
vaullet_db
├─ ledger_entries (immutable, append-only)
│   ├─ entry_id (PK)
│   ├─ transaction_id (FK)
│   ├─ user_id (indexed)
│   ├─ amount (NOT NULL)
│   ├─ type (DEBIT/CREDIT)
│   ├─ category (TRANSACTION/REFUND/BONUS/etc)
│   ├─ created_at (immutable)
│   └─ metadata (JSONB)
│
├─ balances (materialized view)
│   ├─ user_id (PK)
│   ├─ balance (computed from ledger_entries)
│   └─ last_updated_at
│
└─ audit_log (compliance)
    ├─ audit_id (PK)
    ├─ entry_id (FK)
    ├─ action (INSERT only)
    └─ timestamp
```

**Why isolated**:
- ✅ Source of truth for all money movements
- ✅ Immutable data (append-only) requires strict controls
- ✅ Compliance and audit requirements
- ✅ Different backup/retention policies (7+ years)
- ✅ No risk of accidental modification by other services

**Access control**: ONLY Vaullet Service can write. Other services call Vaullet REST API.

**Backup strategy**: Continuous backup, point-in-time recovery, 7-year retention

---

#### 2. Transactions Database (Isolated)

**Service**: Transaction Service only

**Technology**: PostgreSQL

**Schema**:
```sql
transactions_db
├─ transactions (mutable, workflow state)
│   ├─ transaction_id (PK)
│   ├─ user_id (indexed)
│   ├─ amount (NOT NULL)
│   ├─ currency
│   ├─ type (WITHDRAWAL/DEPOSIT/TRANSFER)
│   ├─ status (PENDING/APPROVED/COMPLETED/FAILED)
│   ├─ merchant_id
│   ├─ merchant_category
│   ├─ fraud_analysis_id
│   ├─ created_at
│   ├─ updated_at
│   └─ metadata (JSONB)
│
├─ refunds
│   ├─ refund_id (PK)
│   ├─ original_transaction_id (FK)
│   ├─ refund_amount
│   ├─ reason
│   ├─ status
│   └─ created_at
│
└─ transaction_links (bonuses, refunds, etc)
    ├─ transaction_id (FK)
    ├─ linked_entity_type (BONUS/REFUND)
    ├─ linked_entity_id
    └─ created_at
```

**Why isolated**:
- ✅ High transaction volume (>1000 TPS target)
- ✅ Needs independent scaling (read replicas, partitioning)
- ✅ Mutable state (status updates) separate from immutable ledger
- ✅ Complex indexes for different query patterns

**Access control**: ONLY Transaction Service writes

---

#### 3. Fraud Detection Database (Isolated)

**Service**: Fraud Detection Service, Risk Management Service

**Technology**: PostgreSQL + Elasticsearch

**Schema**:
```sql
fraud_db (PostgreSQL)
├─ fraud_rules
│   ├─ rule_id (PK)
│   ├─ rule_name
│   ├─ rule_type (VELOCITY/AMOUNT/LOCATION)
│   ├─ threshold_value
│   ├─ is_active
│   └─ created_at
│
├─ fraud_cases
│   ├─ case_id (PK)
│   ├─ user_id (indexed)
│   ├─ transaction_id (indexed)
│   ├─ risk_score
│   ├─ status (OPEN/INVESTIGATING/CLOSED)
│   ├─ assigned_to
│   ├─ resolution
│   └─ created_at
│
└─ fraud_analysis_history
    ├─ analysis_id (PK)
    ├─ transaction_id
    ├─ risk_score
    ├─ ml_model_score
    ├─ decision (APPROVE/REVIEW/BLOCK)
    ├─ rules_triggered (JSONB array)
    └─ created_at

fraud_search (Elasticsearch)
└─ fraud_patterns (full-text search, pattern matching)
    ├─ pattern_id
    ├─ pattern_type
    ├─ description (searchable)
    ├─ affected_transactions
    └─ detected_at
```

**Why isolated**:
- ✅ Different technology needs (Elasticsearch for pattern detection)
- ✅ Sensitive data (fraud cases, investigations)
- ✅ Complex ML workloads (separate from transactional DBs)
- ✅ Independent scaling for fraud detection spike

**Access control**: Fraud Detection Service and Risk Management Service share this DB

---

#### 4. Support Services Database (Shared)

**Services**: Limits, Bonus, Subscription, Notification, Audit (share this DB)

**Technology**: PostgreSQL with schema-per-service

**Schema**:
```sql
support_db

-- Limits Service schema
limits_schema
├─ user_limits
│   ├─ user_id (PK)
│   ├─ daily_limit
│   ├─ monthly_limit
│   ├─ daily_used (updated from events)
│   ├─ monthly_used
│   └─ last_reset_date
│
└─ limit_configurations
    ├─ config_id (PK)
    ├─ user_type (BASIC/PREMIUM)
    ├─ default_daily_limit
    └─ default_monthly_limit

-- Bonus Service schema
bonus_schema
├─ bonuses
│   ├─ bonus_id (PK)
│   ├─ user_id (indexed)
│   ├─ transaction_id (indexed)
│   ├─ bonus_amount
│   ├─ bonus_type (CASHBACK/PROMOTION/LOYALTY)
│   ├─ status (ACTIVE/REVOKED/EXPIRED)
│   ├─ expires_at
│   └─ created_at
│
└─ bonus_campaigns
    ├─ campaign_id (PK)
    ├─ campaign_name
    ├─ bonus_percentage
    ├─ merchant_category_filter
    ├─ start_date
    └─ end_date

-- Subscription Service schema
subscription_schema
├─ subscriptions
│   ├─ subscription_id (PK)
│   ├─ user_id (indexed)
│   ├─ plan_id
│   ├─ status (ACTIVE/SUSPENDED/CANCELLED)
│   ├─ next_billing_date
│   ├─ retry_count
│   └─ created_at
│
└─ subscription_plans
    ├─ plan_id (PK)
    ├─ plan_name
    ├─ amount
    └─ billing_cycle

-- Notification Service schema
notification_schema
├─ notification_templates
│   ├─ template_id (PK)
│   ├─ template_name
│   ├─ template_body
│   └─ channel (EMAIL/SMS/PUSH)
│
└─ notification_log
    ├─ notification_id (PK)
    ├─ user_id (indexed)
    ├─ template_id
    ├─ sent_at
    └─ status (SENT/FAILED)

-- Audit Service schema
audit_schema
└─ audit_events
    ├─ event_id (PK)
    ├─ event_type (indexed)
    ├─ user_id (indexed)
    ├─ service_name (indexed)
    ├─ event_data (JSONB)
    └─ created_at (partitioned by month)
```

**Why shared**:
- ✅ Lower transaction volumes (combined <100 TPS)
- ✅ Closely related domains (user preferences, configurations)
- ✅ No special technology requirements (standard PostgreSQL)
- ✅ Simpler operations (one DB to backup/monitor)
- ✅ Cost-effective for smaller services
- ✅ Can be split later if volume grows

**Access control**: Each service has its own PostgreSQL schema with dedicated credentials

**Schema isolation**:
```sql
-- Limits Service user can only access limits_schema
GRANT USAGE ON SCHEMA limits_schema TO limits_service_user;
GRANT ALL ON ALL TABLES IN SCHEMA limits_schema TO limits_service_user;

-- Bonus Service user can only access bonus_schema
GRANT USAGE ON SCHEMA bonus_schema TO bonus_service_user;
GRANT ALL ON ALL TABLES IN SCHEMA bonus_schema TO bonus_service_user;
```

**Future evolution**: If any service outgrows the shared DB, it can be split into its own database.

---

#### 5. Redis Cluster (Shared Cache)

**Services**: Auth, Transaction, Limits, Fraud Detection (shared)

**Technology**: Redis Cluster (3 masters, 3 replicas)

**Namespaces**:
```
redis_cluster
├─ session:* (Auth Service - user sessions)
├─ lock:balance:* (Transaction Service - distributed locks)
├─ limits:counter:* (Limits Service - real-time counters)
└─ fraud:cache:* (Fraud Detection - recent transaction cache)
```

**Why shared**: Redis is designed for multi-tenant use with namespacing

---

### Analytics Database (Separate Read-Optimized Store)

#### 6. Reporting Database

**Purpose**: Complex analytics, business intelligence, reporting

**Technology**: PostgreSQL (Phase 1) → ClickHouse/TimescaleDB (Phase 2)

**Schema**:
```sql
reporting_db

-- Denormalized transaction analytics
├─ transaction_analytics
│   ├─ transaction_id (PK)
│   ├─ user_id (indexed)
│   ├─ amount
│   ├─ currency
│   ├─ merchant_id
│   ├─ merchant_category
│   ├─ transaction_type
│   ├─ transaction_status
│   ├─ bonus_amount (joined from bonus events)
│   ├─ fraud_risk_score (joined from fraud events)
│   ├─ was_refunded (boolean)
│   ├─ refund_amount
│   └─ created_at (partitioned)
│
-- Pre-aggregated daily metrics
├─ daily_metrics
│   ├─ date (PK)
│   ├─ total_transactions
│   ├─ total_volume
│   ├─ total_refunds
│   ├─ total_bonuses_paid
│   ├─ fraud_detected_count
│   ├─ fraud_blocked_count
│   └─ average_transaction_amount
│
-- User summaries
├─ user_summaries
│   ├─ user_id (PK)
│   ├─ total_transactions
│   ├─ total_spent
│   ├─ total_bonuses_earned
│   ├─ total_refunds_received
│   ├─ fraud_incidents_count
│   ├─ first_transaction_date
│   ├─ last_transaction_date
│   └─ lifetime_value (computed)
│
-- Merchant analytics
└─ merchant_analytics
    ├─ merchant_id (PK)
    ├─ merchant_category
    ├─ total_transactions
    ├─ total_volume
    ├─ fraud_rate
    └─ last_updated_at
```

**Data flow**:
```
Operational Services → Kafka Events → Reporting ETL Service → Reporting DB
```

**ETL Service**:
```java
@Service
public class ReportingETLService {

    @KafkaListener(topics = "transaction.created")
    public void handleTransactionCreated(TransactionCreatedEvent event) {
        reportingRepository.save(TransactionAnalytics.builder()
            .transactionId(event.getPayload().getTransactionId())
            .userId(event.getPayload().getUserId())
            .amount(event.getPayload().getAmount())
            .merchantCategory(event.getPayload().getMerchantCategory())
            .createdAt(event.getTimestamp())
            .build());
    }

    @KafkaListener(topics = "bonus.applied")
    public void handleBonusApplied(BonusAppliedEvent event) {
        reportingRepository.updateBonusAmount(
            event.getPayload().getTransactionId(),
            event.getPayload().getBonusAmount()
        );
    }

    @KafkaListener(topics = "fraud.analysis.completed")
    public void handleFraudAnalysis(FraudAnalysisCompletedEvent event) {
        reportingRepository.updateFraudScore(
            event.getPayload().getTransactionId(),
            event.getPayload().getRiskScore()
        );
    }

    @KafkaListener(topics = "refund.completed")
    public void handleRefundCompleted(RefundCompletedEvent event) {
        reportingRepository.markAsRefunded(
            event.getPayload().getOriginalTransactionId(),
            event.getPayload().getRefundAmount()
        );
    }

    @Scheduled(cron = "0 0 1 * * *")  // Daily at 1 AM
    public void calculateDailyMetrics() {
        // Pre-aggregate metrics for fast dashboard queries
        LocalDate yesterday = LocalDate.now().minusDays(1);
        DailyMetrics metrics = reportingRepository.calculateDailyMetrics(yesterday);
        dailyMetricsRepository.save(metrics);
    }
}
```

**Access**: Read-only for most users, BI tools, dashboards

---

## Database Summary

| Database | Services | Technology | Purpose | Volume |
|----------|----------|------------|---------|--------|
| vaullet_db | Vaullet | PostgreSQL | Immutable ledger, source of truth | High (append-only) |
| transactions_db | Transaction | PostgreSQL | Transaction lifecycle, mutable state | Very High (>1000 TPS) |
| fraud_db | Fraud Detection, Risk Mgmt | PostgreSQL + Elasticsearch | Fraud cases, ML, pattern detection | Medium |
| support_db | Limits, Bonus, Subscription, Notification, Audit | PostgreSQL (multi-schema) | Supporting services | Low-Medium |
| redis_cluster | Auth, Transaction, Limits, Fraud | Redis Cluster | Sessions, locks, counters, cache | High (ephemeral) |
| reporting_db | Reporting ETL Service | PostgreSQL → ClickHouse | Analytics, BI, reporting | High (read-heavy) |

**Total infrastructure**: 5 PostgreSQL instances + 1 Redis Cluster + 1 Elasticsearch cluster

---

## Consequences

### Positive Consequences

✅ **Critical data isolated**: Vaullet ledger cannot be corrupted by other services
✅ **Independent scaling**: High-volume services (transactions, fraud) scale independently
✅ **Technology flexibility**: Can use Elasticsearch for fraud patterns, ClickHouse for analytics
✅ **Operational balance**: Not managing 12 separate databases
✅ **Cost-effective**: Shared DB for low-volume services reduces infrastructure cost
✅ **Clear boundaries**: Each service's data ownership is explicit
✅ **Reporting decoupled**: Analytical queries don't impact operational performance
✅ **Room to evolve**: Shared DB services can be split later if needed

### Negative Consequences

⚠️ **No ACID across services**: Can't use database transactions spanning Transaction + Vaullet
⚠️ **Data duplication**: Reporting DB duplicates data from operational DBs
⚠️ **ETL complexity**: Need to maintain Reporting ETL Service
⚠️ **Eventual consistency**: Reporting data lags behind operational data (seconds)
⚠️ **More databases than shared approach**: 6 databases vs 1-2 in monolith
⚠️ **Schema coordination for shared DB**: Bonus and Limits teams share one database

### Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Shared DB becomes bottleneck | Performance degradation | Monitor per-schema metrics, split high-volume schemas first |
| ETL lag for reporting | Stale dashboards | Monitor lag metrics, alert if >1 minute lag, add stream processing if needed |
| Vaullet DB is single point of failure | No transactions can proceed | HA setup with automatic failover, continuous backup, DR plan |
| Reporting DB sync fails | Missing data in reports | Dead letter queue for failed events, reconciliation job, alerts |
| Schema changes in shared DB | Service coordination overhead | Backward-compatible migrations only, versioned schemas |

---

## Alternatives Considered

### Alternative 1: Database Per Service (Full Separation)

**Description**: Each of the 12 services gets its own dedicated database.

**Pros**:
- Maximum isolation
- True microservices
- Independent everything

**Cons**:
- 12+ PostgreSQL instances to manage
- High infrastructure cost
- Operational overhead for small team
- Over-engineered for low-volume services (Bonus, Notification)

**Why Rejected**: Over-engineered for current needs. Support services (Bonus, Limits, etc.) don't need isolation. Can always split later.

---

### Alternative 2: Shared Database (All Services)

**Description**: All services share one database, each with their own tables.

**Pros**:
- Simple (one database)
- Easy JOINs for reports
- Low cost

**Cons**:
- Tight coupling (schema changes affect everyone)
- No isolation (one service's bad query impacts all)
- Can't scale databases independently
- Deployment coordination nightmare
- No technology flexibility (everyone uses PostgreSQL)

**Why Rejected**: Violates microservices principles. Vaullet ledger is too critical to share database with other services.

---

### Alternative 3: Query Operational Databases for Reporting

**Description**: Use existing operational databases for reporting queries instead of separate analytics DB.

**Example**:
```sql
-- Report query hitting operational DBs
SELECT t.*, b.bonus_amount, f.risk_score
FROM transactions_db.transactions t
LEFT JOIN support_db.bonus_schema.bonuses b ON t.transaction_id = b.transaction_id
LEFT JOIN fraud_db.fraud_analysis_history f ON t.transaction_id = f.transaction_id
WHERE t.created_at > NOW() - INTERVAL '30 days';
```

**Pros**:
- No ETL needed
- Real-time data (no lag)
- One less database

**Cons**:
- **Violates service boundaries**: Reporting directly queries other services' databases
- **Performance impact**: Heavy analytical queries slow down operational transactions
- **Can't use cross-database JOINs**: PostgreSQL doesn't support JOINs across DB instances
- **Tight coupling**: Report changes when schemas change

**Why Rejected**:
- Even if technically possible (shared DB), it violates service encapsulation
- Analytical queries would degrade operational performance (transactions, fraud detection)
- Industry best practice: CQRS pattern with separate read models

---

### Alternative 4: Event Sourcing for Everything

**Description**: Store only events, rebuild state from event log.

**Pros**:
- Full audit trail
- Time-travel queries
- Natural fit for reporting

**Cons**:
- Paradigm shift for team
- Complex to implement
- Overkill for most services
- Performance concerns for high-volume events

**Why Rejected**: Over-engineered. Only Vaullet benefits from event sourcing (ledger is already event-sourced). Other services don't need it.

---

## Implementation Notes

### Database Provisioning

**Development environment** (Docker Compose):
```yaml
version: '3.8'

services:
  vaullet-db:
    image: postgres:15
    environment:
      POSTGRES_DB: vaullet_db
      POSTGRES_USER: vaullet_user
      POSTGRES_PASSWORD: ${VAULLET_DB_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - vaullet-data:/var/lib/postgresql/data

  transactions-db:
    image: postgres:15
    environment:
      POSTGRES_DB: transactions_db
      POSTGRES_USER: transactions_user
      POSTGRES_PASSWORD: ${TRANSACTIONS_DB_PASSWORD}
    ports:
      - "5433:5432"

  fraud-db:
    image: postgres:15
    environment:
      POSTGRES_DB: fraud_db
      POSTGRES_USER: fraud_user
      POSTGRES_PASSWORD: ${FRAUD_DB_PASSWORD}
    ports:
      - "5434:5432"

  support-db:
    image: postgres:15
    environment:
      POSTGRES_DB: support_db
      POSTGRES_USER: support_user
      POSTGRES_PASSWORD: ${SUPPORT_DB_PASSWORD}
    ports:
      - "5435:5432"

  reporting-db:
    image: postgres:15
    environment:
      POSTGRES_DB: reporting_db
      POSTGRES_USER: reporting_user
      POSTGRES_PASSWORD: ${REPORTING_DB_PASSWORD}
    ports:
      - "5436:5432"

  redis-cluster:
    image: redis:7
    ports:
      - "6379:6379"

  elasticsearch:
    image: elasticsearch:8.10.0
    environment:
      - discovery.type=single-node
    ports:
      - "9200:9200"
```

**Production** (Kubernetes):
- Use managed databases (AWS RDS, GCP Cloud SQL, Azure Database)
- Or PostgreSQL StatefulSets with persistent volumes
- Redis: Managed Redis (ElastiCache, MemoryStore) or Redis Operator

### Service Configuration

**Spring Boot application.yml**:
```yaml
# Transaction Service
spring:
  datasource:
    url: jdbc:postgresql://transactions-db:5432/transactions_db
    username: transactions_user
    password: ${TRANSACTIONS_DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate  # Never auto-create in production
  flyway:
    enabled: true
    locations: classpath:db/migration
```

**Flyway migrations** (schema versioning):
```sql
-- transactions-db/src/main/resources/db/migration/V1__create_transactions_table.sql
CREATE TABLE transactions (
    transaction_id VARCHAR(255) PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    amount DECIMAL(19, 4) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    type VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_transactions_user_id ON transactions(user_id);
CREATE INDEX idx_transactions_created_at ON transactions(created_at);
```

### Shared Database Schema Isolation

```sql
-- support_db initialization
CREATE SCHEMA limits_schema;
CREATE SCHEMA bonus_schema;
CREATE SCHEMA subscription_schema;
CREATE SCHEMA notification_schema;
CREATE SCHEMA audit_schema;

-- Create service-specific users
CREATE USER limits_service_user WITH PASSWORD 'secure_password';
CREATE USER bonus_service_user WITH PASSWORD 'secure_password';

-- Grant access only to respective schemas
GRANT USAGE ON SCHEMA limits_schema TO limits_service_user;
GRANT ALL ON ALL TABLES IN SCHEMA limits_schema TO limits_service_user;
GRANT ALL ON ALL SEQUENCES IN SCHEMA limits_schema TO limits_service_user;

GRANT USAGE ON SCHEMA bonus_schema TO bonus_service_user;
GRANT ALL ON ALL TABLES IN SCHEMA bonus_schema TO bonus_service_user;
GRANT ALL ON ALL SEQUENCES IN SCHEMA bonus_schema TO bonus_service_user;

-- Prevent cross-schema access
REVOKE ALL ON SCHEMA limits_schema FROM bonus_service_user;
REVOKE ALL ON SCHEMA bonus_schema FROM limits_service_user;
```

### Reporting ETL Service

```java
@SpringBootApplication
public class ReportingETLApplication {
    public static void main(String[] args) {
        SpringApplication.run(ReportingETLApplication.class, args);
    }
}

@Service
@Slf4j
public class TransactionAnalyticsETL {

    @Autowired
    private TransactionAnalyticsRepository analyticsRepo;

    // Consume transaction created event
    @KafkaListener(topics = "transaction.created", groupId = "reporting-etl")
    public void processTransactionCreated(TransactionCreatedEvent event) {
        log.info("Processing transaction.created: {}", event.getPayload().getTransactionId());

        TransactionAnalytics analytics = TransactionAnalytics.builder()
            .transactionId(event.getPayload().getTransactionId())
            .userId(event.getPayload().getUserId())
            .amount(event.getPayload().getAmount())
            .currency(event.getPayload().getCurrency())
            .merchantId(event.getPayload().getMerchantId())
            .merchantCategory(event.getPayload().getMerchantCategory())
            .transactionType(event.getPayload().getType())
            .transactionStatus(event.getPayload().getStatus())
            .createdAt(event.getTimestamp())
            .build();

        analyticsRepo.save(analytics);
    }

    // Enrich with bonus data
    @KafkaListener(topics = "bonus.applied", groupId = "reporting-etl")
    public void processBonusApplied(BonusAppliedEvent event) {
        log.info("Updating bonus for transaction: {}", event.getPayload().getTransactionId());

        analyticsRepo.findById(event.getPayload().getTransactionId())
            .ifPresent(analytics -> {
                analytics.setBonusAmount(event.getPayload().getBonusAmount());
                analyticsRepo.save(analytics);
            });
    }

    // Enrich with fraud score
    @KafkaListener(topics = "fraud.analysis.completed", groupId = "reporting-etl")
    public void processFraudAnalysis(FraudAnalysisCompletedEvent event) {
        if (event.getPayload().getTransactionId() != null) {
            analyticsRepo.findById(event.getPayload().getTransactionId())
                .ifPresent(analytics -> {
                    analytics.setFraudRiskScore(event.getPayload().getRiskScore());
                    analyticsRepo.save(analytics);
                });
        }
    }

    // Daily aggregation job
    @Scheduled(cron = "0 0 1 * * *")  // 1 AM daily
    public void calculateDailyMetrics() {
        LocalDate yesterday = LocalDate.now().minusDays(1);
        log.info("Calculating daily metrics for {}", yesterday);

        DailyMetrics metrics = analyticsRepo.calculateDailyMetrics(yesterday);
        dailyMetricsRepo.save(metrics);

        log.info("Daily metrics saved: {} transactions, {} volume",
            metrics.getTotalTransactions(), metrics.getTotalVolume());
    }
}
```

### Migration from Shared to Isolated (Future)

If a service in `support_db` outgrows the shared database:

**Step 1**: Create new database
```bash
createdb bonus_db
```

**Step 2**: Export schema and data
```bash
pg_dump -h support-db -U support_user -n bonus_schema support_db > bonus_export.sql
```

**Step 3**: Import to new database
```bash
psql -h bonus-db -U bonus_user bonus_db < bonus_export.sql
```

**Step 4**: Update Bonus Service config
```yaml
spring:
  datasource:
    url: jdbc:postgresql://bonus-db:5432/bonus_db  # Changed
```

**Step 5**: Deploy and verify

**Step 6**: Drop from shared DB
```sql
DROP SCHEMA bonus_schema CASCADE;
```

---

## Monitoring and Observability

### Key Metrics

**Per database**:
- Connection pool utilization
- Query latency (p50, p95, p99)
- Active connections
- Lock waits
- Replication lag (for read replicas)
- Disk I/O

**Reporting ETL**:
- Event processing lag (time between event published and processed)
- Failed event count
- Reporting DB sync lag

**Alerts**:
- Vaullet DB down → P0 (critical, all transactions stop)
- Transactions DB down → P0
- Support DB down → P1 (some features degraded)
- Reporting ETL lag > 5 minutes → P2
- Connection pool exhaustion → P1

### Backup Strategy

| Database | Backup Frequency | Retention | RPO | RTO |
|----------|-----------------|-----------|-----|-----|
| vaullet_db | Continuous (WAL) | 7 years | 0 (point-in-time) | 15 min |
| transactions_db | Hourly snapshots | 30 days | 1 hour | 30 min |
| fraud_db | Daily snapshots | 90 days | 24 hours | 1 hour |
| support_db | Daily snapshots | 30 days | 24 hours | 1 hour |
| reporting_db | Daily snapshots | 7 days | 24 hours | 2 hours (rebuildable from events) |

---

## Evolution Path

### Phase 1 (Current): Hybrid with PostgreSQL Reporting

- 4 operational databases + Redis + Elasticsearch
- PostgreSQL for reporting
- Simple Kafka consumer for ETL

### Phase 2 (6-12 months): Optimize Reporting

When reporting queries become slow (>5 seconds):
- Migrate from PostgreSQL to **ClickHouse** or **TimescaleDB**
- Add stream processing (Kafka Streams, Flink) for real-time aggregations
- Introduce data warehouse concepts (star schema, fact tables)

### Phase 3 (12-24 months): Split Support DB if Needed

If Bonus Service or Limits Service volumes grow significantly:
- Split `bonus_schema` into `bonus_db`
- Split `limits_schema` into `limits_db`
- Follow migration procedure above

### Phase 4 (Future): Cloud Data Warehouse

When data volume is massive (billions of rows):
- Migrate reporting to cloud data warehouse (BigQuery, Snowflake, Redshift)
- Keep operational databases as-is
- Stream events to cloud storage → warehouse ingestion

---

## References

- [Database per Service Pattern](https://microservices.io/patterns/data/database-per-service.html)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- [Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- [PostgreSQL Schema Management](https://www.postgresql.org/docs/current/ddl-schemas.html)
- [ClickHouse for Analytics](https://clickhouse.com/docs/en/intro)
- Related ADRs:
  - [ADR-001: Redis Distributed Locking](001-distributed-locking-for-balance-consistency.md)
  - [ADR-002: Shared Contracts Strategy](002-shared-contracts-versioning-strategy.md)

---

**Date**: 2026-08-27
**Author**: Predrag
**Reviewers**: TBD
**Next Review**: 2027-02-27 (6 months) - Evaluate if support DB needs to be split, assess reporting DB performance
