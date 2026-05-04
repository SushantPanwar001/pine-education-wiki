# Pine Coin System

## Overview

Pine Coin is the platform's virtual currency. Users purchase Pine Coins in bundles (via payment gateway or admin manual credit) and spend them to start assessments. The system is designed to be simple, non-refundable, and fully auditable.

**Conversion Rate**: 1 INR = 5 Pine Coins

## Core Concepts

### Pine Coin Balance

- Stored on the `user` table as `coinBalance` (integer, default 0)
- Updated atomically on every credit/debit operation
- Never negative — transactions that would result in a negative balance are rejected

### Coin Transaction (Ledger)

Every coin movement creates an immutable ledger entry:

| Field | Type | Description |
|-------|------|-------------|
| id | uuid | Primary key |
| userId | text | FK to user |
| type | enum | `PURCHASE`, `ADMIN_CREDIT`, `ASSESSMENT_DEBIT`, `BUNDLE_BONUS` |
| amount | integer | Positive for credits, negative for debits |
| balanceAfter | integer | Running balance after this transaction |
| description | text | Human-readable description (e.g., "Purchased Starter Pack", "JEE Mock Test") |
| referenceId | uuid (nullable) | FK to the related entity (assessment id, bundle id, etc.) |
| createdAt | timestamp | Transaction timestamp |

### Bundle

Admin-created coin packs purchasable with real currency:

| Field | Type | Description |
|-------|------|-------------|
| id | uuid | Primary key |
| name | varchar(100) | Display name (e.g., "Starter Pack") |
| description | text (nullable) | Optional description |
| priceInRupees | integer | Cost in INR (e.g., 1000) |
| coins | integer | Pine Coins awarded (e.g., 10000) |
| bonusCoins | integer (default 0) | Extra coins on top (e.g., 500) |
| isActive | boolean (default true) | Toggle visibility |
| createdAt | timestamp | Creation timestamp |
| updatedAt | timestamp | Last update timestamp |

### Exam Price

- New field `priceInCoins` (integer) on the `exam` table
- Configurable per exam by admin
- Example: 499 Pine Coins for a mock test

---

## Flows

### User Purchases a Bundle (Payment Gateway — Planned)

```
User                  Client                Backend              Payment Gateway
 │                      │                      │                      │
 │  "Buy Starter Pack"  │                      │                      │
 │─────────────────────►│                      │                      │
 │                      │  POST /api/coins/    │                      │
 │                      │  purchase            │                      │
 │                      │─────────────────────►│                      │
 │                      │                      │  Create payment      │
 │                      │                      │  order               │
 │                      │                      │─────────────────────►│
 │                      │                      │                      │
 │                      │                      │  Payment pending     │
 │                      │                      │◄─────────────────────│
 │                      │  { checkoutUrl }     │                      │
 │                      │◄─────────────────────│                      │
 │                      │                      │                      │
 │                      │  [User completes     │                      │
 │                      │   payment on PG]     │                      │
 │                      │                      │                      │
 │                      │                      │  Webhook: payment    │
 │                      │                      │  confirmed           │
 │                      │                      │◄─────────────────────│
 │                      │                      │                      │
 │                      │                      │  Credit Pine Coins   │
 │                      │                      │  + Create Coin       │
 │                      │                      │    Transaction       │
 │                      │                      │                      │
```

### Admin Manually Credits Coins

```
Admin                 Admin Panel            Backend
 │                      │                      │
 │  "Credit 5000 coins" │                      │
 │─────────────────────►│                      │
 │                      │  POST /api/admin/    │
 │                      │  users/:id/credit    │
 │                      │─────────────────────►│
 │                      │                      │
 │                      │                      │  Validate admin
 │                      │                      │  Credit coins
 │                      │                      │  + Create Coin
 │                      │                      │    Transaction
 │                      │                      │  (type: ADMIN_CREDIT)
 │                      │                      │
 │                      │  { newBalance }      │
 │                      │◄─────────────────────│
 │                      │                      │
```

### User Starts an Assessment (Coin Deduction)

```
User                  Client                Backend
 │                      │                      │
 │  "Start Exam"        │                      │
 │─────────────────────►│                      │
 │                      │  POST /api/          │
 │                      │  assessment          │
 │                      │  { examId }          │
 │                      │─────────────────────►│
 │                      │                      │
 │                      │                      │  Check balance >=
 │                      │                      │  exam.priceInCoins
 │                      │                      │
 │                      │                      │  [In a single
 │                      │                      │   transaction:]
 │                      │                      │  1. Debit coins
 │                      │                      │  2. Create Coin
 │                      │                      │     Transaction
 │                      │                      │     (type: ASSESSMENT_DEBIT)
 │                      │                      │  3. Create Assessment
 │                      │                      │  4. Create Responses
 │                      │                      │  5. Create Section
 │                      │                      │     Attempts
 │                      │                      │
 │                      │  { assessment }      │
 │                      │◄─────────────────────│
 │  Assessment starts   │                      │
 │◄─────────────────────│                      │
```

**If insufficient balance:**
```
                      │                      │  balance < priceInCoins
                      │                      │  → 400 Bad Request
                      │                      │  { error: "Insufficient balance",
                      │                      │    required: 499,
                      │                      │    current: 250 }
```

### User Views Coin History

```
User                  Client                Backend
 │                      │                      │
 │  "My Transactions"   │                      │
 │─────────────────────►│                      │
 │                      │  GET /api/coins/     │
 │                      │  transactions        │
 │                      │─────────────────────►│
 │                      │                      │
 │                      │  { transactions[],   │
 │                      │    balance }         │
 │                      │◄─────────────────────│
```

---

## Database Changes (Migration Required)

### New Tables

#### `coin_transaction`
```sql
CREATE TABLE coin_transaction (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id TEXT NOT NULL REFERENCES "user"(id),
  type VARCHAR(20) NOT NULL, -- PURCHASE, ADMIN_CREDIT, ASSESSMENT_DEBIT, BUNDLE_BONUS
  amount INTEGER NOT NULL,   -- positive = credit, negative = debit
  balance_after INTEGER NOT NULL,
  description TEXT,
  reference_id UUID,         -- FK to assessment or bundle
  created_at TIMESTAMP DEFAULT now()
);

CREATE INDEX idx_coin_transaction_user_id ON coin_transaction(user_id);
CREATE INDEX idx_coin_transaction_created_at ON coin_transaction(created_at);
```

#### `bundle`
```sql
CREATE TABLE bundle (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  description TEXT,
  price_in_rupees INTEGER NOT NULL,
  coins INTEGER NOT NULL,
  bonus_coins INTEGER DEFAULT 0,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP DEFAULT now()
);
```

### Modified Tables

#### `user` — add coin balance, remove subscription
```sql
ALTER TABLE "user" ADD COLUMN coin_balance INTEGER DEFAULT 0;
ALTER TABLE "user" DROP COLUMN subscription_tier; -- vestigial, no longer used
```

#### `exam` — add price
```sql
ALTER TABLE exam ADD COLUMN price_in_coins INTEGER NOT NULL DEFAULT 499;
```

---

## API Endpoints (Planned)

### User Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/coins/balance` | Get current Pine Coin balance |
| GET | `/api/coins/transactions` | List transaction history (paginated) |
| POST | `/api/coins/purchase` | Purchase a bundle (initiates payment) |

### Admin Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/admin/bundles` | List all bundles |
| POST | `/api/admin/bundles` | Create a bundle |
| PATCH | `/api/admin/bundles/:id` | Update a bundle |
| DELETE | `/api/admin/bundles/:id` | Delete/deactivate a bundle |
| POST | `/api/admin/users/:id/credit` | Manually credit coins to a user |
| GET | `/api/admin/coins/transactions` | View all coin transactions (filterable) |

### Webhook Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/webhooks/payment` | Payment gateway confirmation webhook |

---

## Error Codes

| Code | HTTP | Description |
|------|------|-------------|
| `INSUFFICIENT_BALANCE` | 400 | User's coin balance is below the exam's price |
| `INVALID_BUNDLE` | 400 | Bundle not found or inactive |
| `PAYMENT_PENDING` | 402 | Payment initiated but not confirmed |
| `PAYMENT_FAILED` | 400 | Payment gateway rejected the transaction |

---

## Implementation Priority

1. **Phase 1 — Foundation**: `coin_transaction` table, `coinBalance` on user, `priceInCoins` on exam, atomic deduction in assessment creation
2. **Phase 2 — Admin Credit**: Admin endpoint to manually credit coins, transaction history endpoints
3. **Phase 3 — Bundles**: Bundle CRUD, bundle listing for users
4. **Phase 4 — Payment Gateway**: Razorpay/Stripe integration, webhook handling
5. **Phase 5 — Ad Rewards**: Ad verification, reward crediting (deferred)
