# Docker Setup for Cogento

## Overview

Cogento uses Docker Compose to manage services. The compose project is named `cogento-services`.

## Services

### PostgreSQL (`cogento-postgres`)

PostgreSQL database service for Cogento.

**Configuration:**
- Image: `postgres:15-alpine`
- Container name: `cogento-postgres`
- Port: `5432` (configurable via `POSTGRES_PORT` env var)
- User: `cogento`
- Password: Set via `POSTGRES_PASSWORD` env var

**Databases:**
- `cogento_shared` - Shared database for tenant configuration, Stripe keys, and users (with role = 'superuser')
- `cogento_kgtc` - First tenant database (KGTC)

**Volumes:**
- `./volumes/postgres/data` - PostgreSQL data directory
- `./volumes/postgres/init` - Initialization scripts (runs on first startup)

### pgAdmin (`cogento-pgadmin`)

Web-based PostgreSQL administration tool for managing databases, running SQL queries, and viewing data.

**Configuration:**
- Image: `dpage/pgadmin4:latest`
- Container name: `cogento-pgadmin`
- Port: `5050` (configurable via `PGADMIN_PORT` env var)
- Email: Set via `PGADMIN_EMAIL` env var (default: `admin@cogento.com`)
- Password: Set via `PGADMIN_PASSWORD` env var (default: `admin`)

**Access:**
- Web UI: `http://localhost:5050`
- Login with email and password from environment variables

**Volumes:**
- `./volumes/pgadmin/data` - pgAdmin data directory (saved server connections, etc.)

## Getting Started

### 1. Create Environment File

Copy the example environment file:

```bash
cp .env.example .env
```

Edit `.env` and set your PostgreSQL password, pgAdmin credentials, and other configuration:

```bash
POSTGRES_PASSWORD=your-secure-password
PGADMIN_EMAIL=admin@cogento.com
PGADMIN_PASSWORD=your-secure-password
```

### 2. Start Services

```bash
docker-compose up -d
```

This will:
- Start the PostgreSQL container
- Start the pgAdmin container
- Create the `cogento_shared` and `cogento_kgtc` databases
- Set up data volumes

### 3. Verify Services

Check that PostgreSQL is running:

```bash
docker-compose ps
```

Check PostgreSQL logs:

```bash
docker-compose logs cogento-postgres
```

### 4. Access pgAdmin

Open your browser and navigate to:

```
http://localhost:5050
```

Login with:
- Email: `admin@cogento.com` (or your `PGADMIN_EMAIL` value)
- Password: `admin` (or your `PGADMIN_PASSWORD` value)

### 5. Add PostgreSQL Server in pgAdmin

After logging into pgAdmin:

1. Right-click on "Servers" in the left sidebar
2. Select "Register" → "Server"
3. In the "General" tab:
   - Name: `Cogento PostgreSQL`
4. In the "Connection" tab:
   - Host name/address: `cogento-postgres`
   - Port: `5432`
   - Maintenance database: `postgres`
   - Username: `cogento`
   - Password: Your `POSTGRES_PASSWORD` value
5. Click "Save"

You can now browse databases, run SQL queries, and manage your PostgreSQL instance.

### 6. Connect to Database via Command Line (Alternative)

You can also connect to PostgreSQL using:

```bash
# Connect to shared database
docker-compose exec cogento-postgres psql -U cogento -d cogento_shared

# Connect to tenant database
docker-compose exec cogento-postgres psql -U cogento -d cogento_kgtc
```

Or from your host machine (if port is exposed):

```bash
psql -h localhost -p 5432 -U cogento -d cogento_shared
```

## Volume Structure

```
volumes/
├── postgres/
│   ├── data/          # PostgreSQL data files (created automatically)
│   └── init/          # Initialization scripts
│       └── 01_create_databases.sql
└── pgadmin/
    └── data/          # pgAdmin data (saved connections, etc.)
```

**Important:** The `volumes/postgres/data` directory contains all database data. Do not delete this directory unless you want to lose all data.

## Database Initialization

The initialization script `volumes/postgres/init/01_create_databases.sql` runs automatically on first container startup. It creates:

1. `cogento_shared` - Shared database
2. `cogento_kgtc` - First tenant database

Additional tenant databases will be created via the API when new tenants are added.

## Stopping Services

```bash
# Stop services (keeps data)
docker-compose stop

# Stop and remove containers (keeps data)
docker-compose down

# Stop and remove containers and volumes (DELETES DATA)
docker-compose down -v
```

## Backup and Restore

### Backup

```bash
# Backup shared database
docker-compose exec cogento-postgres pg_dump -U cogento cogento_shared > backup_shared.sql

# Backup tenant database
docker-compose exec cogento-postgres pg_dump -U cogento cogento_kgtc > backup_kgtc.sql
```

### Restore

```bash
# Restore shared database
docker-compose exec -T cogento-postgres psql -U cogento cogento_shared < backup_shared.sql

# Restore tenant database
docker-compose exec -T cogento-postgres psql -U cogento cogento_kgtc < backup_kgtc.sql
```

**Note:** Since Stripe is the source of truth, tenant databases can be regenerated from Stripe. Backups are optional for tenant databases but recommended for the shared database.

## Troubleshooting

### PostgreSQL won't start

Check logs:

```bash
docker-compose logs cogento-postgres
```

Common issues:
- Port already in use: Change `POSTGRES_PORT` in `.env`
- Permission issues: Ensure `volumes/postgres/data` directory is writable
- Data corruption: Remove `volumes/postgres/data` and restart (WILL DELETE DATA)

### Reset Database

To completely reset PostgreSQL (DELETES ALL DATA):

```bash
docker-compose down -v
rm -rf volumes/postgres/data
docker-compose up -d
```

### Check Database Status

```bash
# List all databases
docker-compose exec cogento-postgres psql -U cogento -c "\l"

# Check database size
docker-compose exec cogento-postgres psql -U cogento -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database WHERE datname LIKE 'cogento%';"
```

## Environment Variables

See `.env.example` for all available environment variables.

**Required:**
- `POSTGRES_PASSWORD` - PostgreSQL password

**Optional:**
- `POSTGRES_PORT` - PostgreSQL port (default: 5432)
- `PGADMIN_EMAIL` - pgAdmin login email (default: `admin@cogento.com`)
- `PGADMIN_PASSWORD` - pgAdmin login password (default: `admin`)
- `PGADMIN_PORT` - pgAdmin web UI port (default: 5050)

## Next Steps

After PostgreSQL is running:

1. Run migrations for `cogento_shared` database
2. Run migrations for `cogento_kgtc` database
3. Add API and UI services to docker-compose.yml
4. Configure Stripe keys for tenant
