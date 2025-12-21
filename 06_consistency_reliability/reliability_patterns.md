# Reliability Patterns

Reliability patterns ensure systems remain operational and perform correctly under various conditions. These patterns help build systems that can handle failures, scale effectively, and maintain service quality. This guide covers essential reliability patterns used in building robust distributed systems.

## What is System Reliability?

System reliability is the ability of a system to consistently deliver correct service. It encompasses availability, fault tolerance, performance, and the ability to recover from failures.

### Key Reliability Metrics

#### Availability
- **Uptime Percentage**: Time system is operational
- **Mean Time Between Failures (MTBF)**: Average time between failures
- **Mean Time To Recovery (MTTR)**: Average time to recover from failures

#### Performance Reliability
- **Response Time Consistency**: Stable response times under load
- **Throughput Stability**: Consistent request processing capacity
- **Error Rate**: Percentage of failed requests

#### Data Reliability
- **Data Durability**: Assurance data won't be lost
- **Data Consistency**: Data remains accurate and consistent
- **Data Integrity**: Data maintains its correctness

## Reliability Patterns

### 1. Health Check Pattern

#### Application Health Checks
```java
@RestController
public class HealthCheckController {
    
    @Autowired
    private List<HealthIndicator> healthIndicators;
    
    @Autowired
    private MetricsService metricsService;
    
    @GetMapping("/health")
    public ResponseEntity<HealthStatus> health() {
        HealthStatus status = new HealthStatus();
        status.setTimestamp(Instant.now());
        
        boolean overallHealthy = true;
        Map<String, ComponentHealth> components = new HashMap<>();
        
        for (HealthIndicator indicator : healthIndicators) {
            try {
                Health health = indicator.health();
                ComponentHealth componentHealth = new ComponentHealth(
                    health.getStatus().equals(Status.UP),
                    health.getStatus().toString(),
                    health.getDetails()
                );
                components.put(indicator.getComponentName(), componentHealth);
                
                if (!componentHealth.isHealthy()) {
                    overallHealthy = false;
                }
                
            } catch (Exception e) {
                components.put(indicator.getComponentName(), 
                    new ComponentHealth(false, "ERROR", Map.of("error", e.getMessage())));
                overallHealthy = false;
            }
        }
        
        status.setHealthy(overallHealthy);
        status.setComponents(components);
        
        // Add performance metrics
        status.setMetrics(metricsService.getCurrentMetrics());
        
        HttpStatus httpStatus = overallHealthy ? HttpStatus.OK : HttpStatus.SERVICE_UNAVAILABLE;
        return ResponseEntity.status(httpStatus).body(status);
    }
    
    @GetMapping("/health/ready")
    public ResponseEntity<Void> readiness() {
        // Check if application is ready to serve traffic
        boolean ready = checkReadiness();
        HttpStatus status = ready ? HttpStatus.OK : HttpStatus.SERVICE_UNAVAILABLE;
        return ResponseEntity.status(status).build();
    }
    
    @GetMapping("/health/live")
    public ResponseEntity<Void> liveness() {
        // Check if application is alive (not deadlocked, etc.)
        boolean alive = checkLiveness();
        HttpStatus status = alive ? HttpStatus.OK : HttpStatus.SERVICE_UNAVAILABLE;
        return ResponseEntity.status(status).build();
    }
    
    private boolean checkReadiness() {
        // Check database connectivity, external services, etc.
        return databaseService.isConnected() && 
               externalServices.allAvailable() &&
               applicationState.isInitialized();
    }
    
    private boolean checkLiveness() {
        // Check for deadlocks, infinite loops, etc.
        return !Thread.holdsLock(this) && // No deadlocks
               System.currentTimeMillis() - lastActivity < 30000; // Recent activity
    }
}
```

#### Database Health Check
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DataSource dataSource;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            // Test connection
            if (!conn.isValid(5)) {
                return Health.down().withDetail("connection", "invalid").build();
            }
            
            // Test simple query
            Integer result = jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            if (!result.equals(1)) {
                return Health.down().withDetail("query", "failed").build();
            }
            
            // Check connection pool stats
            if (dataSource instanceof HikariDataSource) {
                HikariDataSource hikari = (HikariDataSource) dataSource;
                return Health.up()
                    .withDetail("activeConnections", hikari.getHikariPoolMXBean().getActiveConnections())
                    .withDetail("idleConnections", hikari.getHikariPoolMXBean().getIdleConnections())
                    .withDetail("totalConnections", hikari.getHikariPoolMXBean().getTotalConnections())
                    .build();
            }
            
            return Health.up().build();
            
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

### 2. Circuit Breaker Pattern

#### Distributed Circuit Breaker
```java
@Service
public class DistributedCircuitBreaker {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private final String circuitKey;
    private final int failureThreshold;
    private final Duration timeout;
    private final Duration retryTimeout;
    
    public DistributedCircuitBreaker(String serviceName, int failureThreshold, 
                                   Duration timeout, Duration retryTimeout) {
        this.circuitKey = "circuit:" + serviceName;
        this.failureThreshold = failureThreshold;
        this.timeout = timeout;
        this.retryTimeout = retryTimeout;
    }
    
    public <T> T execute(Supplier<T> operation) throws CircuitBreakerException {
        CircuitState state = getCurrentState();
        
        if (state.getState() == CircuitState.State.OPEN) {
            if (shouldAttemptReset(state)) {
                setState(CircuitState.State.HALF_OPEN);
                state = getCurrentState();
            } else {
                throw new CircuitBreakerException("Circuit is open");
            }
        }
        
        try {
            T result = operation.get();
            
            // Success - reset circuit
            if (state.getState() == CircuitState.State.HALF_OPEN) {
                resetCircuit();
            }
            
            return result;
            
        } catch (Exception e) {
            recordFailure();
            
            if (state.getState() == CircuitState.State.HALF_OPEN) {
                openCircuit();
            }
            
            throw e;
        }
    }
    
    private CircuitState getCurrentState() {
        Map<Object, Object> stateMap = redisTemplate.opsForHash().entries(circuitKey);
        return new CircuitState(
            CircuitState.State.valueOf((String) stateMap.getOrDefault("state", "CLOSED")),
            (Integer) stateMap.getOrDefault("failureCount", 0),
            (Long) stateMap.getOrDefault("lastFailureTime", 0L)
        );
    }
    
    private void setState(CircuitState.State newState) {
        redisTemplate.opsForHash().put(circuitKey, "state", newState.toString());
    }
    
    private void recordFailure() {
        redisTemplate.opsForHash().increment(circuitKey, "failureCount", 1);
        redisTemplate.opsForHash().put(circuitKey, "lastFailureTime", System.currentTimeMillis());
        
        Integer failureCount = (Integer) redisTemplate.opsForHash().get(circuitKey, "failureCount");
        if (failureCount >= failureThreshold) {
            openCircuit();
        }
    }
    
    private void openCircuit() {
        setState(CircuitState.State.OPEN);
    }
    
    private void resetCircuit() {
        redisTemplate.opsForHash().put(circuitKey, "state", CircuitState.State.CLOSED.toString());
        redisTemplate.opsForHash().put(circuitKey, "failureCount", 0);
    }
    
    private boolean shouldAttemptReset(CircuitState state) {
        long timeSinceLastFailure = System.currentTimeMillis() - state.getLastFailureTime();
        return timeSinceLastFailure > retryTimeout.toMillis();
    }
}
```

### 3. Bulkhead Pattern

#### Resource Pool Isolation
```java
public class ResourcePoolBulkhead<T extends AutoCloseable> {
    
    private final Semaphore semaphore;
    private final BlockingQueue<T> availableResources;
    private final int maxPoolSize;
    private final Supplier<T> resourceFactory;
    
    public ResourcePoolBulkhead(int maxPoolSize, Supplier<T> resourceFactory) {
        this.maxPoolSize = maxPoolSize;
        this.resourceFactory = resourceFactory;
        this.semaphore = new Semaphore(maxPoolSize);
        this.availableResources = new ArrayBlockingQueue<>(maxPoolSize);
        
        // Pre-populate pool
        for (int i = 0; i < maxPoolSize; i++) {
            availableResources.offer(resourceFactory.get());
        }
    }
    
    public <R> R execute(Function<T, R> operation) throws InterruptedException {
        // Acquire permit (bulkhead)
        semaphore.acquire();
        
        T resource = null;
        try {
            // Get resource from pool
            resource = availableResources.poll(5, TimeUnit.SECONDS);
            if (resource == null) {
                throw new TimeoutException("No resource available in pool");
            }
            
            // Execute operation
            return operation.apply(resource);
            
        } finally {
            // Return resource to pool
            if (resource != null) {
                availableResources.offer(resource);
            }
            semaphore.release();
        }
    }
    
    public void shutdown() {
        availableResources.forEach(resource -> {
            try {
                resource.close();
            } catch (Exception e) {
                logger.warn("Error closing resource", e);
            }
        });
        availableResources.clear();
    }
}
```

#### Service Isolation
```java
@Configuration
public class ServiceIsolationConfig {
    
    @Bean
    public ThreadPoolExecutor userServiceExecutor() {
        return new ThreadPoolExecutor(
            5, 10, 60, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(50),
            new ThreadFactoryBuilder().setNameFormat("user-service-%d").build(),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
    
    @Bean
    public ThreadPoolExecutor orderServiceExecutor() {
        return new ThreadPoolExecutor(
            3, 8, 60, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(30),
            new ThreadFactoryBuilder().setNameFormat("order-service-%d").build(),
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
public class IsolatedUserService {
    
    @Autowired
    private ThreadPoolExecutor executor;
    
    public CompletableFuture<User> getUserAsync(Long userId) {
        return CompletableFuture.supplyAsync(() -> {
            // This operation is isolated in its own thread pool
            return userRepository.findById(userId);
        }, executor);
    }
}
```

### 4. Retry Pattern with Backoff

#### Exponential Backoff Retry
```java
public class ExponentialBackoffRetry<T> {
    
    private final int maxRetries;
    private final Duration initialDelay;
    private final double backoffMultiplier;
    private final Duration maxDelay;
    private final Set<Class<? extends Exception>> retryableExceptions;
    
    public ExponentialBackoffRetry(int maxRetries, Duration initialDelay, 
                                 double backoffMultiplier, Duration maxDelay,
                                 Set<Class<? extends Exception>> retryableExceptions) {
        this.maxRetries = maxRetries;
        this.initialDelay = initialDelay;
        this.backoffMultiplier = backoffMultiplier;
        this.maxDelay = maxDelay;
        this.retryableExceptions = retryableExceptions;
    }
    
    public T execute(Supplier<T> operation) throws Exception {
        Exception lastException = null;
        Duration delay = initialDelay;
        
        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                return operation.get();
                
            } catch (Exception e) {
                lastException = e;
                
                if (attempt == maxRetries || !isRetryable(e)) {
                    throw e;
                }
                
                logger.warn("Attempt {} failed, retrying in {}", attempt + 1, delay);
                Thread.sleep(delay.toMillis());
                
                // Calculate next delay with jitter
                delay = Duration.ofMillis(
                    Math.min((long)(delay.toMillis() * backoffMultiplier), 
                            maxDelay.toMillis())
                ).plusMillis(ThreadLocalRandom.current().nextLong(100)); // Add jitter
            }
        }
        
        throw lastException;
    }
    
    private boolean isRetryable(Exception e) {
        return retryableExceptions.stream()
            .anyMatch(retryable -> retryable.isAssignableFrom(e.getClass()));
    }
}
```

#### Context-Aware Retry
```java
@Service
public class ContextAwareRetryService {
    
    @Autowired
    private RetryPolicyProvider policyProvider;
    
    public <T> T executeWithRetry(String operationType, Supplier<T> operation) throws Exception {
        RetryPolicy policy = policyProvider.getPolicy(operationType);
        
        // Get current context (user, request type, etc.)
        ExecutionContext context = ExecutionContext.current();
        
        // Adjust policy based on context
        if (context.isHighPriority()) {
            policy = policy.withMaxRetries(Math.min(policy.getMaxRetries() + 2, 5));
        }
        
        if (context.isBackgroundOperation()) {
            policy = policy.withInitialDelay(policy.getInitialDelay().multipliedBy(2));
        }
        
        return policy.execute(operation);
    }
}
```

### 5. Timeout Pattern

#### Hierarchical Timeouts
```java
public class HierarchicalTimeoutService {
    
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(10);
    
    public <T> CompletableFuture<T> executeWithTimeout(
            Supplier<CompletableFuture<T>> operation,
            Duration timeout,
            String operationName) {
        
        CompletableFuture<T> future = operation.get();
        
        // Schedule timeout
        ScheduledFuture<?> timeoutFuture = scheduler.schedule(() -> {
            if (!future.isDone()) {
                logger.warn("Operation {} timed out after {}", operationName, timeout);
                future.completeExceptionally(new TimeoutException(
                    operationName + " timed out after " + timeout));
            }
        }, timeout.toMillis(), TimeUnit.MILLISECONDS);
        
        // Cancel timeout when operation completes
        future.whenComplete((result, error) -> {
            timeoutFuture.cancel(false);
            if (error != null) {
                logger.error("Operation {} failed", operationName, error);
            } else {
                logger.debug("Operation {} completed successfully", operationName);
            }
        });
        
        return future;
    }
    
    public <T> T executeSyncWithTimeout(Supplier<T> operation, Duration timeout, 
                                      String operationName) throws Exception {
        
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<T> future = executor.submit(operation::get);
        
        try {
            return future.get(timeout.toMillis(), TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            logger.warn("Operation {} timed out", operationName);
            future.cancel(true);
            throw e;
        } finally {
            executor.shutdown();
        }
    }
}
```

### 6. Fallback Pattern

#### Multiple Fallback Levels
```java
@Service
public class FallbackService {
    
    @Autowired
    private PrimaryService primaryService;
    
    @Autowired
    private SecondaryService secondaryService;
    
    @Autowired
    private EmergencyService emergencyService;
    
    public <T> T executeWithFallbacks(Supplier<T> primaryOperation, 
                                    String operationName) throws Exception {
        
        try {
            // Try primary service
            logger.debug("Attempting primary operation: {}", operationName);
            return primaryOperation.get();
            
        } catch (PrimaryServiceException e) {
            logger.warn("Primary service failed for {}, trying secondary", operationName, e);
            
            try {
                // Try secondary service
                return secondaryService.execute(operationName);
                
            } catch (SecondaryServiceException e2) {
                logger.error("Secondary service also failed for {}, using emergency fallback", 
                           operationName, e2);
                
                // Emergency fallback - return cached/default data
                return emergencyService.getFallbackData(operationName);
            }
        }
    }
    
    public <T> T executeWithCustomFallback(Supplier<T> operation, 
                                         Function<Exception, T> fallbackFunction,
                                         String operationName) {
        
        try {
            return operation.get();
        } catch (Exception e) {
            logger.warn("Operation {} failed, using custom fallback", operationName, e);
            return fallbackFunction.apply(e);
        }
    }
}
```

### 7. Rate Limiting Pattern

#### Token Bucket Rate Limiter
```java
public class TokenBucketRateLimiter {
    
    private final long capacity;        // Maximum tokens
    private final double refillRate;    // Tokens per second
    private double tokens;             // Current tokens
    private long lastRefillTime;       // Last refill timestamp
    
    public TokenBucketRateLimiter(long capacity, double refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = capacity;
        this.lastRefillTime = System.nanoTime();
    }
    
    public synchronized boolean tryConsume(double tokensToConsume) {
        refill();
        
        if (tokens >= tokensToConsume) {
            tokens -= tokensToConsume;
            return true;
        }
        
        return false;
    }
    
    private void refill() {
        long now = System.nanoTime();
        double elapsedSeconds = (now - lastRefillTime) / 1_000_000_000.0;
        
        long newTokens = (long) (elapsedSeconds * refillRate);
        if (newTokens > 0) {
            tokens = Math.min(capacity, tokens + newTokens);
            lastRefillTime = now;
        }
    }
    
    public synchronized void setCapacity(long newCapacity) {
        this.capacity = newCapacity;
        this.tokens = Math.min(tokens, capacity);
    }
    
    public synchronized void setRefillRate(double newRefillRate) {
        this.refillRate = newRefillRate;
    }
}
```

#### Distributed Rate Limiting
```java
@Service
public class DistributedRateLimiter {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public boolean isAllowed(String clientId, int maxRequests, long windowSeconds) {
        String key = "ratelimit:" + clientId + ":" + (System.currentTimeMillis() / 1000 / windowSeconds);
        
        // Use Redis atomic operations
        Long currentCount = redisTemplate.opsForValue().increment(key);
        
        if (currentCount == 1) {
            // Set expiration for the window
            redisTemplate.expire(key, Duration.ofSeconds(windowSeconds));
        }
        
        return currentCount <= maxRequests;
    }
    
    public boolean isAllowedSlidingWindow(String clientId, int maxRequests, long windowSeconds) {
        String key = "ratelimit:" + clientId;
        long now = System.currentTimeMillis();
        long windowStart = now - (windowSeconds * 1000);
        
        // Add current request to sorted set
        redisTemplate.opsForZSet().add(key, String.valueOf(now), now);
        
        // Remove old requests outside the window
        redisTemplate.opsForZSet().removeRangeByScore(key, 0, windowStart);
        
        // Count requests in current window
        Long currentCount = redisTemplate.opsForZSet().size(key);
        
        // Set expiration on the key
        redisTemplate.expire(key, Duration.ofSeconds(windowSeconds));
        
        return currentCount != null && currentCount <= maxRequests;
    }
}
```

## Reliability Monitoring

### Service Level Objectives (SLOs)

#### SLO Definition and Monitoring
```java
@Service
public class SLOMonitor {
    
    @Autowired
    private MetricsService metricsService;
    
    private final List<SLODefinition> slos = Arrays.asList(
        new SLODefinition("api_response_time", "< 500ms", 0.95, Duration.ofHours(1)),
        new SLODefinition("api_availability", "> 99.9%", 0.999, Duration.ofHours(1)),
        new SLODefinition("error_rate", "< 0.1%", 0.001, Duration.ofHours(1))
    );
    
    @Scheduled(fixedRate = 300000) // Check every 5 minutes
    public void checkSLOs() {
        for (SLODefinition slo : slos) {
            double actualValue = getActualValue(slo);
            boolean sloMet = evaluateSLO(slo, actualValue);
            
            if (!sloMet) {
                alertService.sendAlert(
                    "SLO Violation: " + slo.getName(),
                    String.format("SLO %s requires %s, actual: %.4f", 
                                slo.getName(), slo.getTarget(), actualValue)
                );
            }
            
            // Record SLO compliance
            metricsService.recordSLOMetric(slo.getName(), actualValue, sloMet);
        }
    }
    
    private double getActualValue(SLODefinition slo) {
        switch (slo.getName()) {
            case "api_response_time":
                return metricsService.get95thPercentileResponseTime(Duration.ofHours(1));
            case "api_availability":
                return metricsService.getAvailabilityPercentage(Duration.ofHours(1));
            case "error_rate":
                return metricsService.getErrorRate(Duration.ofHours(1));
            default:
                return 0.0;
        }
    }
    
    private boolean evaluateSLO(SLODefinition slo, double actualValue) {
        // Parse target (e.g., "< 500ms", "> 99.9%", "< 0.1%")
        String operator = slo.getTarget().split(" ")[0];
        double targetValue = parseTargetValue(slo.getTarget());
        
        switch (operator) {
            case "<": return actualValue < targetValue;
            case ">": return actualValue > targetValue;
            case "<=": return actualValue <= targetValue;
            case ">=": return actualValue >= targetValue;
            default: return false;
        }
    }
    
    private double parseTargetValue(String target) {
        // Parse values like "500ms", "99.9%", "0.1%"
        String value = target.split(" ")[1];
        if (value.endsWith("ms")) {
            return Double.parseDouble(value.substring(0, value.length() - 2));
        } else if (value.endsWith("%")) {
            return Double.parseDouble(value.substring(0, value.length() - 1)) / 100.0;
        } else {
            return Double.parseDouble(value);
        }
    }
}
```

### Reliability Metrics Dashboard
```java
@Service
public class ReliabilityDashboard {
    
    @Autowired
    private MetricsService metricsService;
    
    public ReliabilityReport generateReport(Duration period) {
        ReliabilityReport report = new ReliabilityReport();
        report.setPeriod(period);
        report.setGeneratedAt(Instant.now());
        
        // Availability metrics
        report.setUptimePercentage(metricsService.getUptimePercentage(period));
        report.setMTBF(metricsService.getMTBF(period));
        report.setMTTR(metricsService.getMTTR(period));
        
        // Performance metrics
        report.setAverageResponseTime(metricsService.getAverageResponseTime(period));
        report.set95thPercentileResponseTime(metricsService.get95thPercentileResponseTime(period));
        report.setThroughput(metricsService.getAverageThroughput(period));
        
        // Error metrics
        report.setErrorRate(metricsService.getErrorRate(period));
        report.setTopErrors(metricsService.getTopErrors(period, 10));
        
        // Capacity metrics
        report.setResourceUtilization(metricsService.getResourceUtilization(period));
        report.setScalingEvents(metricsService.getScalingEvents(period));
        
        // Generate recommendations
        report.setRecommendations(generateRecommendations(report));
        
        return report;
    }
    
    private List<String> generateRecommendations(ReliabilityReport report) {
        List<String> recommendations = new ArrayList<>();
        
        if (report.getUptimePercentage() < 0.999) {
            recommendations.add("Consider implementing additional redundancy to improve uptime");
        }
        
        if (report.get95thPercentileResponseTime() > 1000) {
            recommendations.add("Response times are high - consider optimizing database queries or adding caching");
        }
        
        if (report.getErrorRate() > 0.001) {
            recommendations.add("Error rate is above 0.1% - investigate and fix root causes");
        }
        
        if (report.getResourceUtilization().getCpuUtilization() > 0.8) {
            recommendations.add("CPU utilization is high - consider horizontal scaling");
        }
        
        return recommendations;
    }
}
```

## Testing Reliability

### Chaos Engineering
```java
@Service
public class ChaosEngineeringService {
    
    @Autowired
    private InstanceManager instanceManager;
    
    @Autowired
    private TrafficGenerator trafficGenerator;
    
    @Autowired
    private ReliabilityMonitor monitor;
    
    public ChaosExperimentResult runReliabilityExperiment() {
        ChaosExperimentResult result = new ChaosExperimentResult();
        
        // Phase 1: Baseline measurement
        result.setBaselineMetrics(measureReliabilityMetrics("baseline", Duration.ofMinutes(10)));
        
        // Phase 2: Inject failures
        List<String> killedInstances = killRandomInstances(0.3); // Kill 30% of instances
        networkPartitionRandomNodes(0.2); // Partition 20% of nodes
        
        result.setFailureInjectionTime(Instant.now());
        
        // Phase 3: Measure during failure
        result.setDuringFailureMetrics(measureReliabilityMetrics("during_failure", Duration.ofMinutes(10)));
        
        // Phase 4: Recovery
        restoreInstances(killedInstances);
        healNetworkPartitions();
        
        // Phase 5: Measure after recovery
        result.setAfterRecoveryMetrics(measureReliabilityMetrics("after_recovery", Duration.ofMinutes(10)));
        
        // Analyze results
        result.setAnalysis(analyzeExperimentResults(result));
        
        return result;
    }
    
    private ReliabilityMetrics measureReliabilityMetrics(String phase, Duration duration) {
        // Start monitoring
        Instant startTime = Instant.now();
        
        // Generate traffic
        trafficGenerator.generateLoad(duration);
        
        // Wait for measurements
        try {
            Thread.sleep(duration.toMillis());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        // Collect metrics
        return monitor.getMetrics(startTime, Instant.now());
    }
    
    private ExperimentAnalysis analyzeExperimentResults(ChaosExperimentResult result) {
        ExperimentAnalysis analysis = new ExperimentAnalysis();
        
        // Compare metrics
        double availabilityDegradation = result.getBaselineMetrics().getAvailability() - 
                                       result.getDuringFailureMetrics().getAvailability();
        
        analysis.setAvailabilityImpact(availabilityDegradation);
        analysis.setRecoveryTime(Duration.between(result.getFailureInjectionTime(), 
                                                Instant.now().minus(Duration.ofMinutes(10))));
        
        // Generate insights
        if (availabilityDegradation > 0.1) {
            analysis.addInsight("High availability impact during failures - consider more redundancy");
        }
        
        if (result.getAfterRecoveryMetrics().getAverageResponseTime() > 
            result.getBaselineMetrics().getAverageResponseTime() * 1.5) {
            analysis.addInsight("Performance degradation persists after recovery - check for state inconsistencies");
        }
        
        return analysis;
    }
}
```

### Load Testing with Reliability Validation
```java
@SpringBootTest
public class ReliabilityLoadTest {
    
    @Autowired
    private LoadGenerator loadGenerator;
    
    @Autowired
    private ReliabilityMonitor monitor;
    
    @Test
    public void testReliabilityUnderLoad() {
        // Start monitoring
        Instant testStart = Instant.now();
        
        // Generate increasing load
        for (int concurrency : Arrays.asList(10, 50, 100, 200, 500)) {
            logger.info("Testing with {} concurrent users", concurrency);
            
            loadGenerator.setConcurrency(concurrency);
            loadGenerator.generateLoad(Duration.ofMinutes(5));
            
            // Check reliability metrics
            ReliabilitySnapshot snapshot = monitor.takeSnapshot();
            
            // Validate SLOs
            assertTrue(snapshot.getAvailability() > 0.99, 
                     "Availability below 99% at " + concurrency + " users");
            
            assertTrue(snapshot.getAverageResponseTime() < 2000, 
                     "Response time above 2s at " + concurrency + " users");
            
            assertTrue(snapshot.getErrorRate() < 0.01, 
                     "Error rate above 1% at " + concurrency + " users");
        }
        
        // Generate final report
        ReliabilityReport report = monitor.generateReport(testStart, Instant.now());
        logger.info("Load test completed. Report: {}", report);
    }
}
```

## Real-World Reliability Examples

### Netflix Chaos Monkey
```java
@Service
public class ChaosMonkeyService {
    
    @Autowired
    private InstanceManager instanceManager;
    
    @Autowired
    private AlertService alertService;
    
    @Scheduled(cron = "0 0 * * * ?") // Run once per hour during business hours
    public void runChaosMonkey() {
        if (!isBusinessHours()) {
            return;
        }
        
        // Select random instance to terminate
        Optional<String> instanceToKill = instanceManager.selectRandomInstance();
        
        if (instanceToKill.isPresent()) {
            String instanceId = instanceToKill.get();
            
            alertService.sendAlert(
                "Chaos Monkey",
                "Terminating instance " + instanceId + " for chaos testing"
            );
            
            // Terminate instance
            instanceManager.terminateInstance(instanceId);
            
            logger.info("Chaos Monkey terminated instance {}", instanceId);
        }
    }
    
    private boolean isBusinessHours() {
        ZonedDateTime now = ZonedDateTime.now(ZoneId.of("America/Los_Angeles"));
        int hour = now.getHour();
        DayOfWeek day = now.getDayOfWeek();
        
        // Business hours: Monday-Friday, 9 AM - 5 PM PST
        return day != DayOfWeek.SATURDAY && day != DayOfWeek.SUNDAY && 
               hour >= 9 && hour <= 17;
    }
}
```

### Amazon Service Health Dashboard
```java
@Service
public class ServiceHealthDashboard {
    
    @Autowired
    private List<HealthCheckService> healthCheckServices;
    
    @Autowired
    private AlertService alertService;
    
    @Scheduled(fixedRate = 30000) // Check every 30 seconds
    public void updateServiceHealth() {
        Map<String, ServiceHealth> healthStatus = new HashMap<>();
        
        for (HealthCheckService healthService : healthCheckServices) {
            try {
                ServiceHealth health = healthService.checkHealth();
                healthStatus.put(healthService.getServiceName(), health);
                
                // Alert on health changes
                if (health.getStatus() == HealthStatus.DOWN && 
                    health.getPreviousStatus() == HealthStatus.UP) {
                    
                    alertService.sendAlert(
                        "Service Down: " + healthService.getServiceName(),
                        "Service is no longer responding. Last error: " + health.getLastError()
                    );
                }
                
            } catch (Exception e) {
                healthStatus.put(healthService.getServiceName(), 
                    ServiceHealth.down("Health check failed: " + e.getMessage()));
            }
        }
        
        // Update dashboard
        dashboardService.updateHealthStatus(healthStatus);
        
        // Calculate overall system health
        double healthyServices = healthStatus.values().stream()
            .mapToInt(h -> h.getStatus() == HealthStatus.UP ? 1 : 0)
            .sum();
        
        double overallHealth = healthyServices / healthStatus.size();
        
        if (overallHealth < 0.95) { // Less than 95% services healthy
            alertService.sendAlert(
                "System Health Degraded",
                String.format("%.1f%% of services are healthy", overallHealth * 100)
            );
        }
    }
}
```

### Google SRE Error Budgets
```java
@Service
public class ErrorBudgetService {
    
    // Service Level Objectives
    private static final double TARGET_AVAILABILITY = 0.9995; // 99.95%
    private static final Duration MEASUREMENT_WINDOW = Duration.ofDays(28); // Monthly
    
    @Autowired
    private MetricsService metricsService;
    
    @Autowired
    private AlertService alertService;
    
    public ErrorBudgetStatus calculateErrorBudget() {
        ErrorBudgetStatus status = new ErrorBudgetStatus();
        status.setMeasurementWindow(MEASUREMENT_WINDOW);
        status.setTargetAvailability(TARGET_AVAILABILITY);
        
        // Calculate actual availability
        double actualAvailability = metricsService.getAvailabilityPercentage(MEASUREMENT_WINDOW);
        status.setActualAvailability(actualAvailability);
        
        // Calculate error budget
        double errorBudgetUsed = (1 - actualAvailability) / (1 - TARGET_AVAILABILITY);
        status.setErrorBudgetUsed(errorBudgetUsed);
        
        // Calculate remaining error budget
        double remainingBudget = Math.max(0, 1 - errorBudgetUsed);
        status.setRemainingBudget(remainingBudget);
        
        // Determine status
        if (errorBudgetUsed > 1.0) {
            status.setStatus(ErrorBudgetStatus.Status.EXHAUSTED);
            alertService.sendAlert(
                "Error Budget Exhausted",
                String.format("Error budget used: %.1f%%. System availability: %.4f%%", 
                            errorBudgetUsed * 100, actualAvailability * 100)
            );
        } else if (errorBudgetUsed > 0.8) {
            status.setStatus(ErrorBudgetStatus.Status.WARNING);
        } else {
            status.setStatus(ErrorBudgetStatus.Status.HEALTHY);
        }
        
        return status;
    }
    
    public boolean canDeploy(ErrorBudgetStatus status) {
        // Don't allow risky deployments when error budget is low
        return status.getRemainingBudget() > 0.2; // Keep 20% buffer
    }
}
```

## Conclusion

Reliability patterns are essential for building systems that can withstand failures and maintain service quality. By implementing health checks, circuit breakers, bulkheads, retries, timeouts, and fallbacks, systems become more resilient and dependable.

**Key Takeaways:**
- **Defense in Depth**: Multiple reliability patterns work together
- **Monitoring and Alerting**: Track reliability metrics continuously
- **Testing**: Use chaos engineering and load testing to validate reliability
- **Error Budgets**: Balance innovation with reliability requirements
- **Continuous Improvement**: Regularly assess and improve system reliability

Effective reliability requires understanding failure modes, implementing appropriate patterns, monitoring system behavior, and continuously improving based on observed performance.
