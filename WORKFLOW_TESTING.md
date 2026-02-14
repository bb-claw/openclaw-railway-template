# Workflow Testing Guide

This document explains how to test and verify the deployment workflows.

## Test Scenarios

### Scenario 1: Feature Branch Push (Safe Testing)
**Goal:** Verify Docker build works on feature branches (no redeploy/health-check)

**Steps:**
1. Push to a `feature/*` branch
2. Watch GitHub Actions
3. Expected: Only `Docker build (Feature Branches)` runs
4. Expected: Builds and tags image with `dev` tag
5. Expected: NO redeploy, NO health-check, NO buddy deployment

**Verification:**
- Check Docker build workflow: `.github/workflows/docker-build-feature.yml`
- Image tagged: `ghcr.io/bb-claw/openclaw-railway-template:feature-name`
- No Railway redeploy happens

### Scenario 2: Main Branch Push (Full Pipeline)
**Goal:** Verify complete pipeline: build → redeploy → health-check → buddy → buddy-health-check

**Steps:**
1. Push to `main` branch
2. Watch GitHub Actions → All workflows tab
3. Expected sequence:
   - `Docker build` (main) - Build & redeploy primary
   - `Health Check & Buddy Deployment` - Verify primary health & trigger buddy
   - `Deploy Buddy` - Redeploy buddy, verify health, run for 2 hours

**Verification:**
```
Push to main (commit X)
    ↓
Docker build starts (~1-2 min)
    ├─ Build image
    ├─ Push to GHCR
    └─ Redeploy primary service
    ↓
Docker build completes
    ↓
health-check-and-buddy.yml auto-triggers
    ├─ health-check job: Wait 60s → Poll /setup/healthz
    └─ trigger-buddy job: Dispatch deploy-buddy.yml
    ↓
deploy-buddy.yml starts
    ├─ Redeploy buddy service
    ├─ Wait 60s for buddy to stabilize
    ├─ Health check buddy: Poll /setup/healthz ✅ (NEW)
    ├─ Run for 2 hours
    └─ Scale down
```

**Expected Duration:**
- Docker build: ~1-2 minutes
- Health check (primary): ~2-3 minutes (60s wait + polling)
- Buddy deployment start: ~1 minute
- Buddy health check: ~2-3 minutes (60s wait + polling)
- **Total:** ~7-12 minutes

### Scenario 3: Workflow Dispatch (Manual Buddy Trigger)
**Goal:** Manually trigger buddy deployment without health-check

**Steps:**
1. Go to GitHub Actions → Deploy Buddy Instance
2. Click **Run workflow**
3. Set `buddy_duration_hours` (optional, default 2)
4. Click **Run workflow**

**Verification:**
- Deploy Buddy workflow starts immediately
- Buddy service redeploys
- Runs for specified duration
- Scales down after duration

## Monitoring Workflows

### GitHub Actions Interface
1. Go to repository **Actions** tab
2. Select workflow name from left sidebar
3. Click on run to see detailed logs

### Key Log Indicators

**Successful Docker build:**
```
✅ Redeploy triggered — new deployment status: RUNNING
```

**Successful health check:**
```
🔍 Checking health endpoint: https://openclaw-primary.up.railway.app/setup/healthz
⏳ Attempt 1/60 - instance not ready yet...
⏳ Attempt 2/60 - instance not ready yet...
✅ Primary instance is healthy (attempt 3/60)
```

**Successful buddy trigger:**
```
🔍 Triggering health check and buddy deployment...
✅ Health check & buddy workflow triggered
```

**Successful deploy buddy:**
```
🤝 Deploying buddy instance...
✅ Buddy redeploy triggered — new deployment status: RUNNING
⏳ Waiting 60 seconds for buddy instance to stabilize...
🔍 Checking buddy health endpoint: https://buddy-url/setup/healthz
⏳ Attempt 1/60 - buddy instance not ready yet...
✅ Buddy instance is healthy (attempt X/60)
⏱️ Running buddy for 2 hour(s)...
✅ Buddy session complete
```

## Troubleshooting

### Docker Build Fails
- Check Dockerfile syntax
- Verify Railway secrets are set correctly
- Check Node dependencies

### Health Check Fails
- Verify `RAILWAY_PRIMARY_URL` secret
- Check if primary instance is running on Railway
- Ensure `/setup/healthz` endpoint exists

### Buddy Trigger Fails
- Check if health-check job passed
- Verify `RAILWAY_BUDDY_SERVICE_ID` and `RAILWAY_BUDDY_ENVIRONMENT_ID` secrets
- Check GitHub Actions workflow has `actions: write` permission

### Buddy Deployment Fails
- Check Railway buddy service exists
- Verify service has correct image configured
- Check if buddy service environment variables are set

## Performance Benchmarks

Monitor these metrics:

| Step | Target | Typical |
|------|--------|---------|
| Docker build | < 5 min | 1-2 min |
| Redeploy | < 2 min | 30-60 sec |
| Health check | < 10 min | 2-3 min |
| Buddy deploy | < 5 min | 1-2 min |
| **Total** | < 25 min | 5-10 min |

## Success Criteria

✅ **Feature branch push:**
- Docker build (Feature Branches) passes
- No redeploy/health-check/buddy runs

✅ **Main branch push:**
- All 5 checks pass (docker-build-feature, docker-build, lint, smoke test, integration)
- Docker build completes successfully
- Health check & buddy workflow triggers and runs
- Both jobs (health-check, trigger-buddy) complete successfully
- Deploy buddy workflow runs and completes

✅ **Health check success:**
- Waits ~60 seconds before checking
- Polls endpoint successfully
- Logs show "✅ Primary instance is healthy"

✅ **Buddy deployment:**
- Triggers automatically after health check
- Shows "🤝 Triggering buddy instance deployment"
- Deploy buddy workflow starts and runs
- Buddy instance health verified (shows "✅ Buddy instance is healthy")
- Buddy runs for configured duration (default 2 hours)

## CI/CD Status

Current workflow files:
- ✅ `.github/workflows/docker-build.yml` - Build & redeploy
- ✅ `.github/workflows/docker-build-feature.yml` - Feature branch build
- ✅ `.github/workflows/health-check-and-buddy.yml` - Health & buddy trigger
- ✅ `.github/workflows/health-check.yml` - Reusable health check (unused, kept for reference)
- ✅ `.github/workflows/deploy-buddy.yml` - Buddy instance deployment

All workflows configured and tested. Ready for production use! 🚀
