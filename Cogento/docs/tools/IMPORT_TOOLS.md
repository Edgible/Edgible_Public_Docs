# Customer Import Tools

Two tools for importing customer data from CSV into Stripe.

## Tool 1: Import Customers (`import_customers.py`)

Creates new Stripe customers from a CSV file. Maps CSV fields to Stripe customer attributes.

### Field Mapping

**Direct Stripe Customer Fields:**
- `email` → Stripe `email` (required)
- `first_name` + `last_name` → Stripe `name`
- `mobile` or `phone` → Stripe `phone`
- `street_address`, `city_address`, `postcode_address`, `state_address`, `country_address` → Stripe `address` object

**Metadata Fields (stored in Stripe `metadata`):**
- `title` → `metadata.title`
- `is_committee` → `metadata.is_committee` (boolean as string)
- `is_life_member` → `metadata.is_life_member` (boolean as string)
- `is_spoc` → `metadata.is_spoc` (boolean as string)
- `is_smmlta` → `metadata.is_smmlta` (boolean as string)
- `is_coach` → `metadata.is_coach` (boolean as string)
- `is_secretary` → `metadata.is_secretary` (boolean as string)
- `is_tranmere` → `metadata.is_tranmere` (boolean as string)
- `Gender` → `metadata.gender`
- `join_date` → `metadata.join_date` (parsed to YYYY-MM-DD format)
- `birth_date` → `metadata.birth_date` (parsed to YYYY-MM-DD format)
- `phone` (if mobile exists) → `metadata.phone_landline`

### Usage

**From Cogento root directory:**

```bash
# Interactive mode
python tools/import_customers.py

# Command line mode
python tools/import_customers.py kgtc tenants/kgtc/Customers.csv

# Dry run (test without creating)
python tools/import_customers.py kgtc tenants/kgtc/Customers.csv --dry-run
```

### Example Output

```
Reading CSV file: /app/tenants/kgtc/Customers.csv
Tenant: kgtc (KGTC)
Dry run: False
------------------------------------------------------------
Row 2: CREATED - cslabbott@barchambers.com.au (ID: cus_xxx)
Row 3: CREATED - magostino@holidaywonders.com.au (ID: cus_yyy)
Row 4: SKIPPED - No email address
------------------------------------------------------------
Summary:
  Created: 200
  Skipped: 5
  Errors: 2
  Total rows: 207
```

## Tool 2: Update Customer Metadata (`update_customer_metadata.py`)

Updates existing Stripe customers with metadata from CSV. Customers are matched by email address.

### Usage

**From Cogento root directory:**

```bash
# Interactive mode
python tools/update_customer_metadata.py

# Command line mode
python tools/update_customer_metadata.py kgtc tenants/kgtc/Customers.csv

# Dry run (test without updating)
python tools/update_customer_metadata.py kgtc tenants/kgtc/Customers.csv --dry-run
```

### Example Output

```
Reading CSV file: /app/tenants/kgtc/Customers.csv
Tenant: kgtc (KGTC)
Dry run: False
------------------------------------------------------------
Row 2: UPDATED - cslabbott@barchambers.com.au (ID: cus_xxx)
Row 3: UPDATED - magostino@holidaywonders.com.au (ID: cus_yyy)
Row 4: NOT FOUND - [email protected] (customer does not exist in Stripe)
------------------------------------------------------------
Summary:
  Updated: 195
  Not found: 5
  Skipped: 2
  Errors: 0
  Total rows: 202
```

## CSV File Format

Expected CSV columns:
- `last_name`, `first_name`, `title`
- `email` (required)
- `mobile`, `phone`
- `street_address`, `city_address`, `postcode_address`, `state_address`, `country_address`
- `join_date`, `birth_date`
- `is_committee`, `is_life_member`, `is_spoc`, `is_smmlta`, `is_coach`, `is_secretary`, `is_tranmere`
- `Gender`

## Date Format Support

The tools support multiple date formats:
- `5-Dec-2011` (DD-MMM-YYYY)
- `5/12/2011` (DD/MM/YYYY)
- `2011-12-05` (YYYY-MM-DD)
- `5-12-2011` (DD-MM-YYYY)

Dates are normalized to `YYYY-MM-DD` format in metadata.

## Boolean Field Handling

Boolean fields (`is_committee`, `is_life_member`, etc.) are parsed from:
- `TRUE` → `"true"` (stored as string in metadata)
- `FALSE` → `"false"` (stored as string in metadata)
- Empty → `"false"`

## Workflow

1. **First import**: Use `import_customers.py` to create all customers in Stripe
2. **Update metadata**: Use `update_customer_metadata.py` to add/update metadata for existing customers

Or if customers already exist:
1. Use `update_customer_metadata.py` to add metadata to existing customers

## Notes

- **Email matching**: The update tool uses Stripe's search API to find customers by email
- **Metadata merging**: When updating, existing metadata is preserved and new metadata is merged in
- **Dry run**: Always test with `--dry-run` first to see what will happen
- **Error handling**: Rows with errors are reported but don't stop the import
- **Phone cleaning**: Phone numbers are cleaned (spaces removed, keeps digits and +)

## Troubleshooting

### "Tenant not found"
- Ensure the tenant exists in the `tenants` table
- Check tenant ID spelling

### "Stripe keys not configured"
- Run `encrypt_stripe_keys.py` to store Stripe keys for the tenant

### "Customer not found" (update tool)
- Customer must exist in Stripe first
- Use `import_customers.py` to create customers first
- Check email address matches exactly

### "CSV file not found"
- Use path relative to Cogento root: `tenants/kgtc/Customers.csv`
- Or use absolute path: `/full/path/to/Customers.csv`
