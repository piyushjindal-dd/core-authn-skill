# Security Considerations

These services handle **ALL authentication traffic** for Datadog. Security is paramount.

## Critical Security Areas

### 1. Credential Handling

**Never log credentials:**
```go
// BAD - logs the actual key
log.Info("Processing request", log.String("api_key", apiKey))

// GOOD - log fingerprint or UUID only
log.Info("Processing request", log.String("api_key_fingerprint", fingerprint))
log.Info("Processing request", log.String("api_key_uuid", uuid))
```

**Use fingerprints for lookups:**
```go
// Convert keys to fingerprints immediately
fingerprint := ddutil.APIKeyToBlake2sFingerprint(apiKey)
```

**Constant-time comparison for secrets:**
```go
import "crypto/subtle"

// Use constant-time comparison to prevent timing attacks
if subtle.ConstantTimeCompare([]byte(provided), []byte(expected)) != 1 {
    return ErrInvalidCredential
}
```

### 2. Token Validation

**Always validate JWT signatures:**
```go
// The terminator validates JWTs using JWKS
jwtResult, err := jwksValidator.Validate(token)
if err != nil {
    return nil, fmt.Errorf("%w: %w", ErrInvalidToken, err)
}
```

**Check token expiration:**
```go
if time.Now().After(claims.ExpiresAt) {
    return nil, ErrAccessTokenExpired
}
```

**Validate issuer and audience:**
```go
if claims.Issuer != expectedIssuer {
    return nil, ErrInvalidIssuer
}
```

### 3. Permission Enforcement

**OBO uses permission intersection (least privilege):**
```go
// The JWT issued to a service can only have permissions
// that are BOTH in the user's permissions AND the service's allowlist
if authorizationConfig.ComputePermissionsIntersection {
    permissions = userctx.FilterPermissions(
        authorizationConfig.PermissionIds,  // Service's allowed perms
        userPermissions,                     // User's actual perms
    )
}
```

**IP Allowlist checks:**
```go
// Terminator enforces IP allowlist at the org level
if ipAllowlistEnabled {
    allowed, err := authorization.CheckIPAllowList(ctx, ar.OrgData, clientIP)
    if !allowed {
        return generateDeniedResponse(ctx, "ip_not_allowed")
    }
}
```

### 4. Session Security

**Session data validation:**
- Sessions include org_id, user_id, and permissions
- Cross-site requests require additional validation
- CSRF tokens validated for cookie-based auth

**Session metadata is base64 encoded but NOT encrypted:**
```go
// Session data passed in metadata - treat as sensitive
sessionMetadata := req.GetMetadataContext().GetFilterMetadata()
// Validate before use
```

### 5. Cross-Site Request Handling

**Secondary datacenter validation:**
```go
// When handling cross-site requests, validate the datacenter
if requestContext.IsRequestToCrossSiteFailoverApi {
    dc = secondaryOfDatacenter  // Use configured secondary
} else {
    dc = localDatacenter
}
```

### 6. Key Rotation

**Terminator handles key rotation:**
- Keys are rotated via PKI service
- Old keys remain valid during rotation window
- Monitor `dd.terminator.well_known_keys` for rotation issues

```go
// Key refresh runs in background
func (ts *terminatorServer) start(pkiConfig *pki.Config) {
    ts.ks.StartRefreshKeys(pkiConfig)
}
```

### 7. Rate Limiting Awareness

Authentication services sit behind rate limiters:
- Be aware of retry behavior
- Don't create infinite retry loops
- Log rate limit events for debugging

### 8. Error Messages

**Don't leak information in error messages:**
```go
// BAD - reveals internal state
return fmt.Errorf("user %s not found in org %d", userUUID, orgID)

// GOOD - generic message, details in logs
log.Error("user not found", log.String("user_uuid", userUUID), log.Int("org_id", orgID))
return ErrNotFound
```

### 9. Input Validation

**Validate all inputs at boundaries:**
```go
// Validate UUIDs
func isUUID(s string) bool {
    _, err := uuid.Parse(s)
    return err == nil
}

// Validate before processing
if !isUUID(request.GetUserUuid()) {
    return nil, status.Error(codes.InvalidArgument, "invalid user uuid")
}
```

**Sanitize paths for logging:**
```go
// Don't log full paths with potential PII
sanitizedPath := sanitizePath(request.GetPath())
observability_accumulator.AddMetricField(ctx, "path", sanitizedPath)
```

### 10. Secure Defaults

**Config should be secure by default:**
```go
// If config is missing, use empty (deny) not permissive
func getConfigWithDefault[T any](watcher, key string, defaultValue T) T {
    config, exists := content[key]
    if !exists {
        log.Warn("config not found, using secure default")
        return defaultValue  // Empty/restrictive default
    }
    return config
}
```

## Security Review Checklist

Before merging changes to auth services:

- [ ] No credentials logged (only fingerprints/UUIDs)
- [ ] Token validation includes signature, expiry, issuer checks
- [ ] Permission checks use intersection (least privilege)
- [ ] Error messages don't leak internal details
- [ ] Input validation at all boundaries
- [ ] Nil checks for OrgData and UserData
- [ ] No timing attacks possible (constant-time comparison)
- [ ] Secure defaults when config is missing
- [ ] Rate limiting behavior considered
- [ ] Cross-site handling validated
