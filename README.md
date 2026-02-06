# Core AuthN Skill

Claude Code skill for investigating, developing, and deploying Datadog's Core Authentication services.

## Services Covered

| Service | Description | Code Location |
|---------|-------------|---------------|
| **authenticator** | JWT generation and validation | `dd-go/apps/authenticator` |
| **authenticator-intake** | Request authentication at edge | `dd-go/apps/authenticator-intake` |
| **terminator** | API key and app key validation | `dd-go/apps/terminator` |
| **obo** | On-Behalf-Of token service | `dd-source/domains/aaa_authn/apps/apis/obo` |

## Skill Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill definition and capabilities |
| `investigate.md` | Incident investigation workflows using Bits and Datadog MCP |
| `develop.md` | Development guidelines and testing procedures |
| `deploy.md` | Deployment procedures and rollout strategies |
| `feature-flags.md` | Feature flag management for auth services |
| `pitfalls.md` | Common issues and solutions |
| `security.md` | Security considerations and best practices |

## Usage

This skill is designed to be used with Claude Code. Place the skill files in your `~/.claude/skills/core-authn/` directory.

### Example Invocations

```
"Investigate the authenticator-intake alert"
"Help me add a new endpoint to terminator"
"What's the rollout status of feature flag X?"
"Review the security implications of this change"
```

## Key Features

- **Bits-first investigation** - Uses Bits agent for Datadog telemetry access
- **MCP fallback** - Falls back to Datadog MCP tools when Bits unavailable
- **Service-specific context** - Deep knowledge of auth service architecture
- **Dashboard links** - Quick access to relevant dashboards
- **Runbook integration** - Links to Confluence runbooks

## Related Resources

- [AAA-AUTHN Confluence Space](https://datadoghq.atlassian.net/wiki/spaces/AAAAUTHN/overview)
- [Authenticator Dashboard](https://app.datadoghq.com/dashboard/3gw-hz2-up5)
- [Intake AuthN Dashboard](https://app.datadoghq.com/dashboard/xqk-uux-yq6)
- [Terminator Dashboard](https://app.datadoghq.com/dashboard/kbv-5mz-8e6)
