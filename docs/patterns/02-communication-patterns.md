# Communication Patterns

Communication between microservices is one of the most critical architectural decisions. The choice impacts performance, reliability, coupling, and overall system complexity.

---

## Overview: Synchronous vs Asynchronous

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMMUNICATION STYLES COMPARISON                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   SYNCHRONOUS (Request/Response)         ASYNCHRONOUS (Event-Driven)        │
│   ─────────────────────────────         ─────────────────────────────       │
│                                                                             │
│   ┌─────────┐    Request    ┌─────────┐  ┌─────────┐  Event  ┌──────────┐  │
│   │Service A│──────────────►│Service B│  │Service A│────────►│  Message │  │
│   │         │◄──────────────│         │  │         │         │  Broker  │  │
│   └─────────┘    Response   └─────────┘  └─────────┘         └────┬─────┘  │
│                                                                   │        │
│   • Caller waits for response            ┌─────────┐◄─────────────┘        │
│   • Tight temporal coupling              │Service B│  (subscribes)         │
│   • Simpler to understand                └─────────┘                       │
│   • Harder to scale                      • Fire and forget                 │
│                                          • Loose coupling                  │
│                                          • Better scalability              │
│                                          • More complex                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Aspect | Synchronous | Asynchronous |
|--------|-------------|--------------|
| **Coupling** | Tight (temporal) | Loose |
| **Latency** | Depends on slowest service | Non-blocking |
| **Complexity** | Lower | Higher |
| **Debugging** | Easier | Harder |
| **Scalability** | Limited | Better |
| **Use Case** | Queries, real-time needs | Commands, events, batch |

---

## API Gateway Pattern

**Definition**: A single entry point for all client requests that routes to appropriate backend services.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           API GATEWAY ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         ┌───────────────────┐                               │
│      Web App ──────────►│                   │                               │
│                         │                   │                               │
│      Mobile App ───────►│   API GATEWAY     │                               │
│                         │                   │                               │
│      3rd Party ────────►│   (Kong, NGINX,   │                               │
│                         │    AWS API GW,    │                               │
│      IoT Device ───────►│    Zuul, Traefik) │                               │
│                         │                   │                               │
│                         └─────────┬─────────┘                               │
│                                   │                                         │
│              ┌────────────────────┼────────────────────┐                    │
│              │                    │                    │                    │
│              ▼                    ▼                    ▼                    │
│       ┌───────────┐        ┌───────────┐        ┌───────────┐              │
│       │   User    │        │  Product  │        │   Order   │              │
│       │  Service  │        │  Service  │        │  Service  │              │
│       └───────────┘        └───────────┘        └───────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Responsibilities

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        API GATEWAY RESPONSIBILITIES                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐          │
│  │    ROUTING       │  │  AUTHENTICATION  │  │  RATE LIMITING   │          │
│  │                  │  │                  │  │                  │          │
│  │  /users → user   │  │  JWT validation  │  │  100 req/min     │          │
│  │  /orders → order │  │  OAuth2/OIDC     │  │  per client      │          │
│  │  /products → prod│  │  API keys        │  │  Throttling      │          │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘          │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐          │
│  │  LOAD BALANCING  │  │   CACHING        │  │  TRANSFORMATION  │          │
│  │                  │  │                  │  │                  │          │
│  │  Round robin     │  │  Response cache  │  │  Protocol trans  │          │
│  │  Least conn      │  │  CDN integration │  │  (REST↔gRPC)     │          │
│  │  Weighted        │  │  Cache headers   │  │  Payload mapping │          │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Popular API Gateway Solutions

| Gateway | Type | Best For |
|---------|------|----------|
| **Kong** | Open Source | Kubernetes, plugins ecosystem |
| **AWS API Gateway** | Managed | AWS-native, serverless |
| **Azure API Management** | Managed | Azure ecosystem |
| **NGINX** | Open Source | High performance, simple setup |
| **Traefik** | Open Source | Docker/Kubernetes native |
| **Spring Cloud Gateway** | Framework | Java/Spring ecosystem |
| **Envoy** | Open Source | Service mesh, high performance |

### Pros & Cons

| Pros | Cons |
|------|------|
| ✅ Single entry point | ❌ Single point of failure |
| ✅ Cross-cutting concerns in one place | ❌ Additional latency |
| ✅ Simplified client code | ❌ Can become bottleneck |
| ✅ Protocol translation | ❌ Requires maintenance |

---

## High RPS: ALB + Sidecar vs API Gateway

**⚠️ Important**: At high RPS (1000+ req/sec), API Gateway can become a bottleneck. Consider using ALB directly with sidecar containers for JWT validation.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               API GATEWAY vs ALB + SIDECAR ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   TRADITIONAL API GATEWAY (Lower RPS)        ALB + SIDECAR (High RPS)       │
│   ────────────────────────────────────       ───────────────────────        │
│                                                                             │
│   ┌──────────────┐                          ┌──────────────┐                │
│   │ API GATEWAY  │ ← Bottleneck!            │     ALB      │ ← Scales      │
│   │              │                          │ (AWS/Cloud)  │   automatically│
│   │ • Routing    │                          │              │                │
│   │ • JWT Valid  │                          │ • Routing    │                │
│   │ • Rate Limit │                          │ • SSL Term   │                │
│   └──────┬───────┘                          └──────┬───────┘                │
│          │                                         │                        │
│          ▼                                         ▼                        │
│   ┌──────────────┐                     ┌─────────────────────────┐          │
│   │   Service    │                     │   Service Pod/Container │          │
│   └──────────────┘                     │  ┌───────────────────┐  │          │
│                                        │  │   SIDECAR PROXY   │  │          │
│                                        │  │   (Envoy/NGINX)   │  │          │
│                                        │  │  • JWT Validation │  │          │
│                                        │  │  • Rate Limiting  │  │          │
│                                        │  │  • mTLS           │  │          │
│                                        │  └─────────┬─────────┘  │          │
│                                        │  ┌─────────▼─────────┐  │          │
│                                        │  │    APP CONTAINER  │  │          │
│                                        │  └───────────────────┘  │          │
│                                        └─────────────────────────┘          │
│                                                                             │
│   Throughput: ~5K-10K RPS              Throughput: ~100K+ RPS               │
│   Latency: +5-20ms                     Latency: +1-2ms                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### RPS Threshold Decision

| RPS Level | Recommended Architecture |
|-----------|-------------------------|
| < 500 RPS | API Gateway (simple) |
| 500 - 5K RPS | API Gateway (scaled) or ALB + Sidecar |
| 5K - 50K RPS | ALB + Sidecar (Service Mesh) |
| > 50K RPS | ALB + Sidecar + CDN Edge |

---

## Backend for Frontend (BFF) Pattern

**Definition**: Create separate API gateways tailored for each client type (web, mobile, IoT).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BFF ARCHITECTURE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐       │
│   │   Web    │      │  Mobile  │      │  Admin   │      │   IoT    │       │
│   │  Client  │      │   App    │      │  Portal  │      │ Devices  │       │
│   └────┬─────┘      └────┬─────┘      └────┬─────┘      └────┬─────┘       │
│        │                 │                 │                 │              │
│        ▼                 ▼                 ▼                 ▼              │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐       │
│   │ Web BFF  │      │Mobile BFF│      │Admin BFF │      │ IoT BFF  │       │
│   │          │      │          │      │          │      │          │       │
│   │• Full    │      │• Compact │      │• All data│      │• Minimal │       │
│   │  payload │      │  payload │      │• Reports │      │  payload │       │
│   │• GraphQL │      │• REST    │      │• Batch   │      │• Binary  │       │
│   └────┬─────┘      └────┬─────┘      └────┬─────┘      └────┬─────┘       │
│        │                 │                 │                 │              │
│        └─────────────────┴────────┬────────┴─────────────────┘              │
│                                   │                                         │
│                                   ▼                                         │
│                    ┌──────────────────────────────┐                         │
│                    │      BACKEND SERVICES        │                         │
│                    └──────────────────────────────┘                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### When to Use BFF

| Use BFF | Avoid BFF |
|---------|-----------|
| Multiple client types with different needs | Single client type |
| Different teams per client | Small team |
| Varying network conditions (mobile vs web) | Similar requirements |
| Need protocol translation | Simple pass-through |

---

## Synchronous Communication

### REST (Representational State Transfer)

**The most common communication pattern for microservices.**

```
REST API DESIGN:

GET    /users              → List all users
GET    /users/{id}         → Get specific user
POST   /users              → Create new user
PUT    /users/{id}         → Update entire user
PATCH  /users/{id}         → Partial update
DELETE /users/{id}         → Delete user

NESTED RESOURCES:
GET    /users/{id}/orders  → User's orders
POST   /users/{id}/orders  → Create order for user

QUERY PARAMETERS:
GET /products?category=electronics&sort=price&page=2&limit=20
```

### gRPC (Google Remote Procedure Call)

**High-performance RPC framework using Protocol Buffers.**

| Feature | REST (JSON) | gRPC (Protobuf) |
|---------|-------------|-----------------|
| Format | Text (JSON) | Binary |
| Protocol | HTTP/1.1 | HTTP/2 |
| Performance | ~1000 req/s | ~5000 req/s |
| Streaming | Limited | Bidirectional |
| Type Safety | Weak | Strong |

### When to Use REST vs gRPC

| Use REST | Use gRPC |
|----------|----------|
| Public APIs | Internal service-to-service |
| Browser clients | High-performance needs |
| Simple CRUD operations | Streaming required |
| Human-readable debugging | Polyglot microservices |

---

## Asynchronous Communication

### Message Queue Pattern

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      MESSAGE QUEUE PATTERNS                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   POINT-TO-POINT (Queue)                                                   │
│   ┌──────────┐                                        ┌──────────┐         │
│   │ Producer │──────►│  ORDER_QUEUE   │──────►       │ Consumer │         │
│   └──────────┘       │  [msg][msg][msg]│              └──────────┘         │
│                                                                             │
│   • One message consumed by ONE consumer                                    │
│   • Load balancing across consumers                                         │
│                                                                             │
│   PUBLISH-SUBSCRIBE (Topic)                                                │
│   ┌──────────┐       ┌────────────────┐     ┌──────────┐                   │
│   │ Publisher│──────►│  ORDER_EVENTS  │────►│Subscriber│ (Email Svc)       │
│   └──────────┘       │                │────►│Subscriber│ (Analytics)        │
│                      │                │────►│Subscriber│ (Inventory)        │
│                      └────────────────┘     └──────────┘                   │
│                                                                             │
│   • One message consumed by ALL subscribers                                 │
│   • Broadcast pattern                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Message Broker Comparison

| Feature | RabbitMQ | Apache Kafka | AWS SQS |
|---------|----------|--------------|---------|
| **Pattern** | Queue + Pub/Sub | Log-based | Queue |
| **Throughput** | ~50K msg/s | ~1M msg/s | ~3K msg/s |
| **Retention** | Until consumed | Configurable | 14 days max |
| **Use Case** | Task queues | Event streaming | Cloud-native |

---

## Service Mesh

**Infrastructure layer for service-to-service communication.**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SERVICE MESH ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                           CONTROL PLANE                                     │
│                    ┌─────────────────────────┐                              │
│                    │  (Istio/Linkerd/Consul) │                              │
│                    │  • Configuration        │                              │
│                    │  • Policy Engine        │                              │
│                    └───────────┬─────────────┘                              │
│                                │                                            │
│              ┌─────────────────┼─────────────────┐                          │
│              ▼                 ▼                 ▼                          │
│   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐              │
│   │   Service A     │ │   Service B     │ │   Service C     │              │
│   │  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │              │
│   │  │   App     │  │ │  │   App     │  │ │  │   App     │  │              │
│   │  └─────┬─────┘  │ │  └─────┬─────┘  │ │  └─────┬─────┘  │              │
│   │  ┌─────▼─────┐  │ │  ┌─────▼─────┐  │ │  ┌─────▼─────┐  │              │
│   │  │  Sidecar  │◄─┼─┼─►│  Sidecar  │◄─┼─┼─►│  Sidecar  │  │              │
│   │  │  (Envoy)  │  │ │  │  (Envoy)  │  │ │  │  (Envoy)  │  │              │
│   │  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │              │
│   └─────────────────┘ └─────────────────┘ └─────────────────┘              │
│                                                                             │
│                           DATA PLANE                                        │
│                    (All traffic flows through proxies)                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Service Mesh Capabilities

| Category | Capabilities |
|----------|--------------|
| **Traffic** | Load balancing, canary, circuit breaking |
| **Security** | mTLS, JWT validation, authorization |
| **Observability** | Distributed tracing, metrics, logging |

---

## Communication Anti-Patterns

```
❌ CHATTY SERVICES
   BAD:  10+ API calls to get order details
   GOOD: Use aggregation endpoints or BFF

❌ SYNC CHAINS (Distributed Monolith)
   BAD:  A → B → C → D → E (all synchronous)
   GOOD: Use async where possible, limit sync depth

❌ SHARED DATABASE COMMUNICATION
   BAD:  Service A writes, Service B polls DB
   GOOD: Use events/messages for communication

❌ MISSING TIMEOUTS & RETRIES
   BAD:  client.get("/api") // No timeout!
   GOOD: client.get("/api", { timeout: 5000, retries: 3 })
```

---

## Communication Pattern Decision Matrix

| Scenario | Recommended Pattern |
|----------|---------------------|
| Public APIs, Browser clients | REST + API Gateway |
| High-performance internal calls | gRPC |
| Mobile apps with varied needs | BFF + REST |
| Fire-and-forget operations | Message Queue (Async) |
| Event notifications to multiple services | Pub/Sub (Kafka) |
| Complex traffic management needs | Service Mesh |

---

[← Back to Index](./README.md) | [Next: Data Management Patterns →](./03-data-management-patterns.md)

