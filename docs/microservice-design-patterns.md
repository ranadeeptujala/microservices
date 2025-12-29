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

Communication between microservices is one of the most critical architectural decisions. The choice impacts performance, reliability, coupling, and overall system complexity.

### Overview: Synchronous vs Asynchronous

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

### 2.1 API Gateway Pattern

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

**Key Responsibilities:**

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
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐          │
│  │  SSL TERMINATION │  │   MONITORING     │  │  REQUEST/RESPONSE│          │
│  │                  │  │                  │  │   AGGREGATION    │          │
│  │  HTTPS → HTTP    │  │  Metrics         │  │                  │          │
│  │  Certificate mgmt│  │  Logging         │  │  Combine multiple│          │
│  │  TLS offloading  │  │  Tracing         │  │  service calls   │          │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**API Gateway Implementation Example (Kong):**

```yaml
# kong.yml - Declarative Configuration
_format_version: "2.1"

services:
  - name: user-service
    url: http://user-service:8080
    routes:
      - name: user-routes
        paths:
          - /api/v1/users
          - /api/v1/auth
        strip_path: false
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: local
      - name: jwt
        config:
          secret_is_base64: false
          
  - name: order-service
    url: http://order-service:8080
    routes:
      - name: order-routes
        paths:
          - /api/v1/orders
    plugins:
      - name: rate-limiting
        config:
          minute: 50
      - name: request-transformer
        config:
          add:
            headers:
              - "X-Request-ID:$(uuid)"

consumers:
  - username: mobile-app
    keyauth_credentials:
      - key: mobile-api-key-12345
  - username: web-app
    keyauth_credentials:
      - key: web-api-key-67890
```

**Request Flow Through API Gateway:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          REQUEST FLOW                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   CLIENT                API GATEWAY                    BACKEND SERVICE      │
│     │                       │                               │               │
│     │  1. HTTPS Request     │                               │               │
│     │──────────────────────►│                               │               │
│     │                       │                               │               │
│     │              2. SSL Termination                       │               │
│     │              3. Authentication ✓                      │               │
│     │              4. Rate Limit Check ✓                    │               │
│     │              5. Request Transform                     │               │
│     │                       │                               │               │
│     │                       │  6. HTTP Request (internal)   │               │
│     │                       │──────────────────────────────►│               │
│     │                       │                               │               │
│     │                       │  7. Response                  │               │
│     │                       │◄──────────────────────────────│               │
│     │                       │                               │               │
│     │              8. Response Transform                    │               │
│     │              9. Cache (if applicable)                 │               │
│     │                       │                               │               │
│     │  10. Response         │                               │               │
│     │◄──────────────────────│                               │               │
│     │                       │                               │               │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Popular API Gateway Solutions:**

| Gateway | Type | Best For |
|---------|------|----------|
| **Kong** | Open Source | Kubernetes, plugins ecosystem |
| **AWS API Gateway** | Managed | AWS-native, serverless |
| **Azure API Management** | Managed | Azure ecosystem |
| **NGINX** | Open Source | High performance, simple setup |
| **Traefik** | Open Source | Docker/Kubernetes native |
| **Spring Cloud Gateway** | Framework | Java/Spring ecosystem |
| **Envoy** | Open Source | Service mesh, high performance |

**Pros & Cons:**

| Pros | Cons |
|------|------|
| ✅ Single entry point | ❌ Single point of failure |
| ✅ Cross-cutting concerns in one place | ❌ Additional latency |
| ✅ Simplified client code | ❌ Can become bottleneck |
| ✅ Protocol translation | ❌ Requires maintenance |
| ✅ Security enforcement | ❌ Team coordination needed |

---

### 2.1.1 High RPS: ALB + Sidecar vs API Gateway

**⚠️ Important**: At high RPS (1000+ req/sec), API Gateway can become a bottleneck. Consider using ALB (Application Load Balancer) directly with sidecar containers for JWT validation.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               API GATEWAY vs ALB + SIDECAR ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   TRADITIONAL API GATEWAY (Lower RPS)        ALB + SIDECAR (High RPS)       │
│   ────────────────────────────────────       ───────────────────────        │
│                                                                             │
│   ┌──────────┐                               ┌──────────┐                   │
│   │  Client  │                               │  Client  │                   │
│   └────┬─────┘                               └────┬─────┘                   │
│        │                                          │                         │
│        ▼                                          ▼                         │
│   ┌──────────────┐                          ┌──────────────┐                │
│   │ API GATEWAY  │ ← Bottleneck!            │     ALB      │ ← Scales       │
│   │              │                          │ (AWS/Cloud)  │   automatically│
│   │ • Routing    │                          │              │                │
│   │ • JWT Valid  │                          │ • Routing    │                │
│   │ • Rate Limit │                          │ • SSL Term   │                │
│   │ • Transform  │                          │ • Health Chk │                │
│   └──────┬───────┘                          └──────┬───────┘                │
│          │                                         │                        │
│          ▼                                         ▼                        │
│   ┌──────────────┐                     ┌─────────────────────────┐          │
│   │   Service    │                     │   Service Pod/Container │          │
│   └──────────────┘                     │  ┌───────────────────┐  │          │
│                                        │  │   SIDECAR PROXY   │  │          │
│                                        │  │   (Envoy/NGINX)   │  │          │
│                                        │  │                   │  │          │
│                                        │  │  • JWT Validation │  │          │
│                                        │  │  • Rate Limiting  │  │          │
│                                        │  │  • mTLS           │  │          │
│                                        │  │  • Metrics        │  │          │
│                                        │  └─────────┬─────────┘  │          │
│                                        │            │            │          │
│                                        │  ┌─────────▼─────────┐  │          │
│                                        │  │    APP CONTAINER  │  │          │
│                                        │  │    (Your Service) │  │          │
│                                        │  └───────────────────┘  │          │
│                                        └─────────────────────────┘          │
│                                                                             │
│   Throughput: ~5K-10K RPS              Throughput: ~100K+ RPS               │
│   Latency: +5-20ms                     Latency: +1-2ms                      │
│   Scaling: Manual/Limited              Scaling: Auto with pods              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Why ALB + Sidecar is Better for High RPS:**

| Aspect | API Gateway | ALB + Sidecar |
|--------|-------------|---------------|
| **Throughput** | 5K-50K RPS (depends on vendor) | 100K+ RPS (scales with pods) |
| **Latency** | +5-20ms overhead | +1-2ms overhead |
| **Scaling** | Centralized bottleneck | Distributed, scales horizontally |
| **Cost** | Per-request pricing (AWS API GW) | Per-hour ALB + minimal sidecar |
| **JWT Validation** | Centralized | Distributed (each pod) |
| **Single Point of Failure** | Yes | No (distributed) |
| **Complexity** | Lower | Higher (need service mesh knowledge) |

**RPS Threshold Decision:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         WHEN TO USE WHAT                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   RPS < 500          │  API Gateway is fine                                 │
│   ─────────────────  │  • Simpler architecture                              │
│                      │  • Centralized management                            │
│                      │  • Good for startups/MVPs                            │
│                                                                             │
│   500 < RPS < 5000   │  API Gateway with auto-scaling                       │
│   ─────────────────  │  • Kong/NGINX with multiple replicas                 │
│                      │  • Consider managed solutions (AWS API GW)           │
│                      │  • Monitor latency closely                           │
│                                                                             │
│   RPS > 5000         │  ALB + Sidecar (Service Mesh)                        │
│   ─────────────────  │  • Istio/Linkerd/Envoy                               │
│                      │  • JWT in sidecar                                    │
│                      │  • Distributed rate limiting                         │
│                      │  • Essential for high-scale systems                  │
│                                                                             │
│   RPS > 50000        │  ALB + Sidecar + Edge CDN                            │
│   ─────────────────  │  • CloudFront/Fastly/Cloudflare at edge              │
│                      │  • Cache at edge                                     │
│                      │  • DDoS protection                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Sidecar JWT Validation with Envoy:**

```yaml
# Envoy sidecar configuration for JWT validation
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 8080
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: local_service
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            cluster: local_app
                http_filters:
                  # JWT Authentication Filter
                  - name: envoy.filters.http.jwt_authn
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
                      providers:
                        auth0:
                          issuer: "https://your-tenant.auth0.com/"
                          audiences:
                            - "your-api-audience"
                          remote_jwks:
                            http_uri:
                              uri: "https://your-tenant.auth0.com/.well-known/jwks.json"
                              cluster: jwks_cluster
                              timeout: 5s
                            cache_duration: 600s
                          forward: true
                          forward_payload_header: "x-jwt-payload"
                      rules:
                        - match:
                            prefix: "/api/"
                          requires:
                            provider_name: auth0
                        - match:
                            prefix: "/health"
                          # No JWT required for health checks
                  # Rate Limiting Filter
                  - name: envoy.filters.http.local_ratelimit
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
                      stat_prefix: http_local_rate_limiter
                      token_bucket:
                        max_tokens: 1000
                        tokens_per_fill: 100
                        fill_interval: 1s
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: local_app
      connect_timeout: 0.25s
      type: STATIC
      load_assignment:
        cluster_name: local_app
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: 127.0.0.1
                      port_value: 8081  # Your app container
    - name: jwks_cluster
      connect_timeout: 5s
      type: LOGICAL_DNS
      dns_lookup_family: V4_ONLY
      load_assignment:
        cluster_name: jwks_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: your-tenant.auth0.com
                      port_value: 443
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
```

**Kubernetes Deployment with Sidecar:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 10  # Scale horizontally
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        # Main application container
        - name: order-service
          image: myregistry/order-service:v1.2.0
          ports:
            - containerPort: 8081  # Internal port
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          env:
            - name: SERVER_PORT
              value: "8081"
        
        # Envoy sidecar for JWT + Rate Limiting
        - name: envoy-sidecar
          image: envoyproxy/envoy:v1.28.0
          ports:
            - containerPort: 8080  # External port (receives traffic)
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
      
      volumes:
        - name: envoy-config
          configMap:
            name: envoy-sidecar-config

---
# ALB Ingress (AWS)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-service-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api/v1/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8080  # Envoy sidecar port
```

**Architecture Comparison for 10,000 RPS:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    10,000 RPS ARCHITECTURE COMPARISON                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   OPTION A: API GATEWAY                    OPTION B: ALB + SIDECAR          │
│   ─────────────────────                    ────────────────────────          │
│                                                                             │
│   ┌──────────────────┐                    ┌──────────────────┐              │
│   │  Kong Gateway    │                    │       ALB        │              │
│   │  (5 replicas)    │                    │  (auto-scales)   │              │
│   │                  │                    │                  │              │
│   │  ~2000 RPS each  │                    │  Routes to pods  │              │
│   │  P99: 15ms       │                    │  P99: 2ms        │              │
│   └────────┬─────────┘                    └────────┬─────────┘              │
│            │                                       │                        │
│            ▼                                       ▼                        │
│   ┌──────────────────┐                    ┌──────────────────┐              │
│   │  Service Pods    │                    │  Service Pods    │              │
│   │  (10 replicas)   │                    │  (10 replicas)   │              │
│   │                  │                    │  + Envoy sidecar │              │
│   │  ~1000 RPS each  │                    │  ~1000 RPS each  │              │
│   └──────────────────┘                    └──────────────────┘              │
│                                                                             │
│   Total Latency:                          Total Latency:                    │
│   Gateway: 15ms + Service: 20ms           ALB: 2ms + Sidecar: 1ms +         │
│   = 35ms P99                              Service: 20ms = 23ms P99          │
│                                                                             │
│   Cost (AWS):                             Cost (AWS):                       │
│   5x m5.large + API calls                 1x ALB + 10x (slightly larger     │
│   ~$500/month                             pods for sidecar) ~$300/month     │
│                                                                             │
│   Failure Mode:                           Failure Mode:                     │
│   Gateway down = 100% outage              Pod down = 10% capacity loss      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Hybrid Approach (Best of Both Worlds):**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HYBRID ARCHITECTURE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Use API Gateway for:                   Use ALB + Sidecar for:             │
│   ───────────────────                    ─────────────────────              │
│   • Public/External APIs                 • Internal service-to-service     │
│   • Third-party integrations             • High-throughput services        │
│   • API monetization/metering            • Latency-sensitive endpoints     │
│   • Complex transformations              • Services needing mTLS           │
│                                                                             │
│                                                                             │
│                         ┌─────────────────┐                                 │
│   External Traffic ────►│   API Gateway   │ (Lower RPS, complex logic)     │
│                         │   (Kong/AWS)    │                                 │
│                         └────────┬────────┘                                 │
│                                  │                                          │
│                                  ▼                                          │
│                         ┌─────────────────┐                                 │
│   Internal Traffic ────►│      ALB        │ (High RPS, simple routing)     │
│                         └────────┬────────┘                                 │
│                                  │                                          │
│              ┌───────────────────┼───────────────────┐                      │
│              ▼                   ▼                   ▼                      │
│       ┌──────────┐        ┌──────────┐        ┌──────────┐                 │
│       │ Service  │        │ Service  │        │ Service  │                 │
│       │ + Envoy  │        │ + Envoy  │        │ + Envoy  │                 │
│       │ Sidecar  │        │ Sidecar  │        │ Sidecar  │                 │
│       └──────────┘        └──────────┘        └──────────┘                 │
│                                                                             │
│       JWT validation, rate limiting, mTLS handled by sidecars               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Takeaways:**

| RPS Level | Recommended Architecture |
|-----------|-------------------------|
| < 500 RPS | API Gateway (simple) |
| 500 - 5K RPS | API Gateway (scaled) or start considering ALB + Sidecar |
| 5K - 50K RPS | ALB + Sidecar (Service Mesh) |
| > 50K RPS | ALB + Sidecar + CDN Edge |

---

### 2.2 Backend for Frontend (BFF) Pattern

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
│   │• SSR     │      │• Offline │      │  ops     │      │• MQTT    │       │
│   └────┬─────┘      └────┬─────┘      └────┬─────┘      └────┬─────┘       │
│        │                 │                 │                 │              │
│        └─────────────────┴────────┬────────┴─────────────────┘              │
│                                   │                                         │
│                                   ▼                                         │
│                    ┌──────────────────────────────┐                         │
│                    │      BACKEND SERVICES        │                         │
│                    │                              │                         │
│                    │  ┌────────┐  ┌────────┐     │                         │
│                    │  │ User   │  │Product │     │                         │
│                    │  │Service │  │Service │     │                         │
│                    │  └────────┘  └────────┘     │                         │
│                    │  ┌────────┐  ┌────────┐     │                         │
│                    │  │ Order  │  │Inventory│    │                         │
│                    │  │Service │  │Service │     │                         │
│                    │  └────────┘  └────────┘     │                         │
│                    └──────────────────────────────┘                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Why BFF? Different Clients, Different Needs:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EXAMPLE: PRODUCT LISTING                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   WEB CLIENT NEEDS:                    MOBILE CLIENT NEEDS:                 │
│   ─────────────────                    ────────────────────                 │
│   {                                    {                                    │
│     "id": "123",                         "id": "123",                       │
│     "name": "iPhone 15",                 "name": "iPhone 15",               │
│     "description": "...",                "price": 999,                      │
│     "price": 999,                        "thumbnail": "thumb.jpg"           │
│     "images": [                        }                                    │
│       "img1.jpg",                                                           │
│       "img2.jpg",                      • Smaller payload                    │
│       "img3.jpg"                       • Single image                       │
│     ],                                 • Optimized for bandwidth            │
│     "specifications": {...},                                                │
│     "reviews": [...],                  IOT DEVICE NEEDS:                    │
│     "relatedProducts": [...],          ────────────────────                 │
│     "seoMetadata": {...}               {                                    │
│   }                                      "id": "123",                       │
│                                          "available": true                  │
│   • Rich data for SEO                  }                                    │
│   • Multiple images                                                         │
│   • Full details                       • Minimal data                       │
│                                        • Binary format possible             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**BFF Implementation Example (Node.js/Express):**

```javascript
// mobile-bff/src/routes/products.js
const express = require('express');
const router = express.Router();

// Mobile BFF - Optimized for mobile clients
router.get('/products', async (req, res) => {
  // Call multiple backend services
  const [products, inventory] = await Promise.all([
    productService.getProducts({ limit: 20 }),
    inventoryService.getAvailability()
  ]);
  
  // Transform and aggregate for mobile
  const mobileResponse = products.map(product => ({
    id: product.id,
    name: product.name,
    price: product.price,
    thumbnail: product.images[0]?.thumbnail || null,  // Single small image
    inStock: inventory[product.id]?.available || false
    // Exclude: description, reviews, specs, SEO data
  }));
  
  res.json({
    products: mobileResponse,
    _meta: {
      count: mobileResponse.length,
      cached: false
    }
  });
});

// web-bff/src/routes/products.js - Different implementation
router.get('/products', async (req, res) => {
  // Web BFF - Rich data for web clients
  const [products, reviews, related, seo] = await Promise.all([
    productService.getProducts({ limit: 50, includeSpecs: true }),
    reviewService.getReviewSummaries(),
    recommendationService.getRelated(),
    seoService.getMetadata()
  ]);
  
  // Rich aggregation for web
  const webResponse = products.map(product => ({
    ...product,
    images: product.images,  // All images
    reviews: reviews[product.id],
    relatedProducts: related[product.id],
    seoMetadata: seo[product.id]
  }));
  
  res.json(webResponse);
});
```

**When to Use BFF:**

| Use BFF | Avoid BFF |
|---------|-----------|
| Multiple client types with different needs | Single client type |
| Different teams per client | Small team |
| Varying network conditions (mobile vs web) | Similar requirements |
| Need protocol translation | Simple pass-through |
| Client-specific aggregation needed | Clients can aggregate |

---

### 2.3 Synchronous Communication

#### 2.3.1 REST (Representational State Transfer)

**The most common communication pattern for microservices.**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          REST COMMUNICATION                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Order Service                                    User Service             │
│   ┌─────────────────┐                             ┌─────────────────┐       │
│   │                 │    GET /users/123           │                 │       │
│   │  Create Order   │────────────────────────────►│  Get User       │       │
│   │                 │                             │                 │       │
│   │                 │    200 OK                   │                 │       │
│   │                 │◄────────────────────────────│  {              │       │
│   │  Validate User  │    {                        │    "id": 123,   │       │
│   │       ↓         │      "id": 123,             │    "name":...   │       │
│   │  Process Order  │      "name": "John",        │  }              │       │
│   │                 │      "email": "..."         │                 │       │
│   └─────────────────┘    }                        └─────────────────┘       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**REST Best Practices:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        REST API DESIGN                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  RESOURCES & HTTP METHODS                                                   │
│  ────────────────────────                                                   │
│                                                                             │
│  GET    /users              → List all users                                │
│  GET    /users/{id}         → Get specific user                             │
│  POST   /users              → Create new user                               │
│  PUT    /users/{id}         → Update entire user                            │
│  PATCH  /users/{id}         → Partial update                                │
│  DELETE /users/{id}         → Delete user                                   │
│                                                                             │
│  NESTED RESOURCES                                                           │
│  ────────────────                                                           │
│                                                                             │
│  GET    /users/{id}/orders  → User's orders                                 │
│  POST   /users/{id}/orders  → Create order for user                         │
│                                                                             │
│  QUERY PARAMETERS                                                           │
│  ────────────────                                                           │
│                                                                             │
│  GET /products?category=electronics&sort=price&page=2&limit=20              │
│                                                                             │
│  VERSIONING                                                                 │
│  ──────────                                                                 │
│                                                                             │
│  /api/v1/users    (URL versioning - recommended)                            │
│  Accept: application/vnd.api.v1+json  (Header versioning)                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**REST Client Implementation (Java/Spring):**

```java
// Using Spring WebClient (reactive)
@Service
public class UserServiceClient {
    
    private final WebClient webClient;
    
    public UserServiceClient(WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("http://user-service:8080")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
    
    public Mono<UserDTO> getUser(Long userId) {
        return webClient.get()
            .uri("/api/v1/users/{id}", userId)
            .retrieve()
            .onStatus(HttpStatus::is4xxClientError, response -> 
                Mono.error(new UserNotFoundException(userId)))
            .onStatus(HttpStatus::is5xxServerError, response -> 
                Mono.error(new ServiceUnavailableException("User service unavailable")))
            .bodyToMono(UserDTO.class)
            .timeout(Duration.ofSeconds(5))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(500)));
    }
    
    public Mono<UserDTO> createUser(CreateUserRequest request) {
        return webClient.post()
            .uri("/api/v1/users")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(UserDTO.class);
    }
}
```

#### 2.3.2 gRPC (Google Remote Procedure Call)

**High-performance RPC framework using Protocol Buffers.**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          gRPC vs REST                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   REST (HTTP/1.1 + JSON)              gRPC (HTTP/2 + Protobuf)              │
│   ──────────────────────              ────────────────────────              │
│                                                                             │
│   {                                   message User {                        │
│     "id": 123,                          int64 id = 1;                       │
│     "name": "John",                     string name = 2;                    │
│     "email": "john@..."                 string email = 3;                   │
│   }                                   }                                     │
│                                                                             │
│   • Text-based (JSON)                 • Binary (smaller, faster)            │
│   • Human readable                    • Schema-first (proto files)          │
│   • HTTP/1.1 (one req/conn)           • HTTP/2 (multiplexing)               │
│   • Unidirectional                    • Bidirectional streaming             │
│   • Language agnostic                 • Strong typing, code generation      │
│                                                                             │
│   PERFORMANCE COMPARISON:                                                   │
│   ───────────────────────                                                   │
│                                                                             │
│   Payload Size:   REST JSON ~500 bytes  vs  gRPC Protobuf ~150 bytes       │
│   Latency:        REST ~50ms            vs  gRPC ~10ms                      │
│   Throughput:     REST ~1000 req/s      vs  gRPC ~5000 req/s               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**gRPC Service Definition (Proto file):**

```protobuf
// user_service.proto
syntax = "proto3";

package user;

option java_package = "com.example.user.grpc";
option java_multiple_files = true;

// Service definition
service UserService {
  // Unary RPC
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  
  // Server streaming
  rpc ListUsers(ListUsersRequest) returns (stream User);
  
  // Client streaming
  rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);
  
  // Bidirectional streaming
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

// Messages
message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  UserStatus status = 4;
  google.protobuf.Timestamp created_at = 5;
}

message GetUserRequest {
  int64 user_id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}

enum UserStatus {
  UNKNOWN = 0;
  ACTIVE = 1;
  INACTIVE = 2;
  SUSPENDED = 3;
}
```

**gRPC Communication Patterns:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       gRPC STREAMING PATTERNS                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. UNARY (Single Request → Single Response)                                │
│     ─────────────────────────────────────────                               │
│     Client ──── Request ────► Server                                        │
│     Client ◄─── Response ──── Server                                        │
│                                                                             │
│  2. SERVER STREAMING (Single Request → Stream of Responses)                 │
│     ──────────────────────────────────────────────────────                  │
│     Client ──── Request ────► Server                                        │
│     Client ◄─── Response 1 ── Server                                        │
│     Client ◄─── Response 2 ── Server                                        │
│     Client ◄─── Response N ── Server                                        │
│                                                                             │
│     Use case: Real-time updates, large result sets                          │
│                                                                             │
│  3. CLIENT STREAMING (Stream of Requests → Single Response)                 │
│     ──────────────────────────────────────────────────────                  │
│     Client ──── Request 1 ───► Server                                       │
│     Client ──── Request 2 ───► Server                                       │
│     Client ──── Request N ───► Server                                       │
│     Client ◄─── Response ───── Server                                       │
│                                                                             │
│     Use case: File uploads, batch operations                                │
│                                                                             │
│  4. BIDIRECTIONAL STREAMING (Stream ↔ Stream)                               │
│     ─────────────────────────────────────────                               │
│     Client ──── Request 1 ───► Server                                       │
│     Client ◄─── Response 1 ─── Server                                       │
│     Client ──── Request 2 ───► Server                                       │
│     Client ◄─── Response 2 ─── Server                                       │
│                                                                             │
│     Use case: Chat, gaming, real-time collaboration                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**When to Use REST vs gRPC:**

| Use REST | Use gRPC |
|----------|----------|
| Public APIs | Internal service-to-service |
| Browser clients | High-performance needs |
| Simple CRUD operations | Streaming required |
| Human-readable debugging | Polyglot microservices |
| Caching important (HTTP semantics) | Low latency critical |

---

### 2.4 Asynchronous Communication

#### 2.4.1 Message Queue Pattern

**Decouple services using message brokers.**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      MESSAGE QUEUE ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         POINT-TO-POINT (Queue)                              │
│   ┌──────────┐                                        ┌──────────┐         │
│   │ Producer │                                        │ Consumer │         │
│   │(Order Svc)────┐      ┌────────────────┐     ┌────│(Inventory)│         │
│   └──────────┘    │      │                │     │    └──────────┘         │
│                   └─────►│  ORDER_QUEUE   │─────┘                          │
│   ┌──────────┐           │                │                                │
│   │ Producer │───────────│  [msg][msg][msg]│                               │
│   │(Order Svc)           │                │                                │
│   └──────────┘           └────────────────┘                                │
│                                                                             │
│   • One message consumed by ONE consumer                                    │
│   • Load balancing across consumers                                         │
│   • Guaranteed delivery                                                     │
│                                                                             │
│                                                                             │
│                         PUBLISH-SUBSCRIBE (Topic)                           │
│                                                                             │
│   ┌──────────┐           ┌────────────────┐     ┌──────────┐               │
│   │ Publisher│           │                │     │Subscriber│               │
│   │(Order Svc)──────────►│  ORDER_EVENTS  │────►│(Email Svc)│              │
│   └──────────┘           │                │     └──────────┘               │
│                          │  [OrderCreated]│     ┌──────────┐               │
│                          │                │────►│Subscriber│               │
│                          │                │     │(Analytics)│              │
│                          └────────────────┘     └──────────┘               │
│                                           │     ┌──────────┐               │
│                                           └────►│Subscriber│               │
│                                                 │(Inventory)│              │
│                                                 └──────────┘               │
│                                                                             │
│   • One message consumed by ALL subscribers                                 │
│   • Broadcast pattern                                                       │
│   • Loose coupling                                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Message Broker Comparison:**

| Feature | RabbitMQ | Apache Kafka | AWS SQS | Redis Streams |
|---------|----------|--------------|---------|---------------|
| **Pattern** | Queue + Pub/Sub | Log-based | Queue | Stream |
| **Ordering** | Per-queue | Per-partition | FIFO queues | Per-stream |
| **Retention** | Until consumed | Configurable | 14 days max | Configurable |
| **Throughput** | ~50K msg/s | ~1M msg/s | ~3K msg/s | ~100K msg/s |
| **Use Case** | Task queues | Event streaming | Cloud-native | Caching + queue |
| **Delivery** | At-least-once | At-least-once | At-least-once | At-least-once |

**Message Structure Best Practices:**

```json
// Event message structure
{
  "eventId": "evt_abc123",
  "eventType": "order.created",
  "eventVersion": "1.0",
  "timestamp": "2024-01-15T10:30:00Z",
  "source": "order-service",
  "correlationId": "corr_xyz789",
  "data": {
    "orderId": "ord_456",
    "customerId": "cust_123",
    "totalAmount": 99.99,
    "items": [
      {"productId": "prod_1", "quantity": 2}
    ]
  },
  "metadata": {
    "traceId": "trace_111",
    "spanId": "span_222"
  }
}
```

**Kafka Producer/Consumer Example (Java):**

```java
// Producer
@Service
public class OrderEventPublisher {
    
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType("order.created")
            .timestamp(Instant.now())
            .data(OrderData.from(order))
            .build();
        
        kafkaTemplate.send("order-events", order.getId(), event)
            .addCallback(
                result -> log.info("Published event: {}", event.getEventId()),
                ex -> log.error("Failed to publish event", ex)
            );
    }
}

// Consumer
@Service
public class InventoryEventConsumer {
    
    @KafkaListener(
        topics = "order-events",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderEvent(
        @Payload OrderEvent event,
        @Header(KafkaHeaders.RECEIVED_KEY) String key,
        Acknowledgment ack
    ) {
        try {
            switch (event.getEventType()) {
                case "order.created":
                    inventoryService.reserveStock(event.getData());
                    break;
                case "order.cancelled":
                    inventoryService.releaseStock(event.getData());
                    break;
            }
            ack.acknowledge();  // Manual acknowledgment
        } catch (Exception e) {
            log.error("Failed to process event: {}", event.getEventId(), e);
            // Will be retried or sent to DLQ
        }
    }
}
```

#### 2.4.2 Event-Driven Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      EVENT-DRIVEN ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                              EVENT FLOW                                     │
│                                                                             │
│   ┌───────────┐                                                             │
│   │  Order    │  1. User places order                                       │
│   │  Service  │────────────────────────────────┐                            │
│   └───────────┘                                │                            │
│                                                ▼                            │
│                                    ┌─────────────────────┐                  │
│                                    │    EVENT BUS        │                  │
│                                    │    (Kafka)          │                  │
│                                    │                     │                  │
│                                    │  OrderCreated       │                  │
│                                    │  OrderPaid          │                  │
│                                    │  OrderShipped       │                  │
│                                    │  OrderDelivered     │                  │
│                                    └──────────┬──────────┘                  │
│                                               │                             │
│              ┌────────────────┬───────────────┼───────────────┬────────┐    │
│              │                │               │               │        │    │
│              ▼                ▼               ▼               ▼        ▼    │
│        ┌──────────┐    ┌──────────┐    ┌──────────┐   ┌──────────┐ ┌─────┐ │
│        │Inventory │    │ Payment  │    │Shipping  │   │  Email   │ │Audit│ │
│        │ Service  │    │ Service  │    │ Service  │   │ Service  │ │ Log │ │
│        │          │    │          │    │          │   │          │ │     │ │
│        │Reserve   │    │Charge    │    │Create    │   │Send      │ │Store│ │
│        │Stock     │    │Customer  │    │Shipment  │   │Confirm   │ │Event│ │
│        └──────────┘    └──────────┘    └──────────┘   └──────────┘ └─────┘ │
│                                                                             │
│   BENEFITS:                                                                 │
│   • Services don't know about each other (loose coupling)                   │
│   • Easy to add new consumers                                               │
│   • Temporal decoupling (async processing)                                  │
│   • Event replay for debugging/recovery                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Event Types:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          EVENT TYPES                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. DOMAIN EVENTS (Business Facts)                                          │
│     ─────────────────────────────                                           │
│     • OrderCreated, PaymentReceived, UserRegistered                         │
│     • Represent something that happened in the business                     │
│     • Immutable - cannot be changed                                         │
│     • Named in past tense                                                   │
│                                                                             │
│  2. INTEGRATION EVENTS (Cross-Service Communication)                        │
│     ──────────────────────────────────────────────                          │
│     • Published to external message broker                                  │
│     • May be filtered/transformed from domain events                        │
│     • Contract between services                                             │
│                                                                             │
│  3. COMMAND EVENTS (Request for Action)                                     │
│     ───────────────────────────────────                                     │
│     • ProcessPayment, SendEmail, ReserveInventory                           │
│     • Named in imperative form                                              │
│     • Expects action to be taken                                            │
│                                                                             │
│  4. NOTIFICATION EVENTS (Fire and Forget)                                   │
│     ─────────────────────────────────────                                   │
│     • Informational only                                                    │
│     • No response expected                                                  │
│     • UserLoggedIn, PageViewed                                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2.4.3 Request-Reply Pattern (Async)

**When you need a response but want async benefits.**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      ASYNC REQUEST-REPLY                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Order Service                Message Broker               Payment Service │
│   ─────────────               ──────────────               ─────────────── │
│        │                            │                            │          │
│        │  1. Send Request           │                            │          │
│        │  (with correlation ID)     │                            │          │
│        │───────────────────────────►│                            │          │
│        │                            │  2. Forward Request        │          │
│        │                            │───────────────────────────►│          │
│        │                            │                            │          │
│        │                            │                            │ Process  │
│        │                            │                            │ Payment  │
│        │                            │                            │          │
│        │                            │  3. Send Reply             │          │
│        │                            │◄───────────────────────────│          │
│        │  4. Receive Reply          │  (same correlation ID)     │          │
│        │  (match correlation ID)    │                            │          │
│        │◄───────────────────────────│                            │          │
│        │                            │                            │          │
│                                                                             │
│   Implementation:                                                           │
│   • Request Queue: payment-requests                                         │
│   • Reply Queue: order-service-replies (or temp queue)                      │
│   • Correlation ID links request to response                                │
│   • Timeout handling required                                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 2.5 Service Mesh

**Infrastructure layer for service-to-service communication.**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SERVICE MESH ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                           CONTROL PLANE                                     │
│                    ┌─────────────────────────┐                              │
│                    │  (Istio/Linkerd/Consul) │                              │
│                    │                         │                              │
│                    │  • Configuration        │                              │
│                    │  • Service Discovery    │                              │
│                    │  • Certificate Mgmt     │                              │
│                    │  • Policy Engine        │                              │
│                    └───────────┬─────────────┘                              │
│                                │                                            │
│              ┌─────────────────┼─────────────────┐                          │
│              │                 │                 │                          │
│              ▼                 ▼                 ▼                          │
│   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐              │
│   │   Service A     │ │   Service B     │ │   Service C     │              │
│   │   Pod/Container │ │   Pod/Container │ │   Pod/Container │              │
│   │  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │              │
│   │  │   App     │  │ │  │   App     │  │ │  │   App     │  │              │
│   │  └─────┬─────┘  │ │  └─────┬─────┘  │ │  └─────┬─────┘  │              │
│   │        │        │ │        │        │ │        │        │              │
│   │  ┌─────▼─────┐  │ │  ┌─────▼─────┐  │ │  ┌─────▼─────┐  │              │
│   │  │  Sidecar  │  │ │  │  Sidecar  │  │ │  │  Sidecar  │  │              │
│   │  │  Proxy    │◄─┼─┼─►│  Proxy    │◄─┼─┼─►│  Proxy    │  │              │
│   │  │  (Envoy)  │  │ │  │  (Envoy)  │  │ │  │  (Envoy)  │  │              │
│   │  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │              │
│   └─────────────────┘ └─────────────────┘ └─────────────────┘              │
│                                                                             │
│                           DATA PLANE                                        │
│                    (All traffic flows through proxies)                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Service Mesh Capabilities:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      SERVICE MESH FEATURES                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  TRAFFIC MANAGEMENT              │  SECURITY                                │
│  ──────────────────              │  ────────                                │
│  • Load balancing                │  • Mutual TLS (mTLS)                     │
│  • Traffic splitting (canary)    │  • Service-to-service auth              │
│  • Circuit breaking              │  • Policy enforcement                   │
│  • Retries & timeouts            │  • Certificate rotation                 │
│  • Rate limiting                 │                                         │
│                                  │                                         │
│  OBSERVABILITY                   │  RESILIENCE                             │
│  ─────────────                   │  ──────────                             │
│  • Distributed tracing           │  • Fault injection                      │
│  • Metrics collection            │  • Chaos testing                        │
│  • Access logging                │  • Health checks                        │
│  • Service graph visualization   │  • Outlier detection                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Popular Service Mesh Solutions:**

| Solution | Platform | Key Features |
|----------|----------|--------------|
| **Istio** | Kubernetes | Most feature-rich, complex |
| **Linkerd** | Kubernetes | Lightweight, simple |
| **Consul Connect** | Any | HashiCorp ecosystem |
| **AWS App Mesh** | AWS | AWS-native |
| **Cilium** | Kubernetes | eBPF-based, high performance |

**Istio Traffic Management Example:**

```yaml
# Virtual Service - Traffic splitting (Canary)
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

---
# Destination Rule - Circuit breaker
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
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

---

### 2.6 Communication Anti-Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     COMMUNICATION ANTI-PATTERNS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ❌ CHATTY SERVICES                                                         │
│  ──────────────────                                                         │
│                                                                             │
│  BAD:                                                                       │
│  Order Service makes 10+ calls to get order details:                        │
│  1. GET /users/123                                                          │
│  2. GET /users/123/address                                                  │
│  3. GET /users/123/preferences                                              │
│  4. GET /products/456                                                       │
│  5. GET /products/456/price                                                 │
│  6. GET /inventory/456                                                      │
│  7. ... and so on                                                           │
│                                                                             │
│  GOOD:                                                                      │
│  • Use aggregation endpoints: GET /users/123/full-profile                   │
│  • Implement BFF to aggregate                                               │
│  • Consider GraphQL for flexible queries                                    │
│                                                                             │
│  ❌ SYNC CHAINS (Distributed Monolith)                                      │
│  ──────────────────────────────────────                                     │
│                                                                             │
│  BAD:                                                                       │
│  A → B → C → D → E (all synchronous)                                        │
│  • Latency = sum of all services                                            │
│  • Failure in any breaks the chain                                          │
│  • Tight coupling                                                           │
│                                                                             │
│  GOOD:                                                                      │
│  • Use async where possible                                                 │
│  • Limit sync depth to 2-3 services                                         │
│  • Implement circuit breakers                                               │
│                                                                             │
│  ❌ SHARED DATABASE COMMUNICATION                                           │
│  ────────────────────────────────                                           │
│                                                                             │
│  BAD:                                                                       │
│  Service A writes to DB → Service B polls DB for changes                    │
│                                                                             │
│  GOOD:                                                                      │
│  • Use events/messages for communication                                    │
│  • Each service owns its data                                               │
│                                                                             │
│  ❌ MISSING TIMEOUTS & RETRIES                                              │
│  ────────────────────────────────                                           │
│                                                                             │
│  BAD:                                                                       │
│  client.get("/api/users/123")  // No timeout!                               │
│                                                                             │
│  GOOD:                                                                      │
│  client.get("/api/users/123", {                                             │
│    timeout: 5000,                                                           │
│    retries: 3,                                                              │
│    retryDelay: exponentialBackoff                                           │
│  })                                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 2.7 Communication Pattern Decision Matrix

| Scenario | Recommended Pattern |
|----------|---------------------|
| Public APIs, Browser clients | REST + API Gateway |
| High-performance internal calls | gRPC |
| Mobile apps with varied needs | BFF + REST |
| Fire-and-forget operations | Message Queue (Async) |
| Event notifications to multiple services | Pub/Sub (Kafka) |
| Complex orchestration | Saga with Message Broker |
| Need response but async benefits | Request-Reply Pattern |
| Complex traffic management needs | Service Mesh |
| Real-time bidirectional | gRPC Streaming / WebSockets |

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

## 9. Polyglot Microservices

**Definition**: Using different programming languages, frameworks, and databases for different microservices based on what's best suited for each service's specific requirements.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      POLYGLOT MICROSERVICES ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                           ┌─────────────────┐                               │
│                           │   API Gateway   │                               │
│                           └────────┬────────┘                               │
│                                    │                                        │
│       ┌────────────────────────────┼────────────────────────────┐           │
│       │                            │                            │           │
│       ▼                            ▼                            ▼           │
│  ┌─────────────┐            ┌─────────────┐            ┌─────────────┐      │
│  │   User      │            │   Product   │            │  Analytics  │      │
│  │   Service   │            │   Service   │            │   Service   │      │
│  │             │            │             │            │             │      │
│  │  ┌───────┐  │            │  ┌───────┐  │            │  ┌───────┐  │      │
│  │  │ Java  │  │            │  │Node.js│  │            │  │Python │  │      │
│  │  │Spring │  │            │  │Express│  │            │  │FastAPI│  │      │
│  │  └───────┘  │            │  └───────┘  │            │  └───────┘  │      │
│  └──────┬──────┘            └──────┬──────┘            └──────┬──────┘      │
│         │                          │                          │             │
│    ┌────▼────┐                ┌────▼────┐                ┌────▼────┐        │
│    │PostgreSQL│               │ MongoDB │                │ ClickHouse│       │
│    │  (RDBMS) │               │(Document)│               │(Columnar) │       │
│    └─────────┘                └─────────┘                └──────────┘       │
│                                                                             │
│       ▼                            ▼                            ▼           │
│  ┌─────────────┐            ┌─────────────┐            ┌─────────────┐      │
│  │  Payment    │            │   Search    │            │  Real-time  │      │
│  │  Service    │            │   Service   │            │  Messaging  │      │
│  │             │            │             │            │             │      │
│  │  ┌───────┐  │            │  ┌───────┐  │            │  ┌───────┐  │      │
│  │  │  Go   │  │            │  │ Rust  │  │            │  │ Elixir│  │      │
│  │  │       │  │            │  │       │  │            │  │Phoenix│  │      │
│  │  └───────┘  │            │  └───────┘  │            │  └───────┘  │      │
│  └──────┬──────┘            └──────┬──────┘            └──────┬──────┘      │
│         │                          │                          │             │
│    ┌────▼────┐                ┌────▼────┐                ┌────▼────┐        │
│    │  MySQL  │                │Elastic  │                │  Redis  │        │
│    │ (ACID)  │                │ search  │                │(In-memory)│       │
│    └─────────┘                └─────────┘                └──────────┘       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 9.1 Polyglot Programming (Multiple Languages)

**Choose the right language for the job:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LANGUAGE SELECTION BY USE CASE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  USE CASE                    │ BEST LANGUAGES         │ WHY                 │
│  ────────────────────────────┼────────────────────────┼──────────────────── │
│                              │                        │                     │
│  Enterprise/Complex Business │ Java, Kotlin, C#       │ Strong typing,      │
│  Logic, Banking, Insurance   │                        │ mature ecosystem    │
│                              │                        │                     │
│  High Performance, Low       │ Go, Rust, C++          │ Memory efficiency,  │
│  Latency, System Programming │                        │ no GC pauses        │
│                              │                        │                     │
│  Data Science, ML/AI,        │ Python                 │ ML libraries        │
│  Analytics, ETL              │                        │ (TensorFlow, PyTorch│
│                              │                        │  Pandas, NumPy)     │
│                              │                        │                     │
│  Rapid Prototyping, APIs,    │ Node.js, Python        │ Fast development,   │
│  CRUD Services               │                        │ async I/O           │
│                              │                        │                     │
│  Real-time, Concurrent,      │ Elixir/Erlang, Go      │ Actor model,        │
│  Chat, Gaming                │                        │ lightweight threads │
│                              │                        │                     │
│  Search, Text Processing,    │ Rust, Go, Java         │ Performance +       │
│  Indexing                    │                        │ concurrency         │
│                              │                        │                     │
│  Frontend/Full-stack,        │ Node.js, TypeScript    │ Same language       │
│  Server-side Rendering       │                        │ frontend/backend    │
│                              │                        │                     │
│  Scripting, Automation,      │ Python, Go             │ Simple syntax,      │
│  DevOps Tools                │                        │ easy deployment     │
│                              │                        │                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Real-World Example - E-Commerce Platform:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   E-COMMERCE POLYGLOT ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        SERVICE BREAKDOWN                             │    │
│  ├──────────────────┬─────────────┬─────────────────────────────────────┤    │
│  │ Service          │ Language    │ Reason                              │    │
│  ├──────────────────┼─────────────┼─────────────────────────────────────┤    │
│  │ User Service     │ Java/Spring │ Complex auth, enterprise patterns   │    │
│  │ Product Catalog  │ Node.js     │ High read, JSON-heavy, fast dev     │    │
│  │ Search Service   │ Rust        │ High performance text search        │    │
│  │ Order Service    │ Java/Spring │ Complex transactions, workflows     │    │
│  │ Payment Service  │ Go          │ High throughput, low latency        │    │
│  │ Recommendation   │ Python      │ ML models (TensorFlow/PyTorch)      │    │
│  │ Notification     │ Node.js     │ Async I/O, webhooks, simple logic   │    │
│  │ Analytics        │ Python      │ Data processing, Pandas, reporting  │    │
│  │ Real-time Feed   │ Elixir      │ WebSockets, millions of connections │    │
│  │ Image Processing │ Go          │ Concurrent image resizing           │    │
│  └──────────────────┴─────────────┴─────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 9.2 Polyglot Persistence (Multiple Databases)

**Choose the right database for each service's data model:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATABASE SELECTION BY DATA TYPE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DATA TYPE / USE CASE        │ DATABASE TYPE        │ EXAMPLES             │
│  ────────────────────────────┼──────────────────────┼───────────────────── │
│                              │                      │                      │
│  Structured, Transactions,   │ Relational (RDBMS)   │ PostgreSQL, MySQL,   │
│  ACID, Complex Queries       │                      │ Oracle, SQL Server   │
│                              │                      │                      │
│  Flexible Schema, JSON,      │ Document Store       │ MongoDB, CouchDB,    │
│  Hierarchical Data           │                      │ Amazon DocumentDB    │
│                              │                      │                      │
│  High Write Throughput,      │ Wide-Column          │ Cassandra, ScyllaDB, │
│  Time Series, IoT Data       │                      │ HBase, DynamoDB      │
│                              │                      │                      │
│  Caching, Sessions,          │ Key-Value / Cache    │ Redis, Memcached,    │
│  Real-time Counters          │                      │ etcd                 │
│                              │                      │                      │
│  Full-Text Search,           │ Search Engine        │ Elasticsearch,       │
│  Log Analytics               │                      │ OpenSearch, Solr     │
│                              │                      │                      │
│  Social Networks, Fraud      │ Graph Database       │ Neo4j, Amazon        │
│  Detection, Recommendations  │                      │ Neptune, JanusGraph  │
│                              │                      │                      │
│  Analytics, OLAP,            │ Columnar / OLAP      │ ClickHouse, Redshift,│
│  Data Warehousing            │                      │ BigQuery, Snowflake  │
│                              │                      │                      │
│  Time Series, Metrics,       │ Time Series DB       │ InfluxDB, TimescaleDB│
│  Monitoring Data             │                      │ Prometheus           │
│                              │                      │                      │
│  Event Sourcing,             │ Event Store          │ EventStoreDB,        │
│  Audit Logs                  │                      │ Kafka (as store)     │
│                              │                      │                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Polyglot Persistence Example:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    E-COMMERCE DATA ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────┐                                                       │
│  │   User Service   │──────► PostgreSQL (RDBMS)                             │
│  │                  │        • User profiles, roles, permissions            │
│  │                  │        • ACID transactions for auth                   │
│  └──────────────────┘                                                       │
│                                                                             │
│  ┌──────────────────┐                                                       │
│  │ Product Catalog  │──────► MongoDB (Document)                             │
│  │                  │        • Flexible product attributes                  │
│  │                  │        • Nested categories, variants                  │
│  └──────────────────┘                                                       │
│                                                                             │
│  ┌──────────────────┐                                                       │
│  │  Search Service  │──────► Elasticsearch                                  │
│  │                  │        • Full-text search                             │
│  │                  │        • Faceted filtering                            │
│  └──────────────────┘                                                       │
│                                                                             │
│  ┌──────────────────┐                                                       │
│  │  Order Service   │──────► PostgreSQL (RDBMS)                             │
│  │                  │        • Order transactions                           │
│  │                  │        • ACID guarantees                              │
│  └──────────────────┘                                                       │
│                                                                             │
│  ┌──────────────────┐                                                       │
│  │   Cart Service   │──────► Redis (Key-Value)                              │
│  │                  │        • Session-based carts                          │
│  │                  │        • Fast reads/writes, TTL                       │
│  └──────────────────┘                                                       │
│                                                                             │
│  ┌──────────────────┐                                                       │
│  │ Recommendation   │──────► Neo4j (Graph)                                  │
│  │                  │        • User-product relationships                   │
│  │                  │        • "Users who bought X also bought Y"           │
│  └──────────────────┘                                                       │
│                                                                             │
│  ┌──────────────────┐                                                       │
│  │    Analytics     │──────► ClickHouse (Columnar)                          │
│  │                  │        • Sales reports, aggregations                  │
│  │                  │        • Fast OLAP queries                            │
│  └──────────────────┘                                                       │
│                                                                             │
│  ┌──────────────────┐                                                       │
│  │  Activity Feed   │──────► Cassandra (Wide-Column)                        │
│  │                  │        • High write throughput                        │
│  │                  │        • Time-partitioned data                        │
│  └──────────────────┘                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 9.3 Benefits of Polyglot Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           BENEFITS                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ✅ RIGHT TOOL FOR THE JOB                                                  │
│     • ML service in Python (TensorFlow, PyTorch)                            │
│     • High-performance service in Go/Rust                                   │
│     • Rapid development in Node.js                                          │
│                                                                             │
│  ✅ TEAM AUTONOMY                                                           │
│     • Teams choose best tech for their domain                               │
│     • Hire specialists for specific technologies                            │
│     • No "one size fits all" constraints                                    │
│                                                                             │
│  ✅ OPTIMIZED PERFORMANCE                                                   │
│     • Each service tuned for its workload                                   │
│     • Right database for data access patterns                               │
│     • No compromise on critical paths                                       │
│                                                                             │
│  ✅ RISK MITIGATION                                                         │
│     • Not locked into single vendor/technology                              │
│     • Can replace individual services                                       │
│     • Gradual technology adoption                                           │
│                                                                             │
│  ✅ INNOVATION                                                              │
│     • Easy to try new technologies in isolated services                     │
│     • A/B test different implementations                                    │
│     • Stay current with industry trends                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 9.4 Challenges & Solutions

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CHALLENGES & SOLUTIONS                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ❌ CHALLENGE: Operational Complexity                                       │
│  ─────────────────────────────────────                                      │
│  Different languages = different deployment, monitoring, debugging          │
│                                                                             │
│  ✅ SOLUTION:                                                               │
│  • Containerization (Docker) - uniform deployment                           │
│  • Kubernetes - orchestration regardless of language                        │
│  • Standardized observability (OpenTelemetry)                               │
│  • Common CI/CD pipelines with language-specific stages                     │
│                                                                             │
│  ──────────────────────────────────────────────────────────────────────     │
│                                                                             │
│  ❌ CHALLENGE: Cross-Team Knowledge                                         │
│  ──────────────────────────────────                                         │
│  Hard to move between teams, code reviews across languages                  │
│                                                                             │
│  ✅ SOLUTION:                                                               │
│  • Limit to 3-4 core languages                                              │
│  • Shared coding standards per language                                     │
│  • Internal training/documentation                                          │
│  • API contracts (OpenAPI, Protobuf) as common interface                    │
│                                                                             │
│  ──────────────────────────────────────────────────────────────────────     │
│                                                                             │
│  ❌ CHALLENGE: Data Consistency Across DBs                                  │
│  ─────────────────────────────────────────                                  │
│  No cross-database transactions                                             │
│                                                                             │
│  ✅ SOLUTION:                                                               │
│  • Saga pattern for distributed transactions                                │
│  • Event-driven architecture                                                │
│  • Eventual consistency acceptance                                          │
│  • CDC (Change Data Capture) for data sync                                  │
│                                                                             │
│  ──────────────────────────────────────────────────────────────────────     │
│                                                                             │
│  ❌ CHALLENGE: Testing & Debugging                                          │
│  ───────────────────────────────────                                        │
│  Different test frameworks, debugging tools per language                    │
│                                                                             │
│  ✅ SOLUTION:                                                               │
│  • Contract testing (Pact) - language agnostic                              │
│  • End-to-end tests at API level                                            │
│  • Distributed tracing (Jaeger, Zipkin)                                     │
│  • Centralized logging (ELK, Loki)                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 9.5 Polyglot Communication Strategies

**Services in different languages need common communication protocols:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CROSS-LANGUAGE COMMUNICATION                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  OPTION 1: REST + JSON (Most Common)                                        │
│  ───────────────────────────────────                                        │
│                                                                             │
│  Java ───── JSON/HTTP ─────► Node.js ───── JSON/HTTP ─────► Python          │
│                                                                             │
│  Pros: Universal support, human readable, easy debugging                    │
│  Cons: Verbose, slower serialization, no type safety                        │
│                                                                             │
│  ──────────────────────────────────────────────────────────────────────     │
│                                                                             │
│  OPTION 2: gRPC + Protocol Buffers (High Performance)                       │
│  ────────────────────────────────────────────────────                       │
│                                                                             │
│  Java ───── Protobuf/HTTP2 ─────► Go ───── Protobuf/HTTP2 ─────► Rust       │
│                                                                             │
│  Pros: Fast, type-safe, code generation for all languages                   │
│  Cons: Binary (harder to debug), steeper learning curve                     │
│                                                                             │
│  ──────────────────────────────────────────────────────────────────────     │
│                                                                             │
│  OPTION 3: Message Queues (Async)                                           │
│  ────────────────────────────────                                           │
│                                                                             │
│  Java ───► Kafka ◄─── Python                                                │
│              │                                                              │
│              └────► Node.js                                                 │
│                                                                             │
│  Pros: Decoupled, language agnostic, reliable delivery                      │
│  Cons: Eventual consistency, more infrastructure                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Protocol Buffers Example (Language Agnostic Contract):**

```protobuf
// order.proto - Shared contract across all languages
syntax = "proto3";

package ecommerce;

option java_package = "com.example.order";
option go_package = "github.com/example/order";

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (stream Order);
}

message Order {
  string id = 1;
  string user_id = 2;
  repeated OrderItem items = 3;
  OrderStatus status = 4;
  Money total = 5;
  google.protobuf.Timestamp created_at = 6;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  Money price = 3;
}

message Money {
  string currency = 1;
  int64 amount = 2;  // In cents
}

enum OrderStatus {
  PENDING = 0;
  CONFIRMED = 1;
  SHIPPED = 2;
  DELIVERED = 3;
  CANCELLED = 4;
}
```

**Generated Code Usage:**

```java
// Java (generated from proto)
Order order = orderServiceStub.getOrder(
    GetOrderRequest.newBuilder()
        .setOrderId("ord-123")
        .build()
);
```

```go
// Go (generated from proto)
order, err := orderClient.GetOrder(ctx, &pb.GetOrderRequest{
    OrderId: "ord-123",
})
```

```python
# Python (generated from proto)
order = order_service_stub.GetOrder(
    order_pb2.GetOrderRequest(order_id="ord-123")
)
```

---

### 9.6 Polyglot Best Practices

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BEST PRACTICES                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1️⃣  LIMIT LANGUAGE SPRAWL                                                  │
│      • Stick to 3-4 primary languages                                       │
│      • Require justification for new languages                              │
│      • Example: Java (enterprise), Go (performance), Python (ML),           │
│                 Node.js (rapid dev)                                         │
│                                                                             │
│  2️⃣  STANDARDIZE INFRASTRUCTURE                                             │
│      • Docker for all services (language-agnostic)                          │
│      • Kubernetes for orchestration                                         │
│      • Same CI/CD pipeline structure                                        │
│      • Common logging format (JSON structured logs)                         │
│                                                                             │
│  3️⃣  UNIFIED OBSERVABILITY                                                  │
│      • OpenTelemetry (works with all languages)                             │
│      • Common metrics format (Prometheus)                                   │
│      • Centralized logging (ELK/Loki)                                       │
│      • Distributed tracing (Jaeger)                                         │
│                                                                             │
│  4️⃣  CONTRACT-FIRST DEVELOPMENT                                             │
│      • OpenAPI/Swagger for REST                                             │
│      • Protocol Buffers for gRPC                                            │
│      • AsyncAPI for event-driven                                            │
│      • Schema registry for Kafka                                            │
│                                                                             │
│  5️⃣  CHOOSE DATABASES WISELY                                                │
│      • Start with fewer databases                                           │
│      • PostgreSQL can handle many use cases                                 │
│      • Add specialized DBs only when needed                                 │
│      • Consider operational burden                                          │
│                                                                             │
│  6️⃣  SHARED LIBRARIES (CAREFULLY)                                           │
│      • Cross-language SDKs for common operations                            │
│      • Auth/security libraries per language                                 │
│      • Avoid tight coupling through shared code                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 9.7 Polyglot Decision Matrix

| Scenario | Recommendation |
|----------|----------------|
| Startup / Small Team | Monoglot (1-2 languages) - minimize complexity |
| ML/AI Features | Python service + main language for rest |
| High-Performance Critical Path | Go/Rust for that specific service |
| Rapid Feature Development | Node.js/Python for new services |
| Enterprise / Regulated | Java/C# with strong typing |
| Real-time / WebSockets | Elixir/Go for that specific service |
| Legacy Integration | Same language as legacy OR anti-corruption layer |

**When NOT to Go Polyglot:**

| Avoid Polyglot If... | Reason |
|---------------------|--------|
| Small team (< 10 devs) | Operational overhead too high |
| Single problem domain | No real benefit |
| Tight deadlines | Learning curve slows delivery |
| Limited DevOps maturity | Need solid CI/CD first |
| No clear technical benefit | Complexity for complexity's sake |

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
| Different tech requirements | Polyglot Microservices |

---

## Additional Resources

- [microservices.io](https://microservices.io/patterns/index.html) - Comprehensive pattern catalog
- [Martin Fowler's Microservices Guide](https://martinfowler.com/microservices/)
- [Chris Richardson's Microservices Patterns Book](https://www.manning.com/books/microservices-patterns)

