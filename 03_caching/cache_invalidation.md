# Cache Invalidation Strategies

Cache invalidation is the process of removing or updating stale data from cache to ensure data consistency between cache and the underlying data source. Poor invalidation strategies can lead to data inconsistencies, while overly aggressive invalidation can defeat the purpose of caching.

## Why Cache Invalidation Matters

### Data Consistency
- **Stale Data Prevention**: Ensure users see current information
- **Business Logic Integrity**: Maintain correct application behavior
- **User Trust**: Prevent confusion from outdated data

### Performance Balance
- **Cache Effectiveness**: Balance freshness with performance
- **Resource Utilization**: Avoid unnecessary cache operations
- **System Load**: Prevent cache thrashing

## Cache Invalidation Strategies

### 1. Time-Based Expiration (TTL)

#### How It Works
Data expires automatically after a specified time period, regardless of whether it's still valid.

#### Implementation
```java
// Set TTL when storing data
cacheService.set("user:123", userData, Duration.ofHours(1));

// Data automatically expires after 1 hour
```

#### Advantages
- **Simple**: Easy to implement and understand
- **Predictable**: Known expiration times
- **Automatic**: No manual intervention required
- **Memory Bounded**: Prevents unlimited cache growth

#### Disadvantages
- **Stale Data**: Data may be outdated before expiration
- **Fixed Lifetime**: Doesn't account for data usage patterns
- **Wasted Space**: Unused data occupies cache until expiration

#### Use Cases
- **Relatively Static Data**: User profiles, product catalogs
- **Time-Sensitive Data**: News articles, weather data
- **Session Data**: Login sessions, temporary tokens

### 2. Manual Invalidation

#### How It Works
Application explicitly removes cache entries when data changes.

#### Direct Invalidation
```java
@Service
public class UserService {
    
    public User updateUser(User user) {
        // Update database
        User savedUser = userRepository.save(user);
        
        // Invalidate cache
        cacheService.delete("user:" + user.getId());
        
        return savedUser;
    }
    
    public void deleteUser(Long userId) {
        // Delete from database
        userRepository.deleteById(userId);
        
        // Invalidate cache
        cacheService.delete("user:" + userId);
    }
}
```

#### Pattern-Based Invalidation
```java
public void invalidateUserRelatedCache(Long userId) {
    // Direct key invalidation
    cacheService.delete("user:" + userId);
    
    // Pattern-based invalidation (if supported)
    cacheService.deleteByPattern("user:" + userId + ":*");
    
    // Related data invalidation
    cacheService.delete("user_posts:" + userId);
    cacheService.delete("user_friends:" + userId);
}
```

#### Advantages
- **Precise Control**: Only invalidate when necessary
- **Data Freshness**: Immediate cache updates
- **Resource Efficient**: No unnecessary operations

#### Disadvantages
- **Complexity**: Application must track all cache keys
- **Maintenance Burden**: Cache logic scattered across code
- **Race Conditions**: Updates may not invalidate immediately

#### Use Cases
- **Frequently Updated Data**: User profiles, inventory levels
- **Transactional Updates**: Financial data, order status
- **Critical Business Data**: Product prices, availability

### 3. Event-Based Invalidation

#### How It Works
Cache invalidation triggered by data change events from the database or application.

#### Database Triggers
```sql
-- PostgreSQL trigger for cache invalidation
CREATE OR REPLACE FUNCTION invalidate_user_cache()
RETURNS TRIGGER AS $$
BEGIN
    -- Notify application of data change
    PERFORM pg_notify('user_cache_invalidation', 
                     CASE WHEN TG_OP = 'DELETE' THEN OLD.id::text ELSE NEW.id::text END);
    RETURN CASE WHEN TG_OP = 'DELETE' THEN OLD ELSE NEW END;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER user_cache_invalidation_trigger
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION invalidate_user_cache();
```

#### Message Queue Integration
```java
@Service
public class CacheInvalidationListener {
    
    @RabbitListener(queues = "user-updates")
    public void handleUserUpdate(String message) {
        try {
            UserUpdateEvent event = objectMapper.readValue(message, UserUpdateEvent.class);
            
            // Invalidate cache for updated user
            cacheService.delete("user:" + event.getUserId());
            
            // Invalidate related caches
            invalidateUserRelatedCaches(event.getUserId());
            
        } catch (Exception e) {
            logger.error("Failed to process cache invalidation event", e);
        }
    }
}
```

#### Advantages
- **Automatic**: No manual cache management
- **Real-time**: Immediate invalidation on data changes
- **Centralized**: Single place for invalidation logic

#### Disadvantages
- **Infrastructure Complexity**: Requires messaging system
- **Performance Overhead**: Event processing adds latency
- **Coupling**: Tightly couples cache to data changes

#### Use Cases
- **Multi-Service Architectures**: Services need to invalidate each other's caches
- **Real-Time Requirements**: Immediate cache consistency needed
- **Complex Invalidation Logic**: Many related cache entries to invalidate

### 4. Version-Based Invalidation

#### How It Works
Use version numbers to track data freshness and invalidate based on version mismatches.

#### Implementation
```java
@Entity
public class User {
    @Id
    private Long id;
    
    @Version
    private Long version; // Automatically incremented
    
    private String name;
    private String email;
}

@Service
public class VersionedCacheService {
    
    public User getUserWithVersionCheck(Long userId) {
        String cacheKey = "user:" + userId;
        String versionKey = "user_version:" + userId;
        
        // Get cached data and version
        User cachedUser = (User) cacheService.get(cacheKey);
        Long cachedVersion = (Long) cacheService.get(versionKey);
        
        // Check if we have current version
        Long currentVersion = getCurrentVersionFromDatabase(userId);
        
        if (cachedUser != null && Objects.equals(cachedVersion, currentVersion)) {
            return cachedUser; // Cache hit
        }
        
        // Cache miss or stale - fetch from database
        User user = userRepository.findById(userId).orElse(null);
        if (user != null) {
            // Update cache with new version
            cacheService.set(cacheKey, user, Duration.ofHours(1));
            cacheService.set(versionKey, user.getVersion(), Duration.ofHours(24));
        }
        
        return user;
    }
    
    @Transactional
    public User updateUser(User user) {
        // Update increments version automatically
        User updated = userRepository.save(user);
        
        // Version key will be updated on next read
        return updated;
    }
}
```

#### Advantages
- **Precise Invalidation**: Only invalidate when data actually changes
- **Conflict Detection**: Handle concurrent updates
- **Optimistic Concurrency**: Built-in version checking

#### Disadvantages
- **Complexity**: Version management adds overhead
- **Storage Cost**: Additional version data storage
- **Application Logic**: Version checking in application code

#### Use Cases
- **Optimistic Locking**: Prevent lost updates
- **Versioned APIs**: API versioning based on data versions
- **Audit Requirements**: Track data change history

### 5. Probabilistic Invalidation

#### How It Works
Use probability to decide when to invalidate cache entries, reducing thundering herd problems.

#### Early Expiration Technique
```java
public boolean shouldExpireEarly(String key, long ttlSeconds) {
    // Expire 10% of items 10% early to spread invalidation load
    double earlyExpireProbability = 0.1;
    long age = getCacheAge(key);
    long earlyExpireThreshold = (long) (ttlSeconds * (1 - earlyExpireProbability));
    
    return age > earlyExpireThreshold && Math.random() < earlyExpireProbability;
}
```

#### Advantages
- **Load Distribution**: Spread invalidation over time
- **Thundering Herd Prevention**: Avoid simultaneous cache misses
- **Performance Stability**: Reduce sudden load spikes

#### Disadvantages
- **Approximate**: Not perfectly accurate invalidation
- **Tuning Required**: Probability values need adjustment
- **Unpredictable**: Hard to predict exact invalidation timing

#### Use Cases
- **High-Traffic Systems**: Prevent cache stampedes
- **Variable Load**: Handle unpredictable traffic patterns
- **Large Cache Clusters**: Distribute invalidation load

## Hybrid Invalidation Strategies

### TTL + Manual Invalidation
```java
@Service
public class HybridCacheService {
    
    public User getUser(Long userId) {
        String cacheKey = "user:" + userId;
        User cachedUser = (User) cacheService.get(cacheKey);
        
        if (cachedUser != null) {
            return cachedUser;
        }
        
        // Cache miss - fetch and cache with TTL
        User user = userRepository.findById(userId).orElse(null);
        if (user != null) {
            cacheService.set(cacheKey, user, Duration.ofHours(1));
        }
        
        return user;
    }
    
    @Transactional
    public User updateUserCritical(User user) {
        // For critical updates, invalidate immediately
        User updated = userRepository.save(user);
        cacheService.delete("user:" + user.getId());
        
        return updated;
    }
    
    public void updateUserNonCritical(User user) {
        // For non-critical updates, rely on TTL
        userRepository.save(user);
        // Cache will expire naturally via TTL
    }
}
```

### Multi-Level Invalidation
```java
@Service
public class MultiLevelCacheService {
    
    public User getUser(Long userId) {
        // Check L1 cache (Redis)
        User user = (User) redisCache.get("user:" + userId);
        if (user != null) {
            return user;
        }
        
        // Check L2 cache (local cache)
        user = (User) localCache.get("user:" + userId);
        if (user != null) {
            // Promote to L1 cache
            redisCache.set("user:" + userId, user, Duration.ofHours(1));
            return user;
        }
        
        // Cache miss - fetch from database
        user = userRepository.findById(userId).orElse(null);
        if (user != null) {
            // Populate both caches
            localCache.set("user:" + userId, user, Duration.ofMinutes(30));
            redisCache.set("user:" + userId, user, Duration.ofHours(1));
        }
        
        return user;
    }
    
    public void invalidateUser(Long userId) {
        // Invalidate both cache levels
        redisCache.delete("user:" + userId);
        localCache.delete("user:" + userId);
    }
}
```

## Cache Invalidation Best Practices

### 1. Cache Key Design
- **Consistent Naming**: Use predictable key patterns
- **Namespace Isolation**: Separate different data types
- **Hash Long Keys**: Avoid memory issues with long keys
- **Version in Keys**: Include version numbers for breaking changes

### 2. Invalidation Granularity
- **Precise Invalidation**: Invalidate only affected data
- **Avoid Over-Invalidation**: Don't clear entire cache unnecessarily
- **Batch Operations**: Group related invalidations
- **Async Processing**: Don't block on invalidation

### 3. Monitoring and Alerting
```java
@Service
public class CacheInvalidationMetrics {
    
    private final Counter manualInvalidations = Counter.build()
        .name("cache_manual_invalidations_total")
        .help("Total manual cache invalidations")
        .register();
    
    private final Counter ttlExpirations = Counter.build()
        .name("cache_ttl_expirations_total")
        .help("Total TTL-based expirations")
        .register();
    
    private final Histogram invalidationLatency = Histogram.build()
        .name("cache_invalidation_duration_seconds")
        .help("Time taken for cache invalidation")
        .register();
    
    public void recordManualInvalidation() {
        manualInvalidations.inc();
    }
    
    public void recordTtlExpiration() {
        ttlExpirations.inc();
    }
    
    public Timer.Sample startInvalidationTimer() {
        return Timer.start(meterRegistry);
    }
}
```

### 4. Error Handling
- **Idempotent Operations**: Safe to retry invalidations
- **Graceful Degradation**: Continue if invalidation fails
- **Logging**: Track invalidation successes and failures
- **Circuit Breakers**: Handle persistent cache failures

### 5. Performance Considerations
- **Batch Invalidation**: Group multiple invalidations
- **Async Processing**: Don't block application on cache operations
- **Connection Pooling**: Reuse cache connections
- **Load Distribution**: Distribute invalidation load across cluster

## Common Invalidation Pitfalls

### 1. Race Conditions
**Problem:** Cache updated after database but before invalidation
**Solution:** Use transactions or optimistic locking

### 2. Invalidation Storms
**Problem:** Frequent updates cause excessive invalidation
**Solution:** Use TTL-based expiration for volatile data

### 3. Cache Stampede
**Problem:** Mass cache misses after invalidation
**Solution:** Staggered invalidation or probabilistic early expiration

### 4. Inconsistent State
**Problem:** Partial invalidation leaves inconsistent data
**Solution:** Atomic invalidation operations or eventual consistency

### 5. Over-Invalidation
**Problem:** Clearing more cache than necessary
**Solution:** Precise invalidation with proper key design

## Real-World Invalidation Examples

### E-commerce Product Updates
```java
@Service
public class ProductCacheService {
    
    @Transactional
    public Product updateProduct(Product product) {
        Product saved = productRepository.save(product);
        
        // Invalidate product cache
        cacheService.delete("product:" + product.getId());
        
        // Invalidate category cache (products list)
        cacheService.delete("category:" + product.getCategoryId() + ":products");
        
        // Invalidate search cache
        cacheService.deleteByPattern("search:*" + product.getName() + "*");
        
        return saved;
    }
}
```

### Social Media Timeline Updates
```java
@Service
public class TimelineCacheService {
    
    public void postCreated(Post post) {
        // Invalidate user's timeline cache
        cacheService.delete("timeline:" + post.getUserId());
        
        // Invalidate followers' timeline caches
        List<Long> followerIds = followerRepository.findFollowerIds(post.getUserId());
        for (Long followerId : followerIds) {
            // Use TTL-based expiration for follower timelines
            // They will refresh on next access
            cacheService.expire("timeline:" + followerId, Duration.ofMinutes(5));
        }
    }
}
```

### Multi-Region Cache Invalidation
```java
@Service
public class GlobalCacheInvalidationService {
    
    @Autowired
    private List<RegionalCacheService> regionalCaches;
    
    public void invalidateGlobalData(String cacheKey) {
        // Invalidate in all regions asynchronously
        for (RegionalCacheService regionalCache : regionalCaches) {
            CompletableFuture.runAsync(() -> {
                try {
                    regionalCache.delete(cacheKey);
                } catch (Exception e) {
                    logger.warn("Failed to invalidate cache in region", e);
                }
            });
        }
    }
}
```

## Conclusion

Cache invalidation is a critical aspect of caching strategy that directly impacts data consistency and system performance. The choice of invalidation strategy depends on your data freshness requirements, update frequency, and consistency needs.

**Key Takeaways:**
- **TTL Expiration**: Simple but may serve stale data
- **Manual Invalidation**: Precise control but complex to implement
- **Event-Based**: Automatic but requires infrastructure
- **Version-Based**: Conflict detection but adds complexity
- **Hybrid Approaches**: Combine strategies for optimal results

Effective cache invalidation requires understanding your data access patterns, update frequency, and consistency requirements. Monitor invalidation performance and adjust strategies based on observed behavior and business needs.
