# Current Discussion State - Vaullet Architecture

**Date**: 2026-08-27
**Topic**: Auth & User Management Strategy - Deployment Modes

---

## Context: Where We Are

We've completed 3 major architectural decisions (documented as ADRs):

1. ✅ **ADR-001**: Redis distributed locking for balance consistency
2. ✅ **ADR-002**: Shared contracts with dual publishing (Java + TypeScript)
3. ✅ **ADR-003**: Hybrid database strategy (7 operational databases + data warehouse)

**Current task**: Designing authentication and user management strategy.

---

## The Critical Decision: Two Deployment Modes

### Mode A: Full Platform
You own the entire user experience:
- Users register/login on YOUR platform
- You manage: Users, Auth, Transactions, Frontend, Everything
- Example: You ARE the betting platform

### Mode B: API-Only Service (White-Label/B2B)
You're just the transaction/wallet infrastructure:
- External platforms integrate via API
- They manage: Users, Auth, Frontend
- You provide: Transaction processing, Ledger, Limits, Fraud detection
- Example: You provide wallet infrastructure to other betting platforms

---

## Services Finalized (15 Backend + Admin UI)

### Core Wallet
1. Transaction Service
2. Vaullet Service (Ledger)
3. Limits Service
4. Refund Service (part of Transaction Service - same deployment)
5. Subscription Service

### Rewards & Incentives
6. Bonus Service
7. Loyalty Service (points system, redemption)
8. Referral Service (renamed from Affiliate - user referral program)

### Security & Risk
9. Auth Service ⭐ (critical decision pending)
10. Fraud Detection Service
11. Risk Management Service (shares fraud_db)

### Supporting
12. Notification Service
13. Audit Service
14. Scheduler Service
15. Reporting ETL Service

**Admin**: Just frontend UI (aggregates data via REST APIs)

---

## Database Strategy Finalized

### 7 Operational Databases:

1. **vaullet_db** → Vaullet Service (isolated, append-only)
2. **transactions_db** → Transaction + Refund Services (isolated, high volume)
3. **fraud_db** → Fraud Detection + Risk Management (isolated, PostgreSQL + Elasticsearch)
4. **rewards_db** → Bonus + Loyalty + Referral (shared, schema-per-service)
5. **support_db** → Limits + Subscription + Notification + Audit + Scheduler (shared)
6. **auth_db** → Auth Service (isolated for security) ⭐
7. **datawarehouse_db** → Reporting ETL (eventual consistency OK)

Plus:
- **redis_cluster** (cache, locks, sessions)
- **Elasticsearch** (fraud patterns)

**Total**: 10 infrastructure components

See: `/docs/database-grouping-strategy.md`

---

## Current Discussion: Auth Service Architecture

### The Challenge

We need Auth Service to support TWO different use cases:

**Case 1: Platform Users (Mode A)**
- End users who use YOUR wallet platform
- Register with email/password
- Login, MFA, sessions
- Standard user authentication

**Case 2: External API Clients (Mode B)**
- Other platforms integrating your wallet API
- API key authentication
- They send their users' IDs (`external_user_id`)
- Multi-tenant (multiple client platforms)

### Plus: Admin Users

Two types of admins:
1. **Platform Admins**: Your employees (manage system, fraud review, config)
2. **Client Admins** (Mode B): Client's employees (manage their platform - NOT your concern)

---

## Pending Questions (Need Your Answers)

### Question 1: Which mode to start with?
- **Option A**: Build Mode A (Full Platform) first, add Mode B later?
- **Option B**: Build Mode B (API-Only) only?
- **Option C**: Both from day 1? (complex!)

### Question 2: For Mode B (API-Only)
- **Single client** (one betting platform) or **multi-tenant** (many clients)?
- How much user data do you store? (just ID, or profile info?)

### Question 3: Admin users
- Just **platform admins** (your employees)?
- Or also **client admins** (if Mode B multi-tenant)?

### Question 4: Reporting in Mode B
- Do clients get their **own dashboard**?
- Or just **API endpoints** to fetch data?
- Or you provide **embeddable widgets**?

---

## Proposed Auth Architecture (Pending Your Answers)

```sql
auth_db

-- Mode A: Platform users
users table:
├─ user_id (PK)
├─ user_type (END_USER / PLATFORM_ADMIN)
├─ email
├─ password_hash
├─ ...

-- Mode B: API clients
clients table:
├─ client_id (PK)
├─ client_name
├─ api_key
├─ api_secret_hash
├─ webhook_url
├─ ...

-- Mode B: External users (minimal tracking)
external_users table:
├─ client_id (FK)
├─ external_user_id (client's user ID)
├─ created_at
├─ PRIMARY KEY (client_id, external_user_id)
```

### Unified User Identifier Pattern

```java
public class UserIdentifier {
    private UserType type;  // PLATFORM_USER or EXTERNAL_USER
    private String userId;  // platform user ID
    private String clientId;  // (Mode B only)
    private String externalUserId;  // (Mode B only)

    public String getGlobalId() {
        if (type == PLATFORM_USER) {
            return "user:" + userId;
        } else {
            return "client:" + clientId + ":user:" + externalUserId;
        }
    }
}
```

**In ledger**:
```sql
ledger_entries:
├─ entry_id
├─ user_identifier ("user:123" or "client:abc:user:xyz")
├─ amount
└─ ...
```

This allows the same ledger to work for both modes!

---

## Recommendation

**Start with Mode A (Full Platform), design for Mode B**

**Phase 1 (Now)**: Full Platform
- User management
- Full auth system
- All services
- Frontend

**Phase 2 (Later)**: Add API-Only Mode
- Add `clients` table
- Add API key authentication
- Add multi-tenancy support
- Keep existing services, expose via API

**Why**: Easier to build full platform, then extract API, than vice versa.

---

## Next Steps

1. **Answer the 4 questions above**
2. Design complete Auth Service architecture
3. Document as ADR-004
4. Design admin permissions model
5. Design API authentication flow (for Mode B future)

---

## Files to Reference

- Architecture overview: `/arch.md`
- ADRs: `/docs/adr/001-*.md`, `002-*.md`, `003-*.md`
- Database grouping: `/docs/database-grouping-strategy.md`
- Shared contracts example: `/docs/shared-contracts-example.md`
- Event flows: Documented in earlier conversation (transaction, refund, recurring payment, fraud)

---

**Status**: Waiting for answers to 4 questions to proceed with Auth architecture design.
