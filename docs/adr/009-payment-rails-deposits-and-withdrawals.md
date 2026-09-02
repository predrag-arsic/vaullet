# ADR-009: Payment Rails — Deposits, Withdrawals and Chargebacks

## Status

**Accepted** (2026-09-01)
**Revised** (2026-09-02) — `DEBT` is carried in `account_balances.debt_total` and excluded from
allocation, per the corresponding ADR-004 revision

## Context

Vaullet has a ledger, reservations, typed balance buckets, fraud scoring, limits and a bonus platform.
It has no way to put money in or take money out. Every flow documented so far begins with a balance
that already exists.

This is the most conspicuous gap in the architecture: the product is a wallet users top up.

### What has to be decided

- **Who owns the deposit and withdrawal lifecycle**, given that ADR-005's rule forbids optional
  modules on synchronous paths and `arch.md` already contains one corrected case of a service
  duplicating the money path
- **When a deposit becomes spendable** — the point where an external promise becomes ledger money
- **How withdrawals interact with holds and bucket withdrawability** from ADR-004
- **What happens when a settled deposit is reversed weeks later**, after the user has spent it

### The constraint that shapes everything

[ADR-004](004-atomic-balance-reservations.md) enforces:

```sql
CHECK (posted_balance - held_total >= 0)   -- overdraft is a schema violation
```

That constraint is load-bearing and correct. It also means **a chargeback cannot be recorded as a
simple debit**. A user deposits €100, spends it, and six weeks later the card issuer reverses the
deposit. The money is gone, the obligation is real, and the schema will refuse to write it. Any design
that ignores this discovers it in production, on a path that only executes weeks after the code ships.

## Decision

### 1. Deposits and withdrawals are transaction types, not a separate service

Transaction Service owns both lifecycles. They are `DEPOSIT` and `WITHDRAWAL` transaction types
running the standard approval path: auth → fraud → limits → ledger.

There is one path money can move by, and every caller uses it. `arch.md` previously had Subscription
Service orchestrating its own payments — checking limits and balance directly, bypassing Transaction
Service — which put the money path in two places and was corrected for exactly this reason. Creating a
Payment Service that also moved money would reintroduce that error at larger scale.

### 2. PSP integration is a Category 4 adapter

Card acquiring, bank transfers and local payment methods vary per operator and per market. A **PSP
Adapter** (ADR-005 Category 4) presents one internal port with per-contract implementations — Stripe,
Adyen, a local acquirer.

```
Transaction Service  ──>  PSP Adapter port  ──>  [ Stripe | Adyen | local acquirer ]
                     <──  psp.* events      <──
```

The adapter is **infrastructure, not a sellable module**. Every deployment needs some rail; which rail
is a configuration choice, exactly like the Auth provider in ADR-006. This keeps it off the sellable
list and preserves ADR-005's star invariant.

### 3. Deposit lifecycle — credit at capture, not authorization

```
INITIATED ──> AUTHORIZED ──> CAPTURED ──> SETTLED
     │             │              │
     └── ABANDONED └── DECLINED   └── (money spendable from here)
```

| State | Meaning | Ledger effect |
|---|---|---|
| `INITIATED` | User started a top-up; no external call yet | none |
| `AUTHORIZED` | PSP reserved funds on the instrument | none |
| `CAPTURED` | PSP committed the charge | **CREDIT the user's `CASH` bucket** |
| `SETTLED` | Funds landed in the operator's bank | none — already credited |
| `DECLINED` / `ABANDONED` | Terminal failure | none |

**Credit at `CAPTURED`, not `AUTHORIZED` and not `SETTLED`.** Authorization is a reservation on the
user's instrument that may never be captured, so crediting there hands out money that may not arrive.
Settlement can take days, and a wallet that makes deposits unusable for days is not a product anyone
buys. Capture is the point where the charge is committed — the earliest moment the money is real
enough to spend.

The gap between capture and settlement is a real exposure window, deliberately accepted: it is short,
bounded, and priced into every payments business.

### 4. Withdrawal lifecycle — holds are already built

```
REQUESTED ──> APPROVED ──> SENT ──> CONFIRMED
     │            │           │
     └── REJECTED └── (hold released on failure at any point)
```

**A withdrawal takes an ADR-004 reservation at `REQUESTED`.** This is the mechanism working as
designed rather than a new one: the funds are held the moment the user asks, so they cannot be spent
while the payout is in flight. If the payout fails, the hold is released and nothing was ever
journalled.

**Only withdrawable buckets are eligible.** `withdrawable_balance` from ADR-004 is
`SUM(posted - held) WHERE withdrawable` — so bonus money with outstanding wagering is structurally
excluded from payout. The bucket model already answers "how much can this user actually cash out",
and this is the flow it was designed for.

**Approval policy is core; case management is optional.** Auto-approval below a configured threshold
is decided by Transaction Service using Limits (both core). Above the threshold the withdrawal is held
for review, and *if* Risk Management is deployed it opens a case; if it is not, the withdrawal waits
in `REQUESTED` for an Admin UI action. A sellable module never sits in the required path — it enriches
a path that works without it.

### 5. Chargebacks — a receivable, never a negative balance

A chargeback debits the user's `CASH` bucket. When the balance covers it, that is the whole story.

When it does not — the common case, since the money was usually spent — **the shortfall becomes a
receivable, not a negative balance.**

```sql
-- new bucket type; the user owes the operator
bucket_type IN ('CASH','BONUS','LOYALTY','REFERRAL','DEBT')
```

A `DEBT` bucket holds a positive amount representing an obligation, and is excluded from
`available_balance` and from `withdrawable_balance`.

**Three places must agree, and ADR-004 now names all three**: the allocation query filters
`bucket_type <> 'DEBT'`, `account_balances.posted_balance` sums fundable buckets only, and the
obligation is carried separately in `account_balances.debt_total`. Folding debt into `posted_balance`
would let the aggregate pre-check approve an amount the bucket allocation cannot cover — the reserve
path would pass its own gate and then fail. Writing a chargeback shortfall therefore increments
`debt_total`, never `posted_balance`, under the same account-row lock as every other write path.

**Why not simply allow the balance to go negative:** it would require weakening
`CHECK (posted_balance - held_total >= 0)`, and that constraint is the backstop that makes overdrafts
a schema violation rather than a bug (ADR-004). Removing it to model debt would remove it for every
path, including the reserve path it was written for. A separate positive-valued account is the
standard double-entry treatment of a receivable and preserves the invariant everywhere.

Consequences of a `DEBT` bucket, all deliberate:

- Future deposits **net against debt first** before crediting `CASH`
- The account moves to a restricted status — it can receive, it cannot withdraw
- The debt is visible, aged, and reportable rather than hidden in a negative number
- Write-off is an explicit decision with an audit trail, not an accounting side effect

**Chargeback is not a refund.** A refund is voluntary, initiated by the operator, and already handled
inside Transaction Service. A chargeback is imposed by the issuer, arrives without warning, and can
arrive after the funds are gone. They produce different ledger effects and must not share a code path.

### 6. Idempotency is mandatory on every PSP callback

PSP webhooks retry, arrive out of order, and duplicate. Every callback carries the provider's event id
and is deduplicated on it before any ledger effect:

```sql
CREATE TABLE psp_events (
    psp_event_id   TEXT PRIMARY KEY,      -- provider's id, not ours
    provider       TEXT        NOT NULL,
    transaction_id UUID        NULL,
    received_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

A duplicate `CAPTURED` webhook that credits twice is the single most expensive bug available in this
design, and a primary key is a cheaper defence than vigilance. This mirrors ADR-004's
`UNIQUE (idempotency_key)` on reservations and ADR-007's `event_id` envelope: idempotency is enforced
by constraints, never by convention.

**Out-of-order handling**: state transitions are monotonic. A late `AUTHORIZED` arriving after
`CAPTURED` is recorded and ignored, never applied backwards.

### 7. Events

Per ADR-007 naming:

```
deposit.initiated.v1     deposit.captured.v1     deposit.failed.v1
withdrawal.requested.v1  withdrawal.approved.v1  withdrawal.confirmed.v1  withdrawal.failed.v1
chargeback.received.v1   chargeback.resolved.v1
```

`deposit.captured.v1` is what Bonus Service consumes for deposit-match promotions — the flagship
module's most common trigger, and it arrives as an event, so Bonus stays removable.

## Consequences

### Positive Consequences

✅ **The wallet can actually be funded** — the architecture describes a complete money lifecycle for the first time
✅ **One money path preserved** — deposits and withdrawals run the same approval sequence as everything else
✅ **Holds work without modification** — withdrawal reservation is ADR-004 used as designed, not extended
✅ **Bucket withdrawability pays off** — "can this user cash out bonus money" is answered by a flag that already exists
✅ **The overdraft invariant survives contact with chargebacks** — debt is modelled, not smuggled in as a negative balance
✅ **PSPs are swappable per contract** — an existing mechanism, not a new one
✅ **Duplicate credits are structurally impossible** — a primary key, not a code review

### Negative Consequences

⚠️ **Capture-to-settlement exposure.** Users spend money that has not yet reached the operator's bank.
The window is short and bounded, but it is real and it is the cost of a usable product.

⚠️ **A new bucket type touches allocation logic.** `DEBT` must be excluded from spend allocation,
available balance and withdrawable balance — three places, each a defect if missed.

⚠️ **Chargebacks arrive weeks late**, so this path is exercised long after deployment and is the least
likely to be covered by a realistic test. It needs deliberate testing against PSP sandboxes.

⚠️ **Withdrawal approval needs an operator.** Above-threshold payouts wait for a human. That is a
staffing requirement the platform imposes on its customer.

⚠️ **Per-PSP adapters are real integration work.** Each provider has its own webhook semantics, state
names and failure modes, and "one internal port" hides that only from the services above it.

### Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Duplicate `CAPTURED` webhook credits twice | Money created from nothing | `psp_events` primary key on the provider's event id, checked before any ledger effect |
| Webhook missed entirely | Deposit never credited; user paid and got nothing | Scheduled reconciliation against PSP transaction status; unmatched captures alert |
| Chargeback on an account with no funds and no future deposits | Unrecoverable debt | Debt aged and reported; write-off is an explicit decision, never automatic |
| User withdraws immediately, then charges back | Deliberate cash-out fraud | Withdrawal fraud scoring; configurable hold period on newly deposited funds before withdrawal is permitted |
| `DEBT` bucket included in an allocation by mistake | User spends money they owe | Excluded by bucket type in allocation, plus a test asserting a debt-only account has zero available balance |
| PSP adapter maps states incorrectly | Deposits credited at the wrong lifecycle point | Per-adapter conformance test suite against the internal port's state machine |

## Alternatives Considered

### Alternative 1: A dedicated Payment Service that moves money itself

**Description**: A separate service owning deposits and withdrawals end to end, calling the ledger
directly.

**Pros**: Clean separation of payments concerns; PSP integration isolated from transaction logic;
independently scalable.

**Cons**: Two services would hold opinions about how money moves. Every future control — a fraud rule,
a limit type, a jurisdictional check — would need implementing twice, and the second copy would drift.
This is precisely the defect corrected in Subscription Service, where a module orchestrated its own
payments and bypassed the approval path.

**Why rejected**: The mistake is already documented in this repository. The PSP *adapter* is isolated;
the *money path* is not duplicated.

---

### Alternative 2: Credit deposits at authorization

**Description**: Make funds spendable as soon as the PSP authorizes.

**Pros**: The fastest possible user experience.

**Cons**: An authorization is a reservation on the user's instrument, not a committed charge. Captures
fail. Crediting at authorization hands out money that may never arrive, and recovering it means
clawing back funds the user has already spent — creating the chargeback problem deliberately and at
volume.

**Why rejected**: The latency saved is seconds; the exposure created is unbounded.

---

### Alternative 3: Credit deposits at settlement

**Description**: Wait until funds land in the operator's bank.

**Pros**: Zero exposure. The operator holds the money before the user can spend it.

**Cons**: Settlement takes one to three business days on most rails. A wallet where a top-up becomes
usable on Thursday is not competitive in a market where users expect to deposit and bet immediately.

**Why rejected**: Eliminates a bounded risk by eliminating the product.

---

### Alternative 4: Allow negative balances for chargebacks

**Description**: Weaken the `CHECK` constraint; let `CASH` go negative.

**Pros**: Simplest possible model — a chargeback is just a debit.

**Cons**: The constraint would be relaxed for *every* path, not only chargebacks. ADR-004 introduced it
specifically because ADR-001's design had no schema-level backstop and could not catch its own bug. A
negative balance also conflates two different things: money owed, and money available. Every allocation
query would need to reason about sign.

**Why rejected**: It trades a durable invariant for a modelling shortcut, on the one table where the
invariant matters most.

---

### Alternative 5: Refuse the chargeback shortfall — freeze and recover manually

**Description**: Record no ledger effect; freeze the account and pursue recovery outside the system.

**Pros**: No new bucket type; no allocation changes.

**Cons**: The ledger stops reflecting reality. The operator is owed money the system does not record,
so it appears in no report and ages in no queue. Recovery becomes a spreadsheet.

**Why rejected**: A ledger that omits known obligations is not a ledger.

## Implementation Notes

### PSP Adapter port

```
POST   /v1/payments/charge      { account_id, amount, currency, method_token, idempotency_key }
POST   /v1/payments/payout      { account_id, amount, currency, destination, idempotency_key }
GET    /v1/payments/{id}        → provider-neutral status
```

Inbound webhooks are normalised to the internal state machine before publication, so no service above
the adapter sees provider vocabulary. Each adapter ships a conformance suite proving its mapping.

### Deposit-to-bonus interaction

`deposit.captured.v1` carries `account_id`, amount and a deposit sequence number, which is what
deposit-match promotions need. Bonus grants land as `rewards.value-granted` into a new bucket
(ADR-004) — no special deposit path in the ledger.

### Withdrawal hold period

A configurable delay between deposit and eligibility for withdrawal of those funds, defaulting to
zero and set per contract. It exists because deposit-then-immediate-withdrawal is a recognised
laundering and cash-out pattern; whether an operator enables it is their risk decision.

### Testing

PSP sandboxes for the full matrix: capture failure after authorization, duplicate webhooks,
out-of-order webhooks, chargeback against a funded account, chargeback against an empty account, and
payout failure after `SENT`. The last three are the ones that will otherwise be discovered in
production.

### Deferred

**PSP settlement matching** — comparing provider settlement reports against captured deposits — is
adjacent to this ADR but is an operational finance concern rather than a payments-flow one. Tracked
separately.

## References

- [ADR-004: Atomic Balance Reservations](004-atomic-balance-reservations.md) — holds, buckets, `withdrawable`, and the `CHECK` constraint that shapes the chargeback design
- [ADR-005: Module Composition](005-module-composition-and-deployment-topology.md) — Category 4 adapters; why approval policy is core and case management is not
- [ADR-007: Kafka Topics](007-kafka-topics-and-event-schema-evolution.md) — event naming and the idempotency envelope
- `arch.md` — the corrected Subscription Service flow that Alternative 1 would have repeated

---

**Date**: 2026-09-01
**Author**: Predrag
**Reviewers**: TBD
