# Common Pitfalls

These are mistakes frequently caught in PR reviews. Avoid them!

## 1. Nil Pointer Dereference on OrgData

**Problem:** `OrgData` can be nil in auth results, causing panics.

```go
// BAD - will panic if OrgData is nil
orgID := ar.OrgData.OrgID

// GOOD - check first
if ar.OrgData != nil {
    orgID := ar.OrgData.OrgID
}
```

This is the **most common issue** caught in reviews.

## 2. Missing Memory Impact Estimation

**Problem:** Adding caching without estimating memory impact.

**What reviewers ask:**
> "Do you have an estimation on the potential impact on memory increase?"

**Solution:** Calculate and document:
```go
// Example from JWT caching PR:
// - Average JWT size: ~2KB
// - Expected unique JWTs per pod: ~50K
// - Cache TTL: 30s
// - Estimated memory: ~100MiB per pod in US1
// See metrics: https://app.datadoghq.com/dashboard/...
```

## 3. Magic Numbers Without Comments

**Problem:** Constants without explanation.

```go
// BAD
jwtTTL := time.Minute*2 + time.Second*30

// GOOD
const (
    // JWT consumers expect tokens valid for at least 2 mins
    minimalJwtTTLContract = time.Minute * 2
    // Cache eviction after 30s to balance freshness and performance
    jwtCacheTTL = time.Second * 30
    // Total TTL ensures cached JWTs still meet contract
    jwtTTL = minimalJwtTTLContract + jwtCacheTTL
)
```

## 4. Missing Context in Log Statements

**Problem:** Log statements without enough context.

```go
// BAD
log.Error("job failed", log.ErrorField(err))

// GOOD
log.Error("job failed",
    log.String("job_name", jobName),
    log.String("job_id", jobID),
    log.ErrorField(err))
```

## 5. Inefficient Hashing

**Problem:** Using json.Marshal before hashing.

```go
// BAD - json.Marshal is 10x slower than necessary
jsonBytes, _ := json.Marshal(obj)
hash := sha256.Sum256(jsonBytes)

// GOOD - use hashstructure directly
hash, err := hashstructure.Hash(obj, hashstructure.FormatV2, nil)
```

## 6. Helm Hook Lifecycle Issues

**Problem:** Pre-install hooks referencing resources created by normal manifests.

**Common issues:**
- `pre-install` hooks creating ServiceAccounts that get deleted on upgrade
- Missing `helm.sh/hook-delete-policy: before-hook-creation`
- ConfigMaps not rendered when required by CronJobs

**Solution:** Understand hook ordering:
1. `pre-install` runs before ANY manifests
2. `pre-upgrade` runs before upgrade manifests
3. Hook resources are separate from release resources

## 7. Not Handling Edge Cases

**Problem:** Forgetting special cases like websocket JWTs with longer TTLs.

**What reviewers ask:**
> "How will this affect the 15-min TTL websocket JWT use case?"

**Solution:** Document and handle edge cases:
```go
// Special handling for websocket connections that need longer TTLs
if isWebsocketConnection(ctx) {
    return generateLongLivedJWT(...)
}
```

## 8. Empty Category Breaking Auth

**Problem:** App keys with empty categories causing authentication failures.

```go
// BAD - breaks if category is empty
if appKey.Category != expectedCategory {
    return ErrUnauthorized
}

// GOOD - handle empty category
if appKey.Category != "" && appKey.Category != expectedCategory {
    return ErrUnauthorized
}
```

## 9. Test Coverage Drops

**Problem:** PRs that reduce test coverage.

**What automation flags:**
> "Merging this branch will lead to reduced test coverage"

**Solution:**
- Add tests for new code paths
- Don't remove tests without replacement
- Maintain or improve coverage percentages

## 10. Mixing Unrelated Changes

**Problem:** Combining risky changes with safe ones in same PR.

**What reviewers suggest:**
> "Is it better if we make this change in a separate PR?"

**Solution:**
- Separate timeout changes from feature changes
- Keep infrastructure changes separate from logic changes
- Make PRs reviewable and revertable

## 11. Forgetting Metrics for New Features

**Problem:** Adding features without observability.

```go
// BAD - no visibility into feature usage
if featureEnabled {
    doNewThing()
}

// GOOD - track feature usage
if featureEnabled {
    statsd.Incr("dd.service.new_feature", []string{"enabled:true"}, 1)
    doNewThing()
}
```

## 12. Incorrect Error Wrapping

**Problem:** Losing error context or double-wrapping.

```go
// BAD - loses original error
if err != nil {
    return errors.New("operation failed")
}

// BAD - double wrapping
if err != nil {
    return fmt.Errorf("failed: %w", fmt.Errorf("operation: %w", err))
}

// GOOD - single wrap with context
if err != nil {
    return fmt.Errorf("operation failed: %w", err)
}
```

## 13. Not Using errors.Is() for Checks

**Problem:** Direct equality comparison for errors.

```go
// BAD - breaks with wrapped errors
if err == ErrNotFound {
    return nil
}

// GOOD - works with wrapped errors
if errors.Is(err, ErrNotFound) {
    return nil
}
```

## Quick Reference

| Pitfall | Fix |
|---------|-----|
| Nil OrgData | Always check `if ar.OrgData != nil` |
| No memory estimate | Calculate and document in PR |
| Magic numbers | Add const with comment explaining |
| Sparse logs | Include job_name, job_id, context |
| Slow hashing | Use hashstructure instead of json+sha |
| Helm hooks | Understand lifecycle, use delete-policy |
| Edge cases | Document and handle special scenarios |
| Coverage drop | Add tests for new code |
| Mixed PRs | Split by risk/topic |
| No metrics | Add counters for new features |
| Bad error wrap | Use `%w` once with context |
| Error equality | Use `errors.Is()` |
