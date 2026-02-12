# Core AuthN Architecture Guide

**Purpose**: Understanding Datadog's Core Authentication infrastructure
**Audience**: Developers working with authenticator, terminator, Phoenix, OBO
**Last Updated**: February 11, 2026

---

## Overview

Core AuthN is Datadog's centralized authentication and authorization infrastructure, handling ALL authentication traffic for the platform. It consists of four primary services:

| Service | Purpose | Language | Repository |
|---------|---------|----------|------------|
| **Authenticator** | Credential validation, user/org resolution | Go | dd-go/apps/authenticator |
| **Terminator** | Authorization checks, JWT generation | Go | dd-go/apps/terminator |
| **Phoenix** | Session lifecycle management | Go | dd-source/domains/aaa/apps/phoenix |
| **OBO** | On-behalf-of JWT generation | Go | dd-source/domains/aaa_authn/apps/apis/obo |

---

## Service Architecture

### Request Flow

```
┌──────────────────────────────────────────────────────────────┐
│                      CLIENT REQUEST                           │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│                    AUTHENTICATOR                              │
│  • Extracts credentials from request                         │
│  • Validates credentials (DB lookups)                        │
│  • Resolves to user/org context                             │
│  • Returns: ResolvedCredentials                             │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│                     TERMINATOR                                │
│  • Receives ResolvedCredentials from Authenticator           │
│  • Checks permissions (AuthZ)                                │
│  • Validates IP allowlist                                    │
│  • Generates signed JWT                                      │
│  • Returns: JWT in headers                                   │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│                   BACKEND SERVICE                             │
│  • Validates JWT                                             │
│  • Processes request with user/org context                   │
└──────────────────────────────────────────────────────────────┘
```

---

## Authenticator Service

### Responsibility

**Credential validation and user/org resolution**

### Key Concepts

#### 1. AuthN Methods

**Definition**: Types of authentication credentials supported

**Location**: `pkg/routeconfig/authn_methods.go`

**Common Methods**:
- `ValidUser` - Session cookie authentication
- `ValidDatadogAPIUser` - API + Application key pair
- `ValidUserPAT` - Personal Access Token
- `ValidOAuthAccessToken` - OAuth bearer token
- `ValidEmbedToken` - Embed token (passthrough)

**Properties**:
```go
type AuthnMethodProperties struct {
    authenticatorPassthrough    bool  // Skip authenticator validation
    terminatorPassthrough       bool  // Skip terminator authorization
    usesSession                 bool  // Session cookie based
    usesAPIKey                  bool  // API key based
    usesAPIAppKey               bool  // API + App key pair
    usesBearerToken             bool  // OAuth bearer token
    usesPAT                     bool  // Personal Access Token
}
```

#### 2. Credential Resolution Pattern

**Flow**:
```
ParseCheckRequest (extract credentials from request)
    ↓
ResolveCredentials (try each auth method)
    ↓
tryAuthNMethod (dispatcher to specific resolver)
    ↓
resolve{Type} (validate specific credential type)
    ↓
Return ResolvedCredentials (user/org context)
```

**Example Method Resolvers**:
- `resolveCookie()` - Session cookies
- `resolveAPIKey()` - API keys only
- `resolveAPIAppKey()` - API + App key pair
- `resolvePAT()` - Personal Access Tokens
- `resolveAccessToken()` - OAuth access tokens
- `resolveDelegatedToken()` - External JWT tokens

#### 3. ResolvedCredentials Structure

**What Authenticator Returns**:
```go
type ResolvedCredentials struct {
    UserID              int64   // User ID
    OrgID               int64   // Organization ID
    UserUuid            string  // User UUID
    OrgUuid             string  // Organization UUID
    Method              auth_methods.AuthMethod  // Which method succeeded

    // Method-specific data
    SessionData         *models.SessionData       // For session auth
    ApiKeyUuid          string                    // For API key auth
    AppKeyUuid          string                    // For app key auth
    AccessTokenData     *authmodels.AccessTokenData  // For OAuth
    PATUuid             string                    // For PAT auth
}
```

#### 4. Data Sources

**Databases Authenticator Queries**:
- **Cassandra**: Session data (via `sessionsReader`)
- **PostgreSQL** (via libcontext): API keys, App keys, PATs
- **Redis**: Caching layer

**Key Readers**:
- `sessionsReader` - Reads from Cassandra (beaker.sessions table)
- `contextResolver` - Queries libcontext for key resolution
- `accessTokenFingerprintReader` - OAuth token validation

#### 5. Extensibility Patterns

**External Validators** (Dependency Injection):
```go
type CredentialReaders struct {
    sessionsReader            sessions.SessionDBInterface
    contextResolver           context_platform.ContextResolver
    externalTokenJWTValidator *jwks.JWTValidator  // ← Injectable
    ticinoValidator           *authnverify.Authenticator  // ← Injectable
}
```

**Adding External Validation**:
- Inject validator via constructor
- Validator configured via config file
- Authenticator delegates to validator
- Example: `resolveDelegatedToken()` uses `externalTokenJWTValidator`

---

## Terminator Service

### Responsibility

**Authorization checks and JWT generation**

### Key Concepts

#### 1. Authorization Flow

**Steps**:
```
1. Receive ResolvedCredentials from Authenticator
2. Check if passthrough (skip if true)
3. Authenticate (read from Authenticator metadata)
4. Check permissions (AuthZ)
5. Check IP allowlist
6. Generate JWT
7. Return success with JWT in headers
```

#### 2. JWT Generation

**Service**: `JwtGenerationService`

**Flow**:
```
GenerateJWT(AuthResult, RequestContext)
    ↓
Build AuthResponse (user data + metadata)
    ↓
JWTProvider.GetJWT (check cache first)
    ↓
userctx.GenerateSignedJWT (if cache miss)
    ↓
Return signed JWT
```

**JWT Contains**:
- `sub` - User UUID or Org UUID
- `org_id` - Organization ID
- `ddauth` - Private claims (user data, method, permissions)
- `issued_to_route` - Route that will use JWT (scoping)
- `issued_to_service` - Service that will use JWT (scoping)
- `iat`, `exp` - Issue time, expiration time
- `iss`, `aud` - Issuer, audience

#### 3. JWT Caching

**Strategy**: Cache by AuthResponse hash

**Cache Key**:
```
hash(user_uuid + org_uuid + method + route + service + permissions)
```

**Why This Works**:
- Same user + same route + same service → same JWT (cache hit)
- Different route or service → different JWT (cache miss)
- Automatic scoping via cache key

**TTL**: 30 seconds (configurable)

**Feature Flag**: `terminator_jwt_cache_enabled`

#### 4. Authorization Patterns

**Permission Checking**:
- Route-level permissions (from route config)
- MCP tool permissions (optional)
- Unified authorization checker

**IP Allowlist**:
- Org-level feature flag: `ip_allowlist`
- Configured per org
- Validated against client IP

---

## Phoenix Service

### Responsibility

**Centralized session lifecycle management**

### Key Concepts

#### 1. Session Management

**gRPC Interface**:
```
service PhoenixDataPlane {
    rpc CreateSession(CreateSessionRequest) → CreateSessionResponse
    rpc DeleteSession(DeleteSessionRequest) → Empty
    rpc DeleteAllSessionsForUser(DeleteAllSessionsForUserRequest) → Empty
    rpc DebugSession(DebugSessionRequest) → DebugSessionResponse
    rpc GetAllSessionsForUser(GetAllSessionsForUserRequest) → GetAllSessionsForUserResponse
}
```

**CreateSession Flow**:
```
1. Receive SessionKeys (user context + metadata)
2. Build SessionModel (structured session data)
3. Encrypt session data (Beaker encryption)
4. Generate session ID (UUID)
5. Sign session ID (HMAC)
6. Store in Cassandra with TTL
7. Return signed session ID
```

#### 2. Session Storage

**Cassandra Schema**:
```
Table: beaker.sessions
- Partition key: user_uuid
- Clustering key: session_id
- Data: encrypted blob (session metadata)
- TTL: Configurable (default 30 days)
```

**Session Data** (encrypted):
- User information (UUID, ID, handle, org)
- Login metadata (IP, user agent, timestamp)
- Login method (password, SAML, OAuth, etc.)
- Custom fields (extensible via SessionModel)

#### 3. Encryption

**Beaker Encryption**:
- Validate key (HMAC verification)
- Encrypt key (AES encryption)
- Both stored in secrets
- Phoenix handles encryption/decryption

#### 4. Session Lifecycle

**Creation**: Via CreateSession RPC
**Validation**: Authenticator reads from Cassandra
**Expiration**: Automatic (Cassandra TTL)
**Deletion**: Via DeleteSession RPC or TTL expiry

---

## OBO Service

### Responsibility

**Generate User JWTs on behalf of services**

### Key Concepts

#### 1. On-Behalf-Of Pattern

**Use Case**: Service needs to act as a user without having user's credentials

**Flow**:
```
Service (with ISA JWT) → OBO
  "I need a JWT for user X"
  ↓
OBO validates caller (ISA JWT)
  ↓
OBO retrieves user from database
  ↓
OBO generates User JWT
  ↓
Returns JWT to calling service
  ↓
Service uses JWT to access resources as user X
```

#### 2. Caller Authentication

**ISA JWT** (Identity Service Authentication):
- Issued by Fabric (Datadog's service mesh)
- Identifies calling service (not user)
- Validates: Service is allowed to request user JWTs

**Validation**:
```
1. Extract ISA JWT from gRPC metadata
2. Validate signature (Fabric public keys)
3. Check issuer, audience, expiration
4. Identify calling service
5. Authorize service to request JWTs
```

#### 3. User JWT Generation

**Input**:
- `user_uuid` - User to impersonate
- (Optional) `scope` - Restrict JWT usage
- (Optional) `permissions` - Override permissions

**Process**:
```
1. Validate caller's ISA JWT
2. Query user from database (by UUID)
3. Fetch user's permissions (RBAC system)
4. Build JWT claims (user context + permissions)
5. Sign JWT (RSA key)
6. Cache JWT (30s TTL)
7. Return to caller
```

**JWT Format**:
```
{
  "sub": user_uuid,
  "org_id": user's org,
  "usr.handle": user's email,
  "usr.id": user's ID,
  "user_perms": [permissions array],
  "iat": issued at,
  "exp": expiration,
  "iss": "obo.datadoghq.com"
}
```

#### 4. Caching

**Strategy**: Cache by user_uuid

**Why**:
- Multiple services requesting same user → same JWT
- Reduces database load
- Faster response time

**TTL**: 30 seconds

---

## Common Patterns

### Adding New Authentication Methods

**4-Step Process**:

#### Step 1: Define AuthN Method

**Location**: `pkg/routeconfig/authn_methods.go`

**Tasks**:
- Add new `AuthnMethod` constant
- Define `AuthnMethodProperties`
- Add to `authnMethodMapping`
- Add helper function `MethodUses{YourMethod}()`

**Example**:
```go
const (
    ValidNewMethod AuthnMethod = "ValidNewMethod"
)

newMethodAuthentication = AuthnMethodProperties{
    usesSession: true,  // Or appropriate properties
    authenticatorPassthrough: false,
    terminatorPassthrough: false,
}

authnMethodMapping = map[AuthnMethod]AuthnMethodProperties{
    ValidNewMethod: newMethodAuthentication,
}
```

#### Step 2: Add Credential Extraction

**Location**: `pkg/ddextauthz/utils.go`

**Tasks**:
- Add fields to `DDAuthenticationCredentials`
- Extract from request headers/cookies/query params
- Add to `ParseCheckRequest()`

**Example**:
```go
type DDAuthenticationCredentials struct {
    // ... existing fields
    NewMethodToken string  // NEW
}

func ParseCheckRequest(...) (*DDAuthenticationCredentials, error) {
    // ... existing extraction

    newMethodToken := extractFromRequest(req, "x-new-method-token")

    creds := &DDAuthenticationCredentials{
        // ... existing fields
        NewMethodToken: newMethodToken,
    }
}
```

#### Step 3: Implement Resolver in Authenticator

**Location**: `apps/authenticator/datastore/repository/credential_readers.go`

**Tasks**:
- Add to `tryAuthNMethod()` dispatcher
- Implement `resolve{YourMethod}()` function
- Query appropriate database/service
- Return `ResolvedCredentials` with user/org context

**Example**:
```go
func (cr *CredentialReaders) tryAuthNMethod(...) (*model.ResolvedCredentials, error) {
    // ... existing checks

    if routeconfig.MethodUsesNewMethod(method) {
        resolvedCreds, err = cr.resolveNewMethod(ctx, credentials)
        if resolvedCreds != nil || err != nil {
            return resolvedCreds, err
        }
    }
}

func (cr *CredentialReaders) resolveNewMethod(
    ctx context.Context,
    credentials *ddextauthz.DDAuthenticationCredentials,
) (*model.ResolvedCredentials, error) {
    span, ctx := tracer.StartSpanFromContext(ctx, "resolve_new_method")
    defer span.Finish(tracer.WithError(err))

    // Check if credential present
    if credentials.NewMethodToken == "" {
        return nil, nil  // Method not attempted
    }

    // Validate credential (database query, API call, etc.)
    userData, err := cr.validateNewMethodToken(ctx, credentials.NewMethodToken)
    if err != nil {
        return nil, err  // Validation failed
    }

    // Return resolved credentials
    return &model.ResolvedCredentials{
        UserID:   userData.UserID,
        OrgID:    userData.OrgID,
        UserUuid: userData.UserUUID,
        OrgUuid:  userData.OrgUUID,
        Method:   auth_methods.YourNewMethod,
    }, nil
}
```

#### Step 4: Handle in Terminator

**Location**: `apps/terminator/server/resolver.go`

**Tasks**:
- Add to `handleCredsByAuthnMethod()` dispatcher
- Implement handler function
- Build `AuthResult`
- Return for JWT generation

**Example**:
```go
func (dataStores *dataStores) handleCredsByAuthnMethod(...) (*models.AuthResult, error) {
    // ... existing handlers

    if routeconfig.MethodUsesNewMethod(method) {
        ar, err = dataStores.handleNewMethodRequest(ctx, creds, requestContext)
        if ar != nil || err != nil {
            return ar, err
        }
    }
}

func (dataStores *dataStores) handleNewMethodRequest(...) (*models.AuthResult, error) {
    // Build AuthResult from Authenticator's resolved credentials
    authResult := &models.AuthResult{
        UserData: &authctxmodels.UserData{
            UserID:      resolvedCreds.UserID,
            OrgID:       resolvedCreds.OrgID,
            UserUUID:    resolvedCreds.UserUuid,
            OrgUUID:     resolvedCreds.OrgUuid,
            Permissions: resolvedCreds.Permissions,
        },
        Method: auth_methods.YourNewMethod,
    }

    return authResult, nil
}
```

---

### Session-Based Authentication

**How Authenticator Validates Sessions**:

**1. Extract Session Cookie**:
```go
cookie := credentials.SessionCookie  // From "dogweb" cookie
```

**2. Read from Cassandra** (via sessionsReader):
```go
sessionData, encryptedData, err := cr.sessionsReader.ReadSessionWithMetadata(ctx, cookie)
```

**3. Caching Strategy**:
- In-memory cache (fastest)
- Redis cache (fast)
- Cassandra query (slower, cache miss only)

**4. Session Validation**:
- Cookie signature verification (HMAC)
- Session exists in Cassandra
- Session not expired (TTL-based)
- Session not marked invalid

**5. Return User Context**:
```go
return &model.ResolvedCredentials{
    UserID:       sessionData.UserID,
    OrgID:        sessionData.OrgID,
    UserUuid:     sessionData.UserUUID,
    OrgUuid:      sessionData.OrgUUID,
    SessionData:  sessionData,
    Method:       auth_methods.SessionMethod,
}
```

---

### API Key Authentication

**How Authenticator Validates API Keys**:

**1. Hash API Key** (security):
```go
apiKeyFingerprints, err := cr.tokenHasher.TokenHashes(apiKey)
// Returns multiple hashes (different algorithms for migration)
```

**2. Resolve via libcontext** (PostgreSQL):
```go
result, err := cr.contextResolver.ResolveApiKey(ctx, apiKeyFingerprints, requestContext)
```

**3. Return with Key Context**:
```go
return &model.ResolvedCredentials{
    UserID:            result.UserID,
    OrgID:             result.OrgID,
    UserUuid:          result.UserUUID,
    OrgUuid:           result.OrgUUID,
    ApiKeyUuid:        result.ApiKeyUuid,
    ApiKeyFingerprint: result.ApiKeyFingerprint,
    Method:            auth_methods.APIKeyMethod,
}
```

**Usage Tracking**:
- API key usage aggregated (for billing/analytics)
- Aggregator updates on each API key use

---

## Terminator Service

### JWT Generation Patterns

#### 1. Standard JWT Generation

**Entry Point**:
```go
jwt, err := jwtSvc.GenerateJWT(ctx, authResult, requestContext)
```

**Process**:
```
1. Build AuthResponse (user/org + context)
2. Hash AuthResponse (for cache key)
3. Check cache (30s TTL)
4. If miss: Generate new JWT
5. Cache JWT
6. Return JWT
```

#### 2. JWT Scoping via Route Context

**JWT Claims Include**:
- `issued_to_route` - Route path (e.g., "/api/v1/metrics")
- `issued_to_service` - Backend service name
- User/Org context
- Permissions (from RBAC)

**Scoping Benefit**:
- JWT valid only for specific route + service
- Cannot be reused for other endpoints
- Provides defense in depth

#### 3. JWT Caching

**Cache Key Computation**:
```
hash(
    user_uuid +
    org_uuid +
    method +
    route +
    service +
    permissions
)
```

**Benefits**:
- Same context → cache hit
- Different context → cache miss (security)
- Automatic scoping via cache key

**Performance**:
- Cache hit: ~0.5ms
- Cache miss: ~1ms (JWT generation)

#### 4. Permission Filtering

**Filters Applied**:
- `DisablePermissions` - Remove specific permissions
- `BuiltInFeatures` - Add/remove based on feature flags

**When Applied**:
- After authentication succeeds
- Before JWT generation
- Ensures JWT has correct permissions

---

## Phoenix Service

### Session Management

#### 1. Creating Sessions

**gRPC Call**:
```
Phoenix.CreateSession({
    SessionKeys: {
        user_uuid, user_id, user_handle, org_id, org_uuid,
        login_method, ip, user_agent, last_login_time,
        session_ttl
    }
})
```

**Process**:
```
1. Build SessionModel from SessionKeys
2. Convert to map (key-value pairs)
3. Encrypt with Beaker keys
4. Generate session ID (UUID without dashes)
5. Sign session ID (HMAC with cookie secret)
6. Store in Cassandra (user_uuid partition, session_id clustering)
7. Set TTL on row
8. Return signed session ID
```

**Result**: Signed session ID (cookie value)

#### 2. Session Structure

**SessionModel Fields**:
- **User Context**: user_uuid, user_id, user_handle, org_id
- **Session Metadata**: session_id, creation_time, login_method
- **Request Context**: ip, user_agent, last_login_time
- **Auth Context**: authentication_token (optional)
- **Custom Fields**: Extensible (can add more fields)

**Encryption**:
- All fields encrypted in Cassandra blob
- Only Phoenix and Authenticator can decrypt
- Beaker encryption keys stored in secrets

#### 3. Session Validation

**Read by Authenticator**:
```
sessionsReader.ReadSessionWithMetadata(cookie)
    ↓
1. Validate cookie signature (HMAC)
2. Check cache (in-memory, then Redis)
3. If miss: Query Cassandra
4. Decrypt session data
5. Validate session not expired (implicit via TTL)
6. Return SessionData
```

**Caching Layers**:
- In-memory cache (fastest, local to pod)
- Redis cache (shared across pods)
- Cassandra (source of truth)

#### 4. Session Revocation

**Methods**:
- `DeleteSession(user_uuid, session_id)` - Delete specific session
- `DeleteAllSessionsForUser(user_uuid)` - Delete all user sessions
- TTL expiration - Automatic after configured time

**Use Cases**:
- User logout (delete specific session)
- User password change (delete all sessions)
- Security incident (force logout)

---

## OBO Service

### On-Behalf-Of JWT Generation

#### 1. Use Cases

**When to Use OBO**:
- Service needs to act as a user
- Service has user UUID but not credentials
- Need User JWT for downstream service calls

**Common Scenarios**:
- Backend service querying data on user's behalf
- Service-to-service calls with user context
- Impersonation for testing/support

#### 2. Authentication Flow

**Caller Authentication** (ISA JWT):
```
1. Calling service includes ISA JWT in gRPC metadata
2. OBO extracts ISA JWT
3. OBO validates:
   - Signature (Fabric keys)
   - Issuer (Fabric)
   - Audience (OBO service)
   - Expiration
4. Extract service identity from claims
5. Authorize service to request user JWTs
```

#### 3. User JWT Generation

**Input**:
- `user_uuid` - User to impersonate (required)
- `scope` - Optional JWT scope restriction
- `permissions` - Optional permission override

**Process**:
```
1. Validate caller (ISA JWT)
2. Retrieve user from database (by UUID)
3. Verify user is active (not disabled)
4. Fetch user permissions (RBAC system)
5. Build JWT claims (user + permissions)
6. Sign JWT (RSA private key)
7. Cache JWT (by user_uuid, 30s TTL)
8. Return JWT
```

**Caching**:
- Key: user_uuid
- Value: Generated JWT
- TTL: 30 seconds
- Shared across all callers

#### 4. Permission Scoping

**Scoped JWTs** (if supported):
```
Request: GetOboUserJwt({
    user_uuid: "user-123",
    scope: "dashboard:abc-xyz",
    permissions: ["metrics_read", "logs_read"]
})

Returns JWT with:
- Full user context
- Restricted permissions (not all owner perms)
- Scope claim (dashboard-specific)
```

**Benefits**:
- Least privilege enforcement
- Limited blast radius if JWT leaked
- Fine-grained access control

---

## Data Flow Patterns

### 1. Session Authentication

```
Request with cookie
    ↓
Authenticator:
    Extract cookie → Read Cassandra → Decrypt → Return user/org
    ↓
Terminator:
    Receive user/org → Generate JWT → Return
```

### 2. API Key Authentication

```
Request with API key
    ↓
Authenticator:
    Extract key → Hash → Query libcontext → Return user/org
    ↓
Terminator:
    Receive user/org → Generate JWT → Return
```

### 3. OAuth Authentication

```
Request with bearer token
    ↓
Authenticator:
    Extract token → Query OAuth DB → Return user/org
    ↓
Terminator:
    Receive user/org → Generate JWT → Return
```

### 4. Service-to-User Pattern (OBO)

```
Service has user_uuid
    ↓
Service calls OBO with ISA JWT + user_uuid
    ↓
OBO:
    Validate ISA JWT → Query user → Generate JWT → Return
    ↓
Service uses User JWT to access resources
```

---

## Database Schema Patterns

### Sessions (Cassandra)

**Table**: `beaker.sessions`

**Schema**:
```
CREATE TABLE beaker.sessions (
    user_uuid text,        -- Partition key
    session_id text,       -- Clustering key
    data blob,             -- Encrypted session data
    PRIMARY KEY (user_uuid, session_id)
) WITH default_time_to_live = 2592000;
```

**Access Pattern**:
- Partition key: user_uuid (distributes load)
- Clustering key: session_id (multiple sessions per user)
- TTL: Auto-deletes expired sessions

### API Keys (PostgreSQL via libcontext)

**Tables**:
- `api_keys2` - API key metadata
- `application_key2` - Application key metadata

**Access Pattern**:
- Query by fingerprint (hashed key)
- Returns user_id, org_id, permissions

### OAuth Tokens (Cassandra)

**Table**: Access token fingerprints

**Access Pattern**:
- Query by token fingerprint
- Returns user_id, org_id, scopes

---

## Performance Characteristics

### Latency Breakdown

**Session Authentication**:
```
Parse request:           ~0.1ms
Session read (cached):   ~0.1ms (in-memory)
Session read (Redis):    ~0.5ms
Session read (Cassandra): ~1-2ms
JWT generation (cached): ~0.5ms
JWT generation (new):    ~1ms

Total (best case):  ~0.7ms
Total (worst case): ~3-4ms
```

**API Key Authentication**:
```
Parse request:           ~0.1ms
Hash API key:            ~0.2ms
Query libcontext:        ~2-3ms
JWT generation (cached): ~0.5ms

Total: ~3-4ms
```

**OAuth Authentication**:
```
Parse request:           ~0.1ms
Hash token:              ~0.2ms
Query OAuth DB:          ~1-2ms
JWT generation (cached): ~0.5ms

Total: ~2-3ms
```

### Caching Strategy

**Multi-Layer Caching**:

**Sessions**:
1. In-memory cache (pod-local, ~0.1ms)
2. Redis cache (shared, ~0.5ms)
3. Cassandra (source of truth, ~1-2ms)

**JWTs**:
1. Terminator cache (TTL: 30s, ~0.5ms)
2. OBO cache (TTL: 30s, ~0.5ms)

**Credentials**:
1. libcontext cache (varies by credential type)

---

## Monitoring and Observability

### Key Metrics

**Authenticator**:
- `dd.authenticator.request` - Request count (by method, status)
- `dd.authenticator.latency` - Request latency distribution
- `dd.authenticator.{method}.success/failure` - Per-method outcomes

**Terminator**:
- `dd.terminator.request` - Request count (by method, status)
- `dd.terminator.latency` - Request latency distribution
- `dd.terminator.jwt_provider.cache` - JWT cache hit/miss ratio
- `dd.terminator.ip_allowlist.*` - IP allowlist metrics

**Phoenix**:
- `dd.phoenix.create_session.latency` - Session creation time
- `dd.phoenix.delete_sessions.*` - Session deletion metrics

**OBO**:
- `dd.obo.user_jwt.generated` - JWT generation count
- `dd.obo.cache.*` - OBO cache metrics

### Dashboards

**Primary Dashboards**:
- AAA On Call dashboard (operations)
- AAA AuthN Overview (metrics)
- Service-specific dashboards (authenticator, terminator, etc.)

### Logging

**Standard Log Fields**:
- `org_id`, `org_uuid` - Organization context
- `usr.id`, `usr.uuid`, `usr.handle` - User context
- `cred_method` - Authentication method used
- `auth_error` - Error details (if failed)

**Structured Logging**:
```go
observability_accumulator.AddBothLogAndMetricField(ctx, "method", methodName)
observability_accumulator.AddLogField(ctx, log.String("user_uuid", userUuid))

logFields := observability_accumulator.GetLogFields(ctx)
log.FromContext(ctx).Info("Authentication succeeded", logFields...)
```

---

## Configuration Patterns

### Config Sections

**Namespaced Configuration**:
```go
const (
    sectionName = "dd.authenticator.feature_name"
)

// Typed getters with defaults
uri := cfg.GetDefault(sectionName, "uri", defaultURI)
size := cfg.GetInt64Default(sectionName, "cache_size", 10000)
enabled := cfg.GetBoolDefault(sectionName, "enabled", false)
```

### Feature Flags

**Usage**:
```go
import "go.ddbuild.io/dd-source/x/libs/go/experiments"

const EnableFeatureFF = "authenticator_enable_feature"

// Per-org evaluation
expCtx := experiments.WithExperimentContext(ctx, &experiments.ExperimentContext{
    OrgId: orgID,
})

if experimentsClient.IsEnabled(expCtx, EnableFeatureFF) {
    // Feature logic
}
```

**Common Feature Flags**:
- `authenticator_*` - Authenticator features
- `terminator_*` - Terminator features
- `ip_allowlist` - IP allowlist enforcement
- `terminator_jwt_cache_enabled` - JWT caching

---

## Testing Patterns

### Table-Driven Tests

**Standard Pattern**:
```go
func TestFeature(t *testing.T) {
    tests := []struct {
        name     string
        input    InputType
        expected OutputType
        wantErr  error
    }{
        {
            name:     "valid input",
            input:    validInput,
            expected: expectedOutput,
            wantErr:  nil,
        },
        {
            name:     "invalid input",
            input:    invalidInput,
            expected: nil,
            wantErr:  ErrExpected,
        },
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

### Mock Pattern

**Using gomock**:
```go
type testSuite struct {
    suite.Suite
    ctrl          *gomock.Controller
    mockDB        *mocks.MockDatabase
    mockValidator *mocks.MockValidator
    service       *Service
}

func (s *testSuite) SetupTest() {
    s.ctrl = gomock.NewController(s.T())
    s.mockDB = mocks.NewMockDatabase(s.ctrl)
    s.service = NewService(s.mockDB)
}

func (s *testSuite) TearDownTest() {
    s.ctrl.Finish()
}

func TestSuite(t *testing.T) {
    suite.Run(t, new(testSuite))
}
```

---

## Common Pitfalls

### 1. Passthrough Methods

**Problem**: Passthrough methods bypass validation

**Pattern**:
```go
ValidEmbedToken: fullPassthrough  // Both authenticator and terminator skip
```

**When to Use**:
- Legacy methods (temporary)
- External validation (handled elsewhere)
- Open/optional auth routes

**When to Avoid**:
- New authentication methods
- Security-critical paths
- Production features

### 2. Error Handling

**Pattern**: Return nil for "not attempted", error for "attempted but failed"

**Correct**:
```go
func (cr *CredentialReaders) resolveMethod(...) (*model.ResolvedCredentials, error) {
    if credentials.Token == "" {
        return nil, nil  // Method not attempted
    }

    // Validate token
    if invalid {
        return nil, ErrInvalidToken  // Attempted but failed
    }

    return &model.ResolvedCredentials{...}, nil  // Success
}
```

**Why This Matters**:
- Allows multiple auth methods to be tried
- First error is reported (not last)
- Metrics and logs show which method was attempted

### 3. Context Propagation

**Always Use Context**:
```go
span, ctx := tracer.StartSpanFromContext(ctx, "operation_name")
defer span.Finish(tracer.WithError(err))

// Pass ctx to all downstream calls
result, err := someFunction(ctx, ...)
```

**Benefits**:
- Distributed tracing
- Context-aware logging
- Timeout propagation

---

## Integration Points

### Authenticator → Terminator

**Via Envoy Metadata**:
```
Authenticator sets dynamic metadata:
- Resolved credentials (user/org context)
- Session data (encrypted, for JWT)
- Auth method attempted
- Error details (if failed)

Terminator reads from metadata:
- No direct gRPC call between them
- Envoy passes metadata
```

### Terminator → Phoenix

**Via gRPC**:
```
Terminator calls Phoenix.CreateSession()
Phoenix stores in Cassandra
Returns signed session ID
Terminator sets cookie in response
```

### Terminator → OBO

**Via gRPC**:
```
Terminator calls OBO.GetOboUserJwt()
OBO generates User JWT
Returns JWT to Terminator
Terminator includes in response headers
```

### Authenticator → Database

**Direct Queries**:
- Cassandra: Session reads (via sessionsReader)
- PostgreSQL: Credential reads (via libcontext)
- Redis: Caching, nonce tracking

---

## Best Practices

### 1. Credential Validation

**Always**:
- Hash credentials before database lookup (security)
- Use constant-time comparison (timing attack protection)
- Log authentication attempts (audit trail)
- Emit metrics (observability)

### 2. Session Management

**Always**:
- Use Phoenix for session lifecycle (don't DIY)
- Let Cassandra handle TTL (don't manual cleanup)
- Cache aggressively (sessions read frequently)
- Encrypt sensitive data (Beaker encryption)

### 3. JWT Generation

**Always**:
- Use Terminator or OBO (centralized)
- Include route/service context (scoping)
- Cache by context hash (performance)
- Set appropriate TTL (typically 1 hour)

### 4. Error Handling

**Pattern**:
```go
if err != nil {
    switch {
    case errors.Is(err, database.ErrNotFound):
        return nil, fmt.Errorf("%w: credential not found", ErrUnauthorized)
    case errors.Is(err, database.ErrExpired):
        return nil, fmt.Errorf("%w: credential expired", ErrUnauthorized)
    default:
        return nil, err
    }
}
```

**Don't leak information**:
- Return generic errors to clients
- Log detailed errors server-side
- Don't expose internal state

---

## Reference

### Key Repositories

- **dd-go**: Authenticator, Terminator services
- **dd-source**: Phoenix, OBO services
- **dd-source/x/domains/aaa**: AAA shared libraries

### Key Packages

- `pkg/routeconfig` - AuthN method definitions
- `pkg/ddextauthz` - Credential parsing utilities
- `pkg/sessions` - Session database interface
- `x/domains/aaa/libs/terminator` - Terminator libraries
- `x/domains/aaa/apps/phoenix` - Phoenix service

### Key Patterns

- **Plugin-based resolution**: Authenticator credential resolvers
- **Dependency injection**: External validators
- **Interface-based design**: Swappable implementations
- **Caching everywhere**: Multi-layer caching for performance

---

## Summary

### Core AuthN Architecture Principles

1. **Separation of Concerns**:
   - Authenticator: "Who are you?" (AuthN)
   - Terminator: "What can you do?" (AuthZ)
   - Phoenix: "Manage your session"
   - OBO: "Act on behalf of users"

2. **Defense in Depth**:
   - Multiple validation layers
   - Each service validates its inputs
   - No single point of failure

3. **Performance via Caching**:
   - Cache at every layer
   - Intelligent cache keys
   - Appropriate TTLs

4. **Observability First**:
   - Structured logging
   - Comprehensive metrics
   - Distributed tracing

5. **Extensibility via Patterns**:
   - New auth methods follow standard pattern
   - Dependency injection for validators
   - Interface-based design
