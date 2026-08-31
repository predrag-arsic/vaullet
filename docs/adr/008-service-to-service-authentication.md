# ADR-008: Service-to-Service Authentication

## Status

**Partially Accepted** (2026-08-31)

| Layer | Status |
|---|---|
| **Layer 1** — NetworkPolicy, default-deny | ✅ **Accepted** — uncontested baseline |
| **Layer 2** — Linkerd mTLS | ⏸️ **Under review** — see Alternative 6 |
| **Layer 3** — Keycloak service tokens, money path only | ⏸️ **Under review** — scope may widen to all calls |
| **Kafka** — SASL/OAUTHBEARER + ACLs | ✅ **Accepted** — independent of the layer choice |
| **User context** — service identity + explicit `account_id` | ✅ **Accepted** — independent of the layer choice |

**Open question**: whether to adopt a service mesh at all. Alternatives 4 and 5 were written up and
rejected *separately*; **combined** they form a coherent mesh-free design that was never evaluated as a
unit, and that combination is now under active consideration — see **Alternative 6**.

Settled either way: NetworkPolicy as a baseline, Kafka authentication, and how Vaullet learns which
user a reservation is for. None of those depend on the mesh decision.

Closes the item [ADR-006](006-authentication-and-identity.md) explicitly deferred: that ADR settled
*user* identity and assumed services inside a cluster were mutually authenticated, without saying how.

## Context

Vaullet runs ~15 services in one Kubernetes cluster per customer, communicating over REST for
synchronous validation and Kafka for everything else. Nothing currently authenticates one service to
another.

### Three questions, usually conflated

| Question | Example | Mechanism family |
|---|---|---|
| **Transport identity** | Is this really Transaction Service calling `POST /reservations`, or a compromised Notification pod? Can anything on the network read it? | mTLS |
| **Authorization** | Even if it *is* Transaction Service — should it be allowed to call that endpoint? Should Bonus Service? | policy / token scopes |
| **User context** | Transaction Service reserves for user X. How does Vaullet learn that trustworthily? | token propagation |

Most designs answer the first and quietly skip the other two.

### Threat model

Single-tenant clusters (ADR-005) remove cross-customer exposure, so the realistic threats are internal:

- **Lateral movement.** A vulnerability in Notification Service — a large dependency surface, low
  business criticality — becomes a path to the ledger if any pod can call any pod.
- **Traffic interception.** Node-level compromise, or shared/managed infrastructure where pod-to-pod
  traffic is not inherently private.
- **Repudiation.** An adjustment was posted; nothing proves which service posted it.

The blast radius that matters is Vaullet's `POST /v1/reservations`. It is the only synchronous
endpoint that commits money, and it is reachable from inside the cluster.

### Constraints

- **Per-customer cost multiplies.** ADR-005 already flags a high floor, raised again by Keycloak
  (ADR-006). Anything added here is paid once per customer, forever.
- **Keycloak is already core** and already issues tokens against a JWKS every service trusts.
- **Modules are conditionally deployed**, so any policy or ACL scheme must tolerate a service simply
  not existing.

## Decision

We will use **three layers**, because the three questions above have different right answers.

### Layer 1 — NetworkPolicy, default-deny

Every namespace denies ingress by default; each service declares who may reach it.

```yaml
kind: NetworkPolicy
spec:
  podSelector: { matchLabels: { app: vaullet } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: transaction-service } }
        - podSelector: { matchLabels: { app: admin-ui } }
```

Free, and it contains the blast radius of any failure in the layers above. It authenticates a
*network location* rather than a workload, which is precisely why it is not sufficient alone: anything
that lands inside an allowed pod inherits its reachability.

### Layer 2 — Linkerd, mTLS on all internal traffic

Linkerd is installed in every cluster. All meshed pod-to-pod traffic is mutually authenticated and
encrypted, with no application code and no certificates for us to manage.

Workload identity derives from the Kubernetes ServiceAccount, so `transaction-service` is a
cryptographic identity rather than a label anyone can copy. Certificates are issued and rotated by
`linkerd-identity` on a short lifetime; the trust anchor is the only thing with a long-lived rotation
schedule.

Authorization is declarative, and lives beside the service it protects:

```yaml
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata: { name: vaullet-api }
spec:
  podSelector: { matchLabels: { app: vaullet } }
  port: http
  proxyProtocol: HTTP/2
---
apiVersion: policy.linkerd.io/v1alpha1
kind: MeshTLSAuthentication
metadata: { name: reserve-callers }
spec:
  identities:
    - "transaction-service.vaullet.serviceaccount.identity.linkerd.cluster.local"
---
apiVersion: policy.linkerd.io/v1alpha1
kind: AuthorizationPolicy
metadata: { name: vaullet-reserve }
spec:
  targetRef:  { kind: Server, name: vaullet-api }
  requiredAuthenticationRefs:
    - { kind: MeshTLSAuthentication, name: reserve-callers }
```

**Why Linkerd rather than Istio.** Istio's policy engine is richer, and at multi-cluster scale that
richness earns its keep. Here the deciding factor is the cost floor: roughly 30 sidecars per cluster
(15 services × 2 replicas) is on the order of 1.5–3 GB of sidecar memory under Istio versus 300–600 MB
under Linkerd — per customer, permanently. Vaullet needs mTLS and simple L7 authorization, which is
exactly Linkerd's scope. Istio ambient mode removes the sidecar entirely and would change this
calculation; it is worth revisiting when it has more operational history.

### Layer 3 — Keycloak service tokens, on the money path only

Services that move money additionally present a Keycloak-issued service token. Each such caller is a
confidential client with a service account:

```
transaction-service → POST /realms/vaullet/protocol/openid-connect/token
                      grant_type=client_credentials
                    ← { access_token, expires_in: 300 }

                    → POST vaullet/v1/reservations
                      Authorization: Bearer <token>
```

Vaullet validates it against the JWKS it already trusts for user tokens (ADR-006), and checks the
scope.

**Where this applies — and nowhere else:**

| Caller → callee | Scope |
|---|---|
| Transaction Service → Vaullet `POST /reservations`, `DELETE /reservations` | `ledger:reserve` |
| Admin UI → Vaullet manual adjustments | `ledger:adjust` |
| Admin UI → Risk Management account freeze | `risk:act` |

**Why not everywhere.** Applying this to all inter-service calls means token plumbing in fifteen
services to duplicate a guarantee Layer 2 already provides cryptographically. That is cost without
benefit.

**Why here.** A Linkerd `AuthorizationPolicy` is one YAML mistake away from being wrong, and the
reserve endpoint is where that mistake commits money. An application-level check that fails
independently of the mesh is worth its cost on exactly these three paths. The token also carries a
verifiable `azp` claim that goes into the audit record, which answers repudiation in a way network
identity alone does not.

### Kafka is a separate channel and needs its own answer

Layers 1–3 cover REST. Kafka carries most inter-service traffic and is not covered by any of them.

- **Authentication**: SASL/OAUTHBEARER against Keycloak. Services present the same client-credentials
  token they already obtain, so there is one identity system rather than a parallel set of Kafka
  credentials.
- **Authorization**: per-principal ACLs on topics and consumer groups. A service may produce only to
  the topics it owns and consume only what it needs — Bonus Service can read `transaction.created.v1`
  and cannot write it.
- **Encryption**: TLS to brokers.

ACLs are generated from the module composition (ADR-005): **a module that is not deployed has no
principal and no ACLs.** The topic ownership model in ADR-007 — the domain segment names the producer
— maps directly onto produce permissions, so the ACL set is derivable from the topic list rather than
maintained by hand.

### User context: service token plus explicit `account_id`

Vaullet learns which user a reservation is for from an explicit `account_id` in the request body,
authenticated by the caller's service identity rather than by a user token.

Transaction Service has already validated the user's token at the gateway (ADR-006) and authorized the
operation. Vaullet trusts Transaction Service — verified cryptographically by Layer 2 and by token at
Layer 3 — to have done so. Both services are inside one trust boundary, and the reserve call is
idempotent and keyed, so a replay is inert.

**Rejected: forwarding the user's token.** Vaullet would then accept user tokens, which makes a stolen
user token a direct path to the ledger, bypassing every check Transaction Service performs — fraud
scoring, limits, the transaction record itself.

**The hardening path, if this trust boundary ever needs tightening**: RFC 8693 token exchange.
Transaction Service exchanges the user's token for one scoped to *(user, vaullet, reserve)*, useless
anywhere else and carrying genuine user context. Keycloak's `token-exchange` feature is already enabled
for ADR-006's federated flow, so this is a change to two services and no new infrastructure. It is not
adopted now because it adds a Keycloak round trip to the latency-sensitive approval path to defend
against a threat — a compromised Transaction Service — that already has far worse options available
to it.

## Consequences

### Positive Consequences

✅ **Encryption and workload identity with no application code** — Layer 2 covers every service uniformly, including ones written later
✅ **Identity is cryptographic, not a shared secret** — a copied label or leaked config does not impersonate a service
✅ **Defence in depth where it matters** — a mesh policy error does not by itself open the money path
✅ **One identity system** — Keycloak serves users, services and Kafka principals; one JWKS, one rotation story
✅ **Composition-aware** — an undeployed module has no ServiceAccount, no policy and no ACLs, so nothing to misconfigure
✅ **Repudiation answered** — the audit record names a verified calling service, not an inferred one
✅ **Lateral movement is contained** — a compromised Notification pod fails at NetworkPolicy, then at mesh policy, then at the token check

### Negative Consequences

⚠️ **Linkerd is another component per cluster**, with its own upgrade cadence and its own trust-anchor
rotation — an expiry that is easy to forget and takes the whole cluster down when it lapses.

⚠️ **Two authorization mechanisms can drift.** Mesh policy says who may reach `POST /reservations`;
token scopes say who may reserve. Kept in one place and generated where possible, but they are two
statements of one intent and can disagree.

⚠️ **Keycloak sits on the money path.** Mitigated by caching tokens for their lifetime and refreshing
at 80%, plus JWKS caching with stale-if-error — a Keycloak outage stops *new* token issuance rather
than in-flight traffic — but it is a dependency the reserve call did not previously have.

⚠️ **Sidecar latency**, roughly a millisecond per hop. Immaterial against a 200ms approval budget,
measurable in a trace.

⚠️ **mTLS failures are opaque.** "Connection refused" with no application-level explanation is a
recognised operability cost of any mesh, and it will cost debugging time.

⚠️ **Service secrets need rotation**, which is real operational work multiplied per cluster.

### Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Linkerd trust anchor expires | Cluster-wide mTLS failure; total outage | Anchor expiry monitored with 30-day alerting; rotation rehearsed; expiry date is a tracked operational date, not a certificate detail |
| Mesh policy and token scope disagree | An intended-blocked call succeeds, or a legitimate one fails | Both generated from one module-composition source; an integration test asserts the negative case — Bonus Service *cannot* reserve |
| Service secret leaked | Impersonation until rotation | Short token lifetime; secrets from External Secrets Operator, never in Git; rotation on a schedule and on suspicion |
| A new service ships unmeshed | Silent gap — traffic works, unencrypted and unauthenticated | Namespace-level auto-injection; CI asserts every Deployment is meshed; unmeshed pods alert |
| Kafka ACLs drift from deployed modules | A removed module retains permissions, or a new one lacks them | ACLs generated from the same Helm values that decide deployment (ADR-005) |
| Policy blocks a legitimate call in production | Outage on a path nobody tested | Linkerd audit mode before enforcement on any new policy; the ADR-005 CI matrix exercises core-only and full compositions |

## Alternatives Considered

### Alternative 1: Istio

**Description**: Istio in place of Linkerd.

**Pros**: Richer policy engine (JWT validation at the sidecar, request-level authorization, egress
control); larger ecosystem; multi-cluster support Vaullet may eventually want.

**Cons**: Substantially heavier — roughly 1.5–3 GB of sidecar memory per cluster against Linkerd's
300–600 MB, plus a more demanding control plane, multiplied by every customer. The operational
learning curve is steeper on a platform already carrying Kafka, Keycloak, Elasticsearch and seven
databases per cluster.

**Why rejected**: Vaullet needs mTLS and simple L7 authorization. Istio's additional capability does
not offset a per-customer cost that recurs for the life of every contract. **Ambient mode changes this
materially** and should be revisited once it has more production history.

---

### Alternative 2: cert-manager plus Spring Boot mTLS, no mesh

**Description**: Issue per-service certificates with cert-manager; configure mTLS in each service.

**Pros**: No mesh; no sidecars; no additional runtime component.

**Cons**: Truststore and keystore configuration across fifteen services, rotation without restart, and
no authorization layer at all — a certificate says *who*, never *what they may do*. It is the mesh's
work done by hand, minus the mesh's benefits.

**Why rejected**: More application burden than Layer 2 for a strictly weaker result.

---

### Alternative 3: SPIFFE/SPIRE

**Description**: SPIRE issues SVIDs as platform-agnostic workload identity.

**Pros**: The standard for workload identity; portable across clusters and clouds; the correct
foundation at scale.

**Cons**: Another stateful component per cluster, and it still needs something to enforce policy — in
practice a mesh, which then supplies the identity anyway.

**Why rejected**: Solves a heterogeneity problem Vaullet does not have. Fifteen services in one
Kubernetes cluster are exactly the case a mesh's built-in identity covers.

---

### Alternative 4: NetworkPolicy only

**Description**: Layer 1 alone.

**Pros**: Free, native, no new components, no latency.

**Cons**: No encryption, and identity is network position rather than workload — a compromised pod
inherits its reachability, and nothing is recorded about who called what.

**Why rejected**: Retained as a layer, insufficient as the answer. It cannot address interception or
repudiation.

---

### Alternative 5: Keycloak service tokens on every inter-service call, no mesh

**Description**: Layer 3 everywhere; skip the mesh.

**Pros**: One mechanism; reuses Keycloak; fine-grained scopes throughout; no sidecars.

**Cons**: No transport encryption — a Bearer token over plaintext is a replayable password. Token
plumbing in every service, including ones with no security-relevant surface. Keycloak becomes a
dependency of *all* internal traffic rather than three endpoints.

**Why rejected**: Tokens authorize; they do not protect the wire. Without mTLS the tokens themselves
are interceptable, which undermines the mechanism doing the work.

### Alternative 6: NetworkPolicy + Keycloak tokens everywhere, no mesh (Alternatives 4 + 5 combined) — **UNDER ACTIVE CONSIDERATION**

**Description**: Drop the mesh. Layer 1 for reachability, Keycloak client-credentials tokens on
*every* inter-service call for identity and authorization, and encryption solved at the CNI layer
rather than by sidecars.

Alternatives 4 and 5 were each rejected above for a gap the other one fills. Alternative 4 has no
identity or authorization; Alternative 5 supplies both. Alternative 5 has no encryption; that is the
remaining gap, and it does not require a mesh to close:

**CNI-level transparent encryption.** Cilium (or Calico) can encrypt all pod-to-pod traffic with
WireGuard or IPsec at the datapath, with no sidecars and near-zero configuration. If the cluster
already runs Cilium as its CNI — common — this is a flag, not a component.

That reframes the rejection of Alternative 5 above. It was rejected because "tokens authorize, they do
not protect the wire." With WireGuard at the CNI, the wire is protected by something else, and the
objection no longer holds.

**Pros**:
- **No sidecars at all** — the lowest per-cluster footprint of any option, which matters most given the
  ADR-005 cost floor. Better on this axis than Linkerd, not merely competitive.
- **One identity mechanism** for users, services and Kafka principals. Keycloak already issues all
  three; there is no second identity system to reason about.
- **No mesh to operate**, upgrade, or debug — and no trust anchor whose expiry takes the cluster down,
  which is the sharpest operational risk in the current Layer 2.
- **Authorization is explicit in application code**, so it is unit-testable rather than asserted in
  cluster YAML.
- **Failures are legible.** A 403 with a scope name beats an opaque mesh connection reset.

**Cons**:
- **Token plumbing in all fifteen services.** Largely Spring Boot configuration, but it is real work
  and a new failure mode in every service rather than one.
- **Keycloak becomes a dependency of all internal traffic**, not three endpoints. Token and JWKS
  caching make an outage degrade rather than break, but the exposure is much wider.
- **Service identity becomes a bearer secret, not a cryptographic proof.** A leaked client secret
  impersonates a service until rotation; a mesh identity cannot be copied out of a config map.
- **Loses the mesh's free observability** — golden metrics and per-hop tracing would need building.
- **Tokens are replayable within their lifetime** if ever intercepted, where mTLS binds identity to the
  connection.
- **CNI encryption is node-to-node**, not workload-to-workload. Traffic between two pods on the *same*
  node may not traverse the encrypted path, depending on configuration.

**What to evaluate before deciding**:
1. Which CNI the target clusters run, and whether WireGuard encryption is available and supported there
2. Measured cost of token plumbing across services — likely a shared library, so a one-off
3. Whether losing cryptographic workload identity is acceptable given single-tenant isolation
4. What replaces the mesh's observability, and what that costs

**Why it is not yet decided**: it is genuinely competitive with the layered design and better on
per-customer cost, which is the constraint that has driven several decisions in this repository. The
trade is cryptographic workload identity and free observability against a smaller, simpler runtime. It
deserves a real evaluation rather than a paragraph.

## Implementation Notes

> **Note**: the Linkerd notes below apply only if Layer 2 is confirmed. See Alternative 6.

### Linkerd

Installed by the umbrella chart in every cluster. Namespace annotated
`linkerd.io/inject: enabled` so meshing is the default and opting out is explicit. Trust anchor stored
in the per-customer secret store with expiry tracked as an operational date.

New `AuthorizationPolicy` resources go to **audit mode** first — traffic that would be denied is logged
rather than blocked — and are promoted to enforcement after a clean window.

### Keycloak clients

```yaml
clients:
  - clientId: transaction-service
    serviceAccountsEnabled: true
    standardFlowEnabled: false          # no interactive login
    scopes: [ "ledger:reserve" ]
  - clientId: admin-ui
    serviceAccountsEnabled: true
    scopes: [ "ledger:adjust", "risk:act" ]
```

Declarative, applied by `keycloak-config-cli` alongside the realm configuration from ADR-006. Secrets
are delivered by External Secrets Operator; none is committed.

### Spring Boot

Callers use `ClientCredentialsOAuth2AuthorizedClientProvider` with a cached token refreshed at 80% of
lifetime. Callees are resource servers validating against the Keycloak JWKS — the same configuration
already present for user tokens, with an additional scope check.

### Kafka

Strimzi with `type: oauth` listener authentication against the Keycloak realm, and ACLs generated from
the deployed module set. Topic ownership per ADR-007 gives produce permissions directly: a service may
produce only to topics whose domain segment names it.

### CI enforcement

1. Every `Deployment` is meshed — unmeshed workloads fail the build
2. Every service reaching Vaullet's reserve endpoint holds a Keycloak client with `ledger:reserve`
3. A negative test asserts a non-authorized service is refused at both Layer 2 and Layer 3
4. Kafka ACLs match the deployed module set for each configuration in the ADR-005 matrix

### Out of scope

**Human access** — `kubectl exec`, database credentials, break-glass procedures — is operational
access control, not service-to-service authentication. It deserves its own decision and is tracked
separately.

## References

- [ADR-005: Module Composition](005-module-composition-and-deployment-topology.md) — per-customer cost floor; conditional deployment driving policy and ACL generation
- [ADR-006: Authentication and Identity](006-authentication-and-identity.md) — Keycloak, the JWKS every service already trusts, and the deferral this ADR closes
- [ADR-007: Kafka Topics](007-kafka-topics-and-event-schema-evolution.md) — topic ownership mapping onto produce ACLs
- [ADR-004: Atomic Balance Reservations](004-atomic-balance-reservations.md) — why `POST /reservations` is the endpoint that matters
- [Linkerd — Automatic mTLS](https://linkerd.io/2/features/automatic-mtls/)
- [Linkerd — Authorization Policy](https://linkerd.io/2/features/server-policy/)
- [RFC 8693: OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693)

---

**Date**: 2026-08-27
**Author**: Predrag
**Reviewers**: TBD
