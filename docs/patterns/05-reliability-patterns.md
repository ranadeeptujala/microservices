# Reliability Patterns

Patterns for building fault-tolerant microservices that gracefully handle failures.

---

## Circuit Breaker

**Prevents cascading failures to UPSTREAM systems by failing fast when downstream service is down.**

### Why Circuit Breaker? Cascading Failure Problem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WITHOUT CIRCUIT BREAKER - CASCADING FAILURE               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   UPSTREAM                                              DOWNSTREAM          │
│   (callers)                                             (called)            │
│                                                                             │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐       │
│   │  Mobile  │      │  Order   │      │ Payment  │      │ External │       │
│   │   App    │─────►│ Service  │─────►│ Service  │─────►│ Bank API │       │
│   └──────────┘      └──────────┘      └──────────┘      └──────────┘       │
│                           │                │                  │             │
│                           │                │                  │             │
│                           │                │            ❌ Bank API DOWN!   │
│                           │                │                  │             │
│                           │                │◄─── Waiting... (timeout 30s)   │
│                           │                │     Threads blocked            │
│                           │                │     Connections exhausted      │
│                           │                │                                │
│                           │◄─── Payment not responding                      │
│                           │     Order Service threads blocked               │
│                           │     Connection pool exhausted                   │
│                           │                                                 │
│   Mobile App ◄─── Order Service not responding                              │
│   shows errors     ENTIRE SYSTEM DOWN!                                      │
│                                                                             │
│   🔥 One downstream failure → ALL upstream services fail (CASCADING!)       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    WITH CIRCUIT BREAKER - FAILURE ISOLATED                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐       │
│   │  Mobile  │      │  Order   │      │ Payment  │      │ External │       │
│   │   App    │─────►│ Service  │─────►│ Service  │──X──►│ Bank API │       │
│   └──────────┘      └──────────┘      └──────────┘      └──────────┘       │
│        │                 │                 │                  │             │
│        │                 │                 │            ❌ Bank API DOWN!   │
│        │                 │                 │                                │
│        │                 │           ┌─────┴─────┐                          │
│        │                 │           │ CIRCUIT   │                          │
│        │                 │           │ BREAKER   │ ← Detects failures       │
│        │                 │           │  (OPEN)   │   Opens circuit          │
│        │                 │           └─────┬─────┘   Fails FAST!            │
│        │                 │                 │                                │
│        │                 │◄─── Immediate fallback response (10ms)           │
│        │                 │     "Payment pending, will retry later"          │
│        │                 │     Threads NOT blocked!                         │
│        │                 │                                                  │
│   ✅ Mobile App works    │                                                  │
│      Shows: "Order       │◄─── Order Service healthy                        │
│      placed, payment     │     Still serving other requests                 │
│      processing..."      │                                                  │
│                                                                             │
│   ✅ Failure ISOLATED - Upstream services protected!                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Benefits

| Without Circuit Breaker | With Circuit Breaker |
|------------------------|---------------------|
| ❌ Threads blocked waiting | ✅ Fail fast (immediate response) |
| ❌ Connection pool exhausted | ✅ Connections preserved |
| ❌ Cascades to ALL upstream | ✅ Failure isolated |
| ❌ Entire system down | ✅ Other features still work |
| ❌ Slow degradation | ✅ Graceful degradation |

---

### Circuit Breaker + DLQ (Don't Lose Business!)

**Problem**: Circuit breaker fails fast, but you don't want to LOSE those transactions!

**Solution**: Save failed requests to Dead Letter Queue (DLQ) → Replay when service recovers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CIRCUIT BREAKER + DLQ PATTERN                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────┐      ┌──────────────────────────────────────────────────┐   │
│   │  Order   │      │              PAYMENT SERVICE                      │   │
│   │ Service  │─────►│  ┌────────────────┐                               │   │
│   └──────────┘      │  │ Circuit Breaker│                               │   │
│        │            │  │    (OPEN)      │                               │   │
│        │            │  └───────┬────────┘                               │   │
│        │            │          │                                        │   │
│        │            │          │ Downstream DOWN!                       │   │
│        │            │          ▼                                        │   │
│        │            │  ┌────────────────┐      ┌─────────────────┐      │   │
│        │            │  │   FALLBACK     │─────►│      DLQ        │      │   │
│        │            │  │   (save to DLQ)│      │ (Dead Letter Q) │      │   │
│        │            │  └────────────────┘      │                 │      │   │
│        │            │                          │ payment_dlq:    │      │   │
│        │            │                          │ - order_123     │      │   │
│        │            │                          │ - order_124     │      │   │
│        │            │                          │ - order_125     │      │   │
│        │            │                          └────────┬────────┘      │   │
│        │            └──────────────────────────────────┼────────────────┘   │
│        │                                               │                    │
│        │                                               │                    │
│        ▼                                               ▼                    │
│   ┌──────────────────────────────┐      ┌──────────────────────────────┐   │
│   │  Response to User:           │      │  REPLAY SERVICE              │   │
│   │  "Order placed!              │      │  (Scheduled Job / Consumer)  │   │
│   │   Payment processing..."     │      │                              │   │
│   │                              │      │  • Checks circuit state      │   │
│   │  ✅ User not blocked         │      │  • When CLOSED → replay DLQ  │   │
│   │  ✅ Business NOT lost        │      │  • Retry failed payments     │   │
│   └──────────────────────────────┘      └──────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Implementation Example

```java
@Service
@Slf4j
public class PaymentService {

    private final PaymentClient paymentClient;
    private final SqsTemplate sqsTemplate;  // or KafkaTemplate
    private final CircuitBreakerRegistry circuitBreakerRegistry;

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        return paymentClient.charge(request);
    }

    // Fallback: Save to DLQ instead of losing the transaction!
    public PaymentResponse paymentFallback(PaymentRequest request, Exception ex) {
        log.warn("Payment failed for order {}, saving to DLQ", request.getOrderId());
        
        // Save to Dead Letter Queue for later replay
        DlqMessage dlqMessage = DlqMessage.builder()
            .orderId(request.getOrderId())
            .payload(request)
            .failedAt(Instant.now())
            .errorMessage(ex.getMessage())
            .retryCount(0)
            .build();
        
        sqsTemplate.send("payment-dlq", dlqMessage);
        
        // Return pending status to user (not error!)
        return PaymentResponse.builder()
            .orderId(request.getOrderId())
            .status(PaymentStatus.PENDING)
            .message("Payment queued for processing")
            .build();
    }
}
```

#### DLQ Replay Service

```java
@Service
@Slf4j
public class PaymentDlqReplayService {

    private final PaymentClient paymentClient;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    // Run every 5 minutes
    @Scheduled(fixedRate = 300000)
    public void replayFailedPayments() {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        
        // Only replay when circuit is CLOSED (service recovered)
        if (cb.getState() == CircuitBreaker.State.CLOSED) {
            log.info("Circuit CLOSED - replaying DLQ messages");
            
            List<DlqMessage> messages = sqsTemplate.receiveMany("payment-dlq", 10);
            
            for (DlqMessage msg : messages) {
                try {
                    PaymentResponse response = paymentClient.charge(msg.getPayload());
                    log.info("Successfully replayed payment for order {}", msg.getOrderId());
                    
                    // Notify user: "Your payment has been processed!"
                    notificationService.sendPaymentSuccess(msg.getOrderId());
                    
                } catch (Exception e) {
                    log.error("Replay failed for order {}, retry count: {}", 
                        msg.getOrderId(), msg.getRetryCount());
                    
                    if (msg.getRetryCount() < MAX_RETRIES) {
                        // Re-queue with incremented retry count
                        msg.setRetryCount(msg.getRetryCount() + 1);
                        sqsTemplate.send("payment-dlq", msg);
                    } else {
                        // Max retries exceeded - move to permanent failure queue
                        sqsTemplate.send("payment-failed-permanent", msg);
                        notificationService.alertOpsTeam(msg);
                    }
                }
            }
        } else {
            log.info("Circuit still OPEN - skipping DLQ replay");
        }
    }
}
```

#### Complete Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         COMPLETE BUSINESS FLOW                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. USER PLACES ORDER                                                      │
│      └─► Order Service calls Payment Service                                │
│                                                                             │
│   2. CIRCUIT BREAKER OPEN (Bank API down)                                   │
│      └─► Fallback triggered                                                 │
│      └─► Payment saved to DLQ                                               │
│      └─► User sees: "Order placed! Payment processing..."                   │
│      └─► ✅ Business NOT lost!                                              │
│                                                                             │
│   3. BANK API RECOVERS                                                      │
│      └─► Circuit Breaker → HALF-OPEN → CLOSED                               │
│                                                                             │
│   4. DLQ REPLAY SERVICE                                                     │
│      └─► Detects circuit CLOSED                                             │
│      └─► Replays all pending payments from DLQ                              │
│      └─► Retries payment for order_123, order_124, order_125...             │
│                                                                             │
│   5. USER NOTIFIED                                                          │
│      └─► "Your payment for Order #123 has been processed!"                  │
│      └─► ✅ Customer happy, business completed!                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### DLQ Options

| Queue | Best For |
|-------|----------|
| **AWS SQS DLQ** | Cloud-native, managed |
| **Kafka DLQ Topic** | High throughput, replay capability |
| **RabbitMQ DLQ** | Traditional message queues |
| **Database Table** | Simple, queryable (orders with status=PENDING) |

---

### RabbitMQ DLQ Example

#### 1. RabbitMQ Configuration

```java
@Configuration
public class RabbitMQConfig {

    // Main queue for payments
    @Bean
    public Queue paymentQueue() {
        return QueueBuilder.durable("payment-queue")
            .withArgument("x-dead-letter-exchange", "payment-dlx")  // DLX exchange
            .withArgument("x-dead-letter-routing-key", "payment-dlq")
            .build();
    }

    // Dead Letter Exchange (DLX)
    @Bean
    public DirectExchange deadLetterExchange() {
        return new DirectExchange("payment-dlx");
    }

    // Dead Letter Queue (DLQ)
    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable("payment-dlq").build();
    }

    // Bind DLQ to DLX
    @Bean
    public Binding dlqBinding() {
        return BindingBuilder
            .bind(deadLetterQueue())
            .to(deadLetterExchange())
            .with("payment-dlq");
    }
}
```

#### 2. Circuit Breaker Fallback → Send to RabbitMQ DLQ

```java
@Service
@Slf4j
public class PaymentService {

    private final RabbitTemplate rabbitTemplate;
    private final PaymentClient paymentClient;

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        return paymentClient.charge(request);
    }

    // Fallback: Send to RabbitMQ DLQ
    public PaymentResponse paymentFallback(PaymentRequest request, Exception ex) {
        log.warn("Circuit OPEN - sending to DLQ: order={}", request.getOrderId());

        PaymentDlqMessage dlqMessage = PaymentDlqMessage.builder()
            .orderId(request.getOrderId())
            .payload(request)
            .failedAt(Instant.now())
            .errorMessage(ex.getMessage())
            .retryCount(0)
            .build();

        // Send directly to DLQ (not main queue)
        rabbitTemplate.convertAndSend("payment-dlx", "payment-dlq", dlqMessage);

        return PaymentResponse.builder()
            .orderId(request.getOrderId())
            .status(PaymentStatus.PENDING)
            .message("Payment queued, will process shortly")
            .build();
    }
}
```

#### 3. DLQ Consumer - Replay When Circuit Closed

```java
@Service
@Slf4j
public class PaymentDlqConsumer {

    private final PaymentClient paymentClient;
    private final RabbitTemplate rabbitTemplate;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    private static final int MAX_RETRIES = 3;

    @RabbitListener(queues = "payment-dlq")
    public void processFailedPayment(PaymentDlqMessage message, Channel channel, 
                                      @Header(AmqpHeaders.DELIVERY_TAG) long tag) 
            throws IOException {
        
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        
        // Check if circuit is CLOSED (service recovered)
        if (cb.getState() == CircuitBreaker.State.CLOSED) {
            try {
                log.info("Replaying payment for order: {}", message.getOrderId());
                
                PaymentResponse response = paymentClient.charge(message.getPayload());
                
                log.info("Payment successful for order: {}", message.getOrderId());
                channel.basicAck(tag, false);  // Remove from DLQ
                
                // Notify user
                notificationService.sendPaymentSuccess(message.getOrderId());
                
            } catch (Exception e) {
                handleRetry(message, channel, tag, e);
            }
        } else {
            // Circuit still OPEN - requeue and wait
            log.info("Circuit still OPEN, requeueing order: {}", message.getOrderId());
            channel.basicNack(tag, false, true);  // Requeue
        }
    }
    
    private void handleRetry(PaymentDlqMessage message, Channel channel, 
                             long tag, Exception e) throws IOException {
        if (message.getRetryCount() < MAX_RETRIES) {
            // Increment retry and requeue
            message.setRetryCount(message.getRetryCount() + 1);
            log.warn("Retry {} for order: {}", message.getRetryCount(), message.getOrderId());
            
            // Requeue with delay (using TTL or delayed exchange)
            channel.basicNack(tag, false, true);
        } else {
            // Max retries exceeded - permanent failure
            log.error("Max retries exceeded for order: {}", message.getOrderId());
            channel.basicAck(tag, false);  // Remove from DLQ
            
            // Move to permanent failure queue
            rabbitTemplate.convertAndSend("payment-failed-permanent", message);
            notificationService.alertOpsTeam(message);
        }
    }
}
```

#### 4. RabbitMQ DLQ Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RABBITMQ DLQ FLOW                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────┐                                                          │
│   │ Payment Req  │                                                          │
│   └──────┬───────┘                                                          │
│          │                                                                  │
│          ▼                                                                  │
│   ┌──────────────────┐                                                      │
│   │ Circuit Breaker  │                                                      │
│   │    (OPEN)        │                                                      │
│   └──────┬───────────┘                                                      │
│          │ Fallback                                                         │
│          ▼                                                                  │
│   ┌──────────────────┐      ┌──────────────────┐                           │
│   │    payment-dlx   │─────►│   payment-dlq    │                           │
│   │  (Dead Letter    │      │  (Dead Letter    │                           │
│   │   Exchange)      │      │   Queue)         │                           │
│   └──────────────────┘      └────────┬─────────┘                           │
│                                      │                                      │
│                                      │ Consumer listens                     │
│                                      ▼                                      │
│                             ┌──────────────────┐                           │
│                             │ DLQ Consumer     │                           │
│                             │                  │                           │
│                             │ if circuit CLOSED│                           │
│                             │   → retry payment│                           │
│                             │   → ack message  │                           │
│                             │                  │                           │
│                             │ if circuit OPEN  │                           │
│                             │   → nack/requeue │                           │
│                             └──────────────────┘                           │
│                                                                             │
│   User Response: "Payment queued, will process shortly" ✅                  │
│   Later: "Your payment has been processed!" ✅                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 5. application.yml

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    listener:
      simple:
        acknowledge-mode: manual  # Manual ack for DLQ control
        retry:
          enabled: false  # We handle retries manually
```

#### RabbitMQ vs Other DLQ Options

| Feature | RabbitMQ DLQ | Kafka DLQ | SQS DLQ |
|---------|--------------|-----------|---------|
| **Built-in DLQ** | ✅ Yes (DLX) | ❌ Manual topic | ✅ Yes |
| **Message TTL** | ✅ Yes | ❌ No | ✅ Yes (14 days max) |
| **Delayed Retry** | ✅ Plugin available | ✅ Pause consumer | ✅ Visibility timeout |
| **Ordering** | ❌ No guarantee | ✅ Per partition | ❌ No (FIFO extra) |
| **Best For** | Traditional apps | Event streaming | AWS cloud-native |

---

### Idempotency - Prevent Duplicate Transactions! 🔑

**Problem**: Retry/DLQ can process same request MULTIPLE times → Duplicate charges!

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DUPLICATE TRANSACTION PROBLEM                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   WITHOUT IDEMPOTENCY:                                                      │
│   ────────────────────                                                      │
│                                                                             │
│   Order-123 Payment Request                                                 │
│        │                                                                    │
│        ▼                                                                    │
│   ┌──────────────┐                                                          │
│   │ Payment API  │ → Charge $100 ✅                                         │
│   └──────────────┘                                                          │
│        │                                                                    │
│        │ Network timeout (response lost)                                    │
│        │ Caller thinks it failed!                                           │
│        ▼                                                                    │
│   ┌──────────────┐                                                          │
│   │   RETRY      │ → Charge $100 AGAIN! ❌                                  │
│   └──────────────┘                                                          │
│                                                                             │
│   Customer charged $200 instead of $100! 💸                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Solution: Idempotency Key

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WITH IDEMPOTENCY KEY                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Order-123 Payment Request                                                 │
│   Idempotency-Key: "PAY-ORDER-123-UUID-abc"                                │
│        │                                                                    │
│        ▼                                                                    │
│   ┌──────────────────────────────────────────────────────────┐             │
│   │ Payment Service                                          │             │
│   │                                                          │             │
│   │  1. Check: Is "PAY-ORDER-123-UUID-abc" in Redis/DB?     │             │
│   │     → NO: Process payment, save key                      │             │
│   │     → YES: Return cached response (don't charge again!)  │             │
│   │                                                          │             │
│   └──────────────────────────────────────────────────────────┘             │
│        │                                                                    │
│        │ First call: Process & charge $100 ✅                              │
│        │ Save to Redis: "PAY-ORDER-123-UUID-abc" → {status: SUCCESS}       │
│        │                                                                    │
│        │ Network timeout (response lost)                                    │
│        ▼                                                                    │
│   ┌──────────────┐                                                          │
│   │   RETRY      │ Same Idempotency-Key                                    │
│   │              │ → Key EXISTS in Redis!                                  │
│   │              │ → Return cached response                                │
│   │              │ → NO duplicate charge! ✅                               │
│   └──────────────┘                                                          │
│                                                                             │
│   Customer charged exactly $100 ✅                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Implementation

**1. Idempotency Key Storage (Redis)**

```java
@Service
@Slf4j
public class IdempotencyService {

    private final RedisTemplate<String, String> redisTemplate;
    private static final Duration TTL = Duration.ofHours(24);  // Keep for 24h

    // Check if already processed
    public boolean isProcessed(String idempotencyKey) {
        return Boolean.TRUE.equals(redisTemplate.hasKey(idempotencyKey));
    }

    // Get cached response
    public Optional<String> getCachedResponse(String idempotencyKey) {
        String cached = redisTemplate.opsForValue().get(idempotencyKey);
        return Optional.ofNullable(cached);
    }

    // Save response after processing
    public void saveResponse(String idempotencyKey, String response) {
        redisTemplate.opsForValue().set(idempotencyKey, response, TTL);
    }

    // Atomic check-and-set (prevent race condition)
    public boolean tryAcquireLock(String idempotencyKey) {
        return Boolean.TRUE.equals(
            redisTemplate.opsForValue().setIfAbsent(
                idempotencyKey + ":lock", 
                "processing", 
                Duration.ofMinutes(5)
            )
        );
    }
}
```

**2. Payment Service with Idempotency**

```java
@Service
@Slf4j
public class PaymentService {

    private final PaymentClient paymentClient;
    private final IdempotencyService idempotencyService;
    private final ObjectMapper objectMapper;

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        
        // Generate idempotency key from order ID
        String idempotencyKey = "payment:" + request.getOrderId();
        
        // 1. Check if already processed
        Optional<String> cached = idempotencyService.getCachedResponse(idempotencyKey);
        if (cached.isPresent()) {
            log.info("Returning cached response for order: {}", request.getOrderId());
            return objectMapper.readValue(cached.get(), PaymentResponse.class);
        }
        
        // 2. Try to acquire lock (prevent concurrent duplicates)
        if (!idempotencyService.tryAcquireLock(idempotencyKey)) {
            log.warn("Payment already in progress for order: {}", request.getOrderId());
            throw new PaymentInProgressException("Payment being processed");
        }
        
        try {
            // 3. Process payment
            PaymentResponse response = paymentClient.charge(request);
            
            // 4. Cache the response
            idempotencyService.saveResponse(idempotencyKey, 
                objectMapper.writeValueAsString(response));
            
            return response;
            
        } catch (Exception e) {
            // Don't cache failures - allow retry
            throw e;
        }
    }
}
```

**3. DLQ Consumer with Idempotency**

```java
@RabbitListener(queues = "payment-dlq")
public void processFailedPayment(PaymentDlqMessage message) {
    
    String idempotencyKey = "payment:" + message.getOrderId();
    
    // Skip if already processed successfully
    if (idempotencyService.isProcessed(idempotencyKey)) {
        log.info("Order {} already processed, skipping", message.getOrderId());
        return;  // Ack message, don't reprocess
    }
    
    // Process payment (with idempotency check inside)
    paymentService.processPayment(message.getPayload());
}
```

**4. Database-Level Idempotency (Alternative)**

```sql
-- Payments table with unique constraint
CREATE TABLE payments (
    id VARCHAR(50) PRIMARY KEY,
    order_id VARCHAR(50) NOT NULL,
    idempotency_key VARCHAR(100) UNIQUE,  -- Prevents duplicates!
    amount DECIMAL(10,2),
    status VARCHAR(20),
    created_at TIMESTAMP
);

-- Insert with idempotency
INSERT INTO payments (id, order_id, idempotency_key, amount, status)
VALUES ('pay-123', 'order-456', 'PAY-ORDER-456', 100.00, 'SUCCESS')
ON CONFLICT (idempotency_key) DO NOTHING;  -- PostgreSQL
-- or
-- ON DUPLICATE KEY UPDATE id=id;  -- MySQL (no-op)
```

#### Idempotency Key Strategies

| Strategy | Key Format | Best For |
|----------|------------|----------|
| **Order-based** | `payment:{orderId}` | E-commerce |
| **Request-based** | `{userId}:{action}:{timestamp}` | General APIs |
| **Client-generated** | UUID from client | Stripe-style APIs |
| **Hash-based** | `hash(request body)` | Duplicate detection |

#### Complete Flow with Idempotency

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              RETRY + DLQ + IDEMPOTENCY = SAFE TRANSACTIONS                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Order-123 Payment                                                         │
│        │                                                                    │
│        ▼                                                                    │
│   ┌──────────────────┐                                                      │
│   │ Circuit Breaker  │                                                      │
│   │ + Retry (3x)     │                                                      │
│   └────────┬─────────┘                                                      │
│            │                                                                │
│   Attempt 1: Idempotency check → NOT found → Process → FAIL (timeout)      │
│   Attempt 2: Idempotency check → NOT found → Process → FAIL (timeout)      │
│   Attempt 3: Idempotency check → NOT found → Process → FAIL                │
│            │                                                                │
│            ▼                                                                │
│   ┌──────────────────┐                                                      │
│   │   DLQ (Fallback) │  Save to queue with same orderId                    │
│   └────────┬─────────┘                                                      │
│            │                                                                │
│            │ ... Later, service recovers ...                                │
│            │                                                                │
│            ▼                                                                │
│   ┌──────────────────┐                                                      │
│   │  DLQ Consumer    │                                                      │
│   │                  │                                                      │
│   │  Replay attempt: │                                                      │
│   │  Idempotency     │ → Key found? Return cached ✅                        │
│   │  check           │ → Key NOT found? Process payment ✅                  │
│   │                  │                                                      │
│   └──────────────────┘                                                      │
│                                                                             │
│   ✅ Payment processed EXACTLY ONCE - no duplicates!                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Interview Answer

> "To prevent duplicate transactions during retries and DLQ replays, we use **idempotency keys**. Each payment request has a unique key (e.g., `payment:{orderId}`). Before processing, we check Redis - if the key exists, we return the cached response. If not, we process and cache the result. This ensures **exactly-once semantics** even with at-least-once delivery."

### State Machine

```
CLOSED ──(failures)──► OPEN ──(timeout)──► HALF-OPEN
   ▲                                           │
   └────────(success)──────────────────────────┘
```

### States:

| State | Behavior |
|-------|----------|
| **CLOSED** | Normal operation, requests pass through |
| **OPEN** | All requests fail immediately (fast fail) |
| **HALF-OPEN** | Limited requests allowed to test recovery |

### Resilience4j Circuit Breaker - Complete Example

#### 1. Add Dependencies (pom.xml)

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.1.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

#### 2. Configuration (application.yml)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        # When to OPEN the circuit
        failureRateThreshold: 50          # Open if 50% requests fail
        slowCallRateThreshold: 50         # Open if 50% calls are slow
        slowCallDurationThreshold: 2s     # What's considered "slow"
        
        # Sliding window for monitoring
        slidingWindowType: COUNT_BASED    # or TIME_BASED
        slidingWindowSize: 10             # Last 10 calls monitored
        minimumNumberOfCalls: 5           # Min calls before calculating failure rate
        
        # Recovery settings
        waitDurationInOpenState: 30s      # Stay OPEN for 30s before trying
        permittedNumberOfCallsInHalfOpenState: 3  # Test calls in HALF-OPEN
        
        # Auto transition
        automaticTransitionFromOpenToHalfOpenEnabled: true
        
        # What counts as failure
        recordExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignoreExceptions:
          - com.example.BusinessException  # Don't count business errors

      userService:
        failureRateThreshold: 60
        slidingWindowSize: 20
        waitDurationInOpenState: 60s

  # Combine with other patterns
  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 500ms
        
  timelimiter:
    instances:
      paymentService:
        timeoutDuration: 3s
```

#### 3. Service Implementation

```java
@Service
@Slf4j
public class PaymentService {

    private final PaymentClient paymentClient;

    public PaymentService(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }

    // Circuit Breaker + Retry + TimeLimiter combined
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    @Retry(name = "paymentService")
    @TimeLimiter(name = "paymentService")
    public CompletableFuture<PaymentResponse> processPayment(PaymentRequest request) {
        log.info("Processing payment for order: {}", request.getOrderId());
        return CompletableFuture.supplyAsync(() -> 
            paymentClient.charge(request)
        );
    }

    // Fallback method - MUST have same return type + Exception parameter
    public CompletableFuture<PaymentResponse> paymentFallback(
            PaymentRequest request, 
            Exception ex) {
        
        log.error("Payment failed for order: {}, error: {}", 
            request.getOrderId(), ex.getMessage());
        
        // Return cached/default response or throw custom exception
        return CompletableFuture.completedFuture(
            PaymentResponse.builder()
                .orderId(request.getOrderId())
                .status(PaymentStatus.PENDING)
                .message("Payment service unavailable, queued for retry")
                .build()
        );
    }
}
```

#### 4. Synchronous Version (without CompletableFuture)

```java
@Service
public class UserService {

    private final UserClient userClient;

    @CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
    @Retry(name = "userService", fallbackMethod = "getUserFallback")
    public User getUser(Long userId) {
        return userClient.getUser(userId);
    }

    // Fallback - return cached or default user
    public User getUserFallback(Long userId, Exception ex) {
        log.warn("Fallback for user {}: {}", userId, ex.getMessage());
        
        // Option 1: Return from cache
        return userCache.get(userId);
        
        // Option 2: Return default
        // return User.builder()
        //     .id(userId)
        //     .name("Guest User")
        //     .build();
    }
}
```

#### 5. Monitor Circuit Breaker State

```java
@Component
public class CircuitBreakerMonitor {

    private final CircuitBreakerRegistry registry;

    public CircuitBreakerMonitor(CircuitBreakerRegistry registry) {
        this.registry = registry;
        
        // Listen to state changes
        registry.circuitBreaker("paymentService")
            .getEventPublisher()
            .onStateTransition(event -> 
                log.warn("Circuit Breaker '{}' state changed: {} -> {}",
                    event.getCircuitBreakerName(),
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState()
                )
            )
            .onFailureRateExceeded(event ->
                log.error("Circuit Breaker '{}' failure rate exceeded: {}%",
                    event.getCircuitBreakerName(),
                    event.getFailureRate()
                )
            );
    }
    
    // Expose circuit breaker state via endpoint
    public Map<String, String> getCircuitBreakerStates() {
        return registry.getAllCircuitBreakers()
            .stream()
            .collect(Collectors.toMap(
                CircuitBreaker::getName,
                cb -> cb.getState().toString()
            ));
    }
}
```

#### 6. REST Controller with Circuit Breaker

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final PaymentService paymentService;

    @PostMapping("/{orderId}/pay")
    public ResponseEntity<PaymentResponse> payOrder(
            @PathVariable String orderId,
            @RequestBody PaymentRequest request) {
        
        try {
            PaymentResponse response = paymentService
                .processPayment(request)
                .get(5, TimeUnit.SECONDS);  // Wait for async result
            
            return ResponseEntity.ok(response);
            
        } catch (Exception e) {
            return ResponseEntity
                .status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(PaymentResponse.builder()
                    .status(PaymentStatus.FAILED)
                    .message("Payment service unavailable")
                    .build());
        }
    }
}
```

#### 7. Circuit Breaker State Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CIRCUIT BREAKER STATE TRANSITIONS                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  CLOSED (Normal)                                                    │   │
│   │  ✅ All requests ACCEPTED and sent to downstream service            │   │
│   │  ✅ Monitoring failures (counting in sliding window)                │   │
│   └─────────────────────────────┬───────────────────────────────────────┘   │
│                                 │                                           │
│                                 │ Failure rate > 50% (threshold breached)   │
│                                 ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  OPEN (Circuit Tripped!)                                            │   │
│   │  ❌ All requests REJECTED immediately (fail fast)                   │   │
│   │  ❌ No calls to downstream service                                  │   │
│   │  ❌ Returns fallback response instantly                             │   │
│   │  ⏳ Waiting 30 seconds before testing...                            │   │
│   └─────────────────────────────┬───────────────────────────────────────┘   │
│                                 │                                           │
│                                 │ Wait duration expires (30s)               │
│                                 ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  HALF-OPEN (Testing Recovery)                                       │   │
│   │  🔄 Allow LIMITED requests (3 test calls) to check if recovered     │   │
│   │  🔄 Other requests still rejected                                   │   │
│   └─────────────────────────────┬───────────────────────────────────────┘   │
│                                 │                                           │
│               ┌─────────────────┴─────────────────┐                         │
│               │                                   │                         │
│        Test calls SUCCEED                  Test calls FAIL                  │
│               │                                   │                         │
│               ▼                                   ▼                         │
│   ┌───────────────────────┐           ┌───────────────────────┐            │
│   │  → Back to CLOSED     │           │  → Back to OPEN       │            │
│   │                       │           │                       │            │
│   │  ✅ Service recovered!│           │  ❌ Still failing!    │            │
│   │  ✅ ALL new requests  │           │  ❌ Wait another 30s  │            │
│   │     ACCEPTED again    │           │  ❌ Keep rejecting    │            │
│   └───────────────────────┘           └───────────────────────┘            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

SUMMARY:
─────────────────────────────────────────────────────────────────────────────
CLOSED    → Requests ACCEPTED ✅ (normal operation)
OPEN      → Requests REJECTED ❌ (protect system, use fallback)
HALF-OPEN → Test requests only 🔄 (checking if service recovered)
           → If tests pass → CLOSED → ALL requests ACCEPTED again ✅
```

#### 8. Actuator Endpoints (Monitor via HTTP)

```yaml
# Enable actuator endpoints
management:
  endpoints:
    web:
      exposure:
        include: health, circuitbreakers, circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
```

```bash
# Check circuit breaker status
curl http://localhost:8080/actuator/circuitbreakers

# Response:
{
  "circuitBreakers": {
    "paymentService": {
      "state": "CLOSED",
      "failureRate": "0.0%",
      "slowCallRate": "0.0%",
      "bufferedCalls": 5,
      "failedCalls": 0
    }
  }
}
```

---

## Retry with Exponential Backoff

**Retry failed requests with increasing delays to handle transient failures.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXPONENTIAL BACKOFF                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Attempt 1: Immediate                                          │
│   Attempt 2: Wait 100ms                                         │
│   Attempt 3: Wait 200ms                                         │
│   Attempt 4: Wait 400ms                                         │
│   Attempt 5: Wait 800ms (max)                                   │
│                                                                  │
│   + Jitter (random variance) to prevent thundering herd         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation:

```java
@Retry(name = "paymentService", fallbackMethod = "paymentFallback")
public Payment processPayment(PaymentRequest request) {
    return paymentClient.process(request);
}

// Configuration
resilience4j.retry:
  instances:
    paymentService:
      maxAttempts: 3
      waitDuration: 100ms
      enableExponentialBackoff: true
      exponentialBackoffMultiplier: 2
      retryExceptions:
        - java.io.IOException
        - java.util.concurrent.TimeoutException
```

---

## Bulkhead

**Isolate resources per service/operation so failure in one doesn't exhaust resources for others.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    BULKHEAD PATTERN                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   WITHOUT BULKHEAD:                                             │
│   ┌────────────────────────────────────────────────────┐        │
│   │              SHARED THREAD POOL (100)              │        │
│   │  Service A ████████████████████████████████████████ │       │
│   │  Service B (blocked - no threads available)         │       │
│   │  Service C (blocked - no threads available)         │       │
│   └────────────────────────────────────────────────────┘        │
│   If Service A is slow, it consumes ALL threads!               │
│                                                                  │
│   WITH BULKHEAD:                                                │
│   ┌──────────────────┐ ┌──────────────────┐ ┌────────────────┐  │
│   │ Service A Pool   │ │ Service B Pool   │ │ Service C Pool │  │
│   │     (30)         │ │     (30)         │ │     (40)       │  │
│   │ ██████████████   │ │ ░░░░░░░░░░░░░░   │ │ ░░░░░░░░░░░░░  │  │
│   └──────────────────┘ └──────────────────┘ └────────────────┘  │
│   Service A issues don't affect B and C!                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Types of Bulkheads:

| Type | Description | Use Case |
|------|-------------|----------|
| **Thread Pool** | Separate thread pools per dependency | Blocking I/O calls |
| **Semaphore** | Limit concurrent calls | Lightweight isolation |
| **Pod Isolation** | Separate pods per service | Kubernetes deployments |

### Implementation:

```java
@Bulkhead(name = "inventoryService", type = Bulkhead.Type.THREADPOOL)
public Inventory checkInventory(String productId) {
    return inventoryClient.check(productId);
}

// Configuration
resilience4j.bulkhead:
  instances:
    inventoryService:
      maxConcurrentCalls: 25
      maxWaitDuration: 0
```

---

## Timeout Pattern

**Always set timeouts to prevent indefinite waiting.**

```java
@TimeLimiter(name = "userService")
public CompletableFuture<User> getUser(Long id) {
    return CompletableFuture.supplyAsync(() -> userClient.getUser(id));
}

// Configuration
resilience4j.timelimiter:
  instances:
    userService:
      timeoutDuration: 3s
      cancelRunningFuture: true
```

---

## Combining Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│                    RESILIENCE STACK                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Request                                                       │
│      │                                                          │
│      ▼                                                          │
│   ┌─────────────────┐                                           │
│   │    Bulkhead     │  ← Limit concurrent requests              │
│   └────────┬────────┘                                           │
│            │                                                    │
│   ┌────────▼────────┐                                           │
│   │ Circuit Breaker │  ← Fail fast if service is down           │
│   └────────┬────────┘                                           │
│            │                                                    │
│   ┌────────▼────────┐                                           │
│   │     Retry       │  ← Retry transient failures               │
│   └────────┬────────┘                                           │
│            │                                                    │
│   ┌────────▼────────┐                                           │
│   │    Timeout      │  ← Don't wait forever                     │
│   └────────┬────────┘                                           │
│            │                                                    │
│            ▼                                                    │
│      Actual Call                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

[← Back to Index](./README.md) | [Next: Observability Patterns →](./06-observability-patterns.md)

