# Troubleshooting Guide

## Common Issues

### "Failed to decrypt Stripe key" Error

**Error Message:**
```
Failed to decrypt Stripe key for tenant 'kgtc'. 
This usually means STRIPE_KEY_ENCRYPTION_KEY doesn't match the key used to encrypt the keys.
```

**Cause:**
The `STRIPE_KEY_ENCRYPTION_KEY` environment variable doesn't match the encryption key that was used when the Stripe keys were encrypted and stored in the database.

**Solution:**

1. **Find the correct encryption key:**
   - Check your `.env` file or environment variables
   - The key should be the same one generated when you first set up encryption
   - Example: `STRIPE_KEY_ENCRYPTION_KEY=0tJ26o1IXT76Zi6TQUbHpL56Ex-bWs6hpSmQO60XLxE=`

2. **Set the encryption key:**
   ```bash
   # In your .env file or environment
   export STRIPE_KEY_ENCRYPTION_KEY=your_encryption_key_here
   ```

3. **Or re-encrypt the Stripe keys:**
   If you don't have the original key, you'll need to re-encrypt the Stripe keys:
   ```bash
   # Generate a new encryption key
   docker compose run --rm cogento-api python generate_key.py
   
   # Update your .env file with the new key
   # Then re-encrypt your Stripe keys
   docker compose run --rm cogento-api python encrypt_stripe_keys.py kgtc sk_test_xxx pk_test_xxx
   ```

4. **Restart the API:**
   ```bash
   docker compose restart cogento-api
   ```

### "Module not found" Errors

**Solution:**
- Ensure dependencies are installed: `pip install -r api/requirements.txt`
- For Docker: Rebuild the container: `docker compose build cogento-api`

### "Tenant not found" Error

**Solution:**
- Ensure the tenant exists in the `cogento_shared.tenants` table
- Check tenant ID spelling (case-sensitive)

### "Database connection failed"

**Solution:**
- Ensure PostgreSQL is running: `docker compose up cogento-postgres`
- Check database credentials in `.env` or environment variables
- Verify `POSTGRES_HOST` is correct:
  - Inside Docker: `cogento-postgres`
  - Outside Docker: `localhost`

### "CSV file not found" (for tools)

**Solution:**
- Use path relative to Cogento root: `tenants/kgtc/Customers.csv`
- Or use absolute path: `/full/path/to/Customers.csv`

### "Form data requires python-multipart"

**Solution:**
- Install `python-multipart`: `pip install python-multipart`
- Or rebuild Docker container: `docker compose build cogento-api`
