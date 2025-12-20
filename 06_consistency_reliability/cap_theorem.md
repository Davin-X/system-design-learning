# CAP Theorem

The CAP theorem, also known as Brewer's theorem, states that in a distributed computer system, you cannot simultaneously guarantee all three of the following properties:

- **Consistency**: All nodes see the same data simultaneously
- **Availability**: The system remains operational and responsive despite failures
- **Partition Tolerance**: The system continues to operate despite network partitions

## Understanding the Properties

### Consistency (C)
Every read receives the most recent write or an error. In a consistent system:

- All nodes have identical data at any given time
- Strong consistency requires synchronous replication
- Updates are atomic across all nodes
- No stale data is served to clients

**Example**: Banking system where account balance must be accurate across all views

### Availability (A)
Every request receives a response, even if it's not the most recent data. In a highly available system:

- System remains operational during failures
- May serve potentially stale data during partitions
- No single point of failure
- Graceful degradation under load

**Example**: Social media feed that shows slightly old posts rather than failing

### Partition Tolerance (P)
System continues to operate despite network partitions (message loss between nodes). In partition-tolerant systems:

- Nodes can operate independently
- Asynchronous communication
- Eventual consistency models
- Survives network failures

**Example**: Distributed database that works even when servers can't communicate

## CAP Theorem Implications

### The Trade-off
You can only achieve **2 out of 3** properties simultaneously:

- **CP Systems**: Consistency + Partition Tolerance (may sacrifice availability)
- **AP Systems**: Availability + Partition Tolerance (may sacrifice consistency)
- **CA Systems**: Consistency + Availability (rare, assumes no partitions)

### Real-World Examples

#### CP Systems (Prioritize Consistency)
- **Banking/Financial Systems**: Account balances must be accurate
- **E-commerce Inventory**: Stock levels must be precise
- **Databases**: MongoDB, HBase, Redis (with strong consistency)
- **Trading Platforms**: Real-time stock prices must be consistent

**Behavior During Partition:**
- System may become unavailable to maintain consistency
- Clients get errors rather than stale data
- "Fail closed" approach

#### AP Systems (Prioritize Availability)
- **Social Media**: Facebook, Twitter, Instagram
- **Content Delivery**: Netflix, YouTube
- **E-commerce**: Amazon product listings
- **Databases**: Cassandra, DynamoDB, CouchDB

**Behavior During Partition:**
- System stays available but may serve stale data
- Eventual consistency - data becomes consistent over time
- "Fail open" approach

#### CA Systems (Theoretical)
- **Single-node databases**: No network, no partitions
- **Tightly coupled systems**: All components in same data center
- **In-memory single instance**: Redis single instance

**Rare in distributed systems** due to network unreliability

## CAP in Practice

### Hybrid Approaches
Most real systems use **hybrid consistency models**:

- **Strong Consistency**: For critical operations (payments, inventory)
- **Eventual Consistency**: For less critical data (user preferences, views)
- **Causal Consistency**: Related operations are consistent

### Consistency Spectrum

1. **Strong Consistency**: Immediate consistency across all nodes
2. **Read-your-writes**: User sees their own writes immediately
3. **Session Consistency**: Consistent within a user session
4. **Monotonic Read Consistency**: Reads never go backwards
5. **Eventual Consistency**: Data becomes consistent over time

### Choosing CAP Properties

#### Business Requirements Drive Choice

**Choose CP when:**
- Data accuracy is critical (financial, healthcare)
- Inconsistent data causes significant harm
- Business can tolerate temporary unavailability
- Examples: Banking, stock trading, medical records

**Choose AP when:**
- System must always be available
- Slight data inconsistency is acceptable
- Business impact of downtime is high
- Examples: Social media, retail, entertainment

#### Technical Constraints

**Network Reliability:**
- Reliable networks → Can achieve CA
- Unreliable networks → Must choose between CP/AP

**Data Types:**
- Transactional data → CP
- Analytical data → AP
- User-generated content → AP

## CAP Theorem Misconceptions

### Common Misunderstandings

1. **"CA systems don't exist"**
   - CA is possible in non-distributed or highly reliable networks
   - Single datacenter with reliable network can achieve CA

2. **"You must choose CP or AP forever"**
   - Different parts of system can have different CAP choices
   - Hybrid consistency models are common

3. **"CAP means you can't have all three"**
   - Correct, but you can have different levels of each property
   - Consistency is a spectrum, not binary

4. **"Availability means 100% uptime"**
   - Availability is about responsiveness, not perfection
   - Systems can be available but serve errors or stale data

### CAP vs ACID

**ACID** (Database transactions):
- **Atomicity**: All or nothing
- **Consistency**: Database constraints maintained
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed transactions survive failures

**Relationship to CAP:**
- ACID focuses on single database consistency
- CAP addresses distributed system trade-offs
- Both deal with consistency but at different scopes

## Implementing CAP Choices

### Building CP Systems
- **Synchronous replication**: Wait for all nodes to acknowledge
- **Paxos/Raft consensus**: Distributed agreement protocols
- **Two-phase commit**: Coordinated transaction commits
- **Quorum-based reads/writes**: Majority must agree

### Building AP Systems
- **Asynchronous replication**: Don't wait for acknowledgments
- **Eventual consistency**: Conflict resolution later
- **Multi-version concurrency control**: Handle conflicts gracefully
- **Optimistic concurrency**: Assume no conflicts, resolve if they occur

### Monitoring CAP Properties
- **Consistency checks**: Compare data across nodes
- **Availability metrics**: Response time, error rates
- **Partition detection**: Network monitoring, heartbeat checks

## CAP Theorem Evolution

### Beyond CAP
- **PACELC Theorem**: If Partitioned, choose Availability or Consistency; Else choose Latency or Consistency
- **CALM Theorem**: Consistency As Logical Monotonicity
- **HARLAN Theorem**: No system can guarantee Availability, Reliability, and Security simultaneously

### Modern Interpretations
- **Consistency is a spectrum**: Different levels for different needs
- **CAP is about trade-offs**: Not absolute choices
- **Hybrid approaches**: Different CAP for different data types
- **Context matters**: Business requirements drive technical choices

## Real-World Examples

### Netflix (AP)
- Prioritizes availability over consistency
- Shows cached/stale data during outages
- Eventual consistency for user preferences
- Strong consistency only for billing

### Banking Systems (CP)
- Account balances must be consistent
- Temporary unavailability acceptable
- Strong consistency for transactions
- Synchronous replication across data centers

### Amazon (AP with CP pockets)
- Product catalog uses eventual consistency
- Shopping cart uses strong consistency
- Order processing uses distributed transactions
- Hybrid approach based on data importance

## Conclusion

CAP theorem is fundamental to understanding distributed systems. It forces you to make explicit trade-offs based on business requirements:

- **Financial systems**: Choose CP for data accuracy
- **Social platforms**: Choose AP for user experience
- **Most systems**: Use hybrid approaches with different CAP properties for different data types

Understanding CAP helps you design systems that align with business needs while being realistic about technical limitations.
