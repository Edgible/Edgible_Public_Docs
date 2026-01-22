# PostgreSQL Schema Design: Users Table

## Proposed Design

**User's Proposal:**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    stripe_customer_id VARCHAR(255),  -- Reference to Stripe Customer
    -- ... other fields
);
```

## Recommendations & Suggestions

### ✅ Agree with Core Design

**UUID as Primary Key:**
- ✅ Excellent choice - UUIDs are unique, don't reveal info, work well for distributed systems
- ✅ Better than email as PK (allows email changes, more flexible)
- ✅ Standard practice for user tables

**Email with Unique Constraint:**
- ✅ Perfect for enforcing email uniqueness at database level
- ✅ Allows fast indexed lookups by email
- ✅ Aligns with Cogento's email-centric approach

**stripe_customer_id Reference:**
- ✅ Correct approach - links to Stripe Customer

### 📝 Suggestions & Improvements

#### 1. stripe_customer_id Constraints

**Should be:**
- ✅ `UNIQUE NOT NULL` - One-to-one relationship (one user = one Stripe customer)
- ✅ Every user must have a Stripe customer (since registration creates Stripe Customer)

**Recommendation:**
```sql
stripe_customer_id VARCHAR(255) UNIQUE NOT NULL
```

**Note:** This is not a traditional foreign key (can't reference external Stripe DB), but acts as a unique reference/link.

#### 2. Table Naming: `users` vs `members`

**Consideration:**
- **`users`** - More accurate since table includes superusers, admins, AND members
- **`members`** - Was used in initial docs, but less accurate (not all users are members)

**Recommendation:** Use `users` - it's more accurate and aligns with your role system (superuser, admin, user, member).

#### 3. Recommended Schema

```sql
CREATE TABLE users (
    -- Primary Key
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Email (unique identifier for authentication)
    email VARCHAR(255) UNIQUE NOT NULL,
    
    -- Stripe Reference (one-to-one relationship)
    stripe_customer_id VARCHAR(255) UNIQUE NOT NULL,
    
    -- Stripe Customer Fields (synced from Stripe)
    name VARCHAR(255),
    phone VARCHAR(50),
    address JSONB,  -- Store full address object from Stripe
    
    -- Metadata Fields (from Stripe Customer metadata)
    -- These can be flattened from metadata JSONB for easier querying
    -- Or stored as JSONB and queried: metadata->>'member_number'
    metadata JSONB,  -- Full Stripe Customer metadata
    
    -- Stripe Timestamps
    stripe_created_at TIMESTAMP WITH TIME ZONE,
    stripe_updated_at TIMESTAMP WITH TIME ZONE,
    
    -- Sync Tracking
    synced_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_webhook_event_id VARCHAR(255),
    
    -- Additional Fields (if needed)
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes for Performance
CREATE INDEX idx_users_email ON users(email);  -- Email lookups (authentication)
CREATE INDEX idx_users_stripe_customer_id ON users(stripe_customer_id);  -- Stripe lookups
CREATE INDEX idx_users_metadata_gin ON users USING GIN(metadata);  -- Metadata searches
CREATE INDEX idx_users_name ON users(name);  -- Name searches
```

#### 4. Role Storage

**Question:** Where do roles (superuser, admin, user, member) get stored?

**Options:**

**✅ DECISION: Separate user_roles table (Confirmed)**

```sql
CREATE TABLE user_roles (
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL,  -- 'superuser', 'admin', 'user', 'member'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    PRIMARY KEY (user_id, role),
    CONSTRAINT valid_role CHECK (role IN ('superuser', 'admin', 'user', 'member'))
);

-- Indexes
CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
CREATE INDEX idx_user_roles_role ON user_roles(role);
```

**Benefits:**
- ✅ Supports multiple roles per user (e.g., admin + member, superuser + member)
- ✅ Clean separation of concerns
- ✅ Easy to query "all admins" or "users with specific role"
- ✅ Enforced valid roles via CHECK constraint
- ✅ Supports audit trail (created_at)

**Usage Examples:**
```sql
-- Get all roles for a user
SELECT role FROM user_roles WHERE user_id = '...';

-- Check if user is superuser
SELECT EXISTS(SELECT 1 FROM user_roles WHERE user_id = '...' AND role = 'superuser');

-- Get all admins
SELECT u.* FROM users u 
JOIN user_roles ur ON u.user_id = ur.user_id 
WHERE ur.role = 'admin';

-- Get all members
SELECT u.* FROM users u 
JOIN user_roles ur ON u.user_id = ur.user_id 
WHERE ur.role = 'member';
```

#### 5. Additional Considerations

**Email Validation:**
- ✅ Unique constraint enforces uniqueness at DB level
- ✅ Application level should also validate email format before creating Stripe Customer
- ✅ Consider email normalization (lowercase, trim whitespace) in application

**Email Changes:**
- If email changes are allowed:
  - Update in Stripe Customer record
  - Webhook will sync to PostgreSQL
  - Unique constraint will prevent duplicates

**Relationship with Subscriptions:**
- Subscriptions table will reference users via `user_id` or `stripe_customer_id`
- Recommendation: Use `user_id` for referential integrity within PostgreSQL
- Keep `stripe_customer_id` for Stripe API calls

## Summary

**Agreed Design Elements:**
- ✅ UUID as primary key (`user_id`)
- ✅ Email with unique constraint
- ✅ `stripe_customer_id` as reference to Stripe

**Confirmed Decisions:**
1. ✅ **Separate `user_roles` table** - Confirmed for role management
2. ✅ **UUID primary key** (`user_id`) with email unique constraint
3. ✅ **`stripe_customer_id` UNIQUE NOT NULL** - One-to-one relationship with Stripe Customer
4. ✅ **Table name: `users`** (not `members`) - Includes all user types
5. ✅ **Indexes:** email, stripe_customer_id, metadata (GIN), name
6. ✅ **Sync tracking fields:** synced_at, last_webhook_event_id

**Open Questions:**
1. Should email be case-insensitive? (Recommended: store lowercase in application layer)
2. What other fields from Stripe Customer should be flattened vs stored in JSONB?
3. Should roles have additional metadata (e.g., assigned_by, assigned_at, expires_at)?
