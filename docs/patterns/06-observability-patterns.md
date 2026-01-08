# Observability Patterns

Patterns for monitoring, debugging, and understanding distributed systems.

---

## Distributed Tracing

**Track requests as they flow across service boundaries.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    DISTRIBUTED TRACE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Trace ID: abc-123                                             │
│                                                                  │
│   ├─ Span 1: API Gateway (50ms)                                 │
│   │   └─ Span 2: Order Service (120ms)                          │
│   │       ├─ Span 3: User Service (30ms)                        │
│   │       ├─ Span 4: Inventory Service (40ms)                   │
│   │       └─ Span 5: Payment Service (80ms)                     │
│   │           └─ Span 6: External Payment API (60ms)            │
│                                                                  │
│   Total Time: 200ms                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Concepts:

| Concept | Description |
|---------|-------------|
| **Trace** | End-to-end journey of a request |
| **Span** | Single operation within a trace |
| **Trace ID** | Unique identifier for entire request |
| **Span ID** | Unique identifier for each operation |
| **Parent Span** | Links child operations to parent |

### Tools:
- **Jaeger** - Open-source, CNCF graduated
- **Zipkin** - Twitter's distributed tracing
- **OpenTelemetry** - Unified observability standard

### Implementation (OpenTelemetry):

```java
@RestController
public class OrderController {
    
    private final Tracer tracer;
    
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable String id) {
        Span span = tracer.spanBuilder("getOrder")
            .setAttribute("order.id", id)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // Trace propagated to downstream calls
            return orderService.getOrder(id);
        } finally {
            span.end();
        }
    }
}
```

---

## Log Aggregation

**Centralize logs from all services for unified analysis.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOG AGGREGATION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐                        │
│   │Service A│  │Service B│  │Service C│                        │
│   └────┬────┘  └────┬────┘  └────┬────┘                        │
│        │            │            │                              │
│        └────────────┼────────────┘                              │
│                     │                                           │
│              ┌──────▼──────┐                                    │
│              │ Log Shipper │  (Fluentd, Filebeat, Vector)       │
│              └──────┬──────┘                                    │
│                     │                                           │
│              ┌──────▼──────┐                                    │
│              │  Log Store  │  (Elasticsearch, Loki)             │
│              └──────┬──────┘                                    │
│                     │                                           │
│              ┌──────▼──────┐                                    │
│              │ Visualization│  (Kibana, Grafana)                │
│              └─────────────┘                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Tools:
- **ELK Stack** - Elasticsearch, Logstash, Kibana
- **Loki + Grafana** - Lightweight, Prometheus-like for logs
- **Splunk** - Enterprise log management

### Structured Logging (JSON):

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "INFO",
  "service": "order-service",
  "traceId": "abc-123",
  "spanId": "span-456",
  "userId": "user-789",
  "message": "Order created successfully",
  "orderId": "order-001",
  "amount": 99.99
}
```

---

## Health Check API

**Each service exposes endpoints for liveness and readiness checks.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    HEALTH CHECKS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   LIVENESS: Is the service running?                             │
│   GET /health/live                                              │
│   └─ 200 OK → Service is alive                                  │
│   └─ 503 → Service is dead (restart it!)                        │
│                                                                  │
│   READINESS: Can it handle traffic?                             │
│   GET /health/ready                                             │
│   └─ 200 OK → Ready for traffic                                 │
│   └─ 503 → Not ready (DB down, cache warming, etc.)             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Kubernetes Probes:

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Spring Boot Actuator:

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, info
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
```

---

## Metrics Collection

**Collect and visualize service metrics.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    KEY METRICS (RED Method)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   R - Rate:     Requests per second                             │
│   E - Errors:   Error rate (5xx, timeouts)                      │
│   D - Duration: Request latency (P50, P95, P99)                 │
│                                                                  │
│   Additional:                                                   │
│   • Saturation: Resource utilization (CPU, memory, threads)     │
│   • Dependencies: Health of downstream services                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Tools:
- **Prometheus** - Metrics collection
- **Grafana** - Visualization
- **Datadog/New Relic** - Full observability platforms

---

[← Back to Index](./README.md) | [Next: Security Patterns →](./07-security-patterns.md)

