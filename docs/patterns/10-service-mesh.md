# Service Mesh Pattern

Infrastructure layer for service-to-service communication. Control traffic, security, and observability **WITHOUT changing application code**.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SERVICE MESH ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                          CONTROL PLANE                                      │
│                    ┌─────────────────────────┐                              │
│                    │   Istio / Linkerd       │  ← Configuration (YAML)      │
│                    │   • Policy Rules        │    NO code changes!          │
│                    │   • Certificate Mgmt    │                              │
│                    └───────────┬─────────────┘                              │
│                                │                                            │
│              ┌─────────────────┼─────────────────┐                          │
│              ▼                 ▼                 ▼                          │
│   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐              │
│   │   SERVICE A     │ │   SERVICE B     │ │   SERVICE C     │              │
│   │  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │              │
│   │  │   App     │  │ │  │   App     │  │ │  │   App     │  │              │
│   │  └─────┬─────┘  │ │  └─────┬─────┘  │ │  └─────┬─────┘  │              │
│   │  ┌─────▼─────┐  │ │  ┌─────▼─────┐  │ │  ┌─────▼─────┐  │              │
│   │  │  SIDECAR  │◄─┼─┼─►│  SIDECAR  │◄─┼─┼─►│  SIDECAR  │  │              │
│   │  │  (Envoy)  │  │ │  │  (Envoy)  │  │ │  │  (Envoy)  │  │              │
│   │  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │              │
│   └─────────────────┘ └─────────────────┘ └─────────────────┘              │
│                                                                             │
│                          DATA PLANE                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## What Sidecar Handles (WITHOUT Code Changes)

| Category | Capabilities |
|----------|--------------|
| **Traffic** | Load balancing, rate limiting, traffic routing, canary deployments |
| **Security** | mTLS, JWT validation, authorization policies |
| **Resilience** | Circuit breaker, retries, timeouts |
| **Observability** | Distributed tracing, metrics, logging (Fluentbit) |

**All configured via YAML - NO CODE CHANGES!**

---

## Key Components

### Control Plane
- Manages configuration
- Distributes policies to sidecars
- Handles certificate management
- Examples: Istio (istiod), Linkerd (control plane)

### Data Plane
- Sidecar proxies (Envoy)
- Intercepts all network traffic
- Applies policies at runtime
- Reports telemetry data

---

## Popular Service Mesh Solutions

| Solution | Platform | Key Features |
|----------|----------|--------------|
| **Istio** | Kubernetes | Most feature-rich, complex |
| **Linkerd** | Kubernetes | Lightweight, simple |
| **Consul Connect** | Any | HashiCorp ecosystem |
| **AWS App Mesh** | AWS | AWS-native |
| **Cilium** | Kubernetes | eBPF-based, high performance |

---

## Istio Configuration Examples

### Traffic Splitting (Canary Deployment)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service
spec:
  hosts:
    - user-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: user-service
            subset: v2
    - route:
        - destination:
            host: user-service
            subset: v1
          weight: 90
        - destination:
            host: user-service
            subset: v2
          weight: 10
```

### Circuit Breaker

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: user-service
spec:
  host: user-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

### mTLS Policy

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # All traffic must be mTLS
```

### Authorization Policy

```yaml
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

## When to Use Service Mesh

| Use Service Mesh | Avoid Service Mesh |
|------------------|-------------------|
| Many microservices (10+) | Few services |
| Need mTLS without code changes | Simple security needs |
| Complex traffic management | Basic load balancing sufficient |
| Zero-trust security model | Trusted network |
| Need advanced observability | Basic monitoring sufficient |

---

## Service Mesh vs API Gateway

| Feature | Service Mesh | API Gateway |
|---------|-------------|-------------|
| **Scope** | Internal (East-West) | External (North-South) |
| **Location** | Sidecar per service | Edge of network |
| **Traffic** | Service-to-service | Client-to-service |
| **Security** | mTLS, authz policies | JWT, rate limiting |
| **Use Together** | ✅ Yes | ✅ Yes |

---

[← Back to Index](./README.md) | [Next: IoT vs Message Queues →](./11-iot-vs-message-queues.md)

