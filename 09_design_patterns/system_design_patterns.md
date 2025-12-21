# System Design Patterns

System design patterns provide proven solutions to common architectural challenges in building scalable, reliable distributed systems. This guide covers essential system design patterns used by major tech companies, with implementation examples and trade-off analysis.

## What are System Design Patterns?

System design patterns are reusable solutions to common problems in distributed system architecture. They provide templates for solving scalability, reliability, performance, and maintainability challenges that arise when building large-scale systems.

### Pattern Categories
- **Scalability Patterns**: Handling growth and load
- **Reliability Patterns**: Ensuring system availability and fault tolerance
- **Performance Patterns**: Optimizing speed and efficiency
- **Data Management Patterns**: Handling data at scale
- **Communication Patterns**: Service interaction and data flow

## Scalability Patterns

### 1. Load Balancing Pattern

#### Overview
Distribute incoming requests across multiple service instances to prevent overload and ensure optimal resource utilization.

#### Implementation Strategies

##### Hardware Load Balancers
```java
@Configuration
public class HardwareLoadBalancerConfig {
    
    @Bean
    public LoadBalancer hardwareLoadBalancer() {
        return new HardwareLoadBalancer.Builder()
            .withAlgorithm(LoadBalancingAlgorithm.LEAST_CONNECTIONS)
            .withHealthChecks(HealthCheckConfig.builder()
                .withInterval(Duration.ofSeconds(30))
                .withTimeout(Duration.ofSeconds(5))
                .withUnhealthyThreshold(3)
                .withHealthyThreshold(2)
                .build())
            .withSessionAffinity(SessionAffinity.NONE)
            .withSslTermination(true)
            .build();
    }
}
```

##### Software Load Balancers
```java
@Configuration
public class SoftwareLoadBalancerConfig {
    
    @Bean
    public LoadBalancer softwareLoadBalancer() {
        return new NginxLoadBalancer.Builder()
            .withUpstreamServers(List.of(
                new UpstreamServer("app-server-1", 8080, 10), // weight: 10
                new UpstreamServer("app-server-2", 8080, 10),
                new UpstreamServer("app-server-3", 8080, 5)   // weight: 5
            ))
            .withAlgorithm(LoadBalancingAlgorithm.WEIGHTED_ROUND_ROBIN)
            .withHealthChecks(new HttpHealthCheck("/health", 200))
            .withRateLimiting(new RateLimit(1000, Duration.ofMinutes(1))) // 1000 req/min
            .build();
    }
}
```

##### DNS-Based Load Balancing
```java
@Service
public class DnsLoadBalancer {
    
    private final Map<String, List<String>> serviceEndpoints = new ConcurrentHashMap<>();
    private final Random random = new Random();
    
    public void registerService(String serviceName, String endpoint) {
        serviceEndpoints.computeIfAbsent(serviceName, k -> new ArrayList<>()).add(endpoint);
    }
    
    public void unregisterService(String serviceName, String endpoint) {
        List<String> endpoints = serviceEndpoints.get(serviceName);
        if (endpoints != null) {
            endpoints.remove(endpoint);
        }
    }
    
    public String resolveService(String serviceName) {
        List<String> endpoints = serviceEndpoints.get(serviceName);
        if (endpoints == null || endpoints.isEmpty()) {
            throw new ServiceUnavailableException("No endpoints available for " + serviceName);
        }
        
        // Simple random selection
        return endpoints.get(random.nextInt(endpoints.size()));
    }
    
    public List<String> resolveAllEndpoints(String serviceName) {
        return new ArrayList<>(serviceEndpoints.getOrDefault(serviceName, Collections.emptyList()));
    }
}
```

### 2. Database Sharding Pattern

#### Overview
Split large databases into smaller, more manageable pieces called shards, distributed across multiple servers.

#### Sharding Strategies

##### Hash-Based Sharding
```java
public class HashBasedShardingStrategy implements ShardingStrategy {
    
    private final int shardCount;
    private final ConsistentHashRouter router;
    
    public HashBasedShardingStrategy(int shardCount) {
        this.shardCount = shardCount;
        this.router = new ConsistentHashRouter();
        
        // Add virtual nodes for each shard
        for (int i = 0; i < shardCount; i++) {
            for (int j = 0; j < 100; j++) { // 100 virtual nodes per shard
                router.addNode(new ShardNode("shard-" + i, i));
            }
        }
    }
    
    @Override
    public String getShardKey(Object entity) {
        // Use entity ID or primary key for sharding
        String key = extractKey(entity);
        return "shard-" + Math.abs(key.hashCode() % shardCount);
    }
    
    @Override
    public String getShardForKey(String key) {
        ShardNode node = router.getNode(key);
        return node.getShardId();
    }
    
    private String extractKey(Object entity) {
        // Extract sharding key from entity
        // This would use reflection or annotations
        return entity.toString(); // Simplified
    }
    
    public static class ShardNode {
        private final String id;
        private final String shardId;
        
        public ShardNode(String id, int shardIndex) {
            this.id = id;
            this.shardId = "shard-" + shardIndex;
        }
        
        public String getId() { return id; }
        public String getShardId() { return shardId; }
    }
}
```

##### Range-Based Sharding
```java
public class RangeBasedShardingStrategy implements ShardingStrategy {
    
    private final List<ShardRange> ranges;
    
    public RangeBasedShardingStrategy(List<ShardRange> ranges) {
        this.ranges = new ArrayList<>(ranges);
        // Ensure ranges don't overlap and cover all values
        validateRanges(ranges);
    }
    
    @Override
    public String getShardKey(Object entity) {
        Comparable key = extractComparableKey(entity);
        return findShardForKey(key);
    }
    
    @Override
    public String getShardForKey(String key) {
        try {
            Comparable comparableKey = parseKey(key);
            return findShardForKey(comparableKey);
        } catch (Exception e) {
            throw new IllegalArgumentException("Invalid key format: " + key, e);
        }
    }
    
    private String findShardForKey(Comparable key) {
        for (ShardRange range : ranges) {
            if (range.contains(key)) {
                return range.getShardId();
            }
        }
        throw new IllegalArgumentException("No shard found for key: " + key);
    }
    
    private Comparable extractComparableKey(Object entity) {
        // Extract comparable key from entity
        // This would use reflection or annotations
        if (entity instanceof User) {
            return ((User) entity).getId();
        }
        return entity.toString(); // Simplified
    }
    
    private Comparable parseKey(String key) {
        // Parse string key to comparable
        try {
            return Long.parseLong(key);
        } catch (NumberFormatException e) {
            return key;
        }
    }
    
    private void validateRanges(List<ShardRange> ranges) {
        // Ensure ranges are valid and non-overlapping
        for (int i = 0; i < ranges.size() - 1; i++) {
            ShardRange current = ranges.get(i);
            ShardRange next = ranges.get(i + 1);
            
            if (current.overlaps(next)) {
                throw new IllegalArgumentException("Overlapping ranges: " + current + " and " + next);
            }
        }
    }
    
    public static class ShardRange {
        private final Comparable start;
        private final Comparable end;
        private final String shardId;
        
        public ShardRange(Comparable start, Comparable end, String shardId) {
            this.start = start;
            this.end = end;
            this.shardId = shardId;
        }
        
        public boolean contains(Comparable value) {
            return start.compareTo(value) <= 0 && value.compareTo(end) <= 0;
        }
        
        public boolean overlaps(ShardRange other) {
            return this.start.compareTo(other.end) <= 0 && other.start.compareTo(this.end) <= 0;
        }
        
        public String getShardId() { return shardId; }
    }
}
```

#### Shard Management
```java
@Service
public class ShardManager {
    
    @Autowired
    private ShardMetadataRepository metadataRepository;
    
    @Autowired
    private ShardMigrationService migrationService;
    
    public void addShard(ShardConfig config) {
        // Validate new shard configuration
        validateShardConfig(config);
        
        // Create shard metadata
        ShardMetadata metadata = new ShardMetadata();
        metadata.setShardId(config.getShardId());
        metadata.setRangeStart(config.getRangeStart());
        metadata.setRangeEnd(config.getRangeEnd());
        metadata.setStatus(ShardStatus.CREATING);
        
        metadataRepository.save(metadata);
        
        // Initialize shard
        initializeShard(config);
        
        // Update routing tables
        updateRoutingTables(config);
        
        metadata.setStatus(ShardStatus.ACTIVE);
        metadataRepository.save(metadata);
    }
    
    public void splitShard(String shardId, Comparable splitKey) {
        ShardMetadata existingShard = metadataRepository.findById(shardId)
            .orElseThrow(() -> new ShardNotFoundException(shardId));
        
        // Validate split key is within shard range
        if (splitKey.compareTo(existingShard.getRangeStart()) <= 0 || 
            splitKey.compareTo(existingShard.getRangeEnd()) >= 0) {
            throw new IllegalArgumentException("Split key must be within shard range");
        }
        
        // Create two new shards
        String shard1Id = shardId + "_a";
        String shard2Id = shardId + "_b";
        
        ShardMetadata shard1 = new ShardMetadata();
        shard1.setShardId(shard1Id);
        shard1.setRangeStart(existingShard.getRangeStart());
        shard1.setRangeEnd(splitKey);
        
        ShardMetadata shard2 = new ShardMetadata();
        shard2.setShardId(shard2Id);
        shard2.setRangeStart(splitKey);
        shard2.setRangeEnd(existingShard.getRangeEnd());
        
        // Save new shards
        metadataRepository.save(shard1);
        metadataRepository.save(shard2);
        
        // Migrate data
        migrationService.migrateData(existingShard, List.of(shard1, shard2), splitKey);
        
        // Update routing and remove old shard
        updateRoutingTables(List.of(shard1, shard2));
        metadataRepository.delete(existingShard);
    }
    
    private void validateShardConfig(ShardConfig config) {
        // Validate shard configuration
        if (config.getRangeStart().compareTo(config.getRangeEnd()) >= 0) {
            throw new IllegalArgumentException("Range start must be less than range end");
        }
        
        // Check for overlapping ranges
        List<ShardMetadata> existingShards = metadataRepository.findAll();
        for (ShardMetadata existing : existingShards) {
            if (rangesOverlap(config, existing)) {
                throw new IllegalArgumentException("Shard range overlaps with existing shard");
            }
        }
    }
    
    private boolean rangesOverlap(ShardConfig config, ShardMetadata existing) {
        return config.getRangeStart().compareTo(existing.getRangeEnd()) < 0 && 
               config.getRangeEnd().compareTo(existing.getRangeStart()) > 0;
    }
    
    private void initializeShard(ShardConfig config) {
        // Initialize database schema, indexes, etc.
    }
    
    private void updateRoutingTables(ShardConfig config) {
        // Update routing tables in all nodes
    }
    
    private void updateRoutingTables(List<ShardMetadata> shards) {
        // Update routing tables with new shard information
    }
}
```

### 3. CQRS Pattern

#### Overview
Separate read and write operations using different models to optimize performance and scalability.

#### CQRS Implementation
```java
// Write Model (Commands)
public interface Command {
    void execute();
}

public class CreateOrderCommand implements Command {
    
    private final OrderWriteRepository writeRepository;
    private final OrderCreatedEvent event;
    
    public CreateOrderCommand(OrderWriteRepository writeRepository, OrderCreatedEvent event) {
        this.writeRepository = writeRepository;
        this.event = event;
    }
    
    @Override
    public void execute() {
        // Validate business rules
        validateOrder(event);
        
        // Save to write database
        OrderEntity entity = new OrderEntity();
        entity.setId(event.getOrderId());
        entity.setUserId(event.getUserId());
        entity.setItems(event.getItems());
        entity.setTotalAmount(event.getTotalAmount());
        entity.setStatus(OrderStatus.CREATED);
        entity.setCreatedAt(Instant.now());
        
        writeRepository.save(entity);
        
        // Publish event for read model update
        eventPublisher.publish("order-events", event);
    }
    
    private void validateOrder(OrderCreatedEvent event) {
        if (event.getItems().isEmpty()) {
            throw new InvalidOrderException("Order must contain at least one item");
        }
        
        if (event.getTotalAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidOrderException("Order total must be positive");
        }
    }
}

// Read Model (Queries)
public interface Query<T> {
    T execute();
}

public class GetUserOrdersQuery implements Query<List<OrderSummary>> {
    
    private final OrderReadRepository readRepository;
    private final Long userId;
    private final int page;
    private final int size;
    
    public GetUserOrdersQuery(OrderReadRepository readRepository, Long userId, int page, int size) {
        this.readRepository = readRepository;
        this.userId = userId;
        this.page = page;
        this.size = size;
    }
    
    @Override
    public List<OrderSummary> execute() {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        return readRepository.findOrderSummariesByUserId(userId, pageable);
    }
}

public class GetOrderDetailsQuery implements Query<OrderDetails> {
    
    private final OrderReadRepository readRepository;
    private final Long orderId;
    
    public GetOrderDetailsQuery(OrderReadRepository readRepository, Long orderId) {
        this.readRepository = readRepository;
        this.orderId = orderId;
    }
    
    @Override
    public OrderDetails execute() {
        return readRepository.findOrderDetailsById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}

// Command Handler
@Service
public class OrderCommandHandler {
    
    @Autowired
    private OrderWriteRepository writeRepository;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    public void handle(CreateOrderCommand command) {
        command.execute();
    }
    
    public void handle(UpdateOrderStatusCommand command) {
        command.execute();
    }
    
    public void handle(CancelOrderCommand command) {
        command.execute();
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
        String cacheKey = generateCacheKey(query);
        Cache cache = cacheManager.getCache("order-queries");
        
        return cache.get(cacheKey, () -> query.execute());
    }
    
    private String generateCacheKey(Query<?> query) {
        return query.getClass().getSimpleName() + ":" + query.hashCode();
    }
}

// Read Model Updater
@Service
public class OrderReadModelUpdater {
    
    @Autowired
    private OrderReadRepository readRepository;
    
    @KafkaListener(topics = "order-events", groupId = "order-read-model")
    public void handleOrderEvent(OrderEvent event) {
        if (event instanceof OrderCreatedEvent) {
            updateOrderCreated((OrderCreatedEvent) event);
        } else if (event instanceof OrderStatusChangedEvent) {
            updateOrderStatus((OrderStatusChangedEvent) event);
        }
    }
    
    @Transactional
    public void updateOrderCreated(OrderCreatedEvent event) {
        OrderSummary summary = new OrderSummary();
        summary.setId(event.getOrderId());
        summary.setUserId(event.getUserId());
        summary.setStatus(OrderStatus.CREATED);
        summary.setTotalAmount(event.getTotalAmount());
        summary.setItemCount(event.getItems().size());
        summary.setCreatedAt(event.getTimestamp());
        
        readRepository.saveOrderSummary(summary);
        
        // Update user statistics
        updateUserOrderStats(event.getUserId(), 1, event.getTotalAmount());
    }
    
    @Transactional
    public void updateOrderStatus(OrderStatusChangedEvent event) {
        readRepository.updateOrderStatus(event.getOrderId(), event.getNewStatus());
        
        if (event.getNewStatus() == OrderStatus.CANCELLED) {
            // Update user statistics for cancelled orders
            OrderSummary summary = readRepository.findOrderSummaryById(event.getOrderId());
            if (summary != null) {
                updateUserOrderStats(summary.getUserId(), -1, summary.getTotalAmount().negate());
            }
        }
    }
    
    private void updateUserOrderStats(Long userId, int orderCountDelta, BigDecimal amountDelta) {
        UserOrderStats stats = readRepository.findUserOrderStats(userId);
        if (stats == null) {
            stats = new UserOrderStats();
            stats.setUserId(userId);
            stats.setTotalOrders(0);
            stats.setTotalAmount(BigDecimal.ZERO);
        }
        
        stats.setTotalOrders(stats.getTotalOrders() + orderCountDelta);
        stats.setTotalAmount(stats.getTotalAmount().add(amountDelta));
        
        readRepository.saveUserOrderStats(stats);
    }
}
```

## Reliability Patterns

### 1. Circuit Breaker Pattern

#### Overview
Prevent cascading failures by stopping requests to failing services.

#### Advanced Circuit Breaker
```java
@Service
public class AdvancedCircuitBreaker {
    
    public enum State { CLOSED, OPEN, HALF_OPEN }
    
    private final String serviceName;
    private final CircuitBreakerConfig config;
    private final MetricsService metricsService;
    
    private volatile State state = State.CLOSED;
    private final AtomicInteger failureCount = new AtomicInteger(0);
    private final AtomicInteger successCount = new AtomicInteger(0);
    private final AtomicLong lastFailureTime = new AtomicLong(0);
    private final AtomicLong lastStateChangeTime = new AtomicLong(System.currentTimeMillis());
    
    public AdvancedCircuitBreaker(String serviceName, CircuitBreakerConfig config, 
                                MetricsService metricsService) {
        this.serviceName = serviceName;
        this.config = config;
        this.metricsService = metricsService;
    }
    
    public <T> T execute(Supplier<T> operation, Function<Exception, T> fallback) throws Exception {
        if (state == State.OPEN) {
            if (shouldAttemptReset()) {
                state = State.HALF_OPEN;
                lastStateChangeTime.set(System.currentTimeMillis());
                metricsService.recordCircuitBreakerStateChange(serviceName, State.OPEN, State.HALF_OPEN);
            } else {
                metricsService.recordCircuitBreakerRejection(serviceName);
                return fallback.apply(new CircuitBreakerOpenException(serviceName));
            }
        }
        
        try {
            T result = operation.get();
            
            onSuccess();
            metricsService.recordCircuitBreakerSuccess(serviceName);
            
            return result;
            
        } catch (Exception e) {
            onFailure(e);
            metricsService.recordCircuitBreakerFailure(serviceName);
            
            if (state == State.HALF_OPEN) {
                // Single failure in half-open state causes immediate open
                transitionToOpen();
            }
            
            return fallback.apply(e);
        }
    }
    
    private void onSuccess() {
        failureCount.set(0);
        
        if (state == State.HALF_OPEN) {
            int successes = successCount.incrementAndGet();
            
            if (successes >= config.getSuccessThreshold()) {
                transitionToClosed();
            }
        }
    }
    
    private void onFailure(Exception e) {
        lastFailureTime.set(System.currentTimeMillis());
        int failures = failureCount.incrementAndGet();
        successCount.set(0);
        
        // Check if we should open the circuit
        if (state == State.CLOSED && failures >= config.getFailureThreshold()) {
            transitionToOpen();
        }
        
        // Record failure details for analysis
        metricsService.recordCircuitBreakerFailureDetails(serviceName, e.getClass().getSimpleName());
    }
    
    private void transitionToOpen() {
        State oldState = state;
        state = State.OPEN;
        lastStateChangeTime.set(System.currentTimeMillis());
        metricsService.recordCircuitBreakerStateChange(serviceName, oldState, State.OPEN);
        
        logger.warn("Circuit breaker for {} opened after {} failures", 
                   serviceName, failureCount.get());
    }
    
    private void transitionToClosed() {
        State oldState = state;
        state = State.CLOSED;
        failureCount.set(0);
        successCount.set(0);
        lastStateChangeTime.set(System.currentTimeMillis());
        metricsService.recordCircuitBreakerStateChange(serviceName, oldState, State.CLOSED);
        
        logger.info("Circuit breaker for {} closed after successful calls", serviceName);
    }
    
    private boolean shouldAttemptReset() {
        long timeSinceLastFailure = System.currentTimeMillis() - lastFailureTime.get();
        return timeSinceLastFailure >= config.getTimeout().toMillis();
    }
    
    public State getState() {
        return state;
    }
    
    public CircuitBreakerMetrics getMetrics() {
        return new CircuitBreakerMetrics(
            state,
            failureCount.get(),
            successCount.get(),
            lastFailureTime.get(),
            lastStateChangeTime.get()
        );
    }
    
    public static class CircuitBreakerConfig {
        private final int failureThreshold;
        private final int successThreshold;
        private final Duration timeout;
        
        public CircuitBreakerConfig(int failureThreshold, int successThreshold, Duration timeout) {
            this.failureThreshold = failureThreshold;
            this.successThreshold = successThreshold;
            this.timeout = timeout;
        }
        
        // Getters
    }
    
    public static class CircuitBreakerMetrics {
        private final State state;
        private final int failureCount;
        private final int successCount;
        private final long lastFailureTime;
        private final long lastStateChangeTime;
        
        public CircuitBreakerMetrics(State state, int failureCount, int successCount, 
                                   long lastFailureTime, long lastStateChangeTime) {
            this.state = state;
            this.failureCount = failureCount;
            this.successCount = successCount;
            this.lastFailureTime = lastFailureTime;
            this.lastStateChangeTime = lastStateChangeTime;
        }
        
        // Getters
    }
}
```

### 2. Bulkhead Pattern

#### Resource Isolation
```java
@Configuration
public class BulkheadConfiguration {
    
    @Bean
    public ThreadPoolTaskExecutor userServiceExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("user-service-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
    
    @Bean
    public ThreadPoolTaskExecutor orderServiceExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);
        executor.setMaxPoolSize(100);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("order-service-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
    
    @Bean
    public ThreadPoolTaskExecutor paymentServiceExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("payment-service-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

@Service
public class IsolatedServiceClient {
    
    @Autowired
    private ThreadPoolTaskExecutor userServiceExecutor;
    
    @Autowired
    private ThreadPoolTaskExecutor orderServiceExecutor;
    
    @Autowired
    private RestTemplate restTemplate;
    
    public CompletableFuture<User> getUserAsync(Long userId) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                String url = "http://user-service/users/" + userId;
                ResponseEntity<User> response = restTemplate.getForEntity(url, User.class);
                return response.getBody();
            } catch (Exception e) {
                logger.error("Failed to get user {}", userId, e);
                throw new CompletionException(e);
            }
        }, userServiceExecutor);
    }
    
    public CompletableFuture<Order> createOrderAsync(CreateOrderRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                String url = "http://order-service/orders";
                ResponseEntity<Order> response = restTemplate.postForEntity(url, request, Order.class);
                return response.getBody();
            } catch (Exception e) {
                logger.error("Failed to create order for user {}", request.getUserId(), e);
                throw new CompletionException(e);
            }
        }, orderServiceExecutor);
    }
    
    public CompletableFuture<PaymentResult> processPaymentAsync(Long orderId, BigDecimal amount) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                String url = "http://payment-service/payments";
                PaymentRequest request = new PaymentRequest(orderId, amount);
                ResponseEntity<PaymentResult> response = restTemplate.postForEntity(url, request, PaymentResult.class);
                return response.getBody();
            } catch (Exception e) {
                logger.error("Failed to process payment for order {}", orderId, e);
                throw new CompletionException(e);
            }
        }, Executors.newSingleThreadExecutor()); // Separate executor for payments
    }
}
```

### 3. Retry Pattern with Exponential Backoff

#### Intelligent Retry Strategy
```java
public class IntelligentRetryStrategy {
    
    private final int maxRetries;
    private final Duration initialDelay;
    private final double backoffMultiplier;
    private final Duration maxDelay;
    private final Set<Class<? extends Exception>> retryableExceptions;
    private final Set<Class<? extends Exception>> nonRetryableExceptions;
    
    public IntelligentRetryStrategy(int maxRetries, Duration initialDelay, 
                                  double backoffMultiplier, Duration maxDelay,
                                  Set<Class<? extends Exception>> retryableExceptions,
                                  Set<Class<? extends Exception>> nonRetryableExceptions) {
        this.maxRetries = maxRetries;
        this.initialDelay = initialDelay;
        this.backoffMultiplier = backoffMultiplier;
        this.maxDelay = maxDelay;
        this.retryableExceptions = retryableExceptions != null ? retryableExceptions : new HashSet<>();
        this.nonRetryableExceptions = nonRetryableExceptions != null ? nonRetryableExceptions : new HashSet<>();
    }
    
    public <T> T execute(Supplier<T> operation) throws Exception {
        Exception lastException = null;
        Duration delay = initialDelay;
        int attempts = 0;
        
        while (attempts <= maxRetries) {
            try {
                return operation.get();
            } catch (Exception e) {
                lastException = e;
                attempts++;
                
                // Don't retry if we've exhausted attempts
                if (attempts > maxRetries) {
                    break;
                }
                
                // Check if exception is retryable
                if (!isRetryable(e)) {
                    break;
                }
                
                // Check if it's worth retrying (circuit breaker-like behavior)
                if (isLikelyPermanentFailure(e)) {
                    logger.warn("Detected likely permanent failure, not retrying: {}", e.getMessage());
                    break;
                }
                
                logger.warn("Attempt {} failed, retrying in {}: {}", 
                           attempts, delay, e.getMessage());
                
                // Wait before retry with jitter
                Thread.sleep(addJitter(delay));
                
                // Calculate next delay
                delay = Duration.ofMillis(
                    Math.min((long)(delay.toMillis() * backoffMultiplier), 
                            maxDelay.toMillis()));
                
            }
        }
        
        throw lastException;
    }
    
    private boolean isRetryable(Exception e) {
        // Check if exception type is explicitly non-retryable
        for (Class<? extends Exception> nonRetryable : nonRetryableExceptions) {
            if (nonRetryable.isAssignableFrom(e.getClass())) {
                return false;
            }
        }
        
        // Check if exception type is explicitly retryable
        for (Class<? extends Exception> retryable : retryableExceptions) {
            if (retryable.isAssignableFrom(e.getClass())) {
                return true;
            }
        }
        
        // Default behavior: retry network and temporary failures
        return isNetworkOrTemporaryFailure(e);
    }
    
    private boolean isNetworkOrTemporaryFailure(Exception e) {
        return e instanceof IOException || 
               e instanceof TimeoutException ||
               e instanceof ConnectException ||
               e instanceof SocketTimeoutException ||
               (e instanceof HttpServerErrorException && 
                ((HttpServerErrorException) e).getStatusCode().is5xxServerError());
    }
    
    private boolean isLikelyPermanentFailure(Exception e) {
        // Check for patterns that indicate permanent failures
        if (e instanceof HttpClientErrorException) {
            HttpClientErrorException httpException = (HttpClientErrorException) e;
            // 4xx errors are typically client errors, not worth retrying
            return httpException.getStatusCode().is4xxClientError();
        }
        
        // Check for specific exception types
        return e instanceof IllegalArgumentException ||
               e instanceof AuthenticationException;
    }
    
    private long addJitter(Duration delay) {
        // Add random jitter to prevent thundering herd
        long jitter = (long) (delay.toMillis() * 0.1 * ThreadLocalRandom.current().nextDouble(-1, 1));
        return Math.max(0, delay.toMillis() + jitter);
    }
    
    // Builder pattern for easy configuration
    public static class Builder {
        private int maxRetries = 3;
        private Duration initialDelay = Duration.ofMillis(100);
        private double backoffMultiplier = 2.0;
        private Duration maxDelay = Duration.ofSeconds(30);
        private Set<Class<? extends Exception>> retryableExceptions = new HashSet<>();
        private Set<Class<? extends Exception>> nonRetryableExceptions = new HashSet<>();
        
        public Builder maxRetries(int maxRetries) {
            this.maxRetries = maxRetries;
            return this;
        }
        
        public Builder initialDelay(Duration initialDelay) {
            this.initialDelay = initialDelay;
            return this;
        }
        
        public Builder backoffMultiplier(double backoffMultiplier) {
            this.backoffMultiplier = backoffMultiplier;
            return this;
        }
        
        public Builder maxDelay(Duration maxDelay) {
            this.maxDelay = maxDelay;
            return this;
        }
        
        public Builder retryOn(Class<? extends Exception> exceptionType) {
            this.retryableExceptions.add(exceptionType);
            return this;
        }
        
        public Builder dontRetryOn(Class<? extends Exception> exceptionType) {
            this.nonRetryableExceptions.add(exceptionType);
            return this;
        }
        
        public IntelligentRetryStrategy build() {
            return new IntelligentRetryStrategy(maxRetries, initialDelay, backoffMultiplier, 
                                              maxDelay, retryableExceptions, nonRetryableExceptions);
        }
    }
}
```

## Data Management Patterns

### 1. Event Sourcing Pattern

#### Overview
Store state changes as a sequence of events rather than current state, enabling audit trails, temporal queries, and event replay.

#### Event Sourcing Implementation
```java
// Domain Events
public interface DomainEvent {
    String getEventType();
    Instant getTimestamp();
    String getAggregateId();
    int getVersion();
}

public class OrderCreatedEvent implements DomainEvent {
    
    private final String eventType = "OrderCreated";
    private final String orderId;
    private final Long userId;
    private final List<OrderItem> items;
    private final BigDecimal totalAmount;
    private final Instant timestamp;
    private final int version = 1;
    
    public OrderCreatedEvent(String orderId, Long userId, List<OrderItem> items, 
                           BigDecimal totalAmount, Instant timestamp) {
        this.orderId = orderId;
        this.userId = userId;
        this.items = new ArrayList<>(items);
        this.totalAmount = totalAmount;
        this.timestamp = timestamp;
    }
    
    @Override
    public String getEventType() { return eventType; }
    @Override
    public Instant getTimestamp() { return timestamp; }
    @Override
    public String getAggregateId() { return orderId; }
    @Override
    public int getVersion() { return version; }
    
    // Getters for other fields
}

public class OrderStatusChangedEvent implements DomainEvent {
    
    private final String eventType = "OrderStatusChanged";
    private final String orderId;
    private final OrderStatus oldStatus;
    private final OrderStatus newStatus;
    private final Instant timestamp;
    private final int version;
    
    public OrderStatusChangedEvent(String orderId, OrderStatus oldStatus, 
                                 OrderStatus newStatus, Instant timestamp, int version) {
        this.orderId = orderId;
        this.oldStatus = oldStatus;
        this.newStatus = newStatus;
        this.timestamp = timestamp;
        this.version = version;
    }
    
    @Override
    public String getEventType() { return eventType; }
    @Override
    public Instant getTimestamp() { return timestamp; }
    @Override
    public String getAggregateId() { return orderId; }
    @Override
    public int getVersion() { return version; }
}

// Event Store
@Repository
public interface EventStore {
    
    void save(DomainEvent event);
    
    List<DomainEvent> findByAggregateId(String aggregateId);
    
    List<DomainEvent> findByAggregateIdAndVersionGreaterThan(String aggregateId, int version);
    
    List<DomainEvent> findByEventType(String eventType, Instant from, Instant to);
    
    Optional<DomainEvent> findLatestByAggregateId(String aggregateId);
}

@Service
public class JpaEventStore implements EventStore {
    
    @Autowired
    private EventEntityRepository repository;
    
    @Override
    public void save(DomainEvent event) {
        EventEntity entity = new EventEntity();
        entity.setEventId(UUID.randomUUID().toString());
        entity.setEventType(event.getEventType());
        entity.setAggregateId(event.getAggregateId());
        entity.setVersion(event.getVersion());
        entity.setTimestamp(event.getTimestamp());
        entity.setPayload(serializeEvent(event));
        
        repository.save(entity);
    }
    
    @Override
    public List<DomainEvent> findByAggregateId(String aggregateId) {
        return repository.findByAggregateIdOrderByVersion(aggregateId)
            .stream()
            .map(this::deserializeEvent)
            .collect(Collectors.toList());
    }
    
    @Override
    public List<DomainEvent> findByAggregateIdAndVersionGreaterThan(String aggregateId, int version) {
        return repository.findByAggregateIdAndVersionGreaterThanOrderByVersion(aggregateId, version)
            .stream()
            .map(this::deserializeEvent)
            .collect(Collectors.toList());
    }
    
    @Override
    public List<DomainEvent> findByEventType(String eventType, Instant from, Instant to) {
        return repository.findByEventTypeAndTimestampBetween(eventType, from, to)
            .stream()
            .map(this::deserializeEvent)
            .collect(Collectors.toList());
    }
    
    @Override
    public Optional<DomainEvent> findLatestByAggregateId(String aggregateId) {
        return repository.findTopByAggregateIdOrderByVersionDesc(aggregateId)
            .map(this::deserializeEvent);
    }
    
    private String serializeEvent(DomainEvent event) {
        // Serialize event to JSON
        try {
            return objectMapper.writeValueAsString(event);
        } catch (Exception e) {
            throw new RuntimeException("Failed to serialize event", e);
        }
    }
    
    private DomainEvent deserializeEvent(EventEntity entity) {
        // Deserialize JSON to event
        try {
            Class<?> eventClass = Class.forName("com.example.events." + entity.getEventType());
            return (DomainEvent) objectMapper.readValue(entity.getPayload(), eventClass);
        } catch (Exception e) {
            throw new RuntimeException("Failed to deserialize event", e);
        }
    }
}

// Aggregate Root with Event Sourcing
public class OrderAggregate {
    
    private String id;
    private Long userId;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private OrderStatus status;
    private Instant createdAt;
    private Instant updatedAt;
    private int version;
    
    private final List<DomainEvent> uncommittedEvents = new ArrayList<>();
    
    public OrderAggregate() {
        // For framework instantiation
    }
    
    public OrderAggregate(String id, Long userId, List<OrderItem> items, BigDecimal totalAmount) {
        applyChange(new OrderCreatedEvent(id, userId, items, totalAmount, Instant.now()));
    }
    
    public void changeStatus(OrderStatus newStatus) {
        if (status == newStatus) {
            return; // No change needed
        }
        
        applyChange(new OrderStatusChangedEvent(id, status, newStatus, Instant.now(), version + 1));
    }
    
    public void addItem(OrderItem item) {
        List<OrderItem> newItems = new ArrayList<>(items);
        newItems.add(item);
        BigDecimal newTotal = totalAmount.add(item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())));
        
        applyChange(new OrderItemAddedEvent(id, item, newTotal, Instant.now(), version + 1));
    }
    
    private void applyChange(DomainEvent event) {
        applyEvent(event);
        uncommittedEvents.add(event);
    }
    
    private void applyEvent(DomainEvent event) {
        if (event instanceof OrderCreatedEvent) {
            apply((OrderCreatedEvent) event);
        } else if (event instanceof OrderStatusChangedEvent) {
            apply((OrderStatusChangedEvent) event);
        } else if (event instanceof OrderItemAddedEvent) {
            apply((OrderItemAddedEvent) event);
        }
    }
    
    private void apply(OrderCreatedEvent event) {
        this.id = event.getOrderId();
        this.userId = event.getUserId();
        this.items = new ArrayList<>(event.getItems());
        this.totalAmount = event.getTotalAmount();
        this.status = OrderStatus.CREATED;
        this.createdAt = event.getTimestamp();
        this.updatedAt = event.getTimestamp();
        this.version = event.getVersion();
    }
    
    private void apply(OrderStatusChangedEvent event) {
        this.status = event.getNewStatus();
        this.updatedAt = event.getTimestamp();
        this.version = event.getVersion();
    }
    
    private void apply(OrderItemAddedEvent event) {
        this.items.add(event.getItem());
        this.totalAmount = event.getNewTotal();
        this.updatedAt = event.getTimestamp();
        this.version = event.getVersion();
    }
    
    // Reconstruct aggregate from events
    public static OrderAggregate reconstruct(List<DomainEvent> events) {
        OrderAggregate aggregate = new OrderAggregate();
        for (DomainEvent event : events) {
            aggregate.applyEvent(event);
        }
        aggregate.uncommittedEvents.clear(); // Events are already committed
        return aggregate;
    }
    
    public List<DomainEvent> getUncommittedEvents() {
        return new ArrayList<>(uncommittedEvents);
    }
    
    public void markEventsCommitted() {
        uncommittedEvents.clear();
    }
    
    // Getters
    public String getId() { return id; }
    public OrderStatus getStatus() { return status; }
    public int getVersion() { return version; }
}

// Repository with Event Sourcing
@Service
public class EventSourcedOrderRepository {
    
    @Autowired
    private EventStore eventStore;
    
    public OrderAggregate findById(String orderId) {
        List<DomainEvent> events = eventStore.findByAggregateId(orderId);
        if (events.isEmpty()) {
            throw new OrderNotFoundException(orderId);
        }
        
        return OrderAggregate.reconstruct(events);
    }
    
    @Transactional
    public void save(OrderAggregate aggregate) {
        List<DomainEvent> uncommittedEvents = aggregate.getUncommittedEvents();
        
        for (DomainEvent event : uncommittedEvents) {
            eventStore.save(event);
        }
        
        aggregate.markEventsCommitted();
    }
    
    public List<OrderAggregate> findByUserId(Long userId) {
        // This would require a query model or event scanning
        // For simplicity, assuming we have a separate read model
        throw new UnsupportedOperationException("Use read model for queries");
    }
}

// Application Service using Event Sourcing
@Service
public class OrderApplicationService {
    
    @Autowired
    private EventSourcedOrderRepository repository;
    
    @Autowired
    private DomainEventPublisher eventPublisher;
    
    @Transactional
    public OrderAggregate createOrder(CreateOrderRequest request) {
        // Validate request
        validateCreateOrderRequest(request);
        
        // Create aggregate
        OrderAggregate order = new OrderAggregate(
            generateOrderId(),
            request.getUserId(),
            request.getItems(),
            calculateTotal(request.getItems())
        );
        
        // Save events
        repository.save(order);
        
        // Publish events for integration
        for (DomainEvent event : order.getUncommittedEvents()) {
            eventPublisher.publish("order-events", event);
        }
        
        return order;
    }
    
    @Transactional
    public void updateOrderStatus(String orderId, OrderStatus newStatus) {
        OrderAggregate order = repository.findById(orderId);
        
        // Business logic validation
        validateStatusTransition(order.getStatus(), newStatus);
        
        // Update aggregate
        order.changeStatus(newStatus);
        
        // Save events
        repository.save(order);
        
        // Publish events
        for (DomainEvent event : order.getUncommittedEvents()) {
            eventPublisher.publish("order-events", event);
        }
    }
    
    private void validateCreateOrderRequest(CreateOrderRequest request) {
        if (request.getItems() == null || request.getItems().isEmpty()) {
            throw new InvalidOrderException("Order must contain at least one item");
        }
        
        // Additional validations...
    }
    
    private void validateStatusTransition(OrderStatus current, OrderStatus newStatus) {
        // Validate state transitions
        if (current == OrderStatus.SHIPPED && newStatus == OrderStatus.CREATED) {
            throw new InvalidStatusTransitionException("Cannot change shipped order back to created");
        }
    }
    
    private String generateOrderId() {
        return "ORD-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
    
    private BigDecimal calculateTotal(List<OrderItem> items) {
        return items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

## Communication Patterns

### 1. Saga Pattern for Distributed Transactions

#### Choreography-Based Saga
```java
@Service
public class OrderSagaOrchestrator {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void startOrderSaga(CreateOrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        
        // Step 1: Reserve inventory
        InventoryReservationRequest reservation = new InventoryReservationRequest(
            sagaId, request.getItems());
        
        kafkaTemplate.send("inventory-commands", "reserve", reservation);
        
        logger.info("Started inventory reservation for saga {}", sagaId);
    }
}

@Service
public class InventorySagaParticipant {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @KafkaListener(topics = "inventory-commands", groupId = "inventory-saga")
    public void handleInventoryReservation(InventoryReservationRequest request) {
        try {
            // Reserve inventory
            inventoryService.reserveInventory(request.getSagaId(), request.getItems());
            
            // Notify next step: Process payment
            PaymentProcessingRequest payment = new PaymentProcessingRequest(
                request.getSagaId(), calculateTotal(request.getItems()));
            
            kafkaTemplate.send("payment-commands", "process", payment);
            
        } catch (Exception e) {
            // Saga failed - compensate
            SagaFailedEvent failure = new SagaFailedEvent(
                request.getSagaId(), "inventory-reservation", e.getMessage());
            
            kafkaTemplate.send("saga-events", "failed", failure);
        }
    }
    
    @KafkaListener(topics = "saga-events", groupId = "inventory-saga")
    public void handleSagaFailure(SagaFailedEvent event) {
        // Release reserved inventory
        inventoryService.releaseInventoryReservation(event.getSagaId());
        
        logger.info("Released inventory reservation for failed saga {}", event.getSagaId());
    }
}

@Service
public class PaymentSagaParticipant {
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @KafkaListener(topics = "payment-commands", groupId = "payment-saga")
    public void handlePaymentProcessing(PaymentProcessingRequest request) {
        try {
            // Process payment
            PaymentResult result = paymentService.processPayment(request.getAmount());
            
            if (result.isSuccessful()) {
                // Payment successful - create order
                OrderCreationRequest order = new OrderCreationRequest(
                    request.getSagaId(), result.getTransactionId());
                
                kafkaTemplate.send("order-commands", "create", order);
            } else {
                throw new PaymentFailedException("Payment processing failed");
            }
            
        } catch (Exception e) {
            // Saga failed - compensate
            SagaFailedEvent failure = new SagaFailedEvent(
                request.getSagaId(), "payment-processing", e.getMessage());
            
            kafkaTemplate.send("saga-events", "failed", failure);
        }
    }
}

@Service
public class OrderSagaParticipant {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @KafkaListener(topics = "order-commands", groupId = "order-saga")
    public void handleOrderCreation(OrderCreationRequest request) {
        try {
            // Create order
            Order order = orderService.createOrderFromSaga(request);
            
            // Saga completed successfully
            SagaCompletedEvent completion = new SagaCompletedEvent(
                request.getSagaId(), order.getId());
            
            kafkaTemplate.send("saga-events", "completed", completion);
            
            logger.info("Order saga {} completed with order {}", 
                       request.getSagaId(), order.getId());
            
        } catch (Exception e) {
            // Saga failed - compensate
            SagaFailedEvent failure = new SagaFailedEvent(
                request.getSagaId(), "order-creation", e.getMessage());
            
            kafkaTemplate.send("saga-events", "failed", failure);
        }
    }
    
    @KafkaListener(topics = "saga-events", groupId = "order-saga")
    public void handleSagaEvents(SagaEvent event) {
        if (event instanceof SagaCompletedEvent) {
            // Clean up saga state
            cleanupSagaState(event.getSagaId());
        } else if (event instanceof SagaFailedEvent) {
            // Compensate order creation if it happened
            compensateOrderCreation(event.getSagaId());
        }
    }
}
```

## Best Practices

### Pattern Selection
- **Assess Requirements**: Choose patterns based on scalability, consistency, and performance needs
- **Start Simple**: Begin with basic patterns and evolve as complexity grows
- **Consider Trade-offs**: Understand the costs and benefits of each pattern
- **Test Patterns**: Thoroughly test pattern implementations under various conditions

### Implementation Guidelines
- **Consistency**: Ensure patterns are applied consistently across the system
- **Monitoring**: Monitor pattern effectiveness and performance impact
- **Documentation**: Document pattern usage and decisions
- **Evolution**: Plan for pattern evolution as requirements change

### Common Pitfalls
- **Over-Engineering**: Don't apply complex patterns when simple solutions suffice
- **Inconsistent Application**: Apply patterns consistently or not at all
- **Performance Impact**: Monitor and optimize pattern performance
- **Testing Complexity**: Ensure patterns can be adequately tested

## Conclusion

System design patterns provide proven solutions for building scalable, reliable distributed systems. From load balancing and sharding to CQRS and event sourcing, these patterns address common challenges in distributed architecture. Understanding when and how to apply these patterns is crucial for building maintainable, high-performance systems.

**Key Takeaways:**
- **Load Balancing**: Distribute load effectively across system components
- **Database Sharding**: Scale databases horizontally while maintaining performance
- **CQRS**: Separate read and write concerns for optimal performance
- **Circuit Breaker**: Prevent cascading failures in distributed systems
- **Event Sourcing**: Maintain audit trails and enable complex event processing
- **Saga Pattern**: Manage distributed transactions reliably

Effective use of system design patterns requires understanding their trade-offs and applying them appropriately based on specific system requirements and constraints.
