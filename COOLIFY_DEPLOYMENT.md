# Coolify Deployment Guide

## Port Conflict Resolution

### Problem
```
Error: Bind for 0.0.0.0:5432 failed: port is already allocated
```

This happens when your Coolify server already has a service using port 5432 (usually another Postgres instance).

### Solution Options

#### Option 1: Use a Different Port (Recommended)

In Coolify, add this environment variable:

```env
POSTGRES_EXTERNAL_PORT=5433
```

This will map the external port to 5433 instead of 5432. The internal services will still connect via `postgres:5432` on the Docker network.

#### Option 2: Disable External Port Exposure (Most Secure)

If you don't need external database access, remove port exposure entirely:

In Coolify, set:
```env
POSTGRES_EXTERNAL_PORT=
```

Or comment out the ports section in docker-compose.yml:
```yaml
postgres:
  # ports:
  #   - "${POSTGRES_EXTERNAL_PORT:-5432}:5432"
```

**Note:** Services inside the Docker network will still communicate fine via `postgres:5432`.

## Complete Coolify Environment Variables

Set these in your Coolify project's environment variables section:

```env
# Database Configuration
DB_PROVIDER=postgres
DB_HOST=postgres
DB_PORT=5432
DB_NAME=cognee_db
DB_USERNAME=cognee
DB_PASSWORD=your_secure_password_here

# Postgres Configuration
POSTGRES_USER=cognee
POSTGRES_PASSWORD=your_secure_password_here
POSTGRES_DB=cognee_db
POSTGRES_EXTERNAL_PORT=5433  # Change this if 5432 is taken, or leave empty to disable

# Optional: Change other service ports if needed
# (Currently cognee=8000, cognee-mcp=8001, frontend=3000)
```

## Service Architecture

All services communicate via the internal `cognee-network`:

```
┌─────────────────────────────────────────┐
│         Internal Network                │
│                                         │
│  ┌─────────┐      ┌──────────┐        │
│  │ cognee  │─────▶│ postgres │        │
│  │  :8000  │      │  :5432   │        │
│  └─────────┘      └──────────┘        │
│       │                                │
│       │           ┌──────────────┐    │
│       └──────────▶│ cognee-mcp   │    │
│                   │    :8000     │    │
│                   └──────────────┘    │
└─────────────────────────────────────────┘
         │              │
         ▼              ▼
    External       External
    Port 8000      Port 8001
```

**Key Point:** Services use internal DNS names (`postgres`, `cognee`, etc.) and don't need external ports to communicate with each other.

## Port Mapping Reference

| Service | Internal Port | Default External | Configurable Via |
|---------|--------------|------------------|------------------|
| cognee | 8000 | 8000 | Hardcoded |
| cognee-mcp | 8000 | 8001 | Hardcoded |
| frontend | 3000 | 3000 | Hardcoded |
| postgres | 5432 | 5432 | `POSTGRES_EXTERNAL_PORT` |
| neo4j | 7474, 7687 | 7474, 7687 | Hardcoded (profile) |
| chromadb | 8000 | 3002 | Hardcoded (profile) |
| redis | 6379 | 6379 | Hardcoded (profile) |

## Deployment Steps in Coolify

1. **Create New Resource**
   - Select "Docker Compose" type
   - Connect to your forked repository
   - Select the main branch

2. **Set Environment Variables**
   ```env
   DB_PROVIDER=postgres
   DB_HOST=postgres
   DB_PORT=5432
   DB_NAME=cognee_db
   DB_USERNAME=cognee
   DB_PASSWORD=secure_password_123
   POSTGRES_USER=cognee
   POSTGRES_PASSWORD=secure_password_123
   POSTGRES_DB=cognee_db
   POSTGRES_EXTERNAL_PORT=5433
   ```

3. **Deploy**
   - Click "Deploy"
   - Watch logs for any errors
   - Postgres will start automatically (no profile needed)

4. **Access Your Services**
   - Main API: `https://your-domain.com:8000`
   - MCP Server (if profile enabled): `https://your-domain.com:8001`
   - Frontend (if profile enabled): `https://your-domain.com:3000`

## Troubleshooting

### Still Getting Port Conflicts?

Check what's using the port:
```bash
sudo netstat -tlnp | grep 5432
```

Or in Coolify, try these ports instead:
```env
POSTGRES_EXTERNAL_PORT=5433
POSTGRES_EXTERNAL_PORT=5434
POSTGRES_EXTERNAL_PORT=54320
```

### Database Connection Issues?

Verify services are using the correct host:
- ✅ `DB_HOST=postgres` (internal DNS)
- ❌ `DB_HOST=localhost` (won't work in containers)
- ❌ `DB_HOST=127.0.0.1` (won't work in containers)

### Need to Enable Optional Services?

In Coolify, you can't use profiles directly. Instead:
1. Remove the `profiles:` section from services you want to run
2. Or use `COMPOSE_PROFILES` environment variable (less common in Coolify)

## Security Best Practices

1. **Use Strong Passwords**
   ```env
   POSTGRES_PASSWORD=$(openssl rand -base64 32)
   DB_PASSWORD=$(openssl rand -base64 32)
   ```

2. **Disable External Postgres Access in Production**
   ```env
   POSTGRES_EXTERNAL_PORT=  # Empty = no external access
   ```

3. **Use Coolify's Built-in Secrets**
   - Store passwords in Coolify's secret management
   - Reference them with `${{ secrets.POSTGRES_PASSWORD }}`

4. **Enable HTTPS**
   - Configure SSL/TLS in Coolify
   - Use Let's Encrypt for automatic certificates

## Monitoring

Check service health:
```bash
docker ps  # See running containers
docker logs postgres  # View postgres logs
docker logs cognee  # View cognee logs
```

Or use Coolify's built-in logs viewer in the UI.
