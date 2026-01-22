# Stripe + PostgreSQL Sync Architecture

## ⭐ Key Insight: Best of Both Worlds

**Stripe limitations (search, reporting, performance) are easily solved by syncing to PostgreSQL!**

- ✅ **Stripe as Source of Truth** - All writes, master database
- ✅ **PostgreSQL as Read Replica** - Synced from Stripe via webhooks for queries
- ✅ **You Already Have Webhooks!** - Your `webhooks.py` handles Stripe events

## Architecture Overview

### Master-Slave Pattern (Stripe → PostgreSQL)

```
┌─────────────────────────────────────────────────────────────┐
│                    STRIPE (SOURCE OF TRUTH)                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Customers (Member Records)                           │  │
│  │  - email, name, phone, address                        │  │
│  │  - metadata: member_number, join_date, status, etc.   │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Subscriptions (Active Memberships)                   │  │
│  │  - product_id, status, dates, auto-renewal            │  │
│  │  - metadata: membership_type, court_access, etc.      │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Invoices/Payments (Payment History)                  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ Webhooks (Real-time sync)
                          │ customer.created, customer.updated
                          │ subscription.*, invoice.*
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              POSTGRESQL (READ REPLICA / CACHE)               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  members (synced from Stripe Customers)               │  │
│  │  - stripe_customer_id (FK to Stripe)                  │  │
│  │  - All customer fields + metadata                     │  │
│  │  - Indexed for fast search/filter                     │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  subscriptions (synced from Stripe Subscriptions)     │  │
│  │  - stripe_subscription_id (FK to Stripe)              │  │
│  │  - All subscription fields                            │  │
│  │  - Indexed for fast queries                           │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  payments (synced from Stripe Invoices)               │  │
│  │  - stripe_invoice_id (FK to Stripe)                   │  │
│  │  - Payment history for reporting                      │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Stripe is Master (Source of Truth)

- **All writes go to Stripe:**
  - Create member → `stripe.Customer.create()`
  - Update member → `stripe.Customer.modify()`
  - Create subscription → `stripe.Subscription.create()`
  - Cancel subscription → `stripe.Subscription.cancel()`

- **Stripe handles payments automatically** - renewals, invoices, payment failures
- **Stripe is always correct** - If there's a discrepancy, Stripe wins

### 2. Webhooks Sync to PostgreSQL (You Already Have This!)

Your existing `webhooks.py` already handles Stripe events! Just extend it to sync to PostgreSQL:

```python
# When Stripe customer is created/updated
@router.post("/webhooks/stripe")
async def stripe_webhook(request: Request):
    event = stripe.Webhook.construct_event(...)
    
    if event.type == "customer.created":
        # Sync customer to PostgreSQL
        await sync_customer_to_postgres(event.data.object)
    
    if event.type == "customer.updated":
        # Update PostgreSQL record
        await sync_customer_to_postgres(event.data.object)
    
    if event.type == "subscription.created":
        # Sync subscription to PostgreSQL
        await sync_subscription_to_postgres(event.data.object)
    
    # ... other events
```

### 3. PostgreSQL for Queries (Fast Search & Reporting)

- **Member directory:** `SELECT * FROM members WHERE membership_status = 'active'`
- **Search by name/email:** `SELECT * FROM members WHERE name ILIKE '%smith%'`
- **Filter by metadata:** `SELECT * FROM members WHERE metadata->>'membership_type' = 'Adult'`
- **Complex reporting:** JOINs, aggregations, date ranges - all fast in PostgreSQL
- **Analytics:** Member retention, revenue reports, renewal trends

## Sync Strategy

### Webhook Events to Sync

| Stripe Event | PostgreSQL Action | Priority |
|--------------|-------------------|----------|
| `customer.created` | INSERT INTO members | HIGH |
| `customer.updated` | UPDATE members | HIGH |
| `customer.deleted` | UPDATE members SET status='deleted' (soft delete) | HIGH |
| `subscription.created` | INSERT INTO subscriptions | HIGH |
| `subscription.updated` | UPDATE subscriptions | HIGH |
| `subscription.deleted` | UPDATE subscriptions SET status='canceled' | HIGH |
| `invoice.payment_succeeded` | INSERT INTO payments | MEDIUM |
| `invoice.payment_failed` | UPDATE subscriptions SET status='past_due' | MEDIUM |

### PostgreSQL Schema Design

```sql
-- Members table (synced from Stripe Customers)
CREATE TABLE members (
    id UUID PRIMARY KEY,
    stripe_customer_id VARCHAR(255) UNIQUE NOT NULL,  -- FK to Stripe
    
    -- Built-in Stripe fields
    email VARCHAR(255),
    name VARCHAR(255),
    phone VARCHAR(50),
    address JSONB,  -- Store full address object
    
    -- Metadata (flattened for easier querying)
    member_number VARCHAR(50),
    join_date DATE,
    membership_type VARCHAR(50),
    membership_status VARCHAR(50),
    preferred_court VARCHAR(100),
    skill_level VARCHAR(50),
    emergency_contact_name VARCHAR(255),
    emergency_contact_phone VARCHAR(50),
    
    -- Full metadata as JSONB for flexibility
    metadata JSONB,
    
    -- Description field (unlimited text from Stripe)
    description TEXT,
    
    -- Stripe timestamps
    stripe_created_at TIMESTAMP WITH TIME ZONE,
    stripe_updated_at TIMESTAMP WITH TIME ZONE,
    
    -- Sync tracking
    synced_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_webhook_event_id VARCHAR(255)
);

-- Indexes for fast queries
CREATE INDEX idx_members_email ON members(email);
CREATE INDEX idx_members_stripe_customer_id ON members(stripe_customer_id);
CREATE INDEX idx_members_status ON members(membership_status);
CREATE INDEX idx_members_member_number ON members(member_number);
CREATE INDEX idx_members_metadata_gin ON members USING GIN(metadata);

-- Subscriptions table (synced from Stripe Subscriptions)
CREATE TABLE subscriptions (
    id UUID PRIMARY KEY,
    stripe_subscription_id VARCHAR(255) UNIQUE NOT NULL,
    stripe_customer_id VARCHAR(255) NOT NULL REFERENCES members(stripe_customer_id),
    
    -- Stripe subscription fields
    product_id VARCHAR(255),
    status VARCHAR(50),
    current_period_start TIMESTAMP WITH TIME ZONE,
    current_period_end TIMESTAMP WITH TIME ZONE,
    cancel_at_period_end BOOLEAN,
    
    -- Metadata
    metadata JSONB,
    
    -- Sync tracking
    synced_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_subscriptions_customer ON subscriptions(stripe_customer_id);
CREATE INDEX idx_subscriptions_status ON subscriptions(status);
```

## Benefits of This Architecture

### ✅ Advantages

- **Stripe as Source of Truth:** No confusion about which system is authoritative
- **Real-time Sync:** Webhooks keep PostgreSQL in sync instantly (you already have webhook handler!)
- **Fast Queries:** PostgreSQL indexed tables for member directory, search, filters
- **Complex Reporting:** SQL queries, JOINs, aggregations, date ranges - all possible
- **Analytics:** Member retention, revenue trends, renewal rates - all in PostgreSQL
- **Resilient:** If PostgreSQL is out of sync, re-read from Stripe (Stripe is always correct)
- **No Sync Conflicts:** Only Stripe writes, PostgreSQL is read-only (except sync operations)

### Comparison: Stripe-First vs Traditional Approach

| Aspect | Stripe-First + PostgreSQL Sync | PostgreSQL-First + Stripe Sync |
|--------|-------------------------------|-------------------------------|
| **Source of Truth** | Stripe (handles payments) | PostgreSQL (must sync payments) |
| **Payment Reliability** | ✅ Stripe handles everything | ⚠️ Must sync payment events |
| **Query Performance** | ✅ Fast PostgreSQL queries | ✅ Fast PostgreSQL queries |
| **Sync Complexity** | ✅ Simple: Stripe → PostgreSQL (read-only replica) | ⚠️ Complex: Bidirectional sync (conflict resolution needed) |
| **Data Consistency** | ✅ Stripe always correct, PostgreSQL is cache | ⚠️ Must handle conflicts between systems |

## Implementation Plan

### Phase 1: Extend Existing Webhook Handler

Your `Edgible_Stripe_API/src/routes/webhooks.py` already handles webhooks. Extend it:

1. Add sync functions: `sync_customer_to_postgres()`, `sync_subscription_to_postgres()`
2. Update webhook handlers to call sync functions
3. Handle idempotency (don't duplicate on webhook replay)

### Phase 2: Create PostgreSQL Sync Tables

1. Create `members` table (mirror of Stripe Customers)
2. Create `subscriptions` table (mirror of Stripe Subscriptions)
3. Create `payments` table (from Stripe Invoices) - optional for reporting
4. Add indexes for fast queries

### Phase 3: Initial Sync (One-Time)

1. Script to read all Stripe Customers and populate PostgreSQL
2. Script to read all Stripe Subscriptions and populate PostgreSQL
3. Verify data matches

### Phase 4: Update Member Directory UI

1. Change `/ui/users.html` to query PostgreSQL instead of Stripe API
2. Add search/filter capabilities (now fast in PostgreSQL!)
3. Add reporting/analytics pages

## Resync/Recovery Strategy

**If PostgreSQL gets out of sync:**

1. **Option 1: Replay webhooks** - Use Stripe webhook replay API to resync recent events
2. **Option 2: Full resync script** - Read all Customers/Subscriptions from Stripe and refresh PostgreSQL
3. **Option 3: Incremental sync** - Compare timestamps and sync only changed records

**Since Stripe is source of truth, you can always rebuild PostgreSQL from Stripe!**

## Summary

### This architecture solves ALL the limitations:

- ✅ **Search limitations?** → PostgreSQL indexes and queries
- ✅ **Reporting limitations?** → PostgreSQL SQL queries and analytics
- ✅ **Performance issues?** → PostgreSQL is fast for 200+ members
- ✅ **Relational data?** → PostgreSQL JOINs handle relationships

### While keeping Stripe as source of truth:

- ✅ Stripe handles all payments (renewals, invoices, failures)
- ✅ Webhooks keep PostgreSQL in sync (real-time)
- ✅ No sync conflicts (Stripe writes, PostgreSQL reads)
- ✅ If PostgreSQL breaks, rebuild from Stripe

### This is the BEST architecture for Cogento!

It gives you:

- ✅ Stripe's payment reliability
- ✅ PostgreSQL's query power
- ✅ Simple one-way sync (Stripe → PostgreSQL)
- ✅ You already have the infrastructure (webhooks, PostgreSQL, Stripe API)
