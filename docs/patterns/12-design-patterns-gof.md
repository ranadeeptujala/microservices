# Classic Design Patterns (Gang of Four)

Understanding the difference between GoF Design Patterns and Microservice Patterns.

---

## Design Patterns vs Microservice Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              DESIGN PATTERNS vs MICROSERVICE PATTERNS                        │
│                        (DIFFERENT THINGS!)                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   DESIGN PATTERNS (GoF)                 MICROSERVICE PATTERNS               │
│   ─────────────────────                 ─────────────────────               │
│                                                                             │
│   • Object-Oriented patterns            • Distributed system patterns       │
│   • Code-level solutions                • Architecture-level solutions      │
│   • Single application                  • Multiple services                 │
│   • Gang of Four (1994 book)            • Chris Richardson, Sam Newman      │
│                                                                             │
│   Examples:                             Examples:                           │
│   • Singleton                           • API Gateway                       │
│   • Factory                             • Circuit Breaker                   │
│   • Observer                            • Saga                              │
│   • Strategy                            • CQRS                              │
│                                                                             │
│   Scope: CLASS / OBJECT                 Scope: SERVICE / SYSTEM             │
│                                                                             │
│   ─────────────────────────────────────────────────────────────────────     │
│                                                                             │
│   IMPORTANT: Microservices is NOT a pattern!                                │
│   Microservices is an ARCHITECTURAL STYLE / PRACTICE                        │
│   that USES various patterns (API Gateway, Saga, etc.)                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Gang of Four - 23 Design Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    23 GoF DESIGN PATTERNS                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   CREATIONAL (5)              STRUCTURAL (7)          BEHAVIORAL (11)       │
│   How objects created         How objects composed    How objects interact  │
│   ──────────────────          ────────────────────    ────────────────────  │
│                                                                             │
│   ★ Singleton                 ★ Adapter               ★ Observer            │
│   ★ Factory Method            ★ Decorator             ★ Strategy            │
│   ★ Builder                   ★ Facade                ★ Command             │
│     Abstract Factory            Proxy                   Template Method     │
│     Prototype                   Composite               Iterator            │
│                                 Bridge                  State               │
│                                 Flyweight               Chain of Resp       │
│                                                         Mediator            │
│                                                         Memento             │
│                                                         Visitor             │
│                                                         Interpreter         │
│                                                                             │
│   ★ = Most commonly asked in interviews                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Key GoF Patterns Quick Reference

| Pattern | Category | Purpose | Example |
|---------|----------|---------|---------|
| **Singleton** | Creational | One instance only | Database connection |
| **Factory** | Creational | Create without specifying class | Document creators |
| **Builder** | Creational | Step-by-step construction | StringBuilder |
| **Adapter** | Structural | Convert interface | Arrays.asList() |
| **Decorator** | Structural | Add behavior dynamically | Java I/O streams |
| **Facade** | Structural | Simplify complex system | JDBC DriverManager |
| **Observer** | Behavioral | Event notification | Event listeners |
| **Strategy** | Behavioral | Swap algorithms | Comparator |
| **Command** | Behavioral | Encapsulate request | Undo/Redo |

---

## Common Patterns Explained

### Singleton Pattern

**One instance only throughout the application.**

```java
public class DatabaseConnection {
    private static DatabaseConnection instance;
    
    private DatabaseConnection() {}
    
    public static synchronized DatabaseConnection getInstance() {
        if (instance == null) {
            instance = new DatabaseConnection();
        }
        return instance;
    }
}
```

### Factory Pattern

**Create objects without specifying exact class.**

```java
public interface PaymentProcessor {
    void process(Payment payment);
}

public class PaymentProcessorFactory {
    public static PaymentProcessor create(String type) {
        switch (type) {
            case "CREDIT_CARD": return new CreditCardProcessor();
            case "PAYPAL": return new PayPalProcessor();
            case "STRIPE": return new StripeProcessor();
            default: throw new IllegalArgumentException("Unknown type");
        }
    }
}

// Usage
PaymentProcessor processor = PaymentProcessorFactory.create("CREDIT_CARD");
processor.process(payment);
```

### Builder Pattern

**Step-by-step object construction.**

```java
Order order = Order.builder()
    .orderId("ORD-123")
    .customerId("CUST-456")
    .addItem("PROD-1", 2)
    .addItem("PROD-2", 1)
    .shippingAddress(address)
    .build();
```

### Strategy Pattern

**Swap algorithms at runtime.**

```java
public interface SortStrategy {
    void sort(List<Integer> list);
}

public class QuickSort implements SortStrategy { /* ... */ }
public class MergeSort implements SortStrategy { /* ... */ }

// Usage
List<Integer> data = Arrays.asList(5, 2, 8, 1);
SortStrategy strategy = new QuickSort();
strategy.sort(data);
```

### Observer Pattern

**Notify multiple objects of state changes.**

```java
public interface OrderObserver {
    void onOrderPlaced(Order order);
}

public class OrderService {
    private List<OrderObserver> observers = new ArrayList<>();
    
    public void addObserver(OrderObserver observer) {
        observers.add(observer);
    }
    
    public void placeOrder(Order order) {
        // Save order
        orderRepository.save(order);
        
        // Notify all observers
        observers.forEach(o -> o.onOrderPlaced(order));
    }
}

// Register observers
orderService.addObserver(new EmailNotificationObserver());
orderService.addObserver(new InventoryObserver());
orderService.addObserver(new AnalyticsObserver());
```

---

## Front Controller Pattern

**Enterprise pattern** providing single entry point for web applications.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRONT CONTROLLER                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌────────┐                ┌──────────────────┐     ┌─────────────┐       │
│   │        │   ALL requests │                  │     │  Controller │       │
│   │ Client │───────────────►│ FRONT CONTROLLER │────►│  Controller │       │
│   │        │                │   (Dispatcher)   │────►│  Controller │       │
│   └────────┘                │                  │     └─────────────┘       │
│                             │ • Auth           │                           │
│                             │ • Logging        │     Spring MVC:           │
│                             │ • Route          │     DispatcherServlet     │
│                             └──────────────────┘     is Front Controller!  │
│                                                                             │
│   Front Controller (app level) ≈ API Gateway (system level)                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

```java
// Spring MVC uses Front Controller pattern
// DispatcherServlet is the Front Controller

@Controller
public class UserController {
    
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

---

## Command Pattern

**Encapsulate a request as an object.**

```java
public interface Command {
    void execute();
    void undo();
}

public class AddToCartCommand implements Command {
    private Cart cart;
    private Product product;
    
    @Override
    public void execute() {
        cart.add(product);
    }
    
    @Override
    public void undo() {
        cart.remove(product);
    }
}

// Usage with command history
public class CommandInvoker {
    private Stack<Command> history = new Stack<>();
    
    public void execute(Command command) {
        command.execute();
        history.push(command);
    }
    
    public void undo() {
        if (!history.isEmpty()) {
            history.pop().undo();
        }
    }
}
```

---

## Pattern Categories Summary

| Category | Level | Examples |
|----------|-------|----------|
| **Architectural Style** | System | Microservices, Monolith, Serverless |
| **Microservice Patterns** | Service/System | API Gateway, Saga, CQRS, Circuit Breaker |
| **Enterprise Patterns** | Application | Repository, Unit of Work, Front Controller |
| **Design Patterns (GoF)** | Class/Object | Singleton, Factory, Observer, Strategy |

---

[← Back to Index](./README.md) | [Next: Interview Tips →](./13-interview-tips.md)

