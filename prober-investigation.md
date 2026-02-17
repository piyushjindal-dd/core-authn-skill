# AuthN Prober Investigation Guide

**Purpose:** Systematic investigation of authn-prober failures across datacenters.

**Service:** `authn-prober`
**Team:** Core AuthN (AAA-AUTHN)
**Dashboard:** https://app.datadoghq.com/dashboard/an2-7e4-vdz
**Runbook:** https://datadoghq.atlassian.net/wiki/spaces/AAAAUTHN/pages/TBD

---

## Overview

The authn-prober validates authentication latency and correctness across datacenters by:
1. Creating fresh API/App keys via ACE
2. Testing key propagation to authenticator service
3. Validating keys via Smart Edge endpoints
4. Measuring end-to-end latency (SLA: <10 seconds)

**Key Metrics:**
- `latency_slo.probe.latency_sec` - End-to-end latency (timeout at 90s)
- `latency_slo.probe.error` - Probe failures (reason tags)

---

## Architecture

### Prober Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Prober Test Flow                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Create App Key via ACE gRPC                                     │
│     └─ ACE.CreateCustomerApplicationKey(org, owner, name)          │
│     └─ Returns: {key, uuid, hash}                                   │
│                                                                     │
│  2. ACE Writes to PostgreSQL                                        │
│     └─ Table: application_key2                                      │
│     └─ Calls: NotifyContextChange()                                 │
│                                                                     │
│  3. Context Change Notification                                     │
│     └─ ACE → event-context-writer (gRPC)                            │
│     └─ event-context-writer → libcontext blob service               │
│     └─ Publishes to org's home_datacenter context                   │
│                                                                     │
│  4. Authenticator Subscribes via Libcontext                         │
│     └─ 2-layer cache (in-memory LRU + blob service)                 │
│     └─ Keys propagate in <10s (SLA)                                 │
│                                                                     │
│  5. Prober Polls /probe_smart_edge                                  │
│     └─ Headers: DD-API-KEY + DD-APPLICATION-KEY + Bearer token      │
│     └─ Polls every 500ms for max 90 seconds                         │
│     └─ Success: HTTP 200, Failure: HTTP 403/401                     │
│                                                                     │
│  6. Cleanup (revoke + delete key)                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Location | Purpose |
|-----------|----------|---------|
| **authn-prober** | `~/dd/dd-source/domains/aaa_authn/apps/authn-prober/` | Test orchestrator |
| **ACE** | `~/dd/dd-source/domains/aaa/apps/ace/` | Credential creation |
| **event-context-writer** | Rapid service (logs-general namespace) | Context publishing |
| **Authenticator** | `~/dd/dd-go/apps/authenticator/` | Key resolution |
| **Smart Edge** | Envoy proxy layer | API gateway |

---

## Common Failure Modes

### 1. Invalid Key Pair (Org Mismatch) ⭐ MOST COMMON

**Symptom:**
- Prober times out after 90 seconds with 403 errors
- Authenticator logs: `"invalid key pair resolved"`
- API key and app key belong to different orgs

**Root Cause:**
```
API Key org:  1500000002 (from Vault smart-edge-keys)
App Key org:  1500093531 (created by prober)
              ↑ MISMATCH → 403 Forbidden
```

**Why It Happens:**
- Prober config specifies orgId for key creation
- But uses shared API key from Vault (different org)
- Authenticator validates that both keys belong to same org/user

**Fix:**
```yaml
# Ensure API key and app key org match
# Option A: Use API key from same org as prober orgId
# Option B: Create prober org that matches Vault API key org
```

**Files to Check:**
- `domains/aaa_authn/apps/authn-prober/config/k8s/endpoints/{datacenter}.yaml` - Org config
- `domains/aaa_authn/apps/authn-prober/config/k8s/probe.yaml` - API key source (Vault path)

### 2. Context Propagation Failure

**Symptom:**
- Keys created successfully (ACE returns 200)
- Keys exist in database (confirmed via find-key.sh)
- `/api/v1/validate` returns 403/401 (key not in authenticator context)

**Root Cause:**
- Org not configured as datacenter-native
- Keys published to wrong datacenter's context
- Authenticator never receives the keys

**Investigation:**
```bash
# Check if keys propagate
curl https://api.{dc}.datadoghq.com/api/v1/validate \
  -H "DD-API-KEY: {api_key}" \
  -H "DD-APPLICATION-KEY: {app_key}"

# If 401/403: Key not in authenticator context
# If 200: Key propagated successfully
```

**Fix:**
- Ensure org's `home_datacenter` matches deployment DC
- Or create DC-native prober org

### 3. Bearer Token Format Issue

**Symptom:**
- `/api/v1/validate` works (200)
- `/probe_smart_edge` returns 401 Unauthorized

**Root Cause:**
```go
// Authenticator requires 40-char app keys in Bearer header
func stripAppKeyBearerTokenFromHeader(authHeader string) string {
    if len(token) == 40 {  // ← Strict requirement
        return token
    }
    return ""  // Empty = no Bearer auth
}
```

**Fix:**
- Ensure Bearer token is 40-character application key
- Or remove Bearer token requirement from endpoint

### 4. IP Allowlist Restriction

**Symptom:**
- Consistent 403 from specific IPs
- Works from some locations but not others

**Root Cause:**
- McNulty rate limits have IP allowlist
- Prober pod IP not in approved list

**Investigation:**
```bash
# Check IP allowlist config
cat ~/dd/consul-config/datadog/{dc}/consul_config/mcnulty_rate_limits.ini | grep -A5 "ip.whitelist"

# Check prober pod IP (if NAT gateway exists)
aws-vault exec prod-engineering -- aws ec2 describe-nat-gateways ...
```

---

## Investigation Playbook

### Quick Health Check

```bash
# 1. Check prober status
kubectl get deployment -n authenticator authn-prober --context {cluster}

# 2. Check recent errors
kubectl logs -n authenticator deployment/authn-prober --context {cluster} --tail=100 | grep -E "error|timeout|403|401"

# 3. Check prober metrics
# Open: https://app.datadoghq.com/dashboard/an2-7e4-vdz
# Filter by datacenter:{dc}
# Look for latency_slo.probe.error spikes
```

### Systematic Investigation (4-Step Test)

Use the test script to identify exact failure point:

**Script Location:** `~/test_key_propagation_chain.sh`

```bash
bash ~/test_key_propagation_chain.sh
```

**What It Tests:**
1. **STEP 1**: Create app key via ACE (like prober does)
2. **STEP 2**: Verify key in database (via find-key.sh)
3. **STEP 3**: Test authenticator via `/api/v1/validate`
4. **STEP 4**: Test via `/probe_smart_edge` (prober endpoint)

**Interpretation:**

| Step Results | Root Cause | Fix |
|--------------|------------|-----|
| 1✅ 2❌ 3❌ 4❌ | Database write failure | Check ACE → Postgres connectivity |
| 1✅ 2✅ 3❌ 4❌ | Context propagation | Check org home_datacenter, libcontext |
| 1✅ 2✅ 3✅ 4❌ | Endpoint authorization | Check Bearer token, IP allowlist, key pair org match |
| 1✅ 2✅ 3✅ 4✅ | No issue | Prober working correctly |

### Deep Dive Investigation

#### Check 1: Key Pair Org Match

```bash
# Get prober config
cat ~/dd/dd-source/domains/aaa_authn/apps/authn-prober/config/k8s/endpoints/{dc}.yaml

# Check API key org
vault kv get kv/data/k8s/authenticator/authn-prober/smart-edge-keys

# Verify org match
echo "Prober org ID: {from config}"
echo "API key org: {from vault or database}"
echo "These MUST match!"
```

**Common Issue:**
```yaml
# ❌ WRONG: Mismatched orgs
probes:
  smart_edge_fresh_app_key_latency_probe:
    orgId: 1500093531  # Prober creates keys for this org

# But API key is from:
vault: kv/data/.../smart-edge-keys
  api_key: dd9e23...  # Belongs to org 1500000002

# ✅ CORRECT: Matching orgs
probes:
  smart_edge_fresh_app_key_latency_probe:
    orgId: 1500000002  # Same as API key org
```

#### Check 2: Authenticator Logs

```bash
# Find failed requests for prober org
kubectl logs -n authenticator deployment/authenticator --context {cluster} | \
  grep "probe_smart_edge" | \
  grep "1500093531" | \
  grep "failed\|denied" | \
  tail -10

# Look for:
# - "invalid key pair resolved" → Org mismatch
# - "key not found" → Context propagation issue
# - "org_id":X,"api_key_org_id":Y → Extract org IDs to compare
```

#### Check 3: Context Propagation

```bash
# Check ACE notifications
kubectl logs -n ace deployment/ace --context {cluster} | \
  grep "Context change notification" | \
  tail -20

# Check event-context-writer
kubectl logs -n logs-general deployment/event-context-writer-default --context {cluster} | \
  grep "EDGE_API_KEY_CONTEXT\|EDGE_APP_KEY_CONTEXT" | \
  tail -20

# Compare context sizes between DCs
# AP2 should have ~11k keys, AP1 ~470k keys (by design)
```

#### Check 4: Database Verification

```bash
# Use find-key.sh to locate key
~/go/src/github.com/DataDog/team-aaa-internal-tools/scripts/find-key.sh {app_key}

# Check which datacenter has the key
# Should match org's home_datacenter
```

---

## Prober Configuration

### Config Files

**Endpoint-Specific Config:**
```
domains/aaa_authn/apps/authn-prober/config/k8s/endpoints/{datacenter}.yaml
```

Example:
```yaml
endpoints:
  ap2.prod.dog:
    - gateway: edge_gateway
      flavor: edge-pool2
      host: api.ap2.datadoghq.com
      port: 443
      probes:
        - smart_edge_fresh_app_key_latency_probe

probes:
  smart_edge_fresh_app_key_latency_probe:
    orgId: 1500093531
    orgUuid: e9146dd9-adc0-11f0-8d6e-e2db00f6adb5
    appKeyOwnerId: 861628
    appKeyOwnerUuid: 23b03ff3-adc1-11f0-b549-82f6ecb04496
```

**Global Config:**
```
domains/aaa_authn/apps/authn-prober/config/k8s/probe.yaml
```

Contains:
- Bearer token (shared across all DCs)
- API key Vault paths
- Probe timeouts and frequencies

### Vault Secrets

| Secret Path | Contains | Used For |
|-------------|----------|----------|
| `kv/data/k8s/authenticator/authn-prober/smart-edge-keys` | api_key | Main API key for prober |
| `kv/data/k8s/authenticator/authn-prober/intake-authn-keys` | api_key, client_token, slo_api_key, etc. | Various test scenarios |

**⚠️ CRITICAL**: API key org MUST match prober orgId in config!

---

## Investigation Tools

### Tool 1: Key Propagation Chain Test Script

**Location:** `~/test_key_propagation_chain.sh`

**Purpose:** Systematically test each component in the propagation pipeline.

**Usage:**
```bash
bash ~/test_key_propagation_chain.sh
```

**What It Does:**
1. Creates app key via ACE (like prober)
2. Verifies key in database (via find-key.sh)
3. Tests authenticator context (`/api/v1/validate`)
4. Tests Smart Edge endpoint (`/probe_smart_edge`)
5. Identifies exact failure point

**Output:**
```
STEP 1: ACE Key Creation              [✓ PASS / ✗ FAIL]
STEP 2: Database Persistence           [✓ PASS / ✗ FAIL]
STEP 3: Authenticator Context          [✓ PASS / ✗ FAIL]
STEP 4: Smart Edge Probe Endpoint      [✓ PASS / ✗ FAIL]

FAILURE POINT: {detailed diagnosis}
```

### Tool 2: find-key.sh

**Location:** `~/go/src/github.com/DataDog/team-aaa-internal-tools/scripts/find-key.sh`

**Purpose:** Locate API/App keys across datacenters.

**Usage:**
```bash
# Search for app key (40 chars)
./find-key.sh {40_character_app_key}

# Search for API key (32 chars)
./find-key.sh {32_character_api_key}
```

**Output:**
- Which datacenter has the key
- Org ID, owner ID, created_at, revoked status
- Links to SupportDog for org/user details

**Note:** Script searches: US1, AP1, US5, US3, EU1, but NOT AP2 by default.

### Tool 3: Authenticator Logs Analysis

**Find Failed Requests:**
```bash
kubectl logs -n authenticator deployment/authenticator --context {cluster} | \
  grep "probe_smart_edge" | \
  grep "failed\|denied" | \
  jq -r '. | {org_id, user_id, api_key_org_id: (.api_key_uuid as $uuid | .org_id), reason, code}'
```

**Check for:**
- `"reason":"invalid key pair resolved"` → Org mismatch
- `"reason":"key not found"` → Context propagation issue
- `"parsed_api_key":"false"` → Missing API key header
- `"parsed_app_key":"false"` → Missing app key header

**Successful Request Example:**
```json
{
  "msg": "Check: Authentication succeeded",
  "route_name": "probe_smart_edge_get",
  "route_methods": "[ValidFullAPIUser]",
  "user_id": 6968,
  "org_id": 1500000002,
  "api_key_uuid": "...",
  "api_key_org_id": 1500000002,  ← Same org!
  "app_key_uuid": "...",
  "method": "ValidFullAPIUser",
  "status": "success"
}
```

**Failed Request Example:**
```json
{
  "msg": "Check: Authentication failed",
  "user_id": 861628,
  "org_id": 1500093531,           ← App key org
  "api_key_org_id": 1500000002,   ← API key org (DIFFERENT!)
  "status": "failure",
  "reason": "invalid key pair resolved",
  "code": "Unauthenticated"
}
```

---

## Authentication Flow Details

### Endpoint: `/probe_smart_edge`

**Auth Methods Accepted:**
- `ValidFullAPIUser` (API key + App key, same org/user)
- `ValidUserDelegatedToken`
- `ValidUserPAT`

**Headers Required:**
```http
DD-API-KEY: {32_character_api_key}
DD-APPLICATION-KEY: {40_character_app_key}
Authorization: Bearer {40_character_app_key}  ← Optional but validated if present
```

**Validation Logic:**

1. **Parse Headers:**
   - API key from `DD-API-KEY` header
   - App key from `DD-APPLICATION-KEY` header
   - Bearer token from `Authorization: Bearer {token}` (if present)

2. **Validate Bearer Token Format:**
   ```go
   // From ddextauthz/utils.go:430-439
   if len(token) == 40 {  // MUST be exactly 40 chars
       return token
   }
   return ""  // Invalid length = ignored
   ```

3. **Resolve Credentials:**
   - Hash API key → lookup in context
   - Hash App key → lookup in context
   - Validate org_id and user_id match

4. **Check Key Pair Validity:**
   - API key org == App key org ✅
   - API key user == App key user ✅
   - Both keys active (not revoked) ✅

### Comparison: `/api/v1/validate` vs `/probe_smart_edge`

| Aspect | `/api/v1/validate` | `/probe_smart_edge` |
|--------|-------------------|---------------------|
| **Auth Methods** | `ValidFullAPIUser` OR `ValidAPIKey` | `ValidFullAPIUser` only |
| **Requires App Key** | No (optional) | Yes (required) |
| **Bearer Token** | Ignored | Validated if present |
| **Org Match Required** | No (API-only works) | Yes (strict validation) |
| **Use Case** | General API validation | Smart Edge monitoring |

---

## Debugging Checklist

When prober fails with 90-second timeout:

### ☑ Step 1: Identify Failure Point

Run the test script:
```bash
bash ~/test_key_propagation_chain.sh
```

### ☑ Step 2: Check Org Configuration

```bash
# Get prober org from config
grep "orgId" ~/dd/dd-source/domains/aaa_authn/apps/authn-prober/config/k8s/endpoints/{dc}.yaml

# Get API key org from logs
kubectl logs -n authenticator deployment/authenticator --context {cluster} | \
  grep "1500093531" | grep "api_key_org_id" | head -1 | jq '.api_key_org_id'

# Compare orgs
echo "Prober org: {from config}"
echo "API key org: {from logs}"
echo "These MUST match for ValidFullAPIUser auth!"
```

### ☑ Step 3: Verify Context Propagation

```bash
# Check ACE notifications
kubectl logs -n ace deployment/ace --context {cluster} --since=5m | \
  grep "Context change notification"

# Check event-context-writer
kubectl logs -n logs-general deployment/event-context-writer-default --context {cluster} --since=5m | \
  grep "EDGE_APP_KEY_CONTEXT"

# Test key immediately after creation
# Should work within 10 seconds
```

### ☑ Step 4: Check Authenticator Logs

```bash
# Look for the specific failure
kubectl logs -n authenticator deployment/authenticator --context {cluster} | \
  grep "probe_smart_edge" | \
  grep "Authentication failed" | \
  jq -r '. | {reason, org_id, api_key_org_id}'
```

### ☑ Step 5: Verify Database Persistence

```bash
# Create test key and immediately search
~/go/src/github.com/DataDog/team-aaa-internal-tools/scripts/find-key.sh {app_key}

# Should find key in datacenter matching org's home_datacenter
```

---

## Common Fixes

### Fix 1: Org Mismatch

**Problem:** API key org ≠ App key org

**Solution A:** Update prober to use API key from correct org
```bash
# 1. Create API key for prober org
# 2. Store in Vault at custom path
# 3. Update probe.yaml to reference new Vault path
```

**Solution B:** Update prober org to match API key
```yaml
# Change orgId in {dc}.yaml to match API key's org
probes:
  smart_edge_fresh_app_key_latency_probe:
    orgId: 1500000002  # Match API key org
```

### Fix 2: Context Propagation

**Problem:** Keys not reaching datacenter's authenticator

**Solution:** Create DC-native prober org
```sql
-- Create org with correct home_datacenter
INSERT INTO organizations (name, home_datacenter, ...)
VALUES ('authn_prober_ap2', 'ap2.prod.dog', ...);

-- Update prober config
```

### Fix 3: Bearer Token Format

**Problem:** Bearer token is not 40 characters

**Solution:** Use 40-character app key as Bearer token
```yaml
# Remove 32-char Bearer token
# OR ensure Bearer token is valid 40-char app key
```

---

## Key Files Reference

### Prober Code

| File | Purpose |
|------|---------|
| `domains/aaa_authn/apps/authn-prober/bootstrap_keys.go` | Key creation logic |
| `domains/aaa_authn/apps/authn-prober/config/k8s/probe.yaml` | Global config (Bearer token, Vault paths) |
| `domains/aaa_authn/apps/authn-prober/config/k8s/endpoints/*.yaml` | DC-specific config (org, owner) |

### Authenticator Code

| File | Purpose |
|------|---------|
| `apps/authenticator/datastore/repository/credential_readers.go:429-443` | Bearer token resolution |
| `pkg/ddextauthz/utils.go:309-312` | ApplicationKeyBearerToken extraction |
| `pkg/ddextauthz/utils.go:430-439` | Bearer token validation (40-char check) |
| `pkg/ddextauthz/denied_response.go:69-74` | Error code mapping |

### ACE Code

| File | Purpose |
|------|---------|
| `domains/aaa/apps/ace/internal/grpc/ace_server.go:226` | NotifyContextChange call |
| `domains/aaa/apps/ace/internal/context_change/client.go:74` | Context notification implementation |

### Lambo Config

| File | Purpose |
|------|---------|
| `domains/api_platform/apps/apis/lambo/internal/lambo/validation/authn_exceptions/apikey_only_auth.go:136-139` | `/probe_smart_edge` auth exception |

---

## Example Investigation: AP2 Prober Failure (2026-02-17)

### Symptoms
- Prober timing out after 90 seconds
- All requests return 403 Forbidden
- Only affects AP2 datacenter

### Investigation Steps

1. **Ran test script** → Identified Step 4 failure (probe_smart_edge)
2. **Checked authenticator logs** → Found "invalid key pair resolved"
3. **Extracted org IDs** → API key: 1500000002, App key: 1500093531
4. **Root cause** → Org mismatch!

### Fix Applied

```yaml
# Updated ap2.prod.dog.yaml
probes:
  smart_edge_fresh_app_key_latency_probe:
    orgId: 1500000002  # Changed from 1500093531 to match API key org
```

### Verification

```bash
# After fix, prober should succeed in <10s
kubectl logs -n authenticator deployment/authn-prober --context emolga-a.ap2.prod.dog | \
  grep "latency_sec" | \
  grep -v "90"  # Should see values < 10
```

---

## Useful Commands

### Prober Management

```bash
# Check prober deployment
kubectl get deployment -n authenticator authn-prober --context {cluster}

# View prober logs
kubectl logs -n authenticator deployment/authn-prober --context {cluster} -f

# Restart prober
kubectl rollout restart deployment/authn-prober -n authenticator --context {cluster}
```

### ACE Key Operations

```bash
# Port-forward to ACE
POD=$(kubectl get pod -n ace -l service=ace --context {cluster} -o=jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n ace $POD 1640 --context {cluster}

# Create key (in another terminal)
grpcurl -H "$(ddtool auth token ace --datacenter {dc} --http-header)" \
  -H "x-datadog-target-release: ace.ace" \
  -d '{
    "org_id":1500093531,
    "org_uuid":"e9146dd9-adc0-11f0-8d6e-e2db00f6adb5",
    "owner_id":861628,
    "owner_uuid":"23b03ff3-adc1-11f0-b549-82f6ecb04496",
    "name":"test-key"
  }' \
  -plaintext \
  -proto ~/dd/dd-source/domains/aaa/apps/ace/ace_pb/ace.proto \
  -import-path ~/dd/dd-source/domains/aaa/apps/ace/ace_pb \
  localhost:1640 \
  pb.AuthenticationCredentialEnforcer.CreateCustomerApplicationKey
```

### Manual API Testing

```bash
# Test with /api/v1/validate (lenient)
curl "https://api.{dc}.datadoghq.com/api/v1/validate" \
  -H "DD-API-KEY: {api_key}" \
  -H "DD-APPLICATION-KEY: {app_key}"

# Test with /probe_smart_edge (strict)
curl "https://api.{dc}.datadoghq.com/probe_smart_edge?debug=1" \
  -H "DD-API-KEY: {api_key}" \
  -H "DD-APPLICATION-KEY: {app_key}" \
  -H "Authorization: Bearer {bearer_token}"
```

---

## Datacenter Comparison

### Working Datacenters

| DC | Prober Org | API Key Org | Status |
|----|-----------|-------------|--------|
| AP1 | 1400025899 | 1400025899 | ✅ Working |
| US1 | TBD | TBD | ✅ Working |

### Failing Datacenters

| DC | Prober Org | API Key Org | Issue |
|----|-----------|-------------|-------|
| AP2 | 1500093531 | 1500000002 | ❌ Org mismatch |

### How to Compare

```bash
# Get AP1 config
cat ~/dd/dd-source/domains/aaa_authn/apps/authn-prober/config/k8s/endpoints/ap1.prod.dog.yaml

# Get AP2 config
cat ~/dd/dd-source/domains/aaa_authn/apps/authn-prober/config/k8s/endpoints/ap2.prod.dog.yaml

# Compare orgIds
diff <(grep orgId ap1.prod.dog.yaml) <(grep orgId ap2.prod.dog.yaml)
```

---

## Key Metrics to Monitor

### Prober Health

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `latency_slo.probe.latency_sec` | End-to-end latency | > 10s (SLA) |
| `latency_slo.probe.error` | Probe failures | > 0 (continuous) |
| `latency_slo.probe` with `code:403` | Auth failures | > 5 in 10min |

### Context Propagation

| Metric | Description | Expected |
|--------|-------------|----------|
| `dd.authdatastore.app_key_count` | Keys in context | DC-specific (AP2: ~11k, AP1: ~470k) |
| `dd.authenticator.libcontext.key_fetch.error` | Fetch errors | < 10/s |

### ACE Operations

| Metric | Description | Expected |
|--------|-------------|----------|
| `dd.ace.create_app_key.duration` | Key creation time | < 50ms |
| `dd.ace.context_change.notify` | Notification rate | Matches creation rate |

---

## Troubleshooting Guide

### Issue: Prober times out after 90 seconds

**Check 1:** Run test script
```bash
bash ~/test_key_propagation_chain.sh
```

**Check 2:** Review authenticator logs for org mismatch
```bash
kubectl logs -n authenticator deployment/authenticator --context {cluster} | \
  grep "{prober_org_id}" | \
  grep "invalid key pair"
```

**Check 3:** Verify Vault API key org
```bash
vault kv get kv/data/k8s/authenticator/authn-prober/smart-edge-keys
```

**Fix:** Ensure orgId in config matches API key's org.

### Issue: Keys not propagating to authenticator

**Check 1:** Verify ACE sends notifications
```bash
kubectl logs -n ace deployment/ace --context {cluster} | \
  grep "Context change notification sent" | \
  tail -10
```

**Check 2:** Verify event-context-writer processes them
```bash
kubectl logs -n logs-general deployment/event-context-writer-default --context {cluster} | \
  grep "EDGE_APP_KEY_CONTEXT"
```

**Check 3:** Query org's home_datacenter
```sql
SELECT org_id, name, home_datacenter
FROM organizations
WHERE org_id = {prober_org_id};
```

**Fix:** Ensure org's home_datacenter matches deployment datacenter.

### Issue: Bearer token rejected

**Check 1:** Verify Bearer token length
```bash
echo -n "{bearer_token}" | wc -c
# Should be exactly 40 characters
```

**Check 2:** Test without Bearer token
```bash
curl "https://api.{dc}.datadoghq.com/probe_smart_edge" \
  -H "DD-API-KEY: {api_key}" \
  -H "DD-APPLICATION-KEY: {app_key}"
# If works without Bearer: Token format issue
# If still fails: Different problem
```

**Fix:** Use 40-character application key as Bearer token, or remove requirement.

---

## Alerting

### Current Alerts

Monitor alerts for authn-prober in terraform-config:
```bash
cd ~/dd/terraform-config/aaa-authn/resources/authn-prober/
# Review monitor definitions
```

### Recommended Alerts

1. **Prober Timeout Alert**
   - Metric: `latency_slo.probe.latency_sec`
   - Threshold: > 10s for > 5 consecutive minutes
   - Severity: High

2. **Invalid Key Pair Alert**
   - Metric: Custom query on authenticator logs
   - Filter: `reason:"invalid key pair resolved"`
   - Threshold: > 10 occurrences in 10 minutes
   - Severity: Medium (indicates config issue)

3. **Context Propagation Alert**
   - Metric: `dd.authenticator.libcontext.key_fetch.error`
   - Filter: `error:not_found`, recent key creation
   - Threshold: > 100/min sustained
   - Severity: High

---

## Related Documentation

- **ACE Documentation:** `~/dd/dd-source/domains/aaa/apps/ace/CLAUDE.md`
- **Authenticator Code:** `~/dd/dd-go/apps/authenticator/`
- **AAA-AUTHN Confluence:** https://datadoghq.atlassian.net/wiki/spaces/AAAAUTHN/
- **Smart Edge Dashboards:** See CLAUDE.md for full list

---

## Appendix: Test Script

The `test_key_propagation_chain.sh` script systematically tests each component.

**Location:** `~/test_key_propagation_chain.sh`

**Configuration:**
- Cluster: `emolga-a.ap2.prod.dog`
- Org: From `ap2.prod.dog.yaml`
- API Key: From Vault `smart-edge-keys`
- Bearer Token: From `probe.yaml`

**Dependencies:**
- `ddtool` (cluster management)
- `grpcurl` (ACE gRPC calls)
- `find-key.sh` (database verification)
- `kubectl` (log access)
- `jq` (JSON parsing)

**Runtime:** ~60-90 seconds (includes 30s polling window)

**Output:** Clear pass/fail for each step + root cause diagnosis
