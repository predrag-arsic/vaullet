# ADR-002: Shared Contracts with Dual Publishing (Java + TypeScript)

## Status

**Accepted** (2026-08-27)
**Revised** (2026-09-02) — per-domain artifacts rather than one; event schemas are authored, not
generated from Java

## Context

The Vaullet walleting system uses a polyrepo architecture with 12+ microservices. Services need to communicate via:
- REST API calls (synchronous)
- Kafka events (asynchronous)

**Challenge**: How do we share data structures (DTOs, events) across services and frontends without code duplication?

### Requirements

1. **Type safety**: Both backend (Java) and frontend (TypeScript) should have strongly-typed contracts
2. **Single source of truth**: Define schemas once, use everywhere
3. **Independent upgrades**: Services should upgrade contracts at their own pace
4. **Backward compatibility**: New fields shouldn't break existing services
5. **Frontend-friendly**: TypeScript definitions without Java-isms (no "DTO" suffix)
6. **Versioning**: Clear semantic versioning for breaking vs non-breaking changes

### The Coordination Problem

Example: Adding `merchantCategory` field to transactions requires changes across:
- `wallet-transaction-service` (creates transactions)
- `wallet-vaullet-service` (stores in ledger)
- `wallet-fraud-detection-service` (uses for rules)
- `wallet-notification-service` (shows in emails)
- `wallet-web-frontend` (displays to users)
- `wallet-mobile-app` (displays to users)

**Without a strategy**: Manual coordination across 6 repositories, high risk of inconsistency.

## Decision

We will create a **shared contracts repository** (`wallet-shared-contracts`) publishing **one artifact
pair per domain**, in both Java and TypeScript.

### One repository, several artifacts

A single fat `shared-contracts` jar consumed by all fourteen services is the coupling
[ADR-005](005-module-composition-and-deployment-topology.md) exists to remove. ADR-005's invariant —
no sellable module depends on another sellable module — buys 128 valid deployment combinations by
construction, and it holds at *runtime* whatever this repository does, because the wire is events. But
with one artifact the *build* graph is a hub: a Bonus-only field bumps a version that Limits, Audit and
Fraud Detection all consume, and every service that wants the next unrelated fix inherits the churn.
Nothing breaks; everything moves together, which is the property polyrepo was chosen to avoid.

So the repository publishes per-domain artifacts:

| Artifact | Contains | Typical consumers |
|---|---|---|
| `contracts-common` | `Money`, `Problem`, `Cursor`, the ADR-007 envelope, shared enums | everything |
| `contracts-ledger` | reserve/settle DTOs, `ledger.*` events | Vaullet, Transaction |
| `contracts-transaction` | transaction + refund DTOs, `transaction.*`, `refund.*` events | Transaction, Fraud, Limits, Audit |
| `contracts-payments` | deposit/withdrawal/chargeback DTOs and events (ADR-009) | Transaction, PSP adapters, Bonus |
| `contracts-rewards` | `rewards.value-granted`, `bonus.*`, `loyalty.*`, `referral.*` | Bonus, Loyalty, Referral, Vaullet |
| `contracts-identity` | account anchor, `auth.*`, `risk.*` events | Auth, Risk Management, Admin UI |

Published as `com.wallet:contracts-<domain>:x.y.z` and `@wallet/contracts-<domain>`. A service depends
on the domains it actually speaks — usually two or three. `contracts-common` is the only artifact
everyone shares, and it is deliberately small enough to be stable.

This is one repository, one pipeline and one release process; only the published unit is split. The
polyrepo cost of separate contract repositories (cross-repo choreography for a single logical change)
is not worth paying, and is not paid here.

### Two sources of truth, because the two contracts differ in lifetime

| Contract | Source of truth | Generated |
|---|---|---|
| **REST DTOs** | Java classes | TypeScript, and the OpenAPI spec (ADR-011) |
| **Kafka event schemas** | **Authored JSON Schema** | Java classes and TypeScript |

For REST, Java-as-source is right and this ADR's original reasoning stands: the DTO and the controller
that serves it ship together, and a change to either without the other is a compile error.

For events it is inverted, and this is a correction. [ADR-007](007-kafka-topics-and-event-schema-evolution.md)
makes the JSON Schema registered under `FULL_TRANSITIVE` the actual wire contract, and its central
argument is that events outlive the code that wrote them — a field removed in `v3` still exists in
retained messages, and money topics are retained indefinitely. A contract that outlives every binding
should not be *defined by* one of those bindings. Generating the schema from a Java class also makes
compatibility an emergent property of a refactor: rename a field, regenerate, and the registry is asked
to accept a breaking change nobody wrote down.

Authoring the schema makes the reviewable artifact the contract itself. `schemas/<domain>/<topic>.json`
is a file in a pull request; the Java and TypeScript types are build output.

### Key Principles

1. **Backward compatibility by default**: New fields are always optional (`@Nullable`)
2. **Semantic versioning**, per artifact:
   - MAJOR: Breaking changes (field removal, type change)
   - MINOR: New optional fields, new events
   - PATCH: Documentation, bug fixes
3. **REST DTOs**: Java is the source of truth, TypeScript generated
4. **Event schemas**: JSON Schema is the source of truth, Java and TypeScript generated
5. **DTO suffix stripping**: `TransactionDTO.java` → `Transaction.ts` for frontends
6. **Java and TS share a version number** within a domain, so `contracts-ledger` 2.3.0 means the same
   thing in both ecosystems. Domains version independently of each other

### Repository Structure

```
wallet-shared-contracts/
├── schemas/                          # AUTHORED — the wire contract (ADR-007)
│   ├── common/envelope.json
│   ├── ledger/ledger.entry-recorded.v1.json
│   ├── rewards/rewards.value-granted.v1.json
│   └── ...
├── contracts-common/
│   ├── src/main/java/com/wallet/contracts/common/    # AUTHORED — shared REST types
│   └── pom.xml
├── contracts-ledger/
│   ├── src/main/java/com/wallet/contracts/ledger/    # AUTHORED — REST DTOs
│   ├── target/generated-sources/events/              # GENERATED from schemas/ledger
│   └── pom.xml
├── contracts-transaction/  contracts-payments/
├── contracts-rewards/      contracts-identity/
├── dist/typescript/<domain>/         # GENERATED (gitignored); one NPM package per domain
├── pom.xml                           # aggregator
└── .github/workflows/
    └── publish.yml                   # generate → verify → register schemas → publish changed domains
```

Only `schemas/` and the `dto/` trees are hand-written. Everything under a `generated-sources` or
`dist` path is build output, and CI fails if a generated file has been edited by hand.

### Publishing Workflow

```
1. Developer edits an authored file:
     schemas/<domain>/<topic>.json   (an event — the wire contract)
     or a DTO class                  (a REST type)
2. Update the version of the affected domain module only
3. Commit; open a PR
4. CI on the PR:
   ├─ Generate Java + TypeScript from schemas/
   ├─ Fail if any generated file differs from a committed one (no hand-edited output)
   ├─ Check every changed schema against the registry under FULL_TRANSITIVE
   └─ Run compatibility tests (old consumer × new payload, new consumer × old payload)
5. Merge, then tag: contracts-ledger-v2.2.0
6. CI on the tag:
   ├─ Register the schemas for that domain (runtime never registers — ADR-007)
   ├─ Publish com.wallet:contracts-ledger to GHCR Maven
   └─ Publish @wallet/contracts-ledger to the NPM registry
7. Services upgrade the domains they consume, at their own pace
```

Only the domains touched by a change are versioned and published. A rewards-only field never produces
a new `contracts-ledger`.

### Versioning Rules

**MINOR version (backward compatible)**:
```java
// Version 2.1.0 → 2.2.0
public class TransactionDTO {
    private String transactionId;
    private BigDecimal amount;

    @JsonProperty("merchantCategory")
    @Nullable  // ← New optional field
    private String merchantCategory;
}
```

All existing services continue to work. Services can upgrade when ready.

**MAJOR version (breaking change)**:
```java
// Version 2.9.0 → 3.0.0
public class TransactionDTO {
    private String transactionId;
    // REMOVED: private BigDecimal amount;  ← Breaking!

    @JsonProperty("amountInCents")  // New required field
    @NotNull
    private Long amountInCents;  // Changed from decimal to cents
}
```

Requires coordinated upgrade across all services (planned migration).

> **Scope correction (2026-09-01)**: the coordinated-upgrade rule above applies to **REST DTOs only**.
> For **event schemas it is superseded by [ADR-007](007-kafka-topics-and-event-schema-evolution.md)**,
> which rejects coordinated upgrades outright: in a polyrepo there is no controlled deploy order, so a
> breaking event change gets a new topic (`<domain>.<event>.v2`) and a dual-publish migration window,
> with `FULL_TRANSITIVE` registry compatibility making producer-first and consumer-first equally safe.
> This ADR still governs the artifact and its semantic version; ADR-007 governs the wire.

## Consequences

### Positive Consequences

✅ **Single source of truth**: each contract defined once — REST DTOs in Java, event schemas in JSON Schema
✅ **Type safety**: Both Java and TypeScript have compile-time checks
✅ **Independent deployments**: Services upgrade contracts independently
✅ **Gradual rollout**: New fields can be adopted incrementally
✅ **Frontend-friendly**: Clean TypeScript types without DTO suffix
✅ **Automated generation**: TypeScript generated from Java (no manual sync)
✅ **Version clarity**: Semantic versioning makes breaking changes explicit
✅ **Blast radius matches the change**: a rewards field bumps `contracts-rewards` and nothing else, so
   the build graph stops being a hub that every service upgrades through
✅ **The wire contract is reviewable**: an event schema change is a diff in an authored `.json` file,
   not an emergent consequence of renaming a Java field

### Negative Consequences

⚠️ **Backward compatibility overhead**: Must design fields as optional initially
⚠️ **Multiple versions in flight**: Services may use different contract versions temporarily
⚠️ **Build complexity**: Need Maven plugin + npm scripts for dual publishing, now across six modules
⚠️ **Two generators, two directions**: JSON Schema → Java/TS for events, Java → TS for REST DTOs. The
   split is principled but it is a thing a newcomer must be told, and the build enforces it rather than
   the convention
⚠️ **A domain boundary can be wrong**: an artifact split that cuts across a real coupling produces
   circular dependencies between contract modules. `contracts-common` absorbs anything genuinely shared;
   an architecture test forbids a cycle
⚠️ **Versioning discipline**: Team must follow semantic versioning strictly
⚠️ **TypeScript generation limitations**: Complex Java types may not map perfectly

### Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Breaking change deployed accidentally | Services crash | Automated compatibility tests in CI, semantic versioning enforcement |
| Services on very old contract versions | Missing features, inconsistent behavior | Monthly audit, deprecation warnings in old versions |
| TypeScript generation fails | Frontend build breaks | CI checks generation, fallback to manual definitions if needed |
| Versioning mistakes (wrong bump) | Confusion, breaking changes missed | Automated version bump validation, PR checks |
| Divergence between Java and TS | Type mismatches | End-to-end integration tests, contract testing |

## Alternatives Considered

### Alternative 1: Copy-Paste DTOs in Each Service

**Description**: Each service defines its own DTOs locally.

**Pros**:
- Complete independence
- No shared dependency

**Cons**:
- Code duplication across 12+ services
- Inconsistent field names (e.g., `userId` vs `user_id`)
- Breaking changes require updating every service manually
- High risk of inconsistency

**Why Rejected**: Code duplication and inconsistency risk too high for financial system.

---

### Alternative 2: OpenAPI/Swagger as Source of Truth

**Description**: Define schemas in OpenAPI YAML, generate Java + TypeScript from spec.

**Pros**:
- Language-agnostic source
- Good for API-first design
- Generates client libraries automatically

**Cons**:
- YAML is less expressive than Java (no inheritance, complex validation)
- Generators produce boilerplate code
- Two-way sync (code → spec → code) is complex
- Team more familiar with Java than OpenAPI

**Why Rejected, for REST**: Java DTOs are more expressive, and the DTO ships with the controller that
serves it. **Partially adopted for events**: the reasoning above is about REST request/response types,
where the schema and its handler are deployed together. Events are not like that — they are read years
later by code that did not exist when they were written — so for the event half this ADR now does
exactly what this alternative describes, with JSON Schema rather than OpenAPI (ADR-007 §5). See *Two
sources of truth* in the Decision.

---

### Alternative 3: Protocol Buffers (Protobuf)

**Description**: Use `.proto` files as source of truth, generate Java + TypeScript.

**Pros**:
- Very efficient serialization
- Strong typing
- Multi-language support
- Used by gRPC (may adopt later)

**Cons**:
- Learning curve for team
- JSON serialization loses Protobuf benefits
- Verbose generated code
- Overkill for REST/JSON APIs

**Why Rejected**: Not using gRPC yet. REST/JSON sufficient for now. Can migrate later if needed.

---

### Alternative 4: Monorepo

**Description**: Keep all services in one repository, share code directly.

**Pros**:
- Atomic changes across services
- No version management needed
- Easier refactoring

**Cons**:
- Already committed to polyrepo (per ADR planning discussion)
- Large repo size
- Tooling complexity (Nx, Bazel)

**Why Rejected**: Polyrepo strategy already chosen. This ADR provides solution within that constraint.

---

### Alternative 5: One artifact for every domain

**Description**: The original form of this ADR — a single `com.wallet:shared-contracts` jar and one
`@wallet/contracts` package, consumed by all fourteen services.

**Pros**:
- Simplest possible build: one version, one tag, one publish
- No domain boundaries to get wrong, and no risk of a cycle between contract modules
- A consumer can never be on mismatched versions of two contract artifacts

**Cons**:
- Every service upgrades through the same version stream; an unrelated change is still a version bump
  the whole fleet eventually takes
- The build graph becomes a hub, which is the shape [ADR-005](005-module-composition-and-deployment-topology.md)
  removes at runtime and polyrepo removes at the repository level
- A module a customer did not buy still ships its types into every service's classpath
- The version number stops carrying information: `2.9.0` says something changed somewhere

**Why Rejected**: the coupling is real and it lands precisely where this architecture spends the most
effort avoiding it. The mitigation — split the published unit, keep one repository — costs an
aggregator `pom.xml` and buys a blast radius that matches the change. The mismatched-version risk is
answered by `contracts-common` being the only cross-domain dependency and by the compatibility tests
already required here.

## Implementation Notes

### Java DTO Example

```java
package com.wallet.contracts.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Builder;
import lombok.Data;
import javax.annotation.Nullable;
import javax.validation.constraints.NotNull;
import java.math.BigDecimal;
import java.time.Instant;

/**
 * Transaction data transfer object.
 *
 * @since 2.0.0
 */
@Data
@Builder
public class TransactionDTO {

    @JsonProperty("transactionId")
    @NotNull
    private String transactionId;

    @JsonProperty("userId")
    @NotNull
    private String userId;

    @JsonProperty("amount")
    @NotNull
    private BigDecimal amount;

    @JsonProperty("currency")
    @NotNull
    private String currency;

    @JsonProperty("type")
    @NotNull
    private TransactionType type;

    @JsonProperty("status")
    @NotNull
    private TransactionStatus status;

    @JsonProperty("createdAt")
    @NotNull
    private Instant createdAt;

    /**
     * Merchant category (e.g., ELECTRONICS, GROCERIES).
     *
     * @since 2.2.0
     */
    @JsonProperty("merchantCategory")
    @Nullable  // New field, optional for backward compatibility
    private String merchantCategory;
}
```

### Generated TypeScript (dist/typescript/Transaction.ts)

```typescript
/**
 * Transaction data transfer object.
 *
 * Generated from TransactionDTO.java
 * @since 2.0.0
 */
export interface Transaction {
  transactionId: string;
  userId: string;
  amount: number;
  currency: string;
  type: TransactionType;
  status: TransactionStatus;
  createdAt: string;  // ISO 8601 timestamp

  /**
   * Merchant category (e.g., ELECTRONICS, GROCERIES).
   * @since 2.2.0
   */
  merchantCategory?: string;  // Optional
}

export enum TransactionType {
  DEPOSIT = "DEPOSIT",
  WITHDRAWAL = "WITHDRAWAL",
  TRANSFER = "TRANSFER",
  RECURRING_PAYMENT = "RECURRING_PAYMENT"
}

export enum TransactionStatus {
  PENDING = "PENDING",
  APPROVED = "APPROVED",
  COMPLETED = "COMPLETED",
  FAILED = "FAILED",
  BLOCKED_FRAUD = "BLOCKED_FRAUD"
}
```

### Maven Configuration (pom.xml)

One domain module; the aggregator `pom.xml` lists all six and holds the shared plugin configuration.

```xml
<project>
    <groupId>com.wallet</groupId>
    <artifactId>contracts-ledger</artifactId>
    <version>2.2.0</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>com.wallet</groupId>
        <artifactId>contracts-parent</artifactId>
        <version>1.0.0</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <dependency>
            <groupId>javax.validation</groupId>
            <artifactId>validation-api</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Generate TypeScript -->
            <plugin>
                <groupId>cz.habarta.typescript-generator</groupId>
                <artifactId>typescript-generator-maven-plugin</artifactId>
                <version>3.2.1263</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>generate</goal>
                        </goals>
                        <phase>process-classes</phase>
                    </execution>
                </executions>
                <configuration>
                    <jsonLibrary>jackson2</jsonLibrary>
                    <classPatterns>
                        <pattern>com.wallet.contracts.dto.*DTO</pattern>
                        <pattern>com.wallet.contracts.events.*Event</pattern>
                        <pattern>com.wallet.contracts.enums.*</pattern>
                    </classPatterns>
                    <outputFile>dist/typescript/index.ts</outputFile>
                    <outputKind>module</outputKind>
                    <removeTypeNameSuffix>DTO</removeTypeNameSuffix>
                    <mapDate>string</mapDate>
                    <mapEnum>enum</mapEnum>
                </configuration>
            </plugin>

            <!-- Publish to Maven repository -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-deploy-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

    <distributionManagement>
        <repository>
            <id>wallet-releases</id>
            <url>https://maven.wallet.internal/releases</url>
        </repository>
    </distributionManagement>
</project>
```

### NPM Configuration (package.json)

```json
{
  "name": "@wallet/contracts-ledger",
  "version": "2.2.0",
  "description": "Ledger contracts for the Vaullet walleting system",
  "main": "dist/typescript/ledger/index.ts",
  "types": "dist/typescript/ledger/index.ts",
  "files": [
    "dist/typescript/ledger/**/*"
  ],
  "peerDependencies": {
    "@wallet/contracts-common": "^1.0.0"
  },
  "scripts": {
    "build": "mvn clean package",
    "prepublishOnly": "npm run build"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/vaullet/wallet-shared-contracts.git"
  },
  "keywords": ["wallet", "contracts", "dto", "events"],
  "author": "Wallet Team",
  "license": "PRIVATE"
}
```

### CI/CD Pipeline (GitHub Actions)

```yaml
name: Publish Shared Contracts

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          registry-url: 'https://registry.npmjs.org'

      - name: Extract version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

      - name: Verify version matches pom.xml
        run: |
          POM_VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
          if [ "$POM_VERSION" != "${{ steps.version.outputs.VERSION }}" ]; then
            echo "Version mismatch: tag=${{ steps.version.outputs.VERSION }} pom=$POM_VERSION"
            exit 1
          fi

      - name: Run tests
        run: mvn test

      - name: Build and generate TypeScript
        run: mvn clean package

      - name: Publish to Maven
        run: mvn deploy
        env:
          MAVEN_USERNAME: ${{ secrets.MAVEN_USERNAME }}
          MAVEN_PASSWORD: ${{ secrets.MAVEN_PASSWORD }}

      - name: Publish to NPM
        run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Service Upgrade Example

```java
// wallet-transaction-service/pom.xml — only the domains this service speaks
<dependency>
    <groupId>com.wallet</groupId>
    <artifactId>contracts-transaction</artifactId>
    <version>2.2.0</version>  <!-- Upgraded from 2.1.0 -->
</dependency>
<dependency>
    <groupId>com.wallet</groupId>
    <artifactId>contracts-ledger</artifactId>
    <version>2.0.3</version>  <!-- Unaffected by the change above -->
</dependency>

// wallet-transaction-service/TransactionService.java
@Service
public class TransactionService {

    public TransactionDTO createTransaction(CreateTransactionRequest request) {
        return TransactionDTO.builder()
            .transactionId(generateId())
            .userId(request.getUserId())
            .amount(request.getAmount())
            .currency(request.getCurrency())
            .type(request.getType())
            .status(TransactionStatus.PENDING)
            .createdAt(Instant.now())
            .merchantCategory(request.getMerchantCategory())  // ← New field
            .build();
    }
}
```

### Frontend Usage Example

```typescript
// wallet-web-frontend/package.json
{
  "dependencies": {
    "@wallet/contracts-transaction": "^2.2.0",
    "@wallet/contracts-common": "^1.0.0"
  }
}

// wallet-web-frontend/src/components/TransactionList.tsx
import { Transaction, TransactionStatus } from '@wallet/contracts-transaction';

interface Props {
  transactions: Transaction[];
}

export const TransactionList: React.FC<Props> = ({ transactions }) => {
  return (
    <ul>
      {transactions.map(tx => (
        <li key={tx.transactionId}>
          <span>{tx.amount} {tx.currency}</span>
          {tx.merchantCategory && <span>({tx.merchantCategory})</span>}
          <span className={tx.status}>{tx.status}</span>
        </li>
      ))}
    </ul>
  );
};
```

### Backward Compatibility Testing

```java
// contracts-ledger/src/test/java/CompatibilityTest.java
@Test
public void testBackwardCompatibility_TransactionDTO() {
    // Old JSON (from v2.1.0 service)
    String oldJson = """
        {
          "transactionId": "txn-123",
          "userId": "user-456",
          "amount": 100.00,
          "currency": "USD",
          "type": "WITHDRAWAL",
          "status": "COMPLETED",
          "createdAt": "2026-08-27T10:00:00Z"
        }
        """;

    // Should deserialize successfully in v2.2.0
    ObjectMapper mapper = new ObjectMapper();
    TransactionDTO dto = mapper.readValue(oldJson, TransactionDTO.class);

    assertNotNull(dto);
    assertEquals("txn-123", dto.getTransactionId());
    assertNull(dto.getMerchantCategory());  // Optional field is null
}

@Test
public void testForwardCompatibility_TransactionDTO() {
    // New JSON (from v2.2.0 service)
    String newJson = """
        {
          "transactionId": "txn-123",
          "userId": "user-456",
          "amount": 100.00,
          "currency": "USD",
          "type": "WITHDRAWAL",
          "status": "COMPLETED",
          "createdAt": "2026-08-27T10:00:00Z",
          "merchantCategory": "ELECTRONICS"
        }
        """;

    // Old service (v2.1.0) should ignore unknown fields
    ObjectMapper mapper = new ObjectMapper();
    mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    TransactionDTO dto = mapper.readValue(newJson, TransactionDTO.class);

    assertNotNull(dto);
    assertEquals("txn-123", dto.getTransactionId());
    // merchantCategory ignored by old service (field doesn't exist)
}
```

### Deprecation Strategy

When removing fields (breaking change):

```java
/**
 * @deprecated Since 2.9.0, use {@link #amountInCents} instead.
 * Will be removed in 3.0.0.
 */
@Deprecated
@JsonProperty("amount")
@Nullable
private BigDecimal amount;

@JsonProperty("amountInCents")
@NotNull
private Long amountInCents;
```

Migration plan:
1. v2.9.0: Add `amountInCents`, deprecate `amount` (both present)
2. Services upgrade, migrate to use `amountInCents`
3. v3.0.0: Remove `amount` (breaking change)

## Monitoring and Metrics

Track in production:
- `contracts.version.by.service` - Which version each service uses
- `contracts.deserialization.errors` - Compatibility issues
- `contracts.publish.frequency` - How often contracts change
- `contracts.upgrade.lag` - Time between publish and service upgrade

Alerts:
- Service using contracts version >3 months old
- Deserialization errors >1% of requests
- Breaking change published without coordinated deployment plan

## Migration from Current State

**Current**: No shared contracts (or ad-hoc sharing)

**Migration steps**: there is nothing to migrate — no service exists yet. The build order is therefore
the greenfield one:

1. `contracts-common` — the ADR-007 envelope, `Money`, `Problem`, shared enums
2. `schemas/ledger/*.json` and `contracts-ledger` — the first domain, because the ledger is the first
   service (ADR-004)
3. The generator and the CI gates (generated-file drift, `FULL_TRANSITIVE` registry check) **before**
   the second domain, so the discipline exists while there is still only one thing to fix
4. Remaining domains as their services are built

Domains are added, not migrated to. A service consumes the domains it speaks from its first commit.

## Future Considerations

**May adopt later**:
- **GraphQL schema** as additional export format
- **Protocol Buffers** if we move to gRPC
- **JSON Schema** for validation frameworks
- **AsyncAPI** for Kafka event documentation

**Re-evaluate in**:
- 6 months: Check if TypeScript generation works well
- 1 year: Consider Protobuf if gRPC adopted

## References

- [Semantic Versioning](https://semver.org/)
- [TypeScript Generator Maven Plugin](https://github.com/vojtechhabarta/typescript-generator)
- [Stripe API Versioning](https://stripe.com/docs/api/versioning)
- [AWS SDK Versioning Strategy](https://aws.amazon.com/blogs/developer/versioning-in-the-aws-sdk-for-java-version-2-x/)
- [Consumer-Driven Contract Testing](https://martinfowler.com/articles/consumerDrivenContracts.html)

---

**Date**: 2026-08-27
**Author**: Predrag
**Reviewers**: TBD
**Next Review**: 2027-02-27 (6 months) - Evaluate TypeScript generation effectiveness