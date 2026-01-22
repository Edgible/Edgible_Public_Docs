# Stripe Subscription Attributes Mapping

## Overview

This document summarizes Stripe Subscription object attributes and how they should be mapped to PostgreSQL in Cogento.

## Stripe Subscription Object Attributes

### Core Identity Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `id` | string | Unique subscription identifier (e.g., `sub_xxx`) | ✅ **YES** → `stripe_subscription_id` (UNIQUE, indexed) |
| `customer` | string | Customer ID (e.g., `cus_xxx`) | ✅ **YES** → `user_id` (Foreign Key to `users.user_id`) |
| `object` | string | Always `"subscription"` | ❌ **NO** → Not needed |

### Status & State Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `status` | string | Subscription status: `incomplete`, `incomplete_expired`, `trialing`, `active`, `past_due`, `canceled`, `unpaid`, `paused` | ✅ **YES** → `status` (indexed) |
| `cancel_at_period_end` | boolean | If true, subscription will cancel at period end | ✅ **YES** → `cancel_at_period_end` (BOOLEAN) |
| `canceled_at` | integer | Unix timestamp when canceled | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `paused_at` | integer | Unix timestamp when paused (if paused) | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `ended_at` | integer | Unix timestamp when ended | ⚠️ **OPTIONAL** → Store in metadata if needed |

### Product & Plan Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `items` | object/list | Subscription items (products/prices) | ⚠️ **OPTIONAL** → Store summary in metadata or separate table |
| `default_price` | string | ID of default price | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `plan` | object | Plan object (deprecated, use items) | ⚠️ **OPTIONAL** → Query from items |
| `product` | string | Product ID (if single item) | ✅ **YES** → `product_id` (indexed) |
| `price` | string | Price ID (if single item) | ⚠️ **OPTIONAL** → Store in metadata if needed |

### Billing Period Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `current_period_start` | integer | Unix timestamp | ✅ **YES** → `current_period_start` (TIMESTAMP WITH TIME ZONE) |
| `current_period_end` | integer | Unix timestamp | ✅ **YES** → `current_period_end` (TIMESTAMP WITH TIME ZONE) |
| `billing_cycle_anchor` | integer | Unix timestamp | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `days_until_due` | integer | Days until due | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `next_pending_invoice_item_invoice` | integer | Unix timestamp | ⚠️ **OPTIONAL** → Query Stripe API when needed |

### Trial Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `trial_start` | integer | Unix timestamp when trial started | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `trial_end` | integer | Unix timestamp when trial ends | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `trial_settings` | object | Trial settings | ⚠️ **OPTIONAL** → Store in metadata if needed |

### Payment & Billing Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `latest_invoice` | string | ID of most recent invoice | ⚠️ **OPTIONAL** → Query Stripe API when needed |
| `default_payment_method` | string | ID of default payment method | ⚠️ **OPTIONAL** → Query Stripe API when needed |
| `default_source` | string | ID of default payment source (deprecated) | ❌ **NO** → Deprecated |
| `collection_method` | string | `charge_automatically` or `send_invoice` | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `days_until_due` | integer | Days until invoice due | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `currency` | string | Three-letter ISO currency code | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `application_fee_percent` | number | Application fee percentage | ⚠️ **OPTIONAL** → Store in metadata if needed |

### Discount Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `discount` | object | Discount applied to subscription | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `coupon` | string | Coupon ID (deprecated, use discount) | ❌ **NO** → Deprecated |

### Metadata & Custom Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `metadata` | object | Key-value pairs for custom data | ✅ **YES** → `metadata` (JSONB, GIN indexed) |

**This is where custom subscription data goes:**
- Subscription type
- Court access
- Auto-renewal preference
- Membership package details
- Any subscription-specific attributes

### Timestamps

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `created` | integer | Unix timestamp | ⚠️ **OPTIONAL** → Store in metadata or separate `stripe_created_at` field |
| `start_date` | integer | Unix timestamp | ⚠️ **OPTIONAL** → Store in metadata if needed |
| `updated` | integer | Unix timestamp (if subscription was updated) | ⚠️ **OPTIONAL** → Store in metadata if needed |

### Deleted Subscriptions

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `deleted` | boolean | True if subscription was deleted | ⚠️ **HANDLE** → Soft delete or remove from DB (webhook `subscription.deleted`) |

### Relationship Fields

| Stripe Field | Type | Description | Map to PostgreSQL? |
|-------------|------|-------------|-------------------|
| `customer` | string | Customer ID | ✅ **YES** → `user_id` (Foreign Key) |
| `items` | object/list | Subscription items | ⚠️ **OPTIONAL** → Store summary or query Stripe API |
| `latest_invoice` | string | Invoice ID | ⚠️ **OPTIONAL** → Query Stripe API when needed |
| `schedule` | string | Subscription schedule ID | ⚠️ **OPTIONAL** → Store in metadata if needed |

## Recommended PostgreSQL Mapping

### Core Fields (Dedicated Columns)

**High-priority fields that should have dedicated columns:**

```sql
-- Identity
stripe_subscription_id VARCHAR(255) UNIQUE NOT NULL,  -- Stripe Subscription ID
user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,  -- Links to customer

-- Status
status VARCHAR(50),  -- Subscription status (indexed)
cancel_at_period_end BOOLEAN,  -- Will cancel at period end?

-- Product
product_id VARCHAR(255),  -- Product ID (indexed)

-- Billing Period
current_period_start TIMESTAMP WITH TIME ZONE,  -- Current period start
current_period_end TIMESTAMP WITH TIME ZONE,  -- Current period end
```

### Metadata (JSONB)

**All other fields stored in metadata JSONB:**

```sql
metadata JSONB NOT NULL DEFAULT '{}'::jsonb,  -- All Stripe Subscription metadata + custom fields
```

**What goes in metadata:**
- Stripe `metadata` object (custom key-value pairs)
- Stripe `items` (if multiple items or detailed item info needed)
- Stripe `canceled_at`, `paused_at`, `ended_at` (if needed)
- Stripe `trial_start`, `trial_end`, `trial_settings` (if needed)
- Stripe `collection_method`, `currency`, `days_until_due` (if needed)
- Stripe `discount`, `latest_invoice`, `schedule` (if needed)
- Custom subscription attributes:
  - `subscription_type`
  - `court_access`
  - `auto_renewal`
  - `membership_package`
  - Any subscription-specific custom fields

### Fields NOT Stored (Query Stripe API)

**Fields that change frequently or are better queried from Stripe:**

- `items` (detailed items) → Query Stripe API when needed (or store summary in metadata)
- `latest_invoice` → Query Stripe API when needed
- `default_payment_method` → Query Stripe API when needed
- `schedule` → Query Stripe API when needed

## Example Stripe Subscription Object

```json
{
  "id": "sub_ABC123XYZ",
  "object": "subscription",
  "customer": "cus_ABC123XYZ",
  "status": "active",
  "cancel_at_period_end": false,
  "canceled_at": null,
  "current_period_start": 1705276800,
  "current_period_end": 1707955200,
  "created": 1705276800,
  "start_date": 1705276800,
  "ended_at": null,
  "trial_start": null,
  "trial_end": null,
  "items": {
    "object": "list",
    "data": [
      {
        "id": "si_ABC123XYZ",
        "object": "subscription_item",
        "price": {
          "id": "price_ABC123XYZ",
          "object": "price",
          "product": "prod_ABC123XYZ",
          "nickname": "Adult Membership",
          "unit_amount": 5000,
          "currency": "usd",
          "recurring": {
            "interval": "month",
            "interval_count": 1
          }
        },
        "quantity": 1
      }
    ]
  },
  "default_price": "price_ABC123XYZ",
  "product": "prod_ABC123XYZ",
  "price": "price_ABC123XYZ",
  "collection_method": "charge_automatically",
  "currency": "usd",
  "metadata": {
    "subscription_type": "Adult Membership",
    "court_access": "all_courts",
    "auto_renewal": "true",
    "membership_package": "premium"
  },
  "latest_invoice": "in_ABC123XYZ",
  "default_payment_method": "pm_ABC123XYZ"
}
```

## Recommended PostgreSQL Schema (Updated)

Based on this analysis:

```sql
CREATE TABLE subscriptions (
    subscription_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Core Identity (dedicated columns)
    stripe_subscription_id VARCHAR(255) UNIQUE NOT NULL,
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    
    -- Status
    status VARCHAR(50),  -- Subscription status
    cancel_at_period_end BOOLEAN,  -- Will cancel at period end?
    
    -- Product
    product_id VARCHAR(255),  -- Product ID (if single item subscription)
    
    -- Billing Period
    current_period_start TIMESTAMP WITH TIME ZONE,
    current_period_end TIMESTAMP WITH TIME ZONE,
    
    -- Full metadata (JSONB - includes Stripe metadata + custom fields)
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    
    -- Sync Tracking
    synced_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Database Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_subscriptions_status ON subscriptions(status);
CREATE INDEX idx_subscriptions_user_id ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_stripe_subscription_id ON subscriptions(stripe_subscription_id);
CREATE INDEX idx_subscriptions_product_id ON subscriptions(product_id);
CREATE INDEX idx_subscriptions_metadata_gin ON subscriptions USING GIN(metadata);
```

## Status Values

Stripe subscription status can be one of:
- `incomplete` - Payment required to start
- `incomplete_expired` - Payment failed, subscription expired
- `trialing` - In trial period
- `active` - Active and current
- `past_due` - Payment failed but subscription still active
- `canceled` - Canceled (will end at period end if `cancel_at_period_end` is true)
- `unpaid` - Payment failed, subscription ended
- `paused` - Paused (newer feature)

**Recommended:** Store `status` as VARCHAR(50) to accommodate all statuses.

## Multiple Subscription Items

**Note:** Stripe subscriptions can have multiple items (multiple products/prices). 

**Options:**
1. **Store only primary product** → Use `product_id` field for first/primary item
2. **Store all items in metadata** → Store full `items` array in metadata JSONB
3. **Separate subscription_items table** → More complex, only if needed for detailed queries

**For v1, recommended:** Store primary `product_id` in dedicated column, store full `items` array in metadata if needed.

## Summary

**✅ Store as Dedicated Columns:**
- `id` → `stripe_subscription_id` (UNIQUE, indexed)
- `customer` → `user_id` (Foreign Key to users)
- `status` → `status` (indexed)
- `cancel_at_period_end` → `cancel_at_period_end` (BOOLEAN)
- `product` → `product_id` (indexed, if single item)
- `current_period_start` → `current_period_start` (TIMESTAMP)
- `current_period_end` → `current_period_end` (TIMESTAMP)

**✅ Store in Metadata JSONB:**
- Stripe `metadata` object (custom key-value pairs)
- `items` array (if multiple items or detailed info needed)
- `canceled_at`, `paused_at`, `ended_at` (if needed)
- `trial_start`, `trial_end`, `trial_settings` (if needed)
- `collection_method`, `currency`, `days_until_due` (if needed)
- `discount`, `latest_invoice`, `schedule` (if needed)
- All custom subscription attributes

**❌ Don't Store (Query Stripe API):**
- `latest_invoice` → Query Stripe API when needed
- `default_payment_method` → Query Stripe API when needed
- `items` (detailed items) → Query Stripe API when needed (or store summary in metadata)
- `schedule` → Query Stripe API when needed
