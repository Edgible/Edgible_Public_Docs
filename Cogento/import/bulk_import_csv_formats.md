# Bulk Import CSV Formats

This document describes the CSV format requirements for all bulk import operations available in Cogento. Each import type has specific column requirements and field mappings.

## Table of Contents

1. [Stripe: Customer Bulk Create Import](#stripe-customer-bulk-create-import)
2. [Stripe: Customer Bulk Update Import](#stripe-customer-bulk-update-import)
3. [Stripe: Payment Method Bulk Import](#stripe-payment-method-bulk-import)
4. [Stripe: Subscription Bulk Create Import](#stripe-subscription-bulk-create-import)
5. [Stripe: Metadata Bulk Update](#stripe-metadata-bulk-update)
6. [Tenant: Merge Field Bulk Create Import](#tenant-merge-field-bulk-create-import)
7. [Tenant: Metadata Field Bulk Create Import](#tenant-metadata-field-bulk-create-import)
8. [Tenant: User Bulk Create Import](#tenant-user-bulk-create-import)

---

## Stripe: Customer Bulk Create Import

Creates new Stripe customers from CSV. Only core fields (email, name, phone, address) are imported. Metadata is not included in this import type.

### Required Columns

- `email` - Customer email address (must be unique)

### Optional Columns

**Core Customer Fields:**
- `first_name` + `last_name` - Combined into Stripe `Customer.name` (e.g., "John Doe")
- `mobile` (preferred) or `phone` - Customer phone number (uses `mobile` if present, otherwise falls back to `phone`)
- `address_line1` or `street_address` - Street address line 1
- `address_line2` or `suburb_address` - Street address line 2 (optional)
- `address_city` or `city_address` - City name
- `address_postcode` or `postcode_address` - Postal/ZIP code
- `address_state` or `state_address` - State/Province
- `address_country` or `country_address` - Country code or name

**Note:** Phone numbers are automatically formatted to international format. If a number starts with 0 and a country is provided, the 0 is replaced with the country code.

### Example CSV

```csv
email,first_name,last_name,mobile,address_line1,address_city,address_postcode,address_state,address_country
john.doe@example.com,John,Doe,+61448513307,123 Main St,Melbourne,3000,VIC,Australia
jane.smith@example.com,Jane,Smith,0448513307,456 Oak Ave,Sydney,2000,NSW,Australia
bob.jones@example.com,Bob,Jones,,789 Pine Rd,Brisbane,4000,QLD,Australia
```

### Field Mappings

| CSV Column | Stripe Customer Field | Notes |
|------------|---------------------|-------|
| `email` | `email` | Required, must be unique |
| `first_name` + `last_name` | `name` | Combined into single name field |
| `mobile` (preferred) or `phone` | `phone` | Auto-formatted to international format |
| `address_line1` / `street_address` | `address.line1` | |
| `address_line2` / `suburb_address` | `address.line2` | Optional |
| `address_city` / `city_address` | `address.city` | |
| `address_postcode` / `postcode_address` | `address.postal_code` | |
| `address_state` / `state_address` | `address.state` | |
| `address_country` / `country_address` | `address.country` | |

---

## Stripe: Customer Bulk Update Import

Updates existing Stripe customers from CSV. Handles both core customer fields and metadata. Customers are matched by email address.

### Required Columns

- `email` - Customer email address (used for matching existing customers)

### Optional Columns

**Core Customer Fields:**
- `first_name` + `last_name` - Combined into Stripe `Customer.name`
- `mobile` or `phone` - Customer phone number
- `address_line1`, `address_line2`, `address_city`, `address_postcode`, `address_state`, `address_country` - Address fields
- `description` - Customer description

**Metadata Fields:**
- `metadata.X` - Any column starting with `metadata.` will be added to customer metadata
  - Example: `metadata.title` → `metadata['title']`
  - Example: `metadata.company` → `metadata['company']`
  - Example: `metadata.is_vip` → `metadata['is_vip']`

### Example CSV

```csv
email,first_name,last_name,phone,address_line1,address_city,description,metadata.title,metadata.company,metadata.is_vip
john.doe@example.com,John,Doe,+61448513307,123 Main St,Melbourne,Important customer,Mr,Acme Corp,true
jane.smith@example.com,Jane,Smith,,456 Oak Ave,Sydney,Regular customer,Ms,Widget Inc,false
```

### Field Mappings

| CSV Column | Stripe Customer Field | Notes |
|------------|---------------------|-------|
| `email` | N/A | Used for matching (not stored) |
| `first_name` + `last_name` | `name` | Combined into single name field |
| `mobile` / `phone` | `phone` | |
| `address_*` fields | `address` object | All address fields combined |
| `description` | `description` | |
| `metadata.X` | `metadata[X]` | Any column starting with `metadata.` |

**Note:** If a customer with the email is not found, that row is skipped and processing continues.

---

## Stripe: Payment Method Bulk Import

Attaches or creates payment methods for Stripe customers. Supports two modes: attaching existing payment methods or creating test payment methods.

### Required Columns

- `email` or `customer_id` - Customer email or Stripe customer ID

**Mode 1: Attach Existing Payment Method**
- `payment_method_token` - Stripe Payment Method ID (pm_xxx) to attach

**Mode 2: Create Test Card** (Requires "Raw card data APIs" permission)
- `create_test_card` - Set to `true` to create a test payment method

### Optional Columns

**For Test Card Mode:**
- `exp_month` - Expiration month (1-12), defaults to current month
- `exp_year` - Expiration year (e.g., 2025), defaults to next year
- `cvc` - Card security code, defaults to 123

**For All Modes:**
- `set_as_default` - `true`/`false` to set as default payment method (default: `true`)

### Example CSV - Attach Existing Payment Methods

```csv
email,payment_method_token,set_as_default
john.doe@example.com,pm_1234567890abcdef,true
jane.smith@example.com,pm_0987654321fedcba,false
```

### Example CSV - Create Test Cards

```csv
email,create_test_card,exp_month,exp_year,set_as_default
john.doe@example.com,true,12,2025,true
jane.smith@example.com,true,6,2026,true
bob.jones@example.com,true,,,false
```

### Field Mappings

| CSV Column | Stripe Field | Notes |
|------------|-------------|-------|
| `email` or `customer_id` | Customer identifier | Used to find customer |
| `payment_method_token` | Payment Method ID | Mode 1: Attach existing (pm_xxx) |
| `create_test_card` | Flag | Mode 2: Create test card (true/false) |
| `exp_month` | Expiration month | Optional, defaults to current month |
| `exp_year` | Expiration year | Optional, defaults to next year |
| `cvc` | CVC code | Optional, defaults to 123 |
| `set_as_default` | Default flag | true/false, default: true |

**Note:** You must provide either `payment_method_token` OR `create_test_card=true`. If both are missing, the row is skipped.

---

## Stripe: Subscription Bulk Create Import

Creates new Stripe subscriptions from CSV. Customers must have a default payment method for paid subscriptions.

### Required Columns

- `email` or `customer_id` - Customer email or Stripe customer ID
- `price_id` - Stripe Price ID (price_xxx)

### Optional Columns

- `metadata.X` - Metadata fields using `metadata.X` pattern
  - Example: `metadata.plan_name` → `metadata['plan_name']`
- `trial_end` - Unix timestamp for trial end date
- `cancel_at_period_end` - `true`/`false` to cancel at period end
- `collection_method` - `'charge_automatically'` (default) or `'send_invoice'`

### Example CSV

```csv
email,price_id,metadata.plan_name,metadata.billing_cycle,trial_end,cancel_at_period_end,collection_method
john.doe@example.com,price_1234567890,Pro Plan,monthly,1735689600,false,charge_automatically
jane.smith@example.com,price_0987654321,Basic Plan,yearly,,false,charge_automatically
bob.jones@example.com,price_abcdefghij,Enterprise Plan,monthly,,true,send_invoice
```

### Field Mappings

| CSV Column | Stripe Subscription Field | Notes |
|------------|--------------------------|-------|
| `email` or `customer_id` | Customer identifier | Used to find/create customer |
| `price_id` | `items[0].price` | Required Stripe Price ID |
| `metadata.X` | `metadata[X]` | Any column starting with `metadata.` |
| `trial_end` | `trial_end` | Unix timestamp |
| `cancel_at_period_end` | `cancel_at_period_end` | true/false |
| `collection_method` | `collection_method` | charge_automatically or send_invoice |

**Note:** If a customer is not found by email, the row is skipped. For paid subscriptions, customers must have a default payment method already attached.

---

## Stripe: Metadata Bulk Update

Updates metadata on any Stripe object (customers, subscriptions, products, prices) from CSV. This is a flexible tool that works with any Stripe object type.

### Required Columns

- `stripe_source` - Type of Stripe object: `'customer'`, `'subscription'`, `'product'`, `'price'`
- `stripe_id` - The Stripe ID of the object (e.g., `cus_xxx`, `sub_xxx`, `prod_xxx`, `price_xxx`)

### Optional Columns

- `metadata.X` - Any column starting with `metadata.` will be added to the object's metadata
  - Example: `metadata.title` → `metadata['title']`
  - Example: `metadata.first_name` → `metadata['first_name']`
  - Example: `metadata.is_committee` → `metadata['is_committee']`

**Special Column:**
- `metadata` - If set to `'{}'`, all metadata for that record will be reset to empty (ignores `metadata.X` columns)

### Example CSV - Update Customer Metadata

```csv
stripe_source,stripe_id,metadata.title,metadata.first_name,metadata.last_name,metadata.company
customer,cus_TnwfHeFt9km01j,Mr,Stefano,Picozzi,Acme Corp
customer,cus_AbCdEfGhIjKlMn,Ms,Jane,Smith,Widget Inc
```

### Example CSV - Update Subscription Metadata

```csv
stripe_source,stripe_id,metadata.plan_name,metadata.billing_cycle,metadata.discount_code
subscription,sub_1234567890,Pro Plan,monthly,SUMMER2024
subscription,sub_0987654321,Basic Plan,yearly,
```

### Example CSV - Reset Metadata

```csv
stripe_source,stripe_id,metadata
customer,cus_TnwfHeFt9km01j,{}
subscription,sub_1234567890,{}
```

### Field Mappings

| CSV Column | Stripe Field | Notes |
|------------|-------------|-------|
| `stripe_source` | Object type | customer, subscription, product, price |
| `stripe_id` | Object ID | cus_xxx, sub_xxx, prod_xxx, price_xxx |
| `metadata.X` | `metadata[X]` | Any column starting with `metadata.` |
| `metadata` | Reset flag | Set to `{}` to reset all metadata |

**Note:** 
- If `metadata` column is `{}`, all metadata is reset to empty (ignores `metadata.X` columns)
- Otherwise, new metadata is merged with existing metadata (existing keys are updated, new keys are added)
- If the Stripe object is not found, the row is skipped

---

## Tenant: Merge Field Bulk Create Import

Creates new merge fields from CSV. Merge fields are used for email templates and dynamic content.

### Required Columns

- `label` - Unique identifier (alphanumeric + underscore only, e.g., `company_name`, `job_title`)
- `display_name` - Human-readable name (e.g., "Company Name", "Job Title")
- `source_type` - One of: `'customer'`, `'metadata'`, `'address'`, `'system'`
- `source_path` - Path to the data source (e.g., `name`, `metadata.company`, `address.city`)

### Optional Columns

- `description` - Description of the merge field

### Example CSV

```csv
label,display_name,source_type,source_path,description
company_name,Company Name,metadata,metadata.company,The company name from customer metadata
full_name,Full Name,customer,name,The customer's full name
city,City,address,address.city,The city from customer address
member_since,Member Since,system,created,When the customer was created
```

### Field Mappings

| CSV Column | Merge Field Property | Notes |
|------------|---------------------|-------|
| `label` | `label` | Alphanumeric + underscore only, must be unique |
| `display_name` | `display_name` | Human-readable name |
| `source_type` | `source_type` | customer, metadata, address, system |
| `source_path` | `source_path` | Path to data source |
| `description` | `description` | Optional description |

**Note:** Merge fields with duplicate labels will be skipped with a warning.

---

## Tenant: Metadata Field Bulk Create Import

Creates new metadata field definitions from CSV. These define the schema for custom metadata fields that can be used on customers, subscriptions, or payment methods.

### Required Columns

- `entity_type` - One of: `'customer'`, `'subscription'`, `'payment_method'`
- `key` - Unique identifier for this entity type (e.g., `'company_name'`, `'job_title'`)
- `label` - Human-readable name (e.g., "Company Name", "Job Title")
- `type` - Field type: `'string'`, `'text'`, `'number'`, `'boolean'`, `'date'`, `'email'`, `'url'`, `'select'`

### Optional Columns

- `required` - `true`/`false` (default: `false`)
- `order` - Integer order for display (default: `0`)
- `group` - Group name for organizing fields
- `placeholder` - Placeholder text
- `help_text` - Help text description
- `editable_by` - Role required to edit: `'customer'`, `'admin'`, `'superadmin'` (default: all can edit)
- `maxLength` - Maximum length for string/text fields
- `minLength` - Minimum length for string/text fields
- `pattern` - Regex pattern for validation
- `options` - JSON array for select fields: `[{"value": "opt1", "label": "Option 1"}]`
- `default` - Default value (will be converted to JSONB)

### Example CSV - Basic Fields

```csv
entity_type,key,label,type,required,order,placeholder,help_text
customer,company_name,Company Name,string,false,0,Enter company name,The company name
customer,job_title,Job Title,string,false,1,Enter job title,Person's job title
subscription,fiscal_year,Fiscal Year,string,false,0,Enter fiscal year,The fiscal year
```

### Example CSV - Advanced Fields with Validation

```csv
entity_type,key,label,type,required,order,group,placeholder,help_text,editable_by,maxLength,minLength,pattern
customer,company_name,Company Name,string,true,0,Company Info,Enter company name,The company name,admin,100,2,
customer,email_work,Work Email,email,false,1,Contact Info,Enter work email,Work email address,admin,,,^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$
customer,phone_mobile,Mobile Phone,string,false,2,Contact Info,Enter mobile,10-digit mobile number,admin,10,10,^[0-9]{10}$
```

### Example CSV - Select Field

```csv
entity_type,key,label,type,options
customer,membership_tier,Membership Tier,select,"[{""value"": ""basic"", ""label"": ""Basic""}, {""value"": ""premium"", ""label"": ""Premium""}, {""value"": ""enterprise"", ""label"": ""Enterprise""}]"
```

### Field Mappings

| CSV Column | Metadata Field Property | Notes |
|------------|------------------------|-------|
| `entity_type` | `entity_type` | customer, subscription, payment_method |
| `key` | `key` | Unique per entity_type |
| `label` | `label` | Human-readable name |
| `type` | `type` | string, text, number, boolean, date, email, url, select |
| `required` | `required` | true/false |
| `order` | `order` | Display order (integer) |
| `group` | `group` | Group name |
| `placeholder` | `placeholder` | Placeholder text |
| `help_text` | `help_text` | Help text |
| `editable_by` | `editable_by` | customer, admin, superadmin |
| `maxLength` | `maxLength` | Max length for string/text |
| `minLength` | `minLength` | Min length for string/text |
| `pattern` | `pattern` | Regex validation pattern |
| `options` | `options` | JSON array for select fields |
| `default` | `default` | Default value (converted to JSONB) |

**Note:** Duplicate `entity_type` + `key` combinations will be skipped with a warning.

---

## Tenant: User Bulk Create Import

Creates new user records in Cogento from CSV. Users can be linked to Stripe customers.

### Required Columns

- `email` - User email address (must be unique)

### Optional Columns

- `role` - User role: `'superadmin'`, `'admin'`, `'user'`, `'member'`, `'customer'` (default: `'user'`)
- `mobile` - Mobile phone number
- `metadata` - JSON string with metadata (e.g., `'{"key": "value"}'`)
- `stripe_customer_id` - Stripe customer ID (if not provided, will attempt to look up by email)

### Example CSV - Basic Users

```csv
email,role,mobile
john.doe@example.com,user,+61448513307
jane.smith@example.com,admin,
bob.jones@example.com,member,0448513307
```

### Example CSV - Users with Metadata and Stripe Link

```csv
email,role,mobile,metadata,stripe_customer_id
john.doe@example.com,user,+61448513307,"{""department"": ""Sales"", ""employee_id"": ""12345""}",cus_TnwfHeFt9km01j
jane.smith@example.com,admin,,"{""department"": ""IT""}",
bob.jones@example.com,customer,0448513307,,cus_AbCdEfGhIjKlMn
```

### Field Mappings

| CSV Column | User Field | Notes |
|------------|-----------|-------|
| `email` | `email` | Required, must be unique |
| `role` | `role` | superadmin, admin, user, member, customer (default: user) |
| `mobile` | `mobile` | Mobile phone number |
| `metadata` | `metadata` | JSON string (e.g., `'{"key": "value"}'`) |
| `stripe_customer_id` | `stripe_customer_id` | Stripe customer ID (cus_xxx) |

**Note:** 
- If `stripe_customer_id` is not provided, the tool will:
  1. Search Stripe for a customer with the matching email
  2. If found, use that customer's ID
  3. If not found, create the user without `stripe_customer_id`
- Users with duplicate emails will be skipped

---

## General Notes

### CSV Format Requirements

- **Encoding:** UTF-8
- **Header Row:** First row must contain column names
- **Delimiter:** Comma (`,`)
- **Quoting:** Use double quotes (`"`) for fields containing commas or special characters
- **Boolean Values:** Use `true`/`false` (case-insensitive) or `TRUE`/`FALSE`

### Row Processing Options

All bulk import operations support these optional parameters:

- **max_rows** - Maximum number of rows to process
- **start_row** - Start row number (data rows only, excludes header)
- **end_row** - End row number (data rows only, excludes header)
- **dry_run** - If `true`, validate but don't actually import/update

### Error Handling

- Rows with missing required fields are skipped
- Rows that fail validation are skipped with error details
- Processing continues even if individual rows fail
- Import results include counts of successful, skipped, and error rows

### Best Practices

1. **Test with dry_run:** Always test imports with `dry_run=true` first
2. **Start small:** Use `max_rows` to test with a small subset first
3. **Validate data:** Ensure required fields are present and properly formatted
4. **Check results:** Review the import results summary for any skipped or error rows
5. **Backup data:** Consider backing up data before bulk updates
