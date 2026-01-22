# Multi-Tenancy Architecture for Cogento

## Table of Contents

1. [Overview](#overview)
2. [Architecture Overview](#architecture-overview)
   - [Tenant Identification](#tenant-identification)
   - [Database Architecture](#database-architecture)
   - [API Structure](#api-structure)
3. [Shared Database Schema](#shared-database-schema)
   - [tenants Table](#tenants-table)
   - [users Table](#users-table)
4. [Tenant Middleware](#tenant-middleware)
   - [Middleware Flow](#middleware-flow)
   - [Implementation](#middleware-implementation)
   - [Helper Functions](#helper-functions)
5. [Database Connection Manager](#database-connection-manager)
   - [Connection Pooling](#connection-pooling)
   - [Implementation](#db-connection-implementation)
6. [Stripe Key Service](#stripe-key-service)
   - [Service Design](#stripe-key-service-design)
   - [Security](#stripe-key-security)
7. [Webhook Routing](#webhook-routing)
   - [Routing Strategy](#webhook-routing-strategy)
   - [Implementation](#webhook-implementation)
8. [Frontend Routing](#frontend-routing)
   - [Path-Based Routing](#path-based-routing)
   - [API Client](#api-client)
9. [Migration Strategy](#migration-strategy)
   - [Migration Approaches](#migration-approaches)
   - [Migration Tracking](#migration-tracking)
10. [Superuser Access](#superuser-access)
    - [Authentication](#superuser-authentication)
    - [API Endpoints](#superuser-api-endpoints)
11. [Database Recovery](#database-recovery)
12. [Monitoring](#monitoring)
13. [Summary](#summary)

---

## Overview

Enable multi-tenancy support where multiple tenants can use a single Cogento deployment instance, with:

- **Path-based routing** for UI (e.g., `/kgtc/`, `/org1/`)
- **Path-based routing** for API (e.g., `/stripe/kgtc/customers`, `/postgres/kgtc/users`)
- **Separate PostgreSQL database per tenant** (one PostgreSQL server, multiple databases)
- **Separate Stripe account per tenant** (each tenant has own Stripe keys)

---

## Architecture Overview

### Tenant Identification

**UI (Path-based):**
- Routes: `/kgtc/`, `/org1/`, `/org2/`
- Extract tenant identifier from URL path
- Frontend routing handles tenant context

**API (Path-based):**
- Routes: `/stripe/{tenant_id}/customers`, `/postgres/{tenant_id}/users`
- Extract tenant identifier from URL path
- API middleware extracts tenant from path
- Validates tenant exists and user has access

**Benefits:**
- Path-based: User-friendly URLs, clear tenant context in UI and API
- Consistent routing pattern across UI and API
- Tenant visible in URL for easier debugging
- More RESTful API design
- Better for caching/CDN (can cache per tenant path)

### Database Architecture

**PostgreSQL Server Structure:**
```
PostgreSQL Instance
├── cogento_shared (shared database)
│   ├── tenants (tenant configuration, Stripe keys, and settings)
│   └── users (global superuser accounts with role = 'superuser')
│
├── cogento_kgtc (tenant database)
│   ├── users
│   ├── subscriptions
│   └── user_roles
│
├── cogento_org1 (tenant database)
│   ├── users
│   ├── subscriptions
│   └── user_roles
│
└── cogento_org2 (tenant database)
    ├── users
    ├── subscriptions
    └── user_roles
```

**Shared Database (`cogento_shared`):**
- `tenants` table - Tenant configuration, Stripe keys, and settings (all consolidated)
- `users` table - Global user accounts (users with role = 'superuser' can access all tenants)

**Per-Tenant Databases:**
- Each tenant gets its own database: `cogento_{tenant_id}`
- Same schema structure (users, subscriptions, user_roles)
- **Can have different columns** if needed (schema flexibility)
- Complete data isolation

### API Structure

**API Routes (Path-based with tenant):**
- Routes include tenant identifier in path
- `/stripe/{tenant_id}/customers` - Stripe Customer operations
- `/stripe/{tenant_id}/subscriptions` - Stripe Subscription operations
- `/postgres/{tenant_id}/users` - PostgreSQL user queries
- `/postgres/{tenant_id}/subscriptions` - PostgreSQL subscription queries
- `/postgres/{tenant_id}/reports` - Reporting endpoints
- `/webhooks/stripe/{tenant_id}` - Stripe webhook handling

**Admin Routes (No tenant in path):**
- `/admin/tenants` - List all tenants (superuser only)
- `/admin/tenants/{tenant_id}/*` - Tenant management (superuser only)

**Example API Calls:**
```bash
# Get customers for kgtc tenant
curl -H "Authorization: Bearer <token>" \
     https://api.cogento.com/stripe/kgtc/customers

# Get users from PostgreSQL for kgtc tenant
curl -H "Authorization: Bearer <token>" \
     https://api.cogento.com/postgres/kgtc/users
```

---

## Shared Database Schema

### Purpose
The shared database (`cogento_shared`) stores tenant configuration, Stripe keys, and superuser accounts that are used across all tenants.

### tenants Table

**Purpose:** Store tenant configuration, metadata, and Stripe keys (all consolidated in one table).

```sql
CREATE TABLE tenants (
    tenant_id VARCHAR(100) PRIMARY KEY,  -- e.g., 'kgtc', 'org1', 'org2'
    name VARCHAR(255) NOT NULL,         -- Display name: 'KGTC', 'Organization Name'
    domain VARCHAR(255),                -- Optional custom domain: 'example.com'
    status VARCHAR(50) DEFAULT 'active', -- 'active', 'suspended', 'deleted'
    
    -- Flexible configuration (theme, features, limits, etc.)
    config JSONB NOT NULL DEFAULT '{}'::jsonb,
    
    -- Stripe API keys (encrypted at rest)
    stripe_test_secret_key_encrypted TEXT,      -- Encrypted secret key
    stripe_test_publishable_key VARCHAR(255),   -- Public key (not encrypted)
    stripe_live_secret_key_encrypted TEXT,      -- Encrypted secret key
    stripe_live_publishable_key VARCHAR(255),   -- Public key (not encrypted)
    stripe_environment VARCHAR(50) DEFAULT 'test', -- 'test' or 'live'
    stripe_key_encryption_method VARCHAR(50) DEFAULT 'aes-256-gcm', -- Encryption method
    
    -- Audit fields
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_by VARCHAR(255)                      -- Who updated (for audit)
);

CREATE INDEX idx_tenants_status ON tenants(status);
CREATE INDEX idx_tenants_domain ON tenants(domain) WHERE domain IS NOT NULL;
CREATE INDEX idx_tenants_config_gin ON tenants USING GIN(config);
CREATE INDEX idx_tenants_stripe_environment ON tenants(stripe_environment);
```

**Fields:**
- `tenant_id` - Unique identifier for the tenant (used in URLs, headers, database names)
- `name` - Human-readable tenant name
- `domain` - Optional custom domain for the tenant
- `status` - Tenant status (active, suspended, deleted)
- `config` - Flexible JSONB configuration (theme, features, limits, etc.)
- `stripe_*` - Stripe API keys (test and live, secret keys encrypted)
- Timestamps and audit fields

**Example Config Structure:**
```json
{
  "theme": {
    "primary_color": "#3498db",
    "logo_url": "https://example.com/logo.png"
  },
  "features": {
    "enable_member_portal": true,
    "enable_self_service": true
  },
  "limits": {
    "max_members": 500,
    "max_subscriptions_per_member": 3
  }
}
```

**Security:**
- Secret keys MUST be encrypted at rest
- Use strong encryption (AES-256-GCM recommended)
- Encryption key stored in environment variable or key management service
- Never log secret keys
- Publishable keys can be stored unencrypted (they're public)

**Constraints:**
- `tenant_id` must be URL-safe (alphanumeric, hyphens, underscores)
- `tenant_id` used to construct database name: `cogento_{tenant_id}`

### users Table

**Purpose:** Store global user accounts. Users with role = 'superuser' can access all tenants.

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'superuser',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by VARCHAR(255),  -- Who created this user
    last_login_at TIMESTAMP WITH TIME ZONE,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(is_active);
CREATE INDEX idx_users_role ON users(role);
```

---

## Tenant Middleware

### Middleware Flow

```
Request → Tenant Middleware → Route Handler
   ↓              ↓                    ↓
Extract      Validate            Use tenant
tenant_id      tenant              context
   ↓              ↓                    ↓
Load         Load Stripe         Return
database     keys                response
```

### Middleware Implementation

The tenant middleware is responsible for:
1. Extracting tenant identifier from URL path (e.g., `/stripe/club1/customers`)
2. Validating tenant exists and is active
3. Resolving database connection for tenant
4. Loading Stripe keys for tenant
5. Injecting tenant context into request
6. Handling superuser access patterns

```python
@router.middleware("http")
async def tenant_middleware(request: Request, call_next):
    """
    Multi-tenancy middleware that:
    1. Extracts tenant from URL path (e.g., /stripe/club1/customers)
    2. Validates tenant exists
    3. Loads tenant's database connection
    4. Loads tenant's Stripe keys
    5. Injects tenant context into request.state
    """
    # Step 1: Extract tenant identifier from path
    path_parts = request.url.path.split('/')
    
    # Skip admin routes (no tenant in path)
    if len(path_parts) >= 2 and path_parts[1] == 'admin':
        user = get_current_user(request)
        is_superuser = await check_superuser(user.email) if user else False
        if is_superuser:
            request.state.is_superuser = True
            return await call_next(request)
        else:
            return JSONResponse(
                {"error": "Superuser access required"}, 
                status=403
            )
    
    # Extract tenant_id from path
    # Pattern: /{service}/{tenant_id}/{resource}
    # Examples: /stripe/club1/customers, /postgres/club1/users
    if len(path_parts) >= 3:
        tenant_id = path_parts[2]  # e.g., 'club1' from '/stripe/club1/customers'
    else:
        return JSONResponse(
            {"error": "Invalid API path - tenant identifier required"}, 
            status=400
        )
    
    # Step 2: Handle superuser access
    user = get_current_user(request)  # From auth middleware
    is_superuser = await check_superuser(user.email) if user else False
    
    if is_superuser:
        # Superuser can access any tenant via path
        tenant = await get_tenant(tenant_id)
        if not tenant:
            return JSONResponse(
                {"error": "Invalid tenant"}, 
                status=403
            )
    else:
        # Regular users - validate tenant access
        tenant = await get_tenant(tenant_id)
        if not tenant:
            return JSONResponse(
                {"error": "Invalid tenant"}, 
                status=403
            )
        
        # Check user belongs to this tenant (if authenticated)
        if user and user.tenant_id != tenant_id:
            return JSONResponse(
                {"error": "Access denied to this tenant"}, 
                status=403
            )
    
    # Step 3: Validate tenant is active
    if tenant.status != 'active':
        return JSONResponse(
            {"error": f"Tenant {tenant_id} is {tenant.status}"}, 
            status=403
        )
    
    # Step 4: Get database connection for tenant
    db_conn = await get_tenant_db_connection(tenant_id)
    if not db_conn:
        return JSONResponse(
            {"error": "Database connection failed"}, 
            status=500
        )
    
    # Step 5: Get Stripe keys for tenant
    stripe_keys = await get_tenant_stripe_keys(tenant_id)
    if not stripe_keys:
        return JSONResponse(
            {"error": "Stripe keys not configured"}, 
            status=500
        )
    
    # Step 6: Initialize Stripe client with tenant's keys
    stripe.api_key = stripe_keys['secret_key']
    
    # Step 7: Inject tenant context into request.state
    request.state.tenant = tenant
    request.state.tenant_id = tenant_id
    request.state.db = db_conn
    request.state.stripe_keys = stripe_keys
    request.state.is_superuser = is_superuser
    
    # Step 8: Continue to route handler
    response = await call_next(request)
    
    return response
```

### Helper Functions

**Get Tenant:**
```python
async def get_tenant(tenant_id: str) -> Optional[Tenant]:
    """Get tenant configuration from shared database."""
    shared_db = await shared_db_manager.get_connection()
    
    query = """
        SELECT tenant_id, name, domain, status, created_at, updated_at
        FROM tenants
        WHERE tenant_id = :tenant_id
    """
    
    result = await shared_db.fetch_one(query, {"tenant_id": tenant_id})
    
    if not result:
        return None
    
    return Tenant(
        tenant_id=result["tenant_id"],
        name=result["name"],
        domain=result["domain"],
        status=result["status"],
        created_at=result["created_at"],
        updated_at=result["updated_at"]
    )
```

**Request State:**
After middleware processing, `request.state` contains:
```python
request.state.tenant        # Tenant object
request.state.tenant_id       # Tenant identifier (string)
request.state.db            # Database connection for tenant
request.state.stripe_keys   # Stripe keys dictionary
request.state.is_superuser  # Boolean (True if superuser)
```

---

## Database Connection Manager

### Connection Pooling

**Per-Tenant Connection Pools:**
- Each tenant database has its own connection pool
- Pools are created lazily (on first request)
- Pools are reused across requests
- Configurable pool size per tenant

**Connection String Pattern:**
```
postgresql://user:password@host:5432/cogento_{tenant_id}
```

### DB Connection Implementation

```python
class TenantDBManager:
    """Manages database connections for multiple tenant databases."""
    
    def __init__(self):
        self._pools: Dict[str, Database] = {}
        self._pool_config = {
            "min_size": 2,
            "max_size": 10,
            "max_queries": 50000,
            "max_inactive_connection_lifetime": 300.0,
        }
    
    async def get_connection(self, tenant_id: str) -> Database:
        """Get database connection for tenant. Creates pool if needed."""
        db_name = f"cogento_{tenant_id}"
        
        if db_name not in self._pools:
            await self._create_pool(db_name)
        
        return self._pools[db_name]
    
    async def _create_pool(self, db_name: str):
        """Create connection pool for tenant database."""
        connection_string = (
            f"postgresql://{self._db_user}:{self._db_password}"
            f"@{self._db_host}:{self._db_port}/{db_name}"
        )
        
        database = Database(connection_string, **self._pool_config)
        await database.connect()
        self._pools[db_name] = database

# Global instance
tenant_db_manager = TenantDBManager()
```

**Shared Database Connection:**
```python
class SharedDBManager:
    """Manages connection to shared database (cogento_shared)."""
    
    async def get_connection(self) -> Database:
        """Get connection to shared database."""
        if self._database is None:
            connection_string = (
                f"postgresql://{self._db_user}:{self._db_password}"
                f"@{self._db_host}:{self._db_port}/cogento_shared"
            )
            self._database = Database(connection_string)
            await self._database.connect()
        return self._database

# Global instance
shared_db_manager = SharedDBManager()
```

---

## Stripe Key Service

### Stripe Key Service Design

The Stripe Key Service handles storing, retrieving, and managing encrypted Stripe API keys for each tenant.

```python
class StripeKeyService:
    """Service for managing encrypted Stripe keys per tenant."""
    
    def __init__(self):
        self._encryption_key = self._get_encryption_key()
        self._cipher = Fernet(self._encryption_key)
    
    async def store_keys(
        self,
        tenant_id: str,
        test_secret_key: str,
        test_publishable_key: str,
        live_secret_key: Optional[str] = None,
        live_publishable_key: Optional[str] = None,
        environment: str = "test",
        updated_by: Optional[str] = None
    ) -> bool:
        """Store encrypted Stripe keys for tenant."""
        # Encrypt secret keys
        test_secret_encrypted = self._encrypt(test_secret_key)
        live_secret_encrypted = (
            self._encrypt(live_secret_key) if live_secret_key else None
        )
        
        # Upsert into database
        # ... (implementation)
        return True
    
    async def get_keys(self, tenant_id: str) -> Optional[Dict[str, str]]:
        """Get and decrypt Stripe keys for tenant."""
        # Get from database
        # Decrypt secret key
        # Return dictionary with keys
        return {
            "secret_key": secret_key,
            "publishable_key": publishable_key,
            "environment": env
        }
    
    def _encrypt(self, plaintext: str) -> str:
        """Encrypt Stripe secret key."""
        encrypted = self._cipher.encrypt(plaintext.encode())
        return base64.b64encode(encrypted).decode()
    
    def _decrypt(self, encrypted: str) -> str:
        """Decrypt Stripe secret key."""
        encrypted_bytes = base64.b64decode(encrypted.encode())
        decrypted = self._cipher.decrypt(encrypted_bytes)
        return decrypted.decode()

# Global instance
stripe_key_service = StripeKeyService()
```

### Stripe Key Security

**Encryption:**
- Use Fernet (symmetric encryption) from `cryptography` library
- AES-256-GCM encryption
- Base64-encoded for storage

**Key Management:**
- Encryption key stored in environment variable
- Never commit encryption key to version control
- Rotate encryption key periodically
- Use key management service in production (AWS KMS, HashiCorp Vault, etc.)

**Access Control:**
- ✅ Superusers can store/update/delete keys
- ✅ Middleware can retrieve keys (for tenant requests)
- ❌ Regular users cannot access keys
- ❌ Keys never exposed in API responses

---

## Webhook Routing

### Webhook Routing Strategy

**Recommended: Separate Endpoints Per Tenant**

**Approach:**
- Each tenant has its own webhook endpoint
- URL pattern: `/webhooks/stripe/{tenant_id}`
- Stripe webhook URL configured per tenant: `https://api.cogento.com/webhooks/stripe/club1`

**Pros:**
- ✅ Clear tenant identification from URL
- ✅ Easy to configure in Stripe dashboard
- ✅ Simple routing logic
- ✅ Can have different webhook configurations per tenant

### Webhook Implementation

```python
@router.post("/webhooks/stripe/{tenant_id}")
async def stripe_webhook(
    tenant_id: str,
    request: Request,
    background_tasks: BackgroundTasks
):
    """Stripe webhook endpoint for tenant."""
    
    # Validate tenant exists
    tenant = await get_tenant(tenant_id)
    if not tenant:
        raise HTTPException(status_code=404, detail="Tenant not found")
    
    # Get tenant's Stripe keys
    stripe_keys = await get_tenant_stripe_keys(tenant_id)
    stripe.api_key = stripe_keys['secret_key']
    
    # Get webhook secret for tenant
    webhook_secret = await get_webhook_secret(tenant_id)
    
    # Verify webhook signature
    payload = await request.body()
    sig_header = request.headers.get('stripe-signature')
    
    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, webhook_secret
        )
    except ValueError:
        raise HTTPException(status_code=400, detail="Invalid payload")
    except stripe.error.SignatureVerificationError:
        raise HTTPException(status_code=400, detail="Invalid signature")
    
    # Process webhook asynchronously
    background_tasks.add_task(process_webhook_event, tenant_id, event)
    
    return {"status": "received"}
```

**Webhook Event Processing:**
```python
async def process_webhook_event(tenant_id: str, event: stripe.Event):
    """Process Stripe webhook event and sync to tenant's database."""
    event_type = event['type']
    event_data = event['data']['object']
    
    db = await get_tenant_db_connection(tenant_id)
    
    # Route event to appropriate handler
    if event_type == 'customer.created':
        await sync_customer_to_db(tenant_id, event_data, db)
    elif event_type == 'customer.updated':
        await sync_customer_to_db(tenant_id, event_data, db)
    # ... other event types
```

---

## Frontend Routing

### Path-Based Routing

**URL Structure:**
- Tenant routes: `/{tenant_id}/...`
- Examples:
  - `/club1/dashboard`
  - `/club1/members`
  - `/club2/dashboard`

**React Router Configuration:**
```typescript
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/:tenantId/*" element={<TenantRoutes />} />
        <Route path="/admin/*" element={<AdminRoutes />} />
      </Routes>
    </BrowserRouter>
  );
}
```

### API Client

**Automatic Tenant Injection:**
```typescript
export function createApiClient() {
  const { tenantId } = useClub();
  
  async function apiRequest(endpoint: string, options: RequestInit = {}) {
    // Inject tenant_id into endpoint path
    // Pattern: /stripe/customers -> /stripe/{tenantId}/customers
    const parts = endpoint.split('/').filter(p => p);
    if (parts.length >= 2) {
      parts.splice(1, 0, tenantId);
    }
    const url = `/api/${parts.join('/')}`;
    
    return fetch(url, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
        ...options.headers,
      },
    });
  }
  
  return {
    get: (endpoint: string) => apiRequest(endpoint, { method: 'GET' }),
    post: (endpoint: string, data?: any) => 
      apiRequest(endpoint, { method: 'POST', body: JSON.stringify(data) }),
    // ... other methods
  };
}
```

**Usage:**
```typescript
function Members() {
  const api = createApiClient();
  
  useEffect(() => {
    // API client automatically injects tenantId into path
    // '/postgres/users' becomes '/api/postgres/{tenantId}/users'
    api.get('/postgres/users')
      .then(res => res.json())
      .then(data => setMembers(data));
  }, []);
  
  return <div>...</div>;
}
```

---

## Migration Strategy

### Migration Approaches

**Approach 1: Apply to All Tenants (Recommended)**
```python
async def apply_migration_to_all_tenants(migration_sql: str, migration_name: str):
    """Apply migration to all active tenant databases."""
    tenants = await get_all_active_tenants()
    
    for tenant in tenants:
        db = await tenant_db_manager.get_connection(tenant.tenant_id)
        
        # Check if migration already applied
        if await is_migration_applied(db, migration_name):
            continue
        
        # Apply migration
        await db.execute(migration_sql)
        
        # Record migration
        await record_migration(db, migration_name)
```

**Approach 2: Apply to Specific Tenant**
```python
async def apply_migration_to_tenant(
    tenant_id: str, 
    migration_sql: str, 
    migration_name: str
):
    """Apply migration to specific tenant database."""
    db = await tenant_db_manager.get_connection(tenant_id)
    
    if await is_migration_applied(db, migration_name):
        return {"status": "skipped"}
    
    await db.execute(migration_sql)
    await record_migration(db, migration_name)
    
    return {"status": "success"}
```

### Migration Tracking

**Migration History Table (Per-Tenant):**
```sql
CREATE TABLE migration_history (
    migration_name VARCHAR(255) PRIMARY KEY,
    applied_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    applied_by VARCHAR(255)
);
```

**Helper Functions:**
```python
async def is_migration_applied(db: Database, migration_name: str) -> bool:
    """Check if migration already applied to database."""
    try:
        result = await db.fetch_val(
            "SELECT EXISTS(SELECT 1 FROM migration_history WHERE migration_name = :name)",
            {"name": migration_name}
        )
        return result
    except:
        return False  # Table doesn't exist yet

async def record_migration(db: Database, migration_name: str, applied_by: str = "system"):
    """Record that migration was applied."""
    await db.execute(
        "INSERT INTO migration_history (migration_name, applied_by) VALUES (:name, :by)",
        {"name": migration_name, "by": applied_by}
    )
```

**Migration Best Practices:**
1. **Idempotent Migrations** - Always use `IF NOT EXISTS` / `IF EXISTS`
2. **Transactional Migrations** - Wrap in `BEGIN; ... COMMIT;`
3. **Backward Compatible** - Add columns as nullable first
4. **Test on Single Tenant First** - Verify before applying to all

---

## Superuser Access

### Superuser Authentication

**Process:**
1. User requests sign-in with email
2. Check if email exists in `cogento_shared.users` table with `role = 'superuser'`
3. If superuser, generate token and allow access
4. Superuser can then access any tenant via path (e.g., `/stripe/club1/customers`)

**Implementation:**
```python
async def authenticate_user(email: str) -> Dict:
    """Authenticate user and determine if superuser."""
    shared_db = await shared_db_manager.get_connection()
    user = await shared_db.fetch_one(
        "SELECT * FROM users WHERE email = :email AND is_active = TRUE AND role = 'superuser'",
        {"email": email}
    )
    
    if user:
        return {
            "user_id": user["user_id"],
            "email": email,
            "role": user["role"],
            "is_superuser": True,
            "tenant_id": None
        }
    else:
        # Regular user - find their tenant
        tenant_id = await find_user_tenant(email)
        return {
            "email": email,
            "is_superuser": False,
            "tenant_id": tenant_id
        }
```

### Superuser API Endpoints

**Tenant Management:**
```python
@router.get("/admin/tenants")
async def list_tenants(request: Request):
    """List all tenants (superuser only)."""
    if not request.state.is_superuser:
        raise HTTPException(status_code=403)
    
    shared_db = await shared_db_manager.get_connection()
    tenants = await shared_db.fetch_all("SELECT * FROM clubs")
    return [dict(tenant) for tenant in tenants]

@router.post("/admin/tenants")
async def create_tenant(request: Request, tenant_data: CreateTenantRequest):
    """Create new tenant (superuser only)."""
    if not request.state.is_superuser:
        raise HTTPException(status_code=403)
    
    # Create tenant in shared database
    # Create tenant database
    # Apply initial migrations
    return {"status": "success", "tenant_id": tenant_data.tenant_id}
```

**Superuser Operations:**
- Superuser can access any tenant via path (e.g., `/stripe/club1/customers`, `/stripe/club2/customers`)
- Superuser can perform admin operations (no tenant in path): `/admin/tenants`
- Superuser can manage tenant configuration, create/delete tenants

---

## Database Recovery

**Per-Tenant Database Recovery:**
- ✅ **No backup needed** - Tenant databases are read replicas from Stripe
- ✅ **Regeneration from Stripe** - If tenant database is lost/corrupted:

  1. Get tenant's Stripe keys from shared database
  2. Connect to tenant's Stripe account
  3. Read all Customers and Subscriptions from Stripe
  4. Recreate tenant database and populate from Stripe data
  5. Webhooks will continue syncing going forward

**Recovery Script:**
```python
def regenerate_tenant_database(tenant_id):
    # Get tenant Stripe keys
    stripe_keys = get_tenant_stripe_keys(tenant_id)
    
    # Initialize Stripe with tenant's keys
    stripe.api_key = stripe_keys['secret_key']
    
    # Recreate database
    create_tenant_database(tenant_id)
    
    # Read all data from Stripe
    customers = stripe.Customer.list(limit=100)
    subscriptions = stripe.Subscription.list(limit=100)
    
    # Populate database
    sync_all_customers_to_db(tenant_id, customers)
    sync_all_subscriptions_to_db(tenant_id, subscriptions)
```

---

## Monitoring

**Per-Tenant Monitoring:**
- ✅ **Monitor each tenant database separately** - Track performance, queries, connections per tenant
- ✅ **Per-tenant metrics** - Database size, query performance, connection pool usage
- ✅ **Per-tenant logging** - Logs tagged with tenant/tenant_id for filtering
- ✅ **Per-tenant alerts** - Alert on tenant-specific issues
- ✅ **Dashboard per tenant** - Superuser can view metrics for any tenant

**Monitoring Metrics:**
- Database size per tenant
- Query performance per tenant
- Connection pool usage per tenant
- Webhook sync status per tenant
- Stripe API usage per tenant
- Error rates per tenant

---

## Summary

**Multi-Tenancy Architecture:**
- ✅ Path-based routing for UI (`/club1/`, `/club2/`)
- ✅ Path-based routing for API (`/stripe/club1/customers`, `/postgres/club1/users`)
- ✅ Separate PostgreSQL database per tenant
- ✅ Separate Stripe account per tenant
- ✅ Superuser can manage all tenants
- ✅ No cross-tenant operations
- ✅ No backup needed - regenerate from Stripe
- ✅ Per-tenant monitoring required

**Key Design Decisions:**
- One PostgreSQL server instance with multiple databases (one per tenant)
- Shared database (`cogento_shared`) for tenant configuration and Stripe keys
- Path-based API routing - tenant context from URL path (consistent with UI)
- Complete data isolation per tenant
- Schema flexibility - different columns per tenant if needed

**Benefits:**
1. **Complete Data Isolation** - Each tenant's data in separate database
2. **Schema Flexibility** - Different columns per tenant if needed
3. **Stripe Isolation** - Each tenant uses own Stripe account
4. **Scalability** - Can move tenants to separate servers later
5. **Security** - Strong isolation between tenants
6. **Consistency** - Path-based routing across UI and API
