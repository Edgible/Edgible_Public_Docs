# Encrypting and Storing Stripe Keys

This guide explains how to encrypt and store Stripe API keys for your tenants.

## Prerequisites

1. **Encryption Key**: You must have `STRIPE_KEY_ENCRYPTION_KEY` set in your environment or `.env` file.
   - Generate one with: `python generate_key.py` (or via Docker)
   - Add to `.env`: `STRIPE_KEY_ENCRYPTION_KEY=your_key_here`

2. **Tenant Exists**: The tenant must already exist in the `tenants` table in `cogento_shared` database.

3. **Stripe Keys**: You need your Stripe API keys:
   - Test Secret Key: `sk_test_...`
   - Test Publishable Key: `pk_test_...`
   - (Optional) Live Secret Key: `sk_live_...`
   - (Optional) Live Publishable Key: `pk_live_...`

## Usage

### Interactive Mode (Recommended)

Run the script without arguments and it will prompt you for all information:

```bash
# Using Docker (recommended):
docker compose run --rm cogento-api python encrypt_stripe_keys.py

# Or locally:
cd api
python encrypt_stripe_keys.py
```

The script will ask for:
1. Tenant ID (e.g., `kgtc`)
2. Test Secret Key
3. Test Publishable Key
4. Live Secret Key (optional)
5. Live Publishable Key (optional)
6. Environment to use (`test` or `live`)

### Command Line Mode

You can also provide all arguments on the command line:

```bash
docker compose run --rm cogento-api python encrypt_stripe_keys.py \
  kgtc \
  sk_test_51xxx \
  pk_test_51xxx \
  sk_live_51xxx \
  pk_live_51xxx \
  test
```

Arguments:
1. `tenant_id` - Tenant identifier (required)
2. `test_secret_key` - Stripe test secret key (required)
3. `test_publishable_key` - Stripe test publishable key (required)
4. `live_secret_key` - Stripe live secret key (optional)
5. `live_publishable_key` - Stripe live publishable key (optional)
6. `environment` - Environment to use: `test` or `live` (default: `test`)

## Example

```bash
# Interactive mode
$ docker compose run --rm cogento-api python encrypt_stripe_keys.py
Stripe Key Encryption Tool
========================================

Enter tenant ID (e.g., 'kgtc'): kgtc

Test Keys (required):
  Test Secret Key (sk_test_...): sk_test_51xxx...
  Test Publishable Key (pk_test_...): pk_test_51xxx...

Live Keys (optional):
  Live Secret Key (sk_live_...) [optional]: 
  Live Publishable Key (pk_live_...) [optional]: 

Environment to use (test/live) [default: test]: test

Encrypting and storing keys...
Connected to database: cogento_shared

✅ Successfully stored encrypted Stripe keys for tenant 'kgtc'
   Environment: test
   Test keys: ✓
   Live keys: Not provided
```

## Security Notes

- **Never commit Stripe keys to version control**
- **Use environment variables** for the encryption key
- **Encrypted keys are stored** in the database (not plain text)
- **Each tenant** has separate encrypted keys
- **Rotate keys** if compromised

## Troubleshooting

### "STRIPE_KEY_ENCRYPTION_KEY not set"
- Generate a key: `python generate_key.py`
- Add to `.env` file or export as environment variable

### "Tenant does not exist"
- Create the tenant first in the `tenants` table
- Check tenant ID spelling

### "Failed to encrypt keys"
- Verify your encryption key is valid
- Regenerate encryption key if needed

### "Database connection failed"
- Ensure PostgreSQL is running: `docker compose up cogento-postgres`
- Check database credentials in `.env`

## Verifying Keys

After storing keys, you can verify they work by:

1. Starting the API: `docker compose up cogento-api`
2. Testing an endpoint: `curl http://localhost:8000/stripe/kgtc/customers`
3. If keys are correct, you'll get a response (even if empty list)
4. If keys are wrong, you'll get a Stripe authentication error
