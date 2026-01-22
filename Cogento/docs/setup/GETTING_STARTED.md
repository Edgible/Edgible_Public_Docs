# Cogento Getting Started Guide

Complete guide for setting up Cogento from scratch using Docker.

## Prerequisites

- **Docker** (20.10+) and **Docker Compose** (2.0+)
- **Stripe Account** (for payment processing)
  - Get API keys from: https://dashboard.stripe.com/test/apikeys

## Step 1: Clone and Setup Project

```bash
# Navigate to your project directory
cd /path/to/Edgible/Cogento

# Create volumes directory (required for data persistence)
mkdir -p volumes/postgres/data
mkdir -p volumes/pgadmin/data
```

**Note**: The `volumes/` directory stores persistent data and is gitignored. It must exist before starting Docker Compose.

## Step 2: Configure Environment

```bash
# Copy environment template
cp .env.example .env

# Edit .env file with your settings
nano .env  # or use your preferred editor
```

### Required Environment Variables

**PostgreSQL Configuration:**
```bash
POSTGRES_PASSWORD=your-secure-password-here
POSTGRES_PORT=5432
```

**Shared Database Superuser:**
```bash
# Email for the initial superuser account in cogento_shared
# This user can manage all tenants via /shared/superuser interface
SHARED_SUPERUSER_EMAIL=admin@yourdomain.com
```

**Stripe Key Encryption:**
```bash
# Generate a secure encryption key for Stripe secret keys
# You can use: python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
STRIPE_KEY_ENCRYPTION_KEY=your-encryption-key-here
```

**Email Configuration (for login codes):**
```bash
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-app-password
MAIL_FROM=noreply@yourdomain.com
MAIL_FROM_NAME=Cogento
MAIL_SERVER=smtp.gmail.com
MAIL_PORT=587
MAIL_STARTTLS=true
```

**Application URLs:**
```bash
UI_BASE_URL=http://localhost:3000
API_PORT=8000
UI_PORT=3000
```

**pgAdmin (optional):**
```bash
PGADMIN_EMAIL=admin@cogento.com
PGADMIN_PASSWORD=admin
PGADMIN_PORT=5050
```

## Step 3: Start Services

```bash
# Start all services
docker-compose up -d

# View logs to verify startup
docker-compose logs -f
```

**What happens on first startup:**
1. PostgreSQL container initializes
2. Bootstrap script (`scripts/db/bootstrap_shared_db.sh`) runs automatically
3. Creates `cogento_shared` database
4. Creates all tables (tenants, users, login_tokens)
5. Creates initial superuser from `SHARED_SUPERUSER_EMAIL`

**Verify shared database was created:**
```bash
docker exec cogento-postgres psql -U cogento -d postgres -c "\l" | grep cogento_shared
```

You should see `cogento_shared` in the list.

## Step 4: Access the Application

**UI**: http://localhost:3000  
**API**: http://localhost:8000  
**API Docs**: http://localhost:8000/docs  
**pgAdmin**: http://localhost:5050

## Step 5: Login as Shared Superuser

1. Navigate to: **http://localhost:3000/shared/superuser/signin**

2. Enter the email you set in `SHARED_SUPERUSER_EMAIL` (e.g., `admin@yourdomain.com`)

3. Check your email for the login code (6 digits)

4. Enter the code on the verification page

5. You'll be redirected to the Shared Database Management landing page

## Step 6: Create Your First Tenant

### 6.1 Navigate to Tenants Page

From the shared superuser landing page, click **"Tenants"** or navigate to:
**http://localhost:3000/shared/superuser/tenants**

### 6.2 Create Tenant Record

1. Click **"Create Tenant"** button

2. Fill in the tenant form:
   - **Tenant ID**: Unique identifier (e.g., `edgible`, `kgtc`) - used in URLs and database names
   - **Name**: Display name (e.g., `Edgible`, `KGTC`)
   - **Domain**: Optional custom domain
   - **Status**: `active`
   - **Stripe Environment**: Choose `test` or `live`
     - `test`: For development/testing (uses test Stripe keys)
     - `live`: For production (uses live Stripe keys)
   - **Stripe Publishable Key**: Your Stripe publishable key
     - Test: `pk_test_...`
     - Live: `pk_live_...`
   - **Stripe Secret Key**: Your Stripe secret key (will be auto-encrypted)
     - Test: `sk_test_...`
     - Live: `sk_live_...`
   - **Database Name**: Optional (defaults to `cogento_{tenant_id}`)
   - **Superadmin Email**: Email for the tenant's superadmin user (required for database creation)

3. Click **"Create Tenant"**

The tenant record is now created in the `cogento_shared` database.

### 6.3 Create Tenant Database

1. Find your tenant in the tenants table

2. Click the **"Create Database"** action button (next to Edit/Delete)

3. Confirm the action

**What happens:**
- Creates the tenant database (e.g., `cogento_edgible`)
- Installs all schema (users, login_tokens, merge_fields, templates, jobs, subscriptions)
- Creates the superadmin user with the email you specified
- The superadmin can now log into the tenant

**Note**: The "Create Database" button will be disabled if `superadmin_email` is not set. Make sure to fill it in during tenant creation or edit the tenant first.

## Step 7: Login as Tenant Superadmin

1. Navigate to: **http://localhost:3000/{tenant_id}/admin/signin**

   Example: **http://localhost:3000/edgible/admin/signin**

2. Enter the superadmin email you specified when creating the tenant

3. Check your email for the login code

4. Enter the code to verify

5. You'll be logged in as the tenant superadmin

## Step 8: View Stripe Customers

1. From the tenant admin dashboard, navigate to **"Stripe Customers"** or go to:
   **http://localhost:3000/{tenant_id}/admin/customers**

2. The page will:
   - Connect to Stripe using the keys you configured
   - Fetch all Stripe customers
   - Display them in a table
   - Show customer details, subscriptions, and metadata

3. You can also:
   - Sync Stripe customers to the tenant database
   - View customer details
   - Manage subscriptions

## Step 9: Verify Setup

### Check Shared Database

```bash
# Connect to shared database
docker exec -it cogento-postgres psql -U cogento -d cogento_shared

# List tenants
SELECT tenant_id, name, stripe_environment, database_name, superadmin_email FROM tenants;

# List superusers
SELECT email, role, is_active FROM users;

# Exit
\q
```

### Check Tenant Database

```bash
# Connect to tenant database (replace 'edgible' with your tenant_id)
docker exec -it cogento-postgres psql -U cogento -d cogento_edgible

# List users
SELECT email, role, stripe_customer_id FROM users;

# Exit
\q
```

## Troubleshooting

### Shared Database Not Created

**Symptom**: `cogento_shared` database doesn't exist after startup

**Solution**:
1. Ensure data directory is empty for fresh initialization:
   ```bash
   docker-compose down
   rm -rf volumes/postgres/data/*
   docker-compose up -d
   ```

2. Check bootstrap script exists:
   ```bash
   ls -la scripts/db/bootstrap_shared_db.sh
   ls -la api/migrations/bootstrap_shared_schema.sql
   ```

3. Check logs for errors:
   ```bash
   docker-compose logs cogento-postgres | grep -i bootstrap
   ```

### Can't Login as Superuser

**Symptom**: No login code received or "Invalid code" error

**Solution**:
1. Verify `SHARED_SUPERUSER_EMAIL` is set in `.env`
2. Check email configuration in `.env` (MAIL_* variables)
3. Check API logs for email sending errors:
   ```bash
   docker-compose logs cogento-api | grep -i mail
   ```

### Tenant Database Creation Fails

**Symptom**: "Failed to create tenant database" error

**Solution**:
1. Ensure `superadmin_email` is set in the tenant record
2. Check PostgreSQL user has `CREATEDB` privilege (should be automatic)
3. Verify tenant database name doesn't already exist
4. Check API logs for detailed error:
   ```bash
   docker-compose logs cogento-api | grep -i "create.*database"
   ```

### Stripe Keys Not Working

**Symptom**: "Stripe keys not configured" or API errors

**Solution**:
1. Verify keys are entered correctly in tenant record
2. Ensure `STRIPE_KEY_ENCRYPTION_KEY` is set in `.env`
3. Check key format:
   - Test secret: `sk_test_...`
   - Live secret: `sk_live_...`
   - Publishable: `pk_test_...` or `pk_live_...`
4. Keys starting with `sk_` are automatically encrypted when saved

## Next Steps

Once your first tenant is set up:

1. **Sync Stripe Customers**: Use the Stripe Customers page to sync customers from Stripe to your tenant database

2. **Configure Merge Fields**: Set up merge fields for email campaigns

3. **Create Email Templates**: Design email templates for campaigns

4. **Send Campaigns**: Create and send email campaigns to your members

5. **Manage Users**: View and manage users in the Users page

## Environment Indicator

The footer shows the current environment:
- **TEST** (yellow dot): Test Stripe environment
- **LIVE** (green dot): Live/production Stripe environment  
- **SHARED** (blue dot): Shared database management area

## Additional Resources

- **API Documentation**: http://localhost:8000/docs
- **pgAdmin**: http://localhost:5050 (for database management)
- **Troubleshooting**: See `TROUBLESHOOTING.md` in the API directory

## Production Deployment

For production:
1. Use strong passwords in `.env`
2. Set `STRIPE_KEY_ENCRYPTION_KEY` to a secure, randomly generated key
3. Use live Stripe keys (set `stripe_environment = 'live'`)
4. Configure proper email SMTP settings
5. Use HTTPS for `UI_BASE_URL`
6. Set up proper backups for `volumes/postgres/data/`
