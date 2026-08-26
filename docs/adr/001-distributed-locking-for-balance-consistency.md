# ADR-001: Use Redis Distributed Locking for Balance Consistency

## Status

**Accepted** (2026-08-26)

## Context

The Vaullet walleting system must prevent overdrafts and maintain balance consistency when processing concurrent transactions. This is a critical requirement for a financial system.

### The Problem

When multiple withdrawal requests arrive simultaneously for the same user, we must ensure:
1. Both transactions cannot succeed if the sum exceeds the available balance
2. No race conditions lead to negative balances
3. System can handle high transaction volume (target: >1000 TPS across all users)

### Example Scenario

User has $100 balance. Two concurrent requests:
- Request A: Withdraw $60
- Request B: Withdraw $60

Only one should succeed. Both succeeding would result in -$20 balance (unacceptable).

### Requirements

- **Strong consistency**: No overdrafts allowed (zero tolerance)
- **High throughput**: Target >1000 TPS system-wide
- **Low latency**: Transaction approval within 200-500ms
- **Horizontal scalability**: Ability to add capacity as volume grows
- **Operational simplicity**: Prefer proven, well-understood approaches

## Decision

We will use **Redis distributed locks (Redlock algorithm)** to ensure balance consistency during transaction processing.

### How It Works

1. Transaction Service attempts to acquire a Redis lock: `LOCK("balance:user:{userId}")`
2. Lock acquired with 500ms timeout (prevents indefinite waiting)
3. Read current balance from Vaullet service (REST call)
4. Perform balance check: `balance >= requestedAmount`
5. If sufficient, proceed with transaction; otherwise reject
6. Update balance and complete transaction
7. Release Redis lock: `UNLOCK("balance:user:{userId}")`

### Lock Configuration

- **Lock scope**: Per-user (lock key: `balance:user:{userId}`)
- **Timeout**: 500ms maximum hold time
- **Retry**: 3 attempts with exponential backoff (10ms, 50ms, 100ms)
- **Redis deployment**: Redis Cluster (3 master nodes, 3 replicas)
- **Algorithm**: Redlock for distributed consensus

### Integration Points

```
Transaction Service (receives request)
    ↓
    1. Synchronous: Auth Service (validate token)
    2. Synchronous: Fraud Detection (analyze risk)
    3. Acquire Redis Lock (balance:user:{userId})
    4. Synchronous: Limits Service (check limits)
    5. Synchronous: Vaullet (check balance)
    6. Decision: APPROVE / REJECT
    7. Release Redis Lock
    ↓
If approved: Publish transaction.created event
```

## Consequences

### Positive Consequences

✅ **No overdrafts**: Distributed lock guarantees only one transaction processes at a time per user
✅ **Proven approach**: Redis locking is well-understood and battle-tested
✅ **Better than DB locks**: Faster lock acquisition (~1-2ms vs 5-10ms)
✅ **Scales across users**: Different users can transact in parallel
✅ **Simple to implement**: Spring Boot + Redisson library provides ready-made solution
✅ **Good enough throughput**: Can handle 1000-2000 TPS system-wide
✅ **Meets requirements**: Balances consistency, performance, and operational simplicity

### Negative Consequences

⚠️ **Per-user serialization**: Single user limited to ~20-50 TPS (acceptable for most use cases)
⚠️ **Redis dependency**: Redis becomes critical path (mitigated by clustering)
⚠️ **Lock timeout risks**: If operation exceeds 500ms, lock expires (need monitoring)
⚠️ **Network partitions**: Split-brain scenarios possible (Redlock handles this)
⚠️ **Not infinite scale**: Will hit limits at very high per-user volumes (>50 TPS/user)

### Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Redis cluster failure | No transactions can proceed | Redis Cluster with auto-failover, monitoring, alerts |
| Lock timeout during processing | Potential race condition | Set timeout generously (500ms), monitor p95 latency, optimize slow operations |
| Single user high volume | Serialization bottleneck | Rate limiting per user (max 50 TPS), queue excess requests |
| Deadlocks | Transaction failures | Implement lock timeout, retry logic with exponential backoff |
| Network partition | Inconsistent locks | Use Redlock algorithm across 3+ independent Redis nodes |

## Alternatives Considered

### Alternative 1: Database Row-Level Locking (PostgreSQL)

**Description**: Use `SELECT ... FOR UPDATE` to lock balance rows in Vaullet's database.

**Pros**:
- Simple to implement
- ACID guarantees built-in
- No additional infrastructure (Redis)

**Cons**:
- Slower (5-10ms lock acquisition vs 1-2ms for Redis)
- Database becomes bottleneck
- Harder to scale horizontally
- Risk of connection pool exhaustion under load

**Why Rejected**: Won't achieve >1000 TPS target due to database serialization bottleneck.

---

### Alternative 2: Eventual Consistency (No Locking)

**Description**: Allow concurrent transactions, detect overdrafts asynchronously, and reverse transactions via compensating events.

**Pros**:
- Extremely high throughput (5000+ TPS)
- Scales horizontally without limits
- Low latency (no waiting for locks)

**Cons**:
- Temporary overdrafts possible
- Poor user experience (approved then reversed)
- Complex reconciliation logic
- Not suitable for financial compliance requirements

**Why Rejected**: Zero tolerance for overdrafts is a hard requirement. Temporary inconsistency unacceptable for financial system.

---

### Alternative 3: Kafka Partitioning + Stateful Stream Processing

**Description**: Partition transactions by userId into Kafka topics. Each partition consumed by a single stateful processor maintaining in-memory balances.

**Pros**:
- Very high throughput (5000+ TPS system-wide)
- Strong consistency without distributed locks
- Scales by adding partitions
- Natural fit for event-driven architecture

**Cons**:
- Requires stateful service (operational complexity)
- More complex to implement and operate
- State management and recovery needed
- Kafka becomes critical dependency
- Over-engineered for current requirements

**Why Rejected**: Higher complexity than needed for initial launch. Redis locking is simpler and meets current requirements. **Can migrate to this later if needed** (documented as future path).

---

### Alternative 4: Event Sourcing with Optimistic Concurrency

**Description**: Store all balance changes as events with version numbers. Use optimistic locking (compare-and-swap) when appending events.

**Pros**:
- Full audit trail
- Can replay events
- No distributed locks
- Good scalability

**Cons**:
- Requires retry logic for conflicts
- More complex implementation
- Event store becomes critical
- Learning curve for team

**Why Rejected**: Over-engineered for current needs. Adds complexity without significant benefit over Redis locking.

## Implementation Notes

### Technology Stack

- **Redis**: Redis Cluster 7.x (3 masters, 3 replicas)
- **Java Library**: Redisson 3.x (implements Redlock algorithm)
- **Spring Boot**: Integration via `@RedissonClient` bean
- **Deployment**: Kubernetes StatefulSet for Redis

### Code Example

```java
@Service
public class TransactionService {

    @Autowired
    private RedissonClient redissonClient;

    public TransactionResponse processTransaction(TransactionRequest request) {
        String lockKey = "balance:user:" + request.getUserId();
        RLock lock = redissonClient.getLock(lockKey);

        try {
            // Try to acquire lock with 500ms timeout
            boolean acquired = lock.tryLock(100, 500, TimeUnit.MILLISECONDS);

            if (!acquired) {
                return TransactionResponse.rejected("System busy, please retry");
            }

            // Critical section - only one thread per user
            BigDecimal balance = vaulletService.getBalance(request.getUserId());

            if (balance.compareTo(request.getAmount()) < 0) {
                return TransactionResponse.rejected("Insufficient balance");
            }

            // Proceed with transaction
            Transaction txn = createTransaction(request);
            publishEvent("transaction.created", txn);

            return TransactionResponse.approved(txn.getId());

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return TransactionResponse.error("Transaction interrupted");
        } finally {
            // Always release lock
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

### Migration Path to Higher Throughput (Future)

If we hit Redis locking limits (>2000 TPS or >50 TPS per user), we can migrate to **Alternative 3** (Kafka Partitioning):

1. Introduce Transaction Processor Service (stateful)
2. Partition Kafka topic by userId
3. Migrate balance checks to in-memory state
4. Remove Redis locking dependency
5. Maintain same API contracts (transparent to consumers)

**Estimated effort**: 2-3 sprints
**Risk**: Medium (well-documented pattern used by Stripe, Square)

### Monitoring and Alerts

Key metrics to track:

- `redis.lock.acquisition.time.p95` - Should be <5ms
- `redis.lock.timeout.count` - Should be near zero
- `transaction.processing.time.p95` - Should be <200ms
- `redis.cluster.health` - All nodes healthy
- `transactions.per.user.per.second` - Alert if any user >40 TPS (approaching limit)

### Rollout Plan

1. **Phase 1**: Deploy Redis Cluster to staging
2. **Phase 2**: Implement locking in Transaction Service
3. **Phase 3**: Load test with 2000 TPS across 100 users
4. **Phase 4**: Production deployment with feature flag
5. **Phase 5**: Gradual rollout (10% → 50% → 100% traffic)

## References

- [Redlock Algorithm](https://redis.io/docs/manual/patterns/distributed-locks/)
- [Redisson Documentation](https://github.com/redisson/redisson/wiki)
- [Designing Data-Intensive Applications (Kleppmann)](https://dataintensive.net/) - Chapter 9: Consistency and Consensus
- Spike: Redis lock performance test results (see `docs/spikes/redis-lock-benchmark.md`)
- Related: Future ADR for Kafka partitioning migration (when needed)

---

**Date**: 2026-08-26
**Author**: Predrag
**Reviewers**: TBD
**Next Review**: 2027-02-26 (6 months) - Evaluate if migration to Kafka partitioning is needed