# Microservices Design Patterns

Microservices architecture requires specific design patterns to address challenges like service communication, data consistency, and deployment. This guide covers essential microservices patterns, their implementations, and trade-offs for building scalable, maintainable microservices systems.

## What are Microservices Patterns?

Microservices patterns provide solutions to common challenges in microservices architecture. They address communication between services, data management, deployment, and operational concerns while maintaining loose coupling and high cohesion.

### Pattern Categories
- **Communication Patterns**: Service-to-service communication
- **Data Management Patterns**: Data consistency and sharing
- **Deployment Patterns**: Service deployment and scaling
- **Observability Patterns**: Monitoring and debugging
- **Resilience Patterns**: Fault tolerance and recovery

## Communication Patterns

### 1. API Gateway Pattern

#### Overview
API Gateway acts as a single entry point for all client requests, routing them to appropriate microservices and aggregating responses.

#### Implementation
```java
@SpringBootApplication
@EnableEurekaClient
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}

@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .rewritePath("/api/users/(?<segment>.*)", "/${segment}")
                    .circuitBreaker(c -> c
                        .setName("user-service-circuit")
                        .setFallbackUri("forward:/fallback/user"))
                    .requestRateLimiter(c -> c
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver())))
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .rewritePath("/api/orders/(?<segment>.*)", "/${segment}")
                    .circuitBreaker(c -> c
                        .setName("order-service-circuit")
                        .setFallbackUri("forward:/fallback/order"))))
            .build();
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(10, 20, 1); // replenishRate, burstCapacity, requestedTokens
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.just(exchange.getRequest().getHeaders()
            .getFirst("X-User-Id"));
    }
}

@RestController
public class FallbackController {
    
    @GetMapping("/fallback/user")
    public ResponseEntity<String> userServiceFallback() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body("User service is currently unavailable. Please try again later.");
    }
    
    @GetMapping("/fallback/order")
    public ResponseEntity<String> orderServiceFallback() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body("Order service is currently unavailable. Please try again later.");
    }
}
```

### 2. Service-to-Service Communication Patterns

#### Synchronous Communication with Feign Client
```java
@FeignClient(name = "user-service", fallback = UserServiceFallback.class)
public interface UserServiceClient {
    
    @GetMapping("/users/{userId}")
    User getUserById(@PathVariable("userId") Long userId);
    
    @GetMapping("/users")
    List<User> getAllUsers(@RequestParam("page") int page, 
                          @RequestParam("size") int size);
    
    @PostMapping("/users")
    User createUser(@RequestBody CreateUserRequest request);
    
    @PutMapping("/users/{userId}")
    User updateUser(@PathVariable("userId") Long userId, 
                   @RequestBody UpdateUserRequest request);
    
    @DeleteMapping("/users/{userId}")
    void deleteUser(@PathVariable("userId") Long userId);
}

@Component
public class UserServiceFallback implements UserServiceClient {
    
    @Override
    public User getUserById(Long userId) {
        return new User(userId, "Unknown User", "fallback@example.com");
    }
    
    @Override
    public List<User> getAllUsers(int page, int size) {
        return Collections.emptyList();
    }
    
    @Override
    public User createUser(CreateUserRequest request) {
        throw new ServiceUnavailableException("User service is unavailable");
    }
    
    @Override
    public User updateUser(Long userId, UpdateUserRequest request) {
        throw new ServiceUnavailableException("User service is unavailable");
    }
    
    @Override
    public void deleteUser(Long userId) {
        // No-op for delete operations
    }
}
```

#### Asynchronous Communication with Message Queues
```java
@Service
public class OrderEventPublisher {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publishOrderCreatedEvent(Order order) {
        OrderCreatedEvent event = new OrderCreatedEvent(order.getId(), order.getUserId(), 
                                                      order.getTotalAmount(), Instant.now());
        
        kafkaTemplate.send("order-events", order.getId().toString(), event);
        
        logger.info("Published order created event for order {}", order.getId());
    }
    
    public void publishOrderStatusChangedEvent(Long orderId, OrderStatus oldStatus, 
                                             OrderStatus newStatus) {
        OrderStatusChangedEvent event = new OrderStatusChangedEvent(orderId, oldStatus, 
                                                                  newStatus, Instant.now());
        
        kafkaTemplate.send("order-events", orderId.toString(), event);
        
        logger.info("Published order status changed event for order {}", orderId);
    }
}

@Service
public class OrderEventConsumer {
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @KafkaListener(topics = "order-events", groupId = "order-processing")
    public void handleOrderEvents(OrderEvent event) {
        logger.info("Received order event: {}", event);
        
        if (event instanceof OrderCreatedEvent) {
            handleOrderCreated((OrderCreatedEvent) event);
        } else if (event instanceof OrderStatusChangedEvent) {
            handleOrderStatusChanged((OrderStatusChangedEvent) event);
        }
    }
    
    private void handleOrderCreated(OrderCreatedEvent event) {
        // Send order confirmation notification
        notificationService.sendOrderConfirmation(event.getUserId(), event.getOrderId());
        
        // Reserve inventory
        inventoryService.reserveInventoryForOrder(event.getOrderId());
    }
    
    private void handleOrderStatusChanged(OrderStatusChangedEvent event) {
        // Send status update notification
        notificationService.sendOrderStatusUpdate(event.getOrderId(), 
                                                event.getNewStatus());
        
        if (event.getNewStatus() == OrderStatus.SHIPPED) {
            // Update inventory when order is shipped
            inventoryService.updateInventoryOnShipment(event.getOrderId());
        }
    }
}
```

## Data Management Patterns

### 1. Database per Service Pattern

#### Overview
Each microservice has its own database, ensuring loose coupling and independent scaling.

#### Implementation
```java
@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.orderservice.repository",
    entityManagerFactoryRef = "orderEntityManagerFactory",
    transactionManagerRef = "orderTransactionManager"
)
public class OrderServiceDatabaseConfig {
    
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.order")
    public DataSource orderDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public LocalContainerEntityManagerFactoryBean orderEntityManagerFactory(
            @Qualifier("orderDataSource") DataSource dataSource,
            EntityManagerFactoryBuilder builder) {
        
        return builder
            .dataSource(dataSource)
            .packages("com.example.orderservice.entity")
            .persistenceUnit("order")
            .build();
    }
    
    @Bean
    public PlatformTransactionManager orderTransactionManager(
            @Qualifier("orderEntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        
        return new JpaTransactionManager(entityManagerFactory);
    }
}
```

### 2. Saga Pattern for Distributed Transactions

#### Orchestration-Based Saga
```java
@Service
public class OrderSagaOrchestrator {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private SagaRepository sagaRepository;
    
    public SagaResult processOrderSaga(CreateOrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        OrderSaga saga = new OrderSaga(sagaId, request);
        sagaRepository.save(saga);
        
        try {
            // Step 1: Create order
            Order order = orderService.createOrder(request);
            saga.setOrderId(order.getId());
            saga.setStatus(SagaStatus.ORDER_CREATED);
            sagaRepository.save(saga);
            
            // Step 2: Reserve inventory
            inventoryService.reserveInventory(order.getItems());
            saga.setStatus(SagaStatus.INVENTORY_RESERVED);
            sagaRepository.save(saga);
            
            // Step 3: Process payment
            PaymentResult payment = paymentService.processPayment(order.getTotalAmount(), 
                                                                request.getPaymentInfo());
            saga.setPaymentId(payment.getId());
            saga.setStatus(SagaStatus.PAYMENT_PROCESSED);
            sagaRepository.save(saga);
            
            // Step 4: Send confirmation
            notificationService.sendOrderConfirmation(order.getUserId(), order.getId());
            saga.setStatus(SagaStatus.COMPLETED);
            sagaRepository.save(saga);
            
            return SagaResult.success(order);
            
        } catch (Exception e) {
            logger.error("Saga {} failed at step {}", sagaId, saga.getStatus(), e);
            
            // Compensate based on current state
            compensateSaga(saga);
            
            return SagaResult.failure("Order processing failed: " + e.getMessage());
        }
    }
    
    private void compensateSaga(OrderSaga saga) {
        try {
            switch (saga.getStatus()) {
                case PAYMENT_PROCESSED:
                    paymentService.refundPayment(saga.getPaymentId());
                case INVENTORY_RESERVED:
                    inventoryService.releaseInventoryReservation(saga.getOrderId());
                case ORDER_CREATED:
                    orderService.cancelOrder(saga.getOrderId());
                default:
                    break;
            }
            
            saga.setStatus(SagaStatus.COMPENSATED);
            sagaRepository.save(saga);
            
        } catch (Exception e) {
            logger.error("Compensation failed for saga {}", saga.getId(), e);
            saga.setStatus(SagaStatus.COMPENSATION_FAILED);
            sagaRepository.save(saga);
        }
    }
}
```

#### Choreography-Based Saga
```java
@Service
public class OrderChoreographySaga {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void startOrderSaga(CreateOrderRequest request) {
        // Step 1: Create order and publish event
        Order order = orderService.createOrder(request);
        
        OrderCreatedEvent event = new OrderCreatedEvent(order.getId(), 
                                                      order.getUserId(),
                                                      order.getItems(),
                                                      order.getTotalAmount());
        
        kafkaTemplate.send("saga-events", "order-created", event);
        
        logger.info("Started order saga for order {}", order.getId());
    }
}

@Service
public class InventorySagaParticipant {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @KafkaListener(topics = "saga-events", groupId = "inventory-saga")
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            // Reserve inventory
            inventoryService.reserveInventory(event.getOrderId(), event.getItems());
            
            // Publish success event
            InventoryReservedEvent successEvent = new InventoryReservedEvent(
                event.getOrderId(), event.getUserId(), event.getTotalAmount());
            
            kafkaTemplate.send("saga-events", "inventory-reserved", successEvent);
            
        } catch (Exception e) {
            // Publish failure event
            SagaFailedEvent failureEvent = new SagaFailedEvent(
                event.getOrderId(), "inventory-reservation", e.getMessage());
            
            kafkaTemplate.send("saga-events", "saga-failed", failureEvent);
        }
    }
    
    @KafkaListener(topics = "saga-events", groupId = "inventory-saga")
    public void handleSagaFailed(SagaFailedEvent event) {
        // Release inventory reservation
        inventoryService.releaseInventoryReservation(event.getOrderId());
        
        logger.info("Released inventory reservation for failed order {}", event.getOrderId());
    }
}
```

### 3. CQRS Pattern

#### Command and Query Responsibility Segregation
```java
// Commands - Write operations
public interface Command<T> {
    T execute();
}

public class CreateOrderCommand implements Command<Order> {
    
    private final CreateOrderRequest request;
    private final OrderRepository repository;
    
    public CreateOrderCommand(CreateOrderRequest request, OrderRepository repository) {
        this.request = request;
        this.repository = repository;
    }
    
    @Override
    public Order execute() {
        Order order = new Order(request.getUserId(), request.getItems(), Instant.now());
        return repository.save(order);
    }
}

public class UpdateOrderStatusCommand implements Command<Void> {
    
    private final Long orderId;
    private final OrderStatus newStatus;
    private final OrderRepository repository;
    
    public UpdateOrderStatusCommand(Long orderId, OrderStatus newStatus, OrderRepository repository) {
        this.orderId = orderId;
        this.newStatus = newStatus;
        this.repository = repository;
    }
    
    @Override
    public Void execute() {
        Order order = repository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        order.setStatus(newStatus);
        order.setUpdatedAt(Instant.now());
        
        repository.save(order);
        return null;
    }
}

// Queries - Read operations
public interface Query<T> {
    T execute();
}

public class GetOrderByIdQuery implements Query<OrderDto> {
    
    private final Long orderId;
    private final OrderReadRepository readRepository;
    
    public GetOrderByIdQuery(Long orderId, OrderReadRepository readRepository) {
        this.orderId = orderId;
        this.readRepository = readRepository;
    }
    
    @Override
    public OrderDto execute() {
        return readRepository.findOrderById(orderId);
    }
}

public class GetOrdersByUserQuery implements Query<List<OrderSummaryDto>> {
    
    private final Long userId;
    private final OrderReadRepository readRepository;
    
    public GetOrdersByUserQuery(Long userId, OrderReadRepository readRepository) {
        this.userId = userId;
        this.readRepository = readRepository;
    }
    
    @Override
    public List<OrderSummaryDto> execute() {
        return readRepository.findOrderSummariesByUserId(userId);
    }
}

// Command Handler
@Service
public class OrderCommandHandler {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private KafkaTemplate<String, Object> eventPublisher;
    
    @Transactional
    public <T> T handle(Command<T> command) {
        T result = command.execute();
        
        // Publish domain events
        if (result instanceof Order) {
            Order order = (Order) result;
            OrderCreatedEvent event = new OrderCreatedEvent(order.getId(), 
                                                          order.getUserId(), 
                                                          order.getTotalAmount());
            eventPublisher.send("order-events", order.getId().toString(), event);
        }
        
        return result;
    }
}

// Query Handler
@Service
public class OrderQueryHandler {
    
    @Autowired
    private OrderReadRepository readRepository;
    
    @Autowired
    private CacheManager cacheManager;
    
    public <T> T handle(Query<T> query) {
        // Try cache first for read queries
        String cacheKey = query.getClass().getSimpleName() + ":" + query.hashCode();
        Cache cache = cacheManager.getCache("order-queries");
        
        T result = cache.get(cacheKey, () -> query.execute());
        return result;
    }
}
```

## Deployment Patterns

### 1. Blue-Green Deployment

#### Overview
Two identical production environments (blue and green) where one serves live traffic while the other is updated and tested.

#### Implementation
```java
@Configuration
public class BlueGreenDeploymentConfig {
    
    @Value("${deployment.color:blue}")
    private String deploymentColor;
    
    @Bean
    public DeploymentMetadata deploymentMetadata() {
        return new DeploymentMetadata(deploymentColor, 
                                    System.getenv("BUILD_VERSION"),
                                    System.getenv("BUILD_TIME"));
    }
    
    @Bean
    public HealthIndicator deploymentHealthIndicator(DeploymentMetadata metadata) {
        return () -> {
            boolean healthy = "blue".equals(metadata.getColor()) || "green".equals(metadata.getColor());
            return healthy ? Health.up()
                          .withDetail("deployment_color", metadata.getColor())
                          .withDetail("version", metadata.getVersion())
                          .build()
                         : Health.down()
                          .withDetail("error", "Invalid deployment color: " + metadata.getColor())
                          .build();
        };
    }
}

@RestController
@RequestMapping("/deployment")
public class DeploymentController {
    
    @Autowired
    private DeploymentMetadata deploymentMetadata;
    
    @Autowired
    private TrafficRouter trafficRouter;
    
    @GetMapping("/status")
    public DeploymentStatus getDeploymentStatus() {
        return new DeploymentStatus(
            deploymentMetadata.getColor(),
            deploymentMetadata.getVersion(),
            trafficRouter.getCurrentTrafficPercentage(deploymentMetadata.getColor()),
            deploymentMetadata.getDeployTime()
        );
    }
    
    @PostMapping("/switch")
    public ResponseEntity<String> switchDeployment(@RequestParam String targetColor) {
        if (!isValidColor(targetColor)) {
            return ResponseEntity.badRequest()
                .body("Invalid deployment color: " + targetColor);
        }
        
        try {
            trafficRouter.switchToColor(targetColor);
            return ResponseEntity.ok("Successfully switched to " + targetColor + " deployment");
        } catch (Exception e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Failed to switch deployment: " + e.getMessage());
        }
    }
    
    private boolean isValidColor(String color) {
        return "blue".equals(color) || "green".equals(color);
    }
}
```

### 2. Canary Deployment

#### Overview
Gradually roll out new version to a small percentage of users before full deployment.

#### Implementation
```java
@Service
public class CanaryDeploymentService {
    
    @Autowired
    private LoadBalancer loadBalancer;
    
    @Autowired
    private MetricsService metricsService;
    
    @Autowired
    private AlertService alertService;
    
    private final Map<String, CanaryDeployment> activeCanaries = new ConcurrentHashMap<>();
    
    public void startCanaryDeployment(String serviceName, String newVersion, 
                                    double initialTrafficPercentage) {
        
        CanaryDeployment canary = new CanaryDeployment(
            serviceName, newVersion, initialTrafficPercentage, Instant.now());
        
        activeCanaries.put(serviceName, canary);
        
        // Configure load balancer for canary traffic
        loadBalancer.setTrafficPercentage(serviceName, newVersion, initialTrafficPercentage);
        
        // Start monitoring
        startCanaryMonitoring(canary);
        
        logger.info("Started canary deployment for {} version {} with {}% traffic", 
                   serviceName, newVersion, initialTrafficPercentage);
    }
    
    public void increaseCanaryTraffic(String serviceName, double newPercentage) {
        CanaryDeployment canary = activeCanaries.get(serviceName);
        
        if (canary == null) {
            throw new IllegalArgumentException("No active canary deployment for " + serviceName);
        }
        
        if (newPercentage > 100.0) {
            newPercentage = 100.0;
        }
        
        loadBalancer.setTrafficPercentage(serviceName, canary.getNewVersion(), newPercentage);
        canary.setTrafficPercentage(newPercentage);
        
        logger.info("Increased canary traffic for {} to {}%", serviceName, newPercentage);
        
        if (newPercentage >= 100.0) {
            completeCanaryDeployment(serviceName);
        }
    }
    
    public void rollbackCanaryDeployment(String serviceName) {
        CanaryDeployment canary = activeCanaries.get(serviceName);
        
        if (canary == null) {
            logger.warn("No active canary deployment to rollback for {}", serviceName);
            return;
        }
        
        // Route all traffic back to old version
        loadBalancer.setTrafficPercentage(serviceName, canary.getOldVersion(), 100.0);
        
        // Stop monitoring
        stopCanaryMonitoring(canary);
        
        activeCanaries.remove(serviceName);
        
        logger.info("Rolled back canary deployment for {}", serviceName);
        
        // Send alert
        alertService.sendAlert("Canary Deployment Rolled Back",
                             "Canary deployment for " + serviceName + " was rolled back");
    }
    
    private void completeCanaryDeployment(String serviceName) {
        CanaryDeployment canary = activeCanaries.remove(serviceName);
        
        // Stop monitoring
        stopCanaryMonitoring(canary);
        
        logger.info("Completed canary deployment for {}", serviceName);
        
        // Send notification
        alertService.sendNotification("Canary Deployment Completed",
                                    "Canary deployment for " + serviceName + " is now fully deployed");
    }
    
    private void startCanaryMonitoring(CanaryDeployment canary) {
        // Schedule monitoring task
        Executors.newSingleThreadScheduledExecutor()
            .scheduleAtFixedRate(() -> monitorCanaryHealth(canary), 0, 30, TimeUnit.SECONDS);
    }
    
    private void stopCanaryMonitoring(CanaryDeployment canary) {
        // Implementation to stop monitoring
    }
    
    private void monitorCanaryHealth(CanaryDeployment canary) {
        // Check error rates, latency, etc.
        double errorRate = metricsService.getErrorRate(canary.getServiceName(), canary.getNewVersion());
        double latency = metricsService.getAverageLatency(canary.getServiceName(), canary.getNewVersion());
        
        // Define thresholds
        if (errorRate > 0.05 || latency > 2000) { // 5% error rate or 2s latency
            logger.warn("Canary health check failed for {}: errorRate={}, latency={}", 
                       canary.getServiceName(), errorRate, latency);
            
            // Auto-rollback if configured
            if (canary.isAutoRollbackEnabled()) {
                rollbackCanaryDeployment(canary.getServiceName());
            } else {
                // Send alert for manual intervention
                alertService.sendAlert("Canary Health Check Failed",
                                     "Canary deployment for " + canary.getServiceName() + 
                                     " is experiencing issues. Manual intervention required.");
            }
        }
    }
}
```

## Observability Patterns

### 1. Distributed Tracing Pattern

#### Overview
Track requests across multiple microservices to understand performance bottlenecks and failure points.

#### Implementation
```java
@Configuration
public class TracingConfig {
    
    @Bean
    public Tracer tracer() {
        return TracerFactory.create();
    }
    
    @Bean
    public SpanReporter spanReporter() {
        return new ZipkinSpanReporter("http://zipkin:9411/api/v2/spans");
    }
    
    @Bean
    public TracingFilter tracingFilter(Tracer tracer) {
        return new TracingFilter(tracer, Collections.singletonList(spanReporter()));
    }
}

@Service
public class OrderService {
    
    @Autowired
    private Tracer tracer;
    
    @Autowired
    private UserServiceClient userService;
    
    @Autowired
    private InventoryService inventoryService;
    
    public Order createOrder(CreateOrderRequest request) {
        Span span = tracer.buildSpan("createOrder").start();
        span.setTag("userId", request.getUserId().toString());
        span.setTag("itemCount", String.valueOf(request.getItems().size()));
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            // Validate user
            span.log("Validating user");
            User user = userService.getUserById(request.getUserId());
            
            if (user == null) {
                span.log("User not found");
                span.setTag("error", true);
                throw new UserNotFoundException(request.getUserId());
            }
            
            // Check inventory
            span.log("Checking inventory");
            for (OrderItem item : request.getItems()) {
                boolean available = inventoryService.checkAvailability(item.getProductId(), 
                                                                     item.getQuantity());
                if (!available) {
                    span.log("Insufficient inventory for product " + item.getProductId());
                    span.setTag("error", true);
                    throw new InsufficientInventoryException(item.getProductId());
                }
            }
            
            // Create order
            span.log("Creating order");
            Order order = new Order(request.getUserId(), request.getItems(), Instant.now());
            
            span.setTag("orderId", order.getId().toString());
            span.log("Order created successfully");
            
            return order;
            
        } catch (Exception e) {
            span.setTag("error", true);
            span.log(e.getMessage());
            throw e;
        } finally {
            span.finish();
        }
    }
}
```

### 2. Log Aggregation Pattern

#### Centralized Logging with Correlation IDs
```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    
    private static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    private static final String CORRELATION_ID_KEY = "correlationId";
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                  HttpServletResponse response, 
                                  FilterChain filterChain) throws ServletException, IOException {
        
        String correlationId = request.getHeader(CORRELATION_ID_HEADER);
        
        if (correlationId == null || correlationId.isEmpty()) {
            correlationId = UUID.randomUUID().toString();
        }
        
        // Set in MDC for logging
        MDC.put(CORRELATION_ID_KEY, correlationId);
        
        // Add to response header
        response.setHeader(CORRELATION_ID_HEADER, correlationId);
        
        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(CORRELATION_ID_KEY);
        }
    }
}

@Service
public class OrderProcessingService {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderProcessingService.class);
    
    @Autowired
    private UserServiceClient userService;
    
    @Autowired
    private PaymentService paymentService;
    
    public void processOrder(Long orderId) {
        String correlationId = MDC.get("correlationId");
        
        logger.info("Starting order processing for order {} (correlationId: {})", 
                   orderId, correlationId);
        
        try {
            // Get user details
            logger.debug("Retrieving user details for order {}", orderId);
            User user = userService.getUserById(getUserIdFromOrder(orderId));
            
            // Process payment
            logger.debug("Processing payment for order {}", orderId);
            PaymentResult payment = paymentService.processPayment(orderId);
            
            logger.info("Order {} processed successfully (correlationId: {})", 
                       orderId, correlationId);
            
        } catch (Exception e) {
            logger.error("Failed to process order {} (correlationId: {}): {}", 
                        orderId, correlationId, e.getMessage(), e);
            throw e;
        }
    }
}
```

## Resilience Patterns

### 1. Circuit Breaker Pattern

#### Hystrix Implementation
```java
@Service
public class OrderServiceClient {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @HystrixCommand(
        commandKey = "getOrder",
        fallbackMethod = "getOrderFallback",
        commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000"),
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "5000")
        },
        threadPoolProperties = {
            @HystrixProperty(name = "coreSize", value = "5"),
            @HystrixProperty(name = "maxQueueSize", value = "10")
        }
    )
    public Order getOrder(Long orderId) {
        String url = "http://order-service/orders/" + orderId;
        
        ResponseEntity<Order> response = restTemplate.getForEntity(url, Order.class);
        return response.getBody();
    }
    
    public Order getOrderFallback(Long orderId) {
        logger.warn("Circuit breaker activated for getOrder, returning fallback for order {}", orderId);
        
        // Return cached or default order
        return new Order(orderId, null, Collections.emptyList(), OrderStatus.UNKNOWN);
    }
    
    @HystrixCommand(
        commandKey = "createOrder",
        fallbackMethod = "createOrderFallback"
    )
    public Order createOrder(CreateOrderRequest request) {
        String url = "http://order-service/orders";
        
        ResponseEntity<Order> response = restTemplate.postForEntity(url, request, Order.class);
        return response.getBody();
    }
    
    public Order createOrderFallback(CreateOrderRequest request) {
        logger.warn("Circuit breaker activated for createOrder, queuing request for later processing");
        
        // Queue request for later processing
        orderCreationQueue.add(request);
        
        throw new ServiceUnavailableException("Order service is currently unavailable. Your order will be processed when the service is restored.");
    }
}
```

### 2. Bulkhead Pattern

#### Thread Pool Isolation
```java
@Configuration
public class BulkheadConfig {
    
    @Bean
    public ThreadPoolExecutor orderServiceExecutor() {
        return new ThreadPoolExecutor(
            5, 20, 60, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(50),
            new ThreadFactoryBuilder().setNameFormat("order-service-%d").build(),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
    
    @Bean
    public ThreadPoolExecutor paymentServiceExecutor() {
        return new ThreadPoolExecutor(
            3, 10, 60, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(30),
            new ThreadFactoryBuilder().setNameFormat("payment-service-%d").build(),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
    
    @Bean
    public ThreadPoolExecutor notificationServiceExecutor() {
        return new ThreadPoolExecutor(
            2, 5, 60, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(20),
            new ThreadFactoryBuilder().setNameFormat("notification-service-%d").build(),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}

@Service
public class IsolatedOrderService {
    
    @Autowired
    private ThreadPoolExecutor executor;
    
    @Autowired
    private OrderRepository orderRepository;
    
    public CompletableFuture<Order> createOrderAsync(CreateOrderRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            // This operation is isolated in its own thread pool
            validateOrderRequest(request);
            Order order = new Order(request.getUserId(), request.getItems(), Instant.now());
            return orderRepository.save(order);
        }, executor);
    }
    
    public CompletableFuture<Order> getOrderAsync(Long orderId) {
        return CompletableFuture.supplyAsync(() -> 
            orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId)), 
            executor);
    }
}
```

## Best Practices

### Service Design
- **Single Responsibility**: Each microservice should have one clear responsibility
- **API Design**: Use RESTful APIs with proper versioning
- **Contract Testing**: Validate service contracts with consumer-driven contracts
- **Documentation**: Maintain up-to-date API documentation

### Communication
- **Synchronous vs Asynchronous**: Choose appropriate communication patterns
- **Service Discovery**: Use dynamic service discovery instead of hardcoded addresses
- **Load Balancing**: Implement client-side load balancing
- **Timeouts and Retries**: Configure appropriate timeouts and retry policies

### Data Management
- **Eventual Consistency**: Accept eventual consistency for better performance
- **Saga Pattern**: Use sagas for distributed transactions
- **CQRS**: Separate read and write concerns when appropriate
- **Data Migration**: Plan for schema changes across services

### Deployment and Operations
- **CI/CD Pipelines**: Implement automated deployment pipelines
- **Blue-Green Deployments**: Use blue-green or canary deployments for zero-downtime
- **Health Checks**: Implement proper health check endpoints
- **Monitoring**: Set up comprehensive monitoring and alerting

### Security
- **Authentication**: Implement proper authentication between services
- **Authorization**: Use role-based access control
- **API Gateway**: Centralize security concerns in API gateway
- **Secrets Management**: Securely manage service credentials and secrets

## Conclusion

Microservices design patterns provide proven solutions to common challenges in distributed systems. By applying appropriate patterns for communication, data management, deployment, and resilience, teams can build scalable, maintainable microservices architectures.

**Key Takeaways:**
- **API Gateway**: Single entry point for client requests
- **Service Discovery**: Dynamic service location and registration
- **Saga Pattern**: Managing distributed transactions
- **CQRS**: Separating read and write concerns
- **Circuit Breaker**: Preventing cascading failures
- **Deployment Patterns**: Blue-green and canary deployments

Effective use of microservices patterns requires understanding the trade-offs and applying them appropriately based on specific use cases and requirements.
