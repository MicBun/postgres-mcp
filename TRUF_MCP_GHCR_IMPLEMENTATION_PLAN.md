# TRUF-Enhanced PostgreSQL MCP - GHCR Implementation Plan

## 📋 Current State Analysis

### Repository Structure
- **Location**: `/home/micbun/trufnetwork/postgres-mcp`
- **Remote**: `https://github.com/trufnetwork/postgres-mcp.git`
- **Current Branch**: `main`
- **Latest Commit**: `e359576 chore: restricted` (production access mode)

### Existing Infrastructure
- ✅ **Multi-platform Dockerfile** (`linux/amd64`, `linux/arm64`)
- ✅ **GitHub Actions CI/CD** (`.github/workflows/`)
  - `build.yml` - Python CI with uv, ruff, pyright, pytest
  - `docker-build-dockerhub.yml` - DockerHub publishing pipeline
- ✅ **Production-ready configuration**
  - SSE transport support (port 8000)
  - Access modes: `restricted`/`unrestricted`
  - Caddy reverse proxy setup

### TRUF-Specific Features Implemented
- 🎯 **Stream Analytics Tools** (`src/postgres_mcp/truf/`):
  - `get_composed_stream_records` - Time series data queries
  - `get_latest_composed_stream_record` - Latest values
  - `get_index` - Stream index operations
  - `get_index_change` - Percentage change calculations
- 🎯 **Advanced Query Capabilities**:
  - Complex recursive CTEs for stream hierarchies
  - Taxonomy-based stream composition
  - Time-travel queries with `frozen_at` parameter
- 🎯 **Business Intelligence Features**:
  - Database health monitoring
  - Index tuning with LLM optimization
  - Safe SQL execution modes

## 🎯 Implementation Objectives

### Primary Goal
Create GitHub Container Registry (GHCR) publishing pipeline for TRUF-enhanced postgres-mcp to replace basic `crystaldba/postgres-mcp:latest` in AWS AMI deployments.

### Success Criteria
1. **Image Publishing**: `ghcr.io/trufnetwork/postgres-mcp:latest` available
2. **AMI Integration**: Updated docker-compose template uses TRUF-enhanced image
3. **Feature Availability**: TRUF stream tools accessible via SSE in AMI
4. **Business Value**: AI agents can perform TRUF-specific analytics

## 🛠️ Technical Implementation Plan

### Phase 1: GHCR Workflow Creation
**Timeline**: 2-3 hours

#### 1.1 Create GHCR Publishing Workflow
**File**: `.github/workflows/publish-ghcr.yml`

```yaml
name: Publish to GitHub Container Registry

on:
  push:
    branches: [main]
    tags: ['v*']
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: trufnetwork/postgres-mcp

jobs:
  build-and-publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
        with:
          platforms: linux/amd64,linux/arm64

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/trufnetwork/postgres-mcp
          tags: |
            type=ref,event=branch
            type=ref,event=tag
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

#### 1.2 Update Repository Settings
- **Required**: Enable GitHub Packages in repository settings
- **Required**: Ensure `GITHUB_TOKEN` has package write permissions
- **Optional**: Configure package visibility (public recommended)

### Phase 2: AMI Integration
**Timeline**: 1-2 hours

#### 2.1 Update Node Repository AMI Template
**File**: `/home/micbun/trufnetwork/node/deployments/infra/stacks/docker-compose.template.yml`

**Current** (line ~84):
```yaml
postgres-mcp:
  image: crystaldba/postgres-mcp:latest
```

**Updated**:
```yaml
postgres-mcp:
  image: ghcr.io/trufnetwork/postgres-mcp:latest
```

#### 2.2 Update AMI Pipeline Stack
**File**: `/home/micbun/trufnetwork/node/deployments/infra/stacks/ami_pipeline_stack.go`

Add image pull configuration if needed for private GHCR access.

### Phase 3: Testing & Validation
**Timeline**: 2-3 hours

#### 3.1 Local Testing
```bash
# Test GHCR image locally
docker pull ghcr.io/trufnetwork/postgres-mcp:latest
docker run -p 8000:8000 \
  -e DATABASE_URI="postgresql://user:pass@host:5432/db" \
  ghcr.io/trufnetwork/postgres-mcp:latest \
  --transport=sse --sse-host=0.0.0.0 --access-mode=restricted
```

#### 3.2 AMI Pipeline Testing
1. **Build AMI** with updated docker-compose template
2. **Deploy test instance** from new AMI
3. **Verify TRUF tools** accessible via SSE transport:
   - `curl http://instance:8000/sse` (health check)
   - Test `get_composed_stream_records` via AI client

#### 3.3 Integration Testing
- **Claude Desktop** connectivity test
- **TRUF stream analytics** functionality verification
- **Business intelligence** use case validation

### Phase 4: Documentation & Rollout
**Timeline**: 1-2 hours

#### 4.1 Update Documentation
**Files to Update**:
- `README.md` - Add GHCR installation instructions
- `/home/micbun/trufnetwork/node/AwsAmiGitHubIssues.md` - Mark issue #1149 resolved

#### 4.2 Update Installation Scripts
**File**: `/home/micbun/trufnetwork/postgres-mcp/install.sh`
- Add GHCR pull instructions
- Update docker-compose examples

## 🔄 Deployment Strategy

### Rollout Approach
1. **Parallel Deployment**: Keep existing DockerHub pipeline active
2. **Gradual Migration**: Update AMI template to use GHCR image
3. **Validation Period**: Monitor AMI deployments for 1-2 weeks
4. **Full Cutover**: Once validated, deprecate DockerHub builds

### Rollback Plan
- **Quick Revert**: Change AMI template back to `crystaldba/postgres-mcp:latest`
- **Emergency Fix**: Force push corrected image to GHCR if needed
- **Infrastructure Recovery**: Existing DockerHub pipeline remains available

## 📊 Success Metrics

### Technical Metrics
- ✅ **GHCR Image Available**: `docker pull ghcr.io/trufnetwork/postgres-mcp:latest` succeeds
- ✅ **AMI Build Success**: New AMI builds without errors using GHCR image
- ✅ **Container Startup**: postgres-mcp service starts successfully in AMI instances
- ✅ **TRUF Tools Available**: All 7 TRUF-specific MCP tools accessible

### Business Metrics
- 📈 **AI Integration Usage**: Users can perform stream analytics via AI agents
- 📈 **Developer Experience**: TRUF tools available out-of-the-box in AMI
- 📈 **Competitive Advantage**: Advanced MCP capabilities not available elsewhere

## ⚠️ Risk Assessment

### Low Risk
- **GHCR Publishing**: Well-established GitHub feature with good documentation
- **Docker Image Changes**: Minimal changes to existing working Dockerfile
- **CI/CD Integration**: Follows existing patterns from DockerHub workflow

### Medium Risk
- **AMI Integration**: Requires testing full AMI build pipeline
- **Container Registry Migration**: New dependencies on GitHub infrastructure

### Mitigation Strategies
- **Comprehensive Testing**: Test locally before AMI integration
- **Parallel Deployment**: Keep existing DockerHub builds as backup
- **Staged Rollout**: Deploy to development environment first

## 🎯 Next Steps

### Immediate Actions (Today)
1. **Create GHCR workflow** (`.github/workflows/publish-ghcr.yml`)
2. **Test local build** and push to verify GHCR access
3. **Update AMI template** to use new image

### This Week
1. **Deploy test AMI** with GHCR image
2. **Validate TRUF tools** functionality
3. **Update documentation**

### Success Validation
- [ ] GHCR image publishes automatically on push to main
- [ ] AMI builds successfully with GHCR image
- [ ] TRUF stream tools work in deployed AMI instance
- [ ] AI agents can access enhanced MCP capabilities

---

**Implementation Owner**: Development Team
**Timeline**: 1-2 days
**Priority**: High (blocks enhanced AI integration capabilities)