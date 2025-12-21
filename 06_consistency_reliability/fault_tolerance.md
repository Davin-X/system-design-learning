# Fault Tolerance

Fault tolerance is the ability of a system to continue operating correctly despite failures of some of its components. In distributed systems, faults are inevitable, so building fault-tolerant systems is crucial for maintaining reliability and availability. This guide covers fault tolerance patterns, techniques, and implementations.

## What is Fault Tolerance?

Fault tolerance is the system's ability to continue operating properly in the event of failures. It involves detecting faults, recovering from them, and preventing failures from cascading through the system.

### Types of Faults

#### Hardware Faults
- **Server crashes**: Complete machine failure
- **Disk failures**: Storage device malfunctions
- **Network failures**: Connectivity issues
- **Power failures**: Loss of electricity

#### Software Faults
- **Bugs**: Programming errors
- **Configuration errors**: Misconfigurations
- **Resource exhaustion**: Memory leaks, out of disk space
- **Dependency failures**: External service failures

#### Human Faults
- **Operator errors**: Manual mistakes
- **Deployment errors**: Faulty releases
- **Security breaches**: Unauthorized access

## Fault Tolerance Patterns

### 1. Replication

#### Active Replication
All replicas process every request simultaneously.

```java
public class ActiveReplicationService {
    
    private final List<ServiceReplica> replicas;
    private final AtomicInteger requestId = new AtomicInteger(0);
    
    public Response handleRequest(Request request) {
        int id = requestId.incrementAndGet();
        
        // Send request to all replicas
        List<CompletableFuture<Response>> futures = replicas.stream()
            .map(replica -> replica.processRequest(request, id))
            .collect(Collectors.toList());
        
        // Wait for majority response (quorum)
        List<Response> responses = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        
        // Return response from primary replica
        return responses.get(0);
    }
    
    public void addReplica(ServiceReplica replica) {
        replicas.add(replica);
    }
    
    public void removeReplica(ServiceReplica replica) {
        replicas.remove(replica);
    }
}
```

#### Passive Replication (Primary-Backup)
Only primary replica processes requests, backups are updated asynchronously.

```java
public class PrimaryBackupService {
    
    private ServiceReplica primary;
    private final List<ServiceReplica> backups;
    private final AtomicInteger sequenceNumber = new AtomicInteger(0);
    
    public Response handleRequest(Request request) {
        int seqNum = sequenceNumber.incrementAndGet();
        
        try {
            // Process on primary
            Response response = primary.processRequest(request);
            
            // Update backups asynchronously
            updateBackupsAsync(request, seqNum);
            
            return response;
            
        } catch (Exception e) {
            // Primary failed, promote backup
            promoteBackupToPrimary();
            return handleRequest(request); // Retry
        }
    }
    
    private void updateBackupsAsync(Request request, int seqNum) {
        backups.forEach(backup -> 
            CompletableFuture.runAsync(() -> {
                try {
                    backup.updateState(request, seqNum);
                } catch (Exception e) {
                    logger.warn("Failed to update backup", e);
                }
            })
        );
    }
    
    private void promoteBackupToPrimary() {
        if (!backups.isEmpty()) {
            primary = backups.remove(0);
            logger.info("Promoted backup to primary");
        } else {
            throw new NoAvailableReplicasException();
        }
    }
}
```

### 2. Circuit Breaker

#### Circuit Breaker Pattern
Prevents cascading failures by stopping requests to failing services.

```java
public class CircuitBreaker {
    
    public enum State { CLOSED, OPEN, HALF_OPEN }
    
    private State state = State.CLOSED;
    private final int failureThreshold;
    private final Duration timeout;
    private final Duration retryTimeout;
    
    private int failureCount = 0;
    private Instant lastFailureTime;
    
    public CircuitBreaker(int failureThreshold, Duration timeout, Duration retryTimeout) {
        this.failureThreshold = failureThreshold;
        this.timeout = timeout;
        this.retryTimeout = retryTimeout;
    }
    
    public <T> T execute(Supplier<T> operation) throws CircuitBreakerException {
        if (state == State.OPEN) {
            if (shouldAttemptReset()) {
                state = State.HALF_OPEN;
            } else {
                throw new CircuitBreakerException("Circuit is open");
            }
        }
        
        try {
            T result = operation.get();
            
            // Success - reset circuit
            if (state == State.HALF_OPEN) {
                state = State.CLOSED;
                failureCount = 0;
            }
            
            return result;
            
        } catch (Exception e) {
            recordFailure();
            
            if (state == State.HALF_OPEN) {
                state = State.OPEN;
            }
            
            throw e;
        }
    }
    
    private void recordFailure() {
        failureCount++;
        lastFailureTime = Instant.now();
        
        if (failureCount >= failureThreshold) {
            state = State.OPEN;
        }
    }
    
    private boolean shouldAttemptReset() {
        return lastFailureTime != null && 
               Duration.between(lastFailureTime, Instant.now()).compareTo(retryTimeout) > 0;
    }
}
```

#### Circuit Breaker with Resilience4j
```java
@Configuration
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        return CircuitBreakerRegistry.ofDefaults();
    }
    
    @Bean
    public CircuitBreaker externalServiceCircuitBreaker(CircuitBreakerRegistry registry) {
        return registry.circuitBreaker("external-service", 
            CircuitBreakerConfig.custom()
                .failureRateThreshold(50)
                .waitDurationInOpenState(Duration.ofMillis(10000))
                .slidingWindowSize(10)
                .permittedNumberOfCallsInHalfOpenState(3)
                .build());
    }
}

@Service
public class ExternalServiceClient {
    
    @Autowired
    private CircuitBreaker circuitBreaker;
    
    public String callExternalService(String request) {
        return circuitBreaker.executeSupplier(() -> {
            // Actual external service call
            return restTemplate.postForObject("/api/service", request, String.class);
        });
    }
}
```

### 3. Bulkhead

#### Bulkhead Pattern
Isolates failures in one part of the system from affecting others.

```java
public class BulkheadExecutor {
    
    private final ExecutorService executor;
    private final Semaphore semaphore;
    private final int maxConcurrentCalls;
    
    public BulkheadExecutor(int maxConcurrentCalls, int queueSize) {
        this.maxConcurrentCalls = maxConcurrentCalls;
        this.semaphore = new Semaphore(maxConcurrentCalls);
        this.executor = new ThreadPoolExecutor(
            maxConcurrentCalls, maxConcurrentCalls,
            0L, TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<>(queueSize),
            new ThreadPoolExecutor.AbortPolicy()
        );
    }
    
    public <T> CompletableFuture<T> execute(Supplier<T> operation) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                semaphore.acquire();
                return operation.get();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            } finally {
                semaphore.release();
            }
        }, executor);
    }
    
    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

#### Thread Pool Bulkheads
```java
@Configuration
public class BulkheadConfiguration {
    
    @Bean
    public ThreadPoolBulkheadConfig bulkheadConfig() {
        return ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(10)
            .coreThreadPoolSize(2)
            .queueCapacity(20)
            .build();
    }
    
    @Bean
    public BulkheadRegistry bulkheadRegistry() {
        return BulkheadRegistry.ofDefaults();
    }
    
    @Bean
    public ThreadPoolBulkhead databaseBulkhead(BulkheadRegistry registry) {
        return registry.bulkhead("database", 
            ThreadPoolBulkheadConfig.custom()
                .maxThreadPoolSize(5)
                .coreThreadPoolSize(1)
                .queueCapacity(10)
                .build());
    }
}

@Service
public class DatabaseService {
    
    @Autowired
    private ThreadPoolBulkhead bulkhead;
    
    public List<User> getUsers() {
        return bulkhead.executeSupplier(() -> 
            userRepository.findAll());
    }
}
```

### 4. Retry Pattern

#### Exponential Backoff Retry
```java
public class RetryExecutor {
    
    private final int maxRetries;
    private final Duration initialDelay;
    private final double backoffMultiplier;
    private final Duration maxDelay;
    
    public RetryExecutor(int maxRetries, Duration initialDelay, 
                        double backoffMultiplier, Duration maxDelay) {
        this.maxRetries = maxRetries;
        this.initialDelay = initialDelay;
        this.backoffMultiplier = backoffMultiplier;
        this.maxDelay = maxDelay;
    }
    
    public <T> T execute(Supplier<T> operation) throws Exception {
        Duration delay = initialDelay;
        
        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                return operation.get();
            } catch (Exception e) {
                if (attempt == maxRetries) {
                    throw e; // Max retries reached
                }
                
                // Wait before retry
                Thread.sleep(delay.toMillis());
                
                // Calculate next delay
                delay = Duration.ofMillis(
                    Math.min((long)(delay.toMillis() * backoffMultiplier), 
                            maxDelay.toMillis()));
            }
        }
        
        throw new IllegalStateException("Should not reach here");
    }
}
```

#### Retry with Resilience4j
```java
@Configuration
public class RetryConfig {
    
    @Bean
    public RetryRegistry retryRegistry() {
        return RetryRegistry.ofDefaults();
    }
    
    @Bean
    public Retry externalApiRetry(RetryRegistry registry) {
        return registry.retry("external-api", 
            RetryConfig.custom()
                .maxAttempts(3)
                .waitDuration(Duration.ofMillis(100))
                .retryOnException(throwable -> throwable instanceof IOException)
                .build());
    }
}

@Service
public class ExternalApiClient {
    
    @Autowired
    private Retry retry;
    
    public String callApi(String request) {
        return retry.executeSupplier(() -> 
            restTemplate.postForObject("/api/external", request, String.class));
    }
}
```

### 5. Timeout Pattern

#### Request Timeouts
```java
@Service
public class TimeoutService {
    
    @Autowired
    private WebClient webClient;
    
    public Mono<String> callWithTimeout(String url, Duration timeout) {
        return webClient.get()
            .uri(url)
            .retrieve()
            .bodyToMono(String.class)
            .timeout(timeout)
            .onErrorResume(TimeoutException.class, ex -> 
                Mono.just("Request timed out"));
    }
    
    public CompletableFuture<String> callAsyncWithTimeout(String url, Duration timeout) {
        return webClient.get()
            .uri(url)
            .retrieve()
            .bodyToMono(String.class)
            .toFuture()
            .orTimeout(timeout.toMillis(), TimeUnit.MILLISECONDS)
            .exceptionally(throwable -> {
                if (throwable instanceof TimeoutException) {
                    return "Request timed out";
                }
                throw new CompletionException(throwable);
            });
    }
}
```

#### Resource Timeouts
```java
@Configuration
public class DatabaseTimeoutConfig {
    
    @Bean
    public HikariDataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        
        // Connection timeout
        config.setConnectionTimeout(30000); // 30 seconds
        
        // Query timeout
        config.setMaxLifetime(1800000); // 30 minutes
        
        // Idle timeout
        config.setIdleTimeout(600000); // 10 minutes
        
        return new HikariDataSource(config);
    }
}
```

## Fault Detection and Recovery

### Health Checks

#### Application Health Checks
```java
@RestController
public class HealthController {
    
    @Autowired
    private HealthIndicator databaseHealth;
    
    @Autowired
    private HealthIndicator externalServiceHealth;
    
    @GetMapping("/health")
    public ResponseEntity<HealthStatus> health() {
        Health.Builder builder = new Health.Builder();
        
        // Check database
        try {
            databaseHealth.health();
            builder.up().withDetail("database", "available");
        } catch (Exception e) {
            builder.down().withDetail("database", e.getMessage());
        }
        
        // Check external service
        try {
            externalServiceHealth.health();
            builder.up().withDetail("external-service", "available");
        } catch (Exception e) {
            builder.down().withDetail("external-service", e.getMessage());
        }
        
        Health health = builder.build();
        return ResponseEntity.status(health.getStatus() == Status.UP ? 200 : 503)
            .body(new HealthStatus(health));
    }
}

@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            // Simple query to test connection
            conn.createStatement().execute("SELECT 1");
            return Health.up().build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

#### Process Supervision

#### Supervisor Pattern
```java
public class Supervisor {
    
    private final List<Worker> workers = new ArrayList<>();
    private final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(1);
    
    public void supervise(Worker worker) {
        workers.add(worker);
        
        // Monitor worker health
        scheduler.scheduleAtFixedRate(() -> {
            if (!worker.isHealthy()) {
                logger.warn("Worker {} is unhealthy, restarting", worker.getId());
                restartWorker(worker);
            }
        }, 30, 30, TimeUnit.SECONDS);
    }
    
    private void restartWorker(Worker worker) {
        try {
            worker.stop();
            Thread.sleep(1000); // Brief pause
            worker.start();
            logger.info("Successfully restarted worker {}", worker.getId());
        } catch (Exception e) {
            logger.error("Failed to restart worker {}", worker.getId(), e);
        }
    }
}

interface Worker {
    String getId();
    boolean isHealthy();
    void start() throws Exception;
    void stop() throws Exception;
}
```

### Failure Recovery Strategies

#### Graceful Degradation
```java
@Service
public class GracefulDegradationService {
    
    @Autowired
    private CacheService cacheService;
    
    @Autowired
    private DatabaseService databaseService;
    
    @Autowired
    private ExternalApiService externalService;
    
    public Product getProduct(String productId) {
        try {
            // Try cache first
            Product product = cacheService.getProduct(productId);
            if (product != null) {
                return product;
            }
        } catch (Exception e) {
            logger.warn("Cache unavailable, falling back to database", e);
        }
        
        try {
            // Fall back to database
            return databaseService.getProduct(productId);
        } catch (Exception e) {
            logger.warn("Database unavailable, using default product", e);
            
            // Return minimal product data
            return new Product(productId, "Unknown Product", 0.0);
        }
    }
    
    public ProductDetails getProductDetails(String productId) {
        ProductDetails details = new ProductDetails();
        details.setProduct(getProduct(productId));
        
        try {
            // Try to get reviews (can fail)
            details.setReviews(reviewService.getReviews(productId));
        } catch (Exception e) {
            logger.warn("Reviews unavailable, proceeding without", e);
            details.setReviews(Collections.emptyList());
        }
        
        try {
            // Try to get recommendations (can fail)
            details.setRecommendations(
                recommendationService.getRecommendations(productId));
        } catch (Exception e) {
            logger.warn("Recommendations unavailable, proceeding without", e);
            details.setRecommendations(Collections.emptyList());
        }
        
        return details;
    }
}
```

#### State Synchronization
```java
@Service
public class StateSynchronizationService {
    
    @Autowired
    private StateRepository stateRepository;
    
    @Autowired
    private ClusterMembershipService clusterService;
    
    public void synchronizeState(String nodeId) {
        // Get current cluster state
        Map<String, NodeState> clusterState = clusterService.getClusterState();
        
        // Find healthy nodes
        List<String> healthyNodes = clusterState.entrySet().stream()
            .filter(entry -> entry.getValue().isHealthy())
            .map(Map.Entry::getKey)
            .filter(id -> !id.equals(nodeId))
            .collect(Collectors.toList());
        
        if (healthyNodes.isEmpty()) {
            logger.warn("No healthy nodes available for synchronization");
            return;
        }
        
        // Synchronize from a healthy node
        String syncSource = healthyNodes.get(0);
        synchronizeFromNode(syncSource);
    }
    
    private void synchronizeFromNode(String sourceNodeId) {
        try {
            // Get state snapshot from source node
            StateSnapshot snapshot = clusterService.getStateSnapshot(sourceNodeId);
            
            // Apply snapshot locally
            stateRepository.applySnapshot(snapshot);
            
            logger.info("Successfully synchronized state from node {}", sourceNodeId);
            
        } catch (Exception e) {
            logger.error("Failed to synchronize state from node {}", sourceNodeId, e);
        }
    }
}
```

## Monitoring Fault Tolerance

### Fault Tolerance Metrics
```java
@Service
public class FaultToleranceMetrics {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter circuitBreakerOpens = Counter.builder("circuit_breaker_opens_total")
        .description("Total circuit breaker open events")
        .register(meterRegistry);
    
    private final Counter bulkheadRejections = Counter.builder("bulkhead_rejections_total")
        .description("Total bulkhead request rejections")
        .register(meterRegistry);
    
    private final Counter retryAttempts = Counter.builder("retry_attempts_total")
        .description("Total retry attempts")
        .register(meterRegistry);
    
    private final Counter timeouts = Counter.builder("timeouts_total")
        .description("Total timeout events")
        .register(meterRegistry);
    
    private final Histogram recoveryTime = Histogram.builder("fault_recovery_time_seconds")
        .description("Time taken to recover from faults")
        .register(meterRegistry);
    
    public void recordCircuitBreakerOpen() {
        circuitBreakerOpens.increment();
    }
    
    public void recordBulkheadRejection() {
        bulkheadRejections.increment();
    }
    
    public void recordRetryAttempt() {
        retryAttempts.increment();
    }
    
    public void recordTimeout() {
        timeouts.increment();
    }
    
    public void recordRecoveryTime(long recoveryTimeSeconds) {
        recoveryTime.observe(recoveryTimeSeconds);
    }
}
```

### Alerting
```java
@Service
public class FaultToleranceAlerting {
    
    @Autowired
    private AlertService alertService;
    
    @Autowired
    private FaultToleranceMetrics metrics;
    
    @Scheduled(fixedRate = 60000) // Check every minute
    public void checkFaultToleranceHealth() {
        
        // Check circuit breaker status
        if (metrics.getCircuitBreakerOpenRate() > 0.1) { // 10% of requests failing
            alertService.sendAlert(
                "High circuit breaker open rate detected",
                "Circuit breaker is opening frequently, indicating downstream issues"
            );
        }
        
        // Check bulkhead saturation
        if (metrics.getBulkheadRejectionRate() > 0.05) { // 5% of requests rejected
            alertService.sendAlert(
                "Bulkhead saturation detected",
                "Bulkhead is rejecting requests, consider scaling"
            );
        }
        
        // Check timeout rate
        if (metrics.getTimeoutRate() > 0.02) { // 2% of requests timing out
            alertService.sendAlert(
                "High timeout rate detected",
                "Requests are timing out frequently, check service performance"
            );
        }
    }
}
```

## Testing Fault Tolerance

### Chaos Engineering
```java
@Service
public class ChaosEngineeringService {
    
    @Autowired
    private InstanceManager instanceManager;
    
    @Autowired
    private LoadGenerator loadGenerator;
    
    public void runChaosExperiment() {
        logger.info("Starting chaos engineering experiment");
        
        // Phase 1: Establish baseline
        logger.info("Phase 1: Establishing baseline performance");
        PerformanceMetrics baseline = measurePerformance(5); // 5 minutes
        
        // Phase 2: Inject faults
        logger.info("Phase 2: Injecting faults");
        
        // Kill 20% of instances
        List<String> instancesToKill = selectRandomInstances(0.2);
        instancesToKill.forEach(instanceManager::killInstance);
        
        // Wait for system to stabilize
        Thread.sleep(Duration.ofMinutes(2).toMillis());
        
        // Phase 3: Measure impact
        logger.info("Phase 3: Measuring fault impact");
        PerformanceMetrics duringFault = measurePerformance(3);
        
        // Phase 4: Recovery
        logger.info("Phase 4: Testing recovery");
        instanceManager.restartInstances(instancesToKill);
        
        // Wait for recovery
        Thread.sleep(Duration.ofMinutes(5).toMillis());
        
        // Phase 5: Measure recovery
        logger.info("Phase 5: Measuring recovery performance");
        PerformanceMetrics afterRecovery = measurePerformance(5);
        
        // Generate report
        generateChaosReport(baseline, duringFault, afterRecovery);
    }
    
    private List<String> selectRandomInstances(double percentage) {
        List<String> allInstances = instanceManager.getAllInstances();
        int count = (int) (allInstances.size() * percentage);
        Collections.shuffle(allInstances);
        return allInstances.subList(0, count);
    }
}
```

### Fault Injection Testing
```java
@SpringBootTest
public class FaultInjectionTest {
    
    @Autowired
    private CircuitBreakerService circuitBreakerService;
    
    @Autowired
    private FaultInjector faultInjector;
    
    @Test
    public void testCircuitBreakerUnderFaultInjection() {
        // Start with healthy service
        assertTrue(circuitBreakerService.isHealthy());
        
        // Inject faults
        faultInjector.injectFault("external-service", 
            FaultType.CONNECTION_TIMEOUT, 0.8); // 80% failure rate
        
        // Circuit breaker should eventually open
        await().atMost(30, TimeUnit.SECONDS)
            .until(() -> circuitBreakerService.getState() == CircuitBreaker.State.OPEN);
        
        // Service should be marked as unhealthy
        assertFalse(circuitBreakerService.isHealthy());
        
        // Stop fault injection
        faultInjector.clearFault("external-service");
        
        // Circuit breaker should eventually close
        await().atMost(60, TimeUnit.SECONDS)
            .until(() -> circuitBreakerService.getState() == CircuitBreaker.State.CLOSED);
        
        // Service should recover
        assertTrue(circuitBreakerService.isHealthy());
    }
}
```

## Real-World Fault Tolerance Examples

### Netflix Hystrix
```java
@Service
public class HystrixExampleService {
    
    @HystrixCommand(
        fallbackMethod = "fallbackMethod",
        commandProperties = {
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "20"),
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "5000"),
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000")
        },
        threadPoolProperties = {
            @HystrixProperty(name = "coreSize", value = "10"),
            @HystrixProperty(name = "maxQueueSize", value = "5")
        }
    )
    public String callExternalService(String request) {
        // This will be wrapped with circuit breaker, bulkhead, and timeout
        return restTemplate.postForObject("http://external-service/api", request, String.class);
    }
    
    public String fallbackMethod(String request) {
        // Fallback when circuit is open or service fails
        return "Fallback response for request: " + request;
    }
}
```

### Kubernetes Pod Disruption Budget
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2  # At least 2 pods must be available
  selector:
    matchLabels:
      app: my-app

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resilient-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:latest
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
```

### Database Failover
```java
@Configuration
public class DatabaseFailoverConfig {
    
    @Bean
    @Primary
    public DataSource primaryDataSource() {
        return createDataSource("jdbc:mysql://primary-db:3306/mydb");
    }
    
    @Bean
    public DataSource replicaDataSource() {
        return createDataSource("jdbc:mysql://replica-db:3306/mydb");
    }
    
    @Bean
    public DataSource failoverDataSource(
            @Qualifier("primaryDataSource") DataSource primary,
            @Qualifier("replicaDataSource") DataSource replica) {
        
        FailoverDataSource failover = new FailoverDataSource();
        failover.setPrimaryDataSource(primary);
        failover.setReplicaDataSource(replica);
        
        // Configure failover detection
        failover.setFailoverDetectionInterval(5000); // Check every 5 seconds
        failover.setFailoverThreshold(3); // 3 consecutive failures trigger failover
        
        return failover;
    }
    
    private DataSource createDataSource(String url) {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(url);
        ds.setUsername("user");
        ds.setPassword("password");
        ds.setConnectionTimeout(5000);
        return ds;
    }
}
```

## Conclusion

Fault tolerance is essential for building reliable distributed systems that can withstand failures and continue operating. By implementing patterns like circuit breakers, bulkheads, retries, and replication, systems can gracefully handle faults and maintain service availability.

**Key Takeaways:**
- **Multiple Protection Layers**: Combine circuit breakers, bulkheads, retries, and timeouts
- **Monitoring and Alerting**: Track fault tolerance metrics and set up alerts
- **Testing**: Use chaos engineering and fault injection testing
- **Graceful Degradation**: Design systems to work with reduced functionality
- **Recovery**: Implement automatic recovery and state synchronization

Effective fault tolerance requires understanding failure modes, implementing appropriate patterns, and continuously monitoring and testing system resilience.
