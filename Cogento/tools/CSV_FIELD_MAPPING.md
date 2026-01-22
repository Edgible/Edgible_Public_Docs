# CSV Field Mapping for Bulk Import

This document clarifies which CSV fields are used by each bulk import endpoint.

## Customer Import (`/stripe/{tenant_id}/customers/import`)

This endpoint creates Stripe customers with **ONLY non-metadata fields**. Metadata is handled separately.

### CSV Fields → Stripe Customer Attributes

| CSV Column(s) | Stripe Customer Field | Notes |
|---------------|----------------------|-------|
| `email` | `email` | **Required** - Must be unique |
| `first_name` + `last_name` | `name` | Combined into single name field (e.g., "John Doe") |
| `mobile` (preferred) or `phone` | `phone` | Uses `mobile` if present, otherwise falls back to `phone`. Phone numbers are cleaned (digits and + only) |
| `street_address` | `address.line1` | Street address line |
| `city_address` | `address.city` | City name |
| `postcode_address` | `address.postal_code` | Postal/ZIP code |
| `state_address` | `address.state` | State/Province |
| `country_address` | `address.country` | Country code or name |

### CSV Fields IGNORED by Import Endpoint

The following CSV columns are **NOT** used by the import endpoint and should be handled via the separate `update-metadata` endpoint:

- `title`
- `is_committee`
- `is_life_member`
- `is_spoc`
- `is_smmlta`
- `is_coach`
- `is_secretary`
- `is_tranmere`
- `join_date`
- `birth_date`
- `Gender`
- Any other custom fields

## Metadata Update (`/stripe/{tenant_id}/customers/update-metadata`)

This endpoint updates metadata for **existing** Stripe customers (matched by email).

### CSV Fields → Stripe Customer Metadata

All CSV columns except `email` are treated as metadata fields:

| CSV Column | Metadata Key | Value Type | Notes |
|------------|--------------|------------|-------|
| `email` | N/A | N/A | Used to find customer (not stored as metadata) |
| `title` | `title` | String | |
| `is_committee` | `is_committee` | String ("true"/"false") | Boolean converted to string |
| `is_life_member` | `is_life_member` | String ("true"/"false") | Boolean converted to string |
| `is_spoc` | `is_spoc` | String ("true"/"false") | Boolean converted to string |
| `is_smmlta` | `is_smmlta` | String ("true"/"false") | Boolean converted to string |
| `is_coach` | `is_coach` | String ("true"/"false") | Boolean converted to string |
| `is_secretary` | `is_secretary` | String ("true"/"false") | Boolean converted to string |
| `is_tranmere` | `is_tranmere` | String ("true"/"false") | Boolean converted to string |
| `join_date` | `join_date` | String (ISO date) | Parsed from various date formats |
| `birth_date` | `birth_date` | String (ISO date) | Parsed from various date formats |
| `Gender` | `gender` | String | |
| `phone` (if `mobile` exists) | `phone_landline` | String | Only if `mobile` is also present |
| `street_address` | N/A | N/A | **NOT stored in metadata** - only in `Customer.address.line1` |
| `city_address` | N/A | N/A | **NOT stored in metadata** - only in `Customer.address.city` |
| `postcode_address` | N/A | N/A | **NOT stored in metadata** - only in `Customer.address.postal_code` |
| `state_address` | N/A | N/A | **NOT stored in metadata** - only in `Customer.address.state` |
| `country_address` | N/A | N/A | **NOT stored in metadata** - only in `Customer.address.country` |

**Note:** Metadata fields are merged with existing customer metadata. If a metadata key already exists, it will be updated with the new value.

## Typical Workflow

1. **Import customers** (creates basic customer records):
   ```bash
   curl -X POST "http://localhost:8000/stripe/kgtc/customers/import" \
     -F "file=@tenants/kgtc/Customers.csv"
   ```

2. **Update metadata** (adds custom attributes):
   ```bash
   curl -X POST "http://localhost:8000/stripe/kgtc/customers/update-metadata" \
     -F "file=@tenants/kgtc/Customers.csv"
   ```

## Example CSV Row

```csv
last_name,first_name,title,is_committee,is_life_member,email,mobile,phone,street_address,city_address,postcode_address,state_address,country_address,join_date,birth_date,is_spoc,is_smmlta,is_coach,is_secretary,is_tranmere,Gender
Doe,John,Mr,FALSE,FALSE,[email protected],0412345678,98765432,123 Main St,Sydney,2000,NSW,Australia,1-Jan-2020,1-Jan-1980,FALSE,FALSE,FALSE,FALSE,FALSE,M
```

**Import endpoint creates:**
- Email: `[email protected]`
- Name: "John Doe"
- Phone: "0412345678" (from `mobile`)
- Address: `{line1: "123 Main St", city: "Sydney", postal_code: "2000", state: "NSW", country: "Australia"}`

**Update-metadata endpoint adds:**
- Metadata: `{title: "Mr", is_committee: "false", is_life_member: "false", join_date: "2020-01-01", birth_date: "1980-01-01", gender: "M", ...}`
