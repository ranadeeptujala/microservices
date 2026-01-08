# Data Management Patterns

Patterns for managing data across distributed microservices.

---

## Database Per Service

**Rule**: Each service OWNS its data. No direct database sharing!

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATABASE PER SERVICE                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│   │  ORDER SERVICE  │   │  USER SERVICE   │   │PRODUCT SERVICE  │          │
│   └────────┬────────┘   └────────┬────────┘   └────────┬────────┘          │
│            │ OWNS                │ OWNS                │ OWNS               │
│            ▼                     ▼                     ▼                    │
│   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│   │   ORDER DB      │   │    USER DB      │   │   PRODUCT DB    │          │
│   │   (PostgreSQL)  │   │   (PostgreSQL)  │   │   (MongoDB)     │          │
│   └─────────────────┘   └─────────────────┘   └─────────────────┘          │
│                                                                             │
│   ❌ Order Service CANNOT directly query User DB                            │
│   ✅ Services communicate via APIs or Events                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Benefits:**
- ✅ Loose coupling - services evolve independently
- ✅ Independent scaling - scale each DB separately
- ✅ Technology freedom (Polyglot Persistence)
- ✅ Fault isolation

**Challenge:** No cross-service JOINs, distributed transactions → **Saga Pattern** solves this!

---

## Saga Pattern (Distributed Business Transactions)

**Definition**: Pattern to accomplish **DISTRIBUTED BUSINESS TRANSACTIONS** across multiple microservices.

### Important Distinction

- **Business Transaction** ≠ Database Transaction
- Business Transaction = Multiple DB transactions across different services

```
┌─────────────────────────────────────────────────────────────────────────────┐
│           BUSINESS TRANSACTION vs DATABASE TRANSACTION                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   DATABASE TRANSACTION:                 BUSINESS TRANSACTION:               │
│   • Single database                     • Multiple services                 │
│   • ACID                                • Multiple databases                │
│   • BEGIN...COMMIT                      • Multiple DB transactions          │
│   • Milliseconds                        • Can take seconds/minutes          │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │     BUSINESS TRANSACTION: "Place Order"                             │   │
│   │                                                                     │   │
│   │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐            │   │
│   │  │ DB Tx 1 │ + │ DB Tx 2 │ + │ DB Tx 3 │ + │ DB Tx 4 │            │   │
│   │  │ Order   │   │Inventory│   │ Payment │   │Shipping │            │   │
│   │  │ Service │   │ Service │   │ Service │   │ Service │            │   │
│   │  └─────────┘   └─────────┘   └─────────┘   └─────────┘            │   │
│   │                                                                     │   │
│   │  4 Services, 4 DBs, 4 DB Transactions = 1 Business Transaction     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Saga ID / Correlation ID

**Same ID across ALL services** to identify and rollback related records:

```sql
-- ALL tables have saga_id for correlation
CREATE TABLE orders (
    order_id VARCHAR(50) PRIMARY KEY,
    saga_id  VARCHAR(50) NOT NULL,  -- Same across all services!
    status   VARCHAR(20)
);

CREATE TABLE inventory_reservations (
    reservation_id VARCHAR(50) PRIMARY KEY,
    saga_id        VARCHAR(50) NOT NULL,  -- Same ID!
    status         VARCHAR(20)
);

CREATE TABLE payments (
    payment_id VARCHAR(50) PRIMARY KEY,
    saga_id    VARCHAR(50) NOT NULL,  -- Same ID!
    status     VARCHAR(20)
);

-- ROLLBACK by saga_id
UPDATE orders SET status='CANCELLED' WHERE saga_id = 'SAGA-12345';
UPDATE inventory_reservations SET status='RELEASED' WHERE saga_id = 'SAGA-12345';
UPDATE payments SET status='REFUNDED' WHERE saga_id = 'SAGA-12345';
```

### Choreography vs Orchestration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SAGA IMPLEMENTATION STYLES                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   CHOREOGRAPHY (Event-based)            ORCHESTRATION (Coordinator-based)   │
│   ──────────────────────────            ─────────────────────────────────   │
│                                                                             │
│   Services react to events              Central coordinator controls flow   │
│   No central controller                 Orchestrator manages state          │
│   Like a DANCE                          Like an ORCHESTRA                   │
│                                                                             │
│   Order ──► Inventory ──► Payment       Orchestrator ──► Order              │
│     │          │            │                │──────────► Inventory         │
│     │          │            │                │──────────► Payment           │
│     ◄──────────◄────────────┘                                               │
│   (Events flow between services)        (Coordinator calls each service)    │
│                                                                             │
│   ✅ Loose coupling                     ✅ Clear visibility                 │
│   ✅ No single point of failure         ✅ Easier debugging                 │
│   ❌ Hard to track flow                 ❌ Orchestrator is SPOF             │
│   ❌ Complex for many steps             ✅ Good for complex flows           │
│                                                                             │
│   Best for: 2-4 services                Best for: 5+ services               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### SagaState (Plain Object)

Just a simple POJO to track which steps completed:

```java
@Data
public class SagaState {
    private String sagaId;
    private String orderId;
    private boolean orderCreated;      // Step 1
    private boolean stockReserved;     // Step 2
    private boolean paymentProcessed;  // Step 3
    private boolean shipmentCreated;   // Step 4
}

// Used by orchestrator to know what to rollback
private void compensate(SagaState state) {
    if (state.isPaymentProcessed()) paymentService.refundBySagaId(state.getSagaId());
    if (state.isStockReserved()) inventoryService.releaseBySagaId(state.getSagaId());
    if (state.isOrderCreated()) orderService.cancelBySagaId(state.getSagaId());
}
```

---

## CQRS (Command Query Responsibility Segregation)

**Definition**: Separate the **READ database** and **WRITE database**. Databases may be **DIFFERENT types**.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CQRS PATTERN                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   COMMAND = Write operations (Create, Update, Delete)                       │
│   QUERY   = Read operations (Select, Search, Get)                           │
│   SEGREGATION = Separate them!                                              │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                     │   │
│   │   COMMANDS (Write)                    QUERIES (Read)                │   │
│   │        │                                    │                       │   │
│   │        ▼                                    ▼                       │   │
│   │   ┌─────────────┐                    ┌─────────────┐                │   │
│   │   │  WRITE DB   │       SYNC         │  READ DB    │                │   │
│   │   │ PostgreSQL  │ ──────────────────►│Elasticsearch│                │   │
│   │   │ (Normalized)│    (Events/CDC)    │(Denormalized)│               │   │
│   │   └─────────────┘                    └─────────────┘                │   │
│   │                                                                     │   │
│   │   CAN BE DIFFERENT DATABASE TYPES!                                  │   │
│   │                                                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### CQRS vs Read Replicas

| Aspect | Read Replicas | Full CQRS |
|--------|---------------|-----------|
| Schema | SAME | DIFFERENT (optimized) |
| Database Type | SAME (RDS → RDS) | Can be DIFFERENT (PostgreSQL → Elasticsearch) |
| Data | Exact copy | Transformed/Denormalized |
| Sync | Automatic (DB handles) | Manual (Events/CDC) |
| Complexity | Low | Higher |
| Use Case | Scale same queries | Different query patterns, complex search |

### Common CQRS Database Combinations

| Write DB | Read DB | Use Case |
|----------|---------|----------|
| PostgreSQL | Elasticsearch | Full-text search |
| PostgreSQL | Redis | Fast lookups, caching |
| PostgreSQL | MongoDB | Flexible document queries |
| MySQL | ClickHouse | Analytics, OLAP |
| Any | Same DB (Replica) | Simple read scaling |

### When to Use CQRS

```
USE READ REPLICAS (Simple):           USE FULL CQRS:
• Same queries, more volume           • Read/write patterns very different
• Schema works for both               • Complex search requirements
• Want automatic sync                 • Need denormalized read models
• Strong consistency needed           • Different DB types needed
```

---

## Event Sourcing

**Definition**: Store state changes as a **sequence of events**. Rebuild state by replaying events.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EVENT SOURCING                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   TRADITIONAL (Store current state):                                        │
│   ─────────────────────────────────                                         │
│   Account: { id: 123, balance: 500 }                                        │
│                                                                             │
│   EVENT SOURCING (Store all events):                                        │
│   ───────────────────────────────────                                       │
│   Event 1: AccountCreated { id: 123, balance: 0 }                           │
│   Event 2: MoneyDeposited { id: 123, amount: 1000 }                         │
│   Event 3: MoneyWithdrawn { id: 123, amount: 300 }                          │
│   Event 4: MoneyWithdrawn { id: 123, amount: 200 }                          │
│                                                                             │
│   Current State = Replay all events → balance: 500                          │
│                                                                             │
│   BENEFITS:                                                                 │
│   ✅ Complete audit trail                                                   │
│   ✅ Time travel (state at any point)                                       │
│   ✅ Event replay for debugging                                             │
│   ✅ Natural fit with CQRS                                                  │
│                                                                             │
│   CHALLENGES:                                                               │
│   ❌ Complex to implement                                                   │
│   ❌ Event schema evolution                                                 │
│   ❌ Eventual consistency                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Management Decision Matrix

| Challenge | Pattern |
|-----------|---------|
| Each service needs own data | Database per Service |
| Distributed transactions | Saga (Choreography/Orchestration) |
| Heavy read vs write | CQRS |
| Audit trail, time travel | Event Sourcing |
| Cross-service queries | API Composition |
| Data sync between services | Event-Driven / CDC |

---

[← Back to Index](./README.md) | [Next: Service Discovery Patterns →](./04-service-discovery-patterns.md)

