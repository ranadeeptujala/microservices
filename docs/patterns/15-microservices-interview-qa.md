# Microservices Interview Q&A (29 Questions)

---

## 1. Why does a microservice work fine alone but fail when integrated?

**Common Causes:**
```
┌─────────────────────────────────────────────────────────────────┐
│  WORKS ALONE                    │  FAILS IN INTEGRATION         │
├─────────────────────────────────┼───────────────────────────────┤
│  No network latency             │  Network timeouts             │
│  No dependency failures         │  Downstream service down      │
│  Local DB always available      │  Connection pool exhaustion   │
│  No concurrent load             │  Thread pool saturation       │
│  Mocked dependencies            │  Real services behave differently │
│  No auth/security checks        │  Token validation fails       │
│  Same timezone/locale           │  Serialization mismatches     │
└─────────────────────────────────┴───────────────────────────────┘
```

**Real Examples:**
- **Timeout mismatch**: Service A waits 30s, Service B takes 35s → timeout
- **Missing headers**: Auth token not propagated in service-to-service calls
- **Data format**: LocalDateTime works locally, fails with different timezone
- **Connection limits**: Works with 1 request, fails under load (connection pool exhausted)

---

## 2. How does Spring handle service-to-service communication?

```java
// 1. RestTemplate (Legacy - Blocking)
@Bean
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// 2. WebClient (Modern - Non-blocking)
@Bean
public WebClient webClient() {
    return WebClient.builder()
        .baseUrl("http://order-service")
        .build();
}

// 3. OpenFeign (Declarative - Recommended)
@FeignClient(name = "order-service", fallback = OrderFallback.class)
public interface OrderClient {
    @GetMapping("/orders/{id}")
    Order getOrder(@PathVariable String id);
}

// 4. Spring Cloud LoadBalancer (with service discovery)
@LoadBalanced
@Bean
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}
```

**Comparison:**
| Method | Blocking | Declarative | Load Balanced | Recommended |
|--------|----------|-------------|---------------|-------------|
| RestTemplate | Yes | No | Manual | Legacy |
| WebClient | No | No | Yes | Modern |
| OpenFeign | Yes | Yes | Yes | Best DX |

---

## 3. REST vs Messaging (Kafka/RabbitMQ) — when and why?

```
┌─────────────────────────────────────────────────────────────────┐
│                    REST (Synchronous)                           │
├─────────────────────────────────────────────────────────────────┤
│  ✅ Use When:                                                   │
│     • Need immediate response (query data)                      │
│     • Simple request-response pattern                           │
│     • Real-time user interactions                               │
│     • Low latency required                                      │
│                                                                 │
│  ❌ Problems:                                                   │
│     • Tight coupling                                            │
│     • Cascading failures                                        │
│     • Blocks thread while waiting                               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                 MESSAGING (Asynchronous)                        │
├─────────────────────────────────────────────────────────────────┤
│  ✅ Use When:                                                   │
│     • Fire-and-forget (send email, notification)                │
│     • Event-driven architecture                                 │
│     • Decouple services                                         │
│     • Handle traffic spikes (buffer)                            │
│     • Long-running processes                                    │
│                                                                 │
│  Kafka vs RabbitMQ:                                             │
│     • Kafka: High throughput, replay, event sourcing            │
│     • RabbitMQ: Complex routing, lower latency, traditional MQ  │
└─────────────────────────────────────────────────────────────────┘
```

**Decision Matrix:**
```
Need response NOW?        → REST
Can wait for processing?  → Messaging
Multiple consumers?       → Kafka
Complex routing?          → RabbitMQ
Event replay needed?      → Kafka
```

---

## 4. What problems does Service Discovery actually solve?

```
┌─────────────────────────────────────────────────────────────────┐
│  WITHOUT Service Discovery:                                     │
│                                                                 │
│  order-service → http://192.168.1.45:8080/payments  ❌         │
│                                                                 │
│  Problems:                                                      │
│  • IP addresses change when pods restart                        │
│  • Can't scale dynamically                                      │
│  • Manual configuration nightmare                               │
│  • No load balancing                                            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  WITH Service Discovery (Modern: ALB/K8s Service):              │
│                                                                 │
│  order-service → http://payment-service/payments  ✅            │
│                                                                 │
│  Solves:                                                        │
│  • Dynamic IP resolution                                        │
│  • Auto load balancing                                          │
│  • Health-aware routing                                         │
│  • Zero config for new instances                                │
└─────────────────────────────────────────────────────────────────┘
```

**Modern Reality:** In Kubernetes/AWS, you rarely implement discovery yourself:
- **Kubernetes**: Service DNS (`payment-service.namespace.svc.cluster.local`)
- **AWS**: ALB + Target Groups (auto-registers ECS tasks)
- **Legacy**: Eureka (only if not using K8s/cloud)

---

## 5. How does an API Gateway protect backend services?

```
┌─────────────────────────────────────────────────────────────────┐
│                      API GATEWAY                                 │
│                                                                 │
│   Internet → [Gateway] → Backend Services                       │
│                                                                 │
│   Protection Layers:                                            │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │ 1. AUTHENTICATION    - Validate JWT/API Keys            │  │
│   │ 2. RATE LIMITING     - 1000 req/min per client          │  │
│   │ 3. THROTTLING        - Prevent DDoS                     │  │
│   │ 4. IP WHITELISTING   - Block suspicious IPs             │  │
│   │ 5. REQUEST VALIDATION- Reject malformed requests        │  │
│   │ 6. SSL TERMINATION   - HTTPS at edge, HTTP internal     │  │
│   │ 7. CORS HANDLING     - Centralized CORS policy          │  │
│   │ 8. REQUEST TRANSFORM - Add headers, sanitize input      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   Backend services NEVER exposed directly to internet!          │
└─────────────────────────────────────────────────────────────────┘
```

```java
// Spring Cloud Gateway example
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("order-service", r -> r
            .path("/api/orders/**")
            .filters(f -> f
                .stripPrefix(1)
                .requestRateLimiter(c -> c.setRateLimiter(redisRateLimiter()))
                .circuitBreaker(c -> c.setFallbackUri("/fallback")))
            .uri("lb://order-service"))
        .build();
}
```

---

## 6. What happens when one dependent microservice goes down?

```
┌─────────────────────────────────────────────────────────────────┐
│              CASCADING FAILURE (Without Protection)              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   User → OrderService → PaymentService (DOWN!) ❌               │
│              │                                                  │
│              ├── Thread blocked waiting...                      │
│              ├── More requests come in...                       │
│              ├── Thread pool exhausted...                       │
│              └── OrderService ALSO crashes! 💥                  │
│                                                                 │
│   ONE service down → ENTIRE system down                         │
└─────────────────────────────────────────────────────────────────┘
```

**Solutions:**

```java
// 1. Circuit Breaker - Stop calling failed service
@CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
public Payment processPayment(Order order) {
    return paymentClient.charge(order);
}

public Payment paymentFallback(Order order, Exception e) {
    // Queue for later processing
    kafkaTemplate.send("payment-retry", order);
    return Payment.pending(order.getId());
}

// 2. Timeout - Don't wait forever
@TimeLimiter(name = "payment")  // 3 second timeout
public CompletableFuture<Payment> processPaymentAsync(Order order) {
    return CompletableFuture.supplyAsync(() -> paymentClient.charge(order));
}

// 3. Bulkhead - Isolate thread pools
@Bulkhead(name = "payment", type = Bulkhead.Type.THREADPOOL)
public Payment processPayment(Order order) {
    return paymentClient.charge(order);  // Uses separate thread pool
}
```

---

## 7. How do you implement retries without causing cascading failures?

```java
// ❌ BAD: Naive retry (causes thundering herd)
for (int i = 0; i < 3; i++) {
    try {
        return callService();
    } catch (Exception e) {
        // Immediate retry - all instances retry at same time!
    }
}

// ✅ GOOD: Exponential backoff with jitter
@Retry(name = "payment", fallbackMethod = "fallback")
public Payment processPayment(Order order) {
    return paymentClient.charge(order);
}

// application.yml
resilience4j:
  retry:
    instances:
      payment:
        maxAttempts: 3
        waitDuration: 1s
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        randomizedWaitFactor: 0.5  # JITTER - prevents thundering herd
        retryExceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
        ignoreExceptions:
          - com.example.BusinessException  # Don't retry business errors!
```

**Key Principles:**
1. **Exponential backoff**: 1s → 2s → 4s (not immediate)
2. **Jitter**: Random delay to spread retries
3. **Max attempts**: Limit retries (3-5)
4. **Selective retry**: Only for transient failures (network), NOT business errors
5. **Combine with Circuit Breaker**: Stop retrying if service is down

---

## 8. Why is Circuit Breaker important and how does Resilience4j help?

```
┌─────────────────────────────────────────────────────────────────┐
│                   CIRCUIT BREAKER STATES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   CLOSED ──────────────► OPEN ──────────────► HALF-OPEN        │
│   (Normal)    failures    (Reject all)   timeout   (Test)      │
│      ▲        exceed                        │         │         │
│      │        threshold                     │         │         │
│      └──────────────────────────────────────┴─────────┘         │
│              success in half-open                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```java
// Resilience4j Configuration
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallback")
@Retry(name = "paymentService")
@TimeLimiter(name = "paymentService")
@Bulkhead(name = "paymentService")
public CompletableFuture<Payment> processPayment(PaymentRequest req) {
    return CompletableFuture.supplyAsync(() -> paymentClient.charge(req));
}

// application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowSize: 10           # Last 10 calls
        failureRateThreshold: 50        # 50% failures → OPEN
        waitDurationInOpenState: 30s    # Wait before HALF-OPEN
        permittedNumberOfCallsInHalfOpenState: 3  # Test calls
        slowCallRateThreshold: 80       # 80% slow calls → OPEN
        slowCallDurationThreshold: 2s   # What counts as "slow"
```

**Why Important:**
- Fails fast instead of waiting for timeout
- Prevents resource exhaustion
- Gives failing service time to recover
- Returns fallback response immediately

---

## 9. How do you manage configuration across multiple services?

```
┌─────────────────────────────────────────────────────────────────┐
│              CONFIGURATION MANAGEMENT OPTIONS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Spring Cloud Config Server (Git-backed)                     │
│     ┌───────────┐     ┌─────────────┐     ┌──────────┐         │
│     │  Git Repo │────►│Config Server│────►│ Services │         │
│     └───────────┘     └─────────────┘     └──────────┘         │
│                                                                 │
│  2. Kubernetes ConfigMaps/Secrets                               │
│     kubectl create configmap app-config --from-file=app.yml    │
│                                                                 │
│  3. AWS Parameter Store / Secrets Manager                       │
│     /myapp/prod/database-url → jdbc:postgresql://...           │
│     /myapp/prod/api-key → (encrypted)                          │
│                                                                 │
│  4. HashiCorp Vault (Secrets)                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```java
// Spring Cloud Config Client
@RefreshScope  // Refresh config without restart
@RestController
public class MyController {
    
    @Value("${feature.new-ui.enabled}")
    private boolean newUiEnabled;
    
    // POST /actuator/refresh → reloads config
}

// AWS Parameter Store
@Configuration
public class AwsConfig {
    @Value("${spring.datasource.url}")  // Resolved from Parameter Store
    private String dbUrl;
}

// bootstrap.yml
spring:
  cloud:
    config:
      uri: http://config-server:8888
  # OR AWS
  aws:
    parameterstore:
      enabled: true
      prefix: /myapp
```

---

## 10. Why does a microservice behave differently after scaling pods?

**Common Issues:**

```
┌─────────────────────────────────────────────────────────────────┐
│  1. IN-MEMORY STATE (Sessions, Cache)                           │
│                                                                 │
│     Pod 1: session["user123"] = {...}                          │
│     Pod 2: session["user123"] = undefined ❌                    │
│                                                                 │
│     Fix: Use Redis for distributed session/cache                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  2. LOCAL FILE STORAGE                                          │
│                                                                 │
│     Pod 1 writes: /tmp/uploads/file.pdf                        │
│     Pod 2 reads: File not found! ❌                             │
│                                                                 │
│     Fix: Use S3/shared storage                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  3. SCHEDULED TASKS RUN MULTIPLE TIMES                          │
│                                                                 │
│     @Scheduled(cron = "0 0 * * *")  // Runs on ALL pods!       │
│                                                                 │
│     Fix: Use ShedLock or leader election                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  4. DATABASE CONNECTION POOL EXHAUSTION                         │
│                                                                 │
│     1 pod: 10 connections ✅                                    │
│     5 pods: 50 connections → DB max 30 ❌                       │
│                                                                 │
│     Fix: Reduce pool size per pod, use PgBouncer               │
└─────────────────────────────────────────────────────────────────┘
```

**Rule: Design for stateless pods!**

---

## 11. How do you secure microservices using Spring Security?

```java
// 1. API Gateway Authentication (Recommended)
@Configuration
@EnableWebFluxSecurity
public class GatewaySecurityConfig {
    
    @Bean
    public SecurityWebFilterChain securityFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(csrf -> csrf.disable())
            .authorizeExchange(auth -> auth
                .pathMatchers("/public/**").permitAll()
                .pathMatchers("/admin/**").hasRole("ADMIN")
                .anyExchange().authenticated())
            .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()))
            .build();
    }
}

// 2. Service-to-Service: Propagate JWT
@Component
public class JwtRelayInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        String token = SecurityContextHolder.getContext()
            .getAuthentication().getCredentials().toString();
        template.header("Authorization", "Bearer " + token);
    }
}

// 3. Internal Services: mTLS (Service Mesh handles this)
// No code needed - Istio/Linkerd inject sidecar for mTLS
```

**Security Layers:**
```
Internet → [WAF] → [API Gateway + JWT] → [Service Mesh + mTLS] → Service
              ↑           ↑                      ↑
           DDoS      Auth/AuthZ            Service-to-Service
```

---

## 12. JWT vs OAuth2 — where do people misuse them?

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMMON CONFUSIONS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OAuth2 = AUTHORIZATION FRAMEWORK (how to get tokens)           │
│  JWT    = TOKEN FORMAT (structure of the token)                 │
│                                                                 │
│  They're NOT alternatives - JWT is often USED BY OAuth2!        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Common Misuses:**

```
❌ MISUSE 1: Storing sensitive data in JWT
   JWT payload is Base64 encoded, NOT encrypted!
   Anyone can decode: jwt.io

❌ MISUSE 2: No token expiration
   JWT without exp claim = permanent access

❌ MISUSE 3: Using JWT for sessions
   Can't invalidate JWT (it's stateless)
   Use short expiry + refresh tokens

❌ MISUSE 4: Huge JWTs
   JWT in every request → bandwidth overhead
   Keep payload minimal

❌ MISUSE 5: Client-side only validation
   Always validate signature on server!
```

**Correct Usage:**
```java
// Access Token: Short-lived (15 min), contains minimal claims
{
  "sub": "user123",
  "roles": ["USER"],
  "exp": 1706400000  // 15 minutes
}

// Refresh Token: Long-lived, stored securely, used to get new access token
// Stored in HTTP-only cookie or secure storage
```

---

## 13. How do you handle distributed transactions in microservices?

```
┌─────────────────────────────────────────────────────────────────┐
│  ❌ DON'T: 2-Phase Commit (2PC) across microservices            │
│     - Locks resources                                           │
│     - Doesn't scale                                             │
│     - Single point of failure                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  ✅ DO: SAGA PATTERN                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CHOREOGRAPHY (Event-driven):                                   │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                     │
│  │ Order   │───►│ Payment │───►│ Inventory│                    │
│  │ Created │    │ Charged │    │ Reserved │                    │
│  └─────────┘    └─────────┘    └─────────┘                     │
│       │              │              │                           │
│       │   If fails, publish compensating events                │
│       │              │              │                           │
│       ◄──────────────┴──────────────┘                          │
│  Order         Payment          Inventory                       │
│  Cancelled     Refunded         Released                        │
│                                                                 │
│  ORCHESTRATION (Coordinator):                                   │
│           ┌─────────────────┐                                   │
│           │  Saga           │                                   │
│           │  Orchestrator   │                                   │
│           └────────┬────────┘                                   │
│        ┌───────────┼───────────┐                               │
│        ▼           ▼           ▼                               │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐                          │
│   │ Order   │ │ Payment │ │Inventory│                          │
│   └─────────┘ └─────────┘ └─────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```java
// Saga State tracking
public enum OrderSagaState {
    STARTED,
    PAYMENT_PENDING,
    PAYMENT_COMPLETED,
    INVENTORY_RESERVED,
    COMPLETED,
    PAYMENT_FAILED,      // Compensate: cancel order
    INVENTORY_FAILED     // Compensate: refund payment, cancel order
}
```

---

## 14. Why does database-per-service matter?

```
┌─────────────────────────────────────────────────────────────────┐
│  SHARED DATABASE (Anti-pattern)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OrderService ──┐                                               │
│                 ├──► [Shared PostgreSQL] ◄── PaymentService    │
│  UserService ───┘           │                                   │
│                             │                                   │
│  Problems:                  ▼                                   │
│  • Schema change breaks all services                            │
│  • Can't scale DB independently                                 │
│  • Tight coupling (services read each other's tables)          │
│  • Single point of failure                                      │
│  • Can't use different DB types per service                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  DATABASE PER SERVICE (Correct)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OrderService ───► [Order DB - PostgreSQL]                      │
│  PaymentService ──► [Payment DB - PostgreSQL]                   │
│  SearchService ───► [Search DB - Elasticsearch]                 │
│  SessionService ──► [Session DB - Redis]                        │
│                                                                 │
│  Benefits:                                                      │
│  • Independent deployment                                       │
│  • Right DB for the job (Polyglot Persistence)                 │
│  • Scale independently                                          │
│  • Schema changes isolated                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Data Sharing:** API calls or events, NEVER direct DB access!

---

## 15. How do you trace a request across multiple microservices?

```
┌─────────────────────────────────────────────────────────────────┐
│                   DISTRIBUTED TRACING                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request: POST /orders                                          │
│  Trace ID: abc-123 (same across all services)                  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ API Gateway                                              │   │
│  │ trace_id=abc-123, span_id=1, duration=350ms             │   │
│  └─────────────────────────────────────────────────────────┘   │
│        │                                                        │
│        ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Order Service                                            │   │
│  │ trace_id=abc-123, span_id=2, parent=1, duration=200ms   │   │
│  └─────────────────────────────────────────────────────────┘   │
│        │                                                        │
│        ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Payment Service                                          │   │
│  │ trace_id=abc-123, span_id=3, parent=2, duration=150ms   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```java
// Spring Boot 3 + Micrometer Tracing (auto-propagation)
// Just add dependency - works automatically!

// pom.xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>

// application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% sampling (reduce in prod)
  otlp:
    tracing:
      endpoint: http://jaeger:4318/v1/traces

// Logs automatically include trace ID
logger.info("Processing order");  
// Output: [abc-123] Processing order
```

**Tools:** Jaeger, Zipkin, AWS X-Ray, Datadog

---

## 16. How does Spring Cloud load balancing work internally?

```
┌─────────────────────────────────────────────────────────────────┐
│          SPRING CLOUD LOADBALANCER (Client-Side)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Service calls: http://payment-service/pay                  │
│                                                                 │
│  2. LoadBalancer intercepts:                                   │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  ServiceInstanceListSupplier                         │    │
│     │  → Fetch instances from:                             │    │
│     │    • Kubernetes Service                              │    │
│     │    • Eureka (legacy)                                 │    │
│     │    • Static list                                     │    │
│     │                                                      │    │
│     │  Returns: [10.0.0.1:8080, 10.0.0.2:8080, 10.0.0.3]  │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  3. Load Balancing Strategy:                                   │
│     • RoundRobin (default): 1 → 2 → 3 → 1 → 2 → ...          │
│     • Random: Pick random instance                             │
│     • WeightedResponseTime: Prefer faster instances            │
│                                                                 │
│  4. Replace URL: http://10.0.0.2:8080/pay                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```java
@Configuration
public class LoadBalancerConfig {
    
    @Bean
    @LoadBalanced  // This enables the magic!
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}

// Custom load balancing strategy
@Bean
public ReactorLoadBalancer<ServiceInstance> customLoadBalancer(
        ServiceInstanceListSupplier supplier) {
    return new RandomLoadBalancer(supplier, "payment-service");
}
```

---

## 17. What causes latency spikes in microservice architectures?

```
┌─────────────────────────────────────────────────────────────────┐
│                   LATENCY SPIKE CAUSES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. COLD STARTS                                                 │
│     • JVM warmup (JIT compilation)                              │
│     • Lambda cold starts                                        │
│     • New pod initialization                                    │
│     Fix: Keep warm, use GraalVM native                         │
│                                                                 │
│  2. GARBAGE COLLECTION                                          │
│     • Full GC pauses application                                │
│     Fix: Tune GC, use G1/ZGC, right-size heap                  │
│                                                                 │
│  3. CONNECTION POOL EXHAUSTION                                  │
│     • All DB connections in use → queue                        │
│     Fix: Increase pool, check for leaks                        │
│                                                                 │
│  4. N+1 QUERIES                                                 │
│     • 1 query + N lazy loads                                   │
│     Fix: JOIN fetch, batch loading                             │
│                                                                 │
│  5. DOWNSTREAM SERVICE SLOW                                     │
│     • Waiting for dependent service                            │
│     Fix: Timeout, circuit breaker, cache                       │
│                                                                 │
│  6. NETWORK LATENCY                                             │
│     • Cross-region calls                                        │
│     Fix: Co-locate services, use caching                       │
│                                                                 │
│  7. SERIALIZATION OVERHEAD                                      │
│     • Large JSON payloads                                       │
│     Fix: Use gRPC/Protobuf, paginate                           │
│                                                                 │
│  8. THREAD POOL SATURATION                                      │
│     • All threads busy                                          │
│     Fix: Async processing, increase threads, bulkhead          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 18. How do you prevent cascading failures?

```java
// DEFENSE IN DEPTH - Multiple layers of protection

// Layer 1: TIMEOUT - Don't wait forever
@TimeLimiter(name = "payment")
resilience4j.timelimiter.instances.payment.timeout-duration=3s

// Layer 2: CIRCUIT BREAKER - Stop calling failing service
@CircuitBreaker(name = "payment", fallbackMethod = "fallback")
resilience4j.circuitbreaker.instances.payment.failure-rate-threshold=50

// Layer 3: BULKHEAD - Isolate failures
@Bulkhead(name = "payment", type = Bulkhead.Type.THREADPOOL)
resilience4j.bulkhead.instances.payment.max-concurrent-calls=10

// Layer 4: RETRY - Handle transient failures
@Retry(name = "payment")
resilience4j.retry.instances.payment.max-attempts=3

// Layer 5: FALLBACK - Graceful degradation
public Payment fallback(PaymentRequest req, Exception e) {
    return Payment.pending();  // Queue for later
}

// Layer 6: RATE LIMITING - Protect from overload
@RateLimiter(name = "payment")
resilience4j.ratelimiter.instances.payment.limit-for-period=100
```

**Architecture Level:**
- Async communication where possible
- Message queues as buffers
- Cache aggressively
- Health checks + auto-restart

---

## 19. Why does synchronous communication become a bottleneck?

```
┌─────────────────────────────────────────────────────────────────┐
│            SYNCHRONOUS CHAIN = MULTIPLICATION OF LATENCY        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User → A (50ms) → B (100ms) → C (150ms) → D (80ms)            │
│                                                                 │
│  Total latency = 50 + 100 + 150 + 80 = 380ms                   │
│                                                                 │
│  If D is slow (500ms):                                          │
│  Total latency = 50 + 100 + 150 + 500 = 800ms ❌               │
│                                                                 │
│  If D is down:                                                  │
│  A, B, C all blocked waiting! Thread exhaustion! 💥            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│            ASYNC = DECOUPLED, RESILIENT                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User → A (50ms) → [Queue] → B processes async                 │
│              │                                                  │
│              └── Returns immediately: "Order accepted"          │
│                                                                 │
│  B, C, D process in background                                 │
│  If D slow/down → messages buffer in queue → no cascade        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When Sync is OK:**
- Read operations (GET)
- User needs immediate response
- Simple request-response

**When to use Async:**
- Write operations that can be eventual
- Long-running processes
- Fire-and-forget (notifications)

---

## 20. How do you design idempotent APIs?

```java
// IDEMPOTENT = Same request multiple times = Same result

// ❌ NOT Idempotent
POST /payments
{ "amount": 100 }  // Called twice = charged $200!

// ✅ Idempotent with Idempotency Key
POST /payments
Headers: Idempotency-Key: abc-123
{ "amount": 100 }  // Called twice = charged $100 (second is ignored)

@Service
public class PaymentService {
    
    @Autowired
    private RedisTemplate<String, String> redis;
    
    public Payment processPayment(String idempotencyKey, PaymentRequest req) {
        // 1. Check if already processed
        String existing = redis.opsForValue().get("idem:" + idempotencyKey);
        if (existing != null) {
            return objectMapper.readValue(existing, Payment.class);  // Return cached
        }
        
        // 2. Process payment
        Payment payment = paymentGateway.charge(req);
        
        // 3. Cache result with TTL
        redis.opsForValue().set(
            "idem:" + idempotencyKey, 
            objectMapper.writeValueAsString(payment),
            Duration.ofHours(24)
        );
        
        return payment;
    }
}
```

**Idempotent by Design:**
```
GET    - Always idempotent (read-only)
PUT    - Replace entire resource (idempotent)
DELETE - Delete resource (idempotent - already deleted = OK)
POST   - NOT idempotent by default → USE IDEMPOTENCY KEY
PATCH  - Depends on implementation
```

---

## 21. What happens when a message is processed twice?

```
┌─────────────────────────────────────────────────────────────────┐
│              DUPLICATE MESSAGE PROCESSING                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Why it happens:                                                │
│  • Consumer crashes after processing, before ACK               │
│  • Network issue → broker didn't receive ACK                   │
│  • At-least-once delivery guarantee                            │
│                                                                 │
│  Impact without protection:                                     │
│  • Customer charged twice                                       │
│  • Duplicate emails sent                                        │
│  • Inventory reduced twice                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solutions:**

```java
// 1. Idempotent Consumer (Database)
@KafkaListener(topics = "payments")
@Transactional
public void handlePayment(PaymentEvent event) {
    // Check if already processed
    if (processedEventRepository.existsById(event.getEventId())) {
        log.info("Duplicate event ignored: {}", event.getEventId());
        return;
    }
    
    // Process
    paymentService.process(event);
    
    // Mark as processed
    processedEventRepository.save(new ProcessedEvent(event.getEventId()));
}

// 2. Idempotent Consumer (Redis - faster)
public void handlePayment(PaymentEvent event) {
    Boolean isNew = redis.opsForValue().setIfAbsent(
        "processed:" + event.getEventId(), 
        "1",
        Duration.ofDays(7)
    );
    
    if (!isNew) {
        return;  // Already processed
    }
    
    paymentService.process(event);
}

// 3. Database Constraint
CREATE UNIQUE INDEX idx_unique_event ON orders(event_id);
// Insert fails on duplicate → transaction rollback → safe
```

---

## 22. How do you handle schema changes without breaking services?

```
┌─────────────────────────────────────────────────────────────────┐
│              BACKWARD COMPATIBLE CHANGES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ SAFE Changes:                                               │
│     • Add optional field (with default)                        │
│     • Add new endpoint                                          │
│     • Deprecate field (keep it)                                │
│                                                                 │
│  ❌ BREAKING Changes:                                           │
│     • Remove field                                              │
│     • Rename field                                              │
│     • Change field type                                         │
│     • Remove endpoint                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Strategies:**

```java
// 1. API Versioning
@GetMapping("/v1/users/{id}")  // Keep old
@GetMapping("/v2/users/{id}")  // Add new

// 2. Expand & Contract (Database)
// Step 1: Add new column (nullable)
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);

// Step 2: Backfill data
UPDATE users SET full_name = first_name || ' ' || last_name;

// Step 3: Update code to use new column
// Step 4: (Later) Remove old columns after all services migrated

// 3. Tolerant Reader (Jackson)
@JsonIgnoreProperties(ignoreUnknown = true)  // Ignore unknown fields
public class UserDto {
    private String name;
    // Old clients work even if new fields added
}

// 4. Consumer-Driven Contract Testing (Pact)
// Define contracts between services, test compatibility
```

**Message Schema Evolution (Kafka/Avro):**
```json
// Schema Registry enforces compatibility
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null}  // Optional
  ]
}
```

---

## 23. Why do timeouts matter more than retries?

```
┌─────────────────────────────────────────────────────────────────┐
│                 TIMEOUT vs RETRY                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT TIMEOUT:                                               │
│  Request → Wait forever → Thread blocked → Pool exhausted → 💥 │
│                                                                 │
│  WITH ONLY RETRY:                                               │
│  Request → 30s default timeout → Retry → 30s → Retry → 30s    │
│  = 90 seconds of blocked threads!                               │
│                                                                 │
│  WITH TIMEOUT + RETRY:                                          │
│  Request → 3s timeout → Retry → 3s timeout → Fallback          │
│  = 6 seconds max, resources freed quickly                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```java
// ALWAYS set timeout BEFORE retry!
@TimeLimiter(name = "payment")  // 3 seconds
@Retry(name = "payment")        // 3 attempts
@CircuitBreaker(name = "payment")
public Payment processPayment(PaymentRequest req) {
    return paymentClient.charge(req);
}

// application.yml
resilience4j:
  timelimiter:
    instances:
      payment:
        timeout-duration: 3s  # This is CRITICAL
  retry:
    instances:
      payment:
        max-attempts: 3
        wait-duration: 500ms

// HTTP Client timeout (RestTemplate/WebClient)
WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(
        HttpClient.create()
            .responseTimeout(Duration.ofSeconds(3))  // ALWAYS SET!
    ))
    .build();
```

**Rule of Thumb:**
```
Timeout < (Total allowed time) / (Number of retries)
If you have 10s budget and 3 retries: timeout = 3s
```

---

## 24. How does Spring Boot handle graceful shutdown in microservices?

```java
// application.yml
server:
  shutdown: graceful  # Enable graceful shutdown
  
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # Wait up to 30s

// What happens on SIGTERM:
// 1. Stop accepting new requests
// 2. Wait for in-flight requests to complete
// 3. Close connections
// 4. Shutdown

// Kubernetes needs matching config:
// deployment.yaml
spec:
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]  # Wait for LB to remove pod
      terminationGracePeriodSeconds: 45  # Must be > Spring timeout
```

```java
// Custom shutdown hook
@Component
public class GracefulShutdown {
    
    @PreDestroy
    public void onShutdown() {
        log.info("Shutting down...");
        // Flush buffers
        // Close connections
        // Deregister from discovery
    }
}

// For Kafka consumers
@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerFactory() {
    factory.getContainerProperties().setShutdownTimeout(10000);  // 10s
    return factory;
}
```

---

## 25. Why do microservices fail silently sometimes?

```
┌─────────────────────────────────────────────────────────────────┐
│              SILENT FAILURE CAUSES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. SWALLOWED EXCEPTIONS                                        │
│     try { ... } catch (Exception e) { }  // Empty catch!       │
│                                                                 │
│  2. ASYNC ERRORS NOT HANDLED                                    │
│     CompletableFuture.runAsync(() -> {                         │
│         throw new RuntimeException();  // Nobody catches this! │
│     });                                                         │
│                                                                 │
│  3. MESSAGE LOST IN QUEUE                                       │
│     Consumer throws exception → message dropped                │
│                                                                 │
│  4. FALLBACK HIDES FAILURE                                      │
│     Circuit breaker fallback returns default → looks "working" │
│                                                                 │
│  5. FIRE-AND-FORGET WITHOUT MONITORING                         │
│     kafkaTemplate.send("topic", msg);  // No error handling    │
│                                                                 │
│  6. MISSING HEALTH CHECKS                                       │
│     Service "UP" but can't reach database                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solutions:**

```java
// 1. Always log exceptions
catch (Exception e) {
    log.error("Payment failed for order {}", orderId, e);
    throw e;  // Or handle properly
}

// 2. Handle async errors
CompletableFuture.runAsync(() -> process())
    .exceptionally(ex -> {
        log.error("Async failed", ex);
        alertService.notify(ex);
        return null;
    });

// 3. DLQ for message failures
@KafkaListener(topics = "orders")
public void process(Order order) {
    try {
        orderService.process(order);
    } catch (Exception e) {
        kafkaTemplate.send("orders-dlq", order);  // Don't lose it!
        throw e;
    }
}

// 4. Monitor fallback rate
// If fallback called too often → alert!

// 5. Deep health checks
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    public Health health() {
        if (canQueryDatabase()) {
            return Health.up().build();
        }
        return Health.down().withDetail("error", "DB unreachable").build();
    }
}
```

---

## 26. How do you test microservices without mocking everything?

```
┌─────────────────────────────────────────────────────────────────┐
│                   TESTING STRATEGIES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. UNIT TESTS - Mock external dependencies                    │
│  2. INTEGRATION TESTS - Real DB, mocked external services      │
│  3. CONTRACT TESTS - Verify API contracts (Pact)               │
│  4. COMPONENT TESTS - Test service in isolation                │
│  5. E2E TESTS - Full system (use sparingly)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```java
// 1. Testcontainers (Real DB, no mocking!)
@SpringBootTest
@Testcontainers
class OrderServiceIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka"));
    
    @Test
    void shouldCreateOrder() {
        // Test with REAL Postgres and Kafka!
    }
}

// 2. WireMock (Mock external HTTP services)
@WireMockTest
class PaymentClientTest {
    
    @Test
    void shouldHandlePaymentSuccess(WireMockRuntimeInfo wmInfo) {
        stubFor(post("/payments")
            .willReturn(ok().withBody("{\"status\":\"SUCCESS\"}")));
        
        Payment result = paymentClient.charge(request);
        
        assertThat(result.getStatus()).isEqualTo("SUCCESS");
    }
    
    @Test
    void shouldHandleTimeout(WireMockRuntimeInfo wmInfo) {
        stubFor(post("/payments")
            .willReturn(ok().withFixedDelay(5000)));  // Simulate slow
        
        assertThrows(TimeoutException.class, 
            () -> paymentClient.charge(request));
    }
}

// 3. Contract Testing (Pact) - Consumer defines expectation
@PactTestFor(providerName = "payment-service")
void shouldReturnPaymentStatus(MockServer mockServer) {
    // Consumer test ensures provider meets contract
}
```

---

## 27. What causes memory leaks in Spring-based microservices?

```
┌─────────────────────────────────────────────────────────────────┐
│              COMMON MEMORY LEAKS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. UNCLOSED RESOURCES                                          │
│     InputStream, Connection, HttpClient not closed              │
│     Fix: try-with-resources                                     │
│                                                                 │
│  2. STATIC COLLECTIONS GROWING                                  │
│     static List<User> cache = new ArrayList<>();               │
│     Fix: Use bounded cache (Caffeine, Guava)                   │
│                                                                 │
│  3. THREADLOCAL NOT CLEANED                                     │
│     ThreadLocal holds reference → not GC'd                     │
│     Fix: Always remove() in finally block                      │
│                                                                 │
│  4. LISTENER/CALLBACK LEAKS                                     │
│     Added listeners never removed                               │
│     Fix: Weak references, proper cleanup                       │
│                                                                 │
│  5. HIBERNATE SESSION CACHE                                     │
│     Long transaction keeps entities in session                 │
│     Fix: Batch processing, clear session periodically          │
│                                                                 │
│  6. WEBCLIENT/RESTTEMPLATE NOT REUSED                          │
│     Creating new instance per request                          │
│     Fix: Use @Bean singleton                                   │
│                                                                 │
│  7. LOGGING CONTEXT NOT CLEARED                                 │
│     MDC not cleared after request                              │
│     Fix: Use filter to clear MDC                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```java
// ❌ Memory leak
public void processFile(String path) {
    InputStream is = new FileInputStream(path);
    // is never closed!
}

// ✅ Fixed
public void processFile(String path) {
    try (InputStream is = new FileInputStream(path)) {
        // Auto-closed
    }
}

// ❌ ThreadLocal leak
private static final ThreadLocal<User> userContext = new ThreadLocal<>();

public void process() {
    userContext.set(currentUser);
    // Never removed!
}

// ✅ Fixed
public void process() {
    try {
        userContext.set(currentUser);
        doWork();
    } finally {
        userContext.remove();  // ALWAYS clean up!
    }
}
```

---

## 28. How do you manage secrets securely across services?

```
┌─────────────────────────────────────────────────────────────────┐
│              SECRET MANAGEMENT OPTIONS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ DON'T:                                                      │
│     • Hardcode in code                                          │
│     • Commit to Git                                             │
│     • Store in plain text config files                         │
│     • Pass as environment variables (visible in process list)  │
│                                                                 │
│  ✅ DO:                                                         │
│     • AWS Secrets Manager                                       │
│     • HashiCorp Vault                                          │
│     • Kubernetes Secrets (encrypted at rest)                   │
│     • Azure Key Vault / GCP Secret Manager                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```java
// AWS Secrets Manager with Spring
// pom.xml: spring-cloud-starter-aws-secrets-manager-config

// bootstrap.yml
spring:
  cloud:
    aws:
      secretsmanager:
        enabled: true
        prefix: /myapp
        default-context: prod

// Secrets automatically injected:
@Value("${database-password}")  // From /myapp/prod/database-password
private String dbPassword;

// HashiCorp Vault
// bootstrap.yml
spring:
  cloud:
    vault:
      uri: https://vault.company.com
      authentication: KUBERNETES  # or TOKEN, AWS_IAM
      kv:
        enabled: true
        backend: secret

@Value("${api-key}")  // From Vault: secret/myapp/api-key
private String apiKey;

// Kubernetes Secrets (mounted as files)
// deployment.yaml
volumes:
  - name: secrets
    secret:
      secretName: app-secrets
volumeMounts:
  - name: secrets
    mountPath: /etc/secrets
    readOnly: true

// Read in Java
String password = Files.readString(Path.of("/etc/secrets/db-password"));
```

---

## 29. When should you NOT use microservices?

```
┌─────────────────────────────────────────────────────────────────┐
│           DON'T USE MICROSERVICES WHEN:                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. SMALL TEAM (< 5 developers)                                │
│     Overhead > Benefits                                         │
│     One person can't own multiple services                     │
│                                                                 │
│  2. EARLY STAGE / MVP                                          │
│     Domain boundaries unclear                                   │
│     You WILL split wrong                                       │
│     Start monolith → extract later                             │
│                                                                 │
│  3. SIMPLE DOMAIN                                               │
│     CRUD app doesn't need 10 services                          │
│                                                                 │
│  4. TIGHT LATENCY REQUIREMENTS                                  │
│     Network calls add latency                                   │
│     In-process calls faster                                    │
│                                                                 │
│  5. STRONG CONSISTENCY REQUIREMENTS                            │
│     Distributed transactions are HARD                          │
│     Banking core might need monolith                           │
│                                                                 │
│  6. NO DEVOPS CAPABILITY                                        │
│     Need: CI/CD, monitoring, logging, K8s                      │
│     Without these → operational nightmare                      │
│                                                                 │
│  7. ORGANIZATIONAL REASONS ONLY                                │
│     "Everyone is doing it" is not a reason                     │
│     Technical benefit must exist                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           USE MICROSERVICES WHEN:                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Large team (multiple teams working independently)          │
│  ✅ Clear domain boundaries (DDD contexts)                     │
│  ✅ Different scaling requirements per component               │
│  ✅ Different tech requirements per component                  │
│  ✅ Need independent deployments                               │
│  ✅ Have DevOps maturity                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The Golden Rule:**
> "Don't even consider microservices unless you have a system that's too complex to manage as a monolith." — Martin Fowler

**Start with Modular Monolith → Extract to Microservices when needed**
