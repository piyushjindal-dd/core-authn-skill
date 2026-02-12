# Guide: Adding New Authentication Methods to Core AuthN

**Purpose**: Step-by-step guide for implementing new authentication methods
**Audience**: Core AuthN developers
**Last Updated**: February 11, 2026

---

## Overview

This guide walks through the complete process of adding a new authentication method to Datadog's Core AuthN infrastructure. Follow these steps to ensure consistency with existing patterns and best practices.

---

## Prerequisites

Before adding a new auth method, determine:

1. **Credential Type**: What credential will be validated? (cookie, header, token, etc.)
2. **Validation Source**: Where to validate? (database, external service, cache, etc.)
3. **Passthrough Behavior**: Should it be validated by Authenticator/Terminator or passed through?
4. **Session Management**: Does it require session creation/management?
5. **Scoping Requirements**: Does the JWT need special scoping (route, resource, permissions)?

---

## Step-by-Step Implementation

### Step 1: Define AuthN Method

**File**: `pkg/routeconfig/authn_methods.go`

**Actions**:

1. Add constant for your auth method:
```go
const (
    ValidYourNewMethod AuthnMethod = "ValidYourNewMethod"
)
```

2. Define properties:
```go
yourNewMethodAuthentication = AuthnMethodProperties{
    authenticatorPassthrough: false,  // Set to true only if skipping authenticator
    terminatorPassthrough:    false,  // Set to true only if skipping terminator
    usesSession:              true,   // Or false, depending on credential type
    usesAPIKey:               false,
    usesAPIAppKey:            false,
    usesBearerToken:          false,
    usesPAT:                  false,
}
```

3. Add to mapping:
```go
authnMethodMapping = map[AuthnMethod]AuthnMethodProperties{
    ValidYourNewMethod: yourNewMethodAuthentication,
    // ... existing methods
}
```

4. Add helper function:
```go
func MethodUsesYourNewMethod(method AuthnMethod) bool {
    return method == ValidYourNewMethod
}
```

**Commit**: Create PR with just this change first (small, easy to review)

---

### Step 2: Add Credential Extraction

**File**: `pkg/ddextauthz/utils.go`

**Actions**:

1. Add fields to `DDAuthenticationCredentials`:
```go
type DDAuthenticationCredentials struct {
    // ... existing fields

    // Your new method credentials
    YourNewMethodToken    string
    YourNewMethodMetadata map[string]string  // Optional: additional data
}
```

2. Add extraction logic to `findCredentials()`:
```go
func findCredentials(ctx context.Context, req *auth.CheckRequest, ...) (*DDAuthenticationCredentials, error) {
    // ... existing extraction

    // Extract your credential
    yourToken := readYourTokenFromRequest(req)

    creds := &DDAuthenticationCredentials{
        // ... existing fields
        YourNewMethodToken: yourToken,
    }

    return creds, nil
}
```

3. Implement extraction helper:
```go
func readYourTokenFromRequest(req *auth.CheckRequest) string {
    // Extract from header
    headers := req.GetAttributes().GetRequest().GetHttp().GetHeaders()
    if token, ok := headers["x-your-token"]; ok {
        return token
    }

    // Or extract from cookie
    cookie := readCookieFromCheckRequest(req)
    // ...

    // Or extract from query param
    queryParams := parseQueryParams(req.GetAttributes().GetRequest().GetHttp().GetPath())
    return queryParams.Get("your_token")
}
```

**Commit**: PR with credential extraction (can be reviewed independently)

---

### Step 3: Implement Authenticator Resolver

**File**: `apps/authenticator/datastore/repository/credential_readers.go`

**Actions**:

1. Add to dispatcher in `tryAuthNMethod()`:
```go
func (cr *CredentialReaders) tryAuthNMethod(...) (*model.ResolvedCredentials, error) {
    // ... existing dispatching

    if routeconfig.MethodUsesYourNewMethod(method) {
        resolvedCreds, err = cr.resolveYourNewMethod(ctx, credentials, requestContext)
        if resolvedCreds != nil || err != nil {
            return resolvedCreds, err
        }
    }

    return resolvedCreds, err
}
```

2. Implement resolver function:
```go
func (cr *CredentialReaders) resolveYourNewMethod(
    ctx context.Context,
    credentials *ddextauthz.DDAuthenticationCredentials,
    requestContext ddextauthz.RequestContext,
) (*model.ResolvedCredentials, error) {
    var err error
    span, ctx := tracer.StartSpanFromContext(ctx, "resolve_your_new_method")
    defer func() { span.Finish(tracer.WithError(err)) }()

    // Check if credential present
    if credentials.YourNewMethodToken == "" {
        return nil, nil  // Method not attempted (IMPORTANT!)
    }

    // Add observability
    observability_accumulator.AddBothLogAndMetricField(ctx, "auth_method", "your_new_method")

    // Validate credential
    userData, err := cr.validateYourToken(ctx, credentials.YourNewMethodToken)
    if err != nil {
        statsd.Incr("dd.authenticator.your_method.validation.failed", nil, 1)
        return nil, fmt.Errorf("token validation failed: %w", err)
    }

    statsd.Incr("dd.authenticator.your_method.validation.success", nil, 1)

    // Return resolved credentials
    return &model.ResolvedCredentials{
        UserID:   userData.UserID,
        OrgID:    userData.OrgID,
        UserUuid: userData.UserUUID,
        OrgUuid:  userData.OrgUUID,
        Method:   auth_methods.YourNewMethod,
        // Add method-specific data if needed
    }, nil
}
```

3. Implement validation logic:
```go
func (cr *CredentialReaders) validateYourToken(ctx context.Context, token string) (*UserData, error) {
    span, ctx := tracer.StartSpanFromContext(ctx, "validate_your_token")
    defer span.Finish(tracer.WithError(err))

    // Option A: Database query
    userData, err := cr.yourTokenDB.Query(ctx, token)

    // Option B: External service call
    userData, err := cr.yourTokenValidator.Validate(ctx, token)

    // Option C: Cryptographic validation (e.g., JWT)
    claims, err := validateJWT(token)
    userData = extractUserData(claims)

    return userData, err
}
```

**Testing**:
```go
func TestResolveYourNewMethod(t *testing.T) {
    tests := []struct {
        name        string
        token       string
        mockSetup   func(*mocks.MockYourTokenDB)
        expected    *model.ResolvedCredentials
        expectedErr error
    }{
        {
            name:  "valid token",
            token: "valid-token-123",
            mockSetup: func(m *mocks.MockYourTokenDB) {
                m.EXPECT().Query(gomock.Any(), "valid-token-123").
                    Return(&UserData{UserID: 123, OrgID: 456}, nil)
            },
            expected: &model.ResolvedCredentials{
                UserID: 123,
                OrgID:  456,
            },
            expectedErr: nil,
        },
        {
            name:  "invalid token",
            token: "invalid-token",
            mockSetup: func(m *mocks.MockYourTokenDB) {
                m.EXPECT().Query(gomock.Any(), "invalid-token").
                    Return(nil, ErrTokenNotFound)
            },
            expected:    nil,
            expectedErr: ErrTokenNotFound,
        },
        {
            name:        "empty token",
            token:       "",
            mockSetup:   func(m *mocks.MockYourTokenDB) {},
            expected:    nil,
            expectedErr: nil,  // Not attempted
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            // Test implementation
        })
    }
}
```

---

### Step 4: Implement Terminator Handler

**File**: `apps/terminator/server/resolver.go`

**Actions**:

1. Add to dispatcher in `handleCredsByAuthnMethod()`:
```go
func (dataStores *dataStores) handleCredsByAuthnMethod(...) (*models.AuthResult, error) {
    // ... existing handlers

    if routeconfig.MethodUsesYourNewMethod(method) {
        ar, err = dataStores.handleYourNewMethodRequest(ctx, creds, requestContext)
        if ar != nil || err != nil {
            return ar, err
        }
    }

    return ar, err
}
```

2. Implement handler:
```go
func (dataStores *dataStores) handleYourNewMethodRequest(
    ctx context.Context,
    creds util.AuthenticationCredentialsWithMetadata,
    requestContext ddextauthz.RequestContext,
) (*models.AuthResult, error) {
    var err error
    span, ctx := tracer.StartSpanFromContext(ctx, "handle_your_new_method")
    defer func() { span.Finish(tracer.WithError(err)) }()

    // Extract data from Authenticator's dynamic metadata
    userData, err := extractUserDataFromMetadata(ctx, creds.CommonDynamicMetadata)
    if err != nil {
        return nil, fmt.Errorf("failed to extract user data: %w", err)
    }

    // Build AuthResult
    authResult := &models.AuthResult{
        UserData: &authctxmodels.UserData{
            UserID:      userData.UserID,
            OrgID:       userData.OrgID,
            UserUUID:    userData.UserUUID,
            OrgUUID:     userData.OrgUUID,
            Permissions: userData.Permissions,
        },
        Method: auth_methods.YourNewMethod,
        MethodMetadata: map[string]string{
            "source": "your_new_method",
            // Add method-specific metadata
        },
    }

    return authResult, nil
}
```

**Note**: JWT generation happens automatically after this (via `jwtSvc.GenerateJWT`)

---

### Step 5: Update Route Configuration

**File**: `dd-go/k8s/authenticator/templates/route_data.yaml`

**Actions**:

Add your auth method to appropriate routes:
```yaml
- path: "/your/api/endpoint"
  methods: ["GET", "POST"]
  authn_methods: ["ValidYourNewMethod", "ValidUser"]  # Can have multiple
  service: "your-service"
  permissions: ["your_permission"]
```

**Route Config Tips**:
- Multiple auth methods tried in order
- First successful method wins
- Fallback to next method if credentials not present

---

### Step 6: Add Database/Client Dependencies

**File**: `apps/authenticator/server/server_bootstrap.go`

**Actions**:

1. Initialize your database/client:
```go
func mustCredentialReaders(...) *repository.CredentialReaders {
    // ... existing initialization

    // Your database/client
    yourTokenDB := db.MustYourTokenDB(cfg)
    // OR
    yourValidator := initYourValidator(cfg)

    return repository.NewCredentialReaders(
        sessionDB,
        accessTokenReader,
        // ... existing deps
        yourTokenDB,      // NEW
        yourValidator,    // NEW
    )
}
```

2. Update CredentialReaders struct:
```go
type CredentialReaders struct {
    // ... existing fields
    yourTokenDB       db.YourTokenDBInterface
    yourTokenValidator *YourValidator
}
```

3. Update constructor:
```go
func NewCredentialReaders(
    // ... existing params
    yourTokenDB db.YourTokenDBInterface,
) *CredentialReaders {
    return &CredentialReaders{
        // ... existing fields
        yourTokenDB: yourTokenDB,
    }
}
```

---

## Common Patterns

### Pattern 1: Database-Backed Credentials

**Example**: API keys, App keys, PATs

**Steps**:
1. Extract credential from request
2. Hash credential (security - don't store plaintext)
3. Query database by fingerprint
4. Return user/org context

**Code**:
```go
func (cr *CredentialReaders) resolveYourMethod(...) (*model.ResolvedCredentials, error) {
    // Hash token
    fingerprints, err := cr.tokenHasher.TokenHashes(token)
    if err != nil {
        return nil, err
    }

    // Query database
    result, err := cr.yourDB.QueryByFingerprint(ctx, fingerprints)
    if err != nil {
        return nil, err
    }

    return &model.ResolvedCredentials{
        UserID:   result.UserID,
        OrgID:    result.OrgID,
        // ...
    }, nil
}
```

---

### Pattern 2: Session-Based Credentials

**Example**: User sessions, secure embed sessions

**Steps**:
1. Extract session cookie
2. Read session from Cassandra (via Phoenix)
3. Decrypt session data
4. Return user/org context

**Code**:
```go
func (cr *CredentialReaders) resolveYourSession(...) (*model.ResolvedCredentials, error) {
    // Read from Cassandra
    sessionData, encryptedData, err := cr.sessionsReader.ReadSessionWithMetadata(ctx, cookie)
    if err != nil {
        return nil, err
    }

    return &model.ResolvedCredentials{
        UserID:               sessionData.UserID,
        OrgID:                sessionData.OrgID,
        SessionData:          sessionData,
        EncryptedSessionData: encryptedData,
        Method:               auth_methods.YourSessionMethod,
    }, nil
}
```

---

### Pattern 3: External Token Validation (JWT)

**Example**: OAuth tokens, external auth tokens, Ticino tokens

**Steps**:
1. Extract JWT token
2. Validate using JWKS (public keys)
3. Extract user context from claims
4. Return user/org context

**Code**:
```go
func (cr *CredentialReaders) resolveExternalToken(...) (*model.ResolvedCredentials, error) {
    // Validate JWT
    claims, err := cr.jwtValidator.Validate(ctx, token)
    if err != nil {
        return nil, err
    }

    // Extract user context
    return &model.ResolvedCredentials{
        UserID:   claims.UserID,
        OrgID:    claims.OrgID,
        UserUuid: claims.Sub,
        Method:   auth_methods.ExternalTokenMethod,
    }, nil
}
```

---

### Pattern 4: Composite Credentials

**Example**: API + App key pair

**Steps**:
1. Resolve both credentials independently
2. Verify both belong to same org
3. Return combined context

**Code**:
```go
func (cr *CredentialReaders) resolveComposite(...) (*model.ResolvedCredentials, error) {
    // Resolve first credential
    firstResult, err := cr.resolveFirst(ctx, credentials)
    if err != nil {
        return nil, err
    }

    // Resolve second credential
    secondResult, err := cr.resolveSecond(ctx, credentials)
    if err != nil {
        return nil, err
    }

    // Verify match
    if firstResult.OrgID != secondResult.OrgID {
        return nil, ErrMismatchedOrg
    }

    // Combine
    return &model.ResolvedCredentials{
        UserID:     secondResult.UserID,  // Typically from second
        OrgID:      firstResult.OrgID,
        FirstUuid:  firstResult.Uuid,
        SecondUuid: secondResult.Uuid,
    }, nil
}
```

---

## Phoenix Integration (Session Management)

### When to Use Phoenix

Use Phoenix when your auth method involves **session management**:
- Creating sessions after authentication
- Reading sessions for validation
- Deleting sessions on logout

### Creating Sessions via Phoenix

**From Terminator**:

1. Add Phoenix gRPC client:
```go
type dataStores struct {
    // ... existing
    phoenixClient phoenix.PhoenixDataPlaneClient
}
```

2. Call CreateSession:
```go
resp, err := phoenixClient.CreateSession(ctx, &phoenix_pb.CreateSessionRequest{
    SessionKeys: &phoenix_pb.SessionKeys{
        UserUuid:          userUUID,
        UserId:            uint64(userID),
        UserHandle:        userHandle,
        UserHandleOrgId:   uint64(orgID),
        UserHandleOrgUuid: orgUUID,
        LoginMethod:       "your_method_name",
        Ip:                clientIP,
        UserAgent:         userAgent,
        LastLoginTime:     float64(time.Now().Unix()),
        SessionTtl:        2592000,  // 30 days
    },
})

signedSessionId := resp.SignedSessionId
```

3. Set session cookie in response:
```go
cookie := &httpv3.HeaderValueOption{
    Header: &corev3.HeaderValue{
        Key:   "set-cookie",
        Value: fmt.Sprintf("cookie_name=%s; Max-Age=%d; Secure; HttpOnly", signedSessionId, ttl),
    },
}
```

### Reading Sessions (Authenticator)

**Already handled by `sessionsReader`**:
```go
// Existing code works for all session types
sessionData, _, err := cr.sessionsReader.ReadSessionWithMetadata(ctx, cookie)
```

**Differentiation**: Use `login_method` field
```go
if sessionData.LoginMethod == "your_method_name" {
    // Handle your method specifically
}
```

---

## OBO Integration (User Impersonation)

### When to Use OBO

Use OBO when you need to **generate a User JWT** but don't have the user's credentials:
- Service-to-service calls requiring user context
- Acting on behalf of users
- Dashboard viewing as owner
- Delegated access scenarios

### Calling OBO from Terminator

**Pattern**:

1. Receive user context from Authenticator
2. Instead of generating JWT directly, call OBO
3. Return OBO's JWT

**Code**:
```go
func (dataStores *dataStores) handleYourMethod(...) (*models.AuthResult, error) {
    // Get user context from Authenticator
    userUUID := extractUserUUID(ctx, creds)

    // Call OBO for JWT
    jwt, err := dataStores.oboClient.GetOboUserJwt(ctx, &obo_pb.GetOboUserJwtRequest{
        UserUuid: userUUID,
        Scope:    "your_scope",  // Optional: scope the JWT
    })

    // OBO handles JWT generation, caching, signing
    return jwt, nil
}
```

**When to Use OBO vs Terminator JWT**:
- Use OBO: When you need User JWT specifically, service-to-user delegation
- Use Terminator: When standard route-based JWT is sufficient

---

## Testing Guidelines

### Authenticator Tests

**Location**: `apps/authenticator/datastore/repository/credential_readers_test.go`

**Test Structure**:
```go
func TestResolveYourNewMethod(t *testing.T) {
    tests := []struct {
        name          string
        credentials   *ddextauthz.DDAuthenticationCredentials
        mockSetup     func(*mocks.MockYourDB)
        expectedCreds *model.ResolvedCredentials
        expectedErr   error
    }{
        {
            name: "valid token",
            credentials: &ddextauthz.DDAuthenticationCredentials{
                YourNewMethodToken: "valid-token",
            },
            mockSetup: func(m *mocks.MockYourDB) {
                m.EXPECT().Query(gomock.Any(), "valid-token").
                    Return(&UserData{UserID: 123, OrgID: 456}, nil)
            },
            expectedCreds: &model.ResolvedCredentials{
                UserID: 123,
                OrgID:  456,
                Method: auth_methods.YourNewMethod,
            },
            expectedErr: nil,
        },
        {
            name: "empty token - method not attempted",
            credentials: &ddextauthz.DDAuthenticationCredentials{
                YourNewMethodToken: "",
            },
            mockSetup:     func(m *mocks.MockYourDB) {},
            expectedCreds: nil,
            expectedErr:   nil,  // IMPORTANT: nil for not attempted
        },
        {
            name: "invalid token",
            credentials: &ddextauthz.DDAuthenticationCredentials{
                YourNewMethodToken: "invalid",
            },
            mockSetup: func(m *mocks.MockYourDB) {
                m.EXPECT().Query(gomock.Any(), "invalid").
                    Return(nil, ErrInvalidToken)
            },
            expectedCreds: nil,
            expectedErr:   ErrInvalidToken,
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            ctrl := gomock.NewController(t)
            defer ctrl.Finish()

            mockDB := mocks.NewMockYourDB(ctrl)
            tc.mockSetup(mockDB)

            cr := &CredentialReaders{yourDB: mockDB}

            result, err := cr.resolveYourNewMethod(context.Background(), tc.credentials, ddextauthz.RequestContext{})

            if tc.expectedErr != nil {
                assert.ErrorIs(t, err, tc.expectedErr)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tc.expectedCreds.UserID, result.UserID)
                assert.Equal(t, tc.expectedCreds.OrgID, result.OrgID)
            }
        })
    }
}
```

---

### Terminator Tests

**Location**: `apps/terminator/server/resolver_test.go`

**Test Structure**:
```go
func TestHandleYourNewMethodRequest(t *testing.T) {
    tests := []struct {
        name         string
        creds        util.AuthenticationCredentialsWithMetadata
        expectedUser *authctxmodels.UserData
        expectedErr  error
    }{
        {
            name: "valid credentials",
            creds: util.AuthenticationCredentialsWithMetadata{
                CommonDynamicMetadata: buildMetadata(userID, orgID),
            },
            expectedUser: &authctxmodels.UserData{
                UserID: 123,
                OrgID:  456,
            },
            expectedErr: nil,
        },
    }

    // Test implementation
}
```

---

## Deployment Checklist

### Before Deploying

- [ ] All unit tests pass (`rake authenticator:test`, `rake terminator:test`)
- [ ] Integration tests added
- [ ] Feature flag added for rollout control
- [ ] Metrics implemented (success/failure counters)
- [ ] Logging added (structured with observability_accumulator)
- [ ] Tracing added (spans for all operations)
- [ ] Documentation updated
- [ ] Security review completed

### Deployment Strategy

**1. Deploy to Staging**:
```bash
# Authenticator
ddr conductor run --target staging authenticator

# Terminator
ddr conductor run --target staging terminator
```

**2. Test in Staging**:
- Verify auth method works
- Check metrics/logging
- Validate performance

**3. Feature Flag Rollout** (Production):
- Start with 0% (feature flag off)
- Increase to 1% (canary)
- Monitor error rates, latency
- Gradually increase to 100%

**4. Monitor**:
- AAA On Call dashboard
- Service-specific dashboards
- Alert on elevated error rates

---

## Troubleshooting

### Common Issues

**1. Method Not Attempted**:
- **Symptom**: Credentials present but method returns `nil, nil`
- **Cause**: Credential extraction not working
- **Fix**: Check `ParseCheckRequest` logic

**2. Wrong Method Attempted**:
- **Symptom**: Logs show unexpected auth method
- **Cause**: Multiple methods configured, wrong one succeeds first
- **Fix**: Reorder methods or fix credential detection

**3. Cache Issues**:
- **Symptom**: Stale data returned
- **Cause**: Cache TTL too long or invalidation missing
- **Fix**: Adjust TTL or add cache invalidation

**4. Performance Degradation**:
- **Symptom**: Increased latency
- **Cause**: Database queries on critical path, cache misses
- **Fix**: Add caching, optimize queries, async operations

---

## Monitoring Your Auth Method

### Key Metrics to Add

**Authenticator**:
```go
// Success/failure
statsd.Incr("dd.authenticator.your_method.validation.success", nil, 1)
statsd.Incr("dd.authenticator.your_method.validation.failed", nil, 1)

// Latency
statsd.Distribution("dd.authenticator.your_method.latency", durationMs, nil, 1)

// Cache
statsd.Incr("dd.authenticator.your_method.cache.hit", nil, 1)
statsd.Incr("dd.authenticator.your_method.cache.miss", nil, 1)
```

**Terminator**:
```go
statsd.Incr("dd.terminator.your_method.jwt_generated", nil, 1)
statsd.Incr("dd.terminator.your_method.error", nil, 1)
```

### Logging

**Use observability_accumulator**:
```go
observability_accumulator.AddBothLogAndMetricField(ctx, "auth_method", "your_method")
observability_accumulator.AddLogField(ctx, log.String("token_type", tokenType))

logFields := observability_accumulator.GetLogFields(ctx)
log.FromContext(ctx).Info("Authentication succeeded", logFields...)
```

---

## Security Considerations

### 1. Credential Storage

**Never store plaintext credentials**:
- Hash before database lookup (use Clifford)
- Use fingerprints for queries
- Constant-time comparison for validation

### 2. Error Messages

**Don't leak information**:
```go
// ❌ Bad
return nil, fmt.Errorf("user john@example.com not found")

// ✅ Good
return nil, ErrInvalidCredentials  // Generic error
// Log detailed error server-side
log.Error("User not found", log.String("user", user))
```

### 3. Timing Attacks

**Use constant-time comparison**:
```go
import "crypto/subtle"

func compare(a, b string) bool {
    return subtle.ConstantTimeCompare([]byte(a), []byte(b)) == 1
}
```

### 4. Rate Limiting

**Consider adding rate limits**:
- Per user/org
- Per IP address
- Per auth method

---

## Performance Optimization

### 1. Cache Aggressively

**What to cache**:
- Database query results
- JWT validation results
- User/org lookups

**How long**:
- Sessions: In-memory + Redis (30s-5min)
- JWTs: 30 seconds (short for security)
- Credentials: Varies by type

### 2. Optimize Database Queries

**Best practices**:
- Use indexes on query fields
- Query replicas when possible
- Batch queries if multiple lookups
- Use connection pooling

### 3. Async Operations

**Non-critical operations**:
- Usage tracking (API key usage)
- Analytics events
- Non-essential logging

**Pattern**:
```go
go func() {
    cr.aggregator.Aggregate(apiKeyUUID, "web-api", orgID)
}()
```

---

## Checklist

### Implementation

- [ ] AuthN method defined in routeconfig
- [ ] Credential extraction in ddextauthz
- [ ] Resolver implemented in Authenticator
- [ ] Handler implemented in Terminator
- [ ] Database/client dependencies added
- [ ] Routes configured
- [ ] Unit tests written (Authenticator)
- [ ] Unit tests written (Terminator)
- [ ] Integration tests added

### Observability

- [ ] Success/failure metrics
- [ ] Latency metrics
- [ ] Cache metrics
- [ ] Structured logging
- [ ] Distributed tracing
- [ ] Dashboard updated

### Security

- [ ] Security review completed
- [ ] No plaintext credential storage
- [ ] Constant-time comparisons
- [ ] Error messages don't leak info
- [ ] Rate limiting considered
- [ ] IP allowlist support (if needed)

### Documentation

- [ ] Code comments added
- [ ] README updated
- [ ] Runbook created
- [ ] Customer docs (if customer-facing)

### Deployment

- [ ] Feature flag added
- [ ] Staged rollout plan
- [ ] Monitoring alerts configured
- [ ] Rollback plan documented

---

## Examples

### Example 1: Session-Based Method

See: `ValidUser` (session cookie authentication)
- Uses `resolveCookie()`
- Reads from Cassandra via sessionsReader
- Returns SessionData

### Example 2: Token-Based Method

See: `ValidUserPAT` (Personal Access Token)
- Uses `resolvePAT()`
- Hashes token, queries database
- Returns PAT metadata

### Example 3: Composite Method

See: `ValidDatadogAPIUser` (API + App key pair)
- Uses `resolveAPIAppKey()`
- Validates both keys
- Returns combined context

### Example 4: External Validation

See: `ValidUserDelegatedToken` (external JWT)
- Uses `resolveDelegatedToken()`
- Validates via JWKS
- Returns user context from claims

---

## Summary

Adding a new auth method involves:

1. **Define** (routeconfig) - ~50 lines
2. **Extract** (ddextauthz) - ~100 lines
3. **Resolve** (Authenticator) - ~200-300 lines
4. **Handle** (Terminator) - ~100-200 lines
5. **Test** - ~300-500 lines

**Total**: ~750-1150 lines for complete implementation

**Timeline**: 1-2 sprints depending on complexity

**Key Success Factors**:
- Follow existing patterns
- Comprehensive testing
- Good observability
- Security review
- Staged rollout
