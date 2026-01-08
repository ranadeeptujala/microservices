# Security Patterns

Patterns for securing microservices communication and access control.

---

## Access Token (JWT)

**Stateless authentication via signed tokens passed between services.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    JWT AUTHENTICATION FLOW                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Client authenticates                                       │
│      ┌────────┐                    ┌────────────┐               │
│      │ Client │───── Login ───────►│ Auth Service│              │
│      │        │◄─── JWT Token ─────│            │               │
│      └────────┘                    └────────────┘               │
│                                                                  │
│   2. Client uses JWT for subsequent requests                    │
│      ┌────────┐                    ┌────────────┐               │
│      │ Client │── JWT in Header ──►│ API Gateway│               │
│      │        │                    │            │               │
│      └────────┘                    └──────┬─────┘               │
│                                           │                     │
│   3. Services validate JWT internally                           │
│                    ┌──────────────────────┴──────────┐          │
│                    ▼                                 ▼          │
│             ┌──────────┐                      ┌──────────┐      │
│             │ Service A│                      │ Service B│      │
│             │ (Validate│                      │ (Validate│      │
│             │  JWT)    │                      │  JWT)    │      │
│             └──────────┘                      └──────────┘      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### JWT Structure:

```
Header.Payload.Signature

{
  "header": {
    "alg": "RS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user-123",
    "name": "John Doe",
    "roles": ["admin", "user"],
    "iat": 1516239022,
    "exp": 1516242622
  },
  "signature": "..."
}
```

### JWT Validation Options:

| Where | Method | Pros | Cons |
|-------|--------|------|------|
| **API Gateway** | Centralized | Single point of validation | Gateway bottleneck |
| **Sidecar** | Distributed | Scales with pods | Need mesh/sidecar |
| **Service** | In-app | Full control | Code in every service |

---

## Service Mesh Security (mTLS)

**Mutual TLS for service-to-service communication.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    mTLS (Mutual TLS)                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Both sides verify certificates:                               │
│                                                                  │
│   ┌─────────────────┐                    ┌─────────────────┐    │
│   │   Service A     │                    │   Service B     │    │
│   │  ┌───────────┐  │                    │  ┌───────────┐  │    │
│   │  │   App     │  │                    │  │   App     │  │    │
│   │  └─────┬─────┘  │                    │  └─────┬─────┘  │    │
│   │  ┌─────▼─────┐  │    Encrypted       │  ┌─────▼─────┐  │    │
│   │  │  Sidecar  │◄─┼─── Connection ─────┼─►│  Sidecar  │  │    │
│   │  │ + Cert A  │  │  (mTLS verified)   │  │ + Cert B  │  │    │
│   │  └───────────┘  │                    │  └───────────┘  │    │
│   └─────────────────┘                    └─────────────────┘    │
│                                                                  │
│   Certificate Management (automatic):                           │
│   • Certificates issued by Service Mesh CA                      │
│   • Automatic rotation                                          │
│   • No manual certificate management                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Istio Security Policy:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # All traffic must be mTLS

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/production/sa/payment-service"]
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/v1/orders/*"]
```

---

## API Key Authentication

**Simple authentication for third-party integrations.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    API KEY FLOW                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Request:                                                      │
│   GET /api/products                                             │
│   X-API-Key: sk_live_abc123xyz                                  │
│                                                                  │
│   Use Cases:                                                    │
│   • Third-party integrations                                    │
│   • Server-to-server communication                              │
│   • Rate limiting by client                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Rate Limiting

**Protects against abuse, DDoS attacks, and ensures fair resource usage.**

> ⚠️ Rate Limiting is both **Security** AND **Reliability** pattern!

```
┌─────────────────────────────────────────────────────────────────┐
│              RATE LIMITING - DUAL PURPOSE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SECURITY (protect against attacks):                            │
│  • DDoS attacks                                                  │
│  • Brute force login attempts                                    │
│  • API scraping / abuse                                          │
│  • Account enumeration                                           │
│                                                                  │
│  RELIABILITY (protect system stability):                        │
│  • Prevent overload from legitimate traffic spikes              │
│  • Fair usage across clients                                     │
│  • Protect downstream services                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Rate Limiting Algorithms:

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMMON ALGORITHMS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. TOKEN BUCKET (most common)                                  │
│     ┌─────────────┐                                              │
│     │ Bucket: 10  │  ← Refills at constant rate                 │
│     │ tokens      │  ← Each request takes 1 token               │
│     │ ●●●●●●●●●●  │  ← Allows bursts up to bucket size          │
│     └─────────────┘                                              │
│                                                                  │
│  2. SLIDING WINDOW (smoother)                                   │
│     │─────────────│ 60 second window                            │
│     │ ●  ● ●●  ●  │ Counts requests in rolling window           │
│     │─────────────│ Limit: 100 req/min                          │
│                                                                  │
│  3. FIXED WINDOW (simple)                                       │
│     │ Window 1   │ Window 2   │                                 │
│     │ 100 reqs   │ reset to 0 │ ← Edge case: burst at boundary  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation Options:

| Where | Tools | Use Case |
|-------|-------|----------|
| **API Gateway** | Kong, AWS API Gateway | Per-client limits |
| **Service Mesh** | Istio, Envoy | Per-service limits |
| **Application** | Resilience4j, Bucket4j | Custom logic |
| **Redis** | Redis + Lua script | Distributed rate limiting |

### Resilience4j Rate Limiter:

```java
@RateLimiter(name = "loginService", fallbackMethod = "loginFallback")
public LoginResponse login(LoginRequest request) {
    return authService.authenticate(request);
}

public LoginResponse loginFallback(LoginRequest request, RequestNotPermitted ex) {
    throw new TooManyRequestsException("Too many login attempts. Try again later.");
}
```

```yaml
resilience4j:
  ratelimiter:
    instances:
      loginService:
        limitForPeriod: 5           # 5 requests...
        limitRefreshPeriod: 60s     # ...per 60 seconds
        timeoutDuration: 0s         # Fail immediately if limit reached
```

### Redis Distributed Rate Limiter (Lua Script):

```lua
-- Token Bucket in Redis
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local bucket = redis.call('HMGET', key, 'tokens', 'last_update')
local tokens = tonumber(bucket[1]) or capacity
local last_update = tonumber(bucket[2]) or now

-- Refill tokens based on time passed
local elapsed = now - last_update
local new_tokens = math.min(capacity, tokens + (elapsed * refill_rate))

if new_tokens >= 1 then
    -- Allow request, consume 1 token
    redis.call('HMSET', key, 'tokens', new_tokens - 1, 'last_update', now)
    redis.call('EXPIRE', key, 3600)
    return 1  -- Allowed
else
    return 0  -- Rate limited
end
```

### Different Limits for Different Users:

```
┌─────────────────────────────────────────────────────────────────┐
│                    TIERED RATE LIMITS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┬────────────┬──────────────┬────────────────┐   │
│  │ Tier        │ Rate Limit │ Burst        │ Use Case       │   │
│  ├─────────────┼────────────┼──────────────┼────────────────┤   │
│  │ Anonymous   │ 10/min     │ 5            │ Unauthenticated│   │
│  │ Free        │ 100/min    │ 20           │ Free tier      │   │
│  │ Pro         │ 1000/min   │ 100          │ Paid users     │   │
│  │ Enterprise  │ 10000/min  │ 1000         │ Premium        │   │
│  └─────────────┴────────────┴──────────────┴────────────────┘   │
│                                                                  │
│  Key strategies:                                                │
│  • By IP address (anonymous)                                    │
│  • By API Key (third party)                                     │
│  • By User ID (authenticated)                                   │
│  • By Tenant ID (multi-tenant)                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Response Headers (Industry Standard):

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100           # Max requests allowed
X-RateLimit-Remaining: 45        # Requests left in window
X-RateLimit-Reset: 1640000000    # Unix timestamp when limit resets

# When rate limited:
HTTP/1.1 429 Too Many Requests
Retry-After: 30                  # Seconds until client can retry
```

---

## Security Best Practices

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECURITY CHECKLIST                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TRANSPORT:                                                     │
│  ✅ Use HTTPS/TLS for all external traffic                      │
│  ✅ Use mTLS for service-to-service communication               │
│  ✅ Terminate TLS at load balancer or service mesh              │
│                                                                  │
│  AUTHENTICATION:                                                │
│  ✅ Use JWT with short expiration                               │
│  ✅ Validate tokens at edge (gateway/sidecar)                   │
│  ✅ Use strong signing algorithms (RS256, ES256)                │
│                                                                  │
│  AUTHORIZATION:                                                 │
│  ✅ Apply principle of least privilege                          │
│  ✅ Use RBAC (Role-Based Access Control)                        │
│  ✅ Validate permissions at service level                       │
│                                                                  │
│  SECRETS:                                                       │
│  ✅ Use secret management (Vault, AWS Secrets Manager)          │
│  ✅ Never store secrets in code or config files                 │
│  ✅ Rotate secrets regularly                                    │
│                                                                  │
│  NETWORK:                                                       │
│  ✅ Use network policies (K8s NetworkPolicy)                    │
│  ✅ Segment networks (public, private, database)                │
│  ✅ Allow only required traffic between services                │
│                                                                  │
│  RATE LIMITING:                                                 │
│  ✅ Implement rate limiting at API Gateway                      │
│  ✅ Different limits per user tier                              │
│  ✅ Stricter limits on auth endpoints (login, password reset)   │
│  ✅ Return proper 429 status with Retry-After header            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

[← Back to Index](./README.md) | [Next: Deployment Patterns →](./08-deployment-patterns.md)

