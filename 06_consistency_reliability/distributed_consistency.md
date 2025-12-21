# Distributed Consistency

Distributed consistency ensures that all nodes in a distributed system agree on the state of shared data. Achieving consistency in distributed systems is challenging due to network delays, node failures, and concurrent operations. This guide explores different consistency models and their implementation in distributed systems.

## What is Distributed Consistency?

Distributed consistency ensures that all nodes in a distributed system have a consistent view of shared data, even when operations occur concurrently across different nodes.

### Key Concepts
- **Consistency Model**: Defines guarantees about data visibility and ordering
- **Linearizability**: Operations appear to occur in a single, global order
- **Eventual Consistency**: System becomes consistent over time
- **Causal Consistency**: Respects cause-and-effect relationships
- **Quorum**: Minimum number of nodes that must agree

### Consistency vs Availability
- **Strong Consistency**: All reads see latest writes (may sacrifice availability)
- **Weak Consistency**: Reads may see stale data (maintains availability)
- **Eventual Consistency**: System converges to consistent state

## Consistency Models

### Linearizability (Strong Consistency)
Operations appear to occur instantaneously at some point between their start and end times.

**Characteristics:**
- **Real-Time Order**: Operations ordered by real-time occurrence
- **Atomic Visibility**: Either all processes see the operation or none do
- **Immediate Consistency**: No stale reads possible

**Implementation:**
```java
public class LinearizableStore<T> {
    
    private final List<Node> nodes;
    private final AtomicLong timestamp = new AtomicLong(0);
    
    public void write(String key, T value) {
        long writeTimestamp = timestamp.incrementAndGet();
        
        // Send write to all nodes with timestamp
        WriteRequest request = new WriteRequest(key, value, writeTimestamp);
        
        // Wait for majority acknowledgment (quorum)
        List<CompletableFuture<Void>> futures = nodes.stream()
            .map(node -> node.sendWrite(request))
            .collect(Collectors.toList());
        
        // Wait for quorum to respond
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .join();
    }
    
    public T read(String key) {
        long readTimestamp = timestamp.incrementAndGet();
        
        // Read from majority of nodes
        List<CompletableFuture<ReadResponse<T>>> futures = nodes.stream()
            .map(node -> node.sendRead(new ReadRequest(key, readTimestamp)))
            .collect(Collectors.toList());
        
        // Get responses and return latest value
        List<ReadResponse<T>> responses = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        
        return responses.stream()
            .max(Comparator.comparing(ReadResponse::getTimestamp))
            .map(ReadResponse::getValue)
            .orElse(null);
    }
}
```

**Trade-offs:**
- **Latency**: High due to coordination overhead
- **Availability**: Reduced during network partitions
- **Performance**: Coordination limits throughput

### Sequential Consistency
Operations appear in some total order that is consistent with the order seen by each individual process.

**Characteristics:**
- **Per-Process Order**: Preserves order as seen by each process
- **Global Order**: Some total order exists across all operations
- **No Real-Time Guarantee**: No relation to wall-clock time

**Implementation:**
```java
public class SequentialStore<T> {
    
    private final List<Operation<T>> operationLog = Collections.synchronizedList(new ArrayList<>());
    private final Map<String, T> data = new ConcurrentHashMap<>();
    
    public synchronized void write(String key, T value, long clientId, long sequenceNumber) {
        Operation<T> op = new Operation<>(OperationType.WRITE, key, value, clientId, sequenceNumber);
        operationLog.add(op);
        
        // Apply operation in log order
        applyOperations();
    }
    
    public synchronized T read(String key, long clientId, long sequenceNumber) {
        Operation<T> op = new Operation<>(OperationType.READ, key, null, clientId, sequenceNumber);
        operationLog.add(op);
        
        // Apply operations up to this read
        applyOperations();
        
        return data.get(key);
    }
    
    private void applyOperations() {
        // Sort operations by some total order (could be Lamport timestamps)
        operationLog.sort(Comparator.comparing(Operation::getTimestamp));
        
        // Apply operations in order
        Map<String, T> tempData = new HashMap<>(data);
        
        for (Operation<T> op : operationLog) {
            if (op.getType() == OperationType.WRITE) {
                tempData.put(op.getKey(), op.getValue());
            }
        }
        
        data.putAll(tempData);
    }
}
```

**Trade-offs:**
- **Latency**: Moderate coordination required
- **Availability**: Better than linearizability
- **Complexity**: Simpler than linearizability

### Causal Consistency
Operations that are causally related are seen by all processes in the same order.

**Characteristics:**
- **Causal Order**: If A causes B, all processes see A before B
- **Concurrency**: Unrelated operations can be reordered
- **Vector Clocks**: Track causality relationships

**Implementation:**
```java
public class CausalStore<T> {
    
    private final Map<String, VersionVector> versionVectors = new ConcurrentHashMap<>();
    private final Map<String, T> data = new ConcurrentHashMap<>();
    
    public void write(String key, T value, String clientId, VersionVector clientVector) {
        synchronized (this) {
            // Merge version vectors
            VersionVector currentVector = versionVectors.getOrDefault(key, new VersionVector());
            VersionVector newVector = currentVector.merge(clientVector);
            newVector.increment(clientId);
            
            // Update data and version
            data.put(key, value);
            versionVectors.put(key, newVector);
        }
    }
    
    public CausalReadResponse<T> read(String key, String clientId, VersionVector clientVector) {
        synchronized (this) {
            T value = data.get(key);
            VersionVector serverVector = versionVectors.get(key);
            
            // Check if read is causally ready
            if (serverVector != null && !serverVector.happensBefore(clientVector)) {
                // Read is not ready, return current knowledge
                return new CausalReadResponse<>(value, serverVector, false);
            }
            
            return new CausalReadResponse<>(value, serverVector, true);
        }
    }
    
    public static class VersionVector {
        private final Map<String, Long> versions = new ConcurrentHashMap<>();
        
        public void increment(String clientId) {
            versions.merge(clientId, 1L, Long::sum);
        }
        
        public boolean happensBefore(VersionVector other) {
            // Check if this vector happens before other
            return versions.entrySet().stream()
                .allMatch(entry -> 
                    other.versions.getOrDefault(entry.getKey(), 0L) >= entry.getValue()) &&
                other.versions.entrySet().stream()
                    .anyMatch(entry -> 
                        versions.getOrDefault(entry.getKey(), 0L) < entry.getValue());
        }
        
        public VersionVector merge(VersionVector other) {
            VersionVector result = new VersionVector();
            Set<String> allClients = new HashSet<>();
            allClients.addAll(versions.keySet());
            allClients.addAll(other.versions.keySet());
            
            for (String client : allClients) {
                long thisVersion = versions.getOrDefault(client, 0L);
                long otherVersion = other.versions.getOrDefault(client, 0L);
                result.versions.put(client, Math.max(thisVersion, otherVersion));
            }
            
            return result;
        }
    }
}
```

**Trade-offs:**
- **Latency**: Low, minimal coordination
- **Availability**: High, works during partitions
- **Complexity**: High, requires vector clock management

### Eventual Consistency
System will become consistent over time, but not immediately.

**Characteristics:**
- **Convergence**: System eventually reaches consistent state
- **No Ordering Guarantees**: Operations may be seen in different orders
- **Conflict Resolution**: Mechanisms to resolve concurrent updates

**Implementation:**
```java
public class EventualStore<T> {
    
    private final Map<String, T> data = new ConcurrentHashMap<>();
    private final Map<String, Long> versionStamps = new ConcurrentHashMap<>();
    private final ScheduledExecutorService syncExecutor = Executors.newScheduledThreadPool(4);
    
    public void write(String key, T value) {
        // Write locally first
        long version = versionStamps.getOrDefault(key, 0L) + 1;
        data.put(key, value);
        versionStamps.put(key, version);
        
        // Schedule background sync to other nodes
        syncExecutor.schedule(() -> syncToPeers(key, value, version), 100, TimeUnit.MILLISECONDS);
    }
    
    public T read(String key) {
        return data.get(key); // May be stale
    }
    
    private void syncToPeers(String key, T value, long version) {
        // Sync with peer nodes asynchronously
        List<Node> peers = getPeerNodes();
        
        for (Node peer : peers) {
            CompletableFuture.runAsync(() -> {
                try {
                    peer.syncData(key, value, version);
                } catch (Exception e) {
                    // Handle sync failure
                    logger.warn("Failed to sync {} to peer {}", key, peer.getId());
                }
            });
        }
    }
    
    public void receiveSync(String key, T value, long version) {
        Long currentVersion = versionStamps.get(key);
        
        if (currentVersion == null || version > currentVersion) {
            // Accept newer version
            data.put(key, value);
            versionStamps.put(key, version);
        } else if (version == currentVersion) {
            // Same version, resolve conflicts
            resolveConflict(key, value, currentVersion);
        }
        // Ignore older versions
    }
    
    private void resolveConflict(String key, T newValue, long version) {
        // Last-write-wins strategy
        T currentValue = data.get(key);
        
        // Could implement more sophisticated conflict resolution
        if (shouldAcceptNewValue(currentValue, newValue)) {
            data.put(key, newValue);
        }
    }
}
```

**Trade-offs:**
- **Latency**: Very low, no coordination
- **Availability**: Very high
- **Consistency**: Eventual, not immediate

## Consistency Protocols

### Two-Phase Commit (2PC)
Atomic commitment protocol for distributed transactions.

**Phases:**
1. **Prepare Phase**: Coordinator asks participants to prepare
2. **Commit Phase**: Coordinator tells participants to commit or abort

**Implementation:**
```java
public class TwoPhaseCommitCoordinator {
    
    private final List<Participant> participants;
    private TransactionState state = TransactionState.INIT;
    
    public boolean commit(Transaction transaction) {
        try {
            // Phase 1: Prepare
            state = TransactionState.PREPARING;
            boolean canCommit = preparePhase(transaction);
            
            if (!canCommit) {
                abortPhase(transaction);
                return false;
            }
            
            // Phase 2: Commit
            state = TransactionState.COMMITTING;
            commitPhase(transaction);
            
            state = TransactionState.COMMITTED;
            return true;
            
        } catch (Exception e) {
            abortPhase(transaction);
            state = TransactionState.ABORTED;
            return false;
        }
    }
    
    private boolean preparePhase(Transaction transaction) {
        List<CompletableFuture<Boolean>> prepareFutures = participants.stream()
            .map(participant -> participant.prepare(transaction))
            .collect(Collectors.toList());
        
        return prepareFutures.stream()
            .map(CompletableFuture::join)
            .allMatch(Boolean::booleanValue);
    }
    
    private void commitPhase(Transaction transaction) {
        List<CompletableFuture<Void>> commitFutures = participants.stream()
            .map(participant -> participant.commit(transaction))
            .collect(Collectors.toList());
        
        // Wait for all to commit
        commitFutures.forEach(CompletableFuture::join);
    }
    
    private void abortPhase(Transaction transaction) {
        participants.forEach(participant -> 
            participant.abort(transaction).join());
    }
}
```

**Limitations:**
- **Blocking**: Coordinator failure blocks participants
- **Single Point of Failure**: Coordinator is critical
- **Performance**: High latency due to multiple rounds

### Paxos Consensus
Protocol for achieving consensus in distributed systems.

**Roles:**
- **Proposer**: Proposes values to be agreed upon
- **Acceptor**: Vote on proposed values
- **Learner**: Learn the agreed value

**Phases:**
1. **Prepare Phase**: Proposer gets promise from majority
2. **Accept Phase**: Proposer sends accept request
3. **Learn Phase**: Learners receive the accepted value

**Simplified Implementation:**
```java
public class PaxosNode {
    
    private final String nodeId;
    private final List<PaxosNode> peers;
    
    private long currentRound = 0;
    private Object acceptedValue;
    private long acceptedRound = -1;
    
    public synchronized ConsensusResult propose(Object value) {
        long round = ++currentRound;
        
        // Phase 1: Prepare
        PrepareRequest prepareReq = new PrepareRequest(round);
        List<PrepareResponse> prepareResponses = sendToMajority(prepareReq);
        
        // Check if we got majority promise
        if (prepareResponses.size() < getMajoritySize()) {
            return ConsensusResult.REJECTED;
        }
        
        // Find highest accepted value
        PrepareResponse highestResponse = prepareResponses.stream()
            .max(Comparator.comparing(PrepareResponse::getAcceptedRound))
            .orElse(null);
        
        Object valueToPropose = value;
        if (highestResponse != null && highestResponse.getAcceptedValue() != null) {
            valueToPropose = highestResponse.getAcceptedValue();
        }
        
        // Phase 2: Accept
        AcceptRequest acceptReq = new AcceptRequest(round, valueToPropose);
        List<AcceptResponse> acceptResponses = sendToMajority(acceptReq);
        
        if (acceptResponses.size() >= getMajoritySize()) {
            // Phase 3: Learn
            sendToAll(new LearnRequest(round, valueToPropose));
            return ConsensusResult.ACCEPTED;
        }
        
        return ConsensusResult.REJECTED;
    }
    
    public synchronized void receivePrepare(PrepareRequest request) {
        if (request.getRound() > currentRound) {
            currentRound = request.getRound();
            // Send promise with highest accepted value
            sendResponse(new PrepareResponse(acceptedRound, acceptedValue));
        }
    }
    
    public synchronized void receiveAccept(AcceptRequest request) {
        if (request.getRound() >= currentRound) {
            acceptedValue = request.getValue();
            acceptedRound = request.getRound();
            // Send accept acknowledgment
            sendResponse(new AcceptResponse(true));
        }
    }
    
    private int getMajoritySize() {
        return (peers.size() + 1) / 2 + 1;
    }
}
```

## Conflict Resolution Strategies

### Last-Write-Wins (LWW)
Resolve conflicts by accepting the most recent write.

```java
public class LastWriteWinsResolver<T> {
    
    public T resolve(List<VersionedValue<T>> conflictingValues) {
        return conflictingValues.stream()
            .max(Comparator.comparing(VersionedValue::getTimestamp))
            .map(VersionedValue::getValue)
            .orElse(null);
    }
}
```

### Merge Functions
Application-specific conflict resolution.

```java
public class ShoppingCartMerger {
    
    public ShoppingCart merge(ShoppingCart local, ShoppingCart remote) {
        ShoppingCart merged = new ShoppingCart();
        
        // Merge items with conflict resolution
        Map<String, CartItem> allItems = new HashMap<>();
        
        // Add local items
        local.getItems().forEach(item -> 
            allItems.put(item.getProductId(), item));
        
        // Merge remote items
        remote.getItems().forEach(remoteItem -> {
            CartItem localItem = allItems.get(remoteItem.getProductId());
            if (localItem != null) {
                // Conflict: choose higher quantity or merge
                int mergedQuantity = Math.max(localItem.getQuantity(), remoteItem.getQuantity());
                allItems.put(remoteItem.getProductId(), 
                    new CartItem(remoteItem.getProductId(), mergedQuantity));
            } else {
                allItems.put(remoteItem.getProductId(), remoteItem);
            }
        });
        
        merged.setItems(new ArrayList<>(allItems.values()));
        return merged;
    }
}
```

### Operational Transformation
Real-time collaborative editing conflict resolution.

```java
public class OperationalTransformation {
    
    public List<Operation> transform(List<Operation> operations, Operation newOperation) {
        List<Operation> transformed = new ArrayList<>();
        
        for (Operation existing : operations) {
            // Transform new operation against existing
            Operation transformedNew = transformOperation(newOperation, existing);
            
            // Transform existing operation against new
            Operation transformedExisting = transformOperation(existing, newOperation);
            
            transformed.add(transformedExisting);
            newOperation = transformedNew;
        }
        
        transformed.add(newOperation);
        return transformed;
    }
    
    private Operation transformOperation(Operation op1, Operation op2) {
        // Implement transformation logic based on operation types
        // For example: insert vs insert, delete vs insert, etc.
        return op1; // Simplified
    }
}
```

## Consistency in Practice

### Tunable Consistency
Allow applications to choose consistency level per operation.

```java
public class TunableConsistencyStore<T> {
    
    public enum ConsistencyLevel {
        STRONG,  // Linearizable
        WEAK,    // Eventual
        CAUSAL   // Causal consistency
    }
    
    public void write(String key, T value, ConsistencyLevel level) {
        switch (level) {
            case STRONG:
                writeStrong(key, value);
                break;
            case WEAK:
                writeWeak(key, value);
                break;
            case CAUSAL:
                writeCausal(key, value);
                break;
        }
    }
    
    public T read(String key, ConsistencyLevel level) {
        switch (level) {
            case STRONG:
                return readStrong(key);
            case WEAK:
                return readWeak(key);
            case CAUSAL:
                return readCausal(key);
            default:
                return readWeak(key);
        }
    }
    
    private void writeStrong(String key, T value) {
        // Implement strong consistency write
        // Wait for majority acknowledgment
    }
    
    private void writeWeak(String key, T value) {
        // Implement weak consistency write
        // Write locally, sync asynchronously
    }
    
    private void writeCausal(String key, T value) {
        // Implement causal consistency write
        // Use vector clocks
    }
}
```

### Consistency Monitoring
Track consistency violations and performance.

```java
@Service
public class ConsistencyMonitor {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter consistencyViolations = Counter.builder("consistency_violations_total")
        .description("Total consistency violations")
        .register(meterRegistry);
    
    private final Histogram staleness = Histogram.builder("data_staleness_seconds")
        .description("Data staleness duration")
        .register(meterRegistry);
    
    private final Gauge partitionCount = Gauge.builder("network_partitions_active")
        .description("Active network partitions")
        .register(meterRegistry);
    
    public void recordConsistencyViolation(String operation, long stalenessMs) {
        consistencyViolations.increment();
        staleness.observe(stalenessMs / 1000.0);
        
        logger.warn("Consistency violation in {}: {}ms stale", operation, stalenessMs);
    }
    
    public void recordPartitionDetected() {
        // Increment partition counter
        logger.warn("Network partition detected");
    }
    
    public void recordPartitionResolved() {
        // Decrement partition counter
        logger.info("Network partition resolved");
    }
}
```

## Choosing Consistency Models

### Business Requirements Analysis
```java
public class ConsistencyRequirementsAnalyzer {
    
    public ConsistencyLevel recommendLevel(SystemRequirements reqs) {
        
        if (reqs.isFinancialSystem()) {
            return ConsistencyLevel.STRONG;
        }
        
        if (reqs.isRealTimeCollaboration()) {
            return ConsistencyLevel.CAUSAL;
        }
        
        if (reqs.isHighThroughputLogging()) {
            return ConsistencyLevel.WEAK;
        }
        
        if (reqs.isGlobalEcommerce()) {
            return ConsistencyLevel.EVENTUAL;
        }
        
        // Default to eventual consistency for web applications
        return ConsistencyLevel.EVENTUAL;
    }
}
```

### Performance vs Consistency Trade-offs

| Consistency Level | Read Latency | Write Latency | Availability | Complexity |
|------------------|--------------|---------------|--------------|------------|
| **Strong** | High | High | Low | High |
| **Sequential** | Medium | Medium | Medium | Medium |
| **Causal** | Low | Low | High | High |
| **Eventual** | Low | Low | High | Low |

## Real-World Examples

### Banking System (Strong Consistency)
```java
@Service
@Transactional(isolation = Isolation.SERIALIZABLE)
public class BankingService {
    
    public void transferMoney(AccountTransfer transfer) {
        // Lock both accounts for strong consistency
        Account fromAccount = accountRepository.findByIdWithLock(transfer.getFromAccountId());
        Account toAccount = accountRepository.findByIdWithLock(transfer.getToAccountId());
        
        if (fromAccount.getBalance().compareTo(transfer.getAmount()) < 0) {
            throw new InsufficientFundsException();
        }
        
        // Atomic transfer
        fromAccount.setBalance(fromAccount.getBalance().subtract(transfer.getAmount()));
        toAccount.setBalance(toAccount.getBalance().add(transfer.getAmount()));
        
        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);
        
        // Strong consistency guaranteed
    }
}
```

### Social Media Timeline (Causal Consistency)
```java
@Service
public class TimelineService {
    
    public void postToTimeline(String userId, Post post) {
        // Write post with causal metadata
        VersionVector userVector = getUserVersionVector(userId);
        userVector.increment(userId);
        
        PostWithMetadata postWithMetadata = new PostWithMetadata(post, userVector);
        timelineRepository.save(postWithMetadata);
        
        // Update follower timelines causally
        List<String> followers = getFollowers(userId);
        followers.forEach(followerId -> 
            updateFollowerTimelineCausally(followerId, postWithMetadata));
    }
    
    private void updateFollowerTimelineCausally(String followerId, PostWithMetadata post) {
        // Ensure causal ordering for each follower
        VersionVector followerVector = getUserVersionVector(followerId);
        
        if (post.getVersionVector().happensBefore(followerVector)) {
            // Can add to timeline causally
            followerTimelineRepository.addPost(followerId, post);
        } else {
            // Queue for later when causal dependencies are satisfied
            causalQueue.add(new CausalTimelineUpdate(followerId, post));
        }
    }
}
```

### IoT Sensor Network (Eventual Consistency)
```java
@Service
public class SensorDataAggregator {
    
    @Async
    public void processSensorReading(SensorReading reading) {
        // Write locally first (eventual consistency)
        localSensorRepository.save(reading);
        
        // Queue for async replication to central system
        asyncReplicationQueue.add(reading);
        
        // Trigger local analytics
        localAnalyticsEngine.processReading(reading);
    }
    
    @Scheduled(fixedDelay = 30000) // Every 30 seconds
    public void replicateToCentral() {
        List<SensorReading> batch = new ArrayList<>();
        asyncReplicationQueue.drainTo(batch, 100);
        
        if (!batch.isEmpty()) {
            try {
                centralSensorRepository.saveAll(batch);
            } catch (Exception e) {
                // Re-queue failed items
                asyncReplicationQueue.addAll(batch);
                logger.warn("Failed to replicate {} sensor readings", batch.size());
            }
        }
    }
}
```

## Conclusion

Distributed consistency is a complex but essential aspect of building reliable distributed systems. Different applications require different consistency guarantees based on their business requirements.

**Key Takeaways:**
- **Consistency Spectrum**: From strong to eventual consistency
- **Trade-off Analysis**: Balance consistency with performance and availability
- **Context Matters**: Different data may require different consistency levels
- **Conflict Resolution**: Plan for handling concurrent updates
- **Monitoring**: Track consistency violations and performance

Understanding consistency models enables designing systems that meet business requirements while maintaining acceptable performance and availability levels.
