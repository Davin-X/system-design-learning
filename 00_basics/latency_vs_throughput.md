# Latency vs Throughput

Two fundamental performance metrics that often conflict in system design. Understanding their relationship is crucial for building efficient systems.

## What is Latency?

### Definition
Latency is the time it takes for a single operation to complete. It measures the delay between initiating a request and receiving a response.

**Examples:**
- **Network Latency**: Time for data to travel from source to destination (typically 10-100ms)
- **Disk Latency**: Time to read/write data from storage (SSD: ~0.1ms, HDD: ~10ms)
- **Application Latency**: Time for application to process request
- **Total Latency**: Sum of all latencies in the request path

### Measuring Latency
- **Average Latency**: Mean response time
- **Percentile Latency**: p50, p95, p99 response times
- **Tail Latency**: Highest response times (important for user experience)

**Why Percentiles Matter:**
- Average latency can hide performance issues
- p99 latency affects 1% of users but often the most important ones
- Tail latency determines system reliability

## What is Throughput?

### Definition
Throughput is the rate at which operations are completed. It measures how much work a system can perform in a given time period.

**Examples:**
- **Requests per Second (RPS)**: Web requests handled per second
- **Transactions per Second (TPS)**: Database transactions per second
- **Data Throughput**: Bytes/second transferred over network
- **Queries per Second (QPS)**: Database queries per second

### Measuring Throughput
- **Peak Throughput**: Maximum sustainable rate
- **Average Throughput**: Typical operational rate
- **Throughput Under Load**: How throughput changes with increased load

## The Latency-Throughput Trade-off

### Why They Conflict
Most optimization techniques improve one metric at the expense of the other.

**Latency-Focused Optimizations:**
- Caching reduces latency but may not increase throughput
- Fast storage (SSD vs HDD) reduces latency but costs more
- In-memory processing reduces latency but limits data size

**Throughput-Focused Optimizations:**
- Batching increases throughput but adds latency
- Compression increases throughput but adds processing latency
- Asynchronous processing increases throughput but may increase response latency

### Real-World Examples

#### Database Design
```sql
-- Low latency: Indexed query
SELECT * FROM users WHERE id = 123;  -- ~1ms latency

-- High throughput: Batch insert  
INSERT INTO logs VALUES (...), (...), (...);  -- 1000 inserts/second
```

#### Network Communication
- **Low Latency**: Individual HTTP requests (fast response)
- **High Throughput**: HTTP/2 multiplexing, connection pooling (more data per second)

#### Caching Strategies
- **Latency**: Cache entire objects (fast single lookups)
- **Throughput**: Cache compressed data, batch operations

## Little's Law

### The Mathematical Relationship
**L = λ × W**

Where:
- **L**: Average number of requests in system
- **λ**: Average arrival rate (throughput)
- **W**: Average time in system (latency)

**Implications:**
- If you increase throughput (λ), you must either:
  - Reduce latency (W), or
  - Increase system capacity (L - queue length)
- Queueing theory shows latency increases with utilization

### Practical Application
For a system handling 1000 requests/second with 50ms average latency:
- Average requests in system = 1000 × 0.05 = 50 requests
- To reduce latency to 25ms, you'd need to either:
  - Reduce throughput to 500 req/sec, or
  - Handle 100 requests simultaneously

## Optimizing for Both

### Techniques That Help Both

#### 1. Parallelization
Process multiple requests simultaneously to increase throughput while maintaining latency.

**Example:** Connection pooling allows multiple concurrent database connections.

#### 2. Pipelining
Process multiple operations in sequence without waiting for each to complete.

**Example:** HTTP/2 allows multiple requests over single connection.

#### 3. Caching
Reduce both latency (faster responses) and throughput load (fewer backend requests).

#### 4. Optimization
Efficient algorithms reduce both latency and increase throughput capacity.

### Load Balancing
Distribute load across multiple servers to improve both metrics.

## Monitoring and Measurement

### Key Metrics to Track

#### Latency Metrics
- Average response time
- 95th percentile response time
- 99th percentile response time
- Error rate by latency bucket

#### Throughput Metrics
- Requests per second
- Data transfer rate
- Queue depth
- Resource utilization (CPU, memory, network)

### Tools
- **Application Metrics**: Prometheus, Grafana
- **APM Tools**: New Relic, Datadog
- **Load Testing**: JMeter, k6, Artillery
- **Distributed Tracing**: Jaeger, Zipkin

## Common Performance Anti-Patterns

### 1. Optimizing Only One Metric
**Problem:** Focusing solely on latency ignores throughput limits
**Solution:** Set targets for both metrics, understand trade-offs

### 2. Ignoring Tail Latency
**Problem:** Average latency looks good but p99 is terrible
**Solution:** Monitor percentile latencies, optimize for worst-case performance

### 3. Premature Optimization
**Problem:** Optimizing before measuring actual bottlenecks
**Solution:** Profile first, then optimize identified bottlenecks

### 4. Not Considering Scale
**Problem:** Solutions work for 100 users but fail at 100,000
**Solution:** Design for expected scale from day one

## Latency vs Throughput in Different Systems

### Web Applications
- **Latency Critical**: User-facing pages (< 100ms ideal)
- **Throughput Important**: Handle traffic spikes
- **Trade-off**: Caching helps both, but adds complexity

### APIs
- **Latency Critical**: Mobile apps, real-time features
- **Throughput Important**: Background processing, batch operations
- **Trade-off**: Async processing vs immediate responses

### Databases
- **Latency Critical**: OLTP queries (< 10ms)
- **Throughput Important**: Bulk operations, analytics
- **Trade-off**: Indexing helps latency, denormalization helps throughput

### Streaming Systems
- **Latency Critical**: Real-time processing (< 100ms end-to-end)
- **Throughput Important**: High-volume data processing
- **Trade-off**: Buffering adds latency for throughput gains

## Practical Examples

### E-commerce Website
**Requirements:** Fast page loads + handle Black Friday traffic

**Latency Optimizations:**
- CDN for static assets
- Database query optimization
- In-memory caching (Redis)

**Throughput Optimizations:**
- Load balancing across multiple servers
- Database read replicas
- Async processing for non-critical operations

**Trade-off Decisions:**
- Cache product data for fast reads (latency) but accept eventual consistency
- Use microservices for independent scaling but add network latency

### API Service
**Requirements:** Low latency for mobile users + high throughput for background jobs

**Latency Optimizations:**
- API Gateway with response caching
- Regional data replication
- Optimized serialization (Protocol Buffers)

**Throughput Optimizations:**
- Asynchronous request processing
- Message queues for background work
- Horizontal scaling with auto-scaling

**Trade-off Decisions:**
- Accept higher latency for complex operations that require high throughput
- Use different consistency models for different endpoints

## Java Performance Considerations

### Latency Optimization
```java
// Low latency: Direct memory access
ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

// Connection pooling for fast database access
@Bean
public DataSource dataSource() {
    HikariConfig config = new HikariConfig();
    config.setMaximumPoolSize(20);
    config.setMinimumIdle(5);
    return new HikariDataSource(config);
}
```

### Throughput Optimization
```java
// High throughput: Async processing
@RestController
public class AsyncController {
    
    @Async
    @PostMapping("/process")
    public CompletableFuture<String> processAsync(@RequestBody Data data) {
        return CompletableFuture.supplyAsync(() -> {
            // Heavy processing
            return processData(data);
        });
    }
}

// Batch processing
@Service
public class BatchProcessor {
    @Scheduled(fixedRate = 5000)
    public void processBatch() {
        List<Data> batch = repository.findUnprocessed();
        batch.parallelStream().forEach(this::processItem);
    }
}
```

### Measuring Performance
```java
@Service
public class PerformanceMonitor {
    
    @Autowired
    private MeterRegistry registry;
    
    public <T> T measureLatency(String operation, Supplier<T> supplier) {
        Timer.Sample sample = Timer.start(registry);
        try {
            T result = supplier.get();
            sample.stop(Timer.builder("operation.latency")
                .tag("operation", operation)
                .register(registry));
            return result;
        } catch (Exception e) {
            sample.stop(Timer.builder("operation.latency")
                .tag("operation", operation)
                .tag("status", "error")
                .register(registry));
            throw e;
        }
    }
    
    public void recordThroughput(String operation) {
        registry.counter("operation.throughput", "operation", operation).increment();
    }
}
```

## Conclusion

Latency and throughput are interdependent metrics that require careful balancing in system design. Understanding their relationship through Little's Law helps make informed architectural decisions.

**Key Takeaways:**
- Measure both metrics, not just one
- Use percentiles to understand user experience
- Consider scale from the beginning
- Profile before optimizing
- Accept trade-offs based on business requirements

The optimal balance depends on your system's requirements: user-facing applications prioritize latency, while batch processing systems prioritize throughput.
