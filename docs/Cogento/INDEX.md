# Cogento Documentation Index

This directory contains all documentation for the Cogento membership management system. The documentation is organized into logical categories to help you find what you need quickly.

## Getting Started

- **[OVERVIEW.md](./OVERVIEW.md)** - Overview of Cogento, key features, architecture, and use cases
- **[INDEX.md](./INDEX.md)** - This file - complete documentation index

## Documentation Structure

### 📚 [Setup](./setup/)
**Getting started and deployment guides**

- **[GETTING_STARTED.md](./setup/GETTING_STARTED.md)** - Complete guide for setting up Cogento from scratch using Docker
- **[README_DOCKER.md](./setup/README_DOCKER.md)** - Docker setup and service configuration details
- **[AWS_DEPLOYMENT.md](./setup/AWS_DEPLOYMENT.md)** - Instructions for deploying Cogento on AWS

### 💳 [Stripe](./stripe/)
**Stripe integration and API documentation**

- **[stripe-apis.md](./stripe/stripe-apis.md)** - Comprehensive guide to Stripe API endpoints and usage
- **[stripe_customer_attributes.md](./stripe/stripe_customer_attributes.md)** - Customer attribute mapping and structure
- **[stripe_customer_metadata_info.md](./stripe/stripe_customer_metadata_info.md)** - Customer metadata handling and best practices
- **[stripe_native_membership_platforms.md](./stripe/stripe_native_membership_platforms.md)** - Comparison with native Stripe membership platforms
- **[stripe_subscription_attributes.md](./stripe/stripe_subscription_attributes.md)** - Subscription attribute mapping and structure

### 🏗️ [Architecture](./architecture/)
**System architecture and database design**

- **[architecture_stripe_postgres_sync.md](./architecture/architecture_stripe_postgres_sync.md)** - Architecture overview of Stripe to PostgreSQL synchronization
- **[multi_tenancy.md](./architecture/multi_tenancy.md)** - Multi-tenancy design and implementation
- **[postgresql_schema_summary.md](./architecture/postgresql_schema_summary.md)** - PostgreSQL database schema documentation
- **[schema_design_users_table.md](./architecture/schema_design_users_table.md)** - Users table schema design and rationale

### 📋 [Metadata](./metadata/)
**Metadata handling and schema**

- **[metadata_handling.md](./metadata/metadata_handling.md)** - How metadata is stored and managed in Cogento
- **[metadata_schema_proposal.md](./metadata/metadata_schema_proposal.md)** - Proposed metadata schema design

### 🛠️ [Tools](./tools/)
**Operational tools and troubleshooting**

- **[CSV_FIELD_MAPPING.md](./tools/CSV_FIELD_MAPPING.md)** - CSV field mapping for bulk import operations
- **[ENCRYPT_KEYS.md](./tools/ENCRYPT_KEYS.md)** - Stripe key encryption and security
- **[IMPORT_TOOLS.md](./tools/IMPORT_TOOLS.md)** - Bulk import tools and usage
- **[TROUBLESHOOTING.md](./tools/TROUBLESHOOTING.md)** - Common issues and solutions

### 📥 [Import](./import/)
**Bulk import CSV formats and specifications**

- **[bulk_import_csv_formats.md](./import/bulk_import_csv_formats.md)** - Complete CSV format documentation for all bulk import operations with examples

### 📊 [Planning](./planning/)
**Research, requirements, and migration strategies**

- **[market_opportunity.md](./planning/market_opportunity.md)** - Market analysis and opportunity assessment
- **[membership_migration_strategy.md](./planning/membership_migration_strategy.md)** - Strategy for migrating existing memberships
- **[requirements.md](./planning/requirements.md)** - System requirements and specifications
- **[research_stripe_email_handling.md](./planning/research_stripe_email_handling.md)** - Research on Stripe email handling capabilities

### 📐 [Standards](./standards/)
**Coding standards and conventions**

- **[naming.md](./standards/naming.md)** - Naming conventions and standards
- **[ui_standards.md](./standards/ui_standards.md)** - UI/UX standards and design guidelines

## Quick Start

**New to Cogento?** Start here:
1. Read [OVERVIEW.md](./OVERVIEW.md) for an introduction to Cogento
2. Read [GETTING_STARTED.md](./setup/GETTING_STARTED.md) for initial setup
3. Review [README_DOCKER.md](./setup/README_DOCKER.md) for Docker configuration
4. Check [stripe-apis.md](./stripe/stripe-apis.md) for Stripe integration

**Setting up Stripe integration?**
- Start with [stripe-apis.md](./stripe/stripe-apis.md)
- Review customer attributes: [stripe_customer_attributes.md](./stripe/stripe_customer_attributes.md)
- Understand metadata: [stripe_customer_metadata_info.md](./stripe/stripe_customer_metadata_info.md)

**Need to import data?**
- See [bulk_import_csv_formats.md](./import/bulk_import_csv_formats.md) for complete CSV format documentation
- Check [CSV_FIELD_MAPPING.md](./tools/CSV_FIELD_MAPPING.md) for field mappings
- Follow [IMPORT_TOOLS.md](./tools/IMPORT_TOOLS.md) for usage instructions

**Having issues?**
- Check [TROUBLESHOOTING.md](./tools/TROUBLESHOOTING.md)

## Knowledge Base

This documentation is used as the knowledge base for Cogento's help chat widget. The help chat can answer questions about:
- Setup and deployment
- Stripe integration
- Architecture and database design
- Metadata handling
- Import tools and operations (including CSV formats)
- Standards and conventions

Ask the help chat widget any questions about these topics, and it will search through this documentation to provide accurate answers.
