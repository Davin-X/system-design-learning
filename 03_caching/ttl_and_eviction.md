# TTL and Cache Eviction Policies

Time-To-Live (TTL) and eviction policies determine how long data stays in cache and what happens when cache reaches capacity. Proper TTL and eviction strategies are crucial for maintaining cache performance and memory efficiency.

## Time-To-Live (TTL)

### What is TTL?
TTL specifies the maximum time a cached item can exist before it expires and becomes eligible for removal.

### TTL Implementation

#### Fixed TTL
```java
// Set fixed expiration time
cacheService.set("user:123", userData, Duration.ofHours(1));

// Data expires exactly after 1 hour
```

#### Sliding TTL
```java
// Reset TTL on each access
cacheService.set("session:abc", sessionData, Duration.ofMinutes(30));
// TTL resets to 30 minutes every time data is accessed
```

#### Absolute vs Sliding TTL

| Type | Behavior | Use Case |
|------|----------|----------|
| **Absolute TTL** | Expires at fixed time | Content with known freshness requirements |
| **Sliding TTL** | Extends on access | Session data, frequently accessed items |
| **Hybrid TTL** | Combination of both | Complex expiration requirements |

### TTL Best Practices

#### Business-Aligned TTL Values
```java
// User profiles - relatively stable
Duration USER_PROFILE_TTL = Duration.ofHours(6);

// Product prices - frequently updated
Duration PRODUCT_PRICE_TTL = Duration.ofMinutes(5);

// Weather data - time-sensitive
Duration WEATHER_DATA_TTL = Duration.ofMinutes(10);

// Session data - user activity based
Duration SESSION_TTL = Duration.ofHours(24);
```

#### TTL Considerations
- **Data Freshness Requirements**: How stale can data be?
- **Update Frequency**: How often does data change?
- **Access Patterns**: How frequently is data accessed?
- **Business Impact**: Cost of serving stale data

## Cache Eviction Policies

### Least Recently Used (LRU)

#### How It Works
Evicts the least recently accessed items when cache reaches capacity.

**Algorithm:**
1. Track access time for each item
2. When eviction needed, remove item with oldest access time
3. New items and accessed items move to "recent" end

#### Implementation
```java
public class LRUCache<K, V> {
    private final Map<K, V> cache = new LinkedHashMap<>(16, 0.75f, true);
    private final int capacity;
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
    }
    
    public V get(K key) {
        return cache.get(key); // LinkedHashMap moves to end on access
    }
    
    public void put(K key, V value) {
        cache.put(key, value);
        if (cache.size() > capacity) {
            // Remove least recently used (first entry)
            Iterator<K> iterator = cache.keySet().iterator();
            iterator.next();
            iterator.remove();
        }
    }
}
```

#### Advantages
- **Intuitive**: Recently used items likely to be used again
- **Adaptive**: Automatically adapts to access patterns
- **Simple**: Easy to understand and implement

#### Disadvantages
- **Cold Start**: New items may be evicted quickly
- **Sequential Access**: Poor for sequential data access patterns
- **Memory Overhead**: Tracking access order requires extra memory

#### Use Cases
- **General Purpose**: Most web applications
- **User Sessions**: Recently active sessions
- **Popular Content**: Frequently accessed data

### Least Frequently Used (LFU)

#### How It Works
Evicts the least frequently accessed items based on access count.

**Algorithm:**
1. Maintain access count for each item
2. When eviction needed, remove item with lowest access count
3. Break ties by recency (least recently used among LFU items)

#### Implementation
```java
public class LFUCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final Map<K, Integer> frequencies = new HashMap<>();
    private final Map<Integer, LinkedHashSet<K>> frequencyLists = new HashMap<>();
    private final int capacity;
    private int minFrequency = 0;
    
    public V get(K key) {
        if (!cache.containsKey(key)) return null;
        
        int freq = frequencies.get(key);
        frequencies.put(key, freq + 1);
        
        // Move to higher frequency list
        updateFrequencyList(key, freq);
        
        return cache.get(key);
    }
    
    public void put(K key, V value) {
        if (cache.size() >= capacity && !cache.containsKey(key)) {
            // Evict least frequently used
            evictLFU();
        }
        
        cache.put(key, value);
        frequencies.put(key, 1);
        frequencyLists.computeIfAbsent(1, k -> new LinkedHashSet<>()).add(key);
        minFrequency = 1;
    }
    
    private void evictLFU() {
        LinkedHashSet<K> minFreqList = frequencyLists.get(minFrequency);
        K keyToRemove = minFreqList.iterator().next();
        
        cache.remove(keyToRemove);
        frequencies.remove(keyToRemove);
        minFreqList.remove(keyToRemove);
        
        if (minFreqList.isEmpty()) {
            frequencyLists.remove(minFrequency);
        }
    }
}
```

#### Advantages
- **Fairness**: Rewards frequently accessed items
- **Stable**: Less thrashing than LRU for stable workloads
- **Predictive**: Good for workloads with known access patterns

#### Disadvantages
- **Complex**: More complex implementation
- **Memory Intensive**: Stores frequency counts
- **Cold Start**: New items disadvantaged initially

#### Use Cases
- **Stable Workloads**: Predictable access patterns
- **File Systems**: Frequently accessed files
- **Database Buffers**: Commonly queried data

### First-In-First-Out (FIFO)

#### How It Works
Evicts the oldest items first (simple queue).

**Algorithm:**
1. Maintain insertion order
2. When eviction needed, remove oldest item
3. New items added to end of queue

#### Advantages
- **Simple**: Very easy to implement
- **Predictable**: Deterministic eviction order
- **Low Overhead**: Minimal memory requirements

#### Disadvantages
- **Poor Performance**: Doesn't consider access patterns
- **Cache Pollution**: Important data may be evicted early
- **Not Adaptive**: Ignores usage statistics

#### Use Cases
- **Simple Caching**: Basic cache requirements
- **Log Buffering**: Temporary data storage
- **Queue-Like Access**: Processing pipelines

### Random Eviction

#### How It Works
Randomly selects items for eviction when cache is full.

#### Advantages
- **Simple**: Extremely easy to implement
- **Fair**: No bias toward any particular items
- **Low Overhead**: No tracking required

#### Disadvantages
- **Poor Performance**: May evict important data
- **Unpredictable**: No guarantee of good performance
- **Not Optimal**: Better algorithms exist

#### Use Cases
- **Approximate Caching**: When exactness isn't critical
- **Testing**: Simple cache implementations
- **Fallback**: When other algorithms fail

### Size-Based Eviction

#### How It Works
Evicts items based on memory usage when cache reaches size limits.

**Strategies:**
- **Memory Threshold**: Evict when total memory exceeds limit
- **Item Size Aware**: Consider individual item sizes
- **Soft/Hard Limits**: Different thresholds for different actions

#### Advantages
- **Memory Control**: Prevents unlimited memory growth
- **Fairness**: Accounts for variable item sizes
- **Resource Protection**: Prevents memory exhaustion

#### Disadvantages
- **Complex**: Requires size tracking
- **Overhead**: Additional memory for size metadata
- **Approximate**: Hard to predict exact memory usage

#### Use Cases
- **Large Objects**: Variable-sized cached items
- **Memory-Constrained**: Limited memory environments
- **Resource Management**: Strict memory limits

## Advanced Eviction Policies

### Adaptive Replacement Cache (ARC)

#### How It Works
Combines LRU and LFU characteristics adaptively based on workload.

**Algorithm:**
- Maintains two LRU lists: recently accessed and frequently accessed
- Dynamically adjusts list sizes based on hit rates
- Adapts to changing access patterns

#### Advantages
- **Adaptive**: Adjusts to workload changes
- **Better Performance**: Often outperforms pure LRU/LFU
- **Self-Tuning**: No manual parameter tuning

#### Disadvantages
- **Complex**: More complex implementation
- **Memory Overhead**: Maintains multiple data structures

### Clock Algorithm (Second Chance)

#### How It Works
Approximates LRU with a circular buffer and reference bits.

**Algorithm:**
1. Items arranged in circular list
2. Each item has a reference bit
3. On access, set reference bit to 1
4. On eviction needed, scan for bit = 0
5. If bit = 1, set to 0 and continue

#### Advantages
- **Simple**: Easier than full LRU
- **Low Overhead**: Minimal memory requirements
- **Good Performance**: Reasonable approximation of LRU

#### Disadvantages
- **Approximation**: Not exact LRU behavior
- **Tuning Required**: Clock size affects performance

### W-TinyLFU

#### How It Works
Combines windowed LFU with admission policy to prevent cache pollution.

**Components:**
- **Windowed LFU**: Recent access frequency tracking
- **Admission Filter**: Prevents infrequently accessed items
- **Segmented Cache**: Different policies for different segments

#### Advantages
- **Anti-Pollution**: Prevents cache pollution by scans
- **High Hit Rate**: Often achieves higher hit rates
- **Adaptive**: Adjusts to access pattern changes

#### Disadvantages
- **Complex**: Advanced algorithm
- **Memory Intensive**: Multiple data structures

## TTL and Eviction in Redis

### Redis TTL Commands
```bash
# Set key with expiration
SET user:123 "user_data" EX 3600

# Set expiration on existing key
EXPIRE user:123 3600

# Set expiration at specific timestamp
EXPIREAT user:123 1640995200

# Get remaining TTL
TTL user:123

# Remove expiration
PERSIST user:123
```

### Redis Eviction Policies

#### maxmemory-policy Configuration
```redis.conf
# Eviction policies when maxmemory is reached
maxmemory-policy noeviction    # Don't evict, return errors
maxmemory-policy allkeys-lru  # Evict any key using LRU
maxmemory-policy allkeys-lfu  # Evict any key using LFU (Redis 4.0+)
maxmemory-policy volatile-lru # Evict keys with TTL using LRU
maxmemory-policy volatile-lfu # Evict keys with TTL using LFU
maxmemory-policy allkeys-random # Evict random keys
maxmemory-policy volatile-random # Evict random keys with TTL
maxmemory-policy volatile-ttl # Evict keys with shortest TTL
```

#### Choosing Redis Eviction Policy

| Policy | Use Case | Behavior |
|--------|----------|----------|
| **allkeys-lru** | General purpose | Evicts least recently used keys |
| **allkeys-lfu** | Stable workloads | Evicts least frequently used keys |
| **volatile-lru** | Caching with TTL | Evicts LRU keys that have TTL |
| **volatile-ttl** | Short-lived data | Evicts keys closest to expiration |
| **noeviction** | Critical data | Returns errors instead of evicting |

### Redis TTL Best Practices
```java
@Service
public class RedisCacheService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void setWithTTL(String key, Object value, Duration ttl) {
        redisTemplate.opsForValue().set(key, value, ttl);
    }
    
    public void setMultipleWithTTL(Map<String, Object> keyValues, Duration ttl) {
        // Use pipeline for atomic multi-set with TTL
        redisTemplate.executePipelined((RedisConnection connection) -> {
            for (Map.Entry<String, Object> entry : keyValues.entrySet()) {
                connection.set(
                    entry.getKey().getBytes(),
                    serialize(entry.getValue())
                );
                connection.expire(entry.getKey().getBytes(), ttl.getSeconds());
            }
            return null;
        });
    }
    
    public Long getTTL(String key) {
        return redisTemplate.getExpire(key);
    }
    
    public Boolean refreshTTL(String key, Duration newTTL) {
        return redisTemplate.expire(key, newTTL);
    }
}
```

## Multi-Level Caching with TTL

### CDN + Application Cache Strategy
```java
@Service
public class MultiLevelCacheService {
    
    @Autowired
    private RedisService redisCache;
    
    @Autowired
    private LocalCacheService localCache;
    
    @Autowired
    private DatabaseService databaseService;
    
    public User getUser(Long userId) {
        String key = "user:" + userId;
        
        // Check local cache (fastest)
        User user = localCache.get(key);
        if (user != null) {
            return user;
        }
        
        // Check Redis (distributed)
        user = redisCache.get(key);
        if (user != null) {
            // Promote to local cache
            localCache.set(key, user, Duration.ofMinutes(30));
            return user;
        }
        
        // Fetch from database
        user = databaseService.getUser(userId);
        if (user != null) {
            // Set in both caches with different TTLs
            localCache.set(key, user, Duration.ofMinutes(30));
            redisCache.set(key, user, Duration.ofHours(6));
        }
        
        return user;
    }
    
    public void invalidateUser(Long userId) {
        String key = "user:" + userId;
        localCache.delete(key);
        redisCache.delete(key);
    }
}
```

### TTL Hierarchy
- **Browser Cache**: 5-15 minutes (HTTP headers)
- **CDN Cache**: 1-24 hours (CDN configuration)
- **Application Cache**: 5-60 minutes (Redis/local)
- **Database Cache**: 1-24 hours (query cache)

## Monitoring TTL and Eviction

### Cache Metrics to Monitor
```java
@Service
public class CacheMetricsService {
    
    private final Counter evictionCount = Counter.build()
        .name("cache_evictions_total")
        .help("Total cache evictions")
        .register();
    
    private final Counter ttlExpirationCount = Counter.build()
        .name("cache_ttl_expirations_total")
        .help("Total TTL expirations")
        .register();
    
    private final Gauge cacheSize = Gauge.build()
        .name("cache_size_current")
        .help("Current cache size")
        .register();
    
    private final Histogram ttlDistribution = Histogram.build()
        .name("cache_ttl_seconds")
        .help("Distribution of TTL values")
        .buckets(60, 300, 1800, 3600, 86400) // 1min, 5min, 30min, 1hr, 1day
        .register();
    
    public void recordEviction() {
        evictionCount.inc();
    }
    
    public void recordTTLExpiration() {
        ttlExpirationCount.inc();
    }
    
    public void recordTTLSet(long ttlSeconds) {
        ttlDistribution.observe(ttlSeconds);
    }
}
```

### Cache Health Checks
```java
@Service
public class CacheHealthService {
    
    @Autowired
    private CacheService cacheService;
    
    public CacheHealth checkHealth() {
        CacheHealth health = new CacheHealth();
        
        try {
            // Test basic operations
            String testKey = "health_check_" + System.currentTimeMillis();
            cacheService.set(testKey, "test_value", Duration.ofSeconds(10));
            String retrieved = (String) cacheService.get(testKey);
            
            health.setHealthy("test_value".equals(retrieved));
            health.setResponseTime(System.currentTimeMillis() - startTime);
            
            // Clean up
            cacheService.delete(testKey);
            
        } catch (Exception e) {
            health.setHealthy(false);
            health.setErrorMessage(e.getMessage());
        }
        
        return health;
    }
}
```

## Common TTL and Eviction Issues

### Cache Stampede (Thundering Herd)
**Problem:** Mass cache misses when popular items expire simultaneously
**Solutions:**
- **Staggered TTL**: Add random jitter to TTL values
- **Probabilistic Early Expiration**: Expire items slightly early
- **Locking**: Use locks to prevent duplicate fetches

### Cache Pollution
**Problem:** Frequently accessed but unimportant data fills cache
**Solutions:**
- **Admission Policies**: Only cache "important" data
- **Size Limits**: Limit cacheable item sizes
- **Selective Caching**: Cache based on business value

### Cold Cache Performance
**Problem:** Poor performance immediately after cache restart
**Solutions:**
- **Cache Warming**: Pre-populate important data
- **Gradual Loading**: Load data as requests come in
- **Backup Cache**: Use secondary cache during warmup

### Memory Pressure
**Problem:** System memory usage spikes from caching
**Solutions:**
- **Memory Limits**: Set maximum cache memory usage
- **Eviction Tuning**: Adjust eviction policies
- **Memory Monitoring**: Alert on memory pressure

## Real-World TTL and Eviction Examples

### E-commerce Product Cache
```java
@Configuration
public class ProductCacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheManager.RedisCacheManagerBuilder builder = 
            RedisCacheManager.builder(connectionFactory);
        
        // Different TTLs for different product types
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        
        cacheConfigurations.put("product_details", 
            RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(6)));
        
        cacheConfigurations.put("product_inventory", 
            RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(5)));
        
        cacheConfigurations.put("product_reviews", 
            RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(24)));
        
        return builder.cacheDefaults(defaultConfig)
            .withCacheConfiguration("product_details", cacheConfigurations.get("product_details"))
            .build();
    }
}
```

### Social Media Timeline Cache
```java
@Service
public class TimelineCacheService {
    
    // Timeline cache with short TTL for freshness
    @Cacheable(value = "timelines", key = "#userId", unless = "#result.isEmpty()")
    public List<Post> getTimeline(Long userId, int page, int size) {
        return timelineRepository.getTimeline(userId, page, size);
    }
    
    // Invalidate timeline on new post
    @CacheEvict(value = "timelines", key = "#post.userId")
    public Post createPost(Post post) {
        return postRepository.save(post);
    }
    
    // Also invalidate follower timelines with short TTL
    @CacheEvict(value = "timelines", allEntries = true, 
                condition = "#post.visibility == 'public'")
    public void invalidatePublicTimelines(Post post) {
        // This will clear all timeline caches
        // Followers will rebuild on next access
    }
}
```

## Conclusion

TTL and cache eviction policies are fundamental to cache performance and memory management. The right combination depends on your data access patterns, memory constraints, and consistency requirements.

**Key Takeaways:**
- **TTL Strategy**: Align expiration times with data freshness needs
- **Eviction Policies**: Choose based on access patterns and memory constraints
- **LRU**: Good default for most applications
- **LFU**: Better for stable, predictable workloads
- **Monitoring**: Track cache performance and adjust policies
- **Multi-Level**: Combine different TTLs and policies for optimal performance

Effective TTL and eviction management ensures your cache remains efficient, cost-effective, and consistent with your application requirements.
