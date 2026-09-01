# Shared Contracts Repository - Quick Reference

## Overview

The `wallet-shared-contracts` repository is your single source of truth for all data structures shared across services and frontends.

**Publishes to**:
- Maven (for Java services): `com.wallet:shared-contracts:x.y.z`
- NPM (for frontends): `@wallet/contracts:x.y.z`

## Directory Structure

```
wallet-shared-contracts/
├── src/main/java/com/wallet/contracts/
│   ├── dto/                           # Request/Response DTOs
│   │   ├── TransactionDTO.java        → Transaction.ts
│   │   ├── UserDTO.java               → User.ts
│   │   ├── BalanceResponseDTO.java    → BalanceResponse.ts
│   │   └── LimitCheckRequestDTO.java  → LimitCheckRequest.ts
│   │
│   ├── events/                        # Kafka Event Schemas
│   │   ├── TransactionCreatedEvent.java
│   │   ├── RefundInitiatedEvent.java
│   │   ├── BonusAppliedEvent.java
│   │   └── FraudSuspicionDetectedEvent.java
│   │
│   └── enums/                         # Shared Enumerations
│       ├── TransactionStatus.java
│       ├── TransactionType.java
│       ├── Currency.java
│       └── RefundReason.java
│
├── dist/typescript/                   # Generated TypeScript (gitignored)
│   └── index.ts                       # All types exported here
│
├── scripts/
│   └── generate-typescript.sh         # Manual TS generation (optional)
│
├── pom.xml                            # Maven build
├── package.json                       # NPM publish
└── README.md
```

## Example: Adding a New Field

### Scenario: Add `merchantCategory` to TransactionDTO

#### Step 1: Update Java DTO (backward compatible)

```java
// src/main/java/com/wallet/contracts/dto/TransactionDTO.java
@Data
@Builder
public class TransactionDTO {
    @NotNull
    private String transactionId;

    @NotNull
    private BigDecimal amount;

    @NotNull
    private String currency;

    // NEW FIELD (v2.2.0) - Always add as @Nullable for backward compatibility
    @JsonProperty("merchantCategory")
    @Nullable
    private String merchantCategory;
}
```

#### Step 2: Update Version

```xml
<!-- pom.xml -->
<version>2.2.0</version>  <!-- Was 2.1.0 -->
```

```json
// package.json
{
  "version": "2.2.0"  // Was 2.1.0
}
```

#### Step 3: Commit and Tag

```bash
git add .
git commit -m "feat: add merchantCategory to TransactionDTO (v2.2.0)"
git tag v2.2.0
git push origin main --tags
```

#### Step 4: CI/CD Auto-Publishes

GitHub Actions pipeline automatically:
1. Runs tests
2. Generates TypeScript from Java
3. Publishes Maven artifact
4. Publishes NPM package

#### Step 5: Services Upgrade Independently

**Transaction Service** (upgrade immediately):
```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.wallet</groupId>
    <artifactId>shared-contracts</artifactId>
    <version>2.2.0</version>  <!-- Upgraded -->
</dependency>
```

```java
// Now can use the new field
TransactionDTO dto = TransactionDTO.builder()
    .transactionId("txn-123")
    .amount(new BigDecimal("100.00"))
    .currency("USD")
    .merchantCategory("ELECTRONICS")  // NEW
    .build();
```

**Notification Service** (upgrade next week):
```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.wallet</groupId>
    <artifactId>shared-contracts</artifactId>
    <version>2.1.0</version>  <!-- Still on old version -->
</dependency>
```

Still works! The `merchantCategory` field is optional, so old services ignore it.

**Frontend** (upgrade when ready):
```json
// package.json
{
  "dependencies": {
    "@wallet/contracts": "^2.2.0"
  }
}
```

```typescript
import { Transaction } from '@wallet/contracts';

// TypeScript now knows about merchantCategory
const tx: Transaction = {
  transactionId: "txn-123",
  amount: 100.00,
  currency: "USD",
  merchantCategory: "ELECTRONICS"  // Autocomplete works!
};
```

## Usage Examples

### Java Service (Backend)

```java
// Import from shared-contracts
import com.wallet.contracts.dto.TransactionDTO;
import com.wallet.contracts.events.TransactionCreatedEvent;
import com.wallet.contracts.enums.TransactionStatus;

@Service
public class TransactionService {

    public TransactionDTO createTransaction(CreateRequest request) {
        // Use DTOs
        TransactionDTO dto = TransactionDTO.builder()
            .transactionId(UUID.randomUUID().toString())
            .userId(request.getUserId())
            .amount(request.getAmount())
            .currency(request.getCurrency())
            .status(TransactionStatus.PENDING)
            .createdAt(Instant.now())
            .build();

        // Publish event
        TransactionCreatedEvent event = TransactionCreatedEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType("TRANSACTION_CREATED")
            .timestamp(Instant.now())
            .payload(dto)
            .build();

        kafkaTemplate.send("transaction.created.v1", event);   // topic per ADR-007

        return dto;
    }
}
```

### React Frontend (TypeScript)

```typescript
// Import from NPM package (no "DTO" suffix!)
import {
  Transaction,
  TransactionStatus,
  TransactionCreatedEvent
} from '@wallet/contracts';

interface TransactionListProps {
  transactions: Transaction[];
}

export const TransactionList: React.FC<TransactionListProps> = ({ transactions }) => {
  return (
    <div>
      {transactions.map(tx => (
        <div key={tx.transactionId}>
          <span>{tx.amount} {tx.currency}</span>
          <span className={getStatusColor(tx.status)}>
            {tx.status}
          </span>
        </div>
      ))}
    </div>
  );
};

// Type-safe status checking
function getStatusColor(status: TransactionStatus): string {
  switch (status) {
    case TransactionStatus.COMPLETED:
      return 'green';
    case TransactionStatus.PENDING:
      return 'yellow';
    case TransactionStatus.FAILED:
      return 'red';
    default:
      return 'gray';
  }
}
```

### Kafka Consumer (Event Handling)

```java
import com.wallet.contracts.events.TransactionCreatedEvent;

@Service
public class NotificationConsumer {

    @KafkaListener(topics = "transaction.created.v1")
    public void handleTransactionCreated(TransactionCreatedEvent event) {
        // Event schema is shared, type-safe
        String userId = event.getPayload().getUserId();
        BigDecimal amount = event.getPayload().getAmount();

        emailService.send(
            userId,
            "transaction-created",
            Map.of(
                "amount", amount,
                "transactionId", event.getPayload().getTransactionId()
            )
        );
    }
}
```

## Versioning Rules (Quick Reference)

| Change Type | Version Bump | Example | Backward Compatible? |
|-------------|--------------|---------|---------------------|
| Add optional field | MINOR (2.1.0 → 2.2.0) | Add `@Nullable String merchantCategory` | ✅ Yes |
| Add new event | MINOR (2.1.0 → 2.2.0) | Add `BonusExpiredEvent.java` | ✅ Yes |
| Add new enum value | MINOR (2.1.0 → 2.2.0) | Add `REFUNDED` to TransactionStatus | ✅ Yes |
| Deprecate field | MINOR (2.1.0 → 2.2.0) | Mark field `@Deprecated` | ✅ Yes |
| Remove field | MAJOR (2.9.0 → 3.0.0) | Delete `amount` field | ❌ No |
| Rename field | MAJOR (2.9.0 → 3.0.0) | `userId` → `customerId` | ❌ No |
| Change type | MAJOR (2.9.0 → 3.0.0) | `BigDecimal amount` → `Long amountInCents` | ❌ No |
| Remove enum value | MAJOR (2.9.0 → 3.0.0) | Remove `CANCELLED` status | ❌ No |

## Common Patterns

### Pattern 1: Adding Optional Field (Backward Compatible)

```java
// v2.1.0 → v2.2.0 (MINOR bump)
public class TransactionDTO {
    @NotNull
    private String transactionId;

    @JsonProperty("newField")
    @Nullable  // ← Must be nullable initially
    private String newField;
}
```

### Pattern 2: Making Field Required (Two-Phase)

**Phase 1** (v2.2.0 - Add as optional):
```java
@JsonProperty("merchantCategory")
@Nullable
private String merchantCategory;
```

**Phase 2** (v3.0.0 - Make required, MAJOR bump):
```java
@JsonProperty("merchantCategory")
@NotNull  // Now required
private String merchantCategory;
```

### Pattern 3: Changing Field Type (Three-Phase)

**Phase 1** (v2.8.0 - Add new field, deprecate old):
```java
/**
 * @deprecated Use amountInCents instead. Will be removed in 3.0.0.
 */
@Deprecated
@Nullable
private BigDecimal amount;

@JsonProperty("amountInCents")
@Nullable
private Long amountInCents;  // New field
```

**Phase 2** (v2.9.0 - Populate both fields):
```java
// Services write to both fields during transition
dto.setAmount(new BigDecimal("100.00"));
dto.setAmountInCents(10000L);
```

**Phase 3** (v3.0.0 - Remove old field, MAJOR bump):
```java
// amount field deleted

@JsonProperty("amountInCents")
@NotNull
private Long amountInCents;
```

## Event Schema Example

```java
package com.wallet.contracts.events;

@Data
@Builder
public class TransactionCreatedEvent {

    @NotNull
    private String eventId;

    @NotNull
    private String eventType;  // "TRANSACTION_CREATED"

    @NotNull
    private Instant timestamp;

    @NotNull
    private String version;  // "1.0"

    @NotNull
    private TransactionPayload payload;

    @Data
    @Builder
    public static class TransactionPayload {
        @NotNull
        private String transactionId;

        @NotNull
        private String userId;

        @NotNull
        private BigDecimal amount;

        @NotNull
        private String currency;

        @NotNull
        private TransactionType type;

        @NotNull
        private TransactionStatus status;

        @NotNull
        private Instant createdAt;

        @Nullable
        private String merchantCategory;  // Added in v2.2.0
    }
}
```

## TypeScript Output Example

```typescript
// Generated: dist/typescript/index.ts

/**
 * Transaction data transfer object
 */
export interface Transaction {
  transactionId: string;
  userId: string;
  amount: number;
  currency: string;
  type: TransactionType;
  status: TransactionStatus;
  createdAt: string;  // ISO 8601
  merchantCategory?: string;  // Optional (Nullable in Java)
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

export interface TransactionCreatedEvent {
  eventId: string;
  eventType: string;
  timestamp: string;
  version: string;
  payload: TransactionPayload;
}

export interface TransactionPayload {
  transactionId: string;
  userId: string;
  amount: number;
  currency: string;
  type: TransactionType;
  status: TransactionStatus;
  createdAt: string;
  merchantCategory?: string;
}
```

## Testing Compatibility

```java
// shared-contracts/src/test/java/BackwardCompatibilityTest.java

@Test
public void oldServiceCanDeserializeNewContract() {
    // JSON from new service (v2.2.0)
    String json = """
        {
          "transactionId": "txn-123",
          "amount": 100.00,
          "currency": "USD",
          "merchantCategory": "ELECTRONICS"
        }
        """;

    // Old service config (ignore unknown fields)
    ObjectMapper mapper = new ObjectMapper()
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    // Should deserialize without error
    TransactionDTO dto = mapper.readValue(json, TransactionDTO.class);
    assertNotNull(dto);
}

@Test
public void newServiceCanDeserializeOldContract() {
    // JSON from old service (v2.1.0) - missing merchantCategory
    String json = """
        {
          "transactionId": "txn-123",
          "amount": 100.00,
          "currency": "USD"
        }
        """;

    // New service can handle it (field is @Nullable)
    ObjectMapper mapper = new ObjectMapper();
    TransactionDTO dto = mapper.readValue(json, TransactionDTO.class);

    assertNotNull(dto);
    assertNull(dto.getMerchantCategory());  // Optional field is null
}
```

## Quick Commands

```bash
# Clone shared-contracts repo
git clone git@github.com:wallet/wallet-shared-contracts.git
cd wallet-shared-contracts

# Build and generate TypeScript locally
mvn clean package

# Check generated TypeScript
cat dist/typescript/index.ts

# Publish new version (CI/CD will do this automatically)
git tag v2.3.0
git push origin v2.3.0

# Use in a service
cd ../wallet-transaction-service
# Update pom.xml version to 2.3.0
mvn clean install
```

## Common Questions

**Q: When should I create a MAJOR version?**
A: Only when you have a breaking change (remove field, change type, rename field). Try to avoid breaking changes by using deprecation.

**Q: Can I add a required field?**
A: Not directly. First add as optional (MINOR), wait for all services to upgrade, then make required (MAJOR).

**Q: What if I forget to make a field @Nullable?**
A: Old services will crash when they receive the new field. Always make new fields @Nullable initially.

**Q: How do I know which services are on which version?**
A: Set up monitoring/metrics to track `contracts.version.by.service`. You can also check pom.xml in each service repo.

**Q: Should events be versioned separately from DTOs?**
A: No, they share the same version. A new event is a MINOR bump, same as a new field.

**Q: What about frontend-specific types that backend doesn't need?**
A: Create them manually in your frontend repo. Shared-contracts is only for shared types.
