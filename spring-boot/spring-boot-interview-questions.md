# Spring Boot Interview Questions

## Q11. How Does Spring Boot Handle Dependency Injection Internally? Explain the Bean Lifecycle.

### Dependency Injection in Spring

Spring uses **IoC (Inversion of Control)** container to manage beans. DI is achieved through:
1. **Constructor Injection** (recommended)
2. Setter Injection
3. Field Injection (`@Autowired` on field)

### Internal Mechanism

```java
@Service
public class OrderService {
    private final PaymentService paymentService;
    
    // Constructor injection — Spring detects via reflection
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**How Spring resolves this:**
1. Reads `@ComponentScan` annotated packages
2. Creates `BeanDefinition` for each class with `@Component`, `@Service`, `@Repository`, `@Controller`
3. Resolves dependencies via constructor parameters (type → by-name if multiple)
4. Uses `BeanFactory` + `ApplicationContext` to instantiate

### Bean Lifecycle (Full Picture)

```
1. Instantiate (constructor)
       ↓
2. Populate properties (DI — inject dependencies)
       ↓
3. BeanNameAware.setBeanName()
       ↓
4. BeanFactoryAware.setBeanFactory()
       ↓
5. ApplicationContextAware.setApplicationContext()
       ↓
6. BeanPostProcessor.postProcessBeforeInitialization()
       ↓
7. @PostConstruct method
       ↓
8. InitializingBean.afterPropertiesSet()
       ↓
9. Custom init-method
       ↓
10. BeanPostProcessor.postProcessAfterInitialization()
       ↓
   [Bean is ready for use]
       ↓
   [Container shuts down]
       ↓
11. @PreDestroy method
       ↓
12. DisposableBean.destroy()
       ↓
13. Custom destroy-method
```

### Practical Example

```java
@Component
public class DatabaseConnection {
    
    private DataSource dataSource;
    
    public DatabaseConnection(DataSource dataSource) {
        // 1. Instantiation
        this.dataSource = dataSource;
        // 2. Populate (DI)
    }
    
    @PostConstruct
    public void init() {
        // 7. Init — open connections, warm caches
        connect();
    }
    
    @PreDestroy
    public void cleanup() {
        // 11. Cleanup — close resources gracefully
        disconnect();
    }
}
```

### Bean Scopes

| Scope | Description |
|-------|-------------|
| `singleton` (default) | One instance per Spring container |
| `prototype` | New instance every time requested |
| `request` | One per HTTP request (web) |
| `session` | One per HTTP session (web) |
| `application` | One per ServletContext |

### Lazy Initialization
```java
@Component
@Lazy   // Created only when first requested
public class HeavyService { ... }
```

### Why Constructor Injection is Preferred
- **Immutability** — fields can be `final`
- **Testability** — easy to mock in unit tests
- **Fail-fast** — missing dependency fails at startup, not runtime
- **No reflection** — no `ReflectionUtils` overhead
- **No NPE** — guarantees dependency is set

---

## Q12. How Do You Implement Request Idempotency in Backend Services?

### The Problem
Without idempotency:
- Network retries cause **duplicate charges**
- Mobile app retries on timeout cause **double submissions**
- Distributed systems with at-least-once delivery cause **duplicate processing**

### Solution: Idempotency Key

```java
@RestController
@RequestMapping("/api/payments")
public class PaymentController {
    
    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody @Valid PaymentRequest request) {
        
        return idempotencyService.execute(
            idempotencyKey, 
            request, 
            () -> paymentService.process(request)
        );
    }
}

@Service
public class IdempotencyService {
    
    private final RedisTemplate<String, StoredResponse> redis;
    private final ObjectMapper mapper;
    
    public <T> ResponseEntity<T> execute(
            String key, Object request, Supplier<ResponseEntity<T>> action) {
        
        String fingerprint = sha256(serialize(request));
        String cacheKey = "idem:" + key + ":" + fingerprint;
        
        // 1. Check if response already cached
        StoredResponse<T> stored = redis.opsForValue().get(cacheKey);
        if (stored != null) {
            if (stored.isCompleted()) {
                return ResponseEntity.status(stored.getStatus()).body(stored.getBody());
            }
            // Concurrent request — return 409 Conflict
            throw new ConcurrentRequestException("Request in progress");
        }
        
        // 2. Reserve the key (atomic NX — "set if not exists")
        Boolean reserved = redis.opsForValue()
            .setIfAbsent(cacheKey + ":lock", "PENDING", Duration.ofMinutes(5));
        if (Boolean.FALSE.equals(reserved)) {
            throw new ConcurrentRequestException("Duplicate request");
        }
        
        try {
            // 3. Execute the action
            ResponseEntity<T> response = action.get();
            
            // 4. Cache the result
            redis.opsForValue().set(cacheKey,
                new StoredResponse<>(response.getStatusCode(), response.getBody()),
                Duration.ofHours(24));
            
            return response;
        } catch (Exception e) {
            // 5. Clear reservation on failure — allow retry
            redis.delete(cacheKey + ":lock");
            throw e;
        }
    }
}
```

### Database-Level Idempotency

```sql
CREATE TABLE payment_requests (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    idempotency_key VARCHAR(64) NOT NULL,
    request_hash    CHAR(64) NOT NULL,
    response_body   TEXT,
    status          VARCHAR(20) NOT NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_idem (idempotency_key, request_hash)
);
```

```java
@Transactional
public PaymentResponse process(IdempotencyKey key, PaymentRequest req) {
    // 1. Try to insert — fails on duplicate
    try {
        idemRepository.save(new PaymentRequestRecord(key, hash(req)));
    } catch (DuplicateKeyException e) {
        // 2. Return previously stored response
        return idemRepository.findByKey(key).getResponse();
    }
    
    // 3. Process payment
    PaymentResult result = paymentGateway.charge(req);
    
    // 4. Store response
    idemRepository.saveResponse(key, result);
    return result;
}
```

### Best Practices
- **TTL on idempotency key** — typically 24 hours
- **Combine with request hash** — same key but different payload = error
- **Return same response** — including same status code
- **Different keys = different requests** — even if same payload

---

## Q13. How Do You Design Secure and Scalable Authentication and Authorization for Banking APIs?

### Authentication: "Who are you?"
### Authorization: "What can you do?"

### JWT-Based Stateless Authentication

```java
@RestController
@RequestMapping("/api/account")
public class AccountController {
    
    @GetMapping("/{id}/balance")
    @PreAuthorize("hasRole('CUSTOMER') and @accountService.isOwner(#id, authentication.principal.id)")
    public BalanceResponse getBalance(@PathVariable String id) {
        return accountService.getBalance(id);
    }
}
```

### Architecture

```
Client → API Gateway (AuthN) → Microservice (AuthZ check)
                                     ↓
                                  JWT validation
                                  Method-level @PreAuthorize
```

### JWT Structure
```json
{
  "sub": "user123",
  "roles": ["CUSTOMER"],
  "permissions": ["account:read", "payment:create"],
  "tenantId": "bank-abc",
  "iat": 1694000000,
  "exp": 1694003600,
  "jti": "unique-token-id"
}
```

### Multi-Layer Security

**1. Transport Layer**
- TLS 1.3 only
- HSTS headers
- mTLS for service-to-service

**2. Token Security**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public JwtDecoder jwtDecoder() {
        NimbusJwtDecoder decoder = NimbusJwtDecoder
            .withPublicKey(publicKey)
            .build();
        
        // Custom validator
        decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
            JwtValidators.createDefault(),   // exp, nbf
            new IssuerValidator("https://auth.bank.com"),
            new AudienceValidator("banking-api"),
            new TokenBlacklistValidator(redis)
        ));
        return decoder;
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth -> oauth.jwt(jwt -> {}))
            .headers(h -> h
                .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
                .frameOptions(f -> f.deny())
            );
        return http.build();
    }
}
```

**3. Authorization (RBAC + ABAC)**
```java
@PreAuthorize("@accountAccessService.canView(#accountId, authentication)")
public Account getAccount(String accountId) { ... }

@Service
public class AccountAccessService {
    public boolean canView(String accountId, Authentication auth) {
        UserPrincipal user = (UserPrincipal) auth.getPrincipal();
        return accountRepository.isOwnedBy(accountId, user.getId())
            || user.hasRole("AUDITOR");
    }
}
```

**4. Audit Logging**
```java
@Aspect
@Component
public class AuditAspect {
    
    @After("@annotation(auditable)")
    public void audit(JoinPoint jp, Auditable auditable) {
        auditLogService.log(new AuditEvent(
            currentUserId(),
            auditable.action(),
            jp.getArgs(),
            Instant.now()
        ));
    }
}
```

### Token Refresh Flow
```
Access Token (15 min) + Refresh Token (7 days)
- Short-lived access token limits damage if compromised
- Refresh token rotated on use (detect theft)
- Refresh token stored in HttpOnly secure cookie
```

### Rate Limiting + Brute Force Protection
```java
@RateLimiter(name = "auth", fallbackMethod = "rateLimited")
public TokenResponse login(@RequestBody LoginRequest req) { ... }
```

### Scalability
- **JWT** = stateless = horizontally scalable
- **Token introspection** cached in Redis (avoid DB hit per request)
- **Distributed session** (Redis) if stateful needed
- **Short tokens + refresh** = balance security and UX

---

## Q14. How Do You Handle Partial Failures When Multiple Downstream Services Are Involved?

### The Scenario
Order placement needs:
- Inventory Service (reserve stock)
- Payment Service (charge)
- Notification Service (send confirmation)

What if Payment fails after Inventory was reserved?

### Strategies

**1. Saga Pattern (Best for Long-Running Transactions)**

Choreography:
```java
@Service
public class OrderService {
    
    @Transactional
    public Order createOrder(OrderRequest req) {
        Order order = orderRepository.save(new Order(req));
        
        // Publish event — other services listen
        kafkaTemplate.send("order-events", 
            new OrderCreatedEvent(order.getId(), req.getItems(), req.getPayment()));
        
        return order;
    }
}

// Inventory Service listens
@KafkaListener(topics = "order-events")
public void onOrderCreated(OrderCreatedEvent event) {
    try {
        inventoryService.reserve(event.getOrderId(), event.getItems());
        kafkaTemplate.send("inventory-events", new InventoryReservedEvent(event.getOrderId()));
    } catch (OutOfStockException e) {
        kafkaTemplate.send("compensation-events", new ReleasePaymentEvent(event.getOrderId()));
    }
}

// Payment Service listens
@KafkaListener(topics = "inventory-events")
public void onInventoryReserved(InventoryReservedEvent event) {
    try {
        paymentService.charge(event.getOrderId());
        kafkaTemplate.send("payment-events", new PaymentCompletedEvent(event.getOrderId()));
    } catch (PaymentFailedException e) {
        kafkaTemplate.send("compensation-events", new ReleaseInventoryEvent(event.getOrderId()));
    }
}
```

Orchestration:
```java
@Service
public class OrderSagaOrchestrator {
    
    public void execute(OrderRequest req) {
        Saga saga = sagaManager.begin();
        
        saga.step()
            .invoke(() -> inventoryService.reserve(req))
            .compensate(() -> inventoryService.release(reservationId))
            .onException(OutOfStockException.class, saga::compensate);
        
        saga.step()
            .invoke(() -> paymentService.charge(req))
            .compensate(() -> paymentService.refund(chargeId));
        
        saga.step()
            .invoke(() -> notificationService.sendConfirmation(req));
        
        saga.execute();
    }
}
```

**2. Circuit Breaker + Fallback**
```java
@CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
@TimeLimiter(name = "payment")
@Retry(name = "payment")
public PaymentResult charge(PaymentRequest req) {
    return paymentClient.charge(req);
}

public PaymentResult paymentFallback(PaymentRequest req, Exception e) {
    // Queue for later retry
    paymentQueue.enqueue(req);
    return PaymentResult.pending("Payment queued");
}
```

**3. Outbox Pattern (For Distributed Writes)**
```java
@Transactional
public void createOrder(OrderRequest req) {
    // 1. Save order (in DB)
    Order order = orderRepository.save(new Order(req));
    
    // 2. Save event in outbox (SAME transaction)
    outboxRepository.save(new OutboxEvent(
        "ORDER_CREATED", 
        order.getId(), 
        serialize(order)
    ));
    // Both succeed or both fail — atomic
}

// Separate worker reads outbox and publishes to Kafka
@Scheduled(fixedDelay = 1000)
public void publishOutbox() {
    outboxRepository.findUnpublished()
        .forEach(event -> {
            kafkaTemplate.send(event.getTopic(), event.getPayload());
            outboxRepository.markPublished(event.getId());
        });
}
```

**4. Graceful Degradation**
```java
public OrderConfirmation placeOrder(OrderRequest req) {
    String orderId = orderService.create(req);
    
    // Critical path
    PaymentResult payment = paymentService.charge(req);
    
    // Non-critical — degrade gracefully
    CompletableFuture.runAsync(() -> {
        try {
            notificationService.send(req);
            emailService.sendReceipt(req);
        } catch (Exception e) {
            log.warn("Non-critical step failed", e);
            // Queue for retry
            retryQueue.enqueue(event);
        }
    });
    
    return new OrderConfirmation(orderId, payment.getStatus());
}
```

### Key Takeaways
- **Saga** for distributed transactions needing eventual consistency
- **Circuit Breaker** to prevent cascading failures
- **Outbox** for atomic DB + message publish
- **Async + queue** for non-critical steps
- **Compensating actions** must be **idempotent**

---

## Q15. What Strategies Do You Use to Version APIs Without Breaking Existing Consumers?

### Why Version?
- Add new features
- Fix bugs in API design
- Change data formats
- Migrate clients gradually

### Versioning Strategies

**1. URI Versioning (Most Common)**
```
/api/v1/orders
/api/v2/orders
```
✅ Easy to understand  
✅ Easy to route  
✅ Easy to test  

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 { ... }

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 { ... }
```

**2. Header Versioning**
```
GET /api/orders
Accept: application/vnd.myapi.v2+json
```
✅ URI stays clean  
❌ Harder to test in browser  

**3. Query Parameter**
```
GET /api/orders?version=2
```
✅ Simple  
❌ Mixing concerns  

### Backward Compatibility Rules

**Safe Changes (No Version Bump)**
- ✅ Add new optional field to request
- ✅ Add new field to response
- ✅ Add new endpoint
- ✅ Add new enum value (client ignores unknowns)
- ✅ Loosen validation (accept more)

**Breaking Changes (Require Version Bump)**
- ❌ Remove or rename field
- ❌ Change field type
- ❌ Change field semantics
- ❌ Remove endpoint
- ❌ Tighten validation
- ❌ Change auth requirements

### Implementation Pattern

**Parallel Controllers**
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {
    private final OrderService orderService;
    
    @GetMapping("/{id}")
    public OrderV1 getOrder(@PathVariable String id) {
        Order order = orderService.findById(id);
        return OrderV1.from(order);
    }
}

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 {
    private final OrderService orderService;
    
    @GetMapping("/{id}")
    public OrderV2 getOrder(@PathVariable String id) {
        Order order = orderService.findById(id);
        return OrderV2.from(order);   // New structure, additional fields
    }
}
```

**Deprecation Headers**
```java
@GetMapping("/{id}")
public ResponseEntity<OrderV1> getOrder(@PathVariable String id) {
    Order order = orderService.findById(id);
    return ResponseEntity.ok()
        .header("Deprecation", "true")
        .header("Sunset", "Sat, 01 Jan 2025 00:00:00 GMT")
        .header("Link", "</api/v2/orders>; rel=\"successor-version\"")
        .body(OrderV1.from(order));
}
```

**Tolerant Reader Pattern**
```java
@JsonIgnoreProperties(ignoreUnknown = true)
public class OrderV1 {
    private String id;
    private BigDecimal amount;
    // ... doesn't break if API adds new fields
}
```

### Migration Strategy

```
Phase 1: Release V2 (V1 still default)
   ↓ Monitor V1 traffic
Phase 2: Communicate deprecation
   ↓ Headers, email, docs
Phase 3: Migrate known clients to V2
   ↓ Offer migration window
Phase 4: Sunset V1
   ↓ Return 410 Gone or redirect to V2
Phase 5: Decommission V1
```

### Tooling
- **OpenAPI / Swagger** — single source of truth
- **Contract testing** (Pact, Spring Cloud Contract)
- **API changelog** — public, versioned
- **API analytics** — track which versions clients use

### Key Takeaways
- Versioning is **inevitable** — plan for it from day 1
- Prefer **additive changes** — avoid breaking consumers
- Use **deprecation headers** to give clients time
- **Parallel run** old and new versions during transition
- **Document** every change in a public changelog

---

## Production Topic: @Transactional Pitfalls

### Pitfall 1: Self-Invocation → Proxy Bypass

```java
@Service
public class OrderService {
    
    // This method is transactional
    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);
        updateInventory(order);  // ❌ Transaction NEVER starts!
    }
    
    // This method is ALSO transactional, but...
    @Transactional
    public void updateInventory(Order order) {
        // This code runs WITHOUT a transaction!
        // Because it's called from within the same class,
        // the @Transactional proxy is bypassed
        inventoryRepository.updateStock(order.getItemId());
    }
}
```

**Why?** Spring uses **proxy-based AOP**. When you call a method from within the same class, the call doesn't go through the proxy — it's a direct method call. The proxy is what starts the transaction.

**Fix**:
```java
// Option 1: Call via proxy (inject self)
@Service
public class OrderService {
    @Autowired
    private OrderService self;  // Inject the proxy
    
    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);
        self.updateInventory(order);  // ✅ Goes through proxy
    }
}

// Option 2: Use AspectJ (compile-time weaving)
// Option 3: Move methods to separate services
```

### Pitfall 2: REQUIRES_NEW vs REQUIRED — Not Interchangeable

```java
@Service
public class OrderService {
    
    // Outer transaction
    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);
        
        // REQUIRES_NEW — suspends outer transaction, starts NEW one
        processPayment(order);  // Runs in separate transaction
        
        // If processPayment fails, order is STILL saved
        // because outer transaction commits independently
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void processPayment(Order order) {
        paymentRepository.charge(order);
    }
}
```

| Propagation | Behavior |
|-------------|----------|
| `REQUIRED` (default) | Join existing transaction, or create new if none exists |
| `REQUIRES_NEW` | Always create NEW transaction, suspend existing if present |
| `MANDATORY` | Require existing transaction, throw exception if none |
| `SUPPORTS` | Join existing if present, run non-transactional if none |
| `NOT_SUPPORTED` | Always run non-transactional, suspend existing |
| `NEVER` | Require NO transaction, throw exception if one exists |
| `NESTED` | Create nested transaction (savepoint) within existing |

**Common Mistake**: Using `REQUIRES_NEW` when you meant `REQUIRED`. This can cause:
- Unexpected transaction boundaries
- Partial commits (outer fails, inner succeeds)
- Increased DB load (more transactions)

### Pitfall 3: Lazy Loading in Closed Session → LazyInitializationException

```java
@Service
public class OrderService {
    
    @Transactional(readOnly = true)
    public OrderDTO getOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        
        // This works — session is open
        String userName = order.getUser().getName();
        
        return OrderDTO.from(order);
    }
    
    // ❌ This fails — session is closed
    public OrderDTO getOrderBad(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        
        // Transaction ended, session closed
        // Lazy loading fails here!
        String userName = order.getUser().getName();  // LazyInitializationException!
        
        return OrderDTO.from(order);
    }
}
```

**Why?** JPA uses lazy loading for `@OneToMany` and `@ManyToMany` relationships. The proxy needs an open session to fetch the data. Once the transaction ends, the session closes and lazy loading fails.

**Fixes**:
```java
// Option 1: Fetch eagerly (if you always need it)
@OneToMany(fetch = FetchType.EAGER)
private List<OrderItem> items;

// Option 2: Use JOIN FETCH in query
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.id = :id")
Order findByIdWithUser(@Param("id") Long id);

// Option 3: Use @Transactional on the method that accesses lazy data
@Transactional
public OrderDTO getOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    // Session is still open — lazy loading works
    String userName = order.getUser().getName();
    return OrderDTO.from(order);
}

// Option 4: Use DTO projection (recommended)
public OrderDTO getOrder(Long orderId) {
    return orderRepository.findOrderDTOById(orderId);
    // No lazy loading issues — all data fetched in one query
}
```

### Key Takeaways

- **Self-invocation** bypasses the `@Transactional` proxy — always be aware of this
- **REQUIRES_NEW** creates a new transaction, **REQUIRED** joins existing — they're not interchangeable
- **Lazy loading** requires an open session — use `JOIN FETCH`, DTO projections, or keep the transaction open
- These pitfalls are **silent** — no errors at compile time, only runtime failures
