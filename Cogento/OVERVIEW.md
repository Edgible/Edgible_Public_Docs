# Cogento Overview

**Self-hosted membership management**

Cogento is a free, self-hosted membership management system that uses Stripe as source of truth, with PostgreSQL sync for fast queries and reporting. Perfect for sports clubs, nonprofits, and community groups that want full control without monthly SaaS fees.

## Key Features

- ✅ **FREE** - Self-hosted, no monthly SaaS fees
- ✅ **Stripe as Source of Truth** - Professional payment processing with payment integrity
- ✅ **PostgreSQL Sync** - Fast queries, search, reporting, and analytics
- ✅ **Self-Hosted** - Full control, no vendor lock-in
- ✅ **Open Source** - Adapt, extend, contribute
- ✅ **Real-World Proven** - Built for 200+ member tennis club

## Architecture

Cogento uses a hybrid cloud + self-hosted architecture:

- **Stripe (Cloud)** - Source of truth for all member data, subscriptions, and payments
- **PostgreSQL (Self-Hosted)** - Read replica synced from Stripe via webhooks for fast queries
- **FastAPI Backend** - Handles webhooks, syncs data, provides API
- **Web UI** - Member directory, search, filters, reporting, analytics

See [Architecture: Stripe + PostgreSQL Sync](./architecture/architecture_stripe_postgres_sync.md) for detailed architecture documentation.

## Why Cogento?

**The Market Gap:** Thousands of small clubs need FREE, self-hosted, Stripe-powered membership management, but existing solutions are:
- ❌ Expensive SaaS ($40-500/month)
- ❌ Vendor lock-in (can't export easily)
- ❌ Use own database + Stripe sync (data consistency problems)
- ❌ Don't use Stripe as source of truth

**Cogento Fills the Gap:**
- ✅ FREE (self-hosted, no monthly fees)
- ✅ Stripe as source of truth (no sync issues)
- ✅ Fast queries via PostgreSQL sync
- ✅ Full control (no vendor lock-in)

For more details, see:
- [Market Opportunity](./planning/market_opportunity.md) - Market analysis and opportunity
- [Platform Comparison](./stripe/stripe_native_membership_platforms.md) - Why Cogento is unique in the market

## Use Cases

- **Sports clubs** (tennis, cricket, soccer, rugby, golf, etc.)
- **Non-profit organizations** with membership models
- **Community groups** and associations
- **Any membership-based organization** wanting self-hosted solution

## Quick Start

**New to Cogento?** Start with the [Getting Started Guide](./setup/GETTING_STARTED.md) for a complete walkthrough from fresh install to your first tenant.

## Documentation

- [Architecture: Stripe + PostgreSQL Sync](./architecture/architecture_stripe_postgres_sync.md) - How Cogento syncs Stripe to PostgreSQL
- [Stripe Customer Metadata](./stripe/stripe_customer_metadata_info.md) - Using Stripe Customer records for member data
- [Platform Comparison](./stripe/stripe_native_membership_platforms.md) - Why Cogento is unique in the market
- [Migration Strategy](./planning/membership_migration_strategy.md) - How to migrate from existing systems

See [INDEX.md](./INDEX.md) for a complete documentation index.

## Status

🚧 **In Development** - Currently being built for a 200+ member tennis club. Documentation and code coming soon!

## License

[To be determined - likely MIT or Apache 2.0]

## Contributing

[Coming soon - contribution guidelines]
