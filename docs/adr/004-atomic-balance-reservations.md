# ADR-004: Atomic Balance Reservations in the Ledger

## Status

**Accepted** (2026-08-27)
**Revised** (2026-08-27) — bucketed balances, to support independently sellable rewards modules

Supersedes [ADR-001](001-distributed-locking-for-balance-consistency.md).

## Context

[ADR-001](001-distributed-locking-for-balance-consistency.md) decided to prevent overdrafts using Redis distributed locks held by Transaction Service around a balance check. Reviewing that decision before implementation, we found it does not prevent the failure it was written to prevent.

### The Flaw in ADR-001

ADR-001's critical section reads the balance from Vaullet, decides, publishes an event, and releases the lock:

```java
lock.tryLock(...);
BigDecimal balance = vaulletService.getBalance(userId);  // READ
if (balance.compareTo(amount) < 0) reject();
publishEvent("transaction.created", txn);                // WRITE happens later, elsewhere
lock.unlock();
```

Vaullet creates the ledger entry **asynchronously**, when it consumes `transaction.created` (see `arch.md`, Vaullet event consumption). The balance mutation therefore happens *after* the lock is released, in a different service, over a Kafka hop.

The lock serializes the read. Nothing serializes the write. The state the lock is meant to protect is modified outside the critical section, so the lock protects nothing.

### Failure Trace

Account holds $100. Two withdrawals of $60, arriving 50ms apart — not even concurrent:

| t | Request A | Request B | Ledger state |
|---|-----------|-----------|--------------|
| 0ms | acquires lock | — | balance $100 |
| 1ms | reads balance: **$100** | — | balance $100 |
| 2ms | $100 ≥ $60 → **approve** | — | balance $100 |
| 3ms | publishes `transaction.created` | — | balance $100 |
| 4ms | releases lock | — | balance $100 |
| 50ms | — | acquires lock (free) | balance $100 |
| 51ms | — | reads balance: **$100** | balance $100 |
| 52ms | — | $100 ≥ $60 → **approve** | balance $100 |
| ~200ms | Vaullet consumes A | | balance $40 |
| ~250ms | | Vaullet consumes B | balance **-$20** |

This is the exact scenario in ADR-001's opening paragraph. Any lock hold time shorter than the Kafka round-trip leaves the window open, and the whole point of a short lock TTL was to keep hold times low — the mitigation and the bug pull in the same direction.

Two secondary problems in ADR-001, noted for completeness:

- **TTL vs latency budget.** A 500ms lock TTL inside a stated 200–500ms approval budget means the lock expires mid-operation at p95, while the operation continues believing it holds it.
- **Redlock vs Redis Cluster.** Redlock requires N *independent* Redis masters. ADR-001 specifies a single Redis Cluster (3 masters + 3 replicas), where Redisson's `RLock` is a single-master lock replicated asynchronously — a lock that can be silently lost on failover. The named algorithm and the named deployment are incompatible.

### Constraints (unchanged from ADR-001)

- Zero tolerance for overdrafts
- >1000 TPS system-wide, sub-200ms p95 approval latency
- One tenant per deployment (see ADR-005) — isolation is at the cluster boundary, so balances are scoped by `account_id` alone
- The ledger must express non-cash money (bonus, loyalty, referral) with distinct withdrawal rules, while remaining correct when no rewards module is deployed
- Prefer proven, operationally simple mechanisms

## Decision

We will make **check-and-reserve a single atomic operation inside Vaullet**, and remove distributed locking from the balance path entirely.

Available balance becomes `posted_balance − held_total`. Money is *held* synchronously at approval time and *settled* asynchronously at completion. A hold is a real row, committed in the same database transaction as the check that authorized it.

### Balance buckets

A single `posted_balance` scalar cannot express "the account holds $100, but $60 is bonus money with wagering still outstanding and cannot be withdrawn." Since Bonus is the flagship module — and Loyalty and Referral sell independently of it — the ledger models money as **typed buckets**.

`CASH` always exists. Rewards modules create additional buckets by emitting grant events. With no rewards module deployed, every account has exactly one `CASH` bucket and the model degrades to a plain balance.

Buckets are per *grant*, not per type: a user can hold three separate `BONUS` buckets with different wagering requirements and expiry dates, because that is what the terms of three separate promotions actually mean.

**The ledger is authoritative about amounts and dumb about policy.** It stores `wagering_remaining` but never decides how a wager contributes to it — that is the Bonus module's rule, applied by emitting events. This keeps promotional logic out of the service that must stay correct when Bonus isn't deployed.

### Schema (vaullet_db)

```sql
-- Immutable journal. Source of truth. Append-only, never updated.
CREATE TABLE ledger_entries (
    entry_id        UUID PRIMARY KEY,
    account_id      UUID        NOT NULL,
    bucket_id       UUID        NOT NULL REFERENCES balance_buckets(bucket_id),
    direction       TEXT        NOT NULL CHECK (direction IN ('DEBIT','CREDIT')),
    amount          NUMERIC(20,4) NOT NULL CHECK (amount > 0),
    reservation_id  UUID        NULL REFERENCES reservations(reservation_id),
    transaction_id  UUID        NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ON ledger_entries (transaction_id, bucket_id, direction);

-- Per-account aggregate AND the single concurrency anchor.
-- Every balance-changing operation locks this row first, before touching buckets.
-- Derived: rebuildable by replaying ledger_entries.
CREATE TABLE account_balances (
    account_id      UUID          PRIMARY KEY,
    posted_balance  NUMERIC(20,4) NOT NULL DEFAULT 0,   -- sum over buckets
    held_total      NUMERIC(20,4) NOT NULL DEFAULT 0 CHECK (held_total >= 0),
    updated_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),
    CHECK (posted_balance - held_total >= 0)   -- overdraft is a schema violation
);

-- One row per grant. CASH is a singleton per account.
CREATE TABLE balance_buckets (
    bucket_id          UUID PRIMARY KEY,
    account_id         UUID          NOT NULL REFERENCES account_balances(account_id),
    bucket_type        TEXT          NOT NULL CHECK (bucket_type IN ('CASH','BONUS','LOYALTY','REFERRAL')),
    source_module      TEXT          NULL,        -- NULL for CASH
    grant_id           UUID          NULL,        -- idempotency anchor for the granting event
    posted_balance     NUMERIC(20,4) NOT NULL DEFAULT 0,
    held_total         NUMERIC(20,4) NOT NULL DEFAULT 0 CHECK (held_total >= 0),
    withdrawable       BOOLEAN       NOT NULL,
    wagering_remaining NUMERIC(20,4) NOT NULL DEFAULT 0,
    spend_priority     SMALLINT      NOT NULL,    -- lower spends first
    expires_at         TIMESTAMPTZ   NULL,
    CHECK (posted_balance - held_total >= 0)
);
CREATE UNIQUE INDEX ON balance_buckets (grant_id) WHERE grant_id IS NOT NULL;
CREATE UNIQUE INDEX ON balance_buckets (account_id) WHERE bucket_type = 'CASH';
CREATE INDEX ON balance_buckets (account_id, spend_priority);

-- Short-lived authorization holds.
CREATE TABLE reservations (
    reservation_id  UUID PRIMARY KEY,
    account_id      UUID          NOT NULL,
    amount          NUMERIC(20,4) NOT NULL CHECK (amount > 0),
    state           TEXT          NOT NULL CHECK (state IN ('HELD','SETTLED','RELEASED','EXPIRED')),
    idempotency_key TEXT          NOT NULL,
    expires_at      TIMESTAMPTZ   NOT NULL,
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),
    UNIQUE (idempotency_key)
);
CREATE INDEX ON reservations (state, expires_at) WHERE state = 'HELD';

-- Which buckets a hold draws from, and how much from each.
CREATE TABLE reservation_allocations (
    reservation_id  UUID          NOT NULL REFERENCES reservations(reservation_id),
    bucket_id       UUID          NOT NULL REFERENCES balance_buckets(bucket_id),
    amount          NUMERIC(20,4) NOT NULL CHECK (amount > 0),
    PRIMARY KEY (reservation_id, bucket_id)
);
```

The two `CHECK (posted_balance - held_total >= 0)` constraints are the backstop, at both aggregate and bucket level: even with wrong application logic, Postgres refuses to record an overdraft. ADR-001 had no equivalent — nothing in its schema could have caught the bug.

`withdrawable_balance` is `SUM(posted_balance - held_total) WHERE withdrawable`, computed on read. It is deliberately not stored: it is a question, not a fact.

### Reserve (synchronous, one transaction)

```sql
BEGIN;

SELECT posted_balance, held_total
  FROM account_balances
 WHERE account_id = $1
   FOR UPDATE;              -- the ONLY lock taken; serializes this account only

-- if (posted_balance - held_total) < $amount: ROLLBACK, return 409 INSUFFICIENT_FUNDS

SELECT bucket_id, posted_balance - held_total AS available
  FROM balance_buckets
 WHERE account_id = $1
   AND (expires_at IS NULL OR expires_at > now())
   AND available > 0
 ORDER BY spend_priority;   -- allocate greedily across buckets

INSERT INTO reservations (...) VALUES (..., 'HELD', $key, now() + interval '5 minutes');
INSERT INTO reservation_allocations (...) VALUES ...;   -- one row per bucket touched

UPDATE balance_buckets  SET held_total = held_total + $alloc WHERE bucket_id = ANY($allocated);
UPDATE account_balances SET held_total = held_total + $amount, updated_at = now()
 WHERE account_id = $1;

COMMIT;
```

Read and write are in the same transaction, on the same rows, in the same service. There is no window.

**Deadlock safety**: every operation takes `account_balances` `FOR UPDATE` *first* and holds no other lock across a network call. Bucket rows are only ever touched while that account row is held, so two concurrent operations on the same account serialize on one row, and operations on different accounts never contend. Multi-row bucket updates cannot deadlock because they are always reached through the same single gate.

**Spend priority** is set by the granting module and is a business decision (typically: expiring-soonest promotional money first, cash last, so users don't forfeit bonuses). The ledger sorts; it does not choose.

### Idempotency

`UNIQUE (idempotency_key)` — keyed on `transaction_id`. A retried reserve (client timeout, Transaction Service restart) hits the constraint and returns the *existing* reservation rather than placing a second hold. This is not optional: without it, retries double-hold and reject legitimate transactions.

Grants use `UNIQUE (grant_id)` on `balance_buckets` for the same reason — a redelivered `ValueGranted` event must not mint money twice. At-least-once delivery makes this mandatory, not defensive.

### Grants (asynchronous)

Bonus, Loyalty, and Referral each emit a common `ValueGranted` event (shared contract, per ADR-002). Vaullet consumes it and, in one transaction: insert a `balance_buckets` row, insert a `CREDIT` ledger entry, increment `account_balances.posted_balance`.

Grants are event-driven, not synchronous, which is what keeps all three modules independently removable — the rule from ADR-005 that optional functionality lives on the event side. Vaullet consuming an event type that no deployed module emits costs nothing.

### Settle and Release

On `transaction.completed`, in one transaction: insert one `ledger_entries` row per allocated bucket, decrement `held_total` on those buckets and on the account, apply amounts to `posted_balance`, mark the reservation `SETTLED`. Idempotent via the unique index on `(transaction_id, bucket_id, direction)`.

On `transaction.failed` or cancellation: decrement `held_total` at both levels, mark `RELEASED`. No ledger entry — money that never moved leaves no journal record.

### Expiry

A sweeper releases abandoned holds:

```sql
UPDATE reservations SET state = 'EXPIRED'
 WHERE state = 'HELD' AND expires_at < now()
 RETURNING reservation_id;   -- then release allocations, bucket and account held_total
```

Settlement uses `WHERE state = 'HELD'` and expiry uses `WHERE state = 'HELD' AND expires_at < now()`, so exactly one wins. If expiry wins, settlement affects zero rows and Vaullet publishes `ledger.settlement.rejected`; Transaction Service fails the transaction. With a 5-minute expiry against a sub-second happy path, this is a rare, loud, correct outcome rather than a silent overdraft.

Expired *buckets* are separate from expired *reservations*: a promotional bucket past `expires_at` is excluded from allocation and swept to zero with a `DEBIT` entry, so forfeiture is journalled rather than silent.

### Redis

Redis stays — for balance-read caching, rate limiting, and sessions. It is no longer load-bearing for correctness. Losing Redis degrades latency; it can no longer cause an overdraft.

## Consequences

### Positive Consequences

✅ **Overdrafts are actually prevented** — the invariant is enforced by an ACID transaction and a `CHECK` constraint, not by a lock whose scope excludes the write
✅ **One fewer critical dependency** — Redis leaves the correctness path; no lock TTL to tune against a latency budget, no split-brain, no lock lost on failover
✅ **Idempotent by construction** — a unique constraint, not a convention
✅ **Crash-safe** — a crashed caller leaves a hold that expires cleanly; ADR-001's design left a lock that expired *while the operation continued*
✅ **Holds are a product feature** — "pending balance" is something wallet users expect to see; ADR-001 had nowhere to represent in-flight money
✅ **Per-account serialization only** — different accounts never contend, so system-wide throughput is unaffected
✅ **Rewards modules stay independent** — Bonus, Loyalty and Referral each grant into the ledger without knowing about one another, and any subset can be deployed
✅ **Withdrawal rules are enforced where the money lives** — `withdrawable` is a ledger property, so no module can accidentally cash out locked promotional funds

### Negative Consequences

⚠️ **Vaullet exposes a synchronous command API.** It was described as append-only and producer-only (`arch.md`). It now accepts `POST /reservations` on the request path. `arch.md` must be updated; "producer-only" was never compatible with being the authority on available balance.

⚠️ **`account_balances` is mutable state inside the "immutable" service.** To be precise about what append-only means here: `ledger_entries` is the immutable journal and remains the source of truth. `account_balances` and `reservations` are a derived projection, maintained transactionally and fully rebuildable by replaying the journal. This is the standard journal-plus-balances split every real ledger uses; the previous docs overstated the invariant.

⚠️ **Vaullet is now on the latency critical path** for every transaction, and its availability bounds transaction availability.

⚠️ **Bucket allocation is materially more complex than a single balance** — greedy allocation across N buckets, per-bucket holds, and per-bucket settlement are the price of a real bonus platform. A deployment with no rewards module carries the schema but exercises only the one-bucket path.

⚠️ **Two-phase lifecycle to implement** — reserve/settle/release/expire is more moving parts than a lock-and-check, and every path needs a test.

### Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Hot account (house/merchant account) becomes a serialization point | Throughput collapse on that account | Balance-shard high-volume system accounts into N sub-accounts; skip holds for accounts that cannot go negative |
| Holds accumulate from stuck transactions | Legitimate transactions rejected as insufficient funds | 5-minute expiry, sweeper every 30s, alert on `held_total / posted_balance` ratio |
| Settlement/expiry race | Transaction fails after user saw approval | Expiry ≫ happy-path duration; failure is explicit and compensable, never an overdraft |
| Long-running reserve transaction holds a row lock | Head-of-line blocking per account | Statement timeout of 1s; the transaction does no network I/O while holding the lock |
| Projection drifts from journal | Wrong balances | Nightly reconciliation replaying `ledger_entries`; alert on any mismatch |
| Bucket count grows unbounded on heavy promo users | Allocation scan slows | Sweep zero-balance and expired buckets; index on `(account_id, spend_priority)`; alert above ~50 active buckets per account |
| Spend priority misconfigured | Users forfeit bonuses, or cash spent before expiring promo money | Priority is module-owned and covered by tests; reconciliation reports forfeited amounts daily |

## Alternatives Considered

### Alternative 1: Keep the Redis lock, extend it to cover the ledger write

**Description**: Hold the lock until Vaullet has synchronously written the ledger entry.

**Pros**: Smallest change from ADR-001; keeps Vaullet's write path event-driven.

**Cons**: The lock would span a cross-service call plus a database write, making hold times *longer* exactly where ADR-001 needed them short. TTL-vs-latency gets worse, not better. And it still fails on Redis failover, because correctness would depend on a lock that async replication can lose. Using a distributed lock to protect a cross-service invariant is the case distributed locks handle worst.

**Why rejected**: Strictly more complexity than the reservation model for strictly weaker guarantees. If the database transaction can enforce the invariant, the lock is redundant; if it can't, the lock can't either.

---

### Alternative 2: Optimistic concurrency (version column + CAS + retry)

**Description**: `UPDATE account_balances SET ... WHERE version = $expected`; retry on zero rows affected.

**Pros**: No row locks held; same correctness guarantee; often better under low contention.

**Cons**: Retry storms on hot accounts; no fairness, so an unlucky request can starve; retry logic in every caller.

**Why rejected — provisionally**: This is a legitimate variant of the same decision, not a worse one. `SELECT FOR UPDATE` gives fairness and simpler code at our contention levels. If per-account contention proves low in load testing, switching is a localized change inside Vaullet with no API impact.

---

### Alternative 3: Kafka partitioning + stateful stream processing

**Description**: ADR-001's documented future path. Partition by account, single stateful processor per partition holds balances in memory.

**Pros**: Very high throughput; strong consistency without any shared-state coordination.

**Cons**: State recovery and rebalancing complexity; still needs a synchronous response path for a REST API, so it doesn't remove the request/response problem, only moves it.

**Why rejected for now**: Correct future path at much higher scale, unchanged from ADR-001's assessment. Postgres row locks handle 1000 TPS spread across accounts comfortably. Revisit above ~5000 TPS or when a single account exceeds ~50 TPS.

---

### Alternative 4: Eventual consistency with compensating reversals

**Description**: Approve optimistically, detect overdrafts asynchronously, reverse.

**Why rejected**: Unchanged from ADR-001 — zero tolerance for overdrafts is a hard requirement, and "approved then reversed" is unacceptable for a wallet.

---

### Note on ADR-001's rejection of database row locking

ADR-001 rejected `SELECT ... FOR UPDATE` as "5–10ms vs 1–2ms for Redis" and "database becomes bottleneck." That comparison does not hold up:

- It compares Redis lock *acquisition* against a full database round-trip. An uncontended `SELECT FOR UPDATE` on a primary key is well under 1ms.
- The Redis lock never replaced the Postgres write — it added a network hop *on top of* it. It was strictly more work per transaction, not less.
- "Database becomes bottleneck" assumes global contention. Row locks contend per account, and per-account transaction rates are inherently low.

The row-locking option was rejected on a performance argument that was never measured, in favor of an approach that was faster only because it wasn't doing the work that made it correct.

## Implementation Notes

### API (Vaullet)

```
POST   /v1/reservations           → 201 {reservation_id, allocations[], available_after}
                                  | 409 INSUFFICIENT_FUNDS
DELETE /v1/reservations/{id}      → 204 (release)
GET    /v1/accounts/{id}/balance  → {posted, held, available, withdrawable,
                                     buckets: [{type, source_module, available,
                                                withdrawable, wagering_remaining,
                                                expires_at}]}
```

`Idempotency-Key` header required on `POST /v1/reservations`.

### Revised transaction flow

```
Transaction Service
  1. Auth Service (REST)          — validate token
  2. Fraud Detection (REST)       — risk score
  3. Limits Service (REST)        — limit check
  4. Vaullet POST /reservations   — atomic check-and-hold  ← invariant enforced here
  5. Persist transaction (APPROVED), publish transaction.created
  ...
  6. On completion: publish transaction.completed → Vaullet settles
     On failure:    publish transaction.failed    → Vaullet releases
```

No lock acquire/release steps. Step 4 is the only place the balance invariant is decided.

### Migration path

None required. ADR-001 was never implemented — no code exists yet. This is a correction on paper, which is the cheapest place to make it.

### Monitoring

- `vaullet.reserve.latency.p95` — target <20ms
- `vaullet.reserve.rejected{reason=INSUFFICIENT_FUNDS}` — business signal, not an error
- `vaullet.reservations.expired.count` — should be near zero; non-zero means transactions are stalling
- `vaullet.held_ratio{account}` — alert above 0.5
- `vaullet.row_lock.wait.p95` — early warning for hot accounts
- `vaullet.reconciliation.mismatch.count` — must be zero
- `vaullet.buckets.active{account}` — allocation cost scales with this; alert above 50
- `vaullet.buckets.forfeited.amount` — promotional money expired unspent; a business metric the Bonus module owns

## References

- Supersedes [ADR-001: Use Redis Distributed Locking for Balance Consistency](001-distributed-locking-for-balance-consistency.md)
- Related: [ADR-003: Hybrid Database Strategy](003-hybrid-database-strategy-with-analytics.md) — `vaullet_db` isolation
- [Martin Kleppmann — How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — why Redlock is unsuitable for correctness-critical use
- [Designing Data-Intensive Applications](https://dataintensive.net/) — Ch. 7 (Transactions), Ch. 9 (Consistency and Consensus)
- Double-entry bookkeeping: journal (immutable) vs. ledger balances (derived projection)

---

**Date**: 2026-08-27
**Author**: Predrag
**Reviewers**: TBD
