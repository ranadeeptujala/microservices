# Microservice Design Patterns

Microservice design patterns are proven solutions to common challenges when building distributed systems. Here's an overview of the key patterns:

---

## 1. Decomposition Patterns

Decomposition is the **most critical decision** in microservices architecture. It determines how you break down a monolithic application into smaller, independent services. Poor decomposition leads to distributed monoliths—all the complexity of microservices with none of the benefits.

### 1.1 Decompose by Business Capability

**Definition**: Organize services around what the business does, not how it's implemented.

A business capability is something a business does to generate value. It's relatively stable over time, even as technology changes.

```
┌─────────────────────────────────────────────────────────────────┐
│                     E-Commerce Platform                          │
├─────────────┬─────────────┬─────────────┬─────────────┬─────────┤
│   Product   │    Order    │   Payment   │  Shipping   │  User   │
│  Management │  Management │  Processing │   & Delivery│ Account │
├─────────────┼─────────────┼─────────────┼─────────────┼─────────┤
│ - Catalog   │ - Cart      │ - Billing   │ - Tracking  │ - Auth  │
│ - Inventory │ - Checkout  │ - Refunds   │ - Carriers  │ - Prefs │
│ - Pricing   │ - History   │ - Fraud     │ - Labels    │ - Profile│
└─────────────┴─────────────┴─────────────┴─────────────┴─────────┘
```

**How to Identify Business Capabilities:**
1. Analyze the organizational structure
2. Look at business processes and workflows
3. Identify what generates revenue or supports operations
4. Map to departments (Sales, Marketing, Finance, HR, etc.)

**Example Mapping:**

| Business Capability | Microservice | Responsibilities |
|---------------------|--------------|------------------|
| Product Management | `product-service` | CRUD products, categories, search |
| Order Management | `order-service` | Create orders, track status, history |
| Payment Processing | `payment-service` | Process payments, refunds, invoicing |
| User Management | `user-service` | Registration, authentication, profiles |
| Notification | `notification-service` | Email, SMS, push notifications |

**Pros:**
- ✅ Services are stable (business capabilities rarely change)
- ✅ Clear ownership by business teams
- ✅ Natural alignment with organizational structure
- ✅ Easy to understand for non-technical stakeholders

**Cons:**
- ❌ Can be difficult to identify capabilities in complex domains
- ❌ May not align perfectly with data boundaries

---

### 1.2 Decompose by Subdomain (Domain-Driven Design)

**Definition**: Use DDD's strategic design to identify bounded contexts, then create one service per bounded context.

**Key DDD Concepts:**

```
┌────────────────────────────────────────────────────────────────────┐
│                          DOMAIN                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │   CORE DOMAIN    │  │SUPPORTING DOMAIN │  │ GENERIC DOMAIN   │  │
│  │                  │  │                  │  │                  │  │
│  │  Competitive     │  │  Necessary but   │  │  Common across   │  │
│  │  advantage       │  │  not core        │  │  industries      │  │
│  │                  │  │                  │  │                  │  │
│  │  Example:        │  │  Example:        │  │  Example:        │  │
│  │  - Pricing Algo  │  │  - Inventory     │  │  - Auth/Identity │  │
│  │  - Recommendation│  │  - Reporting     │  │  - Email Service │  │
│  │  - Matching      │  │  - CRM           │  │  - Payment Gateway│  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

**Bounded Context**: A boundary within which a domain model is defined and applicable. Same terms can mean different things in different contexts.

**Example - "Customer" in Different Contexts:**

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Sales Context  │     │ Support Context │     │Shipping Context │
├─────────────────┤     ├─────────────────┤     ├─────────────────┤
│ Customer:       │     │ Customer:       │     │ Customer:       │
│ - Lead Score    │     │ - Ticket History│     │ - Address       │
│ - Deal Size     │     │ - SLA Level     │     │ - Delivery Prefs│
│ - Pipeline Stage│     │ - Contact Info  │     │ - Access Code   │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

**Context Mapping - How Bounded Contexts Relate:**

| Relationship | Description | Example |
|--------------|-------------|---------|
| **Shared Kernel** | Two contexts share a subset of the domain model | Order & Shipping share `Address` |
| **Customer-Supplier** | Upstream context provides what downstream needs | Inventory supplies data to Order |
| **Conformist** | Downstream adopts upstream's model as-is | Using external payment API |
| **Anti-Corruption Layer** | Translate between contexts to prevent pollution | Legacy system integration |
| **Open Host Service** | Published API for multiple consumers | Public REST API |
| **Published Language** | Standard data format (JSON Schema, Protobuf) | Event schemas |

**Visualization:**

```
                    ┌─────────────────┐
                    │  Order Context  │
                    │   (CORE)        │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
    ┌─────────────────┐ ┌─────────┐ ┌─────────────────┐
    │Payment Context  │ │Inventory│ │Shipping Context │
    │  (GENERIC)      │ │(SUPPORT)│ │   (SUPPORT)     │
    └─────────────────┘ └─────────┘ └─────────────────┘
              │                              │
              │    Anti-Corruption Layer     │
              ▼                              ▼
    ┌─────────────────┐              ┌─────────────────┐
    │ Stripe/PayPal   │              │  FedEx/UPS API  │
    │  (External)     │              │   (External)    │
    └─────────────────┘              └─────────────────┘
```

**Pros:**
- ✅ Clear boundaries with explicit contracts
- ✅ Prevents model pollution across services
- ✅ Teams can work independently with their own ubiquitous language
- ✅ Natural fit for complex business domains

**Cons:**
- ❌ Requires deep domain understanding
- ❌ Steeper learning curve (DDD knowledge needed)
- ❌ Can lead to over-engineering if misapplied

---

### 1.3 Decompose by Use Case / User Story

**Definition**: Create services based on specific user interactions or use cases.

**Example - Video Streaming Platform:**

```
┌────────────────────────────────────────────────────────┐
│                    User Stories                         │
├────────────────────────────────────────────────────────┤
│ "As a user, I want to search for videos"    → Search   │
│ "As a user, I want to watch a video"        → Playback │
│ "As a user, I want to upload content"       → Upload   │
│ "As a user, I want personalized suggestions"→ Recommend│
│ "As a user, I want to manage my watchlist"  → Watchlist│
└────────────────────────────────────────────────────────┘
```

**When to Use:**
- Event-driven architectures
- Systems with distinct user journeys
- When business capabilities are unclear

---

### 1.4 Strangler Fig Pattern (Incremental Decomposition)

**Definition**: Gradually extract services from a monolith by "strangling" it piece by piece, like a strangler fig tree that grows around its host tree.

```
Phase 1: Monolith          Phase 2: Partial          Phase 3: Complete
┌──────────────────┐       ┌──────────────────┐      ┌─────┐ ┌─────┐
│     MONOLITH     │       │    MONOLITH      │      │Svc A│ │Svc B│
│                  │       │    (Reduced)     │      └─────┘ └─────┘
│  ┌────┐ ┌────┐   │       │    ┌────┐        │      ┌─────┐ ┌─────┐
│  │ A  │ │ B  │   │  →    │    │ C  │        │  →   │Svc C│ │Svc D│
│  ├────┤ ├────┤   │       │    └────┘        │      └─────┘ └─────┘
│  │ C  │ │ D  │   │       └──────────────────┘
│  └────┘ └────┘   │       ┌─────┐ ┌─────┐
└──────────────────┘       │Svc A│ │Svc B│ │Svc D│
                           └─────┘ └─────┘ └─────┘
```

---

## Strangler Fig: Module to Domain Services (Complete Guide)

### Step-by-Step Migration Process

#### Phase 0: Assessment & Planning

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MONOLITHIC APPLICATION                            │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │   User      │  │   Product   │  │   Order     │  │   Payment   │ │
│  │   Module    │  │   Module    │  │   Module    │  │   Module    │ │
│  │             │  │             │  │             │  │             │ │
│  │ - Auth      │  │ - Catalog   │  │ - Cart      │  │ - Checkout  │ │
│  │ - Profile   │  │ - Inventory │  │ - History   │  │ - Refunds   │ │
│  │ - Prefs     │  │ - Search    │  │ - Tracking  │  │ - Invoicing │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘ │
│         │                │                │                │        │
│         └────────────────┴────────────────┴────────────────┘        │
│                                   │                                  │
│                        ┌──────────▼──────────┐                      │
│                        │   SHARED DATABASE   │                      │
│                        │   (Single Schema)   │                      │
│                        └─────────────────────┘                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Assessment Checklist:**

| Question | Analysis |
|----------|----------|
| Which modules have clear boundaries? | Map dependencies between modules |
| Which modules change frequently? | High-change modules benefit most from extraction |
| Which modules have performance issues? | May need independent scaling |
| Which modules have the fewest dependencies? | Easiest to extract first |
| What data does each module own? | Plan database separation |

---

#### Phase 1: Identify Domain Boundaries

**Map Monolith Modules to Domain Services:**

```
┌──────────────────────────────────────────────────────────────────────┐
│                     DOMAIN MAPPING                                    │
├────────────────────┬─────────────────────┬───────────────────────────┤
│  Monolith Module   │   Domain Service    │   Bounded Context         │
├────────────────────┼─────────────────────┼───────────────────────────┤
│  User Module       │   user-service      │   Identity & Access       │
│  Product Module    │   product-service   │   Catalog Management      │
│  Order Module      │   order-service     │   Order Fulfillment       │
│  Payment Module    │   payment-service   │   Financial Transactions  │
│  Notification Code │   notif-service     │   Communication           │
└────────────────────┴─────────────────────┴───────────────────────────┘
```

**Identify Dependencies:**

```
                    ┌───────────────┐
                    │ User Module   │
                    └───────┬───────┘
                            │ used by
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
    ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
    │Product Module │ │ Order Module  │ │Payment Module │
    └───────┬───────┘ └───────┬───────┘ └───────┬───────┘
            │                 │                 │
            │    depends on   │   depends on    │
            └────────────────►├◄────────────────┘
                              ▼
                    ┌───────────────┐
                    │Inventory Data │
                    └───────────────┘
```

---

#### Phase 2: Set Up the Strangler Facade (API Gateway)

**Architecture with Facade:**

```
                         ┌─────────────────────┐
                         │      CLIENTS        │
                         │  (Web, Mobile, API) │
                         └──────────┬──────────┘
                                    │
                         ┌──────────▼──────────┐
                         │    API GATEWAY      │
                         │  (Strangler Facade) │
                         │                     │
                         │  Route Rules:       │
                         │  /users/* → New Svc │
                         │  /products/* → Old  │
                         │  /orders/* → Old    │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
           ┌────────▼────────┐            ┌────────▼────────┐
           │  NEW SERVICES   │            │    MONOLITH     │
           │                 │            │                 │
           │ ┌─────────────┐ │            │ ┌─────────────┐ │
           │ │user-service │ │            │ │Product Mod  │ │
           │ └─────────────┘ │            │ ├─────────────┤ │
           │                 │            │ │Order Module │ │
           │                 │            │ ├─────────────┤ │
           │                 │            │ │Payment Mod  │ │
           │                 │            │ └─────────────┘ │
           └─────────────────┘            └─────────────────┘
```

**API Gateway Configuration Example (Kong/NGINX):**

```yaml
# Route configuration
routes:
  # Extracted service - route to new microservice
  - name: user-service-route
    paths:
      - /api/v1/users
      - /api/v1/auth
      - /api/v1/profiles
    service: user-service
    strip_path: false
    
  # Still in monolith - route to legacy
  - name: legacy-products-route
    paths:
      - /api/v1/products
      - /api/v1/catalog
    service: monolith-service
    strip_path: false
    
  # Still in monolith
  - name: legacy-orders-route
    paths:
      - /api/v1/orders
    service: monolith-service
    strip_path: false
```

---

#### Phase 3: Extract First Service (Start with Lowest Risk)

**Extraction Strategy - Pick the Right Module First:**

| Priority | Module Type | Why |
|----------|-------------|-----|
| 1️⃣ | Leaf modules (no downstream deps) | Easiest, lowest risk |
| 2️⃣ | Utility/Cross-cutting (notifications) | Clear boundaries |
| 3️⃣ | Read-heavy modules | Can use CQRS, less data sync issues |
| 4️⃣ | Core business modules | Most complex, do last |

**Example: Extracting User Module to user-service**

```
BEFORE EXTRACTION:
┌─────────────────────────────────────────┐
│              MONOLITH                    │
│  ┌──────────────────────────────────┐   │
│  │         User Module               │   │
│  │  ┌────────┐ ┌────────┐ ┌───────┐ │   │
│  │  │ Auth   │ │Profile │ │ Prefs │ │   │
│  │  │ Logic  │ │ Logic  │ │ Logic │ │   │
│  │  └────┬───┘ └────┬───┘ └───┬───┘ │   │
│  │       └──────────┴─────────┘     │   │
│  │                  │               │   │
│  │          ┌───────▼───────┐       │   │
│  │          │  User Tables  │       │   │
│  │          │  (users,      │       │   │
│  │          │   roles,      │       │   │
│  │          │   sessions)   │       │   │
│  │          └───────────────┘       │   │
│  └──────────────────────────────────┘   │
│              SHARED DB                   │
└─────────────────────────────────────────┘

AFTER EXTRACTION:
┌─────────────────────────────────────────┐
│           user-service                   │
│  ┌──────────────────────────────────┐   │
│  │  ┌────────┐ ┌────────┐ ┌───────┐ │   │
│  │  │ Auth   │ │Profile │ │ Prefs │ │   │
│  │  │ API    │ │ API    │ │ API   │ │   │
│  │  └────────┘ └────────┘ └───────┘ │   │
│  └──────────────────────────────────┘   │
│          ┌───────────────┐              │
│          │  User DB      │ ← Own Database
│          │  (PostgreSQL) │              │
│          └───────────────┘              │
└─────────────────────────────────────────┘
              │
              │ REST/gRPC calls
              ▼
┌─────────────────────────────────────────┐
│     MONOLITH (User Module Removed)      │
│  ┌─────────────┐  ┌─────────────┐       │
│  │Product Mod  │  │ Order Mod   │       │
│  └──────┬──────┘  └──────┬──────┘       │
│         │                │              │
│         └────────┬───────┘              │
│           ┌──────▼──────┐               │
│           │  Legacy DB  │               │
│           │ (no user    │               │
│           │  tables)    │               │
│           └─────────────┘               │
└─────────────────────────────────────────┘
```

---

#### Phase 4: Handle Data Migration

**Database Decomposition Strategies:**

```
Strategy 1: SHARED DATABASE (Temporary)
┌─────────────────┐     ┌─────────────────┐
│  user-service   │     │    Monolith     │
└────────┬────────┘     └────────┬────────┘
         │                       │
         └───────────┬───────────┘
                     │
              ┌──────▼──────┐
              │ SHARED DB   │  ⚠️ Anti-pattern
              │ (temporary) │     but useful
              └─────────────┘     for transition

Strategy 2: DATABASE PER SERVICE (Target State)
┌─────────────────┐     ┌─────────────────┐
│  user-service   │     │    Monolith     │
└────────┬────────┘     └────────┬────────┘
         │                       │
    ┌────▼────┐             ┌────▼────┐
    │ User DB │             │Legacy DB│
    └─────────┘             └─────────┘

Strategy 3: DATA SYNC (During Migration)
┌─────────────────┐           ┌─────────────────┐
│  user-service   │           │    Monolith     │
└────────┬────────┘           └────────┬────────┘
         │                             │
    ┌────▼────┐    CDC/Events     ┌────▼────┐
    │ User DB │◄──────────────────│Legacy DB│
    │ (New)   │   (Debezium,      │ (Old)   │
    └─────────┘    Kafka Connect) └─────────┘
```

**Change Data Capture (CDC) Pattern:**

```
┌──────────────────────────────────────────────────────────────┐
│                     DATA SYNC FLOW                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   LEGACY DB          MESSAGE BUS           NEW SERVICE DB    │
│  ┌─────────┐        ┌──────────┐          ┌─────────┐       │
│  │ users   │───────►│  Kafka   │─────────►│ users   │       │
│  │ table   │ CDC    │  Topic   │ Consumer │ table   │       │
│  │         │(Debezium)│        │          │ (copy)  │       │
│  └─────────┘        └──────────┘          └─────────┘       │
│                                                              │
│  Write Path (During Migration):                              │
│  1. Writes go to Legacy DB                                   │
│  2. CDC captures changes                                     │
│  3. Events published to Kafka                                │
│  4. New service consumes and syncs                          │
│                                                              │
│  After Migration Complete:                                   │
│  1. Writes go directly to New Service                        │
│  2. CDC disabled                                             │
│  3. Legacy table deprecated                                  │
└──────────────────────────────────────────────────────────────┘
```

---

#### Phase 5: Handle Inter-Service Communication

**Replace Direct Calls with APIs:**

```java
// BEFORE: Direct method call in Monolith
public class OrderService {
    @Autowired
    private UserRepository userRepository;  // Direct DB access
    
    public Order createOrder(Long userId, OrderRequest request) {
        User user = userRepository.findById(userId);  // Direct query
        // ... create order logic
    }
}

// AFTER: API call to extracted service
public class OrderService {
    @Autowired
    private UserServiceClient userServiceClient;  // REST/gRPC client
    
    public Order createOrder(Long userId, OrderRequest request) {
        UserDTO user = userServiceClient.getUser(userId);  // API call
        // ... create order logic
    }
}
```

**Anti-Corruption Layer (ACL):**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORDER SERVICE (in Monolith)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │              ANTI-CORRUPTION LAYER                       │  │
│   │                                                         │  │
│   │  Translates between:                                    │  │
│   │  - Old User model (monolith) ←→ New UserDTO (service)   │  │
│   │  - Old field names ←→ New field names                   │  │
│   │  - Old data formats ←→ New data formats                 │  │
│   │                                                         │  │
│   │  ┌─────────────┐      ┌─────────────┐                   │  │
│   │  │ UserAdapter │ ───► │UserService  │                   │  │
│   │  │             │      │  Client     │                   │  │
│   │  └─────────────┘      └──────┬──────┘                   │  │
│   └──────────────────────────────┼──────────────────────────┘  │
│                                  │                              │
└──────────────────────────────────┼──────────────────────────────┘
                                   │ HTTP/gRPC
                                   ▼
                         ┌─────────────────┐
                         │  user-service   │
                         └─────────────────┘
```

---

#### Phase 6: Incremental Traffic Migration

**Canary Deployment / Feature Flags:**

```
┌──────────────────────────────────────────────────────────────────┐
│                    TRAFFIC ROUTING STRATEGY                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Week 1: 10% traffic to new service                              │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│ │
│  │ New Service (10%)              Monolith (90%)                │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Week 2: 25% traffic to new service                              │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│ │
│  │ New Service (25%)              Monolith (75%)                │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Week 3: 50% traffic to new service                              │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│ │
│  │ New Service (50%)              Monolith (50%)                │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Week 4: 100% traffic to new service                             │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│ │
│  │ New Service (100%)             Monolith (deprecated)         │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Feature Flag Implementation:**

```java
@RestController
public class UserController {
    
    @Autowired
    private FeatureFlagService featureFlags;
    
    @Autowired
    private LegacyUserService legacyService;  // Old monolith code
    
    @Autowired
    private UserServiceClient newService;      // New microservice client
    
    @GetMapping("/users/{id}")
    public UserDTO getUser(@PathVariable Long id) {
        if (featureFlags.isEnabled("use-new-user-service", id)) {
            // Route to new microservice
            return newService.getUser(id);
        } else {
            // Route to legacy monolith code
            return legacyService.getUser(id);
        }
    }
}
```

---

#### Phase 7: Decommission Old Module

**Checklist Before Removing Legacy Code:**

```
□ 100% traffic routed to new service for 2+ weeks
□ No errors in new service logs
□ Performance metrics equal or better than legacy
□ All dependent services updated to use new APIs
□ Database migration complete
□ Rollback plan tested
□ Legacy code marked as deprecated
□ Team communication complete
```

**Final Architecture:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                     DECOMPOSED ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                      ┌─────────────────┐                            │
│                      │   API GATEWAY   │                            │
│                      └────────┬────────┘                            │
│                               │                                     │
│         ┌─────────────────────┼─────────────────────┐               │
│         │                     │                     │               │
│         ▼                     ▼                     ▼               │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐         │
│  │user-service │      │product-svc  │      │order-service│         │
│  │             │      │             │      │             │         │
│  │ Domain:     │      │ Domain:     │      │ Domain:     │         │
│  │ Identity    │      │ Catalog     │      │ Fulfillment │         │
│  └──────┬──────┘      └──────┬──────┘      └──────┬──────┘         │
│         │                    │                    │                 │
│    ┌────▼────┐          ┌────▼────┐          ┌────▼────┐           │
│    │User DB  │          │Prod DB  │          │Order DB │           │
│    │(Postgres)│         │(MongoDB)│          │(Postgres)│          │
│    └─────────┘          └─────────┘          └─────────┘           │
│                                                                     │
│                    ┌─────────────────┐                              │
│                    │  Message Broker │                              │
│                    │    (Kafka)      │                              │
│                    └─────────────────┘                              │
│                                                                     │
│  ✅ Monolith fully decommissioned                                   │
│  ✅ Each service owns its domain                                    │
│  ✅ Independent scaling and deployment                              │
│  ✅ Technology diversity possible                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Complete Migration Timeline Example

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MIGRATION TIMELINE (6 Months)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Month 1-2: PREPARATION                                             │
│  ├── Domain analysis and bounded context mapping                    │
│  ├── Set up API Gateway (Strangler Facade)                          │
│  ├── Establish CI/CD pipelines for new services                     │
│  └── Set up monitoring and observability                            │
│                                                                     │
│  Month 2-3: FIRST EXTRACTION (User Service)                         │
│  ├── Extract user-service                                           │
│  ├── Implement Anti-Corruption Layer                                │
│  ├── Set up CDC for data sync                                       │
│  └── Gradual traffic migration (10% → 50% → 100%)                   │
│                                                                     │
│  Month 3-4: SECOND EXTRACTION (Product Service)                     │
│  ├── Extract product-service                                        │
│  ├── Handle product-user relationships                              │
│  └── Migrate product data                                           │
│                                                                     │
│  Month 4-5: THIRD EXTRACTION (Order Service)                        │
│  ├── Extract order-service (most complex)                           │
│  ├── Implement Saga pattern for distributed transactions            │
│  └── Handle order-product-user relationships                        │
│                                                                     │
│  Month 5-6: CLEANUP & OPTIMIZATION                                  │
│  ├── Decommission monolith                                          │
│  ├── Remove CDC sync (use APIs only)                                │
│  ├── Performance optimization                                       │
│  └── Documentation and team training                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 1.5 Decomposition Anti-Patterns (What to Avoid)

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| **Nano-services** | Too many tiny services | Combine related functionality |
| **Distributed Monolith** | Tightly coupled services | Proper bounded contexts |
| **Data-Driven Decomposition** | Services split by database tables | Focus on behavior, not data |
| **Technical Decomposition** | Services by tech layer (UI, API, DB) | Use business capabilities |

**Signs of Poor Decomposition:**
- 🚩 Changing one service requires changing others
- 🚩 Services need synchronous calls for basic operations
- 🚩 Shared database between services
- 🚩 Circular dependencies
- 🚩 God services that do too much

---

### 1.6 Decision Matrix: Which Pattern to Use?

| Scenario | Recommended Approach |
|----------|---------------------|
| Clear business domains | Decompose by Business Capability |
| Complex domain logic | Decompose by Subdomain (DDD) |
| Legacy modernization | Strangler Fig Pattern |
| Event-driven system | Decompose by Use Case |
| Startup / Unclear domain | Start with modular monolith, extract later |

---

## 2. Communication Patterns

### API Gateway
```
┌─────────────────┐
│   API Gateway   │  ← Single entry point
└────────┬────────┘
         │
    ┌────┴────┬─────────┐
    ▼         ▼         ▼
┌───────┐ ┌───────┐ ┌───────┐
│Service│ │Service│ │Service│
│   A   │ │   B   │ │   C   │
└───────┘ └───────┘ └───────┘
```
- Routes requests to appropriate services
- Handles authentication, rate limiting, load balancing

### Backend for Frontend (BFF)
- Separate API gateway per client type (mobile, web, desktop)
- Tailored responses for each frontend's needs

### Synchronous (Request/Response)
- REST or gRPC for real-time communication
- Simple but creates tight coupling

### Asynchronous (Event-Driven)
- Message queues (RabbitMQ, Kafka) for loose coupling
- Event sourcing and publish/subscribe patterns

---

## 3. Data Management Patterns

### Database per Service
```
┌─────────┐    ┌─────────┐    ┌─────────┐
│Service A│    │Service B│    │Service C│
└────┬────┘    └────┬────┘    └────┬────┘
     │              │              │
┌────▼────┐    ┌────▼────┐    ┌────▼────┐
│  DB A   │    │  DB B   │    │  DB C   │
└─────────┘    └─────────┘    └─────────┘
```
- Each service owns its data exclusively
- No direct database sharing between services

### Saga Pattern
- Manages distributed transactions across services
- **Choreography**: Services react to events
- **Orchestration**: Central coordinator manages the flow

### CQRS (Command Query Responsibility Segregation)
- Separate read and write models
- Optimized for high-read or complex query scenarios

### Event Sourcing
- Store state changes as a sequence of events
- Rebuild state by replaying events

---

## 4. Service Discovery Patterns

### Client-Side Discovery
- Client queries a service registry (e.g., Eureka, Consul)
- Client chooses an instance and makes the request

### Server-Side Discovery
- Load balancer queries the registry
- Client only knows the load balancer address

---

## 5. Reliability Patterns

### Circuit Breaker
```
CLOSED ──(failures)──► OPEN ──(timeout)──► HALF-OPEN
   ▲                                           │
   └────────(success)──────────────────────────┘
```
- Prevents cascade failures
- Fails fast when a service is down

### Retry with Exponential Backoff
- Retry failed requests with increasing delays
- Handles transient failures gracefully

### Bulkhead
- Isolate resources per service/operation
- Failure in one doesn't exhaust resources for others

---

## 6. Observability Patterns

### Distributed Tracing
- Track requests across service boundaries
- Tools: Jaeger, Zipkin, OpenTelemetry

### Log Aggregation
- Centralize logs from all services
- Tools: ELK Stack, Splunk, Loki

### Health Check API
- Each service exposes `/health` endpoint
- Used by orchestrators for liveness/readiness probes

---

## 7. Security Patterns

### Access Token (JWT)
- Stateless authentication via signed tokens
- Passed between services for identity propagation

### Service Mesh
- Infrastructure layer for service-to-service security
- Handles mTLS, traffic management (e.g., Istio, Linkerd)

---

## 8. Deployment Patterns

### Sidecar
- Helper container alongside main service
- Handles cross-cutting concerns (logging, proxying)

### Ambassador
- Proxy that handles outbound connections
- Manages retries, routing, TLS termination

### Strangler Fig
- Gradually migrate from monolith to microservices
- Route traffic incrementally to new services

---

## Quick Reference Table

| Challenge | Pattern |
|-----------|---------|
| Service boundaries | Decompose by Business/Subdomain |
| Single entry point | API Gateway |
| Distributed transactions | Saga |
| Data isolation | Database per Service |
| Service location | Service Discovery |
| Fault tolerance | Circuit Breaker, Retry, Bulkhead |
| Cross-service debugging | Distributed Tracing |
| Gradual migration | Strangler Fig |

---

## Additional Resources

- [microservices.io](https://microservices.io/patterns/index.html) - Comprehensive pattern catalog
- [Martin Fowler's Microservices Guide](https://martinfowler.com/microservices/)
- [Chris Richardson's Microservices Patterns Book](https://www.manning.com/books/microservices-patterns)

