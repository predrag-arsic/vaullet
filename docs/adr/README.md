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
3. **Deprecated**: Decision is outdated but not yet replaced
4. **Superseded**: Replaced by a newer decision (link to new ADR)

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
| [001](001-distributed-locking-for-balance-consistency.md) | Use Redis Distributed Locking for Balance Consistency | ✅ Accepted | 2026-08-26 |
| [002](002-shared-contracts-versioning-strategy.md) | Shared Contracts with Dual Publishing (Java + TypeScript) | ✅ Accepted | 2026-08-27 |
| [003](003-hybrid-database-strategy-with-analytics.md) | Hybrid Database Strategy with Separate Analytics Database | ✅ Accepted | 2026-08-27 |

## Upcoming Decisions

Track decisions that need to be made:

- [ ] Database schema design for Vaullet (append-only ledger)
- [ ] Kafka topic naming conventions
- [ ] Event schema versioning strategy
- [ ] Polyrepo CI/CD pipeline approach
- [ ] Service-to-service authentication (mutual TLS vs JWT)
- [ ] API versioning strategy
- [ ] Multi-currency support approach
- [ ] Data retention and archival policy

## References

- [ADR Template](template.md)
- [Architecture Overview](../../arch.md)
- [GitHub - ADR Tools](https://github.com/npryce/adr-tools)
- [Thoughtworks - Documenting Architecture Decisions](https://www.thoughtworks.com/radar/techniques/lightweight-architecture-decision-records)