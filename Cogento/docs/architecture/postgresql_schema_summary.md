# PostgreSQL Schema Summary

## Overview

Cogento uses PostgreSQL as a read replica for reporting and analytics. Data is synced from Stripe via webhooks.

## Tables

### 1. `users` Table

**Purpose:** Stores Stripe Customer data synced from Stripe.

**Key Fields:**
- `user_id` (UUID, Primary Key)
- `email` (VARCHAR, UNIQUE NOT NULL) - Used for authentication
- `stripe_customer_id` (VARCHAR, UNIQUE NOT NULL) - Reference to Stripe Customer
- `name`, `phone`, `address` - Stripe Customer fields
- `metadata` (JSONB) - **Customer metadata from Stripe**
- `stripe_created_at`, `stripe_updated_at` - Stripe timestamps
- `synced_at`, `last_webhook_event_id` - Sync tracking

**Indexes:**
- `idx_users_email` - For authentication lookups
- `idx_users_stripe_customer_id` - For Stripe lookups
- `idx_users_metadata_gin` - GIN index on metadata JSONB

### 2. `subscriptions` Table

**Purpose:** Stores Stripe Subscription data synced from Stripe.

**Key Fields:**
- `subscription_id` (UUID, Primary Key)
- `stripe_subscription_id` (VARCHAR, UNIQUE NOT NULL) - Reference to Stripe Subscription
- `user_id` (UUID, Foreign Key to `users.user_id`) - Links to user
- `product_id`, `status` - Stripe Subscription fields
- `current_period_start`, `current_period_end` - Subscription dates
- `metadata` (JSONB) - **Subscription metadata from Stripe**
- `synced_at` - Sync tracking

**Indexes:**
- `idx_subscriptions_user_id` - For user lookups
- `idx_subscriptions_stripe_subscription_id` - For Stripe lookups
- `idx_subscriptions_status` - For status filtering
- `idx_subscriptions_metadata_gin` - GIN index on metadata JSONB

### 3. `user_roles` Table

**Purpose:** Stores user roles (superuser, admin, user, member).

**Key Fields:**
- `user_id` (UUID, Foreign Key to `users.user_id`)
- `role` (VARCHAR) - 'superuser', 'admin', 'user', or 'member'
- `created_at` - When role was assigned
- Primary Key: (`user_id`, `role`) - Allows multiple roles per user

**Indexes:**
- `idx_user_roles_user_id` - For user role lookups
- `idx_user_roles_role` - For role-based queries

## Relationships

```
users (1) ──< (many) subscriptions
  │
  └──< (many) user_roles
```

- One user can have many subscriptions
- One user can have many roles
- Subscriptions reference users via `user_id` foreign key
- User roles reference users via `user_id` foreign key

## Metadata Storage

**Important:** Metadata is stored separately based on source:

- **Customer metadata** → `users.metadata` JSONB column
  - All metadata from Stripe Customer records
  - User/member-specific attributes

- **Subscription metadata** → `subscriptions.metadata` JSONB column
  - All metadata from Stripe Subscription records
  - Subscription-specific attributes

**Why separate?**
- Stripe stores metadata on both Customers and Subscriptions
- A user can have multiple subscriptions (different metadata per subscription)
- Keeps data normalized and queryable

## V1 Schema (GIN Index Only)

For v1, we use GIN indexes on JSONB metadata columns (no flattened columns).

**Benefits:**
- Maximum flexibility
- No schema changes when adding new metadata attributes
- Simple implementation

## Example Queries

**Get user with subscriptions:**
```sql
SELECT u.*, s.* 
FROM users u 
LEFT JOIN subscriptions s ON u.user_id = s.user_id 
WHERE u.email = '[email protected]';
```

**Get users by customer metadata:**
```sql
SELECT * FROM users 
WHERE metadata->>'membership_type' = 'Adult';
```

**Get subscriptions by subscription metadata:**
```sql
SELECT * FROM subscriptions 
WHERE metadata->>'subscription_type' = 'Premium';
```

**Get user roles:**
```sql
SELECT u.*, ur.role 
FROM users u 
JOIN user_roles ur ON u.user_id = ur.user_id 
WHERE u.email = '[email protected]';
```

## Summary

✅ **Three main tables:**
1. `users` - Stripe Customer data + customer metadata
2. `subscriptions` - Stripe Subscription data + subscription metadata  
3. `user_roles` - User role assignments

✅ **Metadata storage:**
- Customer metadata in `users.metadata`
- Subscription metadata in `subscriptions.metadata`

✅ **V1 approach:**
- GIN indexes on JSONB metadata columns
- Maximum flexibility, simple implementation
