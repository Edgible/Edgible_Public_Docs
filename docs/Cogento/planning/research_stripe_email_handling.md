# Research: Stripe Email Handling & Search Efficiency

## Key Findings

### 1. Email Uniqueness in Stripe

**Critical Finding:** Stripe does **NOT** enforce email uniqueness across customer records.

- Multiple Stripe Customer records can share the same email address
- Stripe's primary key is the `customer_id` (e.g., `cus_1234567890`), not email
- Email is just a field on the Customer object, not a unique constraint

**Implication for Cogento:**
- If email uniqueness is required, it must be enforced at the **application level** (in Cogento's API service)
- Cannot rely on Stripe to prevent duplicate emails
- Need validation logic in Cogento before creating/updating Stripe Customer records

### 2. Searching Customers by Email in Stripe

**Stripe Search API:**
- Stripe provides a Search API endpoint: `/v1/customers/search`
- Can search using query: `email:'[email protected]'`
- Returns a list of customers (potentially multiple) matching the email

**Limitations & Performance:**
- ⚠️ **Rate limits apply** - Not suitable for high-frequency searches
- ⚠️ **Data propagation delays** - Typically 1 minute, up to 1 hour during outages
- ⚠️ **Network latency** - Each search requires API call to Stripe servers
- ⚠️ **Not real-time** - Search results may not reflect latest data immediately
- ❌ **Not suitable for frequent/real-time searches** - Recommended to maintain local index

**Recommended Approach:**
- Maintain a local copy/index of customer data in PostgreSQL
- Use PostgreSQL for fast email searches (indexed queries)
- Use Stripe API for writes and authoritative data reads
- This aligns perfectly with Cogento's architecture (PostgreSQL as read replica)

### 3. Best Practices

**Primary Keys:**
- Stripe uses `customer_id` as primary key (e.g., `cus_1234567890`)
- Store `customer_id` in PostgreSQL alongside email
- Use `customer_id` for direct Stripe API calls (most efficient)
- Use email for user-friendly lookups (in PostgreSQL, not Stripe)

**Email as Identifier:**
- Email is NOT a unique identifier in Stripe
- For Cogento's use case (membership system), email should be unique per member
- Must enforce email uniqueness in Cogento's application logic
- PostgreSQL can enforce email uniqueness via unique constraint

## Recommendations for Cogento

### Email Uniqueness Strategy

**Application-Level Enforcement:**
1. Before creating Stripe Customer, check if email already exists (via PostgreSQL search)
2. If email exists, prevent duplicate creation
3. Enforce email uniqueness in PostgreSQL reporting database (unique constraint on email)
4. Consider making email the primary key or unique identifier in PostgreSQL

**Stripe-Level:**
- Cannot rely on Stripe to enforce email uniqueness
- Multiple Stripe customers can theoretically have same email (though Cogento should prevent this)

### Search Strategy

**For Authentication/Login:**
- ✅ Use PostgreSQL to search by email (fast, indexed)
- ✅ Find user by email in PostgreSQL reporting database
- ✅ Use `stripe_customer_id` from PostgreSQL to verify/retrieve from Stripe if needed

**For Member Directory/Search:**
- ✅ Use PostgreSQL for all searches (email, name, filters, etc.)
- ✅ PostgreSQL is synced from Stripe via webhooks (near real-time)
- ✅ Much faster than Stripe Search API

**For Creating/Updating Members:**
- ✅ Check email uniqueness in PostgreSQL first
- ✅ If unique, create/update in Stripe
- ✅ Webhook will sync to PostgreSQL

### PostgreSQL Schema Considerations

**Option 1: Email as Primary Key (Recommended for Cogento)**
```sql
CREATE TABLE members (
    email VARCHAR(255) PRIMARY KEY,  -- Email as primary key
    stripe_customer_id VARCHAR(255) UNIQUE NOT NULL,
    -- ... other fields
);
```

**Option 2: UUID as Primary Key, Email as Unique Constraint**
```sql
CREATE TABLE members (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,  -- Unique constraint on email
    stripe_customer_id VARCHAR(255) UNIQUE NOT NULL,
    -- ... other fields
);
```

**Recommendation:** Option 2 is more flexible (allows future changes to email handling), but Option 1 aligns with Cogento's email-centric approach.

## Summary

1. **Email Uniqueness:** Must be enforced by Cogento (application level), not Stripe
2. **Email Search:** Use PostgreSQL, not Stripe Search API (much faster)
3. **Email as Identifier:** Good choice for Cogento (user-friendly, login, navigation)
4. **Architecture Alignment:** PostgreSQL read replica perfectly suited for email searches
5. **Enforcement:** Add unique constraint on email in PostgreSQL + validation logic in API

## Questions for Consideration

1. Should email be the primary key in PostgreSQL, or use UUID/ID with unique constraint?
2. How to handle email changes (if allowed)? Update in Stripe, sync via webhook?
3. What happens if someone tries to create a member with an existing email? Error message?
