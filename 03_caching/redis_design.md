# Redis Design and Architecture

Redis is an open-source, in-memory data structure store used as a database, cache, and message broker. Understanding Redis design principles is essential for building high-performance, scalable systems.

## Redis Core Architecture

### In-Memory Data Store
Redis stores all data in memory (RAM) for maximum performance.

**Key Characteristics:**
- **RAM Storage**: All data resides in memory
- **Persistence Options**: Optional disk persistence
- **Data Structures**: Rich set of built-in data types
- **Single-Threaded**: Single-threaded execution model

### Data Structures

#### Strings
Basic key-value pairs with string values.

```bash
SET user:123 "John Doe"
GET user:123
INCR counter:page_views
APPEND user:123 ": Senior Developer"
```

**Use Cases:**
- Caching user sessions
- Storing configuration data
- Counters and statistics
- Simple key-value storage

#### Hashes
Field-value pairs within a key.

```bash
HSET user:123 name "John Doe" email "john@example.com" age "30"
HGET user:123 name
HGETALL user:123
HINCRBY user:123 login_count 1
```

**Use Cases:**
- Storing user profiles
- Session data
- Configuration objects
- Counters with multiple fields

#### Lists
Ordered collections of strings.

```bash
LPUSH recent_posts "post:456" "post:789" "post:123"
LRANGE recent_posts 0 9
LPOP recent_posts
RPUSH notifications "New message from Alice"
```

**Use Cases:**
- Timeline feeds
- Message queues
- Recently viewed items
- Activity streams

#### Sets
Unordered collections of unique strings.

```bash
SADD user:123:followers "user:456" "user:789"
SCARD user:123:followers
SISMEMBER user:123:followers "user:456"
SINTER user:123:followers user:456:followers
```

**Use Cases:**
- Social network relationships
- Unique item collections
- Tagging systems
- Real-time analytics

#### Sorted Sets
Sets ordered by score.

```bash
ZADD leaderboard 1500 "player:123" 1200 "player:456" 1800 "player:789"
ZRANGE leaderboard 0 9 WITHSCORES
ZREVRANK leaderboard "player:123"
ZINCRBY leaderboard 100 "player:123"
```

**Use Cases:**
- Leaderboards and rankings
- Priority queues
- Rate limiting
- Time-series data

#### Bitmaps and HyperLogLog
Compact data structures for specific use cases.

```bash
SETBIT daily_active:20231201 12345 1
GETBIT daily_active:20231201 12345
BITCOUNT daily_active:20231201

PFADD unique_visitors "user:123" "user:456" "user:789"
PFCOUNT unique_visitors
```

**Use Cases:**
- User activity tracking
- Cardinality estimation
- Analytics and metrics

## Redis Persistence

### RDB (Redis Database)
Point-in-time snapshots of the dataset.

#### How It Works
- **Periodic Snapshots**: Save dataset to disk at specified intervals
- **Fork Process**: Child process creates snapshot without blocking main process
- **Compressed Format**: Space-efficient storage

#### Configuration
```redis.conf
# Save every 15 minutes if at least 1 key changed
save 900 1

# Save every 5 minutes if at least 10 keys changed  
save 300 10

# Save every 1 minute if at least 10000 keys changed
save 60 10000
```

#### Advantages
- **Compact**: Single file representation
- **Fast Restarts**: Quick loading on startup
- **Backup Friendly**: Easy to transfer and archive

#### Disadvantages
- **Data Loss**: May lose recent changes between snapshots
- **Fork Overhead**: Memory usage doubles during save
- **Not Real-Time**: Snapshots are periodic

### AOF (Append Only File)
Log of all write operations.

#### How It Works
- **Command Logging**: Every write command appended to log
- **Sequential Writes**: Fast, sequential disk writes
- **Replay on Startup**: Commands replayed to reconstruct dataset

#### Configuration
```redis.conf
appendonly yes
appendfsync everysec    # fsync every second
# appendfsync always    # fsync every write (slowest)
# appendfsync no        # let OS decide (fastest)
```

#### Advantages
- **Durability**: Every command logged
- **Granular**: Can replay to any point
- **Readable**: Human-readable command log

#### Disadvantages
- **File Size**: Can grow large over time
- **Rewrite Overhead**: Periodic compaction needed
- **Performance**: fsync can impact write performance

### Hybrid Approach
Combining RDB and AOF for optimal performance and durability.

```redis.conf
# Enable both persistence methods
save 900 1
appendonly yes
appendfsync everysec
```

## Redis Clustering

### Master-Slave Replication

#### Architecture
```
Master (Read/Write)
├── Slave 1 (Read-Only)
├── Slave 2 (Read-Only)
└── Slave 3 (Read-Only)
```

#### Setup
```bash
# On slave servers
redis-server --slaveof master-host 6379
```

#### Features
- **Read Scaling**: Distribute reads across slaves
- **High Availability**: Automatic failover (with Sentinel)
- **Data Replication**: Async replication from master to slaves

### Redis Sentinel

#### High Availability Solution
- **Monitoring**: Continuously monitor master and slaves
- **Notification**: Alert when issues detected
- **Automatic Failover**: Promote slave to master when master fails
- **Configuration**: Update clients with new master location

#### Configuration
```sentinel.conf
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1
```

### Redis Cluster

#### Distributed Architecture
- **Automatic Sharding**: Data automatically partitioned across nodes
- **High Availability**: Master-slave setup within cluster
- **Horizontal Scaling**: Add/remove nodes dynamically
- **Fault Tolerance**: Continues operation when nodes fail

#### Key Features
- **16384 Slots**: Data distributed across hash slots
- **Consistent Hashing**: Deterministic key-to-node mapping
- **Multi-Key Operations**: Limited support for multi-key operations
- **Resharding**: Online data redistribution

#### Setup Example
```bash
# Start cluster nodes
redis-server --cluster-enabled yes --cluster-config-file nodes.conf --port 7001
redis-server --cluster-enabled yes --cluster-config-file nodes.conf --port 7002
redis-server --cluster-enabled yes --cluster-config-file nodes.conf --port 7003

# Create cluster
redis-cli --cluster create 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003
```

## Redis Performance Optimization

### Memory Management

#### Memory Usage Monitoring
```bash
INFO memory
MEMORY USAGE user:123
MEMORY STATS
```

#### Memory Optimization Techniques
- **Data Structure Selection**: Choose efficient data types
- **Key Naming**: Use short, descriptive keys
- **Expiration**: Set TTL on temporary data
- **Compression**: Enable compression for large values

### Connection Management

#### Connection Pooling
```java
@Configuration
public class RedisConfig {
    
    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName("localhost");
        config.setPort(6379);
        
        LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
            .commandTimeout(Duration.ofMillis(100))
            .shutdownTimeout(Duration.ofMillis(100))
            .build();
            
        return new LettuceConnectionFactory(config, clientConfig);
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(LettuceConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        
        // Enable connection pooling
        template.setEnableTransactionSupport(true);
        
        return template;
    }
}
```

#### Pipelining
Batch multiple commands to reduce network round trips.

```java
@Service
public class PipelineService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public List<Object> executePipeline() {
        return redisTemplate.executePipelined((RedisConnection connection) -> {
            connection.set("key1".getBytes(), "value1".getBytes());
            connection.set("key2".getBytes(), "value2".getBytes());
            connection.get("key1".getBytes());
            connection.get("key2".getBytes());
            return null;
        });
    }
}
```

### Data Structure Optimization

#### Efficient Key Design
```java
// Good: Structured keys
user:123:profile
user:123:posts:recent
product:456:inventory

// Bad: Generic keys
user_123_profile
user123postsrecent
prod456inv
```

#### Hash Optimization
```java
// Use hashes for related data instead of multiple keys
HSET user:123 name "John" email "john@example.com" age "30"
// Instead of: SET user:123:name "John", SET user:123:email "john@example.com"
```

#### Sorted Set Optimization
```java
// Use sorted sets for ordered data with scores
ZADD leaderboard 1500 "player:123" 1200 "player:456"
// Efficient range queries and ranking operations
```

## Redis Use Cases and Patterns

### Caching Layer

#### Cache-Aside Pattern
```java
@Service
public class CacheAsideService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private UserRepository userRepository;
    
    public User getUser(Long userId) {
        String cacheKey = "user:" + userId;
        
        // Try cache first
        User user = (User) redisTemplate.opsForValue().get(cacheKey);
        if (user != null) {
            return user;
        }
        
        // Cache miss - fetch from database
        user = userRepository.findById(userId).orElse(null);
        if (user != null) {
            redisTemplate.opsForValue().set(cacheKey, user, Duration.ofHours(1));
        }
        
        return user;
    }
    
    @Transactional
    public User updateUser(User user) {
        User saved = userRepository.save(user);
        
        // Invalidate cache
        redisTemplate.delete("user:" + user.getId());
        
        return saved;
    }
}
```

### Session Store

#### Session Management
```java
@Service
public class SessionService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final Duration SESSION_TTL = Duration.ofHours(24);
    
    public void createSession(String sessionId, UserSession session) {
        String key = "session:" + sessionId;
        redisTemplate.opsForValue().set(key, session, SESSION_TTL);
    }
    
    public UserSession getSession(String sessionId) {
        String key = "session:" + sessionId;
        return (UserSession) redisTemplate.opsForValue().get(key);
    }
    
    public void extendSession(String sessionId) {
        String key = "session:" + sessionId;
        redisTemplate.expire(key, SESSION_TTL);
    }
    
    public void destroySession(String sessionId) {
        String key = "session:" + sessionId;
        redisTemplate.delete(key);
    }
}
```

### Rate Limiting

#### Token Bucket Implementation
```java
@Service
public class RateLimitService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final int MAX_REQUESTS = 100;
    private static final Duration WINDOW = Duration.ofMinutes(1);
    
    public boolean isAllowed(String userId, String action) {
        String key = "ratelimit:" + userId + ":" + action;
        
        long currentTime = System.currentTimeMillis();
        long windowStart = currentTime - WINDOW.toMillis();
        
        // Remove old requests outside the window
        redisTemplate.opsForZSet().removeRangeByScore(key, 0, windowStart);
        
        // Count requests in current window
        Long requestCount = redisTemplate.opsForZSet().zCard(key);
        
        if (requestCount != null && requestCount >= MAX_REQUESTS) {
            return false; // Rate limit exceeded
        }
        
        // Add current request
        redisTemplate.opsForZSet().add(key, String.valueOf(currentTime), currentTime);
        
        // Set expiration for the key
        redisTemplate.expire(key, WINDOW);
        
        return true;
    }
}
```

### Message Queue

#### Simple Queue Implementation
```java
@Service
public class QueueService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void enqueue(String queueName, Object message) {
        redisTemplate.opsForList().rightPush(queueName, message);
    }
    
    public Object dequeue(String queueName) {
        return redisTemplate.opsForList().leftPop(queueName);
    }
    
    public Object dequeueWithTimeout(String queueName, Duration timeout) {
        return redisTemplate.opsForList().leftPop(queueName, timeout);
    }
    
    public Long getQueueSize(String queueName) {
        return redisTemplate.opsForList().size(queueName);
    }
}
```

### Distributed Locks

#### Redlock Algorithm Implementation
```java
@Service
public class DistributedLockService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final String LOCK_PREFIX = "lock:";
    private static final Duration LOCK_TTL = Duration.ofSeconds(30);
    
    public boolean acquireLock(String resource, String ownerId) {
        String lockKey = LOCK_PREFIX + resource;
        String lockValue = ownerId + ":" + System.currentTimeMillis();
        
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, LOCK_TTL);
            
        return Boolean.TRUE.equals(acquired);
    }
    
    public boolean releaseLock(String resource, String ownerId) {
        String lockKey = LOCK_PREFIX + resource;
        
        // Use Lua script for atomic check-and-delete
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
            
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(lockKey),
            ownerId + ":" + System.currentTimeMillis()
        );
        
        return result != null && result > 0;
    }
}
```

## Redis Best Practices

### Key Design
- **Consistent Naming**: Use colon-separated namespaces
- **Size Limits**: Keep keys under 512MB
- **Expiration**: Set TTL on temporary data
- **Avoid Hot Keys**: Distribute load across multiple keys

### Memory Management
- **Monitor Usage**: Track memory usage with INFO command
- **Eviction Policies**: Configure appropriate maxmemory-policy
- **Data Structure Choice**: Use memory-efficient structures
- **Compression**: Enable compression for large values

### Performance Tuning
- **Connection Pooling**: Reuse connections to avoid overhead
- **Pipelining**: Batch multiple commands
- **Avoid Blocking Operations**: Don't use blocking commands in production
- **Monitor Slow Logs**: Track slow queries with SLOWLOG

### High Availability
- **Replication**: Set up master-slave replication
- **Sentinel**: Use Redis Sentinel for automatic failover
- **Cluster**: Consider Redis Cluster for large deployments
- **Backup**: Regular RDB/AOF backups

### Security
- **Network Security**: Bind to specific interfaces, use firewalls
- **Authentication**: Enable password authentication
- **TLS**: Use TLS for encrypted connections
- **Access Control**: Limit commands and keys accessible to clients

## Redis Limitations and Considerations

### Memory Constraints
- **RAM Dependency**: All data must fit in memory
- **Cost**: Memory is expensive compared to disk
- **Scaling**: Vertical scaling has limits

### Single-Threaded Nature
- **CPU Bound**: Single-threaded execution limits CPU utilization
- **Blocking Operations**: Certain operations can block the server
- **Concurrency**: Limited concurrent request handling

### Persistence Trade-offs
- **Performance vs Durability**: AOF everysec balances both
- **Fork Overhead**: RDB snapshots require memory duplication
- **Recovery Time**: Large datasets take time to load from disk

### Data Types Limitations
- **No Joins**: No relational capabilities
- **No Transactions**: Limited transaction support (single key)
- **No Secondary Indexes**: No automatic indexing of values

## Redis vs Other Solutions

### Redis vs Memcached
| Aspect | Redis | Memcached |
|--------|-------|-----------|
| **Data Types** | Rich (strings, hashes, lists, etc.) | Simple (key-value only) |
| **Persistence** | Yes (RDB, AOF) | No |
| **Replication** | Yes | No |
| **Clustering** | Yes | Client-side only |
| **Memory Efficiency** | Good | Better for simple use cases |

### Redis vs Database
| Aspect | Redis | Relational Database |
|--------|-------------------|-------------------|
| **Performance** | Microseconds | Milliseconds |
| **Durability** | Configurable | Strong (ACID) |
| **Data Model** | Key-value with types | Relational |
| **Query Language** | Limited | SQL |
| **Scaling** | Horizontal | Vertical (primarily) |

## Monitoring and Troubleshooting

### Key Metrics to Monitor
```bash
# Server information
INFO server
INFO stats
INFO memory
INFO replication
INFO cluster

# Slow log analysis
SLOWLOG GET 10

# Keyspace analysis
INFO keyspace
```

### Common Issues and Solutions

#### Memory Issues
- **High Memory Usage**: Monitor with INFO memory, adjust maxmemory
- **Memory Leaks**: Check for keys without TTL, monitor keyspace
- **Fragmentation**: Use MEMORY PURGE to defragment

#### Performance Issues
- **Slow Commands**: Check slowlog, optimize command usage
- **Connection Issues**: Monitor connection count, use connection pooling
- **Blocking Operations**: Avoid KEYS, SMEMBERS on large datasets

#### Replication Issues
- **Lag Monitoring**: Check master_last_io_seconds_ago
- **Network Issues**: Monitor replication connection status
- **Failover Testing**: Regularly test sentinel/cluster failover

## Conclusion

Redis is a powerful, versatile data structure store that excels as a cache, session store, and message broker. Its in-memory nature provides exceptional performance while its rich data types enable sophisticated use cases.

**Key Takeaways:**
- **In-Memory Performance**: Exceptional speed for read/write operations
- **Rich Data Types**: Beyond simple key-value storage
- **Persistence Options**: Choose between RDB and AOF based on needs
- **Scaling Options**: From simple replication to full clustering
- **Use Case Specific**: Choose Redis for the right scenarios

Redis is not a replacement for traditional databases but a complementary tool that enhances system performance and enables new architectural patterns. Understanding Redis design principles enables building high-performance, scalable systems.
