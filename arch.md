# Vaullet Walleting System - Architecture Overview

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
│  │  Bonus Service ⭐│  │  Loyalty Service │  │ Subscription Svc   │    │
│  │                 │  │                  │  │                    │    │
│  │  - Cashback     │  │  - Points/tiers  │  │  - Recurring pay   │    │
│  │  - Promotions   │  │  - Redemption    │  │  - Billing cycles  │    │
│  │  - Wagering     │  │  - Thresholds    │  │  - Retry logic     │    │
│  │  SELLABLE       │  │  SELLABLE        │  │  SELLABLE          │    │
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
│  │   (Hybrid)       │  │  (Cache, Locks)  │  │  (Prometheus,    │      │
│  │   See below ↓    │  │                  │  │   Grafana)       │      │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘      │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Database Architecture (Hybrid Strategy)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Isolated Databases (Critical)                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│ Vaullet Service │────────→│   vaullet_db    │         │ Transaction Svc │
│                 │         │   PostgreSQL    │         │                 │
│ - Append-only   │         │                 │         │ - Mutable state │
│ - Immutable     │         │ - ledger_entries│         │ - High volume   │
│ - Source of     │         │ - balances      │         │ - Lifecycle     │
│   truth         │         │ - audit_log     │         │                 │
└─────────────────┘         └─────────────────┘         └─────────────────┘
                                                                 │
                                                                 ↓
                                                         ┌─────────────────┐
                                                         │ transactions_db │
                                                         │   PostgreSQL    │
                                                         │                 │
                                                         │ - transactions  │
                                                         │ - refunds       │
┌─────────────────┐                                     │ - metadata      │
│ Fraud Detection │                                     └─────────────────┘
│     Service     │
│                 │
│ - Real-time ML  │
│ - Pattern detect│         ┌─────────────────┐
│ - Case mgmt     │────────→│    fraud_db     │
│                 │         │ PostgreSQL +    │
└─────────────────┘         │ Elasticsearch   │
                            │                 │
                            │ - fraud_cases   │
                            │ - fraud_rules   │
                            │ - patterns (ES) │
                            └─────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                 Auth Database (Isolated) 🔐                         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────┐         ┌─────────────────────────────────────┐
│  Auth Service   │────────→│         auth_db (PostgreSQL)        │
│                 │         │  users, credentials, mfa_configs,   │
│  - Core module  │         │  roles, permissions, api_clients    │
└─────────────────┘         └─────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│        Rewards Database (Shared) — CONDITIONAL on contract          │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────┐         ┌─────────────────────────────────────┐
│ Bonus Service ⭐│────────→│        rewards_db (PostgreSQL)      │
└─────────────────┘         │                                     │
                            │  bonus_schema     (if sold)         │
┌─────────────────┐         │  loyalty_schema   (if sold)         │
│ Loyalty Service │────────→│  referral_schema  (if sold)         │
└─────────────────┘         │                                     │
                            │  Holds RULES only — wagering terms, │
┌─────────────────┐         │  tiers, campaigns. Never balances.  │
│ Referral Service│────────→│  Promotional MONEY lives in         │
└─────────────────┘         │  vaullet_db buckets (ADR-004).      │
                            └─────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│              Support Database (Shared) — mixed                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────┐         ┌─────────────────────────────────────┐
│ Limits Service  │────────→│         support_db (PostgreSQL)     │
│    (core)       │         │                                     │
└─────────────────┘         │  limits_schema      (always)        │
                            │  audit_schema       (always)        │
┌─────────────────┐         │  scheduler_schema   (always)        │
│  Audit Service  │────────→│                                     │
│    (core)       │         │  subscription_schema  (if sold)     │
└─────────────────┘         │  notification_schema  (if sold)     │
                            │                                     │
┌─────────────────┐         │  Schema-per-service isolation with  │
│ Scheduler (infra)│───────→│  dedicated credentials per service. │
└─────────────────┘         └─────────────────────────────────────┘
                                         ↑
┌─────────────────┐  ┌──────────────────┐│
│ Subscription Svc│──│ Notification Svc │┘
│   (sellable)    │  │   (sellable)     │
└─────────────────┘  └──────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                   Analytics / Reporting Layer                       │
└─────────────────────────────────────────────────────────────────────┘

                    ┌────────────────────────────────┐
                    │      Apache Kafka Topics       │
                    │  transaction.*, bonus.*,       │
                    │  refund.*, fraud.*, etc.       │
                    └────────────────────────────────┘
                                   │
                                   ↓ (ETL consumes events)
                    ┌────────────────────────────────┐
                    │   Reporting ETL Service        │
                    │   (SELLABLE — may be absent)   │
                    │   - Joins data from events     │
                    │   - Denormalizes               │
                    │   - Pre-aggregates metrics     │
                    └────────────────────────────────┘
                                   │
                                   ↓
                    ┌────────────────────────────────┐
                    │      datawarehouse_db          │
                    │   PostgreSQL (Phase 1)         │
                    │   → ClickHouse (Phase 2)       │
                    │   Provisioned only if sold     │
                    │                                │
                    │ - transaction_analytics        │
                    │ - daily_metrics                │
                    │ - user_summaries               │
                    │ - rewards_analytics            │
                    │ - merchant_analytics           │
                    └────────────────────────────────┘
                                   ↑
                                   │ (Read-only queries)
                    ┌────────────────────────────────┐
                    │    BI Tools / Dashboards       │
                    │   Metabase, Tableau, etc.      │
                    └────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                         Cache Layer                                 │
└─────────────────────────────────────────────────────────────────────┘

                    ┌────────────────────────────────┐
                    │      Redis Cluster             │
                    │   (3 masters, 3 replicas)      │
                    │                                │
                    │ Namespaces:                    │
                    │ - session:* (Auth)             │
                    │ - balance:cache:* (Vaullet)    │
                    │ - limits:counter:* (Limits)    │
                    │ - fraud:cache:* (Fraud)        │
                    └────────────────────────────────┘

  Note: Redis holds no locks. Balance consistency is enforced inside a
  Postgres transaction in Vaullet (ADR-004). Redis is a cache, not a
  correctness dependency.
```

**Database Summary**:
- **7 PostgreSQL databases** + Redis Cluster + Elasticsearch = **9 infrastructure components** (full deployment)
- **Isolated**: `vaullet_db` (ledger), `transactions_db` (high volume), `fraud_db` (special tech), `auth_db` (security)
- **Shared, schema-per-service**: `rewards_db` (Bonus, Loyalty, Referral), `support_db` (Limits, Subscription, Notification, Audit, Scheduler)
- **Analytics**: `datawarehouse_db`, populated from Kafka events (CQRS pattern)
- **Conditional**: `rewards_db` and `datawarehouse_db` are provisioned only when the customer's
  contract includes a module that needs them. A core-only deployment runs 5 PostgreSQL databases.
- See [ADR-003](docs/adr/003-hybrid-database-strategy-with-analytics.md) and
  [ADR-005](docs/adr/005-module-composition-and-deployment-topology.md) for details

## Service Details

### Core Services

#### 1. **Vaullet (Ledger Service)** ⭐ — CORE
**Role**: Source of truth for all money movements, and the authority on available balance

**Characteristics**:
- **Append-only journal**: `ledger_entries` is never updated or deleted
- **Derived projection**: `account_balances` and `balance_buckets` are mutable but rebuildable by
  replaying the journal. This is the standard journal-plus-balances split; "append-only" describes
  the journal, not the whole service.
- **Synchronous on the write path**: exposes `POST /reservations`, which atomically checks and holds
  balance in one Postgres transaction (ADR-004). This is what makes overdrafts impossible.
- **Typed money**: balances are held in per-grant buckets (`CASH`, `BONUS`, `LOYALTY`, `REFERRAL`)
  with their own `withdrawable` flag, expiry and spend priority.

**Responsibilities**:
- Atomically reserve, settle, release and expire holds
- Record all money movement as immutable double-entry journal lines
- Maintain per-bucket balances and compute withdrawable balance
- Provide transaction history

**Database**: `vaullet_db` — PostgreSQL

**REST API (Synchronous)**:
- `POST /v1/reservations` → atomic check-and-hold (called by Transaction Service) ⭐
- `DELETE /v1/reservations/{id}` → release
- `GET /v1/accounts/{id}/balance` → `{posted, held, available, withdrawable}`

**Event Consumption**:
- `transaction.completed` → settle reservation, write journal entries
- `transaction.failed` → release reservation
- `ValueGranted` (from Bonus / Loyalty / Referral) → mint a new balance bucket
- `bonus.revoked` → debit the granting bucket

**Event Production**:
- `ledger.entry.recorded`
- `ledger.balance.updated`
- `ledger.settlement.rejected` (hold expired before settlement)

---

#### 2. **Transaction Service** 🔧 — CORE
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

#### 3. **Limits Service** — CORE
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

#### 4. **Bonus Service** ⭐ — SELLABLE (flagship)
**Role**: Manage cashback, promotions, loyalty rewards

**Responsibilities**:
- Calculate bonus eligibility
- Apply bonuses to transactions
- Revoke bonuses on refunds
- Track bonus expiration
- Loyalty program management

**Database**: PostgreSQL

**The flagship module.** Bonus does not hold balances — it holds *rules* (wagering terms, campaign
eligibility, expiry policy) in `rewards_db`, and grants money into `vaullet_db` balance buckets by
emitting events. That split is what keeps it removable: a deployment without Bonus has a ledger with
one `CASH` bucket per account and no promotional logic anywhere.

**Event Production**:
- `ValueGranted` — mints a `BONUS` bucket in the ledger (shared contract with Loyalty and Referral)
- `bonus.applied`
- `bonus.revoked`
- `bonus.expired`
- `bonus.wagering.completed` → ledger transfers `BONUS` → `CASH`, making funds withdrawable

**Event Consumption**:
- `transaction.created` → Check eligibility
- `refund.initiated` → Revoke bonus if applicable
- `recurring-payment.processed` → Loyalty rewards

---

#### 5. **Refund** (part of Transaction Service) — CORE
**Role**: Handle full and partial refunds

**Not a separate deployment.** Refund logic ships inside Transaction Service and shares
`transactions_db`. It was previously specified as its own service; splitting it would have put two
services on the same database with no independent scaling or release benefit.

**Responsibilities**:
- Validate refund requests and check refund policies
- Process refund lifecycle
- Trigger bonus reversal (by event; Bonus may not be deployed)

**Database**: `transactions_db` (shared with Transaction Service)

**Event Production**:
- `refund.initiated`
- `refund.completed`
- `refund.failed`

**Refund and buckets**: a refund returns money to the bucket it was drawn from, using the
`reservation_allocations` recorded at reserve time (ADR-004). Cash refunds to cash; bonus refunds to
bonus. Without that record there would be no correct way to reverse a mixed-bucket payment.

---

#### 6. **Subscription Service** — SELLABLE
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

#### 6a. **Loyalty Service** — SELLABLE
**Role**: Points, tiers, and redemption

Sells independently of Bonus. Awards points on qualifying events, manages tier thresholds, and
converts redemptions into ledger value via `ValueGranted` (bucket type `LOYALTY`).

**Database**: `rewards_db` (`loyalty_schema`)

**Event Consumption**: `transaction.completed`, `ValueGranted`
**Event Production**: `ValueGranted`, `loyalty.tier.changed`, `loyalty.points.redeemed`

---

#### 6b. **Referral Service** — SELLABLE
**Role**: User referral programme

Sells independently of Bonus. Tracks referral links and attribution, and pays referral rewards as
ledger value via `ValueGranted` (bucket type `REFERRAL`).

**Database**: `rewards_db` (`referral_schema`)

**Event Consumption**: `user.registered`, `transaction.completed`
**Event Production**: `ValueGranted`, `referral.completed`

---

#### 6c. **Reporting ETL Service** — SELLABLE
**Role**: Populate the analytics store from the event stream

Consumes every domain event, denormalizes and pre-aggregates into `datawarehouse_db` (CQRS). Never
queries an operational database — that is the whole point of the separation (ADR-003).

**Database**: `datawarehouse_db` (provisioned only when this module is sold)

**Event Consumption**: all topics
**Event Production**: none

---

### Security & Risk Services

#### 7. **Auth Service** — CORE
**Role**: Authentication and authorization

**Responsibilities**:
- User login/logout
- JWT token generation and validation
- Multi-factor authentication (MFA)
- Session management
- Permission checks

**Database**: PostgreSQL + Redis (sessions)

---

#### 8. **Fraud Detection Service** — CORE
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

#### 9. **Risk Management Service** — SELLABLE
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

#### 10. **Notification Service** — SELLABLE
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

#### 11. **Audit Service** — CORE (compliance)
**Role**: Compliance and audit logging

**Responsibilities**:
- Log all financial events
- Compliance reporting
- Audit trail maintenance
- Data retention policies

**Database**: PostgreSQL (time-series optimized)

**Event Consumption**: Listens to ALL events for audit logging

---

#### 12. **Scheduler Service** — PLATFORM INFRASTRUCTURE
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
    └──→ Vaullet POST /reservations  ← atomic check-and-HOLD, not a read
    ↓
Decision: APPROVE / REJECT / REVIEW
```

**Every service on this path is core** (ADR-005). That is not a coincidence — it is the rule:
a synchronous dependency cannot be optional, because a missing one fails either closed (halting all
transactions) or open (silently disabling a financial control).

The reservation step is where the overdraft invariant is decided. It is a *write*, committed inside
one Postgres transaction in Vaullet, not a balance read followed by a hopeful approval — see
[ADR-004](docs/adr/004-atomic-balance-reservations.md) for why the earlier lock-and-read design
did not work.

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
- **Immutable journal** - `ledger_entries` is only ever appended to
- **Single source of truth** for balances, including promotional money
- **Isolated database** - no other service writes to `vaullet_db`
- **Strong consistency on the write path** - reservations are ACID, not eventual. Everything
  *downstream* of a settled entry is eventually consistent; the balance decision itself is not.

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
Each service is a separate Git repository. Category per
[ADR-005](docs/adr/005-module-composition-and-deployment-topology.md):

**Core** (always deployed, never optional):
- `wallet-transaction-service` (includes Refund)
- `wallet-vaullet-service` ⭐
- `wallet-auth-service`
- `wallet-fraud-detection-service`
- `wallet-limits-service`
- `wallet-audit-service` (core by compliance, not by architecture)
- `wallet-admin-ui`

**Platform infrastructure** (always deployed, never sold):
- `wallet-scheduler-service`

**Sellable modules** (deployed only if the contract includes them):
- `wallet-bonus-service` ⭐ *flagship*
- `wallet-loyalty-service`
- `wallet-referral-service`
- `wallet-subscription-service`
- `wallet-risk-management-service`
- `wallet-notification-service`
- `wallet-reporting-etl-service`

**Shared**:
- `wallet-shared-contracts` (DTOs, event schemas — defines all events regardless of what's deployed)
- `wallet-infrastructure` (Terraform, Helm umbrella chart, K8s manifests)
- `wallet-contracts` (per-customer Helm values, GitOps)

**Total**: 15 backend services + Admin UI.

## Data Flow Examples

### Example 1: User Makes a Withdrawal

```
1. User → API Gateway → Transaction Service
2. Transaction Service → Auth Service (REST): Validate token ✓
3. Transaction Service → Fraud Detection (REST): Analyze (risk: LOW) ✓
4. Transaction Service → Limits Service (REST): Check limit ✓
5. Transaction Service → Vaullet (REST): POST /reservations
      → Vaullet, in ONE Postgres transaction:
          lock account row → allocate across buckets by spend_priority
          → insert reservation + allocations → increment held_total
      → 201 {reservation_id}   (or 409 INSUFFICIENT_FUNDS)
6. Transaction Service: Create transaction (status: APPROVED)
7. Transaction Service → Kafka: Publish transaction.created

   Event: transaction.created
   ↓
   ├─→ Limits Service: Increment daily usage
   ├─→ Bonus Service (if deployed): Check cashback eligibility → Publish ValueGranted
   ├─→ Audit Service: Log transaction
   └─→ Notification Service (if deployed): Send confirmation email

8. On completion → Transaction Service publishes transaction.completed
   ↓
   └─→ Vaullet: SETTLE — write journal entries, release hold, apply to posted_balance

   On failure → transaction.failed → Vaullet RELEASES the hold. No journal entry:
   money that never moved leaves no record.
```

**Note the money is held at step 5, not step 8.** Between approval and settlement the funds are
unavailable to any other transaction, which is what makes concurrent withdrawals safe. Consumers
that aren't deployed simply don't appear — the producer is unaffected.

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
3. Subscription Service → Transaction Service (REST): POST /transactions
      Subscription does NOT check limits or balance itself. It requests a transaction
      like any other client, and Transaction Service runs the standard approval path
      (auth → fraud → limits → reserve).
4. Transaction Service: normal flow — approve, settle, publish transaction.completed
5. Subscription Service consumes transaction.completed → advance billing cycle
      (or transaction.failed → retry / suspend per policy)

   Event: transaction.completed
   ↓
   ├─→ Vaullet: Settle the hold, write journal entries
   ├─→ Limits Service: Update monthly usage
   ├─→ Bonus Service (if deployed): Apply loyalty cashback → ValueGranted
   ├─→ Audit Service: Log transaction
   └─→ Notification Service (if deployed): Send payment confirmation
```

**Why Subscription doesn't call Limits and Vaullet directly**: it previously did, which put the
money path in two places — one in Transaction Service, one in Subscription. Every future control
(a new fraud rule, a regulatory check) would need implementing twice, and the second copy would
drift. It also broke the ADR-005 rule: a sellable module held synchronous clients for core services
and duplicated core logic. There is exactly one path money can move by, and every caller uses it.

## Technology Stack

- **Language**: Java 17+
- **Framework**: Spring Boot 3.x
- **Event Bus**: Apache Kafka
- **Databases**: PostgreSQL (hybrid grouping — see ADR-003) + Elasticsearch (fraud patterns)
- **Cache**: Redis
- **API Gateway**: Spring Cloud Gateway / Kong
- **Service Discovery**: Kubernetes native / Consul
- **Monitoring**: Prometheus + Grafana
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Tracing**: Jaeger / Zipkin
- **Container**: Docker
- **Orchestration**: Kubernetes

## Deployment Topology

Vaullet is sold as a **single-tenant deployment**: each customer runs their own Kubernetes cluster
with their own instance of the platform. There is no shared infrastructure between customers, no
`tenant_id`, and no cross-customer data path. Isolation is the cluster boundary — stronger than
row-level scoping, and frequently mandatory under gambling licences with data-residency conditions.

A contract determines which modules that customer's cluster runs. Modules a customer did not buy are
**not deployed at all** — no pod, no schema, no consumer group.

```
Customer A (full)                    Customer B (core + Bonus only)
┌──────────────────────────┐         ┌──────────────────────────┐
│ CORE                     │         │ CORE                     │
│  Transaction, Vaullet,   │         │  Transaction, Vaullet,   │
│  Auth, Fraud, Limits,    │         │  Auth, Fraud, Limits,    │
│  Audit, Admin UI         │         │  Audit, Admin UI         │
│ INFRA                    │         │ INFRA                    │
│  Scheduler, Kafka, Redis │         │  Scheduler, Kafka, Redis │
│ MODULES                  │         │ MODULES                  │
│  Bonus ⭐ Loyalty        │         │  Bonus ⭐                │
│  Referral Subscription   │         │                          │
│  RiskMgmt Notification   │         │                          │
│  ReportingETL            │         │                          │
│ DATABASES: 7             │         │ DATABASES: 6             │
│  + datawarehouse_db      │         │  (no datawarehouse_db)   │
└──────────────────────────┘         └──────────────────────────┘
```

**The rule that makes this work**: *a synchronous dependency makes a component core; optional
functionality communicates only through events.* Turning off an event consumer is safe — Kafka
neither knows nor cares that a consumer group is absent. Turning off a synchronous dependency is
not. So nothing optional sits in a synchronous path.

**The invariant**: *no sellable module may depend on another sellable module.* Every module
dependency points at core or infrastructure, so the module graph is a star, not a mesh — and all
2⁷ = 128 combinations of sellable modules are valid by construction. There are no illegal
configurations to detect or guard against.

See [ADR-005](docs/adr/005-module-composition-and-deployment-topology.md) for the full treatment.

## Architectural Decisions

Key architectural decisions are documented as **Architecture Decision Records (ADRs)** in the `docs/adr/` directory.

### Active ADRs

- [ADR-001: Use Redis Distributed Locking for Balance Consistency](docs/adr/001-distributed-locking-for-balance-consistency.md) ⛔ **Superseded by ADR-004**
  - The lock guarded a balance *read* while the *write* happened asynchronously after release, so it
    did not prevent the overdraft it was written to prevent. Never implemented.

- [ADR-002: Shared Contracts with Dual Publishing (Java + TypeScript)](docs/adr/002-shared-contracts-versioning-strategy.md) ✅
  - **Decision**: Create `wallet-shared-contracts` repo that publishes both Maven artifact (for Java services) and NPM package (for frontends)
  - **Strategy**: Backward compatibility by default, semantic versioning, automated TypeScript generation from Java DTOs
  - **Frontend-friendly**: DTO suffix stripped (TransactionDTO → Transaction in TypeScript)
  - **Benefits**: Single source of truth, independent service upgrades, type safety across stack

- [ADR-003: Hybrid Database Strategy with Separate Analytics Database](docs/adr/003-hybrid-database-strategy-with-analytics.md) ✅
  - **Decision**: Use hybrid approach with 3 isolated databases (Vaullet, Transactions, Fraud) + 1 shared database (Support services) + dedicated Reporting database
  - **Isolated**: Vaullet (append-only ledger), Transactions (high volume), Fraud Detection (different tech needs)
  - **Shared**: Limits, Bonus, Subscription, Notification, Audit services share `support_db` with schema-per-service isolation
  - **Reporting**: Separate analytics database populated via Kafka events (CQRS pattern), decouples reporting from operational performance
  - **Benefits**: Critical data isolated, cost-effective, operationally balanced, room to evolve

- [ADR-004: Atomic Balance Reservations in the Ledger](docs/adr/004-atomic-balance-reservations.md) ✅
  - **Decision**: Check-and-reserve is a single atomic Postgres transaction inside Vaullet; no distributed locks
  - **Buckets**: money is typed per grant (`CASH`, `BONUS`, `LOYALTY`, `REFERRAL`) with its own
    withdrawability, expiry and spend priority — so the ledger can express bonus money correctly
  - **Backstop**: `CHECK (posted_balance - held_total >= 0)` makes an overdraft a schema violation
  - **Consequence**: Redis leaves the correctness path entirely

- [ADR-005: Module Composition and Deployment Topology](docs/adr/005-module-composition-and-deployment-topology.md) ✅
  - **Decision**: Single-tenant deployments; modules enabled at build time via per-customer Helm values, not runtime flags
  - **Rule**: a synchronous dependency makes a component core; optional functionality is event-only
  - **Invariant**: no sellable module depends on another, so all 128 module combinations are valid
  - **Consequence**: CI matrix collapses from 128 configurations to 9

See [docs/adr/README.md](docs/adr/README.md) for the complete list of ADRs and how to contribute new ones.

## Next Steps

1. Define detailed API contracts (OpenAPI/Swagger)
2. ~~Design database schemas per service~~ ✅ **Done** - See ADR-003
3. Define Kafka topic naming conventions
4. ~~Set up shared contracts repository~~ ✅ **Done** - See ADR-002
5. Design error handling and retry strategies
6. Plan disaster recovery and data backup
7. ~~Decide on database-per-service vs shared database strategy~~ ✅ **Done** - See ADR-003 (Hybrid approach)
8. Define monitoring and observability requirements
9. Design authentication and authorization strategy — **next**, ADR-006 (local vs federated identity;
   the ADR-005 Category 4 adapter)
10. ~~Decide balance consistency mechanism~~ ✅ **Done** - See ADR-004 (atomic reservations)
11. ~~Decide module composition and deployment topology~~ ✅ **Done** - See ADR-005
12. Define deployment strategy (CI/CD, blue-green, canary) and the per-customer GitOps repo layout
13. Define service-to-service authentication (mTLS vs JWT)