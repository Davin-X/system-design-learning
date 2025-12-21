# System Design Patterns Cheatsheet

Quick reference guide for essential system design patterns, their use cases, trade-offs, and implementation considerations.

## 📋 Load Balancing Patterns

### Round Robin
**When to Use:**
- Equal distribution of load
- Stateless services
- Predictable traffic patterns

**When NOT to Use:**
- Services with different capacities
- Session affinity requirements

**Implementation:**
```java
// Simple round-robin implementation
public class RoundRobinLoadBalancer {
    private final List<String> servers;
    private int currentIndex = 0;

    public String getNextServer() {
        synchronized(this) {
            String server = servers.get(currentIndex);
            currentIndex = (currentIndex + 1) % servers.size();
            return server;
        }
    }
}
```

**Trade-offs:**
| Aspect | Pros | Cons |
|--------|------|------|
| Simplicity | ✅ Very simple | ❌ Ignores server load |
| Fairness | ✅ Equal distribution | ❌ No health checks |
| Performance | ✅ Low overhead | ❌ Not adaptive |

### Least Connections
**When to Use:**
- Variable request processing times
- Server capacity varies
- Real-time load balancing needed

**Trade-offs:**
- ✅ Adapts to server load
- ❌ Requires health monitoring
- ✅ Better resource utilization

### IP Hash
**When to Use:**
- Session persistence required
- Sticky sessions needed
- Cache locality important

**Trade-offs:**
- ✅ Session persistence
- ❌ Uneven load distribution
- ❌ Doesn't handle server failures well

## 🗄️ Database Patterns

### Read Replicas
**When to Use:**
- Read-heavy workloads
- High read-to-write ratio (>80% reads)
- Need to scale reads independently

**Implementation:**
```sql
-- Read replica configuration (MySQL)
CHANGE MASTER TO
    MASTER_HOST='master-host',
    MASTER_USER='replica-user',
    MASTER_PASSWORD='password',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=0;
START SLAVE;
```

**Trade-offs:**
- ✅ Scales reads horizontally
- ✅ Improves read performance
- ❌ Replication lag
- ❌ Increased complexity

### Database Sharding
**When to Use:**
- Large dataset (>1TB)
- High write throughput
- Geographic distribution needed

**Sharding Strategies:**
1. **Range-based:** By date ranges, IDs
2. **Hash-based:** Consistent hashing
3. **Directory-based:** Lookup table

**Trade-offs:**
- ✅ Scales writes horizontally
- ✅ Reduces index size
- ❌ Complex queries across shards
- ❌ Rebalancing complexity

## 🗃️ Caching Patterns

### Cache-Aside (Lazy Loading)
**When to Use:**
- Read-heavy workloads
- Cache miss acceptable
- Simple cache logic preferred

**Implementation:**
```java
@Service
public class CacheAsideService {

    @Autowired
    private Cache<String, Object> cache;

    @Autowired
    private DataRepository repository;

    public Object getData(String key) {
        Object data = cache.get(key);
        if (data == null) {
            data = repository.findByKey(key);
            if (data != null) {
                cache.put(key, data);
            }
        }
        return data;
    }

    public void updateData(String key, Object data) {
        repository.save(key, data);
        cache.put(key, data); // Update cache
    }
}
```

**Trade-offs:**
- ✅ Simple implementation
- ✅ Cache consistency
- ❌ Cache miss penalty
- ❌ Stale data possible

### Write-Through Cache
**When to Use:**
- Strong consistency required
- Write-heavy workloads
- Cache must be up-to-date

**Trade-offs:**
- ✅ Strong consistency
- ✅ No stale data
- ❌ Higher write latency
- ❌ Cache failures affect writes

### Write-Behind (Write-Back) Cache
**When to Use:**
- High write throughput needed
- Temporary inconsistency acceptable
- Batch operations beneficial

**Trade-offs:**
- ✅ High write throughput
- ✅ Asynchronous processing
- ❌ Data loss risk on failure
- ❌ Eventual consistency

## 🔄 Communication Patterns

### Synchronous (Request-Response)
**When to Use:**
- Immediate response needed
- Simple client-server interactions
- Strong consistency required

**Protocols:**
- REST APIs
- GraphQL
- gRPC

**Trade-offs:**
- ✅ Immediate feedback
- ✅ Simple debugging
- ❌ Tight coupling
- ❌ Cascading failures

### Asynchronous (Event-Driven)
**When to Use:**
- Decoupled services
- High throughput needed
- Eventual consistency acceptable

**Patterns:**
- Message queues
- Event sourcing
- CQRS

**Trade-offs:**
- ✅ Loose coupling
- ✅ High scalability
- ❌ Eventual consistency
- ❌ Complex debugging

## 🏗️ Architecture Patterns

### Microservices
**When to Use:**
- Large teams (>10 developers)
- Independent deployments needed
- Technology diversity required
- Domain complexity high

**Trade-offs:**
- ✅ Independent scaling
- ✅ Technology flexibility
- ❌ Distributed complexity
- ❌ Operational overhead

### Monolithic Architecture
**When to Use:**
- Small teams
- Simple domain
- Rapid development needed
- Consistency critical

**Trade-offs:**
- ✅ Simple development
- ✅ Easy testing
- ❌ Scaling challenges
- ❌ Technology lock-in

### Serverless
**When to Use:**
- Variable/unpredictable traffic
- Event-driven workloads
- Cost optimization priority
- Rapid prototyping

**Trade-offs:**
- ✅ Auto-scaling
- ✅ Cost efficiency
- ❌ Cold starts
- ❌ Vendor lock-in

## 📊 Scalability Patterns

### Vertical Scaling
**When to Use:**
- Small to medium applications
- Database bottlenecks
- Quick scaling needed
- Cost not primary concern

**Trade-offs:**
- ✅ Simple implementation
- ✅ No code changes needed
- ❌ Hardware limits
- ❌ Single point of failure

### Horizontal Scaling
**When to Use:**
- Large-scale applications
- High availability required
- Global distribution needed

**Challenges:**
- Data consistency
- Session management
- Load balancing complexity

**Trade-offs:**
- ✅ Unlimited scaling
- ✅ High availability
- ❌ Complex architecture
- ❌ Data consistency challenges

## 🔒 Consistency Patterns

### Strong Consistency
**When to Use:**
- Financial transactions
- Inventory management
- Critical business logic

**Implementation:**
- ACID transactions
- Synchronous replication
- Distributed locks

**Trade-offs:**
- ✅ Data accuracy
- ✅ Simple reasoning
- ❌ Performance impact
- ❌ Availability reduction

### Eventual Consistency
**When to Use:**
- Social media feeds
- Analytics systems
- Content management

**Implementation:**
- Asynchronous replication
- Conflict resolution strategies
- Version vectors

**Trade-offs:**
- ✅ High availability
- ✅ Performance
- ❌ Complex conflict resolution
- ❌ Temporary inconsistency

## 🛡️ Reliability Patterns

### Circuit Breaker
**When to Use:**
- External service dependencies
- Network instability
- Service degradation handling

**States:**
1. **Closed:** Normal operation
2. **Open:** Failing fast
3. **Half-Open:** Testing recovery

**Trade-offs:**
- ✅ Failure isolation
- ✅ Graceful degradation
- ❌ Increased complexity
- ❌ Configuration tuning needed

### Retry with Exponential Backoff
**When to Use:**
- Transient failures
- Network timeouts
- Temporary service unavailability

**Implementation:**
```java
public class RetryService {

    public <T> T executeWithRetry(Supplier<T> operation, int maxRetries) {
        int attempt = 0;
        while (attempt < maxRetries) {
            try {
                return operation.get();
            } catch (Exception e) {
                attempt++;
                if (attempt >= maxRetries) {
                    throw e;
                }
                long delay = (long) (Math.pow(2, attempt) * 100); // Exponential backoff
                Thread.sleep(Math.min(delay, 5000)); // Max 5 seconds
            }
        }
        throw new RuntimeException("Max retries exceeded");
    }
}
```

**Trade-offs:**
- ✅ Handles transient failures
- ✅ Reduces load on failing services
- ❌ Increased latency
- ❌ Thundering herd problem

## 📈 Performance Patterns

### Database Indexing
**When to Use:**
- Slow queries identified
- Large datasets
- Frequent filtering/sorting

**Index Types:**
- B-Tree: Range queries, equality
- Hash: Exact matches only
- Bitmap: Low cardinality columns
- Full-text: Text search

**Trade-offs:**
- ✅ Faster queries
- ✅ Reduced I/O
- ❌ Slower writes
- ❌ Storage overhead

### Connection Pooling
**When to Use:**
- High database traffic
- Connection overhead significant
- Resource constraints

**Benefits:**
- ✅ Reuse connections
- ✅ Reduce overhead
- ✅ Resource management

### CDN (Content Delivery Network)
**When to Use:**
- Global user base
- Static content heavy
- Low latency requirements

**Trade-offs:**
- ✅ Reduced latency
- ✅ Lower origin load
- ❌ Cache invalidation complexity
- ❌ Additional cost

## 🎯 Quick Decision Guide

### Choose Database:
- **Reads > 80%:** Read replicas
- **Writes > 10k/s:** Sharding
- **ACID required:** RDBMS
- **Flexible schema:** NoSQL

### Choose Cache:
- **Consistency critical:** Write-through
- **Performance critical:** Cache-aside
- **Write-heavy:** Write-behind

### Choose Communication:
- **Immediate response:** Synchronous
- **High throughput:** Asynchronous
- **Complex queries:** REST
- **Real-time:** WebSocket

### Choose Architecture:
- **Small team:** Monolithic
- **Large team:** Microservices
- **Variable load:** Serverless
- **Predictable load:** Traditional scaling

---

**Remember: There are no one-size-fits-all solutions. Always consider your specific requirements and constraints!** 🏗️

*Use this cheatsheet as a starting point, then dive deeper into the main repository content for implementation details.*
