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

### Option 1: AWS ALB/NLB (Most Common for AWS)

```
┌─────────────────────────────────────────────────────────────────┐
│                     AWS ALB Architecture                         │
│                                                                  │
│   Client ──► Route53 (DNS) ──► ALB ──► Target Group ──► ECS/EKS │
│                                                                  │
│   DNS: api.example.com → ALB DNS                                │
│   ALB: Handles health checks, routing, SSL termination          │
│   Target Group: Auto-discovers healthy instances                │
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

