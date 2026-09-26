# Spring Boot – Core Concepts

## 1. IoC Container & Dependency Injection

### IoC (Inversion of Control)
```java
// The Spring container manages object creation and wiring
// Instead of new MyService(), the container provides it

@Component
public class OrderService {
    private final PaymentService paymentService;
    
    // Constructor injection (preferred)
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

### DI Methods
```java
// 1. Constructor Injection (Recommended)
@Service
public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; }
}

// 2. Setter Injection
@Service
public class NotificationService {
    private EmailService emailService;
    @Autowired
    public void setEmailService(EmailService e) { this.emailService = e; }
}

// 3. Field Injection (avoid — hides dependencies)
@Service
public class ReportService {
    @Autowired
    private PdfGenerator pdfGenerator;
}
```

### @Component Hierarchy
```java
@Component              // Generic bean
@Service               // Business layer
@Repository            // Data access layer
@Controller            // Web layer (MVC)
@RestController        // REST API (Controller + @ResponseBody)
```

### @Autowired + @Qualifier
```java
@Service
public class OrderService {
    private final PaymentService paymentService;
    
    @Autowired
    public OrderService(
            @Qualifier("stripePayment") PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

---

## 2. Spring Boot Basics

### @SpringBootApplication
```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
// = @Configuration + @EnableAutoConfiguration + @ComponentScan
```

### Auto-Configuration
```
# How it works:
# 1. Spring Boot reads dependencies from classpath
# 2. Enables matching auto-configurations
# 3. E.g., spring-boot-starter-web → Tomcat + Spring MVC
# 4. E.g., spring-boot-starter-data-jpa → Hibernate + DataSource

# Disable auto-configurations:
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
```

### application.properties / application.yml
```properties
# Properties format
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.jpa.hibernate.ddl-auto=update
logging.level.root=INFO
```

```yaml
# YAML format
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
  jpa:
    hibernate:
      ddl-auto: update
logging:
  level:
    root: INFO
```

### Profiles
```java
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        // Embedded DB for dev
        return EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        // Production DataSource
        return DataSourceBuilder.create()
            .url("jdbc:mysql://prod-server/db")
            .build();
    }
}
```

### @ConfigurationProperties
```java
@Component
@ConfigurationProperties(prefix = "app.mail")
public class MailProperties {
    private String host;
    private int port;
    private String from;
    // getters & setters
}

# application.properties
app.mail.host=smtp.gmail.com
app.mail.port=587
app.mail.from=noreply@example.com
```

---

## 3. Spring MVC

### Controller Basics
```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    public ResponseEntity<User> create(
            @RequestBody @Valid UserRequest request) {
        User user = userService.create(request.toEntity());
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<User> update(
            @PathVariable Long id,
            @RequestBody @Valid UserRequest request) {
        return userService.update(id, request.toEntity())
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
    
    @GetMapping
    public ResponseEntity<List<User>> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(userService.findAll(page, size));
    }
}
```

### Exception Handling
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(), ex.getMessage(), 
            System.currentTimeMillis());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach(err -> {
            String field = ((FieldError) err).getField();
            String msg = err.getDefaultMessage();
            errors.put(field, msg);
        });
        return ResponseEntity.badRequest().body(
            new ErrorResponse(400, "Validation failed", errors));
    }
}
```

---

## 4. Spring Data JPA

### Repository
```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Derived query methods
    List<User> findByEmail(String email);
    List<User> findByLastName(String lastName);
    Optional<User> findByEmailAndPassword(String email, String password);
    
    // Pagination & Sorting
    Page<User> findByActiveTrue(Pageable pageable);
    List<User> findByAgeGreaterThanOrderByNameAsc(int age);
    
    // Native query
    @Query(value = "SELECT * FROM users WHERE status = ?1", nativeQuery = true)
    List<User> findByStatusNative(String status);
    
    // JPQL
    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmailJpql(@Param("email") String email);
}
```

### Entity Relationships
```java
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    // Many orders, one customer
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;
    
    // One order, many items
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, 
               orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}

@Entity
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;
}

// ManyToMany
@Entity
public class Student {
    @ManyToMany
    @JoinTable(name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
    private Set<Course> courses = new HashSet<>();
}
```

### Lazy vs Eager Loading
```java
// LAZY (default for @OneToMany, @ManyToMany) — loads on access
@OneToMany(fetch = FetchType.LAZY)
private List<OrderItem> items;

// EAGER (default for @ManyToOne, @OneToOne) — loads immediately
@ManyToOne(fetch = FetchType.EAGER)
private Customer customer;
```

---

## 5. Hibernate/JPA

### Entity Lifecycle
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true, length = 100)
    private String email;
    
    @Version // Optimistic locking
    private Long version;
    
    @CreationTimestamp
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

### Transaction Management
```java
@Service
public class OrderService {
    @Transactional(
        propagation = Propagation.REQUIRED,   // Join existing tx
        isolation = Isolation.READ_COMMITTED,
        timeout = 30,                        // 30 seconds
        rollbackFor = {PaymentException.class, InventoryException.class}
    )
    public Order placeOrder(OrderRequest request) {
        // All DB operations here are transactional
        Order order = orderRepository.save(createOrder(request));
        inventoryService.reserveStock(request.getItems());
        paymentService.charge(request.getPayment());
        return order;
    }
    
    // Read-only for performance
    @Transactional(readOnly = true)
    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();
    }
}
```

---

## 6. Spring Security

### Security Config
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(new JwtAuthenticationFilter(), 
                UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(e -> e
                .authenticationEntryPoint(new JwtAuthenticationEntryPoint())
                .accessDeniedHandler(new CustomAccessDeniedHandler())
            );
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}
```

### JWT Integration
```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Autowired private JwtTokenProvider tokenProvider;
    @Autowired private CustomUserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest req, 
            HttpServletResponse res, FilterChain chain) throws IOException, ServletException {
        String token = extractToken(req);
        if (token != null && tokenProvider.validate(token)) {
            String username = tokenProvider.getUsername(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(req, res);
    }
}
```

### Method-Level Security
```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }

@PreAuthorize("#userId == authentication.principal.id")
public UserProfile getOwnProfile(Long userId) { ... }

@PostAuthorize("returnObject.owner == authentication.principal.id")
public Document getDocument(Long id) { ... }
```

---

## 7. Spring Boot Actuator

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,env  # Or *
  endpoint:
    health:
      show-details: always
    info:
      enabled: true
  info:
    app:
      name: My Application
      version: @project.version@
```

### Custom Actuator Endpoint
```java
@Component
@Endpoint(id = "customHealth")
public class CustomHealthEndpoint {
    
    @ReadOperation
    public Map<String, Object> health() {
        return Map.of(
            "status", "UP",
            "database", checkDb(),
            "cache", checkCache()
        );
    }
    
    @WriteOperation
    public Map<String, String> clearCache() {
        cacheService.clear();
        return Map.of("status", "Cache cleared");
    }
}
```

---

## 8. Caching

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        return RedisCacheManager.builder(redisConnectionFactory)
            .cacheDefaults(config)
            .build();
    }
}

@Service
public class ProductService {
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) { ... }
    
    @Cacheable(value = "products", key = "#category + '_' + #page")
    public List<Product> getByCategory(String category, int page) { ... }
    
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) { ... }
    
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) { ... }
    
    @CacheEvict(value = "products", allEntries = true)
    public void clearAll() { ... }
}
```

---

## 9. Spring Boot Testing

```java
// Slice test — tests one layer
@DataJpaTest
class UserRepositoryTest {
    @Autowired private TestEntityManager em;
    @Autowired private UserRepository repo;
    
    @Test
    void findByEmail_ShouldReturnUser() {
        User user = new User("test@example.com");
        em.persist(user);
        Optional<User> found = repo.findByEmail("test@example.com");
        assertThat(found).isPresent();
    }
}

// Web layer test
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired private MockMvc mockMvc;
    @MockBean private UserService userService;
    
    @Test
    void getUser_ShouldReturnUser() throws Exception {
        User user = new User(1L, "test@example.com");
        when(userService.findById(1L)).thenReturn(Optional.of(user));
        
        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.email").value("test@example.com"));
    }
}

// Full integration test
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class FullIntegrationTest {
    @Autowired private WebTestClient webTestClient;
    
    @Test
    void createAndRetrieveUser() {
        UserRequest request = new UserRequest("test@example.com");
        
        webTestClient.post().uri("/api/v1/users")
            .bodyValue(request)
            .exchange()
            .expectStatus().isCreated();
    }
}

// Testcontainers
@SpringBootTest
class DatabaseTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

---

## 10. REST API Design

### HATEOAS
```java
@RestController
public class UserController {
    @GetMapping("/api/users/{id}")
    public EntityModel<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return EntityModel.of(user,
            linkTo(methodOn(UserController.class).getUser(id)).withSelfRel(),
            linkTo(methodOn(UserController.class).getAllUsers()).withRel("users"),
            linkTo(methodOn(OrderController.class).getOrdersForUser(id)).withRel("orders")
        );
    }
}
```

### API Versioning
```java
// URL versioning
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 { ... }

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 { ... }
```

### DTO Mapping
```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserDto toDto(User user);
    User toEntity(UserDto dto);
    List<UserDto> toDtoList(List<User> users);
}
```

---

## 11. Messaging

### Kafka
```java
@Service
public class OrderProducer {
    @Autowired private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Value("${kafka.topic.orders}")
    private String topic;
    
    public void sendOrderEvent(OrderEvent event) {
        kafkaTemplate.send(topic, event.getOrderId(), event)
            .addCallback(
                success -> log.info("Sent: {}", event),
                failure -> log.error("Failed to send", failure)
            );
    }
}

@Service
public class OrderConsumer {
    @KafkaListener(topics = "${kafka.topic.orders}", groupId = "order-service")
    public void handleOrder(OrderEvent event) {
        log.info("Received order event: {}", event);
        orderService.process(event);
    }
}
```

### RabbitMQ
```java
@Service
public class MessageService {
    @Autowired private RabbitTemplate rabbitTemplate;
    
    public void sendMessage(String queue, Message message) {
        rabbitTemplate.convertAndSend(queue, message);
    }
}

@Component
public class MessageListener {
    @RabbitListener(queues = "${rabbitmq.queue.orders}")
    public void onMessage(OrderMessage message) {
        // Process message
    }
}
```

---

## 12. Scheduling & Async

```java
@Configuration
@EnableScheduling
@EnableAsync
public class AppConfig {
}

@Service
public class NotificationService {
    @Scheduled(fixedRate = 60_000)
    public void cleanupExpiredSessions() { ... }
    
    @Scheduled(cron = "0 0 2 * * ?") // Every day at 2 AM
    public void generateDailyReports() { ... }
    
    @Async
    public CompletableFuture<String> sendEmailAsync(Email email) {
        emailService.send(email);
        return CompletableFuture.completedFuture("Sent");
    }
}
```

---

## 13. Spring Cloud Basics

### Service Discovery (Eureka)
```yaml
# Eureka Server
server:
  port: 8761
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false

# Eureka Client
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### API Gateway
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - AddRequestHeader=X-Request-ID, ${unique-id}
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
```

### Circuit Breaker (Resilience4j)
```java
@Service
public class PaymentService {
    @CircuitBreaker(name = "payment", fallbackMethod = "fallback")
    @TimeLimiter(name = "payment")
    @Retry(name = "payment")
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest req) {
        return paymentClient.charge(req);
    }
    
    public CompletableFuture<PaymentResult> fallback(PaymentRequest req, Exception e) {
        return CompletableFuture.completedFuture(PaymentResult.pending("Queued"));
    }
}
```

---

## 14. Multi-Tenancy

### Schema-Based Multi-Tenancy
```java
public class MultiTenantConnectionProvider implements ConnectionProvider {
    @Override
    public Connection getConnection() throws SQLException {
        String schema = TenantContext.getCurrentSchema();
        Connection conn = dataSource.getConnection();
        conn.createStatement().execute("SET search_path TO " + schema);
        return conn;
    }
}
```

### Discriminator-Based (Single Schema)
```java
@Entity
@Table(name = "orders")
@Where(clause = "tenant_id = :currentTenantId")
public class Order {
    @Column(name = "tenant_id")
    private String tenantId;
}
```

---

## Spring Boot Coding Questions

### Q1: REST API CRUD with Full Validation
Build a complete REST API with validation, pagination, and error handling.

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
}

@Data
public class ProductRequest {
    @NotBlank
    private String name;
    
    @NotNull
    @DecimalMin("0.01")
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
    public ResponseEntity<Product> create(@Valid @RequestBody ProductRequest req) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(service.create(req));
    }
    
    @GetMapping
    public ResponseEntity<Page<Product>> list(
            @PageableDefault(size = 20, sort = "name") Pageable pageable) {
        return ResponseEntity.ok(service.findAll(pageable));
    }
}
```

### Q2: Async Notification System
Build a notification system that processes notifications asynchronously.

```java
@Service
public class NotificationService {
    @Async("notificationExecutor")
    @Retryable(value = EmailException.class, maxAttempts = 3, 
               backoff = @Backoff(delay = 1000))
    public CompletableFuture<String> sendEmail(Email email) {
        // Send email logic
        emailSender.send(email);
        return CompletableFuture.completedFuture("Sent to " + email.getTo());
    }
}

@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean("notificationExecutor")
    public Executor notificationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("Notification-");
        executor.initialize();
        return executor;
    }
}
```

### Q3: Custom Actuator Endpoint with Health Check
Create a custom health check endpoint for external services.

```java
@Component
@Endpoint(id = "externalHealth")
public class ExternalHealthEndpoint {
    
    @Autowired private RestTemplate restTemplate;
    
    @ReadOperation
    public Map<String, HealthDetail> health() {
        Map<String, HealthDetail> health = new LinkedHashMap<>();
        health.put("database", checkDatabase());
        health.put("redis", checkRedis());
        health.put("email", checkEmailService());
        return health;
    }
    
    @WriteOperation
    public Map<String, String> refresh() {
        healthCache.invalidate();
        return Map.of("status", "Refreshed");
    }
    
    private HealthDetail checkDatabase() {
        try {
            jdbcTemplate.execute("SELECT 1");
            return new HealthDetail(true, "Database connected");
        } catch (Exception e) {
            return new HealthDetail(false, e.getMessage());
        }
    }
}
```

### Q4: Kafka Order Processing Pipeline
Implement an order processing pipeline with Kafka.

```java
@Service
public class OrderPipeline {
    @Autowired private KafkaTemplate<String, Object> kafkaTemplate;
    @Autowired private OrderRepository orderRepository;
    
    @Transactional
    public OrderResponse placeOrder(OrderRequest request) {
        Order order = orderRepository.save(request.toOrder());
        
        kafkaTemplate.send("order-created", 
            new OrderCreatedEvent(order.getId(), request.getItems()));
        
        return OrderResponse.from(order);
    }
}

@Component
public class OrderEventHandler {
    @KafkaListener(topics = "order-created", groupId = "inventory")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Reserve stock
        inventoryService.reserve(event.getOrderId(), event.getItems());
    }
    
    @KafkaListener(topics = "inventory-reserved", groupId = "payment")
    public void handleInventoryReserved(InventoryReservedEvent event) {
        // Process payment
        paymentService.charge(event.getOrderId());
    }
}
```

### Q5: Multi-Tenant Repository
Build a multi-tenant data access layer.

```java
@MappedSuperclass
public abstract class TenantAwareEntity {
    @Column(name = "tenant_id", nullable = false, updatable = false)
    private String tenantId;
    
    @PrePersist
    protected void onCreate() {
        this.tenantId = TenantContext.getCurrentTenant();
    }
}

@Entity
public class Customer extends TenantAwareEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
}

@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    @Query("SELECT c FROM Customer c WHERE c.tenantId = :tenantId AND c.id = :id")
    Optional<Customer> findByIdAndTenant(@Param("id") Long id, 
                                         @Param("tenantId") String tenantId);
}
```

### Q6: Swagger/OpenAPI Integration
Configure and use OpenAPI documentation.

```java
@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("My API")
                .version("1.0.0")
                .description("API Documentation"))
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
            .addSecuritySchemes("bearerAuth", new SecurityScheme()
                .type(SecurityScheme.Type.HTTP)
                .scheme("bearer")
                .bearerFormat("JWT"));
    }
}
```

### Q7: File Upload with Validation
Implement secure file upload service.

```java
@Service
public class FileService {
    @Value("${file.upload.max-size:10485760}")
    private long maxSize;
    
    @Value("${file.upload.allowed-types:image/png,image/jpeg,image/gif}")
    private List<String> allowedTypes;
    
    public FileMetadata upload(MultipartFile file) {
        if (file.getSize() > maxSize) {
            throw new FileTooLargeException("Max size: " + maxSize);
        }
        String contentType = file.getContentType();
        if (contentType == null || !allowedTypes.contains(contentType)) {
            throw new FileTypeNotAllowedException("Type: " + contentType);
        }
        
        String filename = UUID.randomUUID() + "_" + file.getOriginalFilename();
        Path dest = uploadDir.resolve(filename);
        try {
            file.transferTo(dest);
        } catch (IOException e) {
            throw new FileUploadException("Upload failed", e);
        }
        return new FileMetadata(filename, file.getOriginalFilename(), 
            file.getSize(), contentType);
    }
}

@RestController
@RequestMapping("/api/v1/files")
public class FileController {
    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<FileMetadata> upload(
            @RequestParam("file") @Valid MultipartFile file) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(fileService.upload(file));
    }
}
```
