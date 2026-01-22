# Metadata Handling in Cogento

## Overview

Stripe Customer and Subscription metadata is stored as JSONB in PostgreSQL. This document covers:
- How metadata is stored (v1 approach)
- Schema design for metadata storage
- How to add new metadata attributes
- Best practices and implementation patterns

## V1 Decision: GIN Index on JSONB (Maximum Flexibility)

**For v1, Cogento uses the most flexible approach:**
- ✅ All metadata stored as JSONB with GIN indexes
- ✅ No flattened columns (keeps schema simple)
- ✅ Maximum flexibility for adding new attributes
- ✅ Can optimize later if performance becomes an issue

**Metadata Storage:**
- **Customer metadata** → Stored in `users.metadata` JSONB column
- **Subscription metadata** → Stored in `subscriptions.metadata` JSONB column

## Metadata Search Options

### Option 1: GIN Index on JSONB Only ✅ **V1 APPROACH**

**Approach:**
- Store all metadata as JSONB in users/subscriptions tables
- Use GIN (Generalized Inverted Index) on JSONB column
- Query using JSONB operators: `metadata->>'field'`, `metadata @>`, etc.

**Pros:**
- ✅ Simple - all metadata in one field
- ✅ Flexible - any metadata structure
- ✅ Stripe-native - mirrors Stripe's metadata structure
- ✅ Good for full JSONB queries and containment searches
- ✅ **Maximum flexibility** - add any attribute without schema changes
- ✅ **No migrations needed** - just add to Stripe metadata

**Cons:**
- ⚠️ Index size can be large (GIN indexes are bigger)
- ⚠️ Less efficient for exact match queries on specific fields
- ⚠️ Complex queries may be slower than indexed columns
- ⚠️ Harder to do range queries, aggregations on metadata fields

**V1 Schema Example:**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    stripe_customer_id VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    phone VARCHAR(50),
    address JSONB,
    metadata JSONB NOT NULL,  -- Customer metadata from Stripe
    stripe_created_at TIMESTAMP WITH TIME ZONE,
    stripe_updated_at TIMESTAMP WITH TIME ZONE,
    synced_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_webhook_event_id VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_metadata_gin ON users USING GIN(metadata);

-- Query examples
SELECT * FROM users WHERE metadata->>'membership_type' = 'Adult';
SELECT * FROM users WHERE metadata @> '{"membership_status": "active"}'::jsonb;
```

**Subscription Metadata Storage:**
```sql
CREATE TABLE subscriptions (
    subscription_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stripe_subscription_id VARCHAR(255) UNIQUE NOT NULL,
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    product_id VARCHAR(255),
    status VARCHAR(50),
    current_period_start TIMESTAMP WITH TIME ZONE,
    current_period_end TIMESTAMP WITH TIME ZONE,
    cancel_at_period_end BOOLEAN,
    metadata JSONB NOT NULL,  -- Subscription metadata from Stripe
    synced_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_subscriptions_metadata_gin ON subscriptions USING GIN(metadata);
CREATE INDEX idx_subscriptions_user_id ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_status ON subscriptions(status);

-- Query examples
SELECT * FROM subscriptions WHERE metadata->>'subscription_type' = 'Premium';
SELECT * FROM subscriptions WHERE metadata @> '{"court_access": "all"}'::jsonb;
```

### Option 2: Flattened Columns + JSONB (Hybrid) - ✅ Recommended

**Approach:**
- Keep JSONB for full metadata (flexibility)
- Extract commonly-searched metadata fields into dedicated columns
- Index the dedicated columns for fast searches
- Use JSONB for rarely-searched or variable fields

**Pros:**
- ✅ **Best performance** - Indexed columns are fastest
- ✅ **Best of both worlds** - Fast searches + flexibility
- ✅ Easy to query - Standard SQL on columns
- ✅ Efficient aggregations, range queries, JOINs
- ✅ Small indexes - Only index what you search

**Cons:**
- ⚠️ Schema changes needed when adding new common search fields
- ⚠️ Some duplication (column + JSONB)
- ⚠️ Need to extract fields during sync

**Example:**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    stripe_customer_id VARCHAR(255) UNIQUE NOT NULL,
    
    -- Flattened metadata fields (commonly searched)
    member_number VARCHAR(50),
    membership_type VARCHAR(50),
    membership_status VARCHAR(50),
    join_date DATE,
    skill_level VARCHAR(50),
    preferred_court VARCHAR(100),
    
    -- Full metadata as JSONB (for rarely-searched fields)
    metadata JSONB,
    ...
);

CREATE INDEX idx_users_member_number ON users(member_number);
CREATE INDEX idx_users_membership_type ON users(membership_type);
CREATE INDEX idx_users_membership_status ON users(membership_status);
CREATE INDEX idx_users_join_date ON users(join_date);
CREATE INDEX idx_users_metadata_gin ON users USING GIN(metadata);  -- For other fields

-- Fast queries on flattened fields
SELECT * FROM users WHERE membership_type = 'Adult';
SELECT * FROM users WHERE membership_status = 'active' AND join_date > '2024-01-01';
SELECT COUNT(*) FROM users GROUP BY membership_type;
```

### Option 3: Normalized Metadata Table (EAV Pattern)

**Approach:**
- Separate `user_metadata` table with key-value pairs
- Each metadata field becomes a row (Entity-Attribute-Value pattern)

**Pros:**
- ✅ Very flexible - Add any metadata without schema changes
- ✅ Normalized structure
- ✅ Can search across all metadata easily

**Cons:**
- ❌ **Poor performance** - Multiple rows per user, complex JOINs
- ❌ Complex queries - Need aggregations to reconstruct user data
- ❌ Not recommended for frequent searches

**Example:**
```sql
CREATE TABLE user_metadata (
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    key VARCHAR(100) NOT NULL,
    value TEXT,
    PRIMARY KEY (user_id, key)
);

-- Query (slow, complex)
SELECT u.* FROM users u
JOIN user_metadata um ON u.user_id = um.user_id
WHERE um.key = 'membership_type' AND um.value = 'Adult';
```

### Option 4: Full-Text Search on JSONB

**Approach:**
- Use PostgreSQL full-text search on JSONB metadata
- Create tsvector from JSONB values

**Pros:**
- ✅ Good for text searches across metadata
- ✅ Supports ranking, relevance

**Cons:**
- ❌ Overkill for exact match queries
- ❌ Complex setup
- ❌ Not ideal for structured queries (type, status, dates)

## Metadata Storage Locations

**Important:** Metadata is stored in separate tables based on source:

1. **Customer Metadata** → `users.metadata` JSONB column
   - All metadata from Stripe Customer records
   - User/member-specific attributes
   - Examples: member_number, membership_type, join_date, skill_level

2. **Subscription Metadata** → `subscriptions.metadata` JSONB column
   - All metadata from Stripe Subscription records
   - Subscription-specific attributes
   - Examples: subscription_type, court_access, auto_renewal

**Why Separate?**
- Stripe stores metadata on both Customers and Subscriptions
- A user can have multiple subscriptions (different metadata per subscription)
- Keeps data normalized and queryable

## V1 Approach: GIN Index on JSONB Only

**Decision:** For v1, use GIN Index approach for maximum flexibility.

**Benefits:**
- ✅ No schema changes when adding new metadata attributes
- ✅ Maximum flexibility
- ✅ Simple implementation
- ✅ Can optimize later if needed

---

## Alternative Approach: Hybrid (Flattened Columns + JSONB) - Future Consideration

### Strategy

1. **Identify commonly-searched metadata fields:**
   - Member number (exact lookups)
   - Membership type (filtering, aggregations)
   - Membership status (filtering)
   - Join date (range queries, reporting)
   - Skill level, preferred court (if commonly filtered)
   - Any other fields used in frequent WHERE clauses or GROUP BY

2. **Flatten commonly-searched fields:**
   - Extract into dedicated columns during sync
   - Index these columns for fast queries

3. **Keep JSONB for flexibility:**
   - Store full metadata as JSONB
   - Use for rarely-searched fields
   - Use GIN index for containment queries

4. **Sync process:**
   - When syncing from Stripe, extract common fields from metadata JSONB
   - Store in dedicated columns AND in metadata JSONB
   - Keep them in sync

## Recommended Schema

### Users Table

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    stripe_customer_id VARCHAR(255) UNIQUE NOT NULL,
    
    -- Stripe Customer fields
    name VARCHAR(255),
    phone VARCHAR(50),
    address JSONB,
    
    -- Flattened metadata fields (commonly searched)
    member_number VARCHAR(50),
    membership_type VARCHAR(50),
    membership_status VARCHAR(50),
    join_date DATE,
    skill_level VARCHAR(50),
    preferred_court VARCHAR(100),
    emergency_contact_name VARCHAR(255),
    emergency_contact_phone VARCHAR(50),
    
    -- Full metadata as JSONB (source of truth + rarely-searched fields)
    metadata JSONB NOT NULL,  -- Full Stripe Customer metadata
    
    -- Stripe timestamps
    stripe_created_at TIMESTAMP WITH TIME ZONE,
    stripe_updated_at TIMESTAMP WITH TIME ZONE,
    
    -- Sync tracking
    synced_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_webhook_event_id VARCHAR(255),
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes on flattened fields (fast searches)
CREATE INDEX idx_users_member_number ON users(member_number);
CREATE INDEX idx_users_membership_type ON users(membership_type);
CREATE INDEX idx_users_membership_status ON users(membership_status);
CREATE INDEX idx_users_join_date ON users(join_date);
CREATE INDEX idx_users_skill_level ON users(skill_level);

-- GIN index on JSONB (for other metadata fields, containment queries)
CREATE INDEX idx_users_metadata_gin ON users USING GIN(metadata);

-- Other indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_stripe_customer_id ON users(stripe_customer_id);
CREATE INDEX idx_users_name ON users(name);
```

### Subscriptions Table

```sql
CREATE TABLE subscriptions (
    subscription_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stripe_subscription_id VARCHAR(255) UNIQUE NOT NULL,
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    
    -- Stripe Subscription fields
    product_id VARCHAR(255),
    status VARCHAR(50),
    current_period_start TIMESTAMP WITH TIME ZONE,
    current_period_end TIMESTAMP WITH TIME ZONE,
    cancel_at_period_end BOOLEAN,
    
    -- Flattened metadata fields (commonly searched)
    subscription_type VARCHAR(50),
    court_access VARCHAR(100),
    auto_renewal BOOLEAN,
    
    -- Full metadata as JSONB
    metadata JSONB NOT NULL,  -- Full Stripe Subscription metadata
    
    -- Sync tracking
    synced_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes on flattened fields
CREATE INDEX idx_subscriptions_status ON subscriptions(status);
CREATE INDEX idx_subscriptions_subscription_type ON subscriptions(subscription_type);
CREATE INDEX idx_subscriptions_user_id ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_stripe_subscription_id ON subscriptions(stripe_subscription_id);
CREATE INDEX idx_subscriptions_metadata_gin ON subscriptions USING GIN(metadata);
```

## Implementation

### Sync Process

**When syncing from Stripe:**

1. Store full metadata as JSONB (source of truth)
2. Extract commonly-searched fields from JSONB
3. Store in dedicated columns
4. Keep both in sync

**Example sync function:**
```python
def sync_customer_to_postgres(stripe_customer):
    metadata = stripe_customer.get('metadata', {})
    
    user_data = {
        'stripe_customer_id': stripe_customer['id'],
        'email': stripe_customer['email'],
        'name': stripe_customer.get('name'),
        'metadata': json.dumps(metadata),  # Full JSONB
        
        # Extract commonly-searched fields
        'member_number': metadata.get('member_number'),
        'membership_type': metadata.get('membership_type'),
        'membership_status': metadata.get('membership_status'),
        'join_date': parse_date(metadata.get('join_date')),
        'skill_level': metadata.get('skill_level'),
        'preferred_court': metadata.get('preferred_court'),
        # ... other common fields
    }
    
    # Upsert to PostgreSQL
    upsert_user(user_data)
```

### Sync Logic Pattern (Configurable)

```python
def sync_customer_to_postgres(stripe_customer):
    metadata = stripe_customer.get('metadata', {})
    
    # Define which fields to flatten (configurable)
    FLATTENED_FIELDS = [
        'member_number',
        'membership_type',
        'membership_status',
        'join_date',
        'skill_level',
        'preferred_court',
        # Add new fields here when flattened
    ]
    
    user_data = {
        'stripe_customer_id': stripe_customer['id'],
        'email': stripe_customer['email'],
        'metadata': json.dumps(metadata),  # Always store full JSONB
    }
    
    # Extract flattened fields (dynamic)
    for field in FLATTENED_FIELDS:
        user_data[field] = metadata.get(field)
    
    # Special handling for dates
    if 'join_date' in metadata:
        user_data['join_date'] = parse_date(metadata['join_date'])
    
    upsert_user(user_data)
```

### Query Examples

**Fast queries on flattened fields:**
```sql
-- Exact match on indexed column (very fast)
SELECT * FROM users WHERE membership_type = 'Adult';

-- Range query (fast with index)
SELECT * FROM users WHERE join_date BETWEEN '2024-01-01' AND '2024-12-31';

-- Aggregation (fast)
SELECT membership_type, COUNT(*) 
FROM users 
GROUP BY membership_type;

-- JOIN queries (fast)
SELECT u.*, s.* 
FROM users u 
JOIN subscriptions s ON u.user_id = s.user_id 
WHERE u.membership_status = 'active' 
  AND s.status = 'active';
```

**Flexible queries on JSONB:**
```sql
-- Rarely-searched fields (still works, uses GIN index)
SELECT * FROM users 
WHERE metadata->>'emergency_contact_phone' = '555-1234';

-- Containment query (uses GIN index)
SELECT * FROM users 
WHERE metadata @> '{"notes": "Preferred court 3"}'::jsonb;
```

## Adding New Metadata Attributes

### Scenario 1: Rarely-Searched Attribute (EASY - No Schema Change)

**If you just need to store the field (don't need fast searches/filters):**

**Difficulty: ✅ Very Easy (~5 minutes)**

**Steps:**
1. Start storing the new field in Stripe Customer metadata
2. Webhook syncs it to PostgreSQL `metadata` JSONB automatically
3. Done! ✅

**No database migration needed** - JSONB already supports any structure.

**Example:**
```python
# In Stripe
stripe.Customer.modify(
    customer_id,
    metadata={
        'existing_field': 'value',
        'new_field': 'new_value'  # Just add it!
    }
)
```

The new field automatically appears in PostgreSQL `metadata` JSONB after webhook sync.

**Querying:**
```sql
-- Can query immediately (uses GIN index)
SELECT * FROM users 
WHERE metadata->>'new_field' = 'new_value';

-- Or use containment
SELECT * FROM users 
WHERE metadata @> '{"new_field": "new_value"}'::jsonb;
```

**Pros:**
- ✅ Zero database changes
- ✅ Immediate availability
- ✅ Flexible structure
- ✅ No migration needed

**Cons:**
- ⚠️ Queries use GIN index (slightly slower than indexed columns)
- ⚠️ Can't easily do GROUP BY, aggregations on this field
- ⚠️ Can't easily do range queries

### Scenario 2: Commonly-Searched Attribute (MEDIUM - Schema Change)

**If you need fast searches, filters, aggregations:**

**Difficulty: ⚠️ Medium (~1-2 hours)**

**Steps Required:**

1. **Add column to users table** (migration)
2. **Create index on new column**
3. **Update sync logic** to extract field from JSONB
4. **Backfill existing data** (extract from JSONB to column)
5. **Update queries** to use new column

**Example Migration:**

```sql
-- Step 1: Add column (allows NULL initially for backfill)
ALTER TABLE users 
ADD COLUMN new_field VARCHAR(255);

-- Step 2: Backfill existing data from JSONB
UPDATE users 
SET new_field = metadata->>'new_field'
WHERE metadata ? 'new_field'  -- Only if field exists
  AND metadata->>'new_field' IS NOT NULL;

-- Step 3: Create index
CREATE INDEX idx_users_new_field ON users(new_field);

-- Step 4: (Optional) Add comment
COMMENT ON COLUMN users.new_field IS 'Extracted from metadata.new_field for fast searches';

-- Step 5: (Optional) Make NOT NULL after backfill
-- ALTER TABLE users ALTER COLUMN new_field SET NOT NULL;
```

**Update Sync Logic:**

Add the new field to your flattened fields list:

```python
FLATTENED_FIELDS = [
    'member_number',
    'membership_type',
    'membership_status',
    'join_date',
    'new_field',  # Add new field here
]
```

**Querying:**
```sql
-- Now use column (fast indexed query)
SELECT * FROM users WHERE new_field = 'value';

-- Can still use JSONB if needed
SELECT * FROM users WHERE metadata->>'new_field' = 'value';
```

**Pros:**
- ✅ Fast searches (indexed column)
- ✅ Efficient aggregations
- ✅ Range queries possible
- ✅ Standard SQL queries

**Cons:**
- ⚠️ Requires database migration
- ⚠️ Requires code changes (sync logic)
- ⚠️ Requires backfill of existing data
- ⚠️ Deployment coordination needed

**Time Estimate:**
- Simple field (string, number): ~15-30 minutes
- Migration script: ~10 minutes
- Code updates: ~10 minutes
- Testing: ~15 minutes
- **Total: ~1-2 hours**

## Decision Guide: When to Flatten?

**Flatten into column if:**
- ✅ You'll frequently search/filter by this field
- ✅ You need aggregations (GROUP BY, COUNT) on this field
- ✅ You need range queries (dates, numbers)
- ✅ It's a core membership attribute (membership_type, status, etc.)
- ✅ Performance is critical for this field

**Keep in JSONB if:**
- ✅ Rarely searched
- ✅ Just for storage/display
- ✅ Frequently changes structure
- ✅ Optional/variable field
- ✅ Don't need fast searches

## Best Practices

### 1. Start in JSONB, Flatten Later

**Recommended approach:**
1. **Initial implementation:** Store new field in JSONB only
2. **Test usage:** Use it in queries, see performance
3. **If needed:** Add column later when you need performance

**Benefit:** No upfront schema changes, can optimize later based on actual usage.

### 2. Migration Script Template

```sql
-- Migration: Add new_field to users table

BEGIN;

-- Step 1: Add column (nullable initially)
ALTER TABLE users 
ADD COLUMN new_field VARCHAR(255);

-- Step 2: Backfill from JSONB
UPDATE users 
SET new_field = metadata->>'new_field'
WHERE metadata ? 'new_field'  -- Only if field exists
  AND metadata->>'new_field' IS NOT NULL;

-- Step 3: Create index
CREATE INDEX idx_users_new_field ON users(new_field);

-- Step 4: (Optional) Add comment
COMMENT ON COLUMN users.new_field IS 'Extracted from metadata.new_field for fast searches';

COMMIT;
```

### 3. Version Control for Flattened Fields

Keep a list of flattened fields in code/docs:

```python
# config/metadata_fields.py
FLATTENED_METADATA_FIELDS = {
    'users': [
        'member_number',
        'membership_type',
        'membership_status',
        'join_date',
        'skill_level',
        'preferred_court',
        # Add new fields here when flattened
    ],
    'subscriptions': [
        'subscription_type',
        'court_access',
        # ...
    ],
}
```

### 4. Real-World Example

**Scenario:** Adding "coach_name" attribute

**Initial decision:** Store in JSONB only (not commonly searched)

```python
# Just add to Stripe metadata
stripe.Customer.modify(
    customer_id,
    metadata={'coach_name': 'John Doe'}
)
```

**Later:** Realize you need to filter by coach frequently

**Migration:**
```sql
-- Add column
ALTER TABLE users ADD COLUMN coach_name VARCHAR(255);

-- Backfill
UPDATE users SET coach_name = metadata->>'coach_name';

-- Index
CREATE INDEX idx_users_coach_name ON users(coach_name);

-- Update sync code to extract coach_name
```

**Result:** Now fast searches on coach_name, still have full JSONB for flexibility.

## Decision Matrix

| Requirement | GIN Index Only | Flattened + JSONB | Normalized Table |
|-------------|----------------|-------------------|------------------|
| **Search Performance** | Good | **Excellent** | Poor |
| **Flexibility** | Excellent | **Good** | Excellent |
| **Schema Changes** | None | Needed | None |
| **Query Complexity** | Medium | **Simple** | Complex |
| **Index Size** | Large | **Small** | Medium |
| **Aggregations** | Slower | **Fast** | Very Slow |
| **Adding New Fields** | ✅ Very Easy | ⚠️ Medium | ✅ Easy |
| **Recommendation** | ⚠️ | ✅ **Best** | ❌ |

## Summary

### V1 Approach: GIN Index on JSONB Only ✅

**For v1, Cogento uses:**
1. ✅ All metadata stored as JSONB (no flattened columns)
2. ✅ GIN indexes on `users.metadata` and `subscriptions.metadata`
3. ✅ Maximum flexibility - add any attribute without schema changes
4. ✅ Simple implementation - no complex sync logic needed
5. ✅ Can optimize later if performance becomes an issue

**Metadata Storage:**
- **Customer metadata** → `users.metadata` JSONB column
- **Subscription metadata** → `subscriptions.metadata` JSONB column

**Adding New Attributes (V1):**
- ✅ **Very Easy** - Just add to Stripe metadata, webhook syncs automatically
- ✅ **No schema changes** - JSONB supports any structure
- ✅ **No migrations** - Immediate availability
- ✅ **~5 minutes** - Just update Stripe and sync

**Querying:**
```sql
-- Customer metadata queries
SELECT * FROM users WHERE metadata->>'membership_type' = 'Adult';
SELECT * FROM users WHERE metadata @> '{"membership_status": "active"}'::jsonb;

-- Subscription metadata queries
SELECT * FROM subscriptions WHERE metadata->>'subscription_type' = 'Premium';
SELECT * FROM subscriptions WHERE metadata @> '{"court_access": "all"}'::jsonb;

-- Combined queries
SELECT u.*, s.* 
FROM users u 
JOIN subscriptions s ON u.user_id = s.user_id 
WHERE u.metadata->>'membership_status' = 'active'
  AND s.metadata->>'subscription_type' = 'Premium';
```

**Future Optimization:**
- If performance becomes an issue, can flatten commonly-searched fields into columns
- Migration process documented in "Alternative Approach" section above
- JSONB always preserved, so no data loss when optimizing

**Key Points:**
1. ✅ **V1: GIN Index only** - Maximum flexibility, simple implementation
2. ✅ **Customer metadata** in `users.metadata` JSONB
3. ✅ **Subscription metadata** in `subscriptions.metadata` JSONB
4. ✅ **Adding attributes:** Very easy - just add to Stripe, no schema changes
5. ✅ **Can optimize later:** Flatten to columns if performance needed
