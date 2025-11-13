# 🚀 Deployment Quick Start

Choose your deployment method based on your needs:

## 🏠 Local Development

**Use**: `docker-compose.yml`

```bash
docker-compose up -d
```

**When to use**: Learning, testing, single deployment  
**Build time**: 10-20 minutes

---

## ☁️ Production Deployment (Recommended for Coolify)

**Use**: `docker-compose.production.yml` + GitHub Actions

### One-Time Setup (5 minutes)

1. **Push to GitHub** (triggers automatic image builds)
```bash
git add .
git commit -m "Setup production deployment"
git push origin main
```

2. **Make images public** (optional, recommended)
   - Go to your GitHub profile → Packages
   - Find: `cognee`, `cognee-mcp`, `cognee-frontend`
   - Package Settings → Change visibility → Public

3. **Configure environment**
```bash
cp .env.production.example .env
nano .env  # Add your GITHUB_USERNAME and passwords
```

### Deploy to Coolify

1. **In Coolify**: Create new Docker Compose project
2. **Point to your repository**
3. **Set compose file**: `docker-compose.production.yml`
4. **Add environment variables**:
```env
GITHUB_USERNAME=your-github-username
IMAGE_TAG=latest
POSTGRES_PASSWORD=your_secure_password
DB_PASSWORD=your_secure_password
```
5. **Deploy!** ⚡ (1-2 minutes)

---

## 📊 Comparison

| Aspect | Development | Production |
|--------|-------------|------------|
| **File** | `docker-compose.yml` | `docker-compose.production.yml` |
| **Build time** | 10-20 min | 1-2 min |
| **Server load** | High | Low |
| **Best for** | Local dev | Coolify/Cloud |

---

## 📚 Detailed Guides

- **[Full Production Guide](./PRODUCTION_DEPLOYMENT.md)** - Complete setup instructions
- **[Deployment Comparison](./DEPLOYMENT_COMPARISON.md)** - Detailed pros/cons analysis

---

## 🆘 Quick Troubleshooting

### "Image not found" error
```bash
# Check your GITHUB_USERNAME is correct in .env
# OR make images public in GitHub
```

### Want to update deployment
```bash
# Coolify will auto-detect new images, or manually:
docker-compose -f docker-compose.production.yml pull
docker-compose -f docker-compose.production.yml up -d
```

### Check build status
Go to: `https://github.com/YOUR_USERNAME/cognee/actions`

---

## 💡 Why Use Pre-built Images?

**Before** (Building on Coolify):
```
Deploy → Build 3 services (10-20 min) → Start
         ⚠️ High CPU/memory usage
         ⚠️ Blocks other operations
```

**After** (Pre-built images):
```
GitHub → Build once (automatic) → Store in registry
Coolify → Pull images (1-2 min) → Start
          ✅ Fast deployment
          ✅ Low server load
          ✅ Deploy to multiple servers easily
```

---

## 🎯 Get Started Now

**For development**:
```bash
docker-compose up -d
```

**For production**:
1. Push to GitHub (wait 5-10 min for first build)
2. Configure Coolify with `docker-compose.production.yml`
3. Deploy in 1-2 minutes!

**Need help?** See [PRODUCTION_DEPLOYMENT.md](./PRODUCTION_DEPLOYMENT.md) for detailed instructions.
