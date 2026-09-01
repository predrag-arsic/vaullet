# ADR-002: Shared Contracts with Dual Publishing (Java + TypeScript)

## Status

**Accepted** (2026-08-27)

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

We will create a **shared contracts repository** (`wallet-shared-contracts`) that publishes two artifacts:

1. **Maven artifact** for Java services: `com.wallet:shared-contracts:x.y.z`
2. **NPM package** for frontends: `@wallet/contracts:x.y.z`

### Key Principles

1. **Backward compatibility by default**: New fields are always optional (`@Nullable`)
2. **Semantic versioning**:
   - MAJOR: Breaking changes (field removal, type change)
   - MINOR: New optional fields, new events
   - PATCH: Documentation, bug fixes
3. **Automated TypeScript generation**: Java DTOs are source of truth, TypeScript generated
4. **DTO suffix stripping**: `TransactionDTO.java` → `Transaction.ts` for frontends
5. **Separate Java/TS versioning**: Both published with same version number for consistency

### Repository Structure

```
wallet-shared-contracts/
├── src/main/java/com/wallet/contracts/
│   ├── dto/                  # Request/Response DTOs
│   ├── events/               # Kafka event schemas
│   └── enums/                # Shared enums
├── scripts/
│   └── generate-typescript.sh
├── dist/typescript/          # Generated TypeScript (gitignored)
├── pom.xml                   # Maven build
├── package.json              # NPM publish
└── .github/workflows/
    └── publish.yml           # Auto-publish on tag
```

### Publishing Workflow

```
1. Developer makes changes (e.g., add optional field)
2. Update version in pom.xml and package.json
3. Commit changes
4. Create Git tag: v2.2.0
5. CI/CD pipeline (GitHub Actions):
   ├─ Run tests
   ├─ Generate TypeScript from Java
   ├─ Publish Maven artifact to Artifactory/Maven Central
   └─ Publish NPM package to NPM registry
6. Services upgrade at their own pace
```

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

✅ **Single source of truth**: DTOs defined once in shared-contracts
✅ **Type safety**: Both Java and TypeScript have compile-time checks
✅ **Independent deployments**: Services upgrade contracts independently
✅ **Gradual rollout**: New fields can be adopted incrementally
✅ **Frontend-friendly**: Clean TypeScript types without DTO suffix
✅ **Automated generation**: TypeScript generated from Java (no manual sync)
✅ **Version clarity**: Semantic versioning makes breaking changes explicit
✅ **Industry standard**: Used by Stripe, Twilio, AWS SDK

### Negative Consequences

⚠️ **Backward compatibility overhead**: Must design fields as optional initially
⚠️ **Multiple versions in flight**: Services may use different contract versions temporarily
⚠️ **Build complexity**: Need Maven plugin + npm scripts for dual publishing
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

**Why Rejected**: Java DTOs are more expressive. Team prefers Java as source of truth.

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

```xml
<project>
    <groupId>com.wallet</groupId>
    <artifactId>shared-contracts</artifactId>
    <version>2.2.0</version>
    <packaging>jar</packaging>

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
  "name": "@wallet/contracts",
  "version": "2.2.0",
  "description": "Shared contracts for Vaullet walleting system",
  "main": "dist/typescript/index.ts",
  "types": "dist/typescript/index.ts",
  "files": [
    "dist/typescript/**/*"
  ],
  "scripts": {
    "build": "mvn clean package",
    "prepublishOnly": "npm run build"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/wallet/shared-contracts.git"
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
// wallet-transaction-service/pom.xml
<dependency>
    <groupId>com.wallet</groupId>
    <artifactId>shared-contracts</artifactId>
    <version>2.2.0</version>  <!-- Upgraded from 2.1.0 -->
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
    "@wallet/contracts": "^2.2.0"
  }
}

// wallet-web-frontend/src/components/TransactionList.tsx
import { Transaction, TransactionStatus } from '@wallet/contracts';

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
// shared-contracts/src/test/java/CompatibilityTest.java
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

**Migration steps**:
1. **Week 1**: Create `wallet-shared-contracts` repo
2. **Week 2**: Extract DTOs from Transaction Service (v1.0.0)
3. **Week 3**: Publish Maven + NPM packages
4. **Week 4**: Migrate Transaction Service to use shared-contracts
5. **Week 5**: Migrate Vaullet Service
6. **Ongoing**: Migrate remaining services one by one

No big-bang migration required. Services can migrate incrementally.

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