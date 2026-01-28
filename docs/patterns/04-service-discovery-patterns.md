# Service Discovery Patterns

How services find and communicate with each other in a distributed system.

---

## Evolution of Service Discovery

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Service Discovery Evolution                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  2014-2018: Netflix OSS Era          2018-Present: Cloud Native Era         │
│  ┌─────────────────────────┐         ┌─────────────────────────┐            │
│  │  CLIENT-SIDE DISCOVERY  │         │  SERVER-SIDE DISCOVERY  │            │
│  │                         │         │                         │            │
│  │  • Eureka               │   ──►   │  • Kubernetes Services  │            │
│  │  • Consul               │         │  • API Gateway          │            │
│  │  • Zookeeper            │         │  • ALB/NLB              │            │
│  │  • Ribbon (Load Bal)    │         │  • Service Mesh         │            │
│  └─────────────────────────┘         └─────────────────────────┘            │
│                                                                              │
│  WHY THE SHIFT?                                                              │
│  ─────────────────────────────────────────────────────────────              │
│  • Kubernetes has built-in service discovery (kube-dns, CoreDNS)            │
│  • Cloud providers offer managed load balancers (ALB, NLB, Cloud LB)        │
│  • Less complexity in application code                                       │
│  • Infrastructure handles discovery, not application                         │
│  • Service mesh (Istio, Linkerd) provides advanced traffic management       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Client-Side Discovery (Legacy - Rarely Used Today)

```
┌──────────────────────────────────────────────────────────────────┐
│                   Client-Side Discovery (Legacy)                  │
│                                                                   │
│    ┌────────────┐      ┌──────────────────┐                      │
│    │   Client   │──1──►│ Service Registry │                      │
│    │  Service   │◄──2──│   (Eureka)       │                      │
│    └─────┬──────┘      │                  │                      │
│          │             │ service-a: [     │                      │
│          │             │   10.0.1.5:8080  │                      │
│          │             │   10.0.1.6:8080  │                      │
│          │             │   10.0.1.7:8080  │                      │
│          │             │ ]                │                      │
│          │             └──────────────────┘                      │
│          │                                                        │
│          │ 3. Client picks one (Ribbon load balancing)           │
│          ▼                                                        │
│    ┌──────────────────────────────────────┐                      │
│    │     Target Service Instances          │                      │
│    │  ┌────────┐ ┌────────┐ ┌────────┐    │                      │
│    │  │10.0.1.5│ │10.0.1.6│ │10.0.1.7│    │                      │
│    │  └────────┘ └────────┘ └────────┘    │                      │
│    └──────────────────────────────────────┘                      │
└──────────────────────────────────────────────────────────────────┘

Flow:
1. Client queries registry for service instances
2. Registry returns list of healthy instances
3. Client-side load balancer (Ribbon) picks one
4. Client calls selected instance directly
```

**Problems with Client-Side Discovery:**
- ❌ Every service needs registry client library
- ❌ Library must exist for every language (polyglot challenge)
- ❌ Application code coupled to infrastructure
- ❌ Complex client-side load balancing logic
- ❌ Registry becomes single point of failure
- ❌ Network calls to registry add latency

---

## Server-Side Discovery (Modern Standard ✅)

**This is what everyone uses today: API Gateway, ALB, Kubernetes Services**

```
┌──────────────────────────────────────────────────────────────────┐
│                Server-Side Discovery (Modern)                     │
│                                                                   │
│    ┌────────────┐                                                │
│    │   Client   │                                                │
│    │  Service   │                                                │
│    └─────┬──────┘                                                │
│          │                                                        │
│          │ 1. Call: https://api.example.com/payments             │
│          │    or: http://payment-service (K8s internal)          │
│          ▼                                                        │
│    ┌──────────────────────────────────────┐                      │
│    │   Load Balancer / API Gateway         │◄── Discovery here!  │
│    │   (ALB / Kong / K8s Service)          │                     │
│    └─────┬────────────────────────────────┘                      │
│          │                                                        │
│          │ 2. LB handles discovery & routing                     │
│          ▼                                                        │
│    ┌──────────────────────────────────────┐                      │
│    │     Target Service Instances          │                      │
│    │  ┌────────┐ ┌────────┐ ┌────────┐    │                      │
│    │  │10.0.1.5│ │10.0.1.6│ │10.0.1.7│    │                      │
│    │  └────────┘ └────────┘ └────────┘    │                      │
│    └──────────────────────────────────────┘                      │
└──────────────────────────────────────────────────────────────────┘

Client only knows ONE address - the load balancer/gateway!
```

---

## Modern Server-Side Discovery Options

### Option 1: AWS Route53 + ALB + CloudFront

```
┌─────────────────────────────────────────────────────────────────┐
│              AWS ARCHITECTURE: STATIC + DYNAMIC                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                      ┌──────────────┐                           │
│                      │   Route53    │                           │
│                      │    (DNS)     │                           │
│                      └──────┬───────┘                           │
│                             │                                    │
│            ┌────────────────┴────────────────┐                  │
│            │                                 │                  │
│            ▼                                 ▼                  │
│   ┌─────────────────┐               ┌─────────────────┐        │
│   │   CloudFront    │               │      ALB        │        │
│   │   (CDN + S3)    │               │  (Load Balancer)│        │
│   │                 │               │                 │        │
│   │ Static Content: │               │ Dynamic APIs:   │        │
│   │ • React/Vue SPA │               │ • /api/orders   │        │
│   │ • Images, CSS   │               │ • /api/payments │        │
│   │ • JS bundles    │               │ • /api/users    │        │
│   └────────┬────────┘               └────────┬────────┘        │
│            │                                 │                  │
│            ▼                                 ▼                  │
│   ┌─────────────────┐               ┌─────────────────┐        │
│   │       S3        │               │   ECS / EKS     │        │
│   │  (Static Files) │               │  (Microservices)│        │
│   └─────────────────┘               └─────────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Route53 Configuration:**
```
┌─────────────────────────────────────────────────────────────────┐
│              ROUTE53 DNS RECORDS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  example.com        → CloudFront (static SPA)                   │
│  www.example.com    → CloudFront (static SPA)                   │
│  api.example.com    → ALB (microservices)                       │
│                                                                  │
│  Route53 Alias Records:                                         │
│  • example.com      → d1234.cloudfront.net                     │
│  • api.example.com  → alb-1234.us-east-1.elb.amazonaws.com     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**CloudFront for Static Content:**
```
┌─────────────────────────────────────────────────────────────────┐
│              CLOUDFRONT BENEFITS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ✅ Global CDN - Edge locations worldwide (low latency)         │
│  ✅ Caching - Don't hit S3 for every request                   │
│  ✅ HTTPS - Free SSL certificate via ACM                        │
│  ✅ DDoS Protection - AWS Shield Standard included              │
│  ✅ Compression - Gzip/Brotli automatic                         │
│  ✅ Cost - S3 + CloudFront cheaper than serving from ALB        │
│                                                                  │
│  Use for:                                                       │
│  • Single Page Apps (React, Vue, Angular)                       │
│  • Static websites                                              │
│  • Images, videos, downloads                                    │
│  • CSS, JavaScript bundles                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**ALB for Dynamic APIs:**
```
┌─────────────────────────────────────────────────────────────────┐
│              ALB ROUTING                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  api.example.com/orders/*   → Order Service Target Group        │
│  api.example.com/payments/* → Payment Service Target Group      │
│  api.example.com/users/*    → User Service Target Group         │
│                                                                  │
│  ALB handles:                                                   │
│  • SSL termination                                              │
│  • Path-based routing                                           │
│  • Health checks                                                │
│  • Auto-scaling integration                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Global Accelerator for Active-Active Multi-Region:**
```
┌─────────────────────────────────────────────────────────────────┐
│              AWS GLOBAL ACCELERATOR                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                     ┌─────────────────────┐                     │
│                     │  Global Accelerator │                     │
│                     │  Static IPs:        │                     │
│                     │  • 75.2.xxx.xxx     │                     │
│                     │  • 99.83.xxx.xxx    │                     │
│                     └──────────┬──────────┘                     │
│                                │                                 │
│           ┌────────────────────┴────────────────────┐           │
│           │                                         │           │
│           ▼                                         ▼           │
│   ┌───────────────┐                        ┌───────────────┐   │
│   │  US-EAST-1    │        ACTIVE          │  EU-WEST-1    │   │
│   │  (50% traffic)│◄──────ACTIVE──────────►│  (50% traffic)│   │
│   │               │                        │               │   │
│   │  ┌─────────┐  │                        │  ┌─────────┐  │   │
│   │  │   ALB   │  │                        │  │   ALB   │  │   │
│   │  └────┬────┘  │                        │  └────┬────┘  │   │
│   │       │       │                        │       │       │   │
│   │  ┌────▼────┐  │                        │  ┌────▼────┐  │   │
│   │  │   ECS   │  │                        │  │   ECS   │  │   │
│   │  └─────────┘  │                        │  └─────────┘  │   │
│   └───────────────┘                        └───────────────┘   │
│                                                                  │
│  BOTH regions serve traffic simultaneously!                     │
│  If one fails → 100% to healthy region (instant failover)      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Route53 vs Global Accelerator:**
```
┌──────────────────────────────────────────────────────────────────┐
│              COMPARISON: ROUTE53 vs GLOBAL ACCELERATOR           │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌───────────────┬─────────────────┬─────────────────────────┐   │
│  │ Feature       │ Route53         │ Global Accelerator      │   │
│  ├───────────────┼─────────────────┼─────────────────────────┤   │
│  │ Mode          │ Active-Passive  │ Active-Active ✅        │   │
│  │ IPs           │ DNS (changes)   │ Static anycast IPs      │   │
│  │ Failover      │ DNS TTL (~60s)  │ Instant (<30s) ✅       │   │
│  │ Network       │ Public internet │ AWS global network ✅   │   │
│  │ Cost          │ Cheaper         │ More expensive          │   │
│  │ Use case      │ Standard apps   │ Real-time, gaming, API  │   │
│  └───────────────┴─────────────────┴─────────────────────────┘   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**When to use Global Accelerator:**
```
┌─────────────────────────────────────────────────────────────────┐
│              USE GLOBAL ACCELERATOR WHEN:                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ✅ Need active-active multi-region                             │
│  ✅ Real-time applications (gaming, video, trading)             │
│  ✅ Need instant failover (not DNS TTL dependent)               │
│  ✅ Static IPs required (firewall whitelisting)                 │
│  ✅ Global users need low latency                               │
│  ✅ Want to avoid public internet hops                          │
│                                                                  │
│  USE ROUTE53 WHEN:                                              │
│  ✅ Active-passive is sufficient                                │
│  ✅ Cost-sensitive                                              │
│  ✅ Standard web applications                                   │
│  ✅ Single region deployment                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Option 2: Kubernetes Service (Standard for K8s)

```yaml
# Kubernetes handles all discovery via kube-dns/CoreDNS
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
spec:
  selector:
    app: payment
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP  # Internal service discovery

# Other services call: http://payment-service.production.svc.cluster.local
# Or simply: http://payment-service (same namespace)
```

```java
// Application code - NO discovery logic needed!
@RestController
public class OrderController {
    
    @Value("${payment.service.url:http://payment-service}")
    private String paymentServiceUrl;
    
    @GetMapping("/order/{id}")
    public Order getOrder(@PathVariable String id) {
        // K8s DNS resolves "payment-service" to ClusterIP
        // ClusterIP load balances to pods
        Payment payment = restTemplate.getForObject(
            paymentServiceUrl + "/payments/" + id,
            Payment.class
        );
        return new Order(id, payment);
    }
}
```

### Option 3: API Gateway (Kong, AWS API Gateway)

```yaml
# Kong configuration
services:
  - name: payment-service
    url: http://payment-upstream
    routes:
      - paths: ["/api/v1/payments"]

upstreams:
  - name: payment-upstream
    targets:
      - target: payment-service.production.svc:8080
        weight: 100
    healthchecks:
      active:
        http_path: /health
        interval: 5
```

---

## Comparison: Why Server-Side Discovery Won

| Aspect | Client-Side (Eureka) | Server-Side (ALB/K8s) |
|--------|---------------------|----------------------|
| **Code Complexity** | High - registry client in every service | Zero - just use hostname |
| **Language Support** | Limited - need library per language | Universal - DNS works everywhere |
| **Infrastructure** | Self-managed registry cluster | Managed by cloud/K8s |
| **Latency** | Extra hop to registry | Direct call to LB |
| **Coupling** | App coupled to discovery | App decoupled |
| **Health Checks** | Registry polls services | LB does health checks |
| **Load Balancing** | Client-side (Ribbon) | Server-side (proven) |
| **Modern Usage** | Legacy Netflix stack | ✅ Standard approach |

---

## When to Still Use Service Registry?

**Rare cases where Consul/Eureka might be useful:**

1. **Multi-datacenter service mesh** - Consul Connect
2. **Non-Kubernetes environments** - VMs without orchestration
3. **Configuration management** - Consul KV store
4. **Legacy migration** - Existing Spring Cloud Netflix apps

---

## Modern Architecture Decision

```
┌─────────────────────────────────────────────────────────────────┐
│                 Modern Architecture Decision                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Q: What are you using for compute?                            │
│                                                                  │
│   ├── Kubernetes (EKS/GKE/AKS)                                  │
│   │   └── Use: K8s Services + Ingress/ALB                       │
│   │                                                              │
│   ├── AWS ECS                                                   │
│   │   └── Use: ALB + Service Discovery (Cloud Map)              │
│   │                                                              │
│   ├── Serverless (Lambda/Cloud Functions)                       │
│   │   └── Use: API Gateway                                      │
│   │                                                              │
│   └── VMs (EC2/GCE without orchestration)                       │
│       └── Consider: Consul or simple ALB with ASG               │
│                                                                  │
│   BOTTOM LINE: If you're on K8s or cloud, you DON'T need       │
│   Eureka/Consul for service discovery. It's built-in!          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example: Typical Modern Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│            Typical Modern Microservices Architecture             │
│                                                                  │
│   External Traffic                 Internal Traffic              │
│   ─────────────────                ─────────────────             │
│                                                                  │
│   Browser/Mobile                                                 │
│        │                                                         │
│        ▼                                                         │
│   ┌─────────┐      ┌─────────────────────────────────────┐      │
│   │ Route53 │      │         Kubernetes Cluster          │      │
│   │  (DNS)  │      │                                     │      │
│   └────┬────┘      │   ┌─────────┐                       │      │
│        │           │   │ Order   │──► payment-service    │      │
│        ▼           │   │ Service │    (K8s Service)      │      │
│   ┌─────────┐      │   └─────────┘         │             │      │
│   │   ALB   │──────│─────────────────────────────────────│      │
│   │ (L7 LB) │      │   ┌─────────┐         ▼             │      │
│   └─────────┘      │   │ Payment │    ┌─────────┐        │      │
│        │           │   │ Pods    │◄───│ ClusterIP│       │      │
│   Handles:         │   └─────────┘    └─────────┘        │      │
│   • SSL/TLS        │                                     │      │
│   • Path routing   │   Internal calls use K8s DNS:       │      │
│   • Health checks  │   http://payment-service/payments   │      │
│   • WAF            │                                     │      │
│                    └─────────────────────────────────────┘      │
│                                                                  │
│   NO EUREKA NEEDED! ALB + K8s handles everything.               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary

- ✅ **Use ALB** for external traffic (internet-facing)
- ✅ **Use K8s Services** for internal service-to-service calls
- ✅ **Use API Gateway** if you need advanced features (rate limiting, auth, transforms)
- ❌ **Skip Eureka/Ribbon** - they're legacy Netflix stack from pre-Kubernetes era

---

[← Back to Index](./README.md) | [Next: Reliability Patterns →](./05-reliability-patterns.md)

