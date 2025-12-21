# Distributed Caching

Distributed caching provides high-performance, scalable data access across multiple nodes in a distributed system. It reduces database load, improves response times, and enables better scalability by caching frequently accessed data. This guide covers distributed caching patterns, implementations, and best practices.

## What is Distributed Caching?

Distributed caching stores data across multiple nodes in a cluster, allowing any node to access cached data regardless of where it was originally stored. Unlike local caching (single JVM), distributed caching works across process boundaries and survives node restarts.

### Key Characteristics
- **Distributed**: Data accessible from any cluster node
- **Scalable**: Grows with cluster size
- **Fault Tolerant**: Survives node failures
- **High Performance**: Low latency data access
- **Consistency**: Various consistency guarantees

### Distributed vs Local Caching

| Aspect | Distributed Caching | Local Caching |
|--------|-------------------|---------------|
| **Scope** | Cluster-wide | Single JVM |
| **Fault Tolerance** | Survives node failures | Lost on restart |
| **Consistency** | Configurable | Always consistent |
| **Scalability** | Limited by cluster size | Limited by JVM memory |
| **Network Overhead** | Required for access | None |
| **Use Cases** | Multi-node applications | Single-node optimization |

## Distributed Cache Architectures

### 1. Replicated Cache

#### Full Replication
All data replicated to every node in the cluster.

**Advantages:**
- **Fast Reads**: Data always local
- **High Availability**: Survives node failures
- **Simple**: No coordination needed for reads

**Disadvantages:**
- **Memory Intensive**: Data duplicated across all nodes
- **Update Overhead**: Updates sent to all nodes
- **Scalability Limits**: Memory usage grows linearly with cluster size

**Implementation:**
```java
@Configuration
public class ReplicatedCacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        return new HazelcastCacheManager(hazelcastInstance());
    }
    
    @Bean
    public HazelcastInstance hazelcastInstance() {
        Config config = new Config();
        
        // Configure replicated map
        MapConfig mapConfig = new MapConfig("replicated-cache");
        mapConfig.setBackupCount(0); // Full replication
        mapConfig.setAsyncBackupCount(0);
        
        config.addMapConfig(mapConfig);
        
        return Hazelcast.newHazelcastInstance(config);
    }
}
```

### 2. Partitioned Cache

#### Data Partitioning
Data divided into partitions distributed across cluster nodes.

**Advantages:**
- **Scalable**: Memory usage scales with data size
- **Efficient**: No data duplication
- **Flexible**: Add nodes to increase capacity

**Disadvantages:**
- **Network Access**: May require network calls for data access
- **Rebalancing**: Data redistribution when nodes join/leave
- **Complexity**: Partition management and routing

**Implementation:**
```java
@Configuration
public class PartitionedCacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        return new HazelcastCacheManager(hazelcastInstance());
    }
    
    @Bean
    public HazelcastInstance hazelcastInstance() {
        Config config = new Config();
        
        // Configure partitioned map
        MapConfig mapConfig = new MapConfig("partitioned-cache");
        mapConfig.setBackupCount(1); // One backup per partition
        mapConfig.setAsyncBackupCount(0);
        
        // Partition configuration
        PartitionGroupConfig partitionGroupConfig = new PartitionGroupConfig()
            .setEnabled(true)
            .setGroupType(PartitionGroupConfig.MemberGroupType.PER_MEMBER);
        
        config.setPartitionGroupConfig(partitionGroupConfig);
        config.addMapConfig(mapConfig);
        
        return Hazelcast.newHazelcastInstance(config);
    }
}
```

### 3. Near Cache

#### Local Cache with Distributed Backing
Combines local caching with distributed cache backing.

**Advantages:**
- **Fast Access**: Most reads from local cache
- **Reduced Network**: Fewer network calls
- **Scalability**: Benefits of distributed cache
- **Consistency**: Configurable consistency levels

**Disadvantages:**
- **Memory Usage**: Additional memory for local cache
- **Staleness**: Potential for stale local data
- **Complexity**: Cache invalidation management

**Implementation:**
```java
@Configuration
public class NearCacheConfig {
    
    @Bean
    public HazelcastInstance hazelcastInstance() {
        Config config = new Config();
        
        // Configure near cache
        NearCacheConfig nearCacheConfig = new NearCacheConfig("near-cache");
        nearCacheConfig.setInvalidateOnChange(true);
        nearCacheConfig.setTimeToLiveSeconds(300);
        nearCacheConfig.setMaxIdleSeconds(60);
        
        // Eviction policy
        EvictionConfig evictionConfig = new EvictionConfig()
            .setEvictionPolicy(EvictionPolicy.LRU)
            .setMaxSizePolicy(MaxSizePolicy.ENTRY_COUNT)
            .setSize(10000);
        
        nearCacheConfig.setEvictionConfig(evictionConfig);
        
        MapConfig mapConfig = new MapConfig("distributed-map");
        mapConfig.setNearCacheConfig(nearCacheConfig);
        
        config.addMapConfig(mapConfig);
        
        return Hazelcast.newHazelcastInstance(config);
    }
}
```

## Cache Consistency Models

### 1. Strong Consistency

#### Synchronous Updates
All cache updates synchronous and acknowledged.

```java
@Service
public class StronglyConsistentCache<K, V> {
    
    @Autowired
    private HazelcastInstance hazelcastInstance;
    
    public void put(K key, V value) {
        IMap<K, V> cache = hazelcastInstance.getMap("strong-cache");
        
        // Synchronous put with acknowledgment
        cache.put(key, value);
        
        // Wait for backup operations (if configured)
        // Hazelcast ensures consistency across backups
    }
    
    public V get(K key) {
        IMap<K, V> cache = hazelcastInstance.getMap("strong-cache");
        return cache.get(key); // Always returns latest value
    }
}
```

### 2. Eventual Consistency

#### Asynchronous Updates
Cache updates asynchronous with eventual convergence.

```java
@Service
public class EventuallyConsistentCache<K, V> {
    
    @Autowired
    private RedisTemplate<K, V> redisTemplate;
    
    public void put(K key, V value) {
        // Asynchronous write
        redisTemplate.opsForValue().set(key, value);
        
        // No waiting for acknowledgment
        // Data eventually consistent across replicas
    }
    
    public V get(K key) {
        return redisTemplate.opsForValue().get(key);
        
        // May return stale data immediately after write
        // Eventually consistent across all nodes
    }
}
```

### 3. Read-Through/Write-Through

#### Cache-Aside Pattern with Database
Cache automatically populated from database.

```java
@Service
public class ReadWriteThroughCache<K, V> {
    
    @Autowired
    private Cache<K, V> cache;
    
    @Autowired
    private Repository<K, V> repository;
    
    public V get(K key) {
        // Try cache first
        V value = cache.get(key);
        
        if (value == null) {
            // Read through to database
            value = repository.findById(key);
            
            if (value != null) {
                // Populate cache
                cache.put(key, value);
            }
        }
        
        return value;
    }
    
    public void put(K key, V value) {
        // Write to database first
        repository.save(value);
        
        // Then update cache
        cache.put(key, value);
    }
    
    public void delete(K key) {
        // Delete from database first
        repository.deleteById(key);
        
        // Then remove from cache
        cache.invalidate(key);
    }
}
```

## Cache Invalidation Strategies

### 1. Time-Based Expiration (TTL)

#### Automatic Expiration
Cache entries automatically expire after time duration.

```java
@Configuration
public class TTLCacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        
        // Configure TTL for different caches
        Caffeine<Object, Object> userCache = Caffeine.newBuilder()
            .expireAfterWrite(Duration.ofMinutes(30))
            .maximumSize(10000)
            .build();
        
        Caffeine<Object, Object> productCache = Caffeine.newBuilder()
            .expireAfterWrite(Duration.ofHours(1))
            .maximumSize(50000)
            .build();
        
        // Register caches
        cacheManager.registerCustomCache("users", userCache);
        cacheManager.registerCustomCache("products", productCache);
        
        return cacheManager;
    }
}
```

### 2. Event-Based Invalidation

#### Cache Invalidation on Data Changes
Cache automatically invalidated when underlying data changes.

```java
@Service
public class EventBasedCacheInvalidation {
    
    @Autowired
    private CacheManager cacheManager;
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleUserUpdate(UserUpdatedEvent event) {
        // Invalidate user cache
        cacheManager.getCache("users").evict(event.getUserId());
        
        // Invalidate related caches
        cacheManager.getCache("user-posts").evict(event.getUserId());
        cacheManager.getCache("user-friends").evict(event.getUserId());
    }
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleProductUpdate(ProductUpdatedEvent event) {
        // Invalidate product cache
        cacheManager.getCache("products").evict(event.getProductId());
        
        // Invalidate category cache if category changed
        if (event.categoryChanged()) {
            cacheManager.getCache("category-products").evict(event.getOldCategoryId());
            cacheManager.getCache("category-products").evict(event.getNewCategoryId());
        }
    }
}
```

### 3. Write-Behind Caching

#### Asynchronous Cache Updates
Cache updates happen asynchronously after database writes.

```java
@Service
public class WriteBehindCache<K, V> {
    
    @Autowired
    private Cache<K, V> cache;
    
    @Autowired
    private Repository<K, V> repository;
    
    private final BlockingQueue<CacheOperation<K, V>> operationQueue = 
        new LinkedBlockingQueue<>();
    
    private final ExecutorService executor = Executors.newSingleThreadExecutor();
    
    @PostConstruct
    public void startWriteBehindProcessor() {
        executor.submit(this::processOperations);
    }
    
    public void put(K key, V value) {
        // Update cache immediately
        cache.put(key, value);
        
        // Queue database update
        operationQueue.offer(new CacheOperation<>(OperationType.PUT, key, value));
    }
    
    public void delete(K key) {
        // Remove from cache immediately
        cache.invalidate(key);
        
        // Queue database delete
        operationQueue.offer(new CacheOperation<>(OperationType.DELETE, key, null));
    }
    
    private void processOperations() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                CacheOperation<K, V> operation = operationQueue.take();
                
                switch (operation.getType()) {
                    case PUT:
                        repository.save(operation.getValue());
                        break;
                    case DELETE:
                        repository.deleteById(operation.getKey());
                        break;
                }
                
            } catch (Exception e) {
                logger.error("Failed to process cache operation", e);
                
                // Implement retry logic or dead letter queue
            }
        }
    }
    
    @PreDestroy
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

## Popular Distributed Cache Implementations

### 1. Redis Cluster

#### Redis Distributed Caching
```java
@Configuration
public class RedisClusterConfig {
    
    @Bean
    public RedisClusterConfiguration redisClusterConfig() {
        RedisClusterConfiguration config = new RedisClusterConfiguration();
        
        // Add cluster nodes
        config.addClusterNode(new RedisNode("redis-node1", 6379));
        config.addClusterNode(new RedisNode("redis-node2", 6379));
        config.addClusterNode(new RedisNode("redis-node3", 6379));
        
        config.setMaxRedirects(3);
        
        return config;
    }
    
    @Bean
    public JedisConnectionFactory connectionFactory(RedisClusterConfiguration config) {
        return new JedisConnectionFactory(config);
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(JedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        
        // Configure serializers
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        
        return template;
    }
    
    @Bean
    public CacheManager cacheManager(RedisTemplate<String, Object> redisTemplate) {
        RedisCacheManager cacheManager = RedisCacheManager.builder(redisTemplate.getConnectionFactory())
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer())))
            .build();
        
        return cacheManager;
    }
}
```

### 2. Hazelcast IMDG

#### Hazelcast Distributed Caching
```java
@Configuration
public class HazelcastConfig {
    
    @Bean
    public Config hazelcastConfig() {
        Config config = new Config();
        
        // Network configuration
        NetworkConfig network = config.getNetworkConfig();
        network.setPort(5701);
        network.setPortAutoIncrement(true);
        
        JoinConfig join = network.getJoin();
        join.getMulticastConfig().setEnabled(false);
        join.getTcpIpConfig().setEnabled(true)
            .addMember("hazelcast-node1")
            .addMember("hazelcast-node2")
            .addMember("hazelcast-node3");
        
        // Map configurations
        MapConfig userCache = new MapConfig("user-cache");
        userCache.setBackupCount(1);
        userCache.setAsyncBackupCount(1);
        userCache.setTimeToLiveSeconds(3600); // 1 hour
        
        // Eviction
        EvictionConfig eviction = new EvictionConfig()
            .setEvictionPolicy(EvictionPolicy.LRU)
            .setMaxSizePolicy(MaxSizePolicy.USED_HEAP_SIZE)
            .setSize(50); // 50% of heap
        
        userCache.setEvictionConfig(eviction);
        
        config.addMapConfig(userCache);
        
        // Near cache configuration
        NearCacheConfig nearCache = new NearCacheConfig("user-cache");
        nearCache.setTimeToLiveSeconds(300);
        nearCache.setInvalidateOnChange(true);
        
        config.getMapConfig("user-cache").setNearCacheConfig(nearCache);
        
        return config;
    }
    
    @Bean
    public HazelcastInstance hazelcastInstance(Config config) {
        return Hazelcast.newHazelcastInstance(config);
    }
    
    @Bean
    public CacheManager cacheManager(HazelcastInstance hazelcastInstance) {
        return new HazelcastCacheManager(hazelcastInstance);
    }
}
```

### 3. Apache Ignite

#### Ignite Data Grid
```java
@Configuration
public class IgniteConfig {
    
    @Bean
    public Ignite ignite() {
        IgniteConfiguration config = new IgniteConfiguration();
        
        // Cluster configuration
        TcpDiscoverySpi discoverySpi = new TcpDiscoverySpi();
        TcpDiscoveryVmIpFinder ipFinder = new TcpDiscoveryVmIpFinder();
        ipFinder.setAddresses(Arrays.asList(
            "ignite-node1:47500..47509",
            "ignite-node2:47500..47509",
            "ignite-node3:47500..47509"
        ));
        discoverySpi.setIpFinder(ipFinder);
        config.setDiscoverySpi(discoverySpi);
        
        // Cache configuration
        CacheConfiguration<Long, User> userCacheCfg = new CacheConfiguration<>("user-cache");
        userCacheCfg.setCacheMode(CacheMode.PARTITIONED);
        userCacheCfg.setBackups(1);
        userCacheCfg.setAtomicityMode(CacheAtomicityMode.ATOMIC);
        
        // Query configuration
        userCacheCfg.setIndexedTypes(Long.class, User.class);
        
        // Expiration
        userCacheCfg.setExpiryPolicyFactory(
            new ExpiryPolicyFactory(new Duration[]{Duration.ZERO, Duration.ZERO, Duration.fromMillis(3600000)}));
        
        config.setCacheConfiguration(userCacheCfg);
        
        // Data region configuration
        DataRegionConfiguration regionCfg = new DataRegionConfiguration();
        regionCfg.setName("default-region");
        regionCfg.setMaxSize(100L * 1024 * 1024); // 100MB
        
        DataStorageConfiguration storageCfg = new DataStorageConfiguration();
        storageCfg.setDataRegionConfigurations(regionCfg);
        config.setDataStorageConfiguration(storageCfg);
        
        return Ignition.start(config);
    }
    
    @Bean
    public IgniteCache<Long, User> userCache(Ignite ignite) {
        return ignite.cache("user-cache");
    }
}
```

## Cache Performance Optimization

### 1. Cache Partitioning

#### Consistent Hashing
Distribute cache keys evenly across cluster nodes.

```java
public class ConsistentHashRouter {
    
    private final SortedMap<Integer, CacheNode> ring = new TreeMap<>();
    private final HashFunction hashFunction = Hashing.murmur3_128();
    
    public void addNode(CacheNode node) {
        for (int i = 0; i < node.getWeight(); i++) {
            int hash = hashFunction.hashString(node.getId() + ":" + i, StandardCharsets.UTF_8).asInt();
            ring.put(hash, node);
        }
    }
    
    public void removeNode(CacheNode node) {
        ring.values().removeIf(n -> n.getId().equals(node.getId()));
    }
    
    public CacheNode getNode(String key) {
        if (ring.isEmpty()) {
            return null;
        }
        
        int hash = hashFunction.hashString(key, StandardCharsets.UTF_8).asInt();
        SortedMap<Integer, CacheNode> tailMap = ring.tailMap(hash);
        
        Integer nodeHash = tailMap.isEmpty() ? ring.firstKey() : tailMap.firstKey();
        return ring.get(nodeHash);
    }
    
    public List<CacheNode> getNodes(String key, int replicaCount) {
        List<CacheNode> nodes = new ArrayList<>();
        Set<String> addedNodes = new HashSet<>();
        
        int hash = hashFunction.hashString(key, StandardCharsets.UTF_8).asInt();
        SortedMap<Integer, CacheNode> tailMap = ring.tailMap(hash);
        
        // Add nodes in hash ring order
        for (Integer nodeHash : tailMap.keySet()) {
            CacheNode node = ring.get(nodeHash);
            if (!addedNodes.contains(node.getId())) {
                nodes.add(node);
                addedNodes.add(node.getId());
                if (nodes.size() >= replicaCount) break;
            }
        }
        
        // Wrap around if needed
        if (nodes.size() < replicaCount) {
            for (Integer nodeHash : ring.keySet()) {
                CacheNode node = ring.get(nodeHash);
                if (!addedNodes.contains(node.getId())) {
                    nodes.add(node);
                    addedNodes.add(node.getId());
                    if (nodes.size() >= replicaCount) break;
                }
            }
        }
        
        return nodes;
    }
}
```

### 2. Cache Preloading

#### Cache Warm-up Strategies
```java
@Service
public class CacheWarmUpService {
    
    @Autowired
    private CacheManager cacheManager;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private ProductRepository productRepository;
    
    @PostConstruct
    public void warmUpCaches() {
        logger.info("Starting cache warm-up");
        
        // Warm up user cache
        warmUpUserCache();
        
        // Warm up product cache
        warmUpProductCache();
        
        logger.info("Cache warm-up completed");
    }
    
    private void warmUpUserCache() {
        Cache userCache = cacheManager.getCache("users");
        
        // Load frequently accessed users
        List<User> frequentUsers = userRepository.findTopUsersByAccessCount(1000);
        
        for (User user : frequentUsers) {
            userCache.put(user.getId(), user);
        }
        
        logger.info("Warmed up {} users in cache", frequentUsers.size());
    }
    
    private void warmUpProductCache() {
        Cache productCache = cacheManager.getCache("products");
        
        // Load popular products
        List<Product> popularProducts = productRepository.findTopProductsBySales(5000);
        
        for (Product product : popularProducts) {
            productCache.put(product.getId(), product);
        }
        
        logger.info("Warmed up {} products in cache", popularProducts.size());
    }
    
    @Scheduled(fixedRate = 3600000) // Hourly refresh
    public void refreshHotData() {
        // Refresh most accessed data
        refreshHotUsers();
        refreshHotProducts();
    }
    
    private void refreshHotUsers() {
        Cache userCache = cacheManager.getCache("users");
        
        // Get recently accessed users
        List<Long> recentUserIds = getRecentlyAccessedUserIds();
        
        for (Long userId : recentUserIds) {
            User user = userRepository.findById(userId);
            if (user != null) {
                userCache.put(userId, user);
            }
        }
    }
    
    private void refreshHotProducts() {
        Cache productCache = cacheManager.getCache("products");
        
        // Get recently accessed products
        List<Long> recentProductIds = getRecentlyAccessedProductIds();
        
        for (Long productId : recentProductIds) {
            Product product = productRepository.findById(productId);
            if (product != null) {
                productCache.put(productId, product);
            }
        }
    }
}
```

### 3. Cache Compression

#### Data Compression in Cache
```java
@Configuration
public class CompressedCacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheManager cacheManager = RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new CompressedJsonSerializer())))
            .build();
        
        return cacheManager;
    }
    
    public static class CompressedJsonSerializer implements RedisSerializer<Object> {
        
        private final ObjectMapper objectMapper = new ObjectMapper();
        private final GZIPOutputStream gzipOut;
        private final GZIPInputStream gzipIn;
        
        @Override
        public byte[] serialize(Object object) throws SerializationException {
            try {
                byte[] jsonBytes = objectMapper.writeValueAsBytes(object);
                
                // Compress if data is large enough
                if (jsonBytes.length > 1024) { // Compress if > 1KB
                    ByteArrayOutputStream baos = new ByteArrayOutputStream();
                    try (GZIPOutputStream gzipOut = new GZIPOutputStream(baos)) {
                        gzipOut.write(jsonBytes);
                    }
                    return baos.toByteArray();
                } else {
                    return jsonBytes;
                }
                
            } catch (Exception e) {
                throw new SerializationException("Failed to serialize object", e);
            }
        }
        
        @Override
        public Object deserialize(byte[] bytes) throws SerializationException {
            try {
                // Try to detect if compressed
                if (isCompressed(bytes)) {
                    ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
                    try (GZIPInputStream gzipIn = new GZIPInputStream(bais)) {
                        return objectMapper.readValue(gzipIn, Object.class);
                    }
                } else {
                    return objectMapper.readValue(bytes, Object.class);
                }
                
            } catch (Exception e) {
                throw new SerializationException("Failed to deserialize object", e);
            }
        }
        
        private boolean isCompressed(byte[] bytes) {
            // Check GZIP magic number
            return bytes.length >= 2 && 
                   bytes[0] == (byte) 0x1f && 
                   bytes[1] == (byte) 0x8b;
        }
    }
}
```

## Cache Monitoring and Observability

### Cache Metrics
```java
@Service
public class DistributedCacheMetrics {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter cacheHits = Counter.builder("cache_hits_total")
        .description("Total cache hits")
        .tag("cache", "")
        .register(meterRegistry);
    
    private final Counter cacheMisses = Counter.builder("cache_misses_total")
        .description("Total cache misses")
        .tag("cache", "")
        .register(meterRegistry);
    
    private final Histogram cacheOperationTime = Histogram.builder("cache_operation_time")
        .description("Cache operation time")
        .tag("operation", "")
        .register(meterRegistry);
    
    private final Gauge cacheSize = Gauge.builder("cache_size")
        .description("Cache size")
        .tag("cache", "")
        .register(meterRegistry);
    
    private final Counter cacheEvictions = Counter.builder("cache_evictions_total")
        .description("Total cache evictions")
        .tag("cache", "")
        .register(meterRegistry);
    
    public void recordCacheHit(String cacheName) {
        cacheHits.withTag("cache", cacheName).increment();
    }
    
    public void recordCacheMiss(String cacheName) {
        cacheMisses.withTag("cache", cacheName).increment();
    }
    
    public void recordCacheOperation(String cacheName, String operation, long timeMs) {
        cacheOperationTime.withTag("cache", cacheName)
                         .withTag("operation", operation)
                         .observe(timeMs / 1000.0);
    }
    
    public void recordCacheEviction(String cacheName) {
        cacheEvictions.withTag("cache", cacheName).increment();
    }
    
    public double getCacheHitRate(String cacheName) {
        double hits = cacheHits.withTag("cache", cacheName).count();
        double misses = cacheMisses.withTag("cache", cacheName).count();
        double total = hits + misses;
        return total > 0 ? hits / total : 0.0;
    }
}
```

### Cache Health Monitoring
```java
@Service
public class CacheHealthIndicator implements HealthIndicator {
    
    @Autowired
    private CacheManager cacheManager;
    
    @Autowired
    private DistributedCacheMetrics metrics;
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        boolean overallHealthy = true;
        
        // Check each cache
        for (String cacheName : cacheManager.getCacheNames()) {
            Cache cache = cacheManager.getCache(cacheName);
            
            try {
                // Test basic cache operations
                String testKey = "health-check-" + System.currentTimeMillis();
                String testValue = "test-value";
                
                // Test put
                cache.put(testKey, testValue);
                
                // Test get
                Object retrieved = cache.get(testKey);
                if (!testValue.equals(retrieved)) {
                    details.put(cacheName + "_status", "inconsistent");
                    overallHealthy = false;
                } else {
                    details.put(cacheName + "_status", "healthy");
                }
                
                // Test hit rate
                double hitRate = metrics.getCacheHitRate(cacheName);
                details.put(cacheName + "_hit_rate", String.format("%.2f%%", hitRate * 100));
                
                if (hitRate < 0.5) { // Less than 50% hit rate
                    details.put(cacheName + "_warning", "low hit rate");
                }
                
            } catch (Exception e) {
                details.put(cacheName + "_status", "unhealthy");
                details.put(cacheName + "_error", e.getMessage());
                overallHealthy = false;
            }
        }
        
        if (overallHealthy) {
            return Health.up().withDetails(details).build();
        } else {
            return Health.down().withDetails(details).build();
        }
    }
}
```

## Best Practices

### Cache Design
- **Appropriate TTL**: Set reasonable expiration times
- **Size Limits**: Prevent cache from consuming too much memory
- **Serialization**: Use efficient serialization formats
- **Key Design**: Design cache keys for optimal distribution

### Cache Operations
- **Cache-Aside**: Load data into cache on demand
- **Write-Through**: Update cache and database together
- **Write-Behind**: Update cache immediately, database asynchronously
- **Refresh-Ahead**: Proactively refresh expiring data

### Operational Considerations
- **Monitoring**: Track cache hit rates, latency, and errors
- **Scaling**: Plan for cache growth and cluster expansion
- **Backup**: Implement cache persistence and recovery
- **Security**: Secure cache access and data encryption

### Performance Tuning
- **Connection Pooling**: Reuse connections to cache servers
- **Async Operations**: Use async cache operations when possible
- **Batch Operations**: Batch multiple cache operations
- **Pipeline Operations**: Use Redis pipelines for multiple commands

## Real-World Examples

### E-commerce Product Caching
```java
@Service
public class ProductCacheService {
    
    @Autowired
    private CacheManager cacheManager;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Cacheable(value = "products", key = "#productId")
    public Product getProduct(Long productId) {
        return productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }
    
    @Cacheable(value = "product-search", key = "#query + '_' + #page + '_' + #size")
    public Page<Product> searchProducts(String query, int page, int size) {
        return productRepository.search(query, PageRequest.of(page, size));
    }
    
    @CacheEvict(value = "products", key = "#product.id")
    @CacheEvict(value = "product-search", allEntries = true)
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }
    
    @Cacheable(value = "product-recommendations", key = "#userId")
    public List<Product> getRecommendations(Long userId) {
        // Complex recommendation algorithm
        return recommendationEngine.getRecommendations(userId);
    }
}
```

### User Session Caching
```java
@Service
public class DistributedSessionService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final String SESSION_PREFIX = "session:";
    private static final Duration SESSION_TTL = Duration.ofHours(24);
    
    public void createSession(String sessionId, UserSession session) {
        String key = SESSION_PREFIX + sessionId;
        redisTemplate.opsForValue().set(key, session, SESSION_TTL);
        
        // Store session ID in user sessions set
        String userSessionsKey = "user-sessions:" + session.getUserId();
        redisTemplate.opsForSet().add(userSessionsKey, sessionId);
        redisTemplate.expire(userSessionsKey, SESSION_TTL);
    }
    
    public UserSession getSession(String sessionId) {
        String key = SESSION_PREFIX + sessionId;
        return (UserSession) redisTemplate.opsForValue().get(key);
    }
    
    public void updateSession(String sessionId, UserSession session) {
        String key = SESSION_PREFIX + sessionId;
        redisTemplate.opsForValue().set(key, session, SESSION_TTL);
    }
    
    public void invalidateSession(String sessionId) {
        String key = SESSION_PREFIX + sessionId;
        
        // Get session to find user ID
        UserSession session = (UserSession) redisTemplate.opsForValue().get(key);
        if (session != null) {
            // Remove from user sessions set
            String userSessionsKey = "user-sessions:" + session.getUserId();
            redisTemplate.opsForSet().remove(userSessionsKey, sessionId);
        }
        
        // Delete session
        redisTemplate.delete(key);
    }
    
    public void invalidateAllUserSessions(Long userId) {
        String userSessionsKey = "user-sessions:" + userId;
        Set<Object> sessionIds = redisTemplate.opsForSet().members(userSessionsKey);
        
        if (sessionIds != null) {
            List<String> sessionKeys = sessionIds.stream()
                .map(id -> SESSION_PREFIX + id)
                .collect(Collectors.toList());
            
            redisTemplate.delete(sessionKeys);
        }
        
        redisTemplate.delete(userSessionsKey);
    }
    
    public List<UserSession> getUserSessions(Long userId) {
        String userSessionsKey = "user-sessions:" + userId;
        Set<Object> sessionIds = redisTemplate.opsForSet().members(userSessionsKey);
        
        if (sessionIds == null) {
            return Collections.emptyList();
        }
        
        return sessionIds.stream()
            .map(id -> (String) id)
            .map(this::getSession)
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
}
```

## Conclusion

Distributed caching is essential for building high-performance, scalable distributed systems. By implementing proper caching strategies and choosing appropriate cache implementations, systems can significantly improve response times and reduce database load.

**Key Takeaways:**
- **Architecture Choice**: Select replication vs partitioning based on data size and access patterns
- **Consistency Model**: Choose appropriate consistency guarantees for your use case
- **Invalidation Strategy**: Implement proper cache invalidation and TTL policies
- **Monitoring**: Track cache performance and hit rates
- **Scaling**: Plan for cache growth and cluster expansion

Effective distributed caching requires understanding cache architectures, consistency models, and operational considerations to achieve optimal performance and reliability.
