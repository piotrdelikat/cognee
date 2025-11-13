# GitHub Actions Disk Space Fix

## Problem
GitHub Actions runners were running out of disk space during Docker builds, causing this error:
```
System.IO.IOException: No space left on device
```

## Root Cause
- GitHub Actions runners have limited disk space (~14GB free)
- Docker builds, especially multi-stage builds with dependencies, consume significant space
- The `build-images.yml` workflow was missing disk cleanup steps that existed in `dockerhub-mcp.yml`

## Solution Applied

Added disk cleanup steps to all three build jobs in `.github/workflows/build-images.yml`:
1. `build-cognee`
2. `build-cognee-mcp` ← This was causing the failure
3. `build-frontend`

### What the cleanup does:
```bash
# Removes preinstalled software that's not needed
sudo rm -rf /usr/share/dotnet         # .NET SDK (~2-3GB)
sudo rm -rf /usr/local/lib/android    # Android SDK (~10GB)
sudo rm -rf /opt/ghc                  # Haskell (~2GB)
sudo rm -rf /opt/hostedtoolcache      # Cached tools (~5GB)

# Clean up package manager
sudo apt-get autoremove -y
sudo apt-get clean

# Remove old Docker images/containers
docker system prune -af --volumes
```

**Expected space freed: ~20-25GB**

## Additional Optimizations (Optional)

If you still encounter space issues, consider these:

### 1. **Use Smaller Base Images**
```dockerfile
# Current: python:3.12-slim-bookworm
# Consider: python:3.12-alpine (even smaller)
```

### 2. **Build for Single Platform in CI**
```yaml
# Instead of:
platforms: linux/amd64,linux/arm64

# Use:
platforms: linux/amd64  # Build only for amd64 in CI
```

### 3. **Use GitHub's Free Disk Space Action**
Add this to workflow:
```yaml
- name: Free Disk Space (Ubuntu)
  uses: jlumbroso/free-disk-space@main
  with:
    tool-cache: true
    android: true
    dotnet: true
    haskell: true
    large-packages: true
    docker-images: true
    swap-storage: true
```

### 4. **Disable BuildKit Cache** (if desperate)
```yaml
# Remove these lines:
cache-from: type=registry,ref=...
cache-to: type=registry,ref=...
```

### 5. **Split into Sequential Jobs**
Instead of running all builds in parallel, run them sequentially:
```yaml
build-cognee-mcp:
  needs: build-cognee  # Wait for previous build to finish
```

## Testing the Fix

1. Commit and push the changes:
```bash
git add .github/workflows/build-images.yml
git commit -m "fix: add disk cleanup to prevent CI space issues"
git push origin main
```

2. Monitor the workflow:
- Watch for "Free disk space" step output
- Check "Before cleanup" vs "After cleanup" disk usage
- Build should now complete successfully

## Expected Results

**Before cleanup:**
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        84G   64G   14G  82% /
```

**After cleanup:**
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        84G   45G   39G  54% /
```

This gives ~25GB more space, which should be sufficient for the Docker builds.

## Monitoring

If builds still fail, check the workflow logs for:
1. How much space was freed in the cleanup step
2. Which step consumes the most space
3. Consider the additional optimizations above

## Related Files
- `.github/workflows/build-images.yml` (fixed)
- `.github/workflows/dockerhub-mcp.yml` (already had cleanup)
- `cognee-mcp/Dockerfile` (could be optimized further)
