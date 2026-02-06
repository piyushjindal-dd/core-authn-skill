# Incident Investigation Guide

## IMPORTANT: Use Bits for Deep Telemetry Investigation

**For comprehensive incident investigation, use Bits agent** which has access to advanced APM tools including:
- `apm_trace_comparison` - Compare fast vs slow traces
- `apm_search_watchdog_stories` - AI-detected anomalies
- `apm_search_change_stories` - Deployment/feature flag changes
- `apm_trace_summary` - AI-generated trace analysis

**See also**: `~/.claude/skills/incident-replay/ENHANCED_INVESTIGATION.md` for:
- Deep telemetry investigation techniques (APM, logs, metrics)
- Systematic replay parameter extraction
- Fault injection parameter calculation

```bash
# Example: Deep investigation with Bits
bits -p "Investigate incident for service:terminator. Find error traces, compare with normal traces, check for deployments 15min before, search watchdog stories for anomalies" --yolo
```

---

## CRITICAL: Understand the Incident FIRST, Then Create Notebook

**DO NOT create a notebook with generic service health charts.** Generic charts (request counts, latency) rarely show the actual incident. Instead:

### Investigation Workflow (MANDATORY ORDER)

1. **FIRST: Search logs to understand what's actually happening**
   - Find the actual error messages
   - Identify affected hosts/services
   - Understand the failure mode

2. **SECOND: Query key metrics from relevant dashboards**
   - For HAProxy/intake/Smart Edge issues → See "MANDATORY: Dashboard Metrics" section below
   - Query `envoy.cluster.upstream_rq_timeout` for customer-facing impact
   - Query `envoy.cluster.internal.upstream_rq_5xx` for auth layer errors
   - Query `edge_prober_smart_edge.probe` for external health

3. **THEN: Create a notebook with INCIDENT-SPECIFIC data**
   - Charts should directly visualize THE INCIDENT
   - Include log-based charts showing the actual errors
   - Add metrics that are RELEVANT to the specific failure (from step 2)

4. **Document root cause, impact, and remediation**

### Common Mistakes to AVOID

| Mistake | Why It's Wrong | Do This Instead |
|---------|---------------|-----------------|
| Adding generic request/latency charts | They don't show what's broken | Add charts for the SPECIFIC error/failure |
| Using wrong metric names | Charts will be empty | ALWAYS validate metrics have data before adding |
| Creating notebook before understanding incident | Wastes time on irrelevant charts | Search logs FIRST to find the actual error |
| Focusing on Core AuthN metrics for non-AuthN issues | Misleading, shows nothing useful | Find metrics from the ACTUAL failing component |

### Logs-First Investigation

**Always start with logs to find the actual error:**

```bash
# Find errors for the affected service
search_datadog_logs: query: "service:<service-name> status:error"

# Get error count by host to identify affected infrastructure
analyze_datadog_logs:
  filter: "service:<service-name> status:error"
  sql_query: "SELECT host, count(*) as errors FROM logs GROUP BY host ORDER BY errors DESC"

# Find the actual error message patterns
search_datadog_logs: query: "service:<service-name> status:error" use_log_patterns: true
```

**The log messages will tell you:**
- What's actually failing (DNS? Connection? Timeout?)
- Which hosts are affected
- How long it's been happening
- What the root cause likely is

### Create Notebook via Datadog API

```bash
# Clean keys first
API_KEY=$(echo $DD_API_KEY | tr -d '\n')
APP_KEY=$(echo $DD_APP_KEY | tr -d '\n')

# Create the investigation notebook
curl -s -X POST "https://api.datadoghq.com/api/v1/notebooks" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY" \
  -d '{
    "data": {
      "type": "notebooks",
      "attributes": {
        "name": "Investigation: <MONITOR_NAME> - <DATE>",
        "time": {
          "live_span": "4h"
        },
        "cells": [
          {
            "type": "notebook_cells",
            "attributes": {
              "definition": {
                "type": "markdown",
                "text": "# Investigation: <MONITOR_NAME>\n\n**Date**: <DATE>\n**Monitor ID**: <MONITOR_ID>\n**Investigator**: <YOUR_NAME>\n\n## Summary\n\n_To be filled after investigation_\n\n## Root Cause\n\n_To be filled after investigation_"
              }
            }
          }
        ]
      }
    }
  }'
```

### Standard Charts to Add for Each Service

After creating the notebook, add relevant metric charts. Use `PATCH` to add cells:

```bash
# Add a timeseries chart to existing notebook
curl -s -X PATCH "https://api.datadoghq.com/api/v1/notebooks/<NOTEBOOK_ID>" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY" \
  -d '{
    "data": {
      "type": "notebooks",
      "attributes": {
        "cells": [
          <EXISTING_CELLS>,
          <NEW_CELL>
        ]
      }
    }
  }'
```

#### Terminator Investigation Charts

```json
[
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "markdown",
        "text": "## Terminator Health Metrics"
      }
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "sum:dd.terminator.request{*} by {datacenter,status}.as_count()", "display_type": "bars"}],
        "title": "Terminator Requests by Datacenter and Status"
      },
      "graph_size": "l"
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "avg:dd.terminator.latency{*} by {datacenter}", "display_type": "line"}],
        "title": "Terminator Latency by Datacenter"
      },
      "graph_size": "l"
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "min:dd.terminator.current_key_seconds_left{*} by {datacenter}", "display_type": "line"}],
        "title": "Key Freshness by Datacenter (seconds until expiry)"
      },
      "graph_size": "l"
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "sum:dd.terminator.rotate_keys{*} by {datacenter,result}.as_count()", "display_type": "bars"}],
        "title": "Key Rotation Events by Datacenter"
      },
      "graph_size": "l"
    }
  }
]
```

#### Authenticator Investigation Charts

```json
[
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "markdown",
        "text": "## Authenticator Health Metrics"
      }
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "sum:dd.authenticator.request{*} by {datacenter,status}.as_count()", "display_type": "bars"}],
        "title": "Authenticator Requests by Datacenter and Status"
      },
      "graph_size": "l"
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "avg:dd.authenticator.latency{*} by {datacenter}", "display_type": "line"}],
        "title": "Authenticator Latency by Datacenter"
      },
      "graph_size": "l"
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "sum:dd.authenticator.request{status:error} by {datacenter,auth_error}.as_count()", "display_type": "bars"}],
        "title": "Authenticator Errors by Type"
      },
      "graph_size": "l"
    }
  }
]
```

#### Authenticator-Intake Investigation Charts

**IMPORTANT: The correct metric prefix is `dd.authenticator.intake.*` (NOT `dd.intake_authenticator.*`)**

```json
[
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "markdown",
        "text": "## Authenticator-Intake Health Metrics"
      }
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "sum:dd.authenticator.intake.request{*} by {datacenter,status}.as_count()", "display_type": "bars"}],
        "title": "Intake Requests by Datacenter and Status"
      },
      "graph_size": "l"
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "avg:dd.authenticator.intake.latency{*} by {datacenter}", "display_type": "line"}],
        "title": "Intake Latency by Datacenter"
      },
      "graph_size": "l"
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "sum:dd.authenticator.intake.api_key_resolution{*} by {datacenter,status}.as_count()", "display_type": "bars"}],
        "title": "API Key Resolution by Datacenter and Status"
      },
      "graph_size": "l"
    }
  }
]
```

#### OBO Investigation Charts

```json
[
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "markdown",
        "text": "## OBO Health Metrics"
      }
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "sum:dd.obo.server.response{*} by {datacenter,code}.as_count()", "display_type": "bars"}],
        "title": "OBO Responses by Datacenter and Code"
      },
      "graph_size": "l"
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "avg:dd.obo.ouiSvc.latency.ms{*} by {datacenter}", "display_type": "line"}],
        "title": "OBO Latency by Datacenter"
      },
      "graph_size": "l"
    }
  }
]
```

### Adding Findings to Notebook

After investigation, update the notebook with findings:

```bash
# Add a findings markdown cell
curl -s -X PATCH "https://api.datadoghq.com/api/v1/notebooks/<NOTEBOOK_ID>" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY" \
  -d '{
    "data": {
      "type": "notebooks",
      "attributes": {
        "cells": [
          ... existing cells ...,
          {
            "type": "notebook_cells",
            "attributes": {
              "definition": {
                "type": "markdown",
                "text": "## Investigation Findings\n\n### Root Cause\n<ROOT_CAUSE_DESCRIPTION>\n\n### Impact\n- Affected datacenters: <LIST>\n- Duration: <TIME_RANGE>\n- Error rate: <PERCENTAGE>\n\n### Remediation\n1. <IMMEDIATE_ACTION>\n2. <LONG_TERM_FIX>\n\n### Timeline\n| Time | Event |\n|------|-------|\n| <TIME> | <EVENT> |"
              }
            }
          }
        ]
      }
    }
  }'
```

### Notebook URL Format

After creation, the notebook URL will be:
```
https://app.datadoghq.com/notebook/<NOTEBOOK_ID>
```

**ALWAYS include this URL in your investigation summary.**

---

## Parallel Investigation Approach

When investigating a monitor alert, incident, or production issue, use **parallel agents** to gather data from multiple sources simultaneously.

### CRITICAL: Always Use Both MCP and API

**Always query Datadog using BOTH the MCP tools AND the REST API in parallel.** MCP tools may have permission limitations or return incomplete data. The REST API uses your personal credentials with full access.

```bash
# Always clean API keys before use (remove newlines)
API_KEY=$(echo $DD_API_KEY | tr -d '\n')
APP_KEY=$(echo $DD_APP_KEY | tr -d '\n')
```

### Phase 0: Find Related Incidents and Alerting Monitors

**Before diving into metrics/logs, always check for related incidents and alerting monitors.** An existing incident may already explain the root cause.

#### Core AuthN Services

Always search for issues across ALL Core AuthN services:

| Service | Monitor/Incident Keywords | Datadog Service Name |
|---------|--------------------------|---------------------|
| authenticator | `authenticator`, `authn` | `authenticator` |
| authenticator-intake | `authenticator-intake`, `intake-authenticator`, `intake` | `intake-authenticator` |
| terminator | `terminator` | `terminator` |
| obo | `obo`, `on-behalf-of` | `obo` |

#### Search for Alerting Monitors

```bash
# Search for alerting monitors across all Core AuthN services
search_datadog_monitors: query: "status:alert (authenticator OR terminator OR obo OR intake)"

# Search for monitors by service tag
search_datadog_monitors: query: "status:alert tag:service:authenticator"
search_datadog_monitors: query: "status:alert tag:service:terminator"
search_datadog_monitors: query: "status:alert tag:service:obo"
search_datadog_monitors: query: "status:alert tag:service:intake-authenticator"
```

#### Search for Related Incidents

```bash
# Search for active incidents related to Core AuthN services
search_datadog_incidents: query: "state:(active OR stable) AND (title:*terminator* OR title:*authenticator* OR title:*obo* OR title:*intake*)"

# Also search for recently resolved incidents that might be related
search_datadog_incidents: query: "state:resolved" from: "now-24h"

# Search by severity for critical issues
search_datadog_incidents: query: "state:(active OR stable) AND (severity:SEV-1 OR severity:SEV-2)"
```

**Include incident findings in your investigation summary** - they often reveal the root cause faster than log analysis.

### Phase 1: Parallel Data Collection

Spawn agents to investigate these areas concurrently:

#### Agent 1: Dashboards
Check the relevant dashboards for anomalies:

| Service | Primary Dashboard | Additional |
|---------|------------------|------------|
| authenticator | https://app.datadoghq.com/dashboard/3gw-hz2-up5 | [Health](https://app.datadoghq.com/dashboard/6ft-5re-qfc), [WPA](https://app.datadoghq.com/dashboard/ee2-dzk-qx3) |
| authenticator-intake | https://app.datadoghq.com/dashboard/xqk-uux-yq6 | [Shadow](https://app.datadoghq.com/dashboard/ywz-8wm-477), [libcontext x Auth](https://app.datadoghq.com/dashboard/ws9-gkw-bqe) |
| terminator | https://app.datadoghq.com/dashboard/kbv-5mz-8e6 | [AuthZ](https://app.datadoghq.com/dashboard/vye-tm6-7cd), [WPA](https://app.datadoghq.com/dashboard/ikz-ka7-dig) |
| obo | https://app.datadoghq.com/dashboard/vkx-5pc-ftz | [Client Cache](https://app.datadoghq.com/dashboard/2hm-f3s-32d) |

**CRITICAL for Authenticator-Intake / libcontext issues**: Always check BOTH of these dashboards:
- **Intake AuthN**: https://app.datadoghq.com/dashboard/xqk-uux-yq6 (intake metrics, API key resolution, latency)
- **libcontext x Auth**: https://app.datadoghq.com/dashboard/ws9-gkw-bqe (context platform health, blob service, cache metrics)

**CRITICAL for HAProxy/Intake issues**: Always check BOTH of these dashboards:
- **Smart Edge V1.2**: https://app.datadoghq.com/dashboard/sdz-xrb-ynf (envoy 5xx, ext_authz errors, probers)
- **Intake AuthN**: https://app.datadoghq.com/dashboard/xqk-uux-yq6 (upstream timeouts, intake metrics)

See "MANDATORY: Dashboard Metrics for HAProxy/Intake/Smart Edge Incidents" section for specific metrics to query.

#### Agent 2: Confluence
Search the AAA-AUTHN space for:
- Runbooks for the affected service
- Known issues and past incidents
- Architecture docs explaining the affected component

**Key runbook**: https://datadoghq.atlassian.net/wiki/spaces/AAAAUTHN/pages/4195909715/Authenticator+Intake+Runbooks

#### Agent 3: Code Analysis
Examine the service code to understand:
- The logic in the affected code path
- Error handling and failure modes
- Recent changes that might have caused the issue

#### Agent 4: Datadog MCP Tools
Query using Datadog MCP tools (run in PARALLEL with Agent 5):
```
# Search for alerting monitors across Core AuthN services
search_datadog_monitors: query: "status:alert (authenticator OR terminator OR obo OR intake)"

# Find related incidents
search_datadog_incidents: query: "state:(active OR stable) AND (title:*terminator* OR title:*authenticator* OR title:*obo* OR title:*intake*)"

# Search for errors
search_datadog_logs: service:<service-name> status:error

# Find related traces
search_datadog_spans: service:<service-name> status:error

# Get metrics (ALWAYS group by datacenter)
get_datadog_metric: queries: ["sum:dd.<service>.request{*} by {datacenter}.as_count()"]
```

#### Agent 5: Datadog REST API (ALWAYS run in parallel with MCP)
**CRITICAL**: Always run API queries alongside MCP - MCP may have permission limitations.

```bash
# Clean keys first
API_KEY=$(echo $DD_API_KEY | tr -d '\n')
APP_KEY=$(echo $DD_APP_KEY | tr -d '\n')

# Search for alerting monitors (Core AuthN services)
curl -s "https://api.datadoghq.com/api/v1/monitor?query=status:alert" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY" | jq '.[] | select(.name | test("authenticator|terminator|obo|intake"; "i"))'

# Search incidents
curl -s "https://api.datadoghq.com/api/v2/incidents?filter[state]=active" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY"

# Search logs
curl -s -X POST "https://api.datadoghq.com/api/v2/logs/events/search" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filter": {"query": "service:<service-name> status:error", "from": "now-1h"}}'

# Query metrics (ALWAYS group by datacenter)
curl -s "https://api.datadoghq.com/api/v1/query?query=sum:dd.<service>.request{*}by{datacenter}.as_count()&from=$(date -v-1H +%s)&to=$(date +%s)" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY"
```

**If keys are expired**, run `dd-auth` to refresh them.

### Phase 2: Synthesis

After parallel investigation completes, combine insights to:
1. **Identify root cause** - Correlate metrics spikes with code paths
2. **Find the blast radius** - Which orgs/users are affected?
3. **Locate runbook steps** - What's the remediation procedure?
4. **Propose actions** - Immediate mitigation + long-term fix

## Common Investigation Patterns

### Authentication Failures Spike

1. Check `dd.authenticator.request` metric with `status:error` tag
2. Look for `auth_error` field in logs to identify failure type
3. Check if specific credential types are affected (API key vs session vs PAT)
4. Verify database connectivity to credential clusters

### JWT Generation Failures

1. Check `dd.terminator.jwt_provider.cache` metrics
2. Look for key rotation issues in `dd.terminator.well_known_keys`
3. Verify PKI service health
4. Check for memory pressure if caching is involved

### OBO Token Exchange Failures

1. Check `dd.obo.server.response` with error codes
2. Verify K8s identity resolution is working
3. Check allowlist configuration in Consul
4. Look for permission intersection issues

### Latency Increases

1. Check distribution metrics: `dd.*.latency`
2. Look for database query slowdowns
3. Check cache hit rates
4. Verify feature flag changes that might affect code paths

### libcontext / Context Platform Issues (Authenticator-Intake)

When investigating issues with context platform, libcontext, or authenticator-intake resolver initialization:

**Key Dashboards:**
- **Intake AuthN**: https://app.datadoghq.com/dashboard/xqk-uux-yq6
- **libcontext x Auth**: https://app.datadoghq.com/dashboard/ws9-gkw-bqe

**Investigation Steps:**

1. **Check resolver initialization status:**
   - Metric: `dd.authenticator.intake.resolver_init` (status:success/failure, type:api_key/client_token)
   - Metric: `dd.authenticator.intake.resolver_init_attempt` (retry attempts)

2. **Check context platform blob service health:**
   - Dashboard: [libcontext x Auth](https://app.datadoghq.com/dashboard/ws9-gkw-bqe)
   - Look for blob fetch failures, cache misses, replication delays

3. **Check API key resolution:**
   - Metric: `dd.authenticator.intake.api_key_resolution` with `status:unavailable` (resolver not initialized)
   - Metric: `dd.context_resolver.token_hashes.unknown` (unrecognized credentials)

4. **Check for context update replication delays:**
   - Metric: `dd.authdatastore.replication_delay_in_milliseconds`
   - Delays > 30s indicate context propagation issues

**Important Code Context:**
- Resolver initialization is **non-blocking** - sidecar starts without waiting for libcontext
- After 30s grace period, readiness probe returns OK regardless of resolver state
- Key resolution returns `AuthenticationError` if resolver not initialized
- See `~/dd/dd-go/apps/authenticator-intake/resolver/credential_resolver.go` for retry logic
- See `~/dd/dd-go/pkg/authdatastore/context_datastore.go:44-46` for blocking behavior

**Log Queries:**
```
# Resolver initialization errors
service:authenticator-intake "resolver cannot start"

# Context platform errors
service:authenticator-intake "Context is not initialized"

# Unknown credentials (may indicate context sync issues)
service:authenticator-intake "Unknown credential"
```

### Context Platform Metrics (logs.contextloader.*)

These metrics provide deep visibility into the context platform health. Use them when investigating authenticator-intake issues related to API key resolution or context synchronization.

**IMPORTANT: Filtering for Authenticator-Intake**

The context platform (libcontext/frames) is used by many services. To filter specifically for authenticator-intake:

| Filter | Value | Description |
|--------|-------|-------------|
| `context_type` | `edge_api_key_context` | **PRIMARY** - API key resolution |
| `context_type` | `edge_client_token_context` | Client token resolution (RUM) |
| `app` | `authenticator` | Authenticator service (includes intake sidecar) |
| `app` | `authenticator-intake-shadow` | Shadow traffic service |
| `short_image` | `authenticator-intake` | Intake container image |

**Context Types for Core AuthN:**
- `edge_api_key_context` - API keys (used by intake for ingest traffic)
- `edge_client_token_context` - Client tokens (RUM traffic)
- `edge_app_key_context` - Application keys
- `edge_pat_context` - Personal access tokens

**Key tags for filtering:**
- `app` - Service name (e.g., `authenticator`, `authenticator-intake-shadow`, `ingress-edge-evp-pool3`)
- `datacenter` - Datacenter (e.g., `us1.prod.dog`, `eu1.prod.dog`)
- `context_type` - Type of context being loaded (see above)

#### Health & Status Metrics

| Metric | Description | Use Case |
|--------|-------------|----------|
| `logs.contextloader.client.blobmanager.single_context_updated` | Individual context updates received | Baseline activity |
| `logs.contextloader.client.blobmanager.context_not_found` | **CRITICAL** - Context lookup failures | Missing API keys, sync issues |
| `logs.contextloader.client.blobmanager.all_context_updated` | Full context refresh events | Bulk updates |
| `logs.contextloader.client.blobmanager.key_invalidated` | Individual key invalidations | Key rotation/deletion |
| `logs.contextloader.client.blobmanager.all_keys_invalidated` | Bulk key invalidations | Mass key rotation |
| `logs.contextloader.client.blobmanager.snapshot_built` | Snapshot build completions | Initial load health |

#### Latency Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `logs.contextloader.client.blobmanager.context_loading.latency.avg` | Avg context loading time | > 1000ms |
| `logs.contextloader.client.blobmanager.context_loading.latency.99percentile` | p99 context loading time | > 5000ms |
| `logs.contextloader.client.blobmanager.blob_update.latency.avg` | Avg blob update time | > 500ms |
| `logs.contextloader.client.blobmanager.context_propagation.latency` | End-to-end propagation delay | > 10000ms |
| `logs.contextloader.client.blobmanager.seek_and_replay.latency.avg` | Avg seek/replay time | > 2000ms |
| `logs.contextloader.client.blobmanager.context_update.process_time.avg` | Update processing time | > 100ms |

#### Freshness Metrics (Staleness Detection)

| Metric | Description | Alert If |
|--------|-------------|----------|
| `logs.contextloader.client.blobmanager.ms_since_blob_creation` | Time since blob was created | > 60000ms |
| `logs.contextloader.client.blobmanager.ms_since_blob_download` | Time since last blob download | > 120000ms |
| `logs.contextloader.client.blobmanager.ms_since_snapshot_updated` | Time since last snapshot update | > 300000ms |
| `logs.contextloader.client.blobmanager.ms_since_successful_seek_and_replay` | Time since last successful replay | > 600000ms |

#### Working Set Metrics

| Metric | Description | Use Case |
|--------|-------------|----------|
| `logs.contextloader.client.blobmanager.working_set.size` | Current working set size | Capacity monitoring |
| `logs.contextloader.client.blobmanager.working_set.status` | Working set health status | Health check |
| `logs.contextloader.client.blobmanager.working_set.invalidation_cycle.done.stale` | Stale entries in working set | Data freshness |
| `logs.contextloader.client.blobmanager.working_set.invalidation_cycle.done.missing` | Missing entries in working set | Sync issues |

#### Download & Presigner Metrics

| Metric | Description | Use Case |
|--------|-------------|----------|
| `logs.contextloader.client.downloader.download` | Blob download count | Download activity |
| `logs.contextloader.client.downloader.download.elapsed.avg` | Avg download duration | Network issues |
| `logs.contextloader.client.downloader.presigner.call` | Presigner API calls | Presigner health |
| `logs.contextloader.client.download_size` | Downloaded blob sizes | Bandwidth usage |

#### Publisher/Provider Metrics (Upstream Health)

| Metric | Description | Use Case |
|--------|-------------|----------|
| `logs.contextloader.publisher.publish.success` | Successful context publishes | Publisher health |
| `logs.contextloader.publisher.publish.failure` | **CRITICAL** - Failed publishes | Publisher failures |
| `logs.contextloader.provider.build.success` | Successful context builds | Provider health |
| `logs.contextloader.provider.build.failure` | **CRITICAL** - Failed builds | Provider failures |
| `logs.contextloader.poller.success` | Successful poll operations | Poller health |
| `logs.contextloader.poller.failure` | Failed poll operations | Poller issues |

#### Example Investigation Queries

```python
# Check context update health for authenticator-intake (API key resolution)
get_datadog_metric:
  queries:
    - "sum:logs.contextloader.client.blobmanager.single_context_updated{context_type:edge_api_key_context} by {datacenter}.as_count()"
    - "sum:logs.contextloader.client.blobmanager.context_not_found{context_type:edge_api_key_context} by {datacenter}.as_count()"
  from: "now-1h"

# Check client token context health (RUM traffic)
get_datadog_metric:
  queries:
    - "sum:logs.contextloader.client.blobmanager.single_context_updated{context_type:edge_client_token_context} by {datacenter}.as_count()"
    - "sum:logs.contextloader.client.blobmanager.context_not_found{context_type:edge_client_token_context} by {datacenter}.as_count()"
  from: "now-1h"

# Check freshness - are API key contexts stale?
get_datadog_metric:
  queries:
    - "max:logs.contextloader.client.blobmanager.ms_since_blob_download{context_type:edge_api_key_context} by {datacenter}"
    - "max:logs.contextloader.client.blobmanager.ms_since_snapshot_updated{context_type:edge_api_key_context} by {datacenter}"
  from: "now-1h"

# Check for publisher failures for API key contexts (upstream issue)
get_datadog_metric:
  queries:
    - "sum:logs.contextloader.publisher.publish.failure{context_type:edge_api_key_context} by {datacenter}.as_count()"
    - "sum:logs.contextloader.publisher.publish.failure{context_type:edge_client_token_context} by {datacenter}.as_count()"
  from: "now-1h"

# Check download latency for authenticator service
get_datadog_metric:
  queries:
    - "avg:logs.contextloader.client.downloader.download.elapsed.avg{app:authenticator} by {datacenter}"
    - "avg:logs.contextloader.client.blobmanager.context_loading.latency.avg{app:authenticator} by {datacenter}"
  from: "now-1h"

# Check working set health for API key contexts
get_datadog_metric:
  queries:
    - "sum:logs.contextloader.client.blobmanager.working_set.invalidation_cycle.done.stale{context_type:edge_api_key_context} by {datacenter}.as_count()"
    - "sum:logs.contextloader.client.blobmanager.working_set.invalidation_cycle.done.missing{context_type:edge_api_key_context} by {datacenter}.as_count()"
  from: "now-1h"

# All edge contexts (API key + client token + app key + PAT)
get_datadog_metric:
  queries:
    - "sum:logs.contextloader.client.blobmanager.context_not_found{context_type:edge_*} by {context_type,datacenter}.as_count()"
  from: "now-1h"
```

#### Troubleshooting Decision Tree

```
Authenticator-Intake Context Platform Issue?
│
│  FIRST: Filter by context_type:edge_api_key_context or context_type:edge_client_token_context
│
├─ High `context_not_found{context_type:edge_api_key_context}`?
│  └─ Check: Is context being published?
│     │   Query: logs.contextloader.publisher.publish.failure{context_type:edge_api_key_context}
│     └─ Failures → Publisher issue (check ACE contexts service)
│     └─ No failures → Check: Is blob being downloaded?
│        │   Query: logs.contextloader.client.downloader.download{app:authenticator}
│        └─ No downloads → Network/presigner issue
│        └─ Downloads OK → Check: Is working set healthy?
│           │   Query: working_set.invalidation_cycle.done.missing{context_type:edge_api_key_context}
│           └─ High missing → Sync issue between publisher and consumer
│
├─ High latency for API key resolution?
│  └─ Check: Download latency
│     │   Query: logs.contextloader.client.downloader.download.elapsed.avg{app:authenticator}
│     └─ High → Network/S3 issue
│     └─ Normal → Check: Context loading latency
│        │   Query: logs.contextloader.client.blobmanager.context_loading.latency.avg{app:authenticator}
│        └─ High → CPU/memory pressure on authenticator pods
│
├─ Stale API key data?
│  └─ Check: ms_since_blob_download{context_type:edge_api_key_context}
│     └─ > 120000ms → Check poller health
│        │   Query: logs.contextloader.poller.failure
│        └─ Poller failing → Network/Kafka connectivity issue
│
└─ New API keys not recognized?
   └─ Check: context_propagation.latency{context_type:edge_api_key_context}
      └─ High → Publisher backlog or Kafka lag in ACE contexts
```

**Quick Reference - Context Types:**
| Issue | Filter |
|-------|--------|
| API key not found | `context_type:edge_api_key_context` |
| Client token not found | `context_type:edge_client_token_context` |
| App key not found | `context_type:edge_app_key_context` |
| PAT not found | `context_type:edge_pat_context` |

### Postgres Database Issues

For database connectivity, query performance, or postgres-related issues affecting auth services:

1. **Use the Postgres Investigation Notebook**: https://app.datadoghq.com/notebook/13828716
2. Check `dd.authenticator.db.*` metrics for query latency and errors
3. Look for `ErrPgReadTimeout` errors in logs
4. Verify credential cluster health
5. Check connection pool exhaustion

**Common postgres-related errors:**
- `ErrDataNotFound` - Record doesn't exist
- `ErrPgReadTimeout` - Query timeout (check load/indexes)
- Connection pool exhaustion - Check active connections

## Key Metrics to Check

| Service | Health Metrics |
|---------|---------------|
| authenticator | `dd.authenticator.request`, `dd.authenticator.latency` |
| authenticator-intake | `dd.authenticator.intake.request`, `dd.authenticator.intake.latency` |
| terminator | `dd.terminator.request`, `dd.terminator.latency`, `dd.terminator.jwt_provider.cache` |
| obo | `dd.obo.server.response`, `dd.obo.ouiSvc.latency.ms` |

**WARNING**: These are generic health metrics. They may NOT show the actual incident!
Always search logs first to understand what's failing, then find metrics specific to that failure.

---

## MANDATORY: Dashboard Metrics for HAProxy/Intake/Smart Edge Incidents

When investigating incidents involving `ingress-haproxy-*`, `authenticator-intake`, or Smart Edge, you **MUST** query metrics from these two dashboards:

### 1. Smart Edge V1.2 Dashboard
**URL**: https://app.datadoghq.com/dashboard/sdz-xrb-ynf/smart-edge-v12

| Metric | Description | When to Use |
|--------|-------------|-------------|
| `envoy.cluster.internal.upstream_rq_5xx{envoy_cluster:authenticator*}` | Authenticator 5xx errors | Authentication failures |
| `envoy.cluster.internal.upstream_rq_5xx{envoy_cluster:terminator*}` | Terminator 5xx errors | JWT/authorization failures |
| `envoy.cluster.ext_authz.ok{stat_prefix:authenticator}` | Successful authentications | Baseline comparison |
| `envoy.cluster.ext_authz.error{stat_prefix:authenticator}` | Auth errors (timeouts, failures) | **KEY INCIDENT METRIC** |
| `envoy.cluster.ext_authz.denied{stat_prefix:authenticator}` | Auth denials | Permission issues |
| `edge_prober_smart_edge.probe{type:success}` | Successful probes | External health |
| `edge_prober_smart_edge.probe{!type:success}` | Failed probes | **EXTERNAL IMPACT** |

### 2. Intake AuthN Dashboard
**URL**: https://app.datadoghq.com/dashboard/xqk-uux-yq6/intake-auth-n

| Metric | Description | When to Use |
|--------|-------------|-------------|
| **`envoy.cluster.upstream_rq_timeout{envoy_cluster:local_cluster_8280}`** | **PRIMARY IMPACT METRIC** - Request timeouts | **ALWAYS CHECK FIRST** |
| `envoy.cluster.upstream_cx_connect_timeout` | Connection timeouts | Network/DNS issues |
| `envoy.cluster.upstream_cx_overflow` | Connection pool overflow | Capacity issues |
| `dd.authenticator.intake.request` | Intake requests by status | Traffic patterns |
| `dd.authenticator.intake.latency` | Intake latency | Performance issues |
| `dd.authenticator.intake.api_key_resolution` | API key lookups | Key resolution issues |

### Investigation Workflow for HAProxy/Intake Incidents

**MANDATORY STEPS** when investigating any `ingress-haproxy-*` or Smart Edge incident:

```python
# Step 1: Query the PRIMARY IMPACT metric first
get_datadog_metric:
  queries:
    - "sum:envoy.cluster.upstream_rq_timeout{service:ingress-haproxy-api,envoy_cluster:local_cluster_8280,datacenter:$DC}.as_count()"
  from: "<incident_start>"
  to: "<incident_end>"

# Step 2: Query authenticator errors from Smart Edge
get_datadog_metric:
  queries:
    - "sum:envoy.cluster.internal.upstream_rq_5xx{service:ingress-*,envoy_cluster:authenticator*,datacenter:$DC}.as_count()"
    - "sum:envoy.cluster.ext_authz.error{service:ingress-*,stat_prefix:authenticator,datacenter:$DC}.as_count()"
  from: "<incident_start>"
  to: "<incident_end>"

# Step 3: Query external prober failures
get_datadog_metric:
  queries:
    - "sum:edge_prober_smart_edge.probe{datacenter:$DC,isprivate:false,!type:success} by {endpoint}.as_count()"
  from: "<incident_start>"
  to: "<incident_end>"

# Step 4: Search logs for the actual error
search_datadog_logs:
  query: "service:ingress-haproxy-api status:error"
  from: "<incident_start>"
  to: "<incident_end>"
```

### Example: DNS Failure Incident

In the 2026-02-04 incident, these metrics revealed the impact:

| Metric | Value | What It Showed |
|--------|-------|----------------|
| `envoy.cluster.upstream_rq_timeout` | **14.3M timeouts** | PRIMARY customer impact |
| `envoy.cluster.internal.upstream_rq_5xx` | 65K errors | Cascading auth failures |
| `envoy.cluster.ext_authz.error` | 65K errors | Auth layer impact |
| DNS error logs | 3,621 logs | Root cause (DNS failures) |

**Key Learning**: The `upstream_rq_timeout` metric showed the actual customer impact (14.3M timeouts), while logs revealed the root cause (DNS resolution failures).

### Notebook Charts for These Metrics

When creating incident notebooks, include these charts:

```json
[
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "sum:envoy.cluster.upstream_rq_timeout{service:ingress-haproxy-api,envoy_cluster:local_cluster_8280,datacenter:us1.prod.dog}.as_count()", "display_type": "bars", "style": {"palette": "warm"}}],
        "title": "Upstream Request Timeouts - PRIMARY IMPACT"
      },
      "graph_size": "l"
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "sum:envoy.cluster.internal.upstream_rq_5xx{service:ingress-*,envoy_cluster:authenticator*,datacenter:us1.prod.dog}.as_count()", "display_type": "bars", "style": {"palette": "warm"}}],
        "title": "Authenticator 5xx Errors"
      },
      "graph_size": "l"
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "sum:envoy.cluster.ext_authz.error{service:ingress-*,stat_prefix:authenticator,datacenter:us1.prod.dog}.as_count()", "display_type": "bars", "style": {"palette": "warm"}}],
        "title": "ext_authz Errors (Authenticator)"
      },
      "graph_size": "l"
    }
  },
  {
    "type": "notebook_cells",
    "attributes": {
      "definition": {
        "type": "timeseries",
        "requests": [{"q": "sum:edge_prober_smart_edge.probe{datacenter:us1.prod.dog,isprivate:false,!type:success} by {endpoint}.as_count()", "display_type": "bars", "style": {"palette": "warm"}}],
        "title": "Smart Edge Prober Failures"
      },
      "graph_size": "l"
    }
  }
]
```

### Related Metrics for Root Cause Analysis

| Root Cause | Metrics to Check |
|------------|------------------|
| DNS failures | Logs: `"dns error"`, Metric: `evp.workers.contexts.presigner.grpc_calls.failures` |
| Presigner issues | `evp.workers.contexts.presigner.grpc_calls`, `evp.workers.contexts.presigner.grpc_calls.failures` |
| Context loader failures | `logs.contextloader.client.blobmanager.single_context_updated_failure` |
| Connection pool exhaustion | `envoy.cluster.upstream_cx_overflow`, `envoy.cluster.upstream_rq_pending_active` |

## Useful Log Queries

```
# All errors for a service
service:authenticator status:error

# Specific auth method failures
service:terminator @cred_method:bearer_token status:error

# Org-specific issues
service:authenticator @org_id:12345

# Recent changes correlation
service:authenticator @auth_error:* | timeseries count by @auth_error
```

## Creating Investigation Notebooks

### CRITICAL: Incident-Specific Notebooks

**The notebook should document THE SPECIFIC INCIDENT, not generic service health.**

A good incident notebook contains:

1. **Clear incident summary** - What's happening, when it started, what's affected
2. **Root cause** - The actual error/failure (from logs!)
3. **Log-based charts** - Show the actual errors over time
4. **Relevant metrics** - Metrics that visualize THIS failure (not generic health)
5. **Affected hosts/services table** - Who's impacted
6. **Impact assessment** - What downstream effects?
7. **Remediation steps** - How to fix it

### Example: Log-Based Chart (Shows Actual Errors)

```json
{
  "type": "notebook_cells",
  "attributes": {
    "definition": {
      "type": "timeseries",
      "requests": [
        {
          "display_type": "bars",
          "queries": [
            {
              "data_source": "logs",
              "name": "query1",
              "search": {"query": "service:ingress-haproxy-api status:error \"dns error\""},
              "indexes": ["*"],
              "group_by": [{"facet": "host", "limit": 10, "sort": {"aggregation": "count", "order": "desc"}}],
              "compute": {"aggregation": "count"}
            }
          ],
          "response_format": "timeseries",
          "style": {"palette": "warm"}
        }
      ],
      "show_legend": true,
      "title": "DNS Error Count by Host",
      "type": "timeseries"
    },
    "graph_size": "l",
    "time": {"live_span": "12h"}
  }
}
```

### Always Group by Datacenter

All metrics should use `by {datacenter}` to show data split by datacenter. This is critical for identifying datacenter-specific issues.

**Correct metric queries:**
```
sum:dd.terminator.request{*} by {datacenter}.as_count()
sum:dd.obo.server.response{*} by {code,datacenter}.as_count()
avg:dd.authenticator.latency{*} by {datacenter}
```

**DO NOT use template variables like `$datacenter`** without proper configuration. Instead, use `*` with `by {datacenter}` grouping which always works.

### Notebook Creation via API

When creating notebooks programmatically, use this pattern:

```bash
API_KEY=$(echo $DD_API_KEY | tr -d '\n')
APP_KEY=$(echo $DD_APP_KEY | tr -d '\n')

curl -s -X POST "https://api.datadoghq.com/api/v1/notebooks" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY" \
  -d @notebook.json
```

**Required notebook cell structure:**
```json
{
  "type": "notebook_cells",
  "attributes": {
    "definition": {
      "type": "timeseries",
      "requests": [
        {
          "q": "sum:dd.terminator.request{*} by {datacenter}.as_count()",
          "display_type": "bars",
          "style": {"palette": "dog_classic"}
        }
      ],
      "title": "Requests by Datacenter"
    },
    "graph_size": "l"
  }
}
```

### CRITICAL: Always Validate After Creation

**After creating a notebook, ALWAYS validate it:**

1. **Query the notebook via API to verify structure:**
```bash
curl -s -X GET "https://api.datadoghq.com/api/v1/notebooks/<notebook_id>" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY" | jq '.data.attributes.cells'
```

2. **Verify metrics return data:**
```bash
# Use MCP tool
get_datadog_metric: queries: ["sum:dd.terminator.request{*}.as_count()"]

# AND verify via API
curl -s "https://api.datadoghq.com/api/v1/query?query=sum:dd.terminator.request{*}.as_count()&from=$(date -v-1H +%s)&to=$(date +%s)" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY"
```

3. **Check template variables are configured (if used):**
```bash
curl ... | jq '.data.attributes.template_variables'
# Should NOT be empty [] if you're using $datacenter in queries
```

### Pre-built Investigation Notebooks

| Service | Notebook |
|---------|----------|
| Terminator SLO | https://app.datadoghq.com/notebook/13835737 |
| OBO SLO | https://app.datadoghq.com/notebook/13835738 |
| Postgres Issues | https://app.datadoghq.com/notebook/13828716 |

## Investigation Summary Template

After completing parallel investigation, structure your findings:

```markdown
## Alert Summary
- **Monitor**: [Monitor name and link]
- **Duration**: When it started alerting
- **Related Incidents**: [List any active/recent incidents]
- **Investigation Notebook**: [Notebook URL]

## Root Cause
[Clear explanation of what's causing the alert]

## Impact by Datacenter
| Datacenter | Error Rate | Volume |
|------------|------------|--------|
| us1.prod.dog | X% | Y/min |
| eu1.prod.dog | X% | Y/min |

## Error Breakdown
[Top error reasons/codes]

## Recommended Actions
1. Immediate mitigation
2. Long-term fix
```

---

## Appendix: Complete Notebook Creation Scripts

### Quick Notebook Creation Function

Add this to your shell profile (~/.zshrc) for quick notebook creation:

```bash
# Create a Core AuthN investigation notebook
create_investigation_notebook() {
  local SERVICE=$1
  local MONITOR_NAME=$2
  local MONITOR_ID=$3
  local DATE=$(date +%Y-%m-%d)

  API_KEY=$(echo $DD_API_KEY | tr -d '\n')
  APP_KEY=$(echo $DD_APP_KEY | tr -d '\n')

  # Build cells based on service
  case $SERVICE in
    terminator)
      METRIC_CELLS='[
        {"type":"notebook_cells","attributes":{"definition":{"type":"markdown","text":"## Terminator Health Metrics"}}},
        {"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"sum:dd.terminator.request{*} by {datacenter,status}.as_count()","display_type":"bars"}],"title":"Requests by Datacenter"},"graph_size":"l"}},
        {"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"avg:dd.terminator.latency{*} by {datacenter}","display_type":"line"}],"title":"Latency by Datacenter"},"graph_size":"l"}},
        {"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"min:dd.terminator.current_key_seconds_left{*} by {datacenter}","display_type":"line"}],"title":"Key Freshness (seconds until expiry)"},"graph_size":"l"}},
        {"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"sum:dd.terminator.rotate_keys{*} by {datacenter,result}.as_count()","display_type":"bars"}],"title":"Key Rotation Events"},"graph_size":"l"}}
      ]'
      ;;
    authenticator)
      METRIC_CELLS='[
        {"type":"notebook_cells","attributes":{"definition":{"type":"markdown","text":"## Authenticator Health Metrics"}}},
        {"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"sum:dd.authenticator.request{*} by {datacenter,status}.as_count()","display_type":"bars"}],"title":"Requests by Datacenter"},"graph_size":"l"}},
        {"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"avg:dd.authenticator.latency{*} by {datacenter}","display_type":"line"}],"title":"Latency by Datacenter"},"graph_size":"l"}},
        {"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"sum:dd.authenticator.request{status:error} by {datacenter,auth_error}.as_count()","display_type":"bars"}],"title":"Errors by Type"},"graph_size":"l"}}
      ]'
      ;;
    authenticator-intake|intake)
      METRIC_CELLS='[
        {"type":"notebook_cells","attributes":{"definition":{"type":"markdown","text":"## Authenticator-Intake Health Metrics"}}},
        {"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"sum:dd.intake_authenticator.request{*} by {datacenter,status}.as_count()","display_type":"bars"}],"title":"Requests by Datacenter"},"graph_size":"l"}},
        {"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"avg:dd.intake_authenticator.latency{*} by {datacenter}","display_type":"line"}],"title":"Latency by Datacenter"},"graph_size":"l"}},
        {"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"sum:dd.intake_authenticator.cache.hit{*} by {datacenter}.as_count()","display_type":"line"}],"title":"Cache Hit Rate"},"graph_size":"l"}}
      ]'
      ;;
    obo)
      METRIC_CELLS='[
        {"type":"notebook_cells","attributes":{"definition":{"type":"markdown","text":"## OBO Health Metrics"}}},
        {"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"sum:dd.obo.server.response{*} by {datacenter,code}.as_count()","display_type":"bars"}],"title":"Responses by Datacenter and Code"},"graph_size":"l"}},
        {"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"avg:dd.obo.ouiSvc.latency.ms{*} by {datacenter}","display_type":"line"}],"title":"Latency by Datacenter"},"graph_size":"l"}}
      ]'
      ;;
    *)
      echo "Unknown service: $SERVICE. Use: terminator, authenticator, authenticator-intake, obo"
      return 1
      ;;
  esac

  RESPONSE=$(curl -s -X POST "https://api.datadoghq.com/api/v1/notebooks" \
    -H "Content-Type: application/json" \
    -H "DD-API-KEY: $API_KEY" \
    -H "DD-APPLICATION-KEY: $APP_KEY" \
    -d "{
      \"data\": {
        \"type\": \"notebooks\",
        \"attributes\": {
          \"name\": \"Investigation: ${MONITOR_NAME} - ${DATE}\",
          \"time\": {\"live_span\": \"4h\"},
          \"cells\": [
            {\"type\":\"notebook_cells\",\"attributes\":{\"definition\":{\"type\":\"markdown\",\"text\":\"# Investigation: ${MONITOR_NAME}\\n\\n**Date**: ${DATE}\\n**Monitor ID**: ${MONITOR_ID}\\n**Service**: ${SERVICE}\\n\\n## Summary\\n\\n_To be filled after investigation_\\n\\n## Root Cause\\n\\n_To be filled after investigation_\"}}},
            ${METRIC_CELLS:1:-1}
          ]
        }
      }
    }")

  NOTEBOOK_ID=$(echo $RESPONSE | jq -r '.data.id')
  echo "Created notebook: https://app.datadoghq.com/notebook/${NOTEBOOK_ID}"
  echo $NOTEBOOK_ID
}

# Usage:
# create_investigation_notebook terminator "Key Rotation Freshness" 105613271
# create_investigation_notebook authenticator "Auth Failures" 12345678
```

### One-Shot Notebook Creation (No Shell Function)

If you don't have the shell function, use this curl command directly:

```bash
# Replace these variables
SERVICE="terminator"
MONITOR_NAME="Key Rotation Freshness"
MONITOR_ID="105613271"
DATE=$(date +%Y-%m-%d)
API_KEY=$(echo $DD_API_KEY | tr -d '\n')
APP_KEY=$(echo $DD_APP_KEY | tr -d '\n')

# Create notebook with terminator charts
curl -s -X POST "https://api.datadoghq.com/api/v1/notebooks" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY" \
  -d "{
    \"data\": {
      \"type\": \"notebooks\",
      \"attributes\": {
        \"name\": \"Investigation: ${MONITOR_NAME} - ${DATE}\",
        \"time\": {\"live_span\": \"4h\"},
        \"cells\": [
          {\"type\":\"notebook_cells\",\"attributes\":{\"definition\":{\"type\":\"markdown\",\"text\":\"# Investigation: ${MONITOR_NAME}\\n\\n**Date**: ${DATE}\\n**Monitor ID**: ${MONITOR_ID}\\n**Service**: ${SERVICE}\\n\\n## Summary\\n\\n_To be filled_\\n\\n## Root Cause\\n\\n_To be filled_\"}}},
          {\"type\":\"notebook_cells\",\"attributes\":{\"definition\":{\"type\":\"markdown\",\"text\":\"## ${SERVICE^} Health Metrics\"}}},
          {\"type\":\"notebook_cells\",\"attributes\":{\"definition\":{\"type\":\"timeseries\",\"requests\":[{\"q\":\"sum:dd.${SERVICE}.request{*} by {datacenter,status}.as_count()\",\"display_type\":\"bars\"}],\"title\":\"Requests by Datacenter\"},\"graph_size\":\"l\"}},
          {\"type\":\"notebook_cells\",\"attributes\":{\"definition\":{\"type\":\"timeseries\",\"requests\":[{\"q\":\"avg:dd.${SERVICE}.latency{*} by {datacenter}\",\"display_type\":\"line\"}],\"title\":\"Latency by Datacenter\"},\"graph_size\":\"l\"}}
        ]
      }
    }
  }" | jq -r '"Created: https://app.datadoghq.com/notebook/" + .data.id'
```

### Adding Custom Charts to Existing Notebook

```bash
NOTEBOOK_ID="<your-notebook-id>"
API_KEY=$(echo $DD_API_KEY | tr -d '\n')
APP_KEY=$(echo $DD_APP_KEY | tr -d '\n')

# First, get existing cells
EXISTING=$(curl -s "https://api.datadoghq.com/api/v1/notebooks/${NOTEBOOK_ID}" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY" | jq '.data.attributes.cells')

# Add new chart (e.g., error rate)
NEW_CELL='{"type":"notebook_cells","attributes":{"definition":{"type":"timeseries","requests":[{"q":"sum:dd.terminator.request{status:error} by {datacenter}.as_count()/sum:dd.terminator.request{*} by {datacenter}.as_count()*100","display_type":"line"}],"title":"Error Rate % by Datacenter"},"graph_size":"l"}}'

# Merge and update
MERGED=$(echo "$EXISTING" | jq ". + [$NEW_CELL]")

curl -s -X PUT "https://api.datadoghq.com/api/v1/notebooks/${NOTEBOOK_ID}" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: $API_KEY" \
  -H "DD-APPLICATION-KEY: $APP_KEY" \
  -d "{\"data\":{\"type\":\"notebooks\",\"attributes\":{\"cells\":$MERGED}}}"
```
