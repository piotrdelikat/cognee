# Coolify Deployment Troubleshooting

## Issue: "No server available" / Database Connection Failed

### Symptoms
- Website shows "no server available"
- Logs show: `socket.gaierror: [Errno -2] Name or service not known`
- Postgres logs: `FATAL: database "cognee" does not exist`

### Root Causes
1. **Service startup race condition**: Cognee starts before Postgres is ready
2. **Database name mismatch**: Wrong database name in environment variables
3. **Network isolation**: Containers can't find each other

## Solutions

### 1. Fix Database Name (Critical)

**Check your Coolify environment variables - these MUST match:**

```env
# Correct (both must be cognee):
DB_NAME=cognee
POSTGRES_DB=cognee

# Incorrect (will cause "database does not exist" error):
DB_NAME=cognee
POSTGRES_DB=cognee_db  # Mismatch!
```

**Action:** In Coolify UI:
1. Go to your deployment
2. Click "Environment Variables"
3. Find `DB_NAME` and `POSTGRES_DB`
4. Make sure **both** are set to `cognee`
5. Save and redeploy

### 2. Use the Correct Docker Compose File

The updated `docker-compose.yml` includes healthchecks that ensure Postgres is ready before Cognee starts.

**In Coolify:**
1. Make sure you're pulling the latest code from your fork
2. The compose file should have these lines:
```yaml
cognee:
  depends_on:
    postgres:
      condition: service_healthy  # ← This is critical!
```

### 3. Verify Environment Variables

**Minimum required environment variables in Coolify:**

```env
# Database Connection (what the app uses)
DB_PROVIDER=postgres
DB_HOST=postgres
DB_PORT=5432
DB_NAME=cognee
DB_USERNAME=cognee
DB_PASSWORD=your_secure_password

# Postgres Configuration (what postgres creates)
POSTGRES_USER=cognee
POSTGRES_PASSWORD=your_secure_password
POSTGRES_DB=cognee

# Port Configuration (to avoid conflicts)
COGNEE_PORT=8080
POSTGRES_EXTERNAL_PORT=5433
```

**Note:** The passwords should match:
- `DB_PASSWORD` = `POSTGRES_PASSWORD`

### 4. Check Container Logs in Coolify

**Postgres logs should show:**
```
PostgreSQL init process complete; ready for start up.
database system is ready to accept connections
```

**NOT this:**
```
FATAL: database "cognee" does not exist
```

**Cognee logs should show:**
```
INFO:     Started server process
INFO:     Waiting for application startup.
```

**NOT this:**
```
socket.gaierror: [Errno -2] Name or service not known
```

### 5. Force Restart All Containers

Sometimes Coolify needs a full restart:

1. **In Coolify UI:**
   - Click "Stop"
   - Wait for all containers to stop
   - Click "Deploy" (not just restart)

2. **This ensures:**
   - Postgres starts first
   - Postgres healthcheck passes
   - Cognee waits and then connects successfully

### 6. Check Network Configuration

If DNS resolution still fails, check that all services are on the same network:

```yaml
# All services should have:
networks:
  - cognee-network

# At the bottom:
networks:
  cognee-network:
    name: cognee-network
```

This should already be correct in the updated compose file.

## Verification Steps

### Step 1: Check Postgres is Running
In Coolify, view Postgres logs:
```
✅ Good: "database system is ready to accept connections"
❌ Bad: "FATAL: database does not exist"
```

### Step 2: Check Database Name
Postgres should create database `cognee`:
```sql
-- You can exec into postgres container and check:
\l
-- Should list: cognee
```

### Step 3: Check Cognee Startup
Cognee logs should show:
```
INFO: Running alembic migrations...
INFO: Successfully connected to database
INFO: Started server process
```

### Step 4: Test the API
Once deployed, test the endpoint:
```bash
curl http://your-domain:8080/health
# or whatever port you configured
```

## Common Mistakes

### ❌ Wrong Database Names
```env
DB_NAME=cognee_db       # Wrong - old default
POSTGRES_DB=cognee      # Mismatch!
```

### ✅ Correct Configuration
```env
DB_NAME=cognee          # Correct
POSTGRES_DB=cognee      # Match!
```

### ❌ Missing Healthcheck
```yaml
depends_on:
  - postgres  # Wrong - doesn't wait for postgres to be ready
```

### ✅ With Healthcheck
```yaml
depends_on:
  postgres:
    condition: service_healthy  # Correct - waits for postgres
```

### ❌ Wrong Database Host
```env
DB_HOST=localhost       # Won't work in containers
DB_HOST=127.0.0.1      # Won't work in containers
```

### ✅ Correct Database Host
```env
DB_HOST=postgres        # Correct - uses Docker DNS
```

## Still Having Issues?

### Check Container Status
In Coolify, check that all containers are running:
- ✅ postgres: healthy
- ✅ cognee: running
- ❌ If cognee keeps restarting, check logs

### View Full Logs
1. Click on the cognee container
2. View full logs (not just recent)
3. Look for the first error message
4. Check if it's before or after database connection attempt

### Manual Database Check
If you need to manually verify the database:

1. **Exec into postgres container** in Coolify
2. **Connect to postgres:**
   ```bash
   psql -U cognee -d cognee
   ```
3. **Check database exists:**
   ```sql
   \l  -- list databases
   ```

If `cognee` doesn't exist, you have a setup issue.

## Quick Fix Checklist

- [ ] Both `DB_NAME` and `POSTGRES_DB` are set to `cognee`
- [ ] Passwords match: `DB_PASSWORD` = `POSTGRES_PASSWORD`
- [ ] Using latest code with healthcheck conditions
- [ ] All environment variables are set in Coolify
- [ ] Performed a full stop + deploy (not just restart)
- [ ] Postgres logs show "ready to accept connections"
- [ ] Cognee logs don't show DNS resolution errors

## Need More Help?

Check the full logs for:
1. **First error that appears** (often the root cause)
2. **Postgres initialization** (should create cognee)
3. **Network resolution** (cognee should find postgres)
4. **Migration errors** (might indicate connection issues)

If all else fails, you can connect to an external postgres database by setting:
```env
DB_HOST=your-external-postgres-host
POSTGRES_EXTERNAL_PORT=  # Disable local postgres
```

But fixing the local postgres is recommended for production deployments.
