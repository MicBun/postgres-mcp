# GHCR Workflow Testing Guide

This document outlines how to test the GitHub Container Registry (GHCR) publishing workflow before and after deployment.

## 🧪 Pre-Deployment Testing

### 1. Local Docker Build Test

Test the Docker build process locally to ensure it works before pushing to GitHub:

```bash
# Test single platform build (faster)
docker buildx build --platform linux/amd64 -t test-truf-mcp .

# Test multi-platform build (matches CI workflow)
docker buildx build --platform linux/amd64,linux/arm64 -t test-truf-mcp .

# Test with load (to use locally)
docker buildx build --platform linux/amd64 -t test-truf-mcp --load .
```

### 2. YAML Syntax Validation

Ensure the GitHub Actions workflow has valid YAML syntax:

```bash
# Using Python (available on most systems)
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/publish-ghcr.yml')); print('✅ YAML syntax valid')"

# Using yamllint (if available)
yamllint .github/workflows/publish-ghcr.yml
```

### 3. GitHub Actions Local Testing (Optional)

If you have `act` installed, you can test the workflow locally:

```bash
# Install act (if not already installed)
curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# List available workflows
act -l

# Test workflow with act (will attempt to run containers)
act workflow_dispatch -W .github/workflows/publish-ghcr.yml

# Test with secrets (requires setup)
act workflow_dispatch -W .github/workflows/publish-ghcr.yml -s GITHUB_TOKEN="your_token"

# Test only specific job without pushing
act workflow_dispatch -W .github/workflows/publish-ghcr.yml -j build-and-publish --dryrun
```

## 🚀 Post-Deployment Testing

### 1. Verify GitHub Actions Workflow

After pushing the workflow file:

1. **Go to GitHub Actions tab**: `https://github.com/trufnetwork/postgres-mcp/actions`
2. **Check for workflow run**: Look for "Publish to GitHub Container Registry" workflow
3. **Monitor build logs**: Ensure all steps pass successfully
4. **Verify multi-platform build**: Check that both `linux/amd64` and `linux/arm64` are built

### 2. Verify GHCR Image Publication

Check that the image is published to GitHub Container Registry:

```bash
# Pull the image from GHCR
docker pull ghcr.io/trufnetwork/postgres-mcp:latest

# Inspect image details
docker inspect ghcr.io/trufnetwork/postgres-mcp:latest

# Check supported platforms
docker buildx imagetools inspect ghcr.io/trufnetwork/postgres-mcp:latest
```

### 3. Test Image Functionality

Verify the GHCR image works correctly:

```bash
# Test basic startup
docker run --rm ghcr.io/trufnetwork/postgres-mcp:latest --help

# Test with SSE transport (requires database connection)
docker run -p 8000:8000 \
  -e DATABASE_URI="postgresql://user:pass@host:5432/db" \
  ghcr.io/trufnetwork/postgres-mcp:latest \
  --transport=sse --sse-host=0.0.0.0 --access-mode=restricted

# Test health endpoint
curl http://localhost:8000/health
```

### 4. Test TRUF-Specific Tools

Verify TRUF-enhanced MCP tools are available:

```bash
# Start the MCP server
docker run -d --name test-truf-mcp -p 8000:8000 \
  -e DATABASE_URI="postgresql://user:pass@host:5432/db" \
  ghcr.io/trufnetwork/postgres-mcp:latest \
  --transport=sse --sse-host=0.0.0.0 --access-mode=restricted

# Test SSE endpoint availability
curl -N -H "Accept: text/event-stream" http://localhost:8000/sse

# Check logs for TRUF tools loading
docker logs test-truf-mcp

# Cleanup
docker stop test-truf-mcp && docker rm test-truf-mcp
```

## 🔧 Troubleshooting

### Common Issues and Solutions

#### Build Failures

**Issue**: Docker build fails during dependency installation
```bash
# Solution: Check if base image is accessible
docker pull ghcr.io/astral-sh/uv:python3.12-bookworm-slim
```

**Issue**: Multi-platform build fails
```bash
# Solution: Ensure buildx is set up correctly
docker buildx inspect --bootstrap
docker buildx ls
```

#### GHCR Permission Issues

**Issue**: "permission denied" when pushing to GHCR
```bash
# Solution: Check if GITHUB_TOKEN has package:write permission
# Verify in repository Settings > Actions > General > Workflow permissions
```

**Issue**: Image not visible in GHCR
```bash
# Solution: Check package visibility settings
# Go to: https://github.com/orgs/trufnetwork/packages/container/postgres-mcp/settings
```

#### Runtime Issues

**Issue**: Container fails to start with TRUF tools
```bash
# Solution: Check environment variables and database connection
docker run --rm ghcr.io/trufnetwork/postgres-mcp:latest \
  -e DATABASE_URI="your_connection_string" \
  --transport=sse --sse-host=0.0.0.0 --access-mode=restricted -v
```

**Issue**: TRUF-specific tools not available
```bash
# Solution: Verify the build included TRUF modules
docker run --rm ghcr.io/trufnetwork/postgres-mcp:latest \
  python -c "import postgres_mcp.truf; print('TRUF tools available')"
```

## 📊 Success Criteria Checklist

- [ ] **Local Docker build succeeds** for both platforms
- [ ] **YAML syntax validation passes**
- [ ] **GitHub Actions workflow runs successfully**
- [ ] **GHCR image is published** and publicly accessible
- [ ] **Multi-platform support confirmed** (amd64 + arm64)
- [ ] **Container starts successfully** with SSE transport
- [ ] **TRUF-specific tools are accessible** via MCP protocol
- [ ] **Health endpoints respond correctly**
- [ ] **Database connectivity works** with test connection

## 🔄 Integration Testing with AMI

Once GHCR testing passes, test integration with AWS AMI:

### 1. Update AMI Docker Compose Template

In `/home/micbun/trufnetwork/node/deployments/infra/stacks/docker-compose.template.yml`:

```yaml
# Change from:
postgres-mcp:
  image: crystaldba/postgres-mcp:latest

# To:
postgres-mcp:
  image: ghcr.io/trufnetwork/postgres-mcp:latest
```

### 2. Test AMI Build Process

```bash
# In node repository
cd /home/micbun/trufnetwork/node/deployments/infra

# Test CDK synthesis with new image
cdk --app 'go run ami-cdk.go' synth --context stage=dev

# Build test AMI (if pipeline is set up)
# Monitor AWS ImageBuilder console for build progress
```

### 3. Verify TRUF Tools in Deployed AMI

1. **Launch AMI instance** from AWS Console
2. **SSH into instance** and check MCP server status
3. **Test TRUF-specific tools** via SSE endpoint
4. **Verify business intelligence capabilities**

---

## 📝 Notes

- **Automated Testing**: Consider adding these tests to CI/CD pipeline for continuous validation
- **Performance Testing**: Monitor build times and image sizes for optimization opportunities
- **Security Testing**: Verify access modes and permissions work correctly
- **Documentation Updates**: Keep this guide updated as workflow evolves