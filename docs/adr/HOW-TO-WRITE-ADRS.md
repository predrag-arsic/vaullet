# How to Write ADRs (Architecture Decision Records)

## Quick Start

1. **Copy the template**:
   ```bash
   cp docs/adr/template.md docs/adr/002-your-decision-title.md
   ```

2. **Fill in the sections** (see guidelines below)

3. **Update the index** in `docs/adr/README.md`

4. **Submit for review** (PR or team discussion)

## When to Write an ADR

Write an ADR when you're making a decision that:
- ❗ **Affects multiple services** (e.g., API contracts, communication patterns)
- 🏗️ **Impacts architecture** (e.g., database choice, caching strategy)
- 🔒 **Has significant trade-offs** (e.g., consistency vs availability)
- 💰 **Has cost implications** (e.g., cloud services, infrastructure)
- 🔄 **Is hard to reverse** (e.g., polyrepo vs monorepo, event sourcing)
- 📈 **Affects scalability or performance** (e.g., load balancing, partitioning)
- 🔐 **Impacts security or compliance** (e.g., encryption, audit logging)

## When NOT to Write an ADR

Don't write an ADR for:
- ✏️ Small implementation details (use code comments)
- 🐛 Bug fixes (use commit messages)
- 📝 Coding style decisions (use linter configs)
- 🧪 Experimental spikes (use spike reports)
- 📚 How-to guides (use documentation)

> **A note on the examples below.** They are drawn from
> [ADR-001](001-distributed-locking-for-balance-consistency.md), which is **superseded and was wrong**:
> its Redis lock guarded a balance read while the write happened asynchronously after the lock was
> released, so it never prevented the overdraft it was written to prevent. It is quoted here purely as
> an illustration of ADR *format* — decisive language, explicit trade-offs, honest alternatives.
>
> That it reads so convincingly while being incorrect is itself the lesson: a well-written ADR is not a
> correct one. See [ADR-004](004-atomic-balance-reservations.md) for the correction, and note that the
> format did its job — the reasoning was written down plainly enough to be checked and found wrong.

## Writing Guidelines

### 1. Title

**Good**:
- "Use Redis Distributed Locking for Balance Consistency"
- "Adopt Kafka for Event-Driven Communication"
- "Implement Polyrepo Structure for Microservices"

**Bad**:
- "Redis" (too vague)
- "We need to make a decision about locking" (not decisive)
- "Fix the concurrency problem" (sounds like a bug, not architecture)

### 2. Context Section

**What to include**:
- The problem you're solving
- Why it matters (business/technical impact)
- Constraints you're operating under
- Current state (if replacing something)

**Example**:
```markdown
## Context

The Vaullet system must prevent overdrafts when processing concurrent
transactions. Without proper coordination, two simultaneous withdrawals
could both succeed even when the sum exceeds the available balance.

We need to support:
- 1000+ TPS system-wide
- Zero tolerance for overdrafts
- Sub-200ms transaction latency

Current state: No locking mechanism exists (prototype phase).
```

### 3. Decision Section

**Be specific**. Use active voice: "We will..."

**Good**:
```markdown
## Decision

We will use Redis distributed locks (Redlock algorithm) to ensure
balance consistency during transaction processing.

Lock configuration:
- Lock scope: Per-user (key: `balance:user:{userId}`)
- Timeout: 500ms
- Retry: 3 attempts with exponential backoff
- Deployment: Redis Cluster (3 masters, 3 replicas)
```

**Bad**:
```markdown
## Decision

We should probably use some kind of locking mechanism to prevent
race conditions. Maybe Redis or something similar.
```

### 4. Consequences Section

**Be honest about trade-offs**. Every decision has downsides.

**Structure**:
- ✅ Positive consequences (benefits)
- ⚠️ Negative consequences (trade-offs)
- 🎯 Risks and mitigations

**Example**:
```markdown
## Consequences

### Positive Consequences

✅ Prevents overdrafts (strong consistency)
✅ Faster than database locks (1-2ms vs 5-10ms)
✅ Can handle 1000+ TPS system-wide

### Negative Consequences

⚠️ Per-user serialization (max ~50 TPS per user)
⚠️ Redis becomes critical dependency
⚠️ Lock timeout can cause race conditions if operation slow

### Risks

| Risk | Mitigation |
|------|------------|
| Redis cluster failure | Auto-failover, monitoring, alerts |
| Lock timeout | Set timeout generously, optimize operations |
```

### 5. Alternatives Section

**Document what you considered and WHY you rejected it**.

This is crucial for future readers who might wonder "Why didn't we just use X?"

**Example**:
```markdown
## Alternatives Considered

### Alternative 1: Database Row-Level Locking

Description: Use PostgreSQL `SELECT FOR UPDATE` to lock balance rows.

Pros:
- Simple to implement
- No additional infrastructure

Cons:
- Slower (5-10ms locks)
- Database bottleneck
- Won't achieve >1000 TPS

**Why Rejected**: Performance requirements not met.

### Alternative 2: Eventual Consistency

Description: No locking, detect overdrafts async, reverse transactions.

Pros:
- Very high throughput
- Scales infinitely

Cons:
- Temporary overdrafts possible
- Poor UX (approved then reversed)

**Why Rejected**: Zero tolerance for overdrafts is hard requirement.
```

### 6. Implementation Notes

Include:
- Key libraries/frameworks
- Code snippets (if helpful)
- Configuration details
- Migration path (if replacing existing approach)
- Monitoring/observability requirements

## ADR Lifecycle

### Proposed → Accepted

```markdown
## Status

**Proposed** (2026-08-26)

Discussion thread: [link to Slack/GitHub discussion]
```

After team review and approval:

```markdown
## Status

**Accepted** (2026-08-30)
```

### Accepted → Deprecated

When you realize the decision is no longer optimal but haven't replaced it:

```markdown
## Status

**Deprecated** (2027-01-15)

Reason: Redis distributed locks hitting performance limits at 3000+ TPS.
Plan: Migrate to Kafka partitioning (ADR-012 in progress).
```

### Deprecated → Superseded

When you've made a new decision that replaces this one:

```markdown
## Status

**Superseded** (2027-02-10) by [ADR-012: Kafka Partitioning for High Throughput](012-kafka-partitioning.md)
```

## Tips for Great ADRs

### ✅ Do

- **Be specific**: "500ms timeout" not "short timeout"
- **Include numbers**: "1000 TPS" not "high throughput"
- **Think long-term**: "Can we understand this in 2 years?"
- **Document alternatives**: Future you will thank you
- **Be honest**: Don't hide trade-offs or risks
- **Link references**: Papers, blog posts, docs
- **Use examples**: Code snippets, diagrams

### ❌ Don't

- **Be vague**: "We'll use a database" (which one? why?)
- **Skip alternatives**: Readers will wonder "why not X?"
- **Ignore consequences**: Every decision has trade-offs
- **Write novels**: Keep it concise (1-2 pages max)
- **Use jargon**: Explain acronyms and technical terms
- **Make it permanent**: ADRs can be superseded

## Examples from Industry

### Good ADRs to Study

1. **Spotify**: [ADRs in Engineering Culture](https://engineering.atspotify.com/2020/04/software-visualization-challenge-accepted/)
2. **GitHub**: [ADR for Actions Architecture](https://github.blog/2019-11-13-universe-day-one/)
3. **AWS**: [Why we chose DynamoDB](https://www.allthingsdistributed.com/2012/01/amazon-dynamodb.html)

### Common ADR Topics

- Database selection (SQL vs NoSQL, specific DB choice)
- Event-driven architecture patterns
- API design (REST vs GraphQL vs gRPC)
- Authentication/Authorization approach
- Caching strategy
- Deployment strategy (blue-green, canary, etc.)
- Monitoring and observability tools
- Testing strategy (unit, integration, e2e)

## Tools

### CLI Tools (Optional)

```bash
# Install ADR tools
npm install -g adr-log

# Create new ADR
adr new "Use Redis for Caching"

# Generate table of contents
adr generate toc > docs/adr/README.md
```

### IDE Integration

- VSCode extension: "ADR Manager"
- IntelliJ plugin: "ADR Tools"

## Questions?

- See existing ADRs in this repo for examples
- Check the [template](template.md)
- Discuss in team chat before writing

Remember: **The goal is clarity, not perfection**. Write the ADR, get feedback, iterate.