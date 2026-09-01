# Architecture Decision Records (ADRs)

This directory contains Architecture Decision Records (ADRs) for the Vaullet walleting system.

## What are ADRs?

ADRs document important architectural decisions made during the development of the system. Each ADR captures:
- **Context**: Why we needed to make a decision
- **Decision**: What we chose to do
- **Consequences**: Trade-offs and implications
- **Alternatives**: What else we considered and why we rejected them

## Why ADRs?

- 📚 **Institutional memory**: Understand why decisions were made
- 🆕 **Onboarding**: Help new team members understand the system
- 🔄 **Evolution**: Easy to revisit and supersede when context changes
- 🤔 **Decision-making**: Forces us to think through trade-offs

## ADR Lifecycle

1. **Proposed**: Decision is being discussed
2. **Accepted**: Decision has been made and implemented
3. **Partially Accepted**: Some parts settled, others still under review — the ADR states which
4. **Deprecated**: Decision is outdated but not yet replaced
5. **Superseded**: Replaced by a newer decision (link to new ADR)

## How to Create an ADR

1. Copy `template.md` to a new file
2. Name it: `XXX-short-title.md` (e.g., `002-kafka-topic-naming.md`)
3. Use the next sequential number
4. Fill in all sections
5. Add it to the index below
6. Submit for review via pull request

## Index of ADRs

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [001](001-distributed-locking-for-balance-consistency.md) | Use Redis Distributed Locking for Balance Consistency | ⛔ Superseded by [004](004-atomic-balance-reservations.md) | 2026-08-26 |
| [002](002-shared-contracts-versioning-strategy.md) | Shared Contracts with Dual Publishing (Java + TypeScript) | ✅ Accepted | 2026-08-27 |
| [003](003-hybrid-database-strategy-with-analytics.md) | Hybrid Database Strategy with Separate Analytics Database | ✅ Accepted | 2026-08-27 |
| [004](004-atomic-balance-reservations.md) | Atomic Balance Reservations in the Ledger | ✅ Accepted | 2026-08-27 |
| [005](005-module-composition-and-deployment-topology.md) | Module Composition and Deployment Topology | ✅ Accepted | 2026-08-27 |
| [006](006-authentication-and-identity.md) | Authentication and Identity | ✅ Accepted | 2026-08-27 |
| [007](007-kafka-topics-and-event-schema-evolution.md) | Kafka Topic Naming and Event Schema Evolution | ✅ Accepted | 2026-08-27 |
| [008](008-service-to-service-authentication.md) | Service-to-Service Authentication | 🟡 Partially Accepted — mesh vs mesh-free under review | 2026-08-27 |

## Upcoming Decisions

Track decisions that need to be made:

- [ ] Database schema design for Vaullet (append-only ledger)
- [ ] **Manual ledger adjustments** (corrections, goodwill credits, dispute resolution) — the need exists, the endpoint does not; surfaced in the 2026-09-01 review
- [ ] Polyrepo CI/CD pipeline approach
- [ ] API versioning strategy
- [ ] Multi-currency within a single deployment (currently one currency per deployment — ADR-004)
- [ ] **ADR-008 Layer 2/3**: mesh (Linkerd) vs mesh-free (CNI encryption + Keycloak tokens everywhere) — Alternative 6
- [ ] Data retention and archival policy
- [ ] Human/operational access control (kubectl, database credentials, break-glass)

## References

- [ADR Template](template.md)
- [Architecture Overview](../../arch.md)
- [GitHub - ADR Tools](https://github.com/npryce/adr-tools)
- [Thoughtworks - Documenting Architecture Decisions](https://www.thoughtworks.com/radar/techniques/lightweight-architecture-decision-records)