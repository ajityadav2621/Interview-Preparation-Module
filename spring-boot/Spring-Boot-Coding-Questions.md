# Spring Boot – Coding Questions

## Q1: Complete REST API with Validation
Build a REST API for managing products with validation, pagination, and exception handling.

```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotBlank
    @Size(min = 2, max = 100)
    private String name;
    
    @NotNull
    @DecimalMin("0.01")
    private BigDecimal price;
    
    @PositiveOrZero
    private int stock;
    
    @Enumerated(EnumType.STRING)
    private ProductStatus status;
}

@Data
public class ProductRequest {
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100)
    private String name;
    
    @NotNull
    @DecimalMin(value = "0.01", message = "Price must be positive")
    private BigDecimal price;
    
    @PositiveOrZero
    private int stock;
}

@RestController
@RequestMapping("/api/v1/products")
public class ProductController {
    @Autowired private ProductService service;
    
    @GetMapping("/{id}")
    public ResponseEntity<Product> get(@PathVariable Long id) {
        return service.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    public ResponseEntity<Product> create(
            @Valid @RequestBody ProductRequest req) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(service.create(req));
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<Product> update(
            @PathVariable Long id,
            @Valid @RequestBody ProductRequest req) {
        return service.update(id, req)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
    
    @GetMapping
    public ResponseEntity<Page<Product>> list(
            @RequestParam(required = false) String status,
            @PageableDefault(size = 20, sort = "name", 
                             direction = Sort.Direction.ASC) 
            Pageable pageable) {
        Page<Product> products = status != null
            ? service.findByStatus(ProductStatus.valueOf(status), pageable)
            : service.findAll(pageable);
        return ResponseEntity.ok(products);
    }
}

@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach(err -> {
            String field = ((FieldError) err).getField();
            errors.put(field, err.getDefaultMessage());
        });
        return ResponseEntity.badRequest().body(errors);
    }
}
```

## Q2: Async Notification System
Build a notification system with async email, SMS, and push notifications.

```java
@Service
public class NotificationService {
    @Autowired private EmailSender emailSender;
    @Autowired private SmsSender smsSender;
    @Autowired private PushSender pushSender;
    
    @Async("notificationPool")
    @Retryable(value = NotificationException.class, maxAttempts = 3,
               backoff = @Backoff(delay = 2000))
    public CompletableFuture<String> sendEmail(Email email) {
        emailSender.send(email);
        return CompletableFuture.completedFuture("Email sent");
    }
    
    @Async("notificationPool")
    public CompletableFuture<String> sendSms(Sms sms) {
        smsSender.send(sms);
        return CompletableFuture.completedFuture("SMS sent");
    }
    
    @Async("notificationPool")
    public CompletableFuture<String> sendPush(PushNotification push) {
        pushSender.send(push);
        return CompletableFuture.completedFuture("Push sent");
    }
}

@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean("notificationPool")
    public Executor notificationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("Notify-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

@Service
public class OrderNotificationService {
    @Autowired private NotificationService notificationService;
    
    public void notifyOrderPlaced(Order order) {
        notificationService.sendEmail(order.getEmail());
        notificationService.sendSms(order.getPhone());
        notificationService.sendPush(order.getDeviceToken());
    }
}
```

## Q3: Custom Actuator Endpoint
Create a custom actuator endpoint for external service health checks.

```java
@Component
@Endpoint(id = "serviceHealth")
public class ServiceHealthEndpoint {
    
    @Autowired private RestTemplate restTemplate;
    private final Map<String, HealthDetail> cache = new ConcurrentHashMap<>();
    
    @ReadOperation
    public Map<String, HealthDetail> health() {
        Map<String, HealthDetail> result = new LinkedHashMap<>();
        result.put("database", checkDatabase());
        result.put("redis", checkRedis());
        result.put("email-service", checkEmailService());
        result.put("payment-gateway", checkPaymentGateway());
        cache.putAll(result);
        return result;
    }
    
    @ReadOperation
    public HealthDetail healthForService(@Selector String service) {
        return cache.getOrDefault(service, 
            new HealthDetail(false, "Unknown service"));
    }
    
    @WriteOperation
    public Map<String, String> forceRefresh() {
        cache.clear();
        health();
        return Map.of("status", "Cache cleared and refreshed");
    }
    
    private HealthDetail checkDatabase() {
        try {
            jdbcTemplate.execute("SELECT 1");
            return new HealthDetail(true, "OK");
        } catch (Exception e) {
            return new HealthDetail(false, e.getMessage());
        }
    }
}
```

## Q4: Kafka Order Pipeline
Build an order processing pipeline with Kafka.

```java
@Service
public class OrderService {
    @Autowired private OrderRepository orderRepository;
    @Autowired private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Transactional
    public OrderResponse placeOrder(OrderRequest request) {
        Order order = orderRepository.save(request.toOrder());
        kafkaTemplate.send("order-created", new OrderCreatedEvent(
            order.getId(), order.getItems(), order.getTotal()
        ));
        return OrderResponse.from(order);
    }
}

@Component
public class InventoryEventHandler {
    @Autowired private InventoryService inventoryService;
    
    @KafkaListener(topics = "order-created", groupId = "inventory-service")
    public void onOrderCreated(OrderCreatedEvent event) {
        try {
            inventoryService.reserve(event.getOrderId(), event.getItems());
            kafkaTemplate.send("inventory-reserved", 
                new InventoryReservedEvent(event.getOrderId()));
        } catch (OutOfStockException e) {
            kafkaTemplate.send("order-failed", 
                new OrderFailedEvent(event.getOrderId(), "Out of stock"));
        }
    }
}

@Component
public class PaymentEventHandler {
    @Autowired private PaymentService paymentService;
    
    @KafkaListener(topics = "inventory-reserved", groupId = "payment-service")
    public void onInventoryReserved(InventoryReservedEvent event) {
        try {
            paymentService.charge(event.getOrderId());
            kafkaTemplate.send("payment-completed", 
                new PaymentCompletedEvent(event.getOrderId()));
        } catch (PaymentFailedException e) {
            kafkaTemplate.send("order-failed", 
                new OrderFailedEvent(event.getOrderId(), "Payment failed"));
        }
    }
}
```

## Q5: Multi-Tenant Data Access
Implement multi-tenant data access with ThreadLocal tenant context.

```java
public class TenantContext {
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();
    
    public static void setTenant(String tenantId) { CURRENT_TENANT.set(tenantId); }
    public static String getCurrentTenant() { return CURRENT_TENANT.get(); }
    public static void clear() { CURRENT_TENANT.remove(); }
}

@Entity
@MappedSuperclass
public abstract class TenantAwareEntity {
    @Column(name = "tenant_id", nullable = false)
    private String tenantId;
    
    @PrePersist
    protected void onCreate() {
        this.tenantId = TenantContext.getCurrentTenant();
    }
}

@Entity
@Table(name = "customers")
@Where(clause = "tenant_id = :currentTenantId")
public class Customer extends TenantAwareEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
}

@Service
public class CustomerService {
    @Autowired private CustomerRepository repo;
    
    public List<Customer> getAll() {
        return repo.findAll(); // @Where filters by tenant
    }
    
    public Optional<Customer> getById(Long id) {
        return repo.findById(id);
    }
}

// Filter: tenant from header
@Component
public class TenantFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req,
            HttpServletResponse res, FilterChain chain) 
            throws ServletException, IOException {
        String tenant = req.getHeader("X-Tenant-ID");
        TenantContext.setTenant(tenant);
        try { chain.doFilter(req, res); }
        finally { TenantContext.clear(); }
    }
}
```

## Q6: REST API with HATEOAS and Versioning
Build a versioned API with HATEOAS links.

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    @Autowired private UserService userService;
    
    @GetMapping("/{id}")
    public EntityModel<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id).orElseThrow();
        return EntityModel.of(user,
            linkTo(methodOn(UserControllerV1.class).getUser(id)).withSelfRel(),
            linkTo(methodOn(UserControllerV1.class).getAll(null, null))
                .withRel("users"),
            linkTo(methodOn(OrderController.class).getOrdersForUser(id))
                .withRel("orders")
        );
    }
    
    @GetMapping
    public ResponseEntity<CollectionModel<EntityModel<User>>> getAll(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        List<EntityModel<User>> users = userService.findAll(page, size)
            .stream()
            .map(u -> EntityModel.of(u,
                linkTo(methodOn(UserControllerV1.class).getUser(u.getId()))
                    .withSelfRel()))
            .collect(Collectors.toList());
        return ResponseEntity.ok(CollectionModel.of(users,
            linkTo(methodOn(UserControllerV1.class).getAll(page, size))
                .withSelfRel()));
    }
}
```

## Q7: WebSocket Chat Application
Build a real-time chat with WebSocket and STOMP.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic", "/queue");
        config.setApplicationDestinationPrefixes("/app");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/chat").setAllowedOriginPatterns("*")
            .withSockJS();
    }
}

@RestController
public class ChatController {
    @Autowired private SimpMessagingTemplate messagingTemplate;
    @Autowired private MessageRepository messageRepository;
    
    @MessageMapping("/chat.send")
    public void sendMessage(ChatMessage message) {
        messageRepository.save(message);
        messagingTemplate.convertAndSend(
            "/topic/room/" + message.getRoomId(), message);
    }
    
    @MessageMapping("/chat.join")
    public void joinRoom(JoinMessage message, 
                         SimpMessageHeaderAccessor headerAccessor) {
        headerAccessor.getSessionAttributes().put("roomId", message.getRoomId());
        messagingTemplate.convertAndSend(
            "/topic/room/" + message.getRoomId(),
            new JoinNotification(message.getUsername() + " joined"));
    }
}
```

## Q8: Caching with Redis
Implement a multi-level caching strategy.

```java
@Service
public class ProductCacheService {
    @Autowired private ProductRepository repository;
    @Autowired private RedisTemplate<String, Object> redisTemplate;
    
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return repository.findById(id).orElseThrow();
    }
    
    @Cacheable(value = "products", key = "#category + '_' + #page")
    public List<Product> getByCategory(String category, int page) {
        return repository.findByCategory(category, 
            PageRequest.of(page, 20)).getContent();
    }
    
    public Product getWithLocalCache(Long id) {
        String key = "product:" + id;
        Product product = (Product) localCache.getIfPresent(key);
        if (product == null) {
            product = getProduct(id); // calls @Cacheable
            localCache.put(key, product);
        }
        return product;
    }
    
    @CacheEvict(value = "products", key = "#id")
    public void updateProduct(Long id, Product product) {
        repository.save(product);
    }
}
```

## Q9: Custom Error Handler with Exception Translation
Build a comprehensive error handling system.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException ex) {
        ApiError error = new ApiError(HttpStatus.NOT_FOUND, ex.getMessage(), 
            LocalDateTime.now());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
            .collect(Collectors.toList());
        ApiError error = new ApiError(HttpStatus.BAD_REQUEST, 
            "Validation failed", errors, LocalDateTime.now());
        return ResponseEntity.badRequest().body(error);
    }
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiError> handleBusiness(BusinessException ex) {
        log.warn("Business error: {}", ex.getMessage());
        ApiError error = new ApiError(HttpStatus.CONFLICT, ex.getMessage());
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleGeneric(Exception ex) {
        log.error("Unexpected error", ex);
        ApiError error = new ApiError(HttpStatus.INTERNAL_SERVER_ERROR,
            "Unexpected error");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

## Q10: Integration with Payment Gateway using Circuit Breaker
Build a resilient payment integration.

```java
@Service
public class PaymentService {
    @Autowired private PaymentClient paymentClient;
    
    @CircuitBreaker(name = "paymentGateway", fallbackMethod = "fallback")
    @TimeLimiter(name = "paymentGateway")
    @Retry(name = "paymentGateway")
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest req) {
        return paymentClient.charge(req)
            .thenApply(response -> {
                if (response.isSuccess()) {
                    return PaymentResult.success(response.getTransactionId());
                }
                throw new PaymentFailedException("Payment declined");
            });
    }
    
    public CompletableFuture<PaymentResult> fallback(PaymentRequest req, Exception e) {
        paymentQueue.enqueue(new QueuedPayment(req, LocalDateTime.now()));
        return CompletableFuture.completedFuture(
            PaymentResult.pending("Payment queued for processing"));
    }
}

@Configuration
public class CircuitBreakerConfig {
    @Bean
    public PaymentService paymentService() {
        return new PaymentService();
    }
    
    @Bean
    public CircuitBreakerConfig customConfig() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofMillis(5000))
            .slidingWindowSize(10)
            .minimumNumberOfCalls(5)
            .build();
        return config;
    }
}
```
