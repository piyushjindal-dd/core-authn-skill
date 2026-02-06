# Feature Flags Management

## Quick Reference

### Dashboard
- **AAA Feature Flag Dashboard**: https://app.datadoghq.com/dashboard/89m-zep-ru2

## Two Feature Flag Systems

Core AuthN uses **two different feature flag systems**:

1. **Old Style (Consul Config)** - Uses `consul_config.last_updated` metric
   - Used for service-level configuration flags
   - Controlled via Consul key-value store

2. **New Style (Experiments Client)** - Uses `dd.dynamic_config.is_experiment_enabled` metric
   - Used for gradual rollout experiments
   - Controlled via Datadog's dynamic config platform
   - Supports org-scoped and global rollouts

---

## New Style Feature Flags (Experiments)

### Querying via Datadog MCP

#### Get All Core AuthN Team Experiments
```
mcp__datadog__get_datadog_metric:
  queries: ["count:dd.dynamic_config.is_experiment_enabled{team:team-aaaauthn} by {experiment.id,datacenter,enabled}"]
  from: "now-24h"
```

#### Get Experiments by Service
```
mcp__datadog__get_datadog_metric:
  queries: ["count:dd.dynamic_config.is_experiment_enabled{app:authenticator} by {experiment.id,datacenter,enabled}"]
  from: "now-24h"
```

### Current Core AuthN Experiments (New Style)

| Experiment ID | Purpose | Rollout Status |
|--------------|---------|----------------|
| `frames-mtls-enabled-for-libcontext` | Enable mTLS for libcontext frame communication | **Fully Deployed** - All DCs |
| `ddgrpc_enable_mtls_credentials` | Enable mTLS credentials for ddgrpc | **In Progress** - Partial rollout |
| `revoke_api_keys_of_disabled_org_ace_ff_rollout` | Revoke API keys when org is disabled (ACE integration) | **In Progress** - Partial rollout |

### Interpreting New Style Results

| `enabled` value | Meaning |
|-----------------|---------|
| `true` | Experiment is enabled for the querying context |
| `false` | Experiment is disabled |
| `N/A` | Enabled status varies (check by datacenter) |

---

## Old Style Feature Flags (Consul Config)

### Querying via Datadog MCP

#### Get All Authenticator Flags
```
mcp__datadog__get_datadog_metric:
  queries: ["count:consul_config.last_updated{flag_name:*authenticator*} by {flag_name,flag_default,flag_has_overrides,flag_dc}"]
  from: "now-1h"
```

#### Get All Terminator Flags
```
mcp__datadog__get_datadog_metric:
  queries: ["count:consul_config.last_updated{flag_name:*terminator*} by {flag_name,flag_default,flag_has_overrides,flag_dc}"]
  from: "now-1h"
```

#### Get IP Allowlist Flags
```
mcp__datadog__get_datadog_metric:
  queries: ["count:consul_config.last_updated{flag_name:*ip_allowlist*} by {flag_name,flag_default,flag_has_overrides,flag_dc}"]
  from: "now-1h"
```

#### Get Flags by Team
```
mcp__datadog__get_datadog_metric:
  queries: ["count:consul_config.last_updated{flag_team:aaa-authn} by {flag_name,flag_default,flag_has_overrides}"]
  from: "now-1h"
```

### Interpreting Old Style Results

| `flag_default` | `flag_has_overrides` | Meaning |
|----------------|----------------------|---------|
| `true` | `false` | Fully enabled globally |
| `false` | `false` | Disabled globally |
| `false` | `true` | **In-progress rollout** - enabled for specific orgs |
| `true` | `true` | Fully enabled with some orgs opted out |

---

## Known Core AuthN Feature Flags

### Authenticator Service

| Flag Name | Purpose | Scope |
|-----------|---------|-------|
| `enable_manual_keep_authenticator` | APM manual keep for spans (prevents trace loss during high load). **Warning**: Can cause APM incidents during DDoS (AUTHN-4196) | Request |
| `authenticator_should_produce_app_key_usage` | Controls app key usage event production to streaming producer | Org-scoped |
| `enable_request_throttler_authenticator` | Request throttling for load protection | Request |
| `authenticator_sanitize_secrets_default_other` | Sanitize secrets in logs (non-rapid) | Global |
| `authenticator_sanitize_secrets_default_rapid` | Sanitize secrets in logs (rapid) | Global |
| `mcnulty_use_grpc_authenticator` | Use gRPC for authenticator calls from Mcnulty | Request |

### Terminator Service

| Flag Name | Purpose | Scope |
|-----------|---------|-------|
| `enable_manual_keep_terminator` | APM manual keep for spans | Request |
| `terminator_failover_enforce_authz` | Controls authz enforcement during cross-site failover | Request |
| `enable_terminator_mcp_tool_permissions_check` | Enables MCP tool permissions checking in authorization | Request |
| `terminator_jwt_cache_enabled` | Enables JWT response caching for performance | Global |
| `ip_allowlist` | Controls IP allowlist enforcement for org IP restrictions | Org-scoped |
| `allow_permission_disablement` | Emergency escape hatch - allows filtering specific permissions | Global |
| `inject_built_in_features_permission` | Emergency escape hatch - auto-injects BuiltInFeatures permission | Global |
| `terminator_validate_v1_passthrough` | V1 validation passthrough | Request |
| `terminator_roles_permissions_cache_enabled` | Roles/permissions caching | Global |
| `terminator_read_org_id_from_metadata` | Read org ID from gRPC metadata | Request |
| `terminator_logger_debug_mode` | Enhanced debug logging | Global |
| `enable_terminator_postgres_sleep_statement` | Postgres connection keep-alive | Global |
| `terminator_user_org_cache_v2` | User/org cache v2 implementation | Global |
| `terminator_generic_cache_v2_*` | Generic cache v2 for various data types | Global |

### Edge AuthZ Flags (Terminator)

| Flag Name | Purpose | Typical Rollout |
|-----------|---------|-----------------|
| `enforce_terminator_edge_authz` | Edge authorization enforcement | Staging first |
| `enforce_mcnulty_terminator_edge_authz` | Mcnulty edge authorization | Staging first |
| `enforce_terminator_edge_ip_allowlist` | IP allowlist at edge | Staging first |
| `enforce_mcnulty_terminator_edge_ip_allowlist` | Mcnulty IP allowlist at edge | Staging first |
| `enable_new_authz_logging_terminator` | New authorization logging format | Staging first |

---

## Code Locations for Feature Flags

### Authenticator
```
~/dd/dd-go/apps/authenticator/server/grpc_listener.go
  - EnableManualKeepFF (line ~168)
  - EnableProduceAppKeyUsageFF (line ~313)
```

### Terminator
```
~/dd/dd-go/apps/terminator/server/grpc_interface.go
  - EnableManualKeepFF
  - EnforceFailoverAuthZResFeatureFlag
  - EnableMCPToolPermissionsCheck

~/dd/dd-go/apps/terminator/server/permissions_filter.go
  - allowPermissionDisablementFF
  - injectBuiltInFeaturesPermissionFF

~/dd/dd-go/apps/terminator/jwt/jwt_provider.go
  - jwtCacheEnabledFF

~/dd/dd-go/apps/terminator/authorization/constants.go
  - IpAllowlistFF
```

---

## Adding a New Feature Flag

### 1. Define the Flag Constant

```go
// In your service's constants or relevant file
const EnableNewFeatureFF = "authenticator_enable_new_feature"
```

### 2. Use with Experiments Client

```go
import "go.ddbuild.io/dd-source/x/libs/go/experiments"

// For org-scoped flags (gradual rollout by org)
expCtx := experiments.WithExperimentContext(ctx, &experiments.ExperimentContext{
    OrgId: resolvedCreds.OrgID,
})
if experimentsClient.IsEnabled(expCtx, EnableNewFeatureFF) {
    // New feature logic
}

// For request-scoped flags (global on/off)
if experimentsClient.IsEnabled(ctx, EnableNewFeatureFF) {
    // New feature logic
}
```

### 3. Add Metrics to Track Usage

```go
statsd.Incr("dd.authenticator.new_feature", []string{
    fmt.Sprintf("enabled:%t", enabled),
    fmt.Sprintf("org_id:%d", orgID),
}, 1)
```

### 4. Register in Consul Config

Feature flags are managed via Consul config. Work with the team to:
1. Add the flag definition
2. Set initial default value (usually `false`)
3. Plan gradual rollout strategy

---

## Rollout Strategy

### Typical Rollout Order

1. **dev** - Internal development environment
2. **test** - Automated test environment
3. **staging** (us1.staging.dog, eu1.staging.dog) - Pre-production
4. **prtest** environments - PR testing
5. **us1.prod.dog** - Primary US production
6. **Other prod DCs** - eu1, ap1, ap2, us3, us4, us5
7. **us1.fed.dog** - Federal/GovCloud (last)

### Monitoring During Rollout

1. Watch the service dashboard for error rate changes
2. Monitor latency percentiles (p50, p95, p99)
3. Check for new error patterns in logs
4. Review APM traces for the affected code paths

### Rollback Process

If issues are detected:
1. Disable the flag immediately via Consul
2. Monitor for recovery
3. Investigate root cause before re-enabling

---

## Querying Flag Status Programmatically

### Using Bits CLI

```bash
# Get current status of a specific flag
bits -p "What is the current rollout status of the ip_allowlist feature flag across all datacenters?" --yolo

# Check flags in staging vs prod
bits -p "Compare the terminator feature flags between us1.staging.dog and us1.prod.dog" --yolo
```

### Using Datadog Metrics Explorer

Navigate to: https://app.datadoghq.com/metric/explorer

Query:
```
count:consul_config.last_updated{flag_name:your_flag_name} by {flag_dc,flag_default,flag_has_overrides}
```

---

## Troubleshooting

### Flag Not Taking Effect

1. **Check cache**: Experiments client caches flags. May take up to 60s to propagate
2. **Verify org context**: Org-scoped flags need correct `OrgId` in experiment context
3. **Check DC**: Flag might be enabled in different DC than where you're testing
4. **Review logs**: Look for experiment evaluation logs

### Finding Flag Usage in Code

```bash
# Search for flag usage in dd-go
cd ~/dd/dd-go
grep -r "your_flag_name" apps/authenticator/ apps/terminator/ apps/authenticator-intake/

# Search for flag usage in dd-source
cd ~/dd/dd-source
grep -r "your_flag_name" domains/aaa_authn/
```
