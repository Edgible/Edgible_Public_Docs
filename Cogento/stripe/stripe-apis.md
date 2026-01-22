# Stripe API Usage in Cogento

This document describes how Cogento uses Stripe APIs across different endpoints and provides important information about Stripe API costs and how to manage them.

## Table of Contents

1. [Stripe API Cost Overview](#stripe-api-cost-overview)
2. [Free Stripe API Calls](#free-stripe-api-calls)
3. [Paid Stripe API Services](#paid-stripe-api-services)
4. [Cogento Stripe API Usage by Endpoint](#cogento-stripe-api-usage-by-endpoint)
5. [Cost Management Strategies](#cost-management-strategies)

---

## Stripe API Cost Overview

### Free API Calls

**All standard Stripe API calls are FREE** - there are no per-call charges for:
- Customer management (create, retrieve, update, delete, list, search)
- Product management (create, retrieve, update, delete, list)
- Price management (create, retrieve, update, delete, list)
- Subscription management (create, retrieve, update, cancel, list)
- Payment Method management (create, attach, detach, list)
- Checkout Session creation
- Webhook event processing
- Metadata operations
- Most other standard Stripe API operations

**Stripe only charges transaction fees** (typically 2.9% + $0.30 per successful card charge) - these are separate from API call costs.

### Paid Stripe API Services

Stripe charges for the following specialized services:

#### 1. Stripe Tax API

Stripe Tax provides tax calculation and transaction recording:

- **Calculation API**: $0.05 per calculation call
- **Transaction API**: $0.50 per transaction call
- **10-to-1 Rebate**: For every 1 Transaction API call, Stripe includes 10 Calculation API calls for free
- **No-Code Integration**: If using Stripe Checkout, Billing, or Invoicing with Tax enabled, the cost is **0.5% of transaction volume** in registered jurisdictions

**Note**: Cogento does NOT currently use Stripe Tax API, so there are no tax-related API costs.

#### 2. Stripe Financials API

Stripe Financials includes:
- **Financial Connections**: Pricing per successful bank account link or data refresh
- **Stripe Sigma**: Custom SQL reporting - monthly infrastructure fee plus per-transaction charges
- **Revenue Recognition**: Automates accounting for complex revenue - typically 0.25% of processed volume

**Note**: Cogento does NOT currently use Stripe Financials API, so there are no financials-related API costs.

---

## Cogento Stripe API Usage by Endpoint

### Customer Management

#### `POST /stripe/{tenant_id}/customers`
- **Stripe API**: `stripe.Customer.create()`
- **Cost**: FREE
- **Purpose**: Create a new Stripe customer
- **Usage**: Single customer creation

#### `GET /stripe/{tenant_id}/customers`
- **Stripe API**: `stripe.Customer.list()`
- **Cost**: FREE
- **Purpose**: List all customers with pagination
- **Usage**: Fetches customers in batches of 100

#### `GET /stripe/{tenant_id}/customers/{customer_id}`
- **Stripe API**: `stripe.Customer.retrieve()`
- **Cost**: FREE
- **Purpose**: Get a specific customer by ID
- **Usage**: Single customer retrieval

#### `PUT/PATCH /stripe/{tenant_id}/customers/{customer_id}`
- **Stripe API**: `stripe.Customer.modify()`
- **Cost**: FREE
- **Purpose**: Update customer information
- **Usage**: Single customer update

#### `DELETE /stripe/{tenant_id}/customers/{customer_id}`
- **Stripe API**: `stripe.Customer.delete()`
- **Cost**: FREE
- **Purpose**: Delete a customer
- **Usage**: Single customer deletion

#### `GET /stripe/{tenant_id}/customers/search`
- **Stripe API**: `stripe.Customer.search()`
- **Cost**: FREE
- **Purpose**: Search customers using Stripe's search query syntax
- **Usage**: Search with query string (e.g., `email:'[email protected]'`)

#### `GET /{tenant_id}/stripe-customers`
- **Stripe API**: `stripe.Customer.list()`
- **Cost**: FREE
- **Purpose**: Admin page for listing customers with filters
- **Usage**: Fetches all customers, then applies filters client-side

#### `POST /postgres/{tenant_id}/customers/import`
- **Stripe API**: `stripe.Customer.create()`, `stripe.Customer.search()`, `stripe.Customer.list()`
- **Cost**: FREE
- **Purpose**: Bulk import customers from CSV
- **Usage**: 
  - Searches for existing customers by email
  - Creates new customers if not found
  - May make multiple API calls per CSV row

#### `POST /postgres/{tenant_id}/customers/update`
- **Stripe API**: `stripe.Customer.modify()`, `stripe.Customer.search()`
- **Cost**: FREE
- **Purpose**: Bulk update customers from CSV
- **Usage**: 
  - Searches for customers by email
  - Updates customer fields and metadata

### Product Management

#### `GET /{tenant_id}/stripe-products`
- **Stripe API**: `stripe.Product.list()`
- **Cost**: FREE
- **Purpose**: List all products with pagination and filters
- **Usage**: Fetches all products in batches of 100

#### `GET /{tenant_id}/stripe-products/{product_id}`
- **Stripe API**: `stripe.Product.retrieve()`, `stripe.Price.retrieve()` (if default_price is string)
- **Cost**: FREE
- **Purpose**: Get a specific product by ID
- **Usage**: 
  - Retrieves product
  - If default_price is a string ID, retrieves price details

### Price Management

#### `GET /{tenant_id}/stripe-prices`
- **Stripe API**: `stripe.Price.list()`, `stripe.Product.retrieve()` (for product names)
- **Cost**: FREE
- **Purpose**: List all prices with pagination and filters
- **Usage**: 
  - Fetches all prices in batches of 100
  - For each price, may retrieve product to get product name

#### `GET /{tenant_id}/stripe-prices/{price_id}`
- **Stripe API**: `stripe.Price.retrieve()`, `stripe.Product.retrieve()` (if product is string ID)
- **Cost**: FREE
- **Purpose**: Get a specific price by ID
- **Usage**: 
  - Retrieves price
  - If product is a string ID, retrieves product to get product name

### Subscription Management

#### `GET /{tenant_id}/stripe-subscriptions`
- **Stripe API**: `stripe.Subscription.list()`, `stripe.Customer.retrieve()`, `stripe.Price.retrieve()`, `stripe.Product.retrieve()`, `stripe.Subscription.retrieve()`
- **Cost**: FREE
- **Purpose**: List subscriptions with pagination and filters
- **Usage**: 
  - Fetches subscriptions by status (may fetch multiple statuses if 'all' is selected)
  - Filters by customer_id at API level when available
  - Retrieves customer email for each subscription
  - Retrieves price/product details when items are empty (fallback)
  - May make multiple API calls per subscription to get full details

#### `POST /postgres/{tenant_id}/subscriptions/checkout`
- **Stripe API**: `stripe.Customer.search()`, `stripe.Price.retrieve()`, `stripe.checkout.Session.create()`
- **Cost**: 
  - API calls: FREE
  - **If `automatic_tax: {'enabled': True}` is set**: 0.5% of transaction volume (for GST/tax compliance)
- **Purpose**: Create a Stripe Checkout session for subscription creation
- **Usage**: 
  - Searches for customer by email
  - Retrieves price to verify it exists and get amount
  - Creates checkout session (for both free and paid subscriptions)
  - **Note**: For GST compliance in Australia, should enable `automatic_tax: {'enabled': True}` in checkout session

#### `POST /postgres/{tenant_id}/subscriptions/create` (Legacy - may not be used)
- **Stripe API**: `stripe.Customer.search()`, `stripe.Price.retrieve()`, `stripe.Subscription.create()`
- **Cost**: FREE
- **Purpose**: Create subscription directly (bypasses checkout)
- **Usage**: 
  - Searches for customer by email
  - Retrieves price details
  - Creates subscription directly

#### `POST /postgres/{tenant_id}/subscriptions/import`
- **Stripe API**: `stripe.Customer.search()`, `stripe.Subscription.create()`
- **Cost**: FREE
- **Purpose**: Bulk import subscriptions from CSV
- **Usage**: 
  - Searches for customers by email
  - Creates subscriptions for each CSV row

### Payment Method Management

#### `POST /postgres/{tenant_id}/payment-methods/import`
- **Stripe API**: `stripe.Customer.search()`, `stripe.PaymentMethod.create()`, `stripe.PaymentMethod.retrieve()`, `stripe.PaymentMethod.attach()`, `stripe.Customer.modify()`
- **Cost**: FREE
- **Purpose**: Bulk import payment methods from CSV
- **Usage**: 
  - Searches for customers by email
  - Creates or retrieves payment methods
  - Attaches payment methods to customers
  - Sets as default payment method

### Metadata Management

#### `POST /postgres/{tenant_id}/stripe-metadata/update`
- **Stripe API**: `stripe.Customer.retrieve()`, `stripe.Customer.modify()`, `stripe.Subscription.retrieve()`, `stripe.Subscription.modify()`, `stripe.Product.retrieve()`, `stripe.Product.modify()`, `stripe.Price.retrieve()`, `stripe.Price.modify()`
- **Cost**: FREE
- **Purpose**: Bulk update metadata on any Stripe object (customer, subscription, product, price)
- **Usage**: 
  - Retrieves object to get current metadata
  - Modifies object with updated metadata
  - Supports metadata reset (set to `{}`)

### User Management (with Stripe Integration)

#### `POST /postgres/{tenant_id}/users/import`
- **Stripe API**: `stripe.Customer.search()`, `stripe.Customer.list()`
- **Cost**: FREE
- **Purpose**: Bulk import users, optionally looking up Stripe customer ID by email
- **Usage**: 
  - Searches for existing Stripe customer by email if `stripe_customer_id` not provided
  - Uses list API as fallback if search doesn't find customer

### Email Campaigns

#### Email Campaign Service
- **Stripe API**: `stripe.Customer.retrieve()`
- **Cost**: FREE
- **Purpose**: Retrieve customer metadata for email merge fields
- **Usage**: 
  - Retrieves customer for each email recipient
  - Extracts metadata for template substitution

### Authentication

#### `POST /{tenant_id}/auth/register`
- **Stripe API**: `stripe.Customer.list()`, `stripe.Customer.create()`
- **Cost**: FREE
- **Purpose**: Create Stripe customer during user registration
- **Usage**: 
  - Checks if customer exists by email
  - Creates customer if not found

#### `POST /{tenant_id}/auth/login`
- **Stripe API**: `stripe.Customer.list()`
- **Cost**: FREE
- **Purpose**: Look up customer during login
- **Usage**: 
  - Searches for customer by email

---

## Cost Management Strategies

### 1. Stripe Tax API - When Required

**Current Status**: ⚠️ Cogento does NOT currently use Stripe Tax API, but **may be required for GST compliance in Australia**

**GST Requirements in Australia**:
- GST (Goods and Services Tax) is **mandatory** at 10% for most goods and services
- Businesses must collect and remit GST to the Australian Taxation Office (ATO)
- Tax calculation must be accurate and compliant with ATO regulations

**Options for GST/Tax Calculation**:

#### Option A: Stripe Tax API (Recommended for Compliance)

**Setup Requirements**:
- ⚠️ **Dashboard Configuration Required First**: Before using `automatic_tax` in API calls, you must configure Stripe Tax in the Stripe Dashboard:
  1. Go to **Stripe Dashboard > Settings > Tax**
  2. Set **Origin Address** (your business location)
  3. Add **Tax Registrations** (e.g., Australia for GST)
  4. Set **Default Product Tax Code** (e.g., `txcd_99999999` for general goods)
  5. Set **Default Tax Behavior** (Exclusive or Inclusive)
- ✅ Once configured in Dashboard, enable in code: `automatic_tax={'enabled': True}`

**Programmatic Verification**:
- ✅ Cogento automatically checks if Stripe Tax is configured before enabling `automatic_tax`
- Uses `stripe.tax.Settings.retrieve()` to check if `status == 'active'`
- If not configured (`status == 'pending'`), automatic tax is disabled and a warning is logged
- This prevents checkout failures when Stripe Tax is not yet set up

**Cost**: 
- **Test Mode**: ✅ **FREE** (up to 1,000 API calls per day) - perfect for development!
- **Production Mode**:
  - **Stripe Checkout with Tax**: 0.5% of transaction volume (recommended, simplest)
  - **Tax Calculation API**: $0.05 per calculation call
  - **Tax Transaction API**: $0.50 per transaction call

**Benefits**:
- ✅ Fully automated and compliant
- ✅ Handles all tax jurisdictions and rules
- ✅ Automatically updates when tax rules change
- ✅ Handles complex scenarios (exemptions, reduced rates, etc.)
- ✅ Provides tax reporting and documentation
- ✅ Free in test mode for development

**Implementation**: 
- Enable in Stripe Checkout: Add `automatic_tax={'enabled': True}` to checkout session
- Or use Tax Calculation API before checkout to show tax amount

**When to Use**: 
- **Recommended** for production in Australia (GST compliance)
- Required if selling to multiple jurisdictions with different tax rates
- Best for businesses that want automated, compliant tax handling

#### Option B: Manual Tax Calculation (Free but Complex)
- **Cost**: FREE (no Stripe API charges)
- **Benefits**:
  - ✅ No per-call charges
  - ✅ Full control over calculation logic
- **Drawbacks**:
  - ❌ Must maintain tax rules and rates yourself
  - ❌ Must update when tax rules change
  - ❌ More complex to implement correctly
  - ❌ Higher risk of compliance errors
  - ❌ Must handle edge cases (exemptions, reduced rates, etc.)
- **Implementation**:
  - Calculate 10% GST on all products (or apply appropriate rates)
  - Add tax amount to line items in checkout
  - Maintain tax-exempt product list if needed
- **When to Use**: 
  - Only if you have a simple, single-jurisdiction tax scenario
  - If you have development resources to maintain tax logic
  - Not recommended for production without careful compliance review

#### Option C: Stripe Checkout with Tax Enabled (Simplest)
- **Cost**: 0.5% of transaction volume in registered jurisdictions
- **Benefits**:
  - ✅ Easiest to implement (just enable in checkout)
  - ✅ Fully automated and compliant
  - ✅ No per-call API charges
  - ✅ Handles all tax complexity automatically
- **Implementation**:
  ```python
  checkout_params = {
      'customer': customer_id,
      'line_items': [...],
      'mode': 'subscription',
      'automatic_tax': {'enabled': True},  # Enable automatic tax
      # ... other params
  }
  ```
- **When to Use**: 
  - **Recommended** for most use cases
  - Simplest implementation
  - Cost is percentage-based, not per-call

**Recommendation for Production in Australia**:
- **Use Stripe Checkout with `automatic_tax: {'enabled': True}`** - This is the simplest and most compliant option
- Cost: 0.5% of transaction volume (much simpler than per-call pricing)
- This ensures GST compliance without complex manual tax logic

### 2. Avoid Stripe Financials API

**Current Status**: ✅ Cogento does NOT use Stripe Financials API

**Recommendation**:
- Do NOT use `stripe.financial_connections.*` APIs
- Do NOT use Stripe Sigma for reporting
- Do NOT use Revenue Recognition API
- Use standard free APIs for financial data retrieval

### 3. Optimize API Call Frequency

**Current Practices**:
- ✅ Using customer_id filter in Subscription.list() to reduce fetched subscriptions
- ✅ Caching customer lookups where possible
- ✅ Using Stripe's search API for efficient customer lookups

**Recommendations**:
- **Batch Operations**: When possible, use bulk operations instead of individual API calls
- **Caching**: Cache frequently accessed data (products, prices) to reduce API calls
- **Pagination**: Use proper pagination to avoid fetching unnecessary data
- **Filter at API Level**: Use Stripe's filtering capabilities (e.g., `customer` parameter in Subscription.list()) instead of fetching all records and filtering client-side

### 4. Monitor API Usage

**Stripe Dashboard**:
- Monitor API call volume in Stripe Dashboard under "Developers" → "Logs"
- Review API usage patterns to identify optimization opportunities
- Set up alerts for unusual API call volumes

**Cogento Logging**:
- All Stripe API calls are logged with INFO level
- Review logs to identify high-frequency API call patterns
- Look for opportunities to reduce redundant API calls

### 5. Code-Level Optimizations

**Current Optimizations**:
- ✅ Customer filtering at API level when customer_id is known
- ✅ Expanding related objects (e.g., `expand=['data.items.data.price']`) to reduce follow-up calls
- ✅ Using search API for efficient customer lookups

**Additional Recommendations**:
- **Expand Parameters**: Use Stripe's `expand` parameter to get related objects in a single call
- **Retrieve Only When Needed**: Only retrieve full objects when necessary (e.g., when items list is empty)
- **Avoid N+1 Queries**: Batch related API calls where possible

### 6. Rate Limiting Awareness

**Stripe Rate Limits**:
- Stripe has rate limits (typically 100 requests per second per API key)
- Exceeding rate limits results in 429 errors, not additional charges
- Implement retry logic with exponential backoff for rate limit errors

**Cogento Implementation**:
- Current code handles rate limits through error handling
- Consider implementing request queuing for bulk operations

---

## Summary

### ✅ Free API Calls (Currently Used by Cogento)

All of the following Stripe API operations used by Cogento are **FREE**:
- Customer CRUD operations
- Product CRUD operations  
- Price CRUD operations
- Subscription CRUD operations
- Payment Method operations
- Checkout Session creation
- Search operations
- List operations with pagination
- Metadata operations

### ⚠️ Paid API Services

**Current Status**: Cogento does NOT currently use these paid services, but **Stripe Tax may be required for GST compliance in Australia**

#### Stripe Tax API
- **Status**: ✅ Enabled in checkout sessions (`automatic_tax: {'enabled': True}`)
- **Setup Required**: Must configure in Stripe Dashboard first (Settings > Tax)
- **When Required**: 
  - **Mandatory for GST compliance in Australia** (10% GST on most products)
  - Required for multi-jurisdiction tax compliance
- **Cost Options**:
  1. **Test Mode**: ✅ FREE (up to 1,000 calls/day) - perfect for development
  2. **Production - Stripe Checkout with Tax**: 0.5% of transaction volume (recommended)
  3. **Production - Tax Calculation API**: $0.05 per calculation call
  4. **Production - Tax Transaction API**: $0.50 per transaction call
- **Implementation**: `automatic_tax: {'enabled': True}` in checkout sessions (see `subscriptions_create.py`)

#### Other Paid Services (NOT Used)
- ❌ Stripe Financials API
- ❌ Stripe Sigma
- ❌ Revenue Recognition API

### 💰 Transaction Fees (Separate from API Costs)

Stripe charges transaction fees for successful payments:
- **Standard**: 2.9% + $0.30 per successful card charge
- These fees are **separate** from API call costs
- Transaction fees apply regardless of API usage

### 📊 Cost Monitoring

To monitor Stripe API usage and costs:
1. **Stripe Dashboard**: Developers → Logs (view API call history)
2. **Stripe Dashboard**: Billing → View invoice details
3. **Cogento Logs**: Review application logs for API call patterns

---

## References

- [Stripe API Pricing](https://stripe.com/pricing)
- [Stripe Tax Pricing](https://stripe.com/tax/pricing)
- [Stripe API Rate Limits](https://stripe.com/docs/rate-limits)
- [Stripe API Reference](https://stripe.com/docs/api)

---

---

## GST/Tax Compliance in Australia

### Mandatory GST Requirements

In Australia, businesses must:
- Collect 10% GST on most goods and services
- Remit GST to the Australian Taxation Office (ATO)
- Provide accurate tax calculations and documentation

### Recommended Implementation

For production in Australia, **enable automatic tax in Stripe Checkout**:

```python
checkout_params = {
    'customer': customer_id,
    'line_items': [{'price': price_id, 'quantity': 1}],
    'mode': 'subscription',
    'automatic_tax': {'enabled': True},  # Enable for GST compliance
    'success_url': success_url,
    'cancel_url': cancel_url,
    # ... other params
}
```

**Cost**: 0.5% of transaction volume (not per-call)
**Benefits**: 
- Fully automated GST calculation
- ATO-compliant
- No manual tax logic required
- Handles tax-exempt products automatically

**Alternative**: Manual tax calculation (free but requires maintaining tax logic)

---

**Last Updated**: 2026-01-20  
**Version**: 1.1
