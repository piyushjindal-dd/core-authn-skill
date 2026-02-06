# Feature Development Guide

## Development Workflow

### 1. Understand the Context

Before writing code:
- **Read existing code** in the affected area
- **Check Confluence** for design docs or RFCs
- **Review recent PRs** in the same area for patterns
- **Understand the blast radius** - these services handle ALL auth traffic

### 2. Design Considerations

#### Adding a New Authentication Method

1. Define the method in `routeconfig.AuthnMethod`
2. Add parsing logic in the request parser
3. Implement resolution in the resolver/handler
4. Add appropriate metrics and logging
5. Update route configurations

#### Adding a New Feature Flag

```go
// 1. Define the flag constant
const EnableNewFeatureFF = "authenticator_enable_new_feature"

// 2. Use with org context for gradual rollout
expCtx := experiments.WithExperimentContext(ctx, &experiments.ExperimentContext{
    OrgId: resolvedCreds.OrgID,
})
if experimentsClient.IsEnabled(expCtx, EnableNewFeatureFF) {
    // New feature logic
}

// 3. Add metric to track feature usage
statsd.Incr("dd.authenticator.new_feature", []string{
    fmt.Sprintf("enabled:%t", enabled),
}, 1)
```

#### Adding Caching

Follow this pattern from JWT caching:

```go
// 1. Define TTL constants with clear documentation
const (
    // Consumers expect JWT valid for at least 2 mins
    minimalJwtTTLContract = time.Minute * 2
    // Cache eviction after 30s
    cacheTTL = time.Second * 30
    // Actual TTL = contract + cache buffer
    jwtTTL = minimalJwtTTLContract + cacheTTL
)

// 2. Check feature flag before using cache
cacheEnabled := experimentsClient.IsEnabled(ctx, cacheEnabledFF)
if !cacheEnabled {
    statsd.Incr(cacheMetric, []string{"result:miss", "reason:disabled"}, 1)
    return generateFresh()
}

// 3. Try cache first
cached, hit, err := cache.Get(ctx, key)
if hit && err == nil {
    statsd.Incr(cacheMetric, []string{"result:hit"}, 1)
    return cached, nil
}

// 4. Generate fresh and cache (best effort)
fresh := generateFresh()
if cacheErr := cache.Add(ctx, key, fresh); cacheErr != nil {
    statsd.Incr(cacheWriteErrorMetric, nil, 1)
    // Don't fail the request on cache errors
}
return fresh, nil
```

### 3. Implementation Patterns

#### Response Generation (Authenticator/Terminator)

Use named generator functions for different response types:

```go
func generateSuccessResponse(ctx context.Context, ...) *auth.CheckResponse
func generateUnauthenticatedDeniedResponse(ctx context.Context, ...) *auth.CheckResponse
func generateInternalErrorResponse(ctx context.Context, reason string, err error) (*auth.CheckResponse, error)
```

#### gRPC Handler Pattern (OBO)

```go
func (s *server) MyMethod(ctx context.Context, req *pb.Request) (resp *pb.Response, err error) {
    methodName := "MyMethod"
    span, ctx := tracer.StartSpanFromContext(ctx, methodName)
    defer func() { span.Finish(tracer.WithError(err)) }()

    observability_accumulator.AddBothLogAndMetricField(ctx, "endpoint", methodName)

    // Validate input
    if err := validateRequest(req); err != nil {
        return nil, handleError(ctx, err, "invalid request")
    }

    // Business logic
    result, err := s.doWork(ctx, req)
    if err != nil {
        return nil, handleError(ctx, err, "operation failed")
    }

    // Success metrics
    IncrementMetricWithAccumulatedTags(ctx, serverMetric, []string{"status:success"})
    return result, nil
}
```

#### Interface-Based Design

Always use interfaces for testability:

```go
// Define interface
type CredentialResolver interface {
    Resolve(ctx context.Context, key string) (*Credentials, error)
}

// Compile-time assertion
var _ CredentialResolver = (*resolverImpl)(nil)

// Constructor
func NewCredentialResolver(db Database, cache Cache) CredentialResolver {
    return &resolverImpl{db: db, cache: cache}
}
```

### 4. Adding Metrics

**Always add metrics for new features:**

```go
// 1. Define metric constants
const (
    newFeatureMetric       = "dd.authenticator.new_feature"
    newFeatureLatencyDist  = "dd.authenticator.new_feature.latency"
)

// 2. Track success/failure
statsd.Incr(newFeatureMetric, []string{
    "status:success",
    fmt.Sprintf("method:%s", method),
}, 1)

// 3. Track latency
statsd.Distribution(newFeatureLatencyDist,
    float64(time.Since(start).Milliseconds()),
    tags, 1)
```

### 5. Testing Requirements

1. **Unit tests required** for all changes
2. **Table-driven tests** for multiple scenarios
3. **Mock external dependencies** using gomock
4. **Test error paths** - not just happy path

```bash
# Run tests before committing
rake authenticator:test          # or appropriate service
go test -race ./apps/authenticator/...

# For obo
bzl test //domains/aaa_authn/apps/apis/obo/...
```

### 6. Monitor Changes

If modifying monitors in terraform-config:

```bash
cd ~/dd/terraform-config

# Format check
terraform fmt -check -recursive aaa-authn/resources/authenticator

# Fix formatting
terraform fmt -recursive aaa-authn/resources/authenticator
```

### 7. Git Workflow

#### Branch Naming Convention

```
piyush.jindal/<JIRA_TICKET>-<description>
```

Examples:
- `piyush.jindal/AUTHN-5151-add-jwt-caching`
- `piyush.jindal/AUTHN-5152-fix-nil-orgdata`

#### Creating a Branch

```bash
# For dd-go
cd ~/dd/dd-go
git checkout prod
git pull origin prod
git checkout -b piyush.jindal/AUTHN-XXXX-description

# For dd-source
cd ~/dd/dd-source
git checkout main
git pull origin main
git checkout -b piyush.jindal/AUTHN-XXXX-description

# For terraform-config
cd ~/dd/terraform-config
git checkout master
git pull origin master
git checkout -b piyush.jindal/AUTHN-XXXX-description
```

#### Commit Messages

Follow conventional commit format with Jira ticket:

```bash
# Format
git commit -m "$(cat <<'EOF'
[AUTHN-XXXX] Brief description of change

- Detail 1
- Detail 2

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>
EOF
)"
```

**Commit message guidelines:**
- Start with Jira ticket in brackets: `[AUTHN-XXXX]`
- Use imperative mood: "Add feature" not "Added feature"
- Keep first line under 72 characters
- Explain *why* not just *what* in the body

**Examples:**
```
[AUTHN-5151] Add JWT caching to reduce latency

- Implement 30s TTL cache for generated JWTs
- Add feature flag for gradual rollout
- Include cache hit/miss metrics

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>
```

```
[AUTHN-5152] Fix nil pointer dereference on OrgData

- Add nil check before accessing OrgData.OrgID
- Add test case for nil OrgData scenario

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>
```

#### Creating a Pull Request

Use `gh` CLI to create PRs:

```bash
# Push branch first
git push -u origin HEAD

# Create PR with template
gh pr create --title "[AUTHN-XXXX] Brief description" --body "$(cat <<'EOF'
## Summary
- What this PR does
- Why it's needed

## Changes
- List of changes

## Testing
- [ ] Unit tests pass (`rake authenticator:test`)
- [ ] Manual testing done
- [ ] Metrics verified in staging

## Rollout Plan
- Feature flag: `authenticator_enable_feature`
- Gradual rollout by org

## Related
- Jira: https://datadoghq.atlassian.net/browse/AUTHN-XXXX
- Slack discussion: [link]

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

#### Useful Git Commands

```bash
# Check current status
git status

# See what will be committed
git diff --staged

# Amend last commit (before push)
git commit --amend

# Rebase on latest base branch
git fetch origin
git rebase origin/prod           # dd-go
git rebase origin/main           # dd-source
git rebase origin/master         # terraform-config

# Interactive rebase to squash commits (use appropriate base)
git rebase -i origin/prod        # dd-go
git rebase -i origin/main        # dd-source
git rebase -i origin/master      # terraform-config

# View PR in browser
gh pr view --web

# Check PR status
gh pr status

# Add reviewers
gh pr edit --add-reviewer hopen528,Alchemille
```

#### After PR is Merged

```bash
# Clean up local branch (use appropriate base branch)
# dd-go
git checkout prod && git pull origin prod

# dd-source
git checkout main && git pull origin main

# terraform-config
git checkout master && git pull origin master

# Delete merged branch
git branch -d piyush.jindal/AUTHN-XXXX-description
```

### 8. PR Checklist

Before submitting:
- [ ] Tests pass locally
- [ ] New metrics added for observability
- [ ] Error handling follows conventions
- [ ] Logging includes relevant context
- [ ] No nil pointer risks (especially `OrgData`)
- [ ] Memory impact estimated (if adding caching)
- [ ] Constants documented with comments
- [ ] Feature flag for gradual rollout (if risky)
- [ ] Commit message includes Jira ticket
- [ ] PR description explains *why* not just *what*
