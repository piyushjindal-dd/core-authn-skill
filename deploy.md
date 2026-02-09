# Deployment Guide for Authenticator & Terminator

## Quick Reference: Conductor Skill

For deployment commands, the **conductor skill** (`~/.claude/skills/conductor/`) provides comprehensive guidance. Key commands:

```bash
# Deploy to staging
ddr conductor run --target staging --branch aaa/authenticator_staging authenticator --yes
ddr conductor run --target staging --branch aaa/terminator_staging terminator --yes

# Deploy to production
ddr conductor run --target commercial authenticator
ddr conductor run --target commercial terminator

# Check status
ddr conductor describe authenticator --target staging
ddr conductor describe terminator --target commercial

# Rollback
ddr conductor rollback list authenticator --target staging
ddr conductor rollback apply authenticator --target staging --sha <sha>

# Cancel deployment
ddr conductor cancel authenticator --target staging
```

**Note**: For integration branches (devflow integrate), use the **integrate skill** instead.

---

## Datadog API Access

### Production Datadog
- **API Endpoint**: https://api.datadoghq.com
- **Credentials**: `~/.dogrc` - `[Connection]` section
- **UI**: https://app.datadoghq.com

### Staging Datadog
- **API Endpoint**: https://api.datadoghq.com (same endpoint, different org)
- **Credentials**: `~/.dogrc` - `[Connection_Staging]` section
- **Environment Variables**: `DD_API_KEY_STAGING`, `DD_APP_KEY_STAGING`
- **UI**: https://ddstaging.datadoghq.com

### Querying Monitors by Environment

```bash
# Production - alerting terminator monitors
API_KEY=$(grep -A2 '^\[Connection\]' ~/.dogrc | grep apikey | cut -d= -f2 | tr -d ' ')
APP_KEY=$(grep -A2 '^\[Connection\]' ~/.dogrc | grep appkey | cut -d= -f2 | tr -d ' ')
curl -s "https://api.datadoghq.com/api/v1/monitor?monitor_tags=service:terminator" \
  --header "DD-API-KEY: $API_KEY" \
  --header "DD-APPLICATION-KEY: $APP_KEY" | jq '[.[] | select(.overall_state == "Alert")]'

# Staging - alerting terminator monitors
API_KEY_STG=$(grep -A2 '^\[Connection_Staging\]' ~/.dogrc | grep apikey | cut -d= -f2 | tr -d ' ')
APP_KEY_STG=$(grep -A2 '^\[Connection_Staging\]' ~/.dogrc | grep appkey | cut -d= -f2 | tr -d ' ')
curl -s "https://api.datadoghq.com/api/v1/monitor?monitor_tags=service:terminator" \
  --header "DD-API-KEY: $API_KEY_STG" \
  --header "DD-APPLICATION-KEY: $APP_KEY_STG" | jq '[.[] | select(.overall_state == "Alert")]'
```

---

## Staging Deployment

### Staging Integration Branches

Both authenticator and terminator have dedicated staging branches that deploy continuously to staging when pushed:

| Service | Staging Branch | Target Environment |
|---------|----------------|-------------------|
| authenticator | `aaa/authenticator_staging` | us1.staging.dog, us3.staging.dog |
| terminator | `aaa/terminator_staging` | us1.staging.dog, us3.staging.dog |

### Deploying Changes to Staging

#### Method 1: Cherry-pick to Staging Branch

For testing changes before merging to prod:

```bash
cd ~/dd/dd-go

# Fetch latest staging branches
git fetch origin aaa/authenticator_staging aaa/terminator_staging

# Cherry-pick your commit to authenticator staging
git checkout aaa/authenticator_staging
git cherry-pick <your-commit-sha>
git push origin aaa/authenticator_staging

# Cherry-pick your commit to terminator staging
git checkout aaa/terminator_staging
git cherry-pick <your-commit-sha>
git push origin aaa/terminator_staging
```

#### Method 2: Direct Push (if already on staging branch)

```bash
git push origin aaa/authenticator_staging
git push origin aaa/terminator_staging
```

### Config-Only Changes (k8s values files)

**Important**: Conductor's Build Impact Analysis may skip deployment for config-only changes.

The Conductor `source_path` is set to `apps/authenticator` or `apps/terminator`, so changes to `k8s/*/values/` files may be considered "no impactful changes" and skipped.

**Symptoms:**
- Conductor run shows "This run was skipped because we found no impactful changes"
- Message: `TICKET_CHECKER_BUILD_IMPACT_ANALYSIS`

**Solution**: Use `--break-glass` flag to force deployment:

```bash
# Manually trigger deployment for config-only changes
ddr conductor run --target staging --branch aaa/authenticator_staging --break-glass --reason "Config-only change for AUTHN-XXXX" authenticator --yes

ddr conductor run --target staging --branch aaa/terminator_staging --break-glass --reason "Config-only change for AUTHN-XXXX" terminator --yes
```

### Terminator Conductor Activation

The terminator staging Conductor may be deactivated. If you see:
```
The conductor for terminator staging is currently deactivated. Would you like to activate it?
```

Use `--yes` flag to auto-approve activation:
```bash
ddr conductor run --target staging --branch aaa/terminator_staging terminator --yes
```

### Checking Deployment Status

```bash
# Check Conductor status
ddr conductor describe authenticator --target staging
ddr conductor describe terminator --target staging

# View workflow in Atlas UI
# Links shown in describe output, e.g.:
# https://atlas.ddbuild.io/namespaces/default/workflows/conductor_authenticator_staging_1
```

### Monitoring Deployment Progress

#### Via Datadog Logs

```
# Check for deployment events in us3.staging
service:authenticator env:us3.staging.dog
service:terminator env:us3.staging.dog

# Check for canary test logs (if canaries enabled)
service:authenticator env:us3.staging.dog canary
service:terminator env:us3.staging.dog canary
```

#### Via kubectl (if you have cluster access)

```bash
# Watch pod rollout in us3.staging
kubectl --context us3.staging.dog get pods -n authenticator -l app=authenticator -w
kubectl --context us3.staging.dog get pods -n terminator -l app=terminator -w

# Check pod status
kubectl --context us3.staging.dog describe pods -n authenticator -l app=authenticator
kubectl --context us3.staging.dog describe pods -n terminator -l app=terminator
```

---

## Production Deployment

### Production Branches

Production deployments are triggered from the `prod` branch via Conductor.

| Service | Production Branch | Deploy Targets |
|---------|------------------|----------------|
| authenticator | `prod` | All production environments |
| terminator | `prod` | All production environments |

### Standard Production Deploy Flow

1. Merge PR to `prod` branch
2. Conductor automatically picks up changes
3. Deploys to production environments in order (based on deploy.yaml)

### Manual Production Deploy

```bash
# Trigger production deployment
ddr conductor run --target commercial authenticator
ddr conductor run --target commercial terminator
```

### Checking Production Deploy Status

```bash
ddr conductor describe authenticator --target commercial
ddr conductor describe terminator --target commercial
```

---

## Conductor CLI Reference

### Common Commands

```bash
# Trigger a deployment
ddr conductor run --target <target> <service>

# Check status
ddr conductor describe <service> --target <target>

# List all conductors
ddr conductor list --target <target>

# Cancel a running deployment
ddr conductor cancel <service> --target <target>

# Activate a deactivated conductor
ddr conductor activate <service> --target <target>

# Deactivate a conductor
ddr conductor deactivate <service> --target <target>
```

### Useful Flags

| Flag | Description |
|------|-------------|
| `--branch <branch>` | Deploy from specific branch |
| `--sha <sha>` | Deploy from specific commit |
| `--release-from-head` | Deploy from HEAD of current branch |
| `--break-glass` | Bypass checks (use for emergencies or config-only changes) |
| `--reason <reason>` | Provide reason for audit logs |
| `--yes` | Auto-approve prompts |

### Targets

| Target | Description |
|--------|-------------|
| `staging` | Staging environments (us1.staging, us3.staging) |
| `commercial` | Commercial production environments |
| `prod` | Alias for commercial |

---

## Rollback

### Quick Rollback

```bash
# List recent successful deployments
ddr conductor rollback list <service> --target <target>

# Apply rollback to specific version
ddr conductor rollback apply <service> --target <target> --sha <sha>
```

### Manual Rollback via Revert

```bash
cd ~/dd/dd-go
git checkout prod
git revert <problematic-commit>
git push origin prod
# Conductor will automatically deploy the revert
```

---

## Troubleshooting

### Deployment Skipped - No Impactful Changes

**Problem**: Config-only changes (k8s values files) skipped by Build Impact Analysis

**Solution**:
```bash
ddr conductor run --target staging --branch <branch> --break-glass --reason "Config-only change" <service> --yes
```

### Deployment Blocked by Monitor Gate

**Problem**: Deployment blocked because a monitor with `service:<service>` tag is alerting

**How it works**: The deploy.yaml contains monitor gates with queries like:
```yaml
monitorGate:
  query: tag:(service:terminator AND -not-gating)
```
Any monitor matching this query that is in Alert status will block deployment.

**Solutions**:

1. **Override the gate for this deployment** (temporary bypass):
   ```bash
   ddr conductor run --target staging --branch <branch> --override-cordon --reason "Bypassing monitor gate for <reason>" <service> --yes
   ```

2. **Fix the alerting monitor**: Investigate and resolve the underlying issue

3. **Add `not-gating` tag to monitor**: If the monitor shouldn't block deployments, add the `not-gating` tag in terraform-config:
   ```terraform
   tags = concat(local.monitor_tags, ["not-gating"])
   ```

**Finding the blocking monitor**:
- Staging: https://ddstaging.datadoghq.com/monitors/manage - search `tag:service:<service> status:alert`
- Prod: https://app.datadoghq.com/monitors/manage - search `tag:service:<service> status:alert`

### Conductor Deactivated

**Problem**: Conductor shows as deactivated

**Solution**:
```bash
ddr conductor activate <service> --target <target>
# Or use --yes flag when running to auto-activate
ddr conductor run --target <target> <service> --yes
```

### Deployment Stuck

**Problem**: Conductor run seems stuck

**Solutions**:
1. Check Atlas workflow URL for detailed status
2. Check Datadog logs for errors
3. Cancel and retry:
   ```bash
   ddr conductor cancel <service> --target <target>
   ddr conductor run --target <target> <service>
   ```

### Pods Not Starting

**Problem**: Pods in CrashLoopBackOff after deployment

**Investigation**:
```bash
# Check pod logs
kubectl --context <env> logs -n <namespace> -l app=<service> --tail=100

# Check pod events
kubectl --context <env> describe pods -n <namespace> -l app=<service>

# Check startup probe (for canary tests)
kubectl --context <env> describe pods -n <namespace> -l app=<service> | grep -A5 "Startup:"
```

**Quick Recovery**:
```bash
# Scale down
kubectl --context <env> scale deployment <service> -n <namespace> --replicas=0

# Fix config and redeploy, or rollback
ddr conductor rollback apply <service> --target <target> --sha <last-good-sha>

# Scale back up
kubectl --context <env> scale deployment <service> -n <namespace> --replicas=<original>
```
