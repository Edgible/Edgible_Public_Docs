---
name: Membership Migration Strategy
overview: Explore alternative membership platforms with CSV import support (Option A) and hybrid solutions that keep Stripe payments while using separate member management (Option E). Includes data export tools and evaluation criteria.
todos:
  - id: research_platforms
    content: Research and compile detailed comparison of FREE platforms only (SportMember, Admidio, Zeffy, Givebutter, CiviCRM, Join It) - focusing on CSV import capabilities, Stripe integration, free tier limitations (member count, features), and tennis club features
    status: pending
  - id: create_export_script
    content: Create Python script to export member data from PostgreSQL and Google Sheets to CSV format compatible with target platforms
    status: pending
  - id: data_audit
    content: Audit current member data structure in Google Sheets and PostgreSQL to identify all fields, data quality issues, and mapping requirements
    status: pending
  - id: platform_trials
    content: Set up free accounts with top 3-4 FREE platforms and test CSV import with sample member data (10-20 members). Verify free tier limits support 200+ members.
    status: pending
    dependencies:
      - create_export_script
      - data_audit
  - id: evaluation_matrix
    content: Create detailed evaluation matrix comparing platforms on import ease, Stripe integration, cost, features, and club-specific needs
    status: pending
    dependencies:
      - research_platforms
      - platform_trials
  - id: hybrid_architecture_design
    content: Design hybrid architecture approach (if chosen) - document how free member management platform would integrate with existing Stripe payment system via webhooks or manual sync
    status: pending
    dependencies:
      - research_platforms
  - id: stripe_source_of_truth_design
    content: Design "Stripe as Source of Truth" implementation - document how to use Stripe Customers/Subscriptions as membership database, what metadata fields to use, and how to extend existing subscription_service.py
    status: pending
    dependencies:
      - data_audit
  - id: stripe_member_directory_service
    content: Extend subscription_service.py to add member directory functions - list all customers, search/filter, get customer with subscriptions, handle metadata fields
    status: pending
    dependencies:
      - stripe_source_of_truth_design
  - id: stripe_import_script
    content: Create import_members_to_stripe.py script to migrate Google Sheets data into Stripe Customers with proper metadata and subscriptions
    status: pending
    dependencies:
      - data_audit
      - stripe_source_of_truth_design
---

# Membership Migration Strategy for Tennis Club

## Current Situation

- **200+ members** stored in Google Spreadsheets and PostgreSQL database
- Custom website with Stripe payment integration
- ClubSpark does NOT support member import - feature was removed ~2 years ago (only export remains)
- Manual entry is impractical for 200+ members
- **Constraint: Any solution must be FREE** (no paid platforms)

## Key Insight

Given that:

- You already have a working Stripe payment system (Edgible_Stripe_API)
- You have a PostgreSQL database with member data
- You have a custom website infrastructure
- **Any solution must be FREE**
- **Stripe will remain the payment gateway** - you're familiar with Stripe APIs

**⭐ BEST STRATEGY: Stripe as Source of Truth** - Your code already reads from Stripe! You're using:

- `stripe.Customer.list()` and `stripe.Customer.retrieve()` 
- `stripe.Subscription.list()` 
- Customer metadata to store `user_id`
- Stripe webhooks for real-time updates

**This is the most elegant solution:**

- ✅ **FREE** (already using Stripe)
- ✅ Uses existing infrastructure (`subscription_service.py` already queries Stripe)
- ✅ Leverages your Stripe API knowledge
- ✅ Real-time data via webhooks (already implemented)
- ✅ Built-in subscription management (Stripe handles renewals, payments, cancellations)
- ✅ No vendor lock-in beyond Stripe (which you're already committed to)
- ✅ Can store additional member data in Stripe Customer metadata or Description fields

## Approach: Stripe as Source of Truth Strategy

**Primary Strategy:** Build member management system that uses Stripe Customers and Subscriptions as the master database.

### How Stripe as Source of Truth Works

**Stripe Objects as Data Storage:**

- **Stripe Customers** = Member records (email, name, phone, address)
- **Stripe Subscriptions** = Membership subscriptions (plan, status, dates, auto-renewal)
- **Customer Metadata** = Additional member fields (member_number, join_date, notes, custom fields)
- **Subscription Metadata** = Membership-specific data (membership_type, court_access, etc.)
- **Stripe Invoices/Payments** = Payment history (already tracked by Stripe)

**Architecture:**

```
Stripe (Source of Truth)
  ├── Customers (Member Directory)
  │   ├── email, name, phone, address (built-in fields)
  │   └── metadata (custom fields: member_number, join_date, status, notes)
  ├── Subscriptions (Active Memberships)
  │   ├── product_id (membership package)
  │   ├── status (active, canceled, past_due)
  │   ├── current_period_start/end (membership dates)
  │   └── metadata (membership-specific data)
  └── Invoices/PaymentIntents (Payment History)
      └── Already fully managed by Stripe
```

**Implementation Options:**

1. **Pure Stripe** - All reads from Stripe API, PostgreSQL only for auth/sessions
2. **Stripe + Local Cache** - Read from Stripe, cache in PostgreSQL for performance
3. **Stripe + Sync** - Stripe is master, sync to PostgreSQL for reporting/backup

**Your Current Code Already:**

- ✅ Reads subscriptions from Stripe (`get_all_subscriptions_admin()`)
- ✅ Stores `user_id` in Customer metadata
- ✅ Handles webhooks for real-time updates
- ✅ Queries Stripe Products/Prices for membership packages

## Also Explore Options A & E (Alternative Free Platforms)

### Option A: FREE Membership Platforms with Import Support

**FREE Platforms to Evaluate:**

1. **SportMember** ⭐ **TENNIS/SPORTS FOCUSED**

- FREE membership management for sports clubs
- CSV/Excel import supported
- Cloud-based with mobile app
- Payment collection (need to verify Stripe integration)
- Member registration, scheduling, messaging
- Pros: Sports club focused, free, CSV import, modern interface
- Cons: Need to verify Stripe integration, may have usage limits
- Website: sportmember.com

2. **Admidio** ⭐ **OPEN SOURCE**

- FREE open-source membership management
- CSV import/export for users
- Self-hosted (full control) or cloud options
- Access control, member lists, event management
- Customizable fields and roles
- Pros: Completely free, open source, full control, CSV import
- Cons: Requires hosting (or use their free hosting), may need Stripe plugin/add-on
- Website: admidio.org

3. **Zeffy** ⭐ **NONPROFIT FOCUSED**

- FREE membership management for nonprofits
- CSV import/export capability
- Customizable membership forms
- Automatic renewals
- Payment processing (need to verify Stripe integration)
- Pros: Free, easy import, nonprofit focused
- Cons: Need to verify Stripe integration, may be more fundraising-focused
- Website: zeffy.com

4. **Givebutter**

- FREE membership management and CRM
- Import existing members into CRM
- Payment processing included
- Pros: Free, all-in-one, import support
- Cons: More fundraising/event focused, may lack tennis club features
- Website: givebutter.com

5. **Join It**

- Free tier available (check limitations)
- Import tool for existing member records
- Stripe integration mentioned
- Customizable CRM fields
- Pros: Import tool, Stripe support, guided migration
- Cons: Need to verify free tier limitations (member count, features)
- Website: joinit.com

6. **CiviCRM** ⭐ **OPEN SOURCE**

- FREE open-source CRM for nonprofits
- Data import/export capabilities
- Integrates with WordPress, Drupal, Joomla
- Member management, events, contributions
- Pros: Completely free, powerful, flexible
- Cons: Requires WordPress/Drupal setup, learning curve, may need Stripe extension
- Website: civicrm.org

**Evaluation Criteria (FREE Platforms Only):**

- CSV/Excel import capability and ease of use
- Stripe integration quality (or payment gateway support)
- **Must be completely FREE** (no paid tiers, no transaction fees)
- Member capacity (200+ members) - check free tier limits
- Features: member directory, renewals, communication tools
- User-friendliness for committee members
- Support quality and documentation (free platforms may have limited support)
- Data export capability (future-proofing)
- Tennis/sports club specific features (if available)

### Option E: Hybrid Approach (Keep Stripe, Add Member Management)

**Architecture:**

- Keep existing Stripe payment system (current custom website)
- Integrate a member management platform for member data/directory
- Sync between systems via API or manual processes

**Hybrid Options:**

1. **Free Platform + Stripe Webhook Integration**

- Use free platform (SportMember/Admidio/Zeffy/etc.) for member management, directory, renewals
- Keep Stripe for payments (already working)
- Sync payment status via webhooks or manual process
- Pros: Best of both worlds, keep existing payment flow, free member management
- Cons: Requires integration work, potential sync issues, may need manual sync

2. **Stripe as Source of Truth** ⭐⭐⭐ **BEST OPTION - FREE & LEVERAGES EXISTING CODE**

- Use Stripe Customers as member directory (read via `stripe.Customer.list()`)
- Use Stripe Subscriptions for membership status (already implemented in your code)
- Store additional member data in Stripe Customer/Subscription metadata
- Build member management UI that reads from Stripe API
- Import Google Sheets → Create Stripe Customers with metadata
- Your `subscription_service.py` already does most of this!
- Pros: 
  - ✅ FREE (already using Stripe)
  - ✅ Code already partially implemented
  - ✅ Real-time via webhooks (already set up)
  - ✅ Single source of truth (Stripe)
  - ✅ No sync issues (Stripe handles everything)
  - ✅ Familiar API (you know Stripe)
- Cons: 
  - Stripe API rate limits (6,000 req/min - plenty for 200 members)
  - May need caching for performance (PostgreSQL as read cache)
  - Stripe data export for offline backups
- **Implementation:** Extend existing `subscription_service.py` to add:
  - Member directory endpoints (list all customers)
  - Member search/filter
  - Member detail view (customer + subscriptions)
  - Import script (CSV/Google Sheets → Stripe Customers)
  - Admin UI (build on existing `/ui/users.html`)

3. **Self-Hosted Open Source + Stripe Integration**

- Use Admidio or CiviCRM (self-hosted, completely free)
- Connect to existing Stripe account via plugins/extensions
- Pros: Completely free, full control, no vendor lock-in
- Cons: Requires hosting, setup complexity, may need developer help

### Data Migration Tool

**For "Stripe as Source of Truth" Strategy:**

**Import Script** to migrate data TO Stripe:

- Location: `Edgible_Stripe_API/scripts/import_members_to_stripe.py`
- Functionality:
  - Read members from Google Sheets (or PostgreSQL)
  - For each member:
    - Create Stripe Customer with name, email, phone, address
    - Add member data to Customer metadata (member_number, join_date, status, notes, custom fields)
    - If they have active subscription, create Stripe Subscription
    - Link subscription to appropriate Stripe Product/Price (membership package)
  - Handle duplicates (check existing customers by email: `stripe.Customer.list(email=email)`)
  - Validate data before import
  - Generate import report (success/failures)

**For Alternative Free Platforms:**

**Export Script** to prepare data for migration:

- Location: `Edgible_Stripe_API/scripts/export_members_to_csv.py`
- Functionality:
  - Export members from Stripe (read via `stripe.Customer.list()`) OR PostgreSQL/Google Sheets
  - Combine with Google Sheets data (if needed)
  - Generate CSV in format compatible with target platforms
  - Map fields: name, email, membership type, start date, end date, status
  - Handle subscription/payment history
  - Validate data completeness
- Output Format:
  - CSV file with standardized member data
  - Multiple CSV files if platforms require different formats
  - Data mapping document for each platform

### Recommended Action Plan

**Phase 1: Research & Evaluation (Week 1-2)**

1. **Verify free tier limitations** for top platforms (member count, features, storage)
2. Request demos/trials from top 3-4 FREE platforms (SportMember, Admidio, Zeffy, etc.)
3. Test CSV import process with sample data (10-20 members) for each platform
4. **Evaluate Stripe integration options** - verify if free platforms support Stripe
5. **Assess extending current system** - evaluate development effort vs. third-party platforms
6. Check user reviews and support quality (free platforms may have limited support)

**Phase 2: Data Preparation (Week 2-3)**

1. Audit current member data (Google Sheets + PostgreSQL)
2. Clean and standardize data
3. Create data export script
4. Generate migration-ready CSV files
5. Document data mapping for chosen platform

**Phase 3: Migration (Week 3-4)**

1. Set up chosen platform
2. Import member data (staged: test group, then full import)
3. Configure membership packages/subscriptions
4. Test Stripe integration
5. Train committee members on new system

**Phase 4: Transition (Week 4-5)**

1. Run both systems in parallel for 1 month
2. Monitor for issues
3. Migrate payment processing if needed
4. Decommission old system

### Decision Matrix (FREE Options Only)

| Platform | Import Ease | Stripe Integration | Cost | Club Features | Free Tier Limits | Recommendation |
|----------|-------------|-------------------|------|---------------|------------------|----------------|
| **SportMember** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ (verify) | FREE | ⭐⭐⭐⭐⭐ | Need to check | **HIGH** - Sports club focused, CSV import |
| **Admidio** | ⭐⭐⭐⭐ | ⭐⭐ (plugin needed) | FREE | ⭐⭐⭐ | Unlimited (self-hosted) | **HIGH** - Open source, full control |
| **Stripe as Source of Truth** | N/A (use Stripe API) | ⭐⭐⭐⭐⭐ | FREE | ⭐⭐⭐⭐ (build UI) | Unlimited | **HIGHEST** - Already partially implemented, elegant solution |
| Zeffy | ⭐⭐⭐⭐ | ⭐⭐⭐ (verify) | FREE | ⭐⭐ | Need to check | **MEDIUM** - Nonprofit focused |
| Givebutter | ⭐⭐⭐ | ⭐⭐⭐ | FREE | ⭐⭐ | Need to check | **MEDIUM** - More fundraising focused |
| CiviCRM | ⭐⭐⭐ | ⭐⭐ (extension) | FREE | ⭐⭐⭐ | Unlimited (self-hosted) | **MEDIUM** - Complex but powerful |
| Join It | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | FREE* | ⭐⭐⭐ | Need to verify limits | **MEDIUM** - Check free tier restrictions |

### Top Recommendations (FREE Only)

**🥇 Option 1: Stripe as Source of Truth (HIGHEST PRIORITY)** ⭐⭐⭐

- **Why:** Your code already reads from Stripe! Just extend what you have
- **Cost:** FREE (already using Stripe)
- **Effort:** Low-Medium (extend existing `subscription_service.py`, add member directory UI)
- **Pros:** 
  - Already partially implemented in your codebase (`subscription_service.py` queries Stripe)
  - Single source of truth (no sync issues)
  - Real-time via webhooks (already set up)
  - You know Stripe APIs
  - No third-party dependencies
- **Cons:** 
  - Need to build member management UI
  - May want PostgreSQL cache for performance (optional)
- **Best if:** You want the most elegant, maintainable solution
- **Implementation:** 
  - Extend `subscription_service.py` with member directory functions (`stripe.Customer.list()`)
  - Build member management UI (extend existing `/ui/users.html`)
  - Import script: Google Sheets → Stripe Customers with metadata
  - Use Stripe Customer metadata for additional member fields

**🥈 Option 2: SportMember**

- **Why:** Sports club focused, CSV import, free tier
- **Cost:** FREE (need to verify member limits)
- **Effort:** Low (migrate data, set up account)
- **Pros:** Sports-focused, modern interface, CSV import
- **Cons:** Need to verify Stripe integration and free tier limits
- **Best if:** Free tier supports 200+ members and has Stripe integration

**🥉 Option 3: Admidio (Self-Hosted)**

- **Why:** Completely free open source, full control
- **Cost:** FREE (hosting costs if self-hosted)
- **Effort:** Medium-High (setup and configuration)
- **Pros:** Open source, unlimited members, CSV import
- **Cons:** Requires hosting, may need Stripe plugin development
- **Best if:** You have hosting available and technical resources

### Key Questions to Answer Before Decision

**For "Stripe as Source of Truth" Strategy (RECOMMENDED):**

1. **What additional member fields do you need beyond Stripe Customer defaults?**
   - Stripe Customer provides: email, name, phone, address, description
   - Store in Customer metadata: member_number, join_date, emergency_contact, notes, custom fields
   - Plan metadata schema (what keys/values to use)

2. **Do you need member directory/search features?**
   - List all members (paginated `stripe.Customer.list()`)
   - Search by name, email, member number (filter in API or cache)
   - Filter by membership status (via subscriptions)
   - Member detail view (customer + all subscriptions + invoices)

3. **Performance/Caching strategy?**
   - Pure Stripe (read directly from API - simple, may be slower)
   - Stripe + PostgreSQL cache (read from cache, sync via webhooks - faster, more complex)
   - For 200 members, pure Stripe is probably fine (Stripe API is fast)

4. **Do you have technical resources to extend your current system?** 
   - Even part-time/volunteer developer can extend `subscription_service.py`
   - You already know Stripe APIs, so implementation should be straightforward

**For Alternative Free Platforms:**

5. **For free platforms: What are the free tier limits?** (member count - verify 200+ is supported, storage, features)

6. Are you willing to self-host (Admidio/CiviCRM) or need cloud-hosted solution?
