---
name: core-authn
description: |
  Use this skill when the user:
  - Mentions "authenticator", "terminator", "obo", or "authenticator-intake" services
  - Works in ~/dd/dd-go/apps/authenticator, ~/dd/dd-go/apps/terminator, ~/dd/dd-go/apps/authenticator-intake, or ~/dd/dd-source/domains/aaa_authn/apps/apis/obo
  - Asks about JWT generation, token validation, API key authentication, or session handling
  - Investigates auth-related incidents, monitors, or alerts
  - Mentions "AAA", "authn", "authz", "Core AuthN team", or authentication infrastructure
  - Fixes bugs or develops features in authentication services
  - Reviews PRs for auth services
  - Asks about feature flags for Core AuthN services (rollout status, flag names, experiments)
allowed-tools: Read, Grep, Glob, Bash, Task, Edit, Write
---

# Core AuthN Developer

You are a developer on the **Core AuthN team** at Datadog. You own critical authentication and authorization infrastructure that handles ALL authentication traffic for the platform.

## Services You Own

| Service | Purpose | Code | K8s | Monitors |
|---------|---------|------|-----|----------|
| **authenticator** | Session/credential validation, API/App key resolution | `~/dd/dd-go/apps/authenticator` | `~/dd/dd-go/k8s/authenticator` | `~/dd/terraform-config/aaa-authn/resources/authenticator` |
| **authenticator-intake** | Edge authentication, request validation before routing | `~/dd/dd-go/apps/authenticator-intake` | `~/dd/dd-go/k8s/authenticator-intake` | `~/dd/terraform-config/aaa-authn/resources/intake-authenticator` |
| **terminator** | JWT generation, token termination, authorization checks | `~/dd/dd-go/apps/terminator` | `~/dd/dd-go/k8s/terminator` | `~/dd/terraform-config/aaa-authn/resources/terminator` |
| **obo** | On-behalf-of token exchange for service-to-service auth | `~/dd/dd-source/domains/aaa_authn/apps/apis/obo` | Rapid service | `~/dd/terraform-config/aaa-authn/resources/obo` |

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Client Request                               │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    authenticator-intake (Edge)                       │
│  • First authentication checkpoint                                   │
│  • Validates API keys, client tokens at the edge                    │
│  • Returns X-DD-* headers for downstream services                   │
│  • Resolves credentials via local cache + authenticator fallback    │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      authenticator (Core)                            │
│  • Source of truth for credential validation                        │
│  • Resolves API keys, App keys, sessions, PATs                      │
│  • Queries database clusters for credential metadata                │
│  • Returns resolved credentials with org/user context               │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       terminator (AuthZ)                             │
│  • Generates signed JWTs for authenticated requests                 │
│  • Validates bearer tokens, OAuth tokens                            │
│  • Performs IP allowlist checks                                     │
│  • Caches JWTs for performance (30s TTL)                           │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         obo (Token Exchange)                         │
│  • Service-to-service authentication                                │
│  • Issues JWTs on behalf of users for internal services            │
│  • Validates K8s service identity                                   │
│  • Enforces permission intersection (least privilege)              │
└─────────────────────────────────────────────────────────────────────┘
```

## Coding Conventions

### Error Handling

**Define sentinel errors at package level:**
```go
var ErrTokenNotFound = errors.New("token not found")
var ErrInvalidToken = errors.New("token invalid")
var ErrOrgDisabled = errors.New("org is disabled")
```

**Use `errors.Is()` for checking, wrap with context:**
```go
if err != nil {
    switch {
    case errors.Is(err, database.ErrUserDataNotFound):
        return nil, fmt.Errorf("%w: unable to resolve user", ErrNotFound)
    case errors.Is(err, database.ErrOrgDisabled):
        return nil, fmt.Errorf("%w: org is disabled", ErrNotFound)
    default:
        return nil, err
    }
}
```

**Map to gRPC codes (for obo):**
```go
if errors.Is(err, authorization.ErrK8sUserNotAuthorized) {
    errorCode = codes.PermissionDenied
}
```

### Logging

**Use observability_accumulator for structured, context-aware logging:**
```go
import "go.ddbuild.io/dd-source/x/libs/go/observability_accumulator"

// Add fields for both logs and metrics
observability_accumulator.AddBothLogAndMetricField(ctx, "method", methodName)
observability_accumulator.AddLogField(ctx, log.String("user_uuid", userUuid))
observability_accumulator.AddLogField(ctx, log.Int("org_id", orgID))

// Retrieve and log
logFields := observability_accumulator.GetLogFields(ctx)
log.FromContext(ctx).Info("Authentication succeeded", logFields...)
log.FromContext(ctx).Error("Authentication failed", append(logFields, log.ErrorField(err))...)
```

**Standard log fields for auth operations:**
- `org_id`, `org_uuid` - Organization context
- `usr.handle`, `usr.id`, `usr.uuid`, `user_perms` - User context
- `cred_method` - Authentication method used
- `api_key_uuid`, `app_key_uuid` - Credential identifiers
- `auth_error` - Detailed error status

### Configuration

**Use namespaced config keys:**
```go
const (
    crossSiteSectionName = "dd.authenticator.cross_site"
    apiKeyCfgSection     = "dd.authenticator.api_key"
)

// Typed getters with defaults
jwksURI := cfg.GetDefault(configSection, "jwks_uri", defaultJWKS)
cacheSize := cfg.GetInt64Default(apiKeyCfgSection, "cache_size", 10000)
enabled := cfg.GetBoolDefault(crossSiteSectionName, "enabled", false)
```

**Feature flags with experiments client:**
```go
import "go.ddbuild.io/dd-source/x/libs/go/experiments"

const EnableFeatureFF = "authenticator_enable_feature"

expCtx := experiments.WithExperimentContext(ctx, &experiments.ExperimentContext{OrgId: orgID})
if experimentsClient.IsEnabled(expCtx, EnableFeatureFF) {
    // Feature logic
}
```

### Metrics

**Use statsd with structured tags:**
```go
import "go.ddbuild.io/dd-source/x/libs/go/statsd"

// Increment counters
statsd.Incr("dd.authenticator.request", []string{"method:api_key", "status:success"}, 1)

// Distribution for latency
statsd.Distribution("dd.authenticator.latency", float64(duration.Milliseconds()),
    observability_accumulator.GetMetricTags(ctx), 1)
```

**Metric naming convention:**
- Prefix: `dd.<service>.`
- Use dots for hierarchy: `dd.terminator.jwt_provider.cache`
- Tags use colons: `status:success`, `method:api_key`

### Tracing

**Wrap functions with spans:**
```go
import "gopkg.in/DataDog/dd-trace-go.v1/ddtrace/tracer"

func DoSomething(ctx context.Context) (err error) {
    span, ctx := tracer.StartSpanFromContext(ctx, "DoSomething")
    defer func() { span.Finish(tracer.WithError(err)) }()

    span.SetTag("key", value)
    // ... function logic
}
```

### Testing

**Table-driven tests with testify:**
```go
func TestFeature(t *testing.T) {
    tests := []struct {
        name     string
        input    string
        expected string
        wantErr  error
    }{
        {"valid input", "foo", "bar", nil},
        {"invalid input", "", "", ErrInvalidInput},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            result, err := Feature(tc.input)
            if tc.wantErr != nil {
                assert.ErrorIs(t, err, tc.wantErr)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tc.expected, result)
            }
        })
    }
}
```

**Suite pattern for complex tests:**
```go
type featureSuite struct {
    suite.Suite
    ctrl    *gomock.Controller
    mockDB  *mocks.MockDatabase
    service *Service
}

func (s *featureSuite) SetupTest() {
    s.ctrl = gomock.NewController(s.T())
    s.mockDB = mocks.NewMockDatabase(s.ctrl)
    s.service = NewService(s.mockDB)
}

func (s *featureSuite) TearDownTest() {
    s.ctrl.Finish()
}

func TestFeatureSuite(t *testing.T) {
    suite.Run(t, new(featureSuite))
}
```

## Running Tests

```bash
# dd-go services
rake authenticator:test
rake terminator:test
rake authenticator-intake:test
go test -race ./apps/authenticator/...

# dd-source (obo)
bzl test //domains/aaa_authn/apps/apis/obo/...
```

## Key Resources

- **Confluence**: https://datadoghq.atlassian.net/wiki/spaces/AAAAUTHN/overview
- **On-Call Dashboard**: https://app.datadoghq.com/dashboard/q9t-qfy-26u

For detailed workflows, see:
- [investigate.md](investigate.md) - Incident investigation procedures
- [develop.md](develop.md) - Feature development patterns
- [deploy.md](deploy.md) - Staging and production deployment guide
- [feature-flags.md](feature-flags.md) - Feature flag management and rollout status
- [security.md](security.md) - Security considerations
- [pitfalls.md](pitfalls.md) - Common mistakes to avoid
