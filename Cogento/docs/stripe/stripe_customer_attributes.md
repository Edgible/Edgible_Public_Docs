# Stripe Customer Attributes Mapping

## Overview

This document summarizes Stripe Customer object attributes and how they should be mapped to PostgreSQL in Cogento.

## Stripe Customer Object Attributes

### Core Identity Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `id` | string | Unique customer identifier (e.g., `cus_xxx`) | ✅ **YES** → `stripe_customer_id` (UNIQUE, indexed) |
| `email` | string | Customer's email address | ✅ **YES** → `email` (UNIQUE, indexed - used for auth) |
| `name` | string | Customer's full name | ✅ **YES** → `name` (indexed) |
| `phone` | string | Customer's phone number | ✅ **YES** → `phone` |
| `description` | string | Arbitrary text description | ⚠️ **OPTIONAL** → Store in metadata or separate field |

### Address Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `address` | object | Billing address | ✅ **YES** → `address` (JSONB) |
| `shipping` | object | Shipping address | ⚠️ **OPTIONAL** → Store in metadata or separate `shipping` JSONB field |

**Address Object Structure:**
```json
{
  "line1": "123 Main St",
  "line2": "Apt 4B",
  "city": "New York",
  "state": "NY",
  "postal_code": "10001",
  "country": "US"
}
```

### Payment & Billing Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `balance` | integer | Current balance (cents) | ⚠️ **OPTIONAL** → Usually not needed (can query Stripe) |
| `currency` | string | Default currency (e.g., `usd`) | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `default_source` | string | ID of default payment method | ⚠️ **OPTIONAL** → Usually not needed (can query Stripe) |
| `delinquent` | boolean | Has unpaid invoices? | ⚠️ **OPTIONAL** → Can derive from subscriptions |
| `tax_exempt` | string | `none`, `exempt`, or `reverse` | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `tax_ids` | array | Tax ID objects | ⚠️ **OPTIONAL** → Store in metadata if needed |

### Metadata & Custom Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `metadata` | object | Key-value pairs for custom data | ✅ **YES** → `metadata` (JSONB, GIN indexed) |

**This is where custom membership data goes:**
- Member number
- Membership type
- Join date
- Skill level
- Emergency contacts
- Any club-specific attributes

### Relationship Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `subscriptions` | object/list | Customer's subscriptions | ❌ **NO** → Separate `subscriptions` table (linked via `user_id`) |
| `sources` | object/list | Payment methods | ❌ **NO** → Query Stripe API when needed |
| `tax_ids` | object/list | Tax IDs | ⚠️ **OPTIONAL** → Store in metadata if needed |

### Timestamps

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `created` | integer | Unix timestamp | ✅ **YES** → `stripe_created_at` (TIMESTAMP WITH TIME ZONE) |
| `updated` | integer | Unix timestamp (if customer was updated) | ✅ **YES** → `stripe_updated_at` (TIMESTAMP WITH TIME ZONE) |

### Deleted Customers

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `deleted` | boolean | True if customer was deleted | ⚠️ **HANDLE** → Soft delete or remove from DB (webhook `customer.deleted`) |

## Recommended PostgreSQL Mapping

### Core Fields (Dedicated Columns)

**High-priority fields that should have dedicated columns:**

```sql
-- Identity
stripe_customer_id VARCHAR(255) UNIQUE NOT NULL,  -- Stripe Customer ID
email VARCHAR(255) UNIQUE NOT NULL,               -- Email (used for auth)
name VARCHAR(255),                                 -- Full name
phone VARCHAR(50),                                 -- Phone number

-- Address (stored as JSONB for flexibility)
address JSONB,  -- Billing address object from Stripe

-- Timestamps
stripe_created_at TIMESTAMP WITH TIME ZONE,       -- Stripe created timestamp
stripe_updated_at TIMESTAMP WITH TIME ZONE,       -- Stripe updated timestamp
```

### Metadata (JSONB)

**All other fields stored in metadata JSONB:**

```sql
metadata JSONB NOT NULL DEFAULT '{}'::jsonb,  -- All Stripe Customer metadata + custom fields
```

**What goes in metadata:**
- Stripe `metadata` object (custom key-value pairs)
- Stripe `description` (if used)
- Stripe `shipping` address (if different from billing)
- Stripe `balance`, `currency`, `tax_exempt`, `tax_ids` (if needed)
- Custom membership attributes:
  - `member_number`
  - `membership_type`
  - `membership_status`
  - `join_date`
  - `skill_level`
  - `emergency_contact_name`
  - `emergency_contact_phone`
  - Any club-specific custom fields

### Fields NOT Stored (Query Stripe API)

**Fields that change frequently or are better queried from Stripe:**

- `subscriptions` → Separate table (linked via `user_id`)
- `sources` (payment methods) → Query Stripe API when needed
- `default_source` → Query Stripe API when needed
- `balance` → Query Stripe API when needed (unless you need historical tracking)
- `delinquent` → Can derive from subscription status

## Example Stripe Customer Object

```json
{
  "id": "cus_ABC123XYZ",
  "object": "customer",
  "email": "john.doe@example.com",
  "name": "John Doe",
  "phone": "+15551234567",
  "description": "Tennis club member",
  "address": {
    "line1": "123 Main St",
    "line2": "Apt 4B",
    "city": "New York",
    "state": "NY",
    "postal_code": "10001",
    "country": "US"
  },
  "shipping": {
    "name": "John Doe",
    "phone": "+15551234567",
    "address": {
      "line1": "123 Main St",
      "city": "New York",
      "state": "NY",
      "postal_code": "10001",
      "country": "US"
    }
  },
  "balance": 0,
  "currency": "usd",
  "default_source": "card_1234567890",
  "delinquent": false,
  "metadata": {
    "member_number": "KGTC-2024-001",
    "membership_type": "Adult",
    "membership_status": "active",
    "join_date": "2024-01-15",
    "skill_level": "Intermediate",
    "emergency_contact_name": "Jane Doe",
    "emergency_contact_phone": "+15559876543"
  },
  "tax_exempt": "none",
  "created": 1705276800,
  "updated": 1705276800
}
```

## Recommended PostgreSQL Schema (Updated)

Based on this analysis:

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Core Identity (dedicated columns)
    email VARCHAR(255) UNIQUE NOT NULL,
    stripe_customer_id VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    phone VARCHAR(50),
    
    -- Address (JSONB for flexibility)
    address JSONB,  -- Billing address from Stripe
    
    -- Full metadata (JSONB - includes Stripe metadata + custom fields)
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    
    -- Stripe Timestamps
    stripe_created_at TIMESTAMP WITH TIME ZONE,
    stripe_updated_at TIMESTAMP WITH TIME ZONE,
    
    -- Sync Tracking
    synced_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_webhook_event_id VARCHAR(255),
    
    -- Database Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_stripe_customer_id ON users(stripe_customer_id);
CREATE INDEX idx_users_metadata_gin ON users USING GIN(metadata);
CREATE INDEX idx_users_name ON users(name);
```

## Summary

**✅ Store as Dedicated Columns:**
- `id` → `stripe_customer_id` (UNIQUE, indexed)
- `email` → `email` (UNIQUE, indexed - used for auth)
- `name` → `name` (indexed)
- `phone` → `phone`
- `address` → `address` (JSONB)
- `created` → `stripe_created_at`
- `updated` → `stripe_updated_at`

**✅ Store in Metadata JSONB:**
- Stripe `metadata` object (custom key-value pairs)
- `description` (if used)
- `shipping` address (if different from billing)
- `balance`, `currency`, `tax_exempt`, `tax_ids` (if needed)
- All custom membership attributes

**❌ Don't Store (Query Stripe API):**
- `subscriptions` → Separate table
- `sources` (payment methods)
- `default_source`
- `balance` (unless historical tracking needed)
- `delinquent` (can derive from subscriptions)
