if you dont mi# Vaullet Walleting System - Architecture Overview

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            API Gateway                                   │
│                     (Request Routing, Auth, Rate Limiting)              │
└────────────┬────────────────────────────────────────────────────────────┘
             │
             │ REST API
             │
┌────────────┴─────────────────────────────────────────────────────────────┐
│                        Core Services Layer                                │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐     │
│  │   Transaction   │  │     Vaullet      │  │   Limits Service   │     │
│  │     Service     │  │  (Ledger Service)│  │                    │     │
│  │                 │  │                  │  │                    │     │
│  │  - Lifecycle    │  │  - Append-only   │  │  - Daily limits    │     │
│  │  - Validation   │  │  - Immutable log │  │  - Monthly limits  │     │
│  │  - Status mgmt  │  │  - Balance calc  │  │  - Velocity checks │     │
│  │  - Orchestration│  │  - Source of     │  │  - Spending caps   │     │
│  │  - Heavy lifting│  │    truth         │  │                    │     │
│  └────────┬────────┘  └────────┬─────────┘  └──────────┬─────────┘     │
│           │                    │                        │               │
│  ┌────────┴────────┐  ┌───────┴──────────┐  ┌──────────┴─────────┐    │
│  │  Bonus Service  │  │  Refund Service  │  │ Subscription Svc   │    │
│  │                 │  │                  │  │                    │    │
│  │  - Cashback     │  │  - Full refunds  │  │  - Recurring pay   │    │
│  │  - Promotions   │  │  - Partial       │  │  - Billing cycles  │    │
│  │  - Loyalty      │  │  - Policy checks │  │  - Retry logic     │    │
│  │  - Rewards      │  │  - Compensation  │  │  - Suspensions     │    │
│  └─────────────────┘  └──────────────────┘  └────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                      Security & Risk Layer                               │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────────┐  ┌────────────────────┐  ┌──────────────────┐    │
│  │  Auth Service    │  │ Fraud Detection    │  │ Risk Management  │    │
│  │                  │  │     Service        │  │    Service       │    │
│  │  - JWT tokens    │  │                    │  │                  │    │
│  │  - User auth     │  │  - Real-time rules │  │  - Case mgmt     │    │
│  │  - Permissions   │  │  - ML scoring      │  │  - Investigation │    │
│  │  - MFA           │  │  - Pattern detect  │  │  - Reviews       │    │
│  │  - Sessions      │  │  - Risk scoring    │  │  - Actions       │    │
│  └──────────────────┘  └────────────────────┘  └──────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                      Supporting Services Layer                            │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐      │
│  │  Notification    │  │  Audit Service   │  │ Scheduler Service│      │
│  │    Service       │  │                  │  │                  │      │
│  │                  │  │  - Compliance    │  │  - Cron jobs     │      │
│  │  - Email/SMS     │  │  - Audit logs    │  │  - Recurring pay │      │
│  │  - Push notif    │  │  - Reporting     │  │  - Retries       │      │
│  │  - Templates     │  │  - Retention     │  │  - Scheduled evt │      │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘      │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                      Infrastructure Layer                                 │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    Apache Kafka (Event Bus)                       │   │
│  │  Topics: transaction.*, refund.*, fraud.*, bonus.*, etc.         │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐      │
│  │   PostgreSQL     │  │     Redis        │  │   Monitoring     │      │
│  │   (Per Service)  │  │  (Cache, Locks)  │  │  (Prometheus,    │      │
│  │                  │  │                  │  │   Grafana)       │      │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘      │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## Service Details

### Core Services

#### 1. **Vaullet (Ledger Service)** ⭐
**Role**: Source of truth for all money movements

**Characteristics**:
- **Append-only**: Never updates or deletes entries
- **Immutable**: Historical records cannot be changed
- **Producer-only**: Only writes to its own database
- **High consistency**: Critical for financial accuracy

**Responsibilities**:
- Record all financial transactions (debits, credits)
- Calculate account balances
- Provide transaction history
- Ensure data integrity and audit trail

**Database**: PostgreSQL (append-only schema)

**Event Consumption**:
- `transaction.created` → Create ledger entry
- `refund.initiated` → Create credit entry
- `bonus.applied` → Create credit entry
- `bonus.revoked` → Create debit entry
- `recurring-payment.processed` → Create debit entry

**Event Production**:
- `ledger.entry.recorded`
- `ledger.balance.updated`

---

#### 2. **Transaction Service** 🔧
**Role**: Heavy lifting - transaction lifecycle management

**Characteristics**:
- **Mutable**: Transaction status changes over time
- **Orchestrator**: Coordinates validation checks
- **Complex logic**: Handles business rules, state machines

**Responsibilities**:
- Create and manage transaction lifecycle
- Coordinate with Fraud, Limits, Vaullet services
- Handle transaction status transitions (PENDING → APPROVED → COMPLETED)
- Store user-facing transaction metadata
- Link transactions with refunds, bonuses

**Database**: PostgreSQL (mutable schema)

**REST API (Synchronous)**:
- GET /transactions/{id}
- POST /transactions
- GET /transactions/user/{userId}

**Event Production**:
- `transaction.created`
- `transaction.approved`
- `transaction.completed`
- `transaction.failed`
- `transaction.status.updated`

**Event Consumption**:
- `refund.initiated` → Link refund to transaction
- `bonus.applied` → Update transaction with bonus
- `fraud.suspicion.detected` → Update transaction status

---

#### 3. **Limits Service**
**Role**: Enforce spending limits and velocity checks

**Responsibilities**:
- Daily/monthly spending limits per user
- Transaction velocity checks (e.g., max 5 per hour)
- Merchant category limits
- Dynamic limit adjustments based on risk

**Database**: Redis (counters) + PostgreSQL (configuration)

**REST API (Synchronous)**:
- POST /limits/check → Called by Transaction Service
- GET /limits/user/{userId}
- PUT /limits/user/{userId}

**Event Consumption**:
- `transaction.created` → Increment usage counters
- `refund.initiated` → Decrement usage counters
- `limit.threshold.reached` → Alert user

---

#### 4. **Bonus Service**
**Role**: Manage cashback, promotions, loyalty rewards

**Responsibilities**:
- Calculate bonus eligibility
- Apply bonuses to transactions
- Revoke bonuses on refunds
- Track bonus expiration
- Loyalty program management

**Database**: PostgreSQL

**Event Production**:
- `bonus.applied`
- `bonus.revoked`
- `bonus.expired`

**Event Consumption**:
- `transaction.created` → Check eligibility
- `refund.initiated` → Revoke bonus if applicable
- `recurring-payment.processed` → Loyalty rewards

---

#### 5. **Refund Service**
**Role**: Handle full and partial refunds

**Responsibilities**:
- Validate refund requests
- Check refund policies
- Process refund lifecycle
- Handle bonus reversals

**Database**: PostgreSQL

**Event Production**:
- `refund.initiated`
- `refund.completed`
- `refund.failed`

**Event Consumption**:
- Transaction lookup for original transaction

---

#### 6. **Subscription Service**
**Role**: Manage recurring payments

**Responsibilities**:
- Create and manage subscriptions
- Handle billing cycles
- Retry failed payments
- Suspend/cancel subscriptions

**Database**: PostgreSQL

**Event Production**:
- `subscription.created`
- `recurring-payment.due`
- `recurring-payment.processed`
- `recurring-payment.failed`
- `subscription.suspended`

**Event Consumption**:
- `recurring-payment.due` → Process payment (from Scheduler)

---

### Security & Risk Services

#### 7. **Auth Service**
**Role**: Authentication and authorization

**Responsibilities**:
- User login/logout
- JWT token generation and validation
- Multi-factor authentication (MFA)
- Session management
- Permission checks

**Database**: PostgreSQL + Redis (sessions)

---

#### 8. **Fraud Detection Service**
**Role**: Real-time fraud prevention

**Responsibilities**:
- Rule-based fraud checks (velocity, location, amount)
- ML model scoring
- Risk assessment
- Pattern detection

**Database**: PostgreSQL (rules, cases) + Redis (real-time data)

**REST API (Synchronous)**:
- POST /fraud/analyze-transaction → Called by Transaction Service before transaction creation

**Event Production**:
- `fraud.analysis.completed`
- `fraud.suspicion.detected`
- `fraud.pattern.detected`

---

#### 9. **Risk Management Service**
**Role**: Fraud case management and investigation

**Responsibilities**:
- Create fraud cases
- Manual review workflow
- Investigator assignment
- Case resolution
- Account actions (freeze, unfreeze)

**Database**: PostgreSQL

**Event Production**:
- `fraud.case.created`
- `fraud.case.reviewed`
- `account.locked`
- `account.unlocked`

---

### Supporting Services

#### 10. **Notification Service**
**Role**: Multi-channel notifications

**Responsibilities**:
- Email notifications
- SMS notifications
- Push notifications
- Template management
- Delivery tracking

**Database**: PostgreSQL (templates, delivery logs)

**Event Consumption**: Listens to ALL events that require user notification

---

#### 11. **Audit Service**
**Role**: Compliance and audit logging

**Responsibilities**:
- Log all financial events
- Compliance reporting
- Audit trail maintenance
- Data retention policies

**Database**: PostgreSQL (time-series optimized)

**Event Consumption**: Listens to ALL events for audit logging

---

#### 12. **Scheduler Service**
**Role**: Job scheduling and execution

**Responsibilities**:
- Cron-based job execution
- Recurring payment triggers
- Retry scheduling
- One-time scheduled events

**Database**: PostgreSQL (job definitions)

**Event Production**:
- `recurring-payment.due`
- `scheduled-job.executed`

---

## Communication Patterns

### Synchronous (REST) - Pre-transaction Validation
```
User Request
    ↓
API Gateway
    ↓
Transaction Service ──→ Auth Service (validate token)
    ↓
    ├──→ Fraud Detection Service (analyze risk)
    ├──→ Limits Service (check limits)
    └──→ Vaullet (check balance)
    ↓
Decision: APPROVE / REJECT / REVIEW
```

### Asynchronous (Kafka Events) - Post-transaction Processing
```
Transaction Service publishes: transaction.created
    ↓
    ├──→ Vaullet: Create ledger entry
    ├──→ Limits Service: Update counters
    ├──→ Bonus Service: Check eligibility
    ├──→ Audit Service: Log event
    └──→ Notification Service: Send confirmation
```

## Key Architectural Principles

### 1. **Vaullet (Ledger) is Special**
- **Never mutates data** - only appends
- **Single source of truth** for balances
- **Isolated database** - no other service writes to it
- **Eventual consistency** - other services sync to it

### 2. **Transaction Service Orchestrates**
- Makes synchronous calls for validation
- Publishes events for side effects
- Handles complex business logic
- Manages state transitions

### 3. **Event-Driven Choreography**
- Services react to events independently
- No central orchestrator (except Transaction Service for its domain)
- Loose coupling via Kafka
- Each service is autonomous

### 4. **Polyrepo Structure**
Each service is a separate Git repository:
- `wallet-transaction-service`
- `wallet-vaullet-service` ⭐
- `wallet-limits-service`
- `wallet-bonus-service`
- `wallet-refund-service`
- `wallet-subscription-service`
- `wallet-auth-service`
- `wallet-fraud-detection-service`
- `wallet-risk-management-service`
- `wallet-notification-service`
- `wallet-audit-service`
- `wallet-scheduler-service`
- `wallet-shared-contracts` (DTOs, event schemas)
- `wallet-infrastructure` (Terraform, K8s manifests)

## Data Flow Examples

### Example 1: User Makes a Withdrawal

```
1. User → API Gateway → Transaction Service
2. Transaction Service → Auth Service (REST): Validate token ✓
3. Transaction Service → Fraud Detection (REST): Analyze (risk: LOW) ✓
4. Transaction Service → Limits Service (REST): Check limit ✓
5. Transaction Service → Vaullet (REST): Check balance ✓
6. Transaction Service: Create transaction (status: APPROVED)
7. Transaction Service → Kafka: Publish transaction.created event

   Event: transaction.created
   ↓
   ├─→ Vaullet: Create debit entry → Publish ledger.entry.recorded
   ├─→ Limits Service: Increment daily usage
   ├─→ Bonus Service: Check cashback eligibility → Publish bonus.applied
   ├─→ Audit Service: Log transaction
   └─→ Notification Service: Send confirmation email
```

### Example 2: Fraud Detected

```
1. Transaction Service → Fraud Detection (REST): Analyze
2. Fraud Detection: Risk = HIGH, Decision = BLOCK
3. Transaction Service: Create transaction (status: BLOCKED_FRAUD)
4. Fraud Detection → Kafka: Publish fraud.suspicion.detected

   Event: fraud.suspicion.detected
   ↓
   ├─→ Risk Management: Create fraud case
   ├─→ Auth Service: Require MFA on next login
   ├─→ Notification Service: Alert user via SMS
   └─→ Notification Service: Alert fraud team via Slack
```

### Example 3: Recurring Payment

```
1. Scheduler Service (cron trigger) → Kafka: Publish recurring-payment.due
2. Subscription Service consumes event
3. Subscription Service → Limits Service (REST): Check limit ✓
4. Subscription Service → Vaullet (REST): Check balance ✓
5. Subscription Service: Process payment
6. Subscription Service → Kafka: Publish recurring-payment.processed

   Event: recurring-payment.processed
   ↓
   ├─→ Transaction Service: Create transaction record
   ├─→ Vaullet: Create debit entry
   ├─→ Limits Service: Update monthly usage
   ├─→ Bonus Service: Apply loyalty cashback
   └─→ Notification Service: Send payment confirmation
```

## Technology Stack

- **Language**: Java 17+
- **Framework**: Spring Boot 3.x
- **Event Bus**: Apache Kafka
- **Databases**: PostgreSQL (per service)
- **Cache**: Redis
- **API Gateway**: Spring Cloud Gateway / Kong
- **Service Discovery**: Kubernetes native / Consul
- **Monitoring**: Prometheus + Grafana
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Tracing**: Jaeger / Zipkin
- **Container**: Docker
- **Orchestration**: Kubernetes

## Architectural Decisions

Key architectural decisions are documented as **Architecture Decision Records (ADRs)** in the `docs/adr/` directory.

### Active ADRs

- [ADR-001: Use Redis Distributed Locking for Balance Consistency](docs/adr/001-distributed-locking-for-balance-consistency.md) ✅
  - **Decision**: Use Redis distributed locks to prevent overdrafts and ensure balance consistency
  - **Target**: >1000 TPS system-wide, ~20-50 TPS per user
  - **Trade-off**: Strong consistency over eventual consistency
  - **Future path**: Can migrate to Kafka partitioning if higher throughput needed

- [ADR-002: Shared Contracts with Dual Publishing (Java + TypeScript)](docs/adr/002-shared-contracts-versioning-strategy.md) ✅
  - **Decision**: Create `wallet-shared-contracts` repo that publishes both Maven artifact (for Java services) and NPM package (for frontends)
  - **Strategy**: Backward compatibility by default, semantic versioning, automated TypeScript generation from Java DTOs
  - **Frontend-friendly**: DTO suffix stripped (TransactionDTO → Transaction in TypeScript)
  - **Benefits**: Single source of truth, independent service upgrades, type safety across stack

See [docs/adr/README.md](docs/adr/README.md) for the complete list of ADRs and how to contribute new ones.

## Next Steps

1. Define detailed API contracts (OpenAPI/Swagger)
2. Design database schemas per service
3. Define Kafka topic naming conventions
4. ~~Set up shared contracts repository~~ ✅ **Done** - See ADR-002
5. Design error handling and retry strategies
6. Plan disaster recovery and data backup
7. Decide on database-per-service vs shared database strategy
8. Define monitoring and observability requirements