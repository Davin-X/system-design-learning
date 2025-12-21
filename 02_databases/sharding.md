# Database Sharding

Database sharding is a horizontal partitioning strategy that distributes data across multiple database servers to improve scalability, performance, and manageability. It enables systems to handle massive datasets and high throughput by breaking them into smaller, more manageable pieces.

## What is Database Sharding?

Sharding splits a large database into smaller, more manageable pieces called "shards." Each shard is a separate database that contains a subset of the total data. Sharding is different from replication - while replication creates copies of the same data, sharding splits the data itself.

### Why Sharding Matters
- **Scalability**: Handle petabytes of data across multiple servers
- **Performance**: Reduce query time by limiting data scanned
- **Cost Efficiency**: Use commodity hardware instead of expensive single servers
- **Geographic Distribution**: Place data closer to users
- **Maintenance**: Easier backup, restore, and maintenance operations

## Sharding Fundamentals

### Shard vs Partition
- **Shard**: A physical database instance containing a subset of data
- **Partition**: A logical division of data within a single database
- **Sharding**: Distributing data across multiple physical databases
- **Partitioning**: Organizing data within a single database

### Shard Key
The column or set of columns used to determine which shard contains a particular row.

**Characteristics of Good Shard Keys:**
- **High Cardinality**: Many possible values to distribute evenly
- **Uniform Distribution**: Values spread evenly across range
- **Query Alignment**: Matches common query patterns
- **Immutable**: Doesn't change frequently

**Examples:**
- User ID for user-centric applications
- Geographic region for location-based apps
- Timestamp ranges for time-series data
- Hash of primary key for even distribution

## Sharding Strategies

### Range-Based Sharding

#### How It Works
Data is divided based on ranges of the shard key values.

**Example:**
```
Shard 1: User IDs 1-100,000
Shard 2: User IDs 100,001-200,000
Shard 3: User IDs 200,001-300,000
```

#### Advantages
- **Simple Implementation**: Easy to understand and implement
- **Range Queries**: Efficient for queries on consecutive ranges
- **Scalable**: Easy to add new ranges as data grows

#### Disadvantages
- **Uneven Distribution**: Some shards may grow faster than others
- **Hotspots**: Popular ranges create overloaded shards
- **Rebalancing**: Complex to redistribute data when ranges change

#### Use Cases
- **Time-Series Data**: Partition by date ranges
- **Sequential IDs**: Natural ordering benefits
- **Known Distributions**: When data distribution is predictable

### Hash-Based Sharding

#### How It Works
Apply a hash function to the shard key to determine the shard.

**Example:**
```java
int shardId = Math.abs(userId.hashCode()) % numShards;
```

#### Advantages
- **Even Distribution**: Random hash ensures uniform data distribution
- **No Hotspots**: No shard becomes disproportionately large
- **Simple Routing**: Hash function determines shard quickly

#### Disadvantages
- **Range Queries**: Impossible to query across ranges
- **Rebalancing**: Complex to change number of shards
- **Hash Function**: Must be consistent across all clients

#### Use Cases
- **Random Access Patterns**: When queries are primarily by ID
- **Uniform Data Growth**: When data grows evenly across all keys
- **High Write Loads**: Even distribution prevents hotspots

### Directory-Based Sharding

#### How It Works
Use a lookup table to map shard keys to shards.

**Architecture:**
```
Shard Key → Lookup Service → Shard Location
```

#### Advantages
- **Flexible Mapping**: Can change shard assignments dynamically
- **Complex Logic**: Support sophisticated routing rules
- **Rebalancing**: Easy to move data between shards

#### Disadvantages
- **Single Point of Failure**: Lookup service becomes critical
- **Performance Overhead**: Additional lookup for every query
- **Complexity**: More moving parts to manage

#### Use Cases
- **Dynamic Sharding**: When shard assignments change frequently
- **Complex Routing**: Advanced routing logic required
- **Multi-Tenant Systems**: Different tenants on different shards

### Geographic Sharding

#### How It Works
Place data in shards based on geographic location.

**Example:**
```
US East: Users in Eastern US
US West: Users in Western US
Europe: Users in European countries
Asia: Users in Asian countries
```

#### Advantages
- **Latency Reduction**: Data closer to users
- **Compliance**: Meet data residency requirements
- **Regional Scaling**: Scale regions independently

#### Disadvantages
- **Cross-Region Queries**: Complex and slow
- **Data Consistency**: Harder to maintain across regions
- **Network Costs**: Expensive cross-region communication

#### Use Cases
- **Global Applications**: Users worldwide
- **Regulatory Compliance**: GDPR, CCPA requirements
- **Low Latency**: Real-time applications needing fast response

## Implementing Sharding

### Application-Level Sharding
Application code determines which shard contains the data.

#### Advantages
- **Database Agnostic**: Works with any database
- **Full Control**: Application controls all routing logic
- **Flexibility**: Easy to change sharding strategy

#### Disadvantages
- **Application Complexity**: Sharding logic in application code
- **Maintenance Burden**: Application must track shard locations
- **Consistency**: Harder to maintain cross-shard consistency

#### Implementation
```java
@Service
public class UserShardResolver {
    
    private static final int NUM_SHARDS = 4;
    private final List<DataSource> shards;
    
    public UserShardResolver(List<DataSource> shards) {
        this.shards = shards;
    }
    
    public DataSource getShardForUser(Long userId) {
        int shardIndex = (int) (userId % NUM_SHARDS);
        return shards.get(shardIndex);
    }
    
    public User getUser(Long userId) {
        DataSource shard = getShardForUser(userId);
        // Execute query on the appropriate shard
        return userRepository.findByIdAndShard(userId, shard);
    }
}
```

### Database-Level Sharding

#### MySQL Sharding
```sql
-- Create partitioned table
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255),
    created_at TIMESTAMP
) PARTITION BY HASH(id) PARTITIONS 4;
```

#### PostgreSQL Sharding
```sql
-- Create foreign tables for each shard
CREATE FOREIGN TABLE users_shard1 () INHERITS (users) 
SERVER shard1_server OPTIONS (table_name 'users');

-- Use partitioning
CREATE TABLE users (
    id BIGINT,
    name VARCHAR(100),
    email VARCHAR(255),
    created_at TIMESTAMP
) PARTITION BY HASH (id);

CREATE TABLE users_p0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_p1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
```

#### MongoDB Sharding
```javascript
// Enable sharding on database
sh.enableSharding("myapp");

// Create hashed shard key
db.users.createIndex({ "_id": "hashed" });

// Shard the collection
sh.shardCollection("myapp.users", { "_id": "hashed" });

// Check shard status
sh.status();
```

## Cross-Shard Queries

### Challenges
- **Joins**: Cannot join data across shards easily
- **Aggregations**: Complex to aggregate across all shards
- **Consistency**: Maintaining consistency across shards
- **Performance**: Cross-shard queries are expensive

### Solutions

#### Scatter-Gather Pattern
Send query to all shards and aggregate results.

```java
public List<User> findUsersByStatus(String status) {
    List<CompletableFuture<List<User>>> futures = new ArrayList<>();
    
    for (DataSource shard : allShards) {
        CompletableFuture<List<User>> future = CompletableFuture.supplyAsync(() -> 
            userRepository.findByStatusAndShard(status, shard)
        );
        futures.add(future);
    }
    
    // Wait for all shards to respond
    return futures.stream()
        .map(CompletableFuture::join)
        .flatMap(List::stream)
        .collect(Collectors.toList());
}
```

#### Denormalization
Duplicate data across shards to avoid cross-shard joins.

**Example:**
- Store user profile on all shards they interact with
- Use eventual consistency to sync changes
- Accept some data staleness for performance

#### Application-Level Joins
Fetch data from multiple shards in application code.

```java
public OrderDetails getOrderDetails(Long orderId) {
    // Get order from shard
    Order order = orderRepository.findById(orderId);
    
    // Get user from user shard
    User user = userService.getUser(order.getUserId());
    
    // Get items from item shard
    List<OrderItem> items = orderItemService.getItems(orderId);
    
    return new OrderDetails(order, user, items);
}
```

## Rebalancing and Resharding

### Why Rebalancing is Needed
- **Uneven Growth**: Some shards grow faster than others
- **Performance Issues**: Hot shards become bottlenecks
- **Capacity Planning**: Adding new shards as data grows
- **Geographic Changes**: Moving data closer to users

### Rebalancing Strategies

#### Online Rebalancing
Move data while system continues operating.

**Steps:**
1. **Add New Shard**: Add new shard to cluster
2. **Data Migration**: Gradually move data from old to new shards
3. **Update Routing**: Update routing logic for moved data
4. **Remove Old Shards**: Decommission old shards when empty

#### Consistent Hashing
Use consistent hashing to minimize data movement.

**Benefits:**
- **Minimal Movement**: Only affected keys move when adding shards
- **Load Distribution**: Even distribution of keys
- **Scalability**: Easy to add/remove shards

**Implementation:**
```java
public class ConsistentHashRing {
    private final SortedMap<Integer, Shard> ring = new TreeMap<>();
    private final int replicas;
    
    public void addShard(Shard shard) {
        for (int i = 0; i < replicas; i++) {
            int hash = hash(shard.getId() + ":" + i);
            ring.put(hash, shard);
        }
    }
    
    public Shard getShard(String key) {
        int hash = hash(key);
        if (!ring.containsKey(hash)) {
            SortedMap<Integer, Shard> tailMap = ring.tailMap(hash);
            hash = tailMap.isEmpty() ? ring.firstKey() : tailMap.firstKey();
        }
        return ring.get(hash);
    }
}
```

## Sharding Best Practices

### Shard Key Selection
- **Immutable Keys**: Choose keys that don't change
- **High Cardinality**: Many possible values for even distribution
- **Query Patterns**: Align with most common query patterns
- **Growth Patterns**: Consider how data will grow over time

### Data Distribution
- **Monitor Skew**: Track data distribution across shards
- **Capacity Planning**: Plan for growth and rebalancing
- **Hotspot Prevention**: Avoid shard keys that create hotspots
- **Backup Strategy**: Backup shards independently

### Query Optimization
- **Shard-Aware Queries**: Design queries to target specific shards
- **Index Strategy**: Proper indexing within each shard
- **Denormalization**: Consider duplicating data to avoid cross-shard queries
- **Caching**: Cache frequently accessed cross-shard data

### Operational Considerations
- **Monitoring**: Track shard performance and utilization
- **Alerting**: Set up alerts for shard issues
- **Testing**: Test sharding logic thoroughly
- **Documentation**: Document shard key strategy and boundaries

## Sharding Anti-Patterns

### Monolithic Shard Key
**Problem:** Using a single shard key for all queries
**Solution:** Design multiple access patterns, consider compound keys

### Over-Sharding
**Problem:** Too many small shards increase complexity
**Solution:** Balance shard size with management overhead

### Ignoring Cross-Shard Operations
**Problem:** Frequent cross-shard queries kill performance
**Solution:** Design data model to minimize cross-shard operations

### Poor Shard Key Choice
**Problem:** Shard key causes hotspots or uneven distribution
**Solution:** Analyze query patterns and data distribution before choosing

## Sharding in Java Applications

### Spring Boot Sharding Implementation
```java
@Configuration
public class ShardingConfig {
    
    @Bean
    public ShardRouter shardRouter() {
        Map<String, DataSource> shardDataSources = new HashMap<>();
        shardDataSources.put("shard1", createDataSource("db1"));
        shardDataSources.put("shard2", createDataSource("db2"));
        shardDataSources.put("shard3", createDataSource("db3"));
        shardDataSources.put("shard4", createDataSource("db4"));
        
        return new HashBasedShardRouter(shardDataSources);
    }
    
    @Bean
    public UserRepository userRepository(ShardRouter shardRouter) {
        return new ShardedUserRepository(shardRouter);
    }
}
```

### Shard-Aware Repository
```java
@Repository
public class ShardedUserRepository implements UserRepository {
    
    private final ShardRouter shardRouter;
    private final UserRepositoryImpl userRepositoryImpl;
    
    public ShardedUserRepository(ShardRouter shardRouter) {
        this.shardRouter = shardRouter;
        this.userRepositoryImpl = new UserRepositoryImpl();
    }
    
    @Override
    public User findById(Long userId) {
        DataSource shard = shardRouter.getShardForUserId(userId);
        return userRepositoryImpl.findById(userId, shard);
    }
    
    @Override
    public User save(User user) {
        DataSource shard = shardRouter.getShardForUserId(user.getId());
        return userRepositoryImpl.save(user, shard);
    }
}
```

## Real-World Sharding Examples

### Twitter
- **Shard by User ID**: Tweets stored on user's shard
- **Timeline Generation**: Complex cross-shard queries for timelines
- **Caching**: Heavy use of caching to reduce shard hits

### Instagram
- **Geographic Sharding**: Data stored in region closest to user
- **User-Centric Sharding**: All user data on same shard
- **CDN Integration**: Global content delivery

### Uber
- **Geographic Sharding**: Data partitioned by city/region
- **Real-Time Requirements**: Low-latency access to nearby data
- **Dynamic Rebalancing**: Shards adjusted based on demand

### Airbnb
- **Search Optimization**: Listings sharded for efficient search
- **Geographic Queries**: Complex location-based queries
- **Caching Layers**: Multiple caching layers to reduce shard load

## Monitoring Sharding

### Key Metrics
- **Shard Utilization**: CPU, memory, disk usage per shard
- **Query Distribution**: Which shards receive most queries
- **Cross-Shard Queries**: Frequency and performance of multi-shard queries
- **Data Distribution**: Row count and size per shard

### Health Checks
- **Shard Connectivity**: Can application reach all shards
- **Replication Lag**: If shards are replicated
- **Query Performance**: Response times per shard
- **Error Rates**: Failed queries per shard

### Alerting
- **Shard Down**: Alert when shard becomes unreachable
- **High Utilization**: Alert when shard resources are overused
- **Slow Queries**: Alert on queries taking too long
- **Data Skew**: Alert when shards become unbalanced

## Conclusion

Database sharding is a powerful technique for scaling databases beyond the limits of single servers. It enables handling massive datasets and high throughput by distributing data across multiple servers.

**Key Takeaways:**
- **Shard Key Selection**: Critical for performance and scalability
- **Distribution Strategy**: Choose based on access patterns
- **Cross-Shard Challenges**: Minimize when possible
- **Monitoring**: Essential for maintaining performance
- **Rebalancing**: Plan for ongoing data redistribution

Effective sharding requires careful planning, monitoring, and maintenance. When implemented correctly, it enables systems to scale to millions of users and petabytes of data while maintaining good performance and reliability.
