# ISA JWT Reference Guide

## What is ISA JWT?

**ISA (Internal Service Authentication) JWT** is a JSON Web Token issued by **Fabric** (Datadog's service mesh / zero-trust networking infrastructure) to authenticate internal Kubernetes services and Datadog employees when they call other internal services.

ISA JWTs provide a standardized authentication mechanism that works for both:
- **Services** (Kubernetes pods with service accounts)
- **Humans** (Datadog employees via OIDC/SSO)

## ISA JWT vs User JWT

| Aspect | ISA JWT | User JWT |
|--------|---------|----------|
| **Issuer** | Fabric (Emissary/Sycamore) | Terminator (or OBO) |
| **Purpose** | Service-to-service authentication | User context for API requests |
| **Contains** | Service/human identity (K8s service account or employee email) | User/org context (permissions, roles) |
| **Validated by** | `go-service-authn` library | `authctx` library |
| **Audience** | Service name (e.g., "rapid-obo", "rapid-fabric") | "datadog-public" |
| **Injected by** | Fabric Emissary sidecar (automatic for services) | Application (via Terminator/OBO) |
| **Used for** | Proving service/human identity | Proving user identity/permissions |

## How ISA JWTs are Issued

### For Services (Kubernetes Pods)

Services get ISA JWTs automatically from **Kubernetes projected service account tokens**:

**Flow**:
```
┌─────────────────────────────────────────────────────────┐
│ Kubernetes Pod (e.g., retriever)                        │
│ ↓ Has service account: retriever-sa                     │
└─────────────────────────────────────────────────────────┘
                      │
                      │ Kubernetes issues projected SA token
                      ▼
┌─────────────────────────────────────────────────────────┐
│ Fabric Emissary Sidecar                                 │
│ ↓ Reads K8s service account token                       │
│ ↓ Exchanges for ISA JWT via Fabric/Sycamore            │
│ ↓ Automatically injects into outgoing requests          │
└─────────────────────────────────────────────────────────┘
```

**Service Identity Example**:
```json
{
  "iss": "sycamore",
  "sub": "retriever.svc-789@us1.prod.dog",
  "aud": "rapid-obo",
  "email": "retriever.svc-789@us1.prod.dog",
  "groups": ["query-services", "xpq-team"],
  "exp": 1234567890
}
```

**Key Characteristics**:
- **Email format**: `service-name.svc-N@datacenter.dog`
- **Automatic**: No manual token management required
- **Scoped**: Each outgoing request gets audience-specific token
- **Short-lived**: Typically 15-60 minutes

### For Humans (Datadog Employees)

Humans get ISA JWTs through **`ddtool auth login`** (OIDC flow):

**Flow**:
```
┌─────────────────────────────────────────────────────────┐
│ Developer's Machine                                      │
│ $ ddtool auth login --datacenter us1.staging.dog        │
└─────────────────────────────────────────────────────────┘
                      │
                      │ 1. Redirect to OIDC provider
                      ▼
┌─────────────────────────────────────────────────────────┐
│ Datadog Identity Provider (OIDC/SSO)                    │
│ ↓ User authenticates with Datadog credentials           │
│ ↓ Returns OIDC ID token with email                      │
└─────────────────────────────────────────────────────────┘
                      │
                      │ 2. ID token with piyush.jindal@datadoghq.com
                      ▼
┌─────────────────────────────────────────────────────────┐
│ Fabric (Sycamore Issuer)                                │
│ ↓ Validates OIDC token                                  │
│ ↓ Issues ISA JWT with user's email                      │
│ ↓ Cached locally in ~/.ddtool/                          │
└─────────────────────────────────────────────────────────┘
                      │
                      │ 3. ISA JWT ready for use
                      ▼
┌─────────────────────────────────────────────────────────┐
│ Developer uses ddtool commands                          │
│ $ ddtool auth token rapid-obo                           │
└─────────────────────────────────────────────────────────┘
```

**Human Identity Example**:
```json
{
  "iss": "sycamore",
  "sub": "piyush.jindal@datadoghq.com",
  "aud": "rapid-obo",
  "email": "piyush.jindal@datadoghq.com",
  "groups": ["core-authn-team", "eng-team"],
  "exp": 1234567890
}
```

**Key Characteristics**:
- **Email format**: `user.name@datadoghq.com`
- **Manual**: Requires `ddtool auth login`
- **Cached**: Token stored locally, reused until expiration
- **Multi-audience**: Can request tokens for different audiences

## Token Issuers

ISA JWTs can be issued by multiple token issuers (run `ddtool auth issuers list`):

| Issuer | Description | Primary Use |
|--------|-------------|-------------|
| **sycamore** (default) | Globally federated OIDC issuer | Primary ISA JWT issuer for most services |
| **classic** | Vault-based tokens (same DC only) | Legacy, being phased out |
| **ironwood** | Sandbox resource access | Development/testing |
| **maple** | OAuth delegation | Special delegation scenarios |
| **blackthorn** | External system integration | MCP servers, external tools |
| **oleander** | Customer OAuth | Customer-facing APIs (not ISA) |

**Default**: Most services use **sycamore** for ISA JWTs.

## How ISA JWTs are Used

### 1. Service Includes ISA JWT in Request

**Automatic (Production)**:
- Fabric Emissary sidecar automatically injects ISA JWT into gRPC metadata or HTTP headers
- Service code doesn't need to manage tokens

**Manual (Development)**:
```bash
# Get ISA JWT for specific audience
export ISA_JWT=$(ddtool auth token rapid-obo --datacenter us1.staging.dog)

# Use in gRPC call
grpcurl -H "authorization: Bearer $ISA_JWT" \
  obo.us1.staging.dog:443 \
  obo.GetOboUserJwt
```

### 2. Receiving Service Validates ISA JWT

Services use the **`go-service-authn`** library to validate ISA JWTs and extract identity:

**Setup (from ZTN authz library)**:
```go
import (
    "github.com/DataDog/go-service-authn/pkg/serviceauthentication/authnverify"
    "github.com/DataDog/dd-source/domains/ztn/shared/libs/go/authz"
)

// Authentication interceptor (validates ISA JWT)
authenticator, err := authnverify.NewAuthenticatorWith(authnOptions...)
authnInterceptors, err := authenticator.Grpc(pb.File_proto)

// Authorization interceptor (checks permissions)
authorizer, err := authz.NewAuthorizerWith(options...)

// Order is critical: AuthN MUST go before AuthZ
opts := []grpc.ServerOption{
    grpc.ChainUnaryInterceptor(authnInterceptors.Unary()),   // ← Validates ISA JWT
    grpc.ChainStreamInterceptor(authnInterceptors.Stream()),
    grpc.ChainUnaryInterceptor(authorizer.Unary()),          // ← Checks permissions
    grpc.ChainStreamInterceptor(authorizer.Stream()),
}

server := grpc.NewServer(opts...)
```

**Extracting Identity**:
```go
import "github.com/DataDog/go-service-authn/pkg/serviceauthentication/authnverify"

func MyHandler(ctx context.Context, req *Request) (*Response, error) {
    // Extract validated K8s user from context
    k8sUser := authnverify.GetUserPtr(ctx)

    // For services: k8sUser.Email = "retriever.svc-789@us1.prod.dog"
    // For humans:   k8sUser.Email = "piyush.jindal@datadoghq.com"

    // Check if caller is human vs service
    isHuman := strings.HasSuffix(k8sUser.Email, "@datadoghq.com")

    // Extract identity name (before '@')
    identityName := strings.Split(k8sUser.Email, "@")[0]

    // Access groups
    groups := k8sUser.Groups  // ["team-backend", "query-services"]

    // Use identity for authorization decisions
    if isHuman && !isAuthorized(identityName) {
        return nil, errors.New("unauthorized")
    }

    return &Response{}, nil
}
```

### 3. Example: OBO Service Validates ISA JWT

**Location**: `~/dd/dd-source/domains/aaa_authn/apps/apis/obo/internal/authorization/identity_provider.go`

```go
func (i *identityProviderImpl) GetK8sIdentity(ctx context.Context) (ClientIdentity, error) {
    // Extract K8s user from context (populated by go-service-authn)
    k8sUser := authnverify.GetUserPtr(ctx)
    if k8sUser == nil {
        return ClientIdentity{}, ErrK8sUserNotFound
    }

    // Parse identity from email
    identity, err := identityFromEmail(k8sUser.Email)
    if err != nil {
        return ClientIdentity{}, err
    }

    // Determine if human
    isHuman := strings.HasSuffix(k8sUser.Email, "@datadoghq.com")

    return ClientIdentity{
        IdentityName: identity,      // "service-name.svc-123" or "piyush.jindal"
        Groups:       k8sUser.Groups,
        IsHuman:      isHuman,
        Email:        k8sUser.Email,
    }, nil
}

func isHuman(email string) bool {
    return strings.HasSuffix(email, "@datadoghq.com")
}
```

## How OBO Uses ISA JWTs

When a service or human calls OBO to get a User JWT:

**Flow**:
```
┌─────────────────────────────────────────────────────────┐
│ Calling Service/Human                                    │
│ Has: ISA JWT from Fabric (automatically in metadata)    │
└─────────────────────────────────────────────────────────┘
                      │
                      │ gRPC: GetOboUserJwt(user_uuid)
                      │ Metadata: ISA JWT
                      ▼
┌─────────────────────────────────────────────────────────┐
│ OBO Service                                              │
│ Step 1: Validate ISA JWT signature                      │
│         (via go-service-authn library)                   │
│                                                           │
│ Step 2: Extract service identity                         │
│         k8sUser.Email = "retriever.svc-42" or           │
│                        "piyush.jindal@datadoghq.com"    │
│                                                           │
│ Step 3: Check allowlist                                  │
│         Is this service/human allowed to impersonate    │
│         the requested user_uuid?                         │
│                                                           │
│ Step 4: Generate User JWT                                │
│         Signed by Terminator with user context           │
└─────────────────────────────────────────────────────────┘
                      │
                      │ Returns User JWT
                      ▼
┌─────────────────────────────────────────────────────────┐
│ Calling Service                                          │
│ Uses User JWT in dd-auth-jwt header for API requests    │
└─────────────────────────────────────────────────────────┘
```

**Key Points**:
1. **ISA JWT proves service identity** to OBO
2. **OBO validates ISA JWT** before checking authorization
3. **Authorization is separate** from authentication
4. **User JWT is different** - issued by Terminator, not Fabric

## Authorization Modes

From `~/dd/dd-source/domains/ztn/shared/libs/go/authz/authz.go`:

Services can use two authorization modes:

```go
const (
    mTLS mode = iota  // mTLS certificate-based auth
    isa               // ISA JWT-based auth (default)
)

// Authorization reasons
const (
    ISA Reason = "ISA"    // Request allowed based on ISA identity
    MTLS Reason = "mTLS"  // Request allowed based on mTLS identity
)
```

Most modern services use **ISA mode** (default).

## Common Use Cases

### 1. Service-to-Service Calls (Production)

**Scenario**: Retriever service calls OBO to get user JWT

```
Retriever Pod
  ↓ Fabric Emissary auto-injects ISA JWT
  ↓ gRPC call to OBO
OBO validates ISA JWT
  ↓ Checks: retriever.svc-789 is on allowlist
  ↓ Returns User JWT
Retriever uses User JWT to call downstream APIs
```

**Code**: None needed - Emissary handles everything

### 2. Developer CLI Tools (Local Dev)

**Scenario**: Developer uses CLI tool to test OBO

```bash
# Step 1: Login to get ISA JWT
ddtool auth login --datacenter us1.staging.dog

# Step 2: Get ISA JWT for OBO audience
export ISA_JWT=$(ddtool auth token rapid-obo --datacenter us1.staging.dog)

# Step 3: Call OBO with ISA JWT
grpcurl -H "authorization: Bearer $ISA_JWT" \
  obo.us1.staging.dog:443 \
  obo.GetOboUserJwt \
  -d '{"user_uuid": "abc-123"}'
```

### 3. Internal Tools (Scripts/Automation)

**Scenario**: Automation script needs to call internal API

```go
import (
    "os/exec"
    "strings"
)

func getISAToken(audience, datacenter string) (string, error) {
    cmd := exec.Command("ddtool", "auth", "token", audience,
        "--datacenter", datacenter)
    output, err := cmd.Output()
    if err != nil {
        return "", err
    }
    return strings.TrimSpace(string(output)), nil
}

func main() {
    token, err := getISAToken("rapid-obo", "us1.staging.dog")
    if err != nil {
        log.Fatal(err)
    }

    // Use token in API calls
    client := createGRPCClient(token)
    resp, err := client.GetOboUserJwt(ctx, &req)
}
```

## Identity Patterns

### Service Identity Pattern

```
Format: {service-name}.svc-{instance}@{datacenter}.{environment}.dog

Examples:
- retriever.svc-789@us1.prod.dog
- obo.svc-42@eu1.staging.dog
- terminator.svc-1@us5.gov.dog

Parsing:
- Service name: Everything before ".svc-"
- Instance ID: Number after ".svc-"
- Datacenter: Between @ and first .
- Environment: Between datacenter and .dog
```

### Human Identity Pattern

```
Format: {first}.{last}@datadoghq.com

Examples:
- piyush.jindal@datadoghq.com
- jane.doe@datadoghq.com

Detection:
- isHuman := strings.HasSuffix(email, "@datadoghq.com")
```

## Security Considerations

### ISA JWT Validation

Services MUST validate ISA JWTs:
1. **Signature verification** - Verify JWT is signed by Fabric
2. **Expiration check** - Reject expired tokens
3. **Audience validation** - Ensure JWT is for this service
4. **Issuer validation** - Verify issuer is trusted (sycamore, etc.)

**Do NOT**:
- ❌ Trust ISA JWT without validation
- ❌ Use expired tokens
- ❌ Skip audience checks
- ❌ Parse JWT manually (use `go-service-authn`)

### Authorization vs Authentication

ISA JWT provides **authentication** (who is calling), not **authorization** (what they can do):

```go
// ✅ CORRECT: Validate ISA JWT, then check authorization
func MyHandler(ctx context.Context, req *Request) (*Response, error) {
    // Authentication: Who is calling?
    k8sUser := authnverify.GetUserPtr(ctx)  // ISA JWT already validated

    // Authorization: Are they allowed?
    if !isAuthorized(k8sUser.Email, req.Action) {
        return nil, errors.New("unauthorized")
    }

    return doWork(ctx, req)
}

// ❌ INCORRECT: ISA JWT doesn't grant permissions
func BadHandler(ctx context.Context, req *Request) (*Response, error) {
    k8sUser := authnverify.GetUserPtr(ctx)
    // No authorization check - anyone with valid ISA JWT can proceed!
    return doWork(ctx, req)
}
```

### Token Lifetime

ISA JWTs have short lifetimes:
- **Services**: 15-60 minutes (auto-renewed by Emissary)
- **Humans**: 1-12 hours (must re-login via ddtool)

**Best Practices**:
- Don't cache ISA JWTs long-term
- Let Emissary handle renewal for services
- Re-run `ddtool auth login` if token expires

## Troubleshooting

### Common Issues

**Problem**: "unauthorized" error when calling service

**Solution**: Check ISA JWT validity
```bash
# Get token
TOKEN=$(ddtool auth token rapid-obo --datacenter us1.staging.dog)

# Decode JWT (don't verify, just inspect)
echo $TOKEN | cut -d. -f2 | base64 -d | jq

# Check expiration
echo $TOKEN | cut -d. -f2 | base64 -d | jq '.exp' | xargs -I {} date -r {}
```

**Problem**: "user not found" in OBO

**Solution**: Check if you're logged in
```bash
# Check current auth status
ddtool auth whoami --datacenter us1.staging.dog

# Re-login if needed
ddtool auth login --datacenter us1.staging.dog
```

**Problem**: ISA JWT works locally but not in K8s

**Solution**: Check service account configuration
```yaml
# service.datadog.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-sa
  annotations:
    # Ensure Fabric is enabled
    fabric.datadoghq.com/enabled: "true"
```

## References

### Code Locations

| Component | Location |
|-----------|----------|
| **ISA JWT Validation** | `github.com/DataDog/go-service-authn/pkg/serviceauthentication/authnverify` |
| **OBO Identity Extraction** | `~/dd/dd-source/domains/aaa_authn/apps/apis/obo/internal/authorization/identity_provider.go` |
| **ZTN Authorization** | `~/dd/dd-source/domains/ztn/shared/libs/go/authz/authz.go` |
| **Fabric Config** | Service's `service.datadog.yaml` |

### Tools

| Tool | Command | Purpose |
|------|---------|---------|
| **ddtool** | `ddtool auth login` | Get ISA JWT for humans |
| **ddtool** | `ddtool auth token <audience>` | Get audience-specific ISA JWT |
| **ddtool** | `ddtool auth whoami` | Check current identity |
| **ddtool** | `ddtool auth issuers list` | List available issuers |
| **grpcurl** | `grpcurl -H "authorization: Bearer $TOKEN"` | Test gRPC with ISA JWT |

### Related Documentation

- **Fabric Documentation**: https://datadoghq.atlassian.net/wiki/spaces/FABRIC
- **ZTN (Zero Trust Networking)**: https://datadoghq.atlassian.net/wiki/spaces/ZTN
- **go-service-authn**: https://github.com/DataDog/go-service-authn
- **OBO Service**: `~/dd/dd-source/domains/aaa_authn/apps/apis/obo/`
- **Terminator**: `~/dd/dd-go/apps/terminator/`

## Key Takeaways

1. **ISA JWTs are issued by Fabric**, not OBO or Terminator
2. **Two types**: Service ISA JWTs (auto) and Human ISA JWTs (via ddtool)
3. **Identity source differs**: K8s service accounts for services, OIDC/SSO for humans
4. **Validation is mandatory**: Use `go-service-authn` library
5. **ISA JWT ≠ User JWT**: Different purposes, different issuers, different contents
6. **Authorization is separate**: ISA JWT proves identity, not permissions
7. **Default issuer**: Sycamore (globally federated OIDC)
8. **Human detection**: Email ends with `@datadoghq.com`
