# Rate Limiting

Rate limiting controls the rate of requests sent or received by a system to prevent abuse, ensure fair usage, and protect against overwhelming load. It's a critical component of scalable systems that need to handle varying traffic patterns while maintaining service quality.

## What is Rate Limiting?

Rate limiting restricts the number of requests a client can make to an API or service within a specified time window. It prevents abuse, ensures fair resource allocation, and protects backend systems from overload.

### Key Concepts
- **Rate Limit**: Maximum number of requests allowed per time window
- **Time Window**: Period over which requests are counted (e.g., per minute, per hour)
- **Quota**: Total allowance for a longer period (e.g., daily limits)
- **Burst Allowance**: Temporary allowance above normal rate limits

### Why Rate Limiting Matters
- **Prevent Abuse**: Stop malicious users from overwhelming services
- **Fair Usage**: Ensure all users get equitable access to resources
- **Cost Control**: Prevent excessive resource consumption
- **System Protection**: Safeguard against cascading failures
- **Quality of Service**: Maintain consistent performance for legitimate users

## Rate Limiting Algorithms

### Token Bucket Algorithm

#### How It Works
- **Token Generation**: Tokens are added to bucket at fixed rate
- **Token Consumption**: Each request consumes one token
- **Bucket Capacity**: Maximum tokens that can be stored (burst allowance)
- **Token Exhaustion**: Requests are rejected when no tokens available

**Visual Representation:**
```
Tokens Generated: + + + + + (steady rate)
Bucket Capacity: [█████████░] 9/10 tokens

Request 1: Consume 1 token → [████████░░] 8/10 tokens
Request 2: Consume 1 token → [███████░░░] 7/10 tokens
```

#### Implementation
```java
public class TokenBucket {
    private final long capacity;        // Maximum tokens
    private final double refillRate;    // Tokens per second
    private double tokens;             // Current tokens
    private long lastRefillTime;       // Last refill timestamp
    
    public TokenBucket(long capacity, double refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = capacity;
        this.lastRefillTime = System.nanoTime();
    }
    
    public synchronized boolean tryConsume() {
        refill();
        
        if (tokens >= 1) {
            tokens -= 1;
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
}
```

#### Advantages
- **Smooth Rate Limiting**: Allows burst traffic within capacity
- **Predictable Behavior**: Steady token regeneration
- **Configurable Burst**: Adjustable burst allowance
- **Memory Efficient**: Simple state tracking

#### Use Cases
- **API Rate Limiting**: Allow bursts but maintain average rate
- **Traffic Shaping**: Smooth out traffic spikes
- **Resource Protection**: Prevent sudden load spikes

### Leaky Bucket Algorithm

#### How It Works
- **Request Queue**: Incoming requests enter a queue (bucket)
- **Fixed Processing Rate**: Requests are processed at constant rate
- **Overflow**: Excess requests are rejected (bucket "leaks")
- **No Burst Allowance**: Strict rate enforcement

**Visual Representation:**
```
Input Rate: ████████ (bursts allowed)
Bucket:     [██████████████] (queue)
Output Rate: ░░░░ ░░░░ ░░░░ (steady)
```

#### Implementation
```java
public class LeakyBucket {
    private final long capacity;        // Bucket capacity
    private final double leakRate;      // Requests per second
    private double waterLevel;          // Current water level
    private long lastLeakTime;          // Last leak timestamp
    
    public LeakyBucket(long capacity, double leakRate) {
        this.capacity = capacity;
        this.leakRate = leakRate;
        this.waterLevel = 0;
        this.lastLeakTime = System.nanoTime();
    }
    
    public synchronized boolean tryConsume() {
        leak();
        
        if (waterLevel < capacity) {
            waterLevel += 1;
            return true;
        }
        
        return false; // Bucket full
    }
    
    private void leak() {
        long now = System.nanoTime();
        double elapsedSeconds = (now - lastLeakTime) / 1_000_000_000.0;
        
        double leakedAmount = elapsedSeconds * leakRate;
        waterLevel = Math.max(0, waterLevel - leakedAmount);
        
        if (leakedAmount > 0) {
            lastLeakTime = now;
        }
    }
}
```

#### Advantages
- **Strict Rate Control**: Enforces exact processing rate
- **Memory Bounded**: Fixed queue size prevents unbounded growth
- **Predictable Output**: Steady request processing
- **Fair Queuing**: First-in-first-out processing

#### Disadvantages
- **No Burst Handling**: Cannot handle traffic spikes
- **Queue Management**: Requires queue implementation
- **Latency**: Queued requests experience delay

#### Use Cases
- **Real-Time Systems**: Require steady processing rates
- **Resource-Constrained**: Limited processing capacity
- **Fair Scheduling**: Equal treatment of all requests

### Fixed Window Algorithm

#### How It Works
- **Time Windows**: Divide time into fixed intervals (e.g., 1-minute windows)
- **Request Counting**: Count requests within each window
- **Window Reset**: Clear counter when window ends
- **Limit Enforcement**: Reject requests exceeding limit

#### Implementation
```java
public class FixedWindow {
    private final long windowSizeMillis;    // Window duration
    private final long maxRequests;         // Max requests per window
    private final Map<Long, Long> windows;  // Window -> request count
    
    public FixedWindow(long windowSizeMillis, long maxRequests) {
        this.windowSizeMillis = windowSizeMillis;
        this.maxRequests = maxRequests;
        this.windows = new ConcurrentHashMap<>();
    }
    
    public boolean tryConsume(String clientId) {
        long currentWindow = System.currentTimeMillis() / windowSizeMillis;
        long windowKey = (clientId.hashCode() << 32) | currentWindow;
        
        Long currentCount = windows.computeIfAbsent(windowKey, k -> 0L);
        
        if (currentCount >= maxRequests) {
            return false;
        }
        
        windows.put(windowKey, currentCount + 1);
        return true;
    }
    
    // Cleanup old windows (should be called periodically)
    public void cleanup() {
        long currentWindow = System.currentTimeMillis() / windowSizeMillis;
        long cutoffWindow = currentWindow - 1;
        
        windows.entrySet().removeIf(entry -> {
            long windowKey = entry.getKey();
            long window = windowKey & 0xFFFFFFFFL;
            return window < cutoffWindow;
        });
    }
}
```

#### Advantages
- **Simple Implementation**: Easy to understand and implement
- **Memory Efficient**: Minimal state tracking
- **Precise Windows**: Clear time boundaries

#### Disadvantages
- **Boundary Issues**: Requests pile up at window boundaries
- **Burst at Reset**: Sudden traffic spikes when windows reset
- **Inflexible**: Cannot handle gradual rate changes

#### Use Cases
- **Simple Rate Limiting**: Basic protection requirements
- **Periodic Limits**: Clear time-based boundaries
- **Resource Monitoring**: Track usage per time period

### Sliding Window Algorithm

#### How It Works
- **Rolling Windows**: Consider requests in moving time window
- **Weighted Requests**: Give more weight to recent requests
- **Smooth Transitions**: Avoid boundary problems of fixed windows
- **Configurable Granularity**: Adjustable window size and precision

#### Implementation
```java
public class SlidingWindow {
    private final long windowSizeMillis;        // Window duration
    private final long maxRequests;            // Max requests per window
    private final int bucketCount;             // Number of sub-windows
    private final long bucketSizeMillis;       // Size of each sub-window
    
    private final Map<String, CircularBuffer> clientWindows;
    
    public SlidingWindow(long windowSizeMillis, long maxRequests, int bucketCount) {
        this.windowSizeMillis = windowSizeMillis;
        this.maxRequests = maxRequests;
        this.bucketCount = bucketCount;
        this.bucketSizeMillis = windowSizeMillis / bucketCount;
        this.clientWindows = new ConcurrentHashMap<>();
    }
    
    public boolean tryConsume(String clientId) {
        CircularBuffer buffer = clientWindows.computeIfAbsent(
            clientId, k -> new CircularBuffer(bucketCount));
        
        long currentBucket = System.currentTimeMillis() / bucketSizeMillis;
        
        // Slide window and count requests
        long totalRequests = buffer.getTotalRequests(currentBucket);
        
        if (totalRequests >= maxRequests) {
            return false;
        }
        
        buffer.addRequest(currentBucket);
        return true;
    }
    
    private static class CircularBuffer {
        private final long[] buckets;
        private long totalRequests;
        
        public CircularBuffer(int size) {
            this.buckets = new long[size];
        }
        
        public void addRequest(long bucket) {
            int bucketIndex = (int) (bucket % buckets.length);
            buckets[bucketIndex]++;
            totalRequests++;
        }
        
        public long getTotalRequests(long currentBucket) {
            // Reset old buckets and count total
            long total = 0;
            for (int i = 0; i < buckets.length; i++) {
                long bucketTime = currentBucket - (buckets.length - 1 - i);
                if (bucketTime >= 0) {
                    total += buckets[i];
                } else {
                    buckets[i] = 0; // Reset old bucket
                }
            }
            totalRequests = total;
            return total;
        }
    }
}
```

#### Advantages
- **Smooth Limiting**: No boundary burst problems
- **Accurate Counting**: Precise request tracking
- **Flexible Windows**: Adjustable window sizes
- **Memory Efficient**: Bounded memory usage

#### Disadvantages
- **Complex Implementation**: More complex than fixed window
- **Memory Usage**: Requires more memory for sub-windows
- **CPU Overhead**: More computation for sliding calculations

#### Use Cases
- **Accurate Rate Limiting**: Precise control needed
- **Variable Traffic**: Handle varying traffic patterns
- **Quality Service**: Different limits for different user tiers

## Implementing Rate Limiting

### Distributed Rate Limiting

#### Redis-Based Implementation
```java
@Service
public class RedisRateLimiter {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final String RATE_LIMIT_KEY_PREFIX = "ratelimit:";
    
    public boolean isAllowed(String clientId, int maxRequests, long windowSeconds) {
        String key = RATE_LIMIT_KEY_PREFIX + clientId;
        
        long currentTime = System.currentTimeMillis() / 1000;
        long windowStart = currentTime - windowSeconds;
        
        // Use Redis sorted set to track requests
        redisTemplate.opsForZSet().removeRangeByScore(key, 0, windowStart);
        
        Long currentCount = redisTemplate.opsForZSet().size(key);
        
        if (currentCount != null && currentCount >= maxRequests) {
            return false;
        }
        
        // Add current request
        redisTemplate.opsForZSet().add(key, String.valueOf(currentTime), currentTime);
        
        // Set expiration on the key
        redisTemplate.expire(key, Duration.ofSeconds(windowSeconds));
        
        return true;
    }
}
```

#### Distributed Counter with Race Condition Handling
```java
@Service
public class DistributedRateLimiter {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public boolean isAllowed(String clientId, int maxRequests, Duration window) {
        String key = "ratelimit:" + clientId + ":" + getCurrentWindow(window);
        
        // Use Lua script for atomic increment and check
        String luaScript = """
            local current = redis.call('incr', KEYS[1])
            if current == 1 then
                redis.call('expire', KEYS[1], ARGV[1])
            end
            return current
            """;
        
        Long currentCount = redisTemplate.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            Collections.singletonList(key),
            window.getSeconds()
        );
        
        return currentCount != null && currentCount <= maxRequests;
    }
    
    private String getCurrentWindow(Duration window) {
        long windowSeconds = window.getSeconds();
        long currentWindow = System.currentTimeMillis() / (windowSeconds * 1000);
        return String.valueOf(currentWindow);
    }
}
```

### API Gateway Rate Limiting

#### Spring Cloud Gateway Implementation
```java
@Configuration
public class RateLimitConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("api_route", r -> r.path("/api/**")
                .filters(f -> f.requestRateLimiter(c -> c.setRateLimiter(redisRateLimiter())))
                .uri("lb://api-service"))
            .build();
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter("myRateLimiter", 
            new RedisRateLimiter.Config()
                .setBurstCapacity(100)
                .setReplenishRate(10)
                .setRequestedTokens(1));
    }
}
```

### Multi-Tier Rate Limiting

#### User-Based Limits
```java
public enum UserTier {
    FREE(10, Duration.ofMinutes(1)),      // 10 requests per minute
    BASIC(100, Duration.ofMinutes(1)),    // 100 requests per minute  
    PREMIUM(1000, Duration.ofMinutes(1)), // 1000 requests per minute
    ENTERPRISE(-1, null);                 // Unlimited
    
    private final int maxRequests;
    private final Duration window;
    
    UserTier(int maxRequests, Duration window) {
        this.maxRequests = maxRequests;
        this.window = window;
    }
}
```

#### Endpoint-Based Limits
```java
public class EndpointRateLimit {
    private final String pattern;
    private final int maxRequests;
    private final Duration window;
    
    // Constructor and getters
    
    public static EndpointRateLimit create(String pattern, int maxRequests, Duration window) {
        return new EndpointRateLimit(pattern, maxRequests, window);
    }
    
    // Example patterns:
    // "/api/users/*" - 100 requests per minute
    // "/api/orders" - 50 requests per minute  
    // "/api/admin/*" - 10 requests per minute
}
```

### Response Headers

#### Standard Rate Limit Headers
```java
@RestControllerAdvice
public class RateLimitAdvice {
    
    @Autowired
    private RateLimitService rateLimitService;
    
    @ModelAttribute
    public void addRateLimitHeaders(HttpServletResponse response, 
                                  @RequestAttribute("clientId") String clientId) {
        
        RateLimitStatus status = rateLimitService.getStatus(clientId);
        
        response.setHeader("X-RateLimit-Limit", String.valueOf(status.getLimit()));
        response.setHeader("X-RateLimit-Remaining", String.valueOf(status.getRemaining()));
        response.setHeader("X-RateLimit-Reset", String.valueOf(status.getResetTime()));
        
        if (status.isExceeded()) {
            response.setHeader("Retry-After", String.valueOf(status.getRetryAfterSeconds()));
        }
    }
}
```

## Rate Limiting Best Practices

### 1. Multiple Limit Levels
- **Global Limits**: Protect entire system
- **Service Limits**: Per-service rate limits
- **User Limits**: Per-user quotas and burst allowances
- **Endpoint Limits**: Different limits for different endpoints

### 2. Graceful Degradation
- **Informative Responses**: Clear error messages and retry guidance
- **Partial Service**: Degraded functionality during overload
- **Queue Requests**: Queue requests for later processing
- **Fallback Responses**: Cached or default responses

### 3. Monitoring and Alerting
```java
@Service
public class RateLimitMetrics {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter rateLimitExceeded = Counter.build()
        .name("rate_limit_exceeded_total")
        .help("Total rate limit violations")
        .register(meterRegistry);
    
    private final Counter rateLimitAllowed = Counter.build()
        .name("rate_limit_allowed_total")
        .help("Total allowed requests")
        .register(meterRegistry);
    
    private final Histogram requestRate = Histogram.build()
        .name("request_rate_per_second")
        .help("Request rate distribution")
        .register(meterRegistry);
    
    public void recordExceeded(String clientId, String endpoint) {
        rateLimitExceeded.increment();
        // Log client and endpoint for analysis
    }
    
    public void recordAllowed(String clientId, String endpoint) {
        rateLimitAllowed.increment();
    }
    
    public void recordRequestRate(double rate) {
        requestRate.observe(rate);
    }
}
```

### 4. Configuration Management
- **Dynamic Configuration**: Change limits without restart
- **A/B Testing**: Test different limits on user segments
- **Gradual Rollout**: Slowly increase limits to find optimal values
- **Automated Tuning**: Use metrics to automatically adjust limits

### 5. Security Considerations
- **IP-Based Limiting**: Prevent abuse from single IP addresses
- **User Authentication**: Tie limits to authenticated users
- **API Keys**: Different limits for different API keys
- **Bot Detection**: Identify and limit automated traffic

## Common Rate Limiting Challenges

### Distributed Systems
**Challenge:** Coordinating rate limits across multiple instances
**Solutions:**
- **Shared Storage**: Redis for centralized rate limit state
- **Consistent Hashing**: Route same client to same instance
- **Eventual Consistency**: Accept temporary inconsistency

### Microservices Architecture
**Challenge:** Rate limiting across service mesh
**Solutions:**
- **Service Mesh**: Istio, Linkerd for distributed rate limiting
- **API Gateway**: Centralized rate limiting at ingress
- **Sidecar Pattern**: Rate limiting in service sidecars

### High-Throughput Systems
**Challenge:** Rate limiting impact on performance
**Solutions:**
- **Async Processing**: Non-blocking rate limit checks
- **In-Memory Caching**: Fast local rate limit approximations
- **Sampling**: Check rate limits on percentage of requests

### Dynamic Traffic Patterns
**Challenge:** Adapting to changing traffic patterns
**Solutions:**
- **Machine Learning**: Predictive rate limit adjustments
- **Feedback Loops**: Adjust limits based on system performance
- **Tiered Limits**: Different limits for different traffic types

## Real-World Rate Limiting Examples

### Twitter API Rate Limits
```javascript
// Different limits for different endpoints
const rateLimits = {
  '/statuses/update': { limit: 300, window: '3 hours' },      // Post tweets
  '/statuses/home_timeline': { limit: 15, window: '15 minutes' }, // Read timeline
  '/search/tweets': { limit: 180, window: '15 minutes' },     // Search tweets
  '/users/show': { limit: 900, window: '15 minutes' }         // User lookup
};
```

### Stripe API Rate Limits
```javascript
// Sliding window with burst allowance
const stripeLimits = {
  create_charge: { limit: 100, burst: 25, window: '10 seconds' },
  create_customer: { limit: 25, burst: 10, window: '10 seconds' },
  create_refund: { limit: 20, burst: 5, window: '1 minute' }
};
```

### GitHub API Rate Limits
```javascript
// Different limits for authenticated vs unauthenticated
const githubLimits = {
  authenticated: { limit: 5000, window: '1 hour' },
  unauthenticated: { limit: 60, window: '1 hour' },
  search: { limit: 30, window: '1 minute' }  // Stricter for search
};
```

## Conclusion

Rate limiting is essential for building scalable, resilient systems that can handle varying traffic patterns while protecting against abuse and ensuring fair resource allocation.

**Key Takeaways:**
- **Algorithm Choice**: Select algorithm based on traffic patterns and requirements
- **Distributed Implementation**: Use Redis for cluster-wide rate limiting
- **Multi-Level Limits**: Apply different limits at different levels
- **Monitoring**: Track rate limit effectiveness and violations
- **Graceful Handling**: Provide clear feedback when limits are exceeded

Effective rate limiting requires understanding your traffic patterns, selecting appropriate algorithms, and implementing proper monitoring and feedback mechanisms. Start with simple limits and gradually add sophistication as your needs evolve.
