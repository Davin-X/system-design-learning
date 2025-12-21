# CAP Theorem Guide

Quick reference for understanding the CAP theorem, consistency models, and trade-offs in distributed systems.

## 📋 CAP Theorem Fundamentals

### The Theorem
**CAP Theorem:** In a distributed system, you can only guarantee **2 out of 3** properties simultaneously:
- **Consistency (C)**: All nodes see the same data at the same time
- **Availability (A)**: Every request receives a response (success/failure)
- **Partition Tolerance (P)**: System continues to operate despite network failures

### Visual Representation
```
CAP Theorem Triangle:

     Consistency (C)
          /\
         /  \
        /    \
   (A) /      \ (P)
      /        \
     /          \
Availability    Partition Tolerance
```

**Key Insight:** In real distributed systems, network partitions are inevitable, so you must choose between Consistency and Availability.

## 🎯 Consistency Models

### Strong Consistency (CP Systems)
**When to Use:**
- Financial transactions
- Banking systems
- Inventory management
- Any system where data accuracy is critical

**Examples:**
- Relational databases with ACID transactions
- Distributed databases with synchronous replication
- Systems using Paxos/Raft consensus

**Trade-offs:**
- ✅ Data accuracy guaranteed
- ✅ Simple application logic
- ❌ Higher latency
- ❌ Reduced availability during partitions

### Eventual Consistency (AP Systems)
**When to Use:**
- Social media feeds
- Content delivery
- Analytics systems
- User preferences

**Examples:**
- DNS systems
- Cassandra (with tunable consistency)
- CouchDB
- Most NoSQL databases

**Trade-offs:**
- ✅ High availability
- ✅ Low latency
- ❌ Temporary inconsistency
- ❌ Complex conflict resolution

## 🔧 Practical Application

### Database Choices by CAP

| Database | CAP Choice | Use Case |
|----------|------------|----------|
| **PostgreSQL** | CA | Financial systems, ERP |
| **MySQL** | CA | Traditional web apps |
| **MongoDB** | CP/AP* | Content management |
| **Cassandra** | AP | Time-series, IoT |
| **Redis** | CP/AP* | Caching, sessions |
| **DynamoDB** | AP | Serverless apps |

*Configurable consistency levels

### Real-World Examples

#### CP Systems (Consistency + Partition Tolerance)
```java
// Synchronous replication example
@Service
public class BankingService {

    @Transactional
    public void transferMoney(Account from, Account to, BigDecimal amount) {
        // All operations in single transaction
        from.debit(amount);
        to.credit(amount);

        // If any step fails, entire transaction rolls back
        // Strong consistency guaranteed
    }
}
```

- **Banking:** Double-entry bookkeeping requires consistency
- **E-commerce:** Inventory must be accurate
- **Healthcare:** Patient records cannot be inconsistent

#### AP Systems (Availability + Partition Tolerance)
```java
// Eventual consistency example
@Service
public class SocialMediaService {

    public void likePost(String postId, String userId) {
        // Immediate response - don't wait for consistency
        likeQueue.send(new LikeEvent(postId, userId));

        // Background process handles consistency
        // User sees like immediately, eventual consistency
    }
}
```

- **Twitter:** Tweet counts eventually consistent
- **Instagram:** Likes and comments eventually consistent
- **YouTube:** View counts eventually consistent

## 📊 Consistency Levels (in Practice)

### In Cassandra
```sql
-- Different consistency levels
SELECT * FROM users WHERE id = 1;

-- Strong consistency (synchronous)
CONSISTENCY QUORUM;

-- Eventual consistency (asynchronous)
CONSISTENCY ONE;

-- Tunable consistency
CONSISTENCY LOCAL_QUORUM;
```

### In DynamoDB
```java
// Different consistency models
GetItemRequest request = new GetItemRequest()
    .withTableName("users")
    .withKey(key);

request.setConsistentRead(true);   // Strong consistency
request.setConsistentRead(false);  // Eventual consistency (default)
```

## ⚖️ Trade-off Decision Framework

### Questions to Ask:
1. **How critical is data accuracy?**
   - If incorrect data causes financial/business loss → **CP**
   - If temporary inconsistency is acceptable → **AP**

2. **What's the cost of downtime?**
   - If downtime costs > $100k/hour → **AP**
   - If consistency violations cost more → **CP**

3. **How large is your dataset?**
   - Small datasets (< 100GB) → Can often do **CA**
   - Large datasets (> 1TB) → Must handle partitions → **CP or AP**

4. **What's your network reliability?**
   - Reliable network → Can consider **CA**
   - Unreliable network → Must choose **CP or AP**

### Decision Tree:
```
Need Partition Tolerance? (Always YES for distributed systems)
├── YES
│   ├── Data accuracy critical?
│   │   ├── YES → CP (Consistency + Partition Tolerance)
│   │   └── NO → AP (Availability + Partition Tolerance)
│   └── Can tolerate downtime?
│       ├── YES → CP
│       └── NO → AP
└── NO (Rare - single datacenter only)
    └── CA (Consistency + Availability)
```

## 🔄 Consistency Patterns

### Read Repair
**When to Use:** AP systems that need eventual consistency
```java
// Read repair pattern
public User getUser(String userId) {
    // Read from multiple replicas
    List<User> versions = readFromAllReplicas(userId);

    if (versionsHaveConflicts(versions)) {
        User resolved = resolveConflicts(versions);
        writeToAllReplicas(userId, resolved); // Repair during read
        return resolved;
    }

    return versions.get(0);
}
```

### Anti-Entropy (Background Repair)
**When to Use:** Large-scale systems needing automatic repair
```java
// Merkle tree comparison for repair
public void repairData() {
    // Compare hash trees between replicas
    MerkleTree localTree = buildMerkleTree(localData);
    MerkleTree remoteTree = getRemoteMerkleTree();

    List<String> differences = compareTrees(localTree, remoteTree);

    // Repair only differing data
    for (String key : differences) {
        syncData(key);
    }
}
```

### Version Vectors (Conflict Resolution)
**When to Use:** Multiple writers, complex conflict resolution
```java
public class VersionVector {
    private Map<String, Long> versions = new HashMap<>();

    public boolean isConcurrent(VersionVector other) {
        boolean greater = false;
        boolean less = false;

        for (String node : union(keySet(), other.keySet())) {
            long v1 = getOrDefault(node, 0L);
            long v2 = other.getOrDefault(node, 0L);

            if (v1 > v2) greater = true;
            if (v1 < v2) less = true;
        }

        return greater && less; // Concurrent if both greater and less
    }
}
```

## 🏢 System Design Implications

### CP System Design
- Use synchronous replication
- Implement distributed locks
- Expect higher latency
- Design for failure scenarios
- Use consensus algorithms (Paxos, Raft)

### AP System Design
- Use asynchronous replication
- Implement conflict resolution
- Design for eventual consistency
- Use CRDTs (Conflict-free Replicated Data Types)
- Implement read repair mechanisms

## 📈 Monitoring & Metrics

### Key Metrics for CAP Systems

#### For CP Systems:
- **Commit Latency:** Time for operations to complete
- **Failure Rate:** Percentage of failed operations during partitions
- **Data Consistency:** Measure of data accuracy across replicas

#### For AP Systems:
- **Replication Lag:** Time for data to propagate
- **Conflict Rate:** Frequency of data conflicts
- **Repair Time:** Time to resolve inconsistencies
- **Staleness:** How outdated data can become

### Alerting Thresholds:
```yaml
# CP System Alerts
cp_commit_latency_p95: 500ms  # Too slow
cp_failure_rate: 1%          # Too many failures
cp_consistency_violations: 0  # Must be zero

# AP System Alerts
ap_replication_lag: 30s      # Too stale
ap_conflict_rate: 5%         # Too many conflicts
ap_repair_time: 5m          # Taking too long
```

## 🎯 Quick Reference

### Choose CP When:
- ✅ Financial data
- ✅ Inventory systems
- ✅ Legal documents
- ✅ Healthcare records
- ✅ Any "money matters"

### Choose AP When:
- ✅ Social media
- ✅ Content delivery
- ✅ Analytics
- ✅ User preferences
- ✅ High-scale reads

### Hybrid Approaches:
- **CQRS:** Separate read/write models
- **Saga Pattern:** Distributed transactions
- **Event Sourcing:** Audit trail with eventual consistency

## 🔗 Related Concepts

- **ACID vs BASE:** Transaction models
- **Paxos/Raft:** Consensus algorithms for CP
- **CRDTs:** Data types for AP systems
- **Eventual Consistency:** AP implementation patterns

---

**CAP Theorem is not an excuse for poor design - it's a guide for making informed trade-offs.** ⚖️

*Understand your business requirements first, then choose the appropriate CAP properties for your system.*

**Remember:** Most real systems use hybrid approaches, applying different CAP properties to different parts of the system based on their specific requirements.
