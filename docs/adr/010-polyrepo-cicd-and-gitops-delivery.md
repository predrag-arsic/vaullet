# ADR-010: Polyrepo CI/CD and GitOps Delivery

## Status

**Accepted** (2026-09-01)

## Context

Vaullet is 14 backend services plus Admin UI across separate repositories (ADR-002), deployed as one
Kubernetes cluster per customer with a module set determined by contract (ADR-005). Delivery therefore
has a shape most pipelines do not: **N repositories × M customer clusters, where each cluster runs a
different subset of the artifacts.**

### What makes this harder than a normal pipeline

- **No single deploy target.** "Deploy Bonus Service" means "deploy to every customer whose contract
  includes Bonus", which is a query, not a constant.
- **14 pipeline definitions drift.** Copy-pasted workflows diverge within months, and the security
  steps are the first to rot.
- **Auto-deploying to customer production is wrong.** These are contracted B2B deployments, not a SaaS
  where one fleet moves together.
- **A vendor with cluster-admin into every customer cluster is a problem.** The same data-residency
  and isolation reasoning that produced single-tenant deployment (ADR-005) and self-hosted Keycloak
  (ADR-006) applies to the delivery mechanism.
- **The invariants only hold if CI enforces them.** ADRs 005, 007 and 008 each define rules that are
  worthless as review comments and load-bearing as build failures.

### Stated preference

GitHub (Actions), Argo CD (Apache-2.0, self-hosted), build on push, deploy on commit.

## Decision

### 1. The split: repositories build, Git deploys

**A service repository never deploys anything.** Its pipeline's final act is to commit a version bump
to the GitOps repository. Argo CD reconciles from there.

```
push to wallet-bonus-service
        │
        ▼
  GitHub Actions:  test → build → scan → push image → publish Helm chart (OCI)
        │
        ▼
  commit image tag to wallet-gitops (internal envs only)
        │
        ▼
  Argo CD in each cluster pulls its own path → reconciles
```

This is the user's "build on push, deploy on commit" made precise: **push** triggers build in the
service repo; **commit to the GitOps repo** triggers deploy. The two are deliberately different
events in different repositories, so what is deployed is always a reviewable Git state rather than a
side effect of a build.

### 2. Repository layout

| Repository | Contains | Publishes |
|---|---|---|
| `wallet-<service>` × 14 | Source, Dockerfile, Helm chart, migrations | Image + chart to GHCR |
| `wallet-admin-ui` | Frontend | Image + chart |
| `wallet-shared-contracts` | DTOs, event schemas | Maven + NPM (ADR-002), JSON Schemas to registry (ADR-007) |
| `wallet-infrastructure` | Umbrella chart, Terraform, reusable workflows | Umbrella chart to GHCR |
| `wallet-gitops` | Per-customer values, ApplicationSets | Nothing — it *is* the desired state |

**The Helm chart for a service lives with the service**, published as an OCI artifact to GHCR
(`oci://ghcr.io/<org>/charts/bonus-service`). The chart version moves with the code that needs it,
which is the whole reason for a polyrepo. The umbrella chart in `wallet-infrastructure` depends on
them by version.

### 3. Reusable workflows — one pipeline, fourteen callers

The single most important decision for a 14-repo polyrepo. Each service's workflow is a stub:

```yaml
# wallet-bonus-service/.github/workflows/ci.yml
name: CI
on:
  push:      { branches: [main] }
  pull_request:

jobs:
  build:
    uses: vaullet/wallet-infrastructure/.github/workflows/service-ci.yml@v3
    with:
      service: bonus-service
      java-version: '21'
    secrets: inherit
```

Everything real — build, test, architecture tests, image scan, SBOM, signing, chart publish, GitOps
bump — lives once in `wallet-infrastructure`. A new security step ships to all fourteen services by
bumping one tag.

The reusable workflow is pinned by tag, not by branch, so a change to shared CI cannot silently alter
what fourteen pipelines do.

### 4. Image and chart tagging: immutable, always

```
ghcr.io/vaullet/bonus-service:sha-9f3c1a2          ← every main build
ghcr.io/vaullet/bonus-service:2.4.1                ← release tags only
```

**No `latest`, no mutable tags, no re-tagging.** Argo CD compares desired to actual state; if a tag's
contents can change, the cluster and Git can disagree while appearing identical, and rollback stops
meaning anything. Immutability is what makes "revert the commit" a real rollback.

Images are signed with cosign and the cluster verifies signatures on admission.

### 5. Promotion: automatic internally, deliberate to customers

| Environment | Trigger | Approval |
|---|---|---|
| `dev` | Merge to `main` — automatic GitOps commit | None |
| `staging` | Automatic after `dev` is healthy | None |
| Customer production | **Pull request** against that customer's values | Reviewed; per-customer |

Auto-deploying every merge into contracted customer clusters would be wrong regardless of tooling. A
customer's deployed version is part of what was sold, and changing it is an action taken *for* that
customer, not a consequence of an unrelated merge.

Promotion opens one PR per customer, generated from the same source, so a fleet-wide upgrade is a set
of reviewable changes rather than an event.

### 6. Argo CD per cluster, not hub-and-spoke

**Each customer cluster runs its own Argo CD**, pulling only its own path from `wallet-gitops`.

A central Argo managing every customer cluster would need credentials with cluster-admin into all of
them — a single component whose compromise reaches every customer, and an inbound path from vendor
infrastructure into a licensed operator's environment. That contradicts the reasoning behind
single-tenant deployment (ADR-005) and self-hosted identity (ADR-006). Pull-based per-cluster Argo
keeps the trust direction outward: the customer's cluster reaches out to read Git, and nothing reaches in.

The cost is another component per cluster, which is the same trade this architecture has now made
several times.

### 7. Composition comes from ADR-005, unchanged

One Argo `Application` per customer, pointing at the umbrella chart with that customer's values. Module
enablement is already expressed there, so delivery needs no separate concept of composition:

```yaml
# wallet-gitops/customers/acme-betting/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    repoURL: ghcr.io/vaullet/charts
    chart: vaullet-umbrella
    targetRevision: 3.2.0
    helm:
      valueFiles: [ values.yaml ]      # modules.*.enabled, platform.currency, auth.provider
  destination: { server: https://kubernetes.default.svc }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

`selfHeal: true` matters: a manual `kubectl edit` in a customer cluster is reverted. The Git state is
the deployment, and drift is a bug rather than a fact to discover later.

### 8. CI gates — the invariants other ADRs depend on

Every rule declared elsewhere is enforced here, as a build failure:

| Gate | Source | Fails the build when |
|---|---|---|
| No sellable→sellable synchronous client | ADR-005 | A Category 3 module imports another's client |
| No core→sellable synchronous client | ADR-005 | Core takes a hard dependency on an optional module |
| Topic names match `^[a-z]+\.[a-z][a-z-]*\.v[0-9]+$` | ADR-007 | A topic is off-convention |
| Every produced event type is registered | ADR-007 | A schema is missing from the registry |
| Schema compatibility is `FULL_TRANSITIVE` | ADR-007 | A change breaks either direction |
| Every `Deployment` is meshed | ADR-008 | A workload would run unencrypted |
| Negative authorization test | ADR-008 | A non-authorized service can reach the reserve endpoint |
| Account-first lock ordering | ADR-004 | A ledger write path takes bucket locks before the account row |

The last one is a static check on Vaullet's data-access layer, added because that exact defect existed
in the design and was found by review rather than by a test.

### 9. The 9-configuration matrix

ADR-005's star invariant collapses 128 module combinations to 9 meaningful ones. Those run against
ephemeral `k3d` clusters:

- **On changes to `wallet-infrastructure`** — the umbrella chart is what composition lives in
- **Nightly** on `main` of everything else

Not on every service push: spinning nine clusters per commit across fourteen repositories would
dominate CI time for a guarantee that changes only when the chart does.

### 10. Database migrations

Flyway, run as a Helm `pre-upgrade` hook Job that must complete before pods roll.

**Expand–contract, always**, and for `vaullet_db` specifically: add a column, backfill, deploy code
that writes both, then drop in a *later* release. A migration that is not backward compatible with the
running version turns a rolling update into an outage, and on the ledger it can turn it into a
correctness incident.

`ledger_entries` is append-only (ADR-004), so migrations against it are additive by construction.
Destructive DDL on `vaullet_db` requires an explicit override flag in the pipeline — the friction is
the point.

### 11. Secrets

External Secrets Operator, syncing from the customer's own secret store into the cluster. `wallet-gitops`
holds references, never values. This is already assumed by ADR-006 (Keycloak realm) and ADR-008
(service client secrets).

## Consequences

### Positive Consequences

✅ **One pipeline definition, fourteen consumers** — a security step ships everywhere by bumping one tag
✅ **Deployment state is reviewable Git**, not an emergent property of build history
✅ **Rollback is `git revert`**, made real by immutable tags
✅ **No vendor access into customer clusters** — the trust direction is outward-only
✅ **Composition is not re-invented** — Argo consumes the same ADR-005 values that define the contract
✅ **Drift self-heals**, so a manual cluster edit cannot silently become the state
✅ **Architectural rules are enforced, not reviewed** — the ADR-005/007/008 invariants are build failures
✅ **Customer upgrades are per-customer decisions**, matching how the product is actually sold

### Negative Consequences

⚠️ **Argo CD is another component per cluster**, with its own upgrade cadence — the recurring cost of
the single-tenant model, paid again.

⚠️ **Fleet-wide upgrades are N pull requests.** Deliberate, but it means upgrading ten customers is ten
reviews, and that scales linearly with the customer count.

⚠️ **Reusable workflows are a single point of failure.** A bug in `service-ci.yml` breaks fourteen
pipelines at once. Tag pinning contains it; it does not eliminate it.

⚠️ **Cross-repo changes need choreography.** A change spanning shared-contracts and three services is
four PRs in dependency order, which is the standing cost of polyrepo and was accepted in ADR-002.

⚠️ **The 9-config matrix is expensive** even nightly — nine ephemeral clusters, and it will get slower
as modules are added.

⚠️ **GitOps repo access is production access.** Whoever can merge there can change what customers run,
so it needs branch protection and review discipline equal to the code repositories.

### Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| GitOps repo compromised | Attacker changes every customer's deployment | Branch protection, required reviews, signed commits; Argo verifies cosign signatures on admission so an unsigned image will not run |
| Reusable workflow broken | All fourteen pipelines fail | Pinned tags; the workflow is itself tested in `wallet-infrastructure` before tagging |
| Migration hook fails mid-rollout | Deployment stuck, possibly with partial schema | Migrations are transactional and idempotent; the hook fails the sync rather than proceeding; expand–contract means the old version still runs |
| Customer cluster drifts from Git | Undiagnosable production behaviour | `selfHeal: true`; sync status alerted |
| Promotion PR merged for the wrong customer | Unsold module deployed, or an unexpected upgrade | PRs generated from a template, one customer per PR, values diff shown in the PR body |
| Image tag reused | Cluster and Git disagree while appearing identical | Immutable tags enforced at the registry; the pipeline refuses to overwrite |

## Alternatives Considered

### Alternative 1: Monorepo

**Description**: One repository for all services.

**Pros**: Atomic cross-service changes; one pipeline; no contract-versioning choreography; trivially
consistent tooling. Genuinely simpler for a solo developer or a small team.

**Cons**: Reverses ADR-002 and the polyrepo decision in `arch.md`, and undermines ADR-005's model in
which a sellable module is an independently released unit. Needs build tooling (Bazel, Nx) to avoid
rebuilding everything on every change.

**Why rejected — with reservations**: This is a real trade and the monorepo case is stronger than it
is usually given credit for. It is rejected because module boundaries being *repository* boundaries is
what keeps sellable units and deployable units aligned, which is the property ADR-005 depends on
commercially.

---

### Alternative 2: Push-based deployment from CI (`kubectl apply` / `helm upgrade`)

**Description**: The service pipeline deploys directly to clusters.

**Pros**: Fewer moving parts; no GitOps repo; immediate feedback in the build.

**Cons**: CI needs credentials into every customer cluster — the exact inbound vendor access rejected
in §6. Deployed state exists only as build history, so drift is invisible and rollback means finding
and re-running an old build.

**Why rejected**: Same isolation reasoning as single-tenant deployment, and it makes cluster state
unknowable.

---

### Alternative 3: Argo CD hub-and-spoke

**Description**: One central Argo managing all customer clusters.

**Pros**: One Argo to operate and upgrade; a single view of the fleet; materially lower per-cluster cost.

**Cons**: Central cluster-admin into every customer environment, and an inbound path from vendor
infrastructure into licensed operators' clusters. Its compromise reaches every customer at once.

**Why rejected**: Contradicts the isolation model. The per-cluster cost is the price of the property
that makes the deployment model defensible.

---

### Alternative 4: Flux instead of Argo CD

**Description**: Flux CD as the GitOps engine.

**Pros**: Lighter footprint; excellent Helm and OCI support; strong multi-tenancy primitives; arguably
a better fit for many small clusters.

**Cons**: No comparable UI, which matters when a support engineer needs to see what a specific customer
is running without cluster access.

**Why rejected — narrowly**: Genuinely close, and the lighter footprint argues for it given the cost
floor. Argo is chosen for operator visibility. **Worth revisiting if per-cluster resource cost becomes
the binding constraint**, since the GitOps repo layout here is engine-agnostic.

---

### Alternative 5: Auto-deploy to customer production on merge

**Description**: Continuous deployment all the way to customers.

**Pros**: Shortest lead time; smallest batches; the standard SaaS ideal.

**Cons**: A customer's version is part of a contract. Fleet-wide automatic rollout means an unrelated
merge changes what a licensed operator runs, with no per-customer decision point and a blast radius of
every customer at once.

**Why rejected**: Correct for single-tenant SaaS, wrong for contracted per-customer deployments.

## Implementation Notes

### The shared workflow

```yaml
# wallet-infrastructure/.github/workflows/service-ci.yml
on:
  workflow_call:
    inputs:
      service:      { required: true,  type: string }
      java-version: { required: false, type: string, default: '21' }

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: "${{ inputs.java-version }}", distribution: temurin, cache: maven }
      - run: ./mvnw -B verify                    # unit + integration (Testcontainers)
      - run: ./mvnw -B test -Dtest='Arch*Test'   # ADR-005/007/008 invariants

  publish:
    needs: verify
    if: github.ref == 'refs/heads/main'
    permissions: { contents: read, packages: write, id-token: write }
    steps:
      - id: meta
        run: echo "tag=sha-${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/vaullet/${{ inputs.service }}:${{ steps.meta.outputs.tag }}
          provenance: true
          sbom: true
      - uses: sigstore/cosign-installer@v3
      - run: cosign sign --yes ghcr.io/vaullet/${{ inputs.service }}:${{ steps.meta.outputs.tag }}
      - run: |
          helm package charts/${{ inputs.service }} --version 0.0.0-${{ steps.meta.outputs.tag }}
          helm push ./*.tgz oci://ghcr.io/vaullet/charts

  bump-dev:
    needs: publish
    steps:
      - uses: actions/checkout@v4
        with: { repository: vaullet/wallet-gitops, token: "${{ secrets.GITOPS_TOKEN }}" }
      - run: |
          yq -i '.images.${{ inputs.service }} = "sha-${GITHUB_SHA::7}"' envs/dev/values.yaml
          git commit -am "chore(${{ inputs.service }}): dev → sha-${GITHUB_SHA::7}" && git push
```

Trivy scanning and dependency review run in `verify`; failing severities block the merge rather than
the deploy.

### Customer promotion

A scheduled workflow in `wallet-gitops` diffs the staging umbrella version against each customer's
pinned version and opens one PR per customer, with the values diff and the umbrella changelog in the
body. Merging is the deploy.

### Shared contracts

Publishing a new contracts version deploys nothing. Consumers receive a Renovate PR and upgrade when
they choose — the independent-upgrade property ADR-002 exists to protect. The same pipeline registers
JSON Schemas with the registry (ADR-007) using `auto.register.schemas=false` at runtime, so CI is the
only writer.

### Bootstrapping a new customer

1. Terraform provisions the cluster and managed dependencies
2. Argo CD is installed and pointed at `wallet-gitops/customers/<name>`
3. A values file is generated from the signed contract (modules, currency, auth provider)
4. Argo reconciles; databases and schemas are provisioned per ADR-003's conditional rules

Steps 3 and 4 are the entire onboarding, which is the payoff for keeping composition declarative.

## References

- [ADR-002: Shared Contracts](002-shared-contracts-versioning-strategy.md) — polyrepo, independent upgrades, Maven/NPM publishing
- [ADR-003: Hybrid Database Strategy](003-hybrid-database-strategy-with-analytics.md) — conditional schema provisioning at bootstrap
- [ADR-004: Atomic Balance Reservations](004-atomic-balance-reservations.md) — the lock-ordering rule now enforced in CI
- [ADR-005: Module Composition](005-module-composition-and-deployment-topology.md) — Helm values as the contract; the 9-configuration matrix
- [ADR-007: Kafka Topics](007-kafka-topics-and-event-schema-evolution.md) — topic linting and registry gates
- [ADR-008: Service-to-Service Authentication](008-service-to-service-authentication.md) — mesh and authorization gates

---

**Date**: 2026-09-01
**Author**: Predrag
**Reviewers**: TBD
