# Open Items

Everything known to be unresolved across `arch.md` and ADRs 001–012, as of **2026-09-02**.

This is the detailed register. [`README.md`](README.md) keeps the short index of decisions that still
need an ADR; this file also carries the contradictions, accepted gaps and deferred re-evaluations that
are not decisions in their own right.

**How to use it**: items are `B` (blocking implementation), `D` (needs a decision), `G` (known gap,
accepted), `H` (hygiene) and `R` (deferred re-evaluation with a trigger). Delete an item when it is
closed — a register nobody prunes stops being read.

---

## B — Blocking implementation

Fix before writing the code each one touches. All four are contradictions between accepted decisions,
not new decisions.

### B1 · Withdrawal holds have no lifetime set

- **Where**: [ADR-009](009-payment-rails-deposits-and-withdrawals.md) §4, against
  [ADR-004](004-atomic-balance-reservations.md) *Reservation lifetime*
- **What**: ADR-009 opens an ADR-004 reservation at `REQUESTED`. Above the auto-approval threshold the
  withdrawal waits in `REQUESTED` for an Admin UI action — human review, so hours or days. Hold
  lifetime became caller-supplied on 2026-09-02 and the wagering path was given a value; the
  withdrawal path was not, so it inherits the 300-second default.
- **Why it matters**: the hold releases five minutes after the request and the funds become spendable
  while the payout is still pending approval. A user can spend money already on its way out — the
  exact failure the hold at `REQUESTED` exists to prevent.
- **Fix**: ADR-009 §4 states the withdrawal hold TTL explicitly, as a deployment setting rather than a
  constant — the review SLA belongs to the operator. Blocks the withdrawal path, not the ledger core.

### B2 · The account anchor has two creation paths, and one cannot succeed

- **Where**: [ADR-006](006-authentication-and-identity.md) *The account anchor* vs
  [ADR-012](012-external-api-surface.md) §3
- **What**: ADR-006 creates the anchor just-in-time on first successful authentication, with
  `keycloak_sub UUID NOT NULL`. ADR-012 gives the operator `POST /v1/accounts → account_id`. An
  operator provisioning a user who has never logged in has no `keycloak_sub` to supply.
- **Why it matters**: as specified the insert fails, and this is on the first call of every
  integration. It is also a real modelling question — whether an account can exist before an identity
  does — not a typo.
- **Fix**: either `keycloak_sub` is nullable until first login (and every reader handles an unlinked
  anchor, including the ledger), or `POST /v1/accounts` is not account creation and ADR-012 says what
  it is. Blocks Auth Service and the external API; does not block the ledger.

### B3 · `external_ref` is specified but has no column

- **Where**: ADR-012 §3 and *Alternative 5*, against ADR-006's `accounts` and
  [ADR-003](003-hybrid-database-strategy-with-analytics.md) `identity_schema`
- **What**: ADR-012 declares `external_ref` unique per deployment and immutable once set, and gives it
  a lookup endpoint (`GET /v1/accounts/by-ref/{external_ref}`). No schema anywhere has the column.
- **Why it matters**: uniqueness and immutability are constraint claims. Unless they are constraints,
  they are hopes — and ADR-012 already commits to telling operators they cannot change the value.
- **Fix**: add it to the anchor with `UNIQUE`, nullable, and no `UPDATE` path. Same change as B2.

### B4 · `ledger:reserve` scope table is incomplete

- **Where**: [ADR-008](008-service-to-service-authentication.md) Layer 3
- **What**: the table lists `POST /reservations` and `DELETE /reservations`. `GET /v1/reservations/{id}`
  was added to ADR-004 on 2026-09-02 and is not in it.
- **Why it matters**: small, but ADR-008's whole argument is that the money path is enumerated
  explicitly. An endpoint missing from the enumeration is either unauthorized or silently ungoverned.
- **Fix**: one row. Decide whether reading a reservation needs `ledger:reserve` or a read scope.

---

## D — Needs a decision

Ordered by what blocks or exposes the most.

### D1 · Manual ledger adjustments ⭐

Corrections, goodwill credits, dispute resolution, write-offs.

- **Where**: promised in [ADR-006](006-authentication-and-identity.md) (`FINANCE` role, two-principal
  rule), explicitly *withdrawn* in ADR-008 (the `ledger:adjust` scope was removed because the endpoint
  does not exist), absent from ADR-004's ledger paths and ADR-012's surface.
- **Why first**: it is the only open decision that already has access controls and a
  separation-of-duties rule written against it. ADR-008 was right that authorising a non-existent
  operation is worse than not authorising it — but the resolution is to design the operation, not to
  leave the control dangling.
- **Shape it has to take**: an adjustment is a journal entry like any other (the journal is
  append-only — a correction is a new entry, never an edit), it needs a reason code and a
  two-principal approval trail, it must name which bucket it credits or debits, and `DEBT` write-off
  is one of its cases. Interacts with B1's operator-review flow.

### D2 · Human and operational access control

`kubectl exec`, database credentials, break-glass procedure.

- **Where**: explicitly out of scope in ADR-008; listed as open in `CURRENT_DISCUSSION_STATE.md`.
- **Why it matters**: ADR-008 secures service-to-service traffic thoroughly and says nothing about the
  humans who can reach the same data directly. Under a gambling licence this is usually the part an
  auditor asks about first, and "we have mTLS between services" is not an answer to it. ADR-010 adds
  to the surface: whoever can merge to `wallet-gitops` can change what every customer runs.
- **Depends on nothing** — can be written now.

### D3 · Data retention and archival

- **Where**: `CURRENT_DISCUSSION_STATE.md`; interacts with ADR-007 §8 (money topics retained
  indefinitely, tiered) and the seven-year ledger immutability asserted in ADR-006.
- **Why it matters**: two retention claims already exist in the architecture and neither has a policy
  behind it. Under GDPR, indefinite retention needs a lawful basis and a deletion story, and "the
  ledger is immutable for seven years" and "the user may request erasure" have to be reconciled
  somewhere. That reconciliation is a decision, not an implementation detail.

### D4 · PSP settlement matching

Provider settlement reports vs captured deposits.

- **Where**: ADR-009 accepts the capture-to-settlement gap as "short, bounded, and priced in", and
  stops there.
- **Why it matters**: nothing currently detects a capture that never settles. That is the exposure
  window ADR-009 accepts, but accepting a window is different from being able to see into it.

### D5 · Multi-currency within one deployment

- **Where**: ADR-004 *Currency* documents the extension path (one account per `(user, currency)`) and
  defers the decision.
- **Why it matters**: it is genuinely deferred rather than forgotten, and the shape is recorded so it
  will not be rediscovered under pressure. Revisit when a contract asks — not before.

### D6 · Disaster recovery and backup

- **Where**: `arch.md` Next Steps item 6; untouched by any ADR.
- **Why it matters**: ADR-003 asserts `datawarehouse_db` is rebuildable from the event stream and
  ADR-007 makes retention support that claim — but no document says how `vaullet_db` itself is backed
  up, what the RPO is, or how a cluster is restored. The ledger is the system of record and it has no
  recovery story.

### D7 · Platform-wide observability

- **Where**: `arch.md` Next Steps item 8. ADR-004 specifies ledger metrics and ADR-003 a few alert
  thresholds; there is no platform decision.
- **Why it matters**: fourteen services across N single-tenant clusters, with no decision on where
  logs, metrics and traces go, or whether a vendor ever sees a customer's data. The last part is a
  data-residency question, which makes it more than a tooling preference.

### D8 · Operator reporting surface

Dashboard vs API vs embeddable widgets.

- **Where**: `CURRENT_DISCUSSION_STATE.md`. The auth half is settled (Admin UI is role-scoped; BI reads
  `datawarehouse_db` with read-only credentials).
- **Why it matters**: a product question rather than an architectural one, and flagged as such — but
  it determines whether `datawarehouse_db` needs a query API in front of it.

---

## G — Known gaps, accepted

Not scheduled. Recorded so they are not rediscovered as surprises.

| ID | Gap | Where | Status |
|---|---|---|---|
| **G1** | **Per-customer cost floor is high** — core alone is 7 services plus Kafka, Redis, Elasticsearch, Keycloak, Argo CD and 5 Postgres databases. Small operators may be unprofitable on this topology | ADR-005, ADR-010 | ADR-005 *Alternative 3* (modular monolith, compile-time module selection) is the documented candidate for a low-cost tier. Nobody has priced either |
| **G2** | **Elasticsearch is core by inheritance** — Fraud Detection is core and ADR-003 gives it ES, so every customer runs an ES cluster | ADR-003, ADR-005 | Candidate: degrade pattern search to Postgres-only and make ES a Category 4 adapter |
| **G3** | **Fourteen services is a lot for a solo project.** The module boundaries have commercial justification, but "sellable unit" and "separately deployed service" are not the same thing | ADR-005 | Accepted deliberately. The mitigation is build order, not architecture — ledger first, extract shared CI at the second service |
| **G4** | **Indefinite money-topic retention has not been sized.** Tiered storage makes it affordable in principle; the per-cluster cost is unknown | ADR-007 §8 | Needs a number before a customer contract prices it |
| **G5** | **Nothing enforces a coherent contract version set.** Six independently versioned artifacts mean a service can run `contracts-ledger` 2.3 with `contracts-transaction` 1.9. An architecture test forbids a *cycle*; nothing checks the *set* | ADR-002 | Candidate: a BOM the services import, so the set is chosen once |

---

## H — Hygiene

- **H1** · `arch.md` writes event names without the `.v1` suffix in ~40 places. A convention note now
  says arch.md uses logical names and the topic appends the major version — defensible, but ADR-007's
  rule is about the topic name, and a reader checking conformance will bounce off it.
- **H2** · Every ADR ends with `**Reviewers**: TBD`. Fine for a solo project; it is the first thing a
  reviewer notices, and the honest fix is to remove the field rather than fill it in falsely.

---

## R — Deferred, with a trigger

Decisions that are correct now and have a named condition for revisiting. Not open items — listed so
the trigger is not forgotten.

| ID | Decision | Revisit when |
|---|---|---|
| **R1** | JSON Schema over Avro (ADR-007 §5) | Serialisation cost becomes measurable at throughput |
| **R2** | Linkerd over Istio (ADR-008 Layer 2) | Istio ambient mode has more operational history — it removes the sidecar cost that decided this |
| **R3** | Argo CD per cluster (ADR-010 §6) | Per-cluster resource cost becomes the binding constraint |
| **R4** | Path versioning over date-based (ADR-011 *Alternative 2*) | The integrator count grows into the hundreds |
| **R5** | PostgreSQL for `datawarehouse_db` (ADR-003) | Reporting queries exceed ~5s → ClickHouse or TimescaleDB |
| **R6** | 12 partitions on money topics (ADR-007 §6) | Increasing is a one-way door — changing it remaps keys to partitions |

---

**Maintenance**: reviewed whenever an ADR is accepted or revised. Last full pass 2026-09-02, against
ADRs 001–012 and `arch.md`.
