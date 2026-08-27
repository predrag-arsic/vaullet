# Current Discussion State - Vaullet Architecture

**Date**: 2026-08-27
**Next topic**: Authentication strategy (ADR-006)

---

## Product Context

Vaullet is a **private portfolio project built to production-grade B2B standards** — the goal is an
architecture that survives review by a senior engineer, not a system that currently ships.

The commercial model it is designed against:

- **Single-tenant deployments.** Each customer (a wallet operator, initially betting platforms) gets
  their own Kubernetes cluster. No shared infrastructure, no `tenant_id`, no cross-customer data
  path. Isolation is the cluster boundary — stronger than row-level scoping, and often mandatory
  under gambling licences with data-residency conditions.
- **Modules are line items.** A contract determines which modules that customer's cluster runs.
- **Bonus is the flagship.** The core wallet is the land; the bonus platform is the expected upsell.

---

## Decisions Made

| ADR | Decision | Status |
|-----|----------|--------|
| 001 | Redis distributed locking for balance consistency | ⛔ Superseded by 004 |
| 002 | Shared contracts, dual publishing (Java + TypeScript) | ✅ Accepted |
| 003 | Hybrid database strategy with separate analytics DB | ✅ Accepted (revised) |
| 004 | Atomic balance reservations in the ledger | ✅ Accepted |
| 005 | Module composition and deployment topology | ✅ Accepted |

### What ADR-004 changed

ADR-001's lock guarded a balance *read* while the *write* happened asynchronously in Vaullet after
the lock was released — so it did not prevent the overdraft it was written to prevent. Two
withdrawals 50ms apart both passed. Never implemented; corrected on paper.

Replacement: `available = posted − held`. Check-and-reserve is one atomic Postgres transaction inside
Vaullet, with `CHECK (posted_balance - held_total >= 0)` making an overdraft a schema violation.
Redis leaves the correctness path entirely.

Money is typed into **per-grant buckets** (`CASH`, `BONUS`, `LOYALTY`, `REFERRAL`), each with its own
`withdrawable` flag, expiry and spend priority — because a single scalar cannot express "$100, of
which $60 is bonus money with wagering outstanding." With no rewards module deployed, every account
has one `CASH` bucket and the model degrades to a plain balance.

### What ADR-005 established

**The rule**: *a synchronous dependency makes a component core; optional functionality communicates
only through events.* Disabling an event consumer is safe — Kafka doesn't care that a consumer group
is absent. Disabling a synchronous dependency is not, and both default recoveries are wrong
(fail-closed halts transactions, fail-open silently disables a financial control). So the case is
removed rather than handled.

**The invariant**: *no sellable module may depend on another sellable module.* The module graph is a
star, so all 2⁷ = 128 combinations are valid by construction and the CI matrix collapses to 9.

**Categories**:

- **Core**: Transaction (incl. Refund), Vaullet, Auth, Fraud Detection, Limits, Audit, Admin UI
- **Infrastructure**: Kafka, Redis, PostgreSQL, Elasticsearch, Scheduler
- **Sellable**: Bonus ⭐, Loyalty, Referral, Subscription, Risk Management, Notification, Reporting ETL
- **Adapters**: Auth identity provider (local | federated)

Two deliberate overrides: **Audit** is event-driven but core, because an audit trail is a licence
condition rather than a feature. **Scheduler** is infrastructure rather than a module, which removes
Subscription's only sellable-to-sellable dependency.

Enablement is **build-time** (per-customer Helm values), not runtime flags — "not installed" is
demonstrable to a regulator; "flag is false" requires trusting runtime state.

---

## Databases

6 operational PostgreSQL + 1 analytics + Redis + Elasticsearch = **9 components** full, **7** core-only.

| Database | Services | Provisioned |
|----------|----------|-------------|
| `vaullet_db` | Vaullet | Always |
| `transactions_db` | Transaction (incl. Refund) | Always |
| `fraud_db` | Fraud Detection, Risk Management | Always |
| `auth_db` | Auth | Always |
| `support_db` | Limits, Audit, Scheduler (+ Subscription, Notification if sold) | Always, schemas conditional |
| `rewards_db` | Bonus, Loyalty, Referral | Only if a rewards module is sold |
| `datawarehouse_db` | Reporting ETL | Only if sold |

**Boundary that matters**: `rewards_db` holds *rules* — wagering terms, tiers, campaigns. It never
holds balances. Promotional *money* lives in `vaullet_db` buckets, reached only by emitting
`ValueGranted`. Two services must not both believe they know what a user's money is.

---

## Open Questions for ADR-006 (Auth)

1. **Identity model** — does Vaullet own end-user identity, or does the operator federate its
   existing users in via OIDC? ADR-005 assumes both are supported as adapters; ADR-006 has to define
   the internal contract both satisfy.
2. **Admin roles** — what does the operator's own staff need? Fraud reviewer, support agent, finance,
   super-admin? Permissions model follows from this.
3. **Service-to-service auth** — mTLS or JWT between services inside the cluster.
4. **Session strategy** — JWT lifetime, refresh, revocation. Revocation is the hard part with JWTs and
   interacts with the `account.locked` event from Risk Management.

---

## Known Gaps

- **Per-customer cost floor is high** — core alone is 7 services plus Kafka, Redis, Elasticsearch and
  5 Postgres databases. Small operators may be unprofitable on this topology. ADR-005 Alternative 3
  (modular monolith) is the documented candidate for a low-cost tier.
- **Elasticsearch is core by inheritance** — Fraud Detection is core and ADR-003 gives it ES. Worth
  revisiting whether pattern search can degrade to Postgres-only.
- **15 services is a lot for a solo project.** The module boundaries have commercial justification, but
  "sellable unit" and "separately deployed service" are not the same thing.

---

## Files

- Architecture overview: `/arch.md`
- ADRs: `/docs/adr/00{1..5}-*.md`
- Database grouping: `/docs/database-grouping-strategy.md`
- Shared contracts example: `/docs/shared-contracts-example.md`
