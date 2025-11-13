# Coolify Deployment Guide

## Port Conflict Resolution

### Problem
Port conflicts occur when your Coolify server already has services using the same ports:

```
Error: Bind for 0.0.0.0:8000 failed: port is already allocated
Error: Bind for 0.0.0.0:5432 failed: port is already allocated
```

### Solution: Configure Alternative Ports

All service ports are now configurable via environment variables. Add these in Coolify:

```env
# Main API (default: 8000) - REQUIRED, this is your main service
COGNEE_PORT=8080

# MCP Server (default: 8001) - only if using MCP profile
COGNEE_MCP_PORT=8081

# Frontend (default: 3000) - only if using UI profile
FRONTEND_PORT=3001

# Postgres (default: 5432)
POSTGRES_EXTERNAL_PORT=5433

# Debug ports (usually not needed in production)
COGNEE_DEBUG_PORT=5678
COGNEE_MCP_DEBUG_PORT=5679
```

**Important:** Only set the ports you actually need. Internal communication between services happens via Docker network names (`postgres:5432`, etc.) and doesn't depend on external ports.

## Complete Coolify Environment Variables

Set these in your Coolify project's environment variables section:

```env
# Database Configuration
DB_PROVIDER=postgres
DB_HOST=postgres
DB_PORT=5432
DB_NAME=cognee              # ⚠️ CRITICAL: Must match POSTGRES_DB
DB_USERNAME=cognee
DB_PASSWORD=your_secure_password_here

# Postgres Configuration
POSTGRES_USER=cognee
POSTGRES_PASSWORD=your_secure_password_here  # ⚠️ Must match DB_PASSWORD
POSTGRES_DB=cognee          # ⚠️ CRITICAL: Must match DB_NAME

# Port Configuration (change if defaults conflict with existing services)
COGNEE_PORT=8080              # Main API (change if 8000 conflicts)
POSTGRES_EXTERNAL_PORT=5433   # Change if 5432 conflicts
COGNEE_MCP_PORT=8081          # MCP Server (if using mcp profile)
FRONTEND_PORT=3001            # Frontend (if using ui profile)

# Debug Ports (optional, usually not needed in production)
COGNEE_DEBUG_PORT=5678
COGNEE_MCP_DEBUG_PORT=5679
```

**⚠️ Critical Requirements:**
- `DB_NAME` and `POSTGRES_DB` **MUST be identical** (both `cognee`)
- `DB_PASSWORD` and `POSTGRES_PASSWORD` **MUST be identical**
- If these don't match, you'll get "database does not exist" errors

**📖 Having issues?** See [COOLIFY_TROUBLESHOOTING.md](./COOLIFY_TROUBLESHOOTING.md) for detailed solutions.

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
| cognee | 8000 | 8000 | `COGNEE_PORT` |
| cognee-mcp | 8000 | 8001 | `COGNEE_MCP_PORT` |
| frontend | 3000 | 3000 | `FRONTEND_PORT` |
| postgres | 5432 | 5432 | `POSTGRES_EXTERNAL_PORT` |
| cognee-debug | 5678 | 5678 | `COGNEE_DEBUG_PORT` |
| mcp-debug | 5678 | 5679 | `COGNEE_MCP_DEBUG_PORT` |
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
   # Database
   DB_PROVIDER=postgres
   DB_HOST=postgres
   DB_PORT=5432
   DB_NAME=cognee
   DB_USERNAME=cognee
   DB_PASSWORD=secure_password_123
   
   # Postgres
   POSTGRES_USER=cognee
   POSTGRES_PASSWORD=secure_password_123
   POSTGRES_DB=cognee
   POSTGRES_EXTERNAL_PORT=5433
   
   # Service Ports (adjust based on what's available on your server)
   COGNEE_PORT=8080           # Change from default 8000
   COGNEE_MCP_PORT=8081      # If using MCP
   FRONTEND_PORT=3001        # If using UI
   ```

3. **Deploy**
   - Click "Deploy"
   - Watch logs for any port conflict errors
   - If you still see conflicts, adjust the port numbers above
   - Postgres will start automatically (no profile needed)

4. **Access Your Services**
   - Main API: `https://your-domain.com:8080` (or whatever you set `COGNEE_PORT` to)
   - MCP Server (if profile enabled): `https://your-domain.com:8081`
   - Frontend (if profile enabled): `https://your-domain.com:3001`

## Troubleshooting

### Still Getting Port Conflicts?

Check what's using the port on your server:
```bash
sudo netstat -tlnp | grep 8000  # Check main API port
sudo netstat -tlnp | grep 5432  # Check postgres port
sudo lsof -i :8000              # Alternative command
```

Try different port numbers until you find available ones:
```env
# Example: If 8000 and 5432 are taken, try:
COGNEE_PORT=8080
COGNEE_PORT=8888
COGNEE_PORT=9000

POSTGRES_EXTERNAL_PORT=5433
POSTGRES_EXTERNAL_PORT=5434
POSTGRES_EXTERNAL_PORT=54320
```

**Common Coolify conflicts:**
- Port 8000: Often used by other APIs
- Port 5432: Usually taken by system Postgres
- Port 3000: Commonly used by Node.js apps

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
