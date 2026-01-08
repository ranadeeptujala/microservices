# Microservice Design Patterns

Microservice design patterns are proven solutions to common challenges when building distributed systems.

---

## 📚 Pattern Categories

| # | Pattern Category | Description |
|---|------------------|-------------|
| 1 | [Decomposition Patterns](./01-decomposition-patterns.md) | How to break down monoliths into services |
| 2 | [Communication Patterns](./02-communication-patterns.md) | Sync/Async, API Gateway, BFF, gRPC, REST |
| 3 | [Data Management Patterns](./03-data-management-patterns.md) | Database per Service, Saga, CQRS, Event Sourcing |
| 4 | [Service Discovery Patterns](./04-service-discovery-patterns.md) | Server-side vs Client-side discovery, ALB, K8s |
| 5 | [Reliability Patterns](./05-reliability-patterns.md) | Circuit Breaker, Retry, Bulkhead |
| 6 | [Observability Patterns](./06-observability-patterns.md) | Distributed Tracing, Log Aggregation, Health Checks |
| 7 | [Security Patterns](./07-security-patterns.md) | JWT, mTLS, Service Mesh security |
| 8 | [Deployment Patterns](./08-deployment-patterns.md) | Sidecar, Ambassador, Strangler Fig |
| 9 | [Polyglot Microservices](./09-polyglot-microservices.md) | Multi-language, Multi-database architectures |
| 10 | [Service Mesh](./10-service-mesh.md) | Istio, Linkerd, traffic management without code |
| 11 | [IoT vs Message Queues](./11-iot-vs-message-queues.md) | MQTT vs Kafka, when to use what |
| 12 | [Design Patterns (GoF)](./12-design-patterns-gof.md) | Classic OOP patterns vs Microservice patterns |
| 13 | [Interview Tips](./13-interview-tips.md) | How to answer pattern questions |

---

## 🎯 Quick Reference

| Challenge | Pattern |
|-----------|---------|
| Service boundaries | Decompose by Business/Subdomain |
| Single entry point | API Gateway |
| Distributed transactions | Saga (Choreography/Orchestration) |
| Data isolation | Database per Service |
| Heavy read systems | CQRS |
| Service location | Service Discovery |
| Fault tolerance | Circuit Breaker, Retry, Bulkhead |
| Cross-service debugging | Distributed Tracing |
| Gradual migration | Strangler Fig |
| Different tech requirements | Polyglot Microservices |
| Traffic control without code | Service Mesh |

---

## 📊 Pattern Categories Summary

| Category | Level | Examples |
|----------|-------|----------|
| **Architectural Style** | System | Microservices, Monolith, Serverless |
| **Microservice Patterns** | Service/System | API Gateway, Saga, CQRS, Circuit Breaker |
| **Enterprise Patterns** | Application | Repository, Unit of Work, Front Controller |
| **Design Patterns (GoF)** | Class/Object | Singleton, Factory, Observer, Strategy |

---

## 📖 Additional Resources

- [microservices.io](https://microservices.io/patterns/index.html) - Comprehensive pattern catalog
- [Martin Fowler's Microservices Guide](https://martinfowler.com/microservices/)
- [Chris Richardson's Microservices Patterns Book](https://www.manning.com/books/microservices-patterns)

