# Interview Tips: Pattern Questions

How to handle design pattern questions in interviews.

---

## The Common Confusion

Interviewers often confuse or mix up different types of patterns:
- **GoF Design Patterns** (Singleton, Factory, Observer...)
- **Microservice Patterns** (API Gateway, Saga, CQRS...)
- **Enterprise Patterns** (Repository, Unit of Work...)

---

## How to Answer "What Design Patterns Do You Know?"

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    INTERVIEW STRATEGY                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   OPTION 1: CLARIFY PROFESSIONALLY                                          │
│   ────────────────────────────────                                          │
│                                                                             │
│   "Are you asking about classic GoF design patterns like Singleton          │
│    and Factory, or architectural patterns like API Gateway and              │
│    Circuit Breaker? I can cover both."                                      │
│                                                                             │
│   OPTION 2: ANSWER BOTH                                                     │
│   ─────────────────────                                                     │
│                                                                             │
│   "I work with patterns at multiple levels:                                 │
│                                                                             │
│    At the code level - GoF patterns like Factory, Strategy, Observer        │
│    for object-oriented design.                                              │
│                                                                             │
│    At the architecture level - patterns like API Gateway, Circuit           │
│    Breaker, Saga for microservices.                                         │
│                                                                             │
│    Which area would you like to explore?"                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Guide: What They Probably Mean

| Question | They Probably Mean |
|----------|-------------------|
| "What design patterns do you know?" | GoF (Singleton, Factory, Observer...) |
| "What microservice patterns do you use?" | API Gateway, Saga, CQRS, Circuit Breaker |
| "What patterns have you used in your projects?" | Both - give examples of each |
| "How would you design this system?" | Architectural + Microservice patterns |

---

## Sample Answers

### "Tell me about design patterns you've used"

> "I regularly use both classic OOP design patterns and architectural patterns:
> 
> **At the code level**, I use Factory pattern for creating payment processors, Strategy pattern for different pricing algorithms, and Observer pattern for event handling.
> 
> **At the architecture level**, I've implemented API Gateway for routing, Circuit Breaker with Resilience4j for fault tolerance, and Saga pattern for distributed transactions.
> 
> Would you like me to dive deeper into any specific pattern?"

### "What's the difference between API Gateway and Facade?"

> "Great question! They're similar concepts at different levels:
> 
> **Facade (GoF)** is a code-level pattern that provides a simplified interface to a complex subsystem within a single application.
> 
> **API Gateway** is an architectural pattern that provides a single entry point for multiple microservices in a distributed system.
> 
> Both simplify complexity, but API Gateway also handles cross-cutting concerns like authentication, rate limiting, and routing across services."

---

## Patterns by Context

### If They Ask About Code-Level Patterns

| Pattern | Use Case | Example |
|---------|----------|---------|
| Singleton | Single instance | DB connection pool |
| Factory | Object creation | PaymentProcessorFactory |
| Strategy | Swap algorithms | Sorting, pricing strategies |
| Observer | Event notification | Order event listeners |
| Builder | Complex object construction | Order.builder()... |

### If They Ask About Architecture-Level Patterns

| Pattern | Use Case | Example |
|---------|----------|---------|
| API Gateway | Single entry point | Kong, AWS API Gateway |
| Circuit Breaker | Fault tolerance | Resilience4j, Istio |
| Saga | Distributed transactions | Order → Inventory → Payment |
| CQRS | Read/Write separation | PostgreSQL + Elasticsearch |
| Event Sourcing | Audit trail | Banking transactions |

---

## Common Follow-up Questions

### "Why would you use Saga instead of 2PC?"

> "Two-phase commit (2PC) requires all participants to be available and creates tight coupling. Saga uses compensating transactions, allowing:
> - Better availability (no distributed locks)
> - Independent service scaling
> - Eventual consistency (acceptable for most use cases)
> 
> I'd use 2PC only when strong consistency is absolutely required and latency isn't critical."

### "When would you NOT use microservice patterns?"

> "Not every system needs microservices patterns:
> - Small team (< 10 devs) - overhead isn't worth it
> - Simple domain - monolith is often better
> - Tight consistency requirements - distributed systems add complexity
> - Startups - start with modular monolith, extract later
> 
> The pattern should solve a real problem, not add unnecessary complexity."

---

## Pro Tips

1. **Always relate to real experience** - "In my last project, we used..."
2. **Mention trade-offs** - Shows you understand when NOT to use a pattern
3. **Ask clarifying questions** - Shows you think before answering
4. **Draw diagrams** - Especially for architectural patterns
5. **Know the classics** - Singleton, Factory, Observer come up often

---

## Topics to Avoid or Keep Brief

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TOPICS TO SKIP IN INTERVIEWS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ❌ SERVICE DISCOVERY (Eureka, Consul, Ribbon)                             │
│   ──────────────────────────────────────────────                            │
│   WHY AVOID:                                                                │
│   • Eureka/Ribbon are LEGACY (Netflix OSS, pre-2018)                        │
│   • Modern stacks handle it automatically:                                  │
│     - Kubernetes: Built-in DNS (CoreDNS)                                    │
│     - Cloud: ALB, API Gateway auto-discovers targets                        │
│   • Not much to discuss - it's infrastructure, not code                     │
│   • Shows you're outdated if you focus on Eureka                            │
│                                                                             │
│   IF ASKED: "We use ALB + K8s Services - discovery is handled by           │
│   infrastructure. I've never needed Eureka in modern cloud setups."         │
│                                                                             │
│   ──────────────────────────────────────────────────────────────────────    │
│                                                                             │
│   ✅ BETTER TOPICS TO FOCUS ON:                                             │
│   ─────────────────────────────                                             │
│   • API Gateway (routing, rate limiting, auth)                              │
│   • Saga Pattern (distributed transactions)                                 │
│   • CQRS (read/write separation)                                            │
│   • Circuit Breaker (fault tolerance)                                       │
│   • Event-Driven Architecture                                               │
│   • Service Mesh (if they ask about traffic management)                     │
│                                                                             │
│   These show you understand REAL challenges in distributed systems.         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### If They Insist on Service Discovery

> "In modern cloud-native environments, service discovery is largely handled by the platform:
> 
> - **Kubernetes** provides built-in DNS - services call `http://payment-service` and K8s resolves it
> - **ALB/Cloud Load Balancers** auto-discover healthy targets via target groups
> - **Service Mesh** (Istio) handles it through sidecars
> 
> Client-side discovery with Eureka was popular in the Netflix OSS era (2014-2018), but it added complexity to application code. Now infrastructure handles it, so our services stay decoupled from discovery logic."

---

## Pattern Hierarchy Reference

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PATTERN HIERARCHY                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ARCHITECTURAL STYLE (Highest level)                                       │
│   ├── Microservices                                                         │
│   ├── Monolith                                                              │
│   ├── Serverless                                                            │
│   └── Event-Driven                                                          │
│                                                                             │
│   ARCHITECTURAL PATTERNS (System level)                                     │
│   ├── API Gateway                                                           │
│   ├── Service Mesh                                                          │
│   ├── CQRS                                                                  │
│   └── Event Sourcing                                                        │
│                                                                             │
│   MICROSERVICE PATTERNS (Service level)                                     │
│   ├── Circuit Breaker                                                       │
│   ├── Saga                                                                  │
│   ├── Database per Service                                                  │
│   └── Strangler Fig                                                         │
│                                                                             │
│   ENTERPRISE PATTERNS (Application level)                                   │
│   ├── Repository                                                            │
│   ├── Unit of Work                                                          │
│   ├── Front Controller                                                      │
│   └── Service Layer                                                         │
│                                                                             │
│   DESIGN PATTERNS - GoF (Code level)                                        │
│   ├── Creational: Singleton, Factory, Builder                               │
│   ├── Structural: Adapter, Decorator, Facade                                │
│   └── Behavioral: Observer, Strategy, Command                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

[← Back to Index](./README.md)

