# Stripe Customer Metadata & Custom Fields - Tennis Club Membership

## Overview
Stripe Customer records are **very flexible** for storing custom member attributes. You have multiple options beyond just metadata.

## 1. Metadata (Recommended for Most Custom Fields)

**Limits:**
- ✅ **Up to 50 key-value pairs** per Customer
- ✅ **Key length:** Up to 40 characters
- ✅ **Value length:** Up to 500 characters per value
- ✅ **Data type:** Strings only (but you can store JSON strings)

**Example Usage:**
```python
stripe.Customer.modify(
    customer_id,
    metadata={
        'member_number': 'TC-2024-123',
        'join_date': '2024-01-15',
        'membership_type': 'Adult',
        'emergency_contact': 'John Doe - 555-1234',
        'notes': 'Preferred court: Court 3',
        'member_status': 'active',
        # ... up to 50 total keys
    }
)
```

**Your Current Usage:**
Your `Edgible_Stripe_API` already uses metadata to store:
- `user_id` - Links Customer to your PostgreSQL user record
- `user_email` - Stores email in metadata

## 2. Built-in Customer Fields (No Limits)

Stripe Customer has these **unlimited** built-in fields:

- **`name`** - Full name (unlimited text)
- **`email`** - Email address
- **`phone`** - Phone number
- **`address`** - Full address object (line1, line2, city, state, postal_code, country)
- **`description`** - **Free-form text field (unlimited length!)** - Perfect for longer notes

**Example:**
```python
stripe.Customer.create(
    name="John Smith",
    email="john@example.com",
    phone="555-1234",
    address={
        "line1": "123 Main St",
        "city": "Melbourne",
        "state": "VIC",
        "postal_code": "3000",
        "country": "AU"
    },
    description="Adult member since 2020. Preferred court booking: Court 3. Notes: Needs wheelchair access.",  # Unlimited length!
    metadata={
        'member_number': 'TC-2024-123',
        'join_date': '2024-01-15',
        # ... other structured data
    }
)
```

## 3. Workarounds for Long Text (>500 chars)

If you need to store text longer than 500 characters:

**Option A: Use `description` field**
- Unlimited length
- Good for member notes, special instructions, etc.

**Option B: Store JSON in metadata value**
- Encode complex data as JSON string in a single metadata field
- Example: `metadata={'member_details': json.dumps({'bio': '...long text...', 'history': [...]})}`

**Option C: Split across multiple metadata fields**
- Example: `notes_1`, `notes_2`, `notes_3` if you need structured notes

## 4. Complete Field Strategy for Tennis Club Members

**Recommended Approach:**

```python
# Built-in fields (unlimited)
name="John Smith"
email="john@example.com"
phone="+61 412 345 678"
address={...full address...}
description="Additional notes, special requirements, long-form member information"  # Unlimited!

# Metadata (structured data - 50 fields max, 500 chars each)
metadata={
    # Membership details
    'member_number': 'TC-2024-123',
    'join_date': '2024-01-15',
    'membership_type': 'Adult',
    'membership_status': 'active',
    'membership_expires': '2024-12-31',
    
    # Contact info
    'emergency_contact_name': 'Jane Smith',
    'emergency_contact_phone': '555-5678',
    'emergency_contact_relation': 'Spouse',
    
    # Tennis club specific
    'preferred_court': 'Court 3',
    'skill_level': 'Intermediate',
    'doubles_partner_pref': 'Mixed',
    'playing_days': 'Weekends',
    
    # Administrative
    'membership_fee_paid': '2024-01-15',
    'last_renewal': '2024-01-15',
    'renewal_reminder_sent': '2024-11-01',
    'committee_member': 'false',
    'volunteer_hours': '10',
    
    # Custom fields (add more as needed - up to 50 total)
    'medical_conditions': 'None',
    'waiver_signed': '2024-01-15',
    'photo_consent': 'true',
    'communication_pref': 'email',
    
    # Link to your system (you already do this!)
    'user_id': 'uuid-here',  # If you need PostgreSQL link
}
```

## 5. Important Limitations & Considerations

**Metadata Limitations:**
- ⚠️ **50 key-value pairs maximum** - Plan your schema carefully
- ⚠️ **500 characters per value** - Use `description` field for longer text
- ⚠️ **40 characters per key** - Use descriptive but concise keys
- ⚠️ **Strings only** - No numbers, dates, booleans (store as strings, parse in your code)
- ⚠️ **Not searchable** - You can't search metadata directly via Stripe API filters
  - Solution: Read all customers and filter in your application code
  - Or cache in PostgreSQL for faster searching

**Best Practices:**
- ✅ Use metadata for **structured, searchable data** (dates, statuses, IDs)
- ✅ Use `description` for **unstructured, long-form text** (notes, comments)
- ✅ Use built-in fields (`name`, `email`, `phone`, `address`) for standard contact info
- ✅ Keep keys consistent across all customers (easier to query/filter)
- ✅ Store dates in ISO format (`YYYY-MM-DD`) for easy sorting

**Not Recommended:**
- ❌ Don't store sensitive PII beyond what's needed (use description field cautiously)
- ❌ Don't store payment info (Stripe handles this separately)
- ❌ Don't use metadata for very long text (>500 chars) - use `description` instead

## 6. Search & Filter Capabilities

**What Stripe API Can Search Directly:**
- ✅ `email` - `stripe.Customer.list(email='john@example.com')`
- ✅ `name` - Limited (partial match not supported well)
- ✅ Created date ranges
- ❌ **Metadata fields** - Cannot filter by metadata directly

**Workaround for Metadata Searches:**
```python
# Get all customers
all_customers = stripe.Customer.list(limit=100)

# Filter in your code
active_members = [
    c for c in all_customers 
    if c.metadata.get('membership_status') == 'active'
]

# Or cache in PostgreSQL for faster searches
```

## 7. Summary

**YES, Stripe Customer records are flexible enough** for tennis club member attributes:

✅ **Built-in fields:** name, email, phone, address (unlimited)  
✅ **Description field:** Free-form text (unlimited)  
✅ **Metadata:** Up to 50 custom key-value pairs (500 chars each)  
✅ **Total capacity:** More than enough for 200+ members with comprehensive data

**Recommendation:**
- Use built-in fields for standard contact info
- Use metadata for structured member attributes (member_number, join_date, status, etc.)
- Use `description` field for longer notes/special requirements
- With 50 metadata fields × 500 chars = 25,000 characters of structured data per member
- Plus unlimited description field
- **This is MORE than sufficient for tennis club member management!**
