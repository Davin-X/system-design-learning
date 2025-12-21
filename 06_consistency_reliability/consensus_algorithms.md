# Consensus Algorithms

Consensus algorithms enable distributed systems to agree on a single value or decision despite failures and network issues. They are fundamental to building reliable distributed systems that can maintain consistency and make coordinated decisions. This guide covers the major consensus algorithms and their implementations.

## What is Consensus?

Consensus is the process by which a group of distributed processes agree on a single value or decision, even in the presence of failures and network partitions.

### Key Properties
- **Agreement**: All non-faulty processes agree on the same value
- **Validity**: The agreed value was proposed by some process
- **Termination**: All non-faulty processes eventually decide on a value
- **Integrity**: Each process decides at most once

### FLP Impossibility
The FLP impossibility result proves that no deterministic consensus algorithm can guarantee termination in asynchronous networks with even a single faulty process.

## Paxos Algorithm

### Overview
Paxos is a consensus algorithm that ensures agreement among a group of processes, even when some processes fail.

### Roles in Paxos
- **Proposer**: Proposes values to be agreed upon
- **Acceptor**: Vote on proposed values and store decisions
- **Learner**: Learn the agreed value from acceptors
- **Client**: Initiates consensus by contacting proposers

### Paxos Phases

#### Phase 1: Prepare
Proposer selects a unique proposal number and sends prepare requests to majority of acceptors.

```java
public class PaxosProposer {
    
    private final String proposerId;
    private final List<PaxosAcceptor> acceptors;
    private long proposalNumber = 0;
    
    public ConsensusResult propose(Object value) {
        // Increment proposal number
        proposalNumber = generateUniqueProposalNumber();
        
        // Phase 1: Send prepare requests
        PrepareRequest prepareReq = new PrepareRequest(proposalNumber);
        List<PrepareResponse> responses = sendToMajority(prepareReq);
        
        if (responses.size() < getMajoritySize()) {
            return ConsensusResult.REJECTED; // Not enough responses
        }
        
        // Find the highest numbered proposal among responses
        PrepareResponse highestResponse = responses.stream()
            .filter(r -> r.getAcceptedValue() != null)
            .max(Comparator.comparing(PrepareResponse::getAcceptedProposal))
            .orElse(null);
        
        // Choose value to propose
        Object valueToPropose = value;
        if (highestResponse != null) {
            valueToPropose = highestResponse.getAcceptedValue();
        }
        
        // Phase 2: Send accept requests
        AcceptRequest acceptReq = new AcceptRequest(proposalNumber, valueToPropose);
        List<AcceptResponse> acceptResponses = sendToMajority(acceptReq);
        
        if (acceptResponses.size() >= getMajoritySize()) {
            // Success! Send learn requests to all learners
            sendToAllLearners(new LearnRequest(proposalNumber, valueToPropose));
            return ConsensusResult.ACCEPTED;
        }
        
        return ConsensusResult.REJECTED;
    }
    
    private long generateUniqueProposalNumber() {
        // Combine timestamp, proposer ID, and counter
        return (System.currentTimeMillis() << 32) | 
               (Integer.parseInt(proposerId) << 16) | 
               (++proposalNumber & 0xFFFF);
    }
    
    private int getMajoritySize() {
        return (acceptors.size() / 2) + 1;
    }
}
```

#### Phase 2: Accept
Proposer sends accept requests with the chosen value to majority of acceptors.

#### Phase 3: Learn
Once a majority accepts, the value is learned by all processes.

### Paxos Acceptor Implementation
```java
public class PaxosAcceptor {
    
    private long highestPromisedProposal = -1;
    private long highestAcceptedProposal = -1;
    private Object acceptedValue = null;
    
    public synchronized PrepareResponse receivePrepare(PrepareRequest request) {
        if (request.getProposalNumber() > highestPromisedProposal) {
            highestPromisedProposal = request.getProposalNumber();
            return new PrepareResponse(true, highestAcceptedProposal, acceptedValue);
        }
        return new PrepareResponse(false, -1, null);
    }
    
    public synchronized AcceptResponse receiveAccept(AcceptRequest request) {
        if (request.getProposalNumber() >= highestPromisedProposal) {
            highestAcceptedProposal = request.getProposalNumber();
            acceptedValue = request.getValue();
            return new AcceptResponse(true);
        }
        return new AcceptResponse(false);
    }
}
```

## Raft Algorithm

### Overview
Raft is a consensus algorithm designed to be easier to understand and implement than Paxos, while providing the same guarantees.

### Raft Roles
- **Leader**: Handles client requests and replicates log entries
- **Follower**: Passive participants that respond to leader requests
- **Candidate**: Contest leadership during elections

### Raft Algorithm Phases

#### 1. Leader Election
When no leader exists or current leader fails, followers become candidates and start election.

```java
public class RaftNode {
    
    private RaftState state = RaftState.FOLLOWER;
    private int currentTerm = 0;
    private String votedFor = null;
    private final List<RaftNode> peers;
    private final String nodeId;
    
    // Election timeout handling
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    private ScheduledFuture<?> electionTimeout;
    
    public void startElection() {
        synchronized (this) {
            if (state == RaftState.LEADER) return;
            
            state = RaftState.CANDIDATE;
            currentTerm++;
            votedFor = nodeId;
            int votes = 1; // Vote for self
            
            // Send vote requests to all peers
            List<CompletableFuture<VoteResponse>> voteFutures = peers.stream()
                .map(peer -> peer.requestVote(new VoteRequest(currentTerm, nodeId)))
                .collect(Collectors.toList());
            
            // Count votes
            for (CompletableFuture<VoteResponse> future : voteFutures) {
                try {
                    VoteResponse response = future.get(5, TimeUnit.SECONDS);
                    if (response.isGranted()) {
                        votes++;
                    }
                } catch (Exception e) {
                    // Vote request failed
                }
            }
            
            if (votes > peers.size() / 2) {
                // Won election
                becomeLeader();
            } else {
                // Lost election, become follower
                becomeFollower();
            }
        }
    }
    
    private void becomeLeader() {
        state = RaftState.LEADER;
        // Start heartbeat timer
        startHeartbeatTimer();
        logger.info("Became leader for term {}", currentTerm);
    }
    
    private void becomeFollower() {
        state = RaftState.FOLLOWER;
        votedFor = null;
        resetElectionTimeout();
    }
}
```

#### 2. Log Replication
Leader accepts client requests, appends to log, and replicates to followers.

```java
public class RaftLeader extends RaftNode {
    
    private final List<LogEntry> log = new ArrayList<>();
    private final Map<String, Integer> nextIndex = new HashMap<>();
    private final Map<String, Integer> matchIndex = new HashMap<>();
    
    public void appendEntry(LogEntry entry) {
        // Append to leader's log
        log.add(entry);
        
        // Replicate to followers
        replicateToFollowers();
    }
    
    private void replicateToFollowers() {
        for (String peerId : peers.keySet()) {
            int nextIdx = nextIndex.get(peerId);
            
            if (nextIdx <= log.size()) {
                List<LogEntry> entriesToSend = log.subList(nextIdx - 1, log.size());
                AppendEntriesRequest request = new AppendEntriesRequest(
                    currentTerm, nodeId, nextIdx - 1, 
                    log.get(nextIdx - 2).getTerm(), entriesToSend);
                
                peers.get(peerId).appendEntries(request)
                    .thenAccept(response -> {
                        if (response.isSuccess()) {
                            nextIndex.put(peerId, nextIdx + entriesToSend.size());
                            matchIndex.put(peerId, nextIdx + entriesToSend.size() - 1);
                        } else {
                            // Decrement nextIndex and retry
                            nextIndex.put(peerId, Math.max(1, nextIdx - 1));
                        }
                    });
            }
        }
    }
    
    public boolean isCommitted(int index) {
        // Check if majority of peers have replicated this entry
        int replicatedCount = 1; // Leader has it
        
        for (String peerId : matchIndex.keySet()) {
            if (matchIndex.get(peerId) >= index) {
                replicatedCount++;
            }
        }
        
        return replicatedCount > peers.size() / 2;
    }
}
```

#### 3. Safety
Raft ensures that committed entries are never lost and followers stay consistent.

## Zab (ZooKeeper Atomic Broadcast)

### Overview
Zab is the consensus protocol used by Apache ZooKeeper for atomic broadcast and leader election.

### Zab Phases
1. **Discovery**: Leaders are elected and followers learn about the latest epoch
2. **Synchronization**: Leader synchronizes followers with the latest state
3. **Broadcast**: Leader broadcasts updates using atomic broadcast

### Implementation
```java
public class ZabNode {
    
    private ZabState state = ZabState.FOLLOWING;
    private long epoch = 0;
    private final List<ZabNode> peers;
    
    public void startElection() {
        // Simplified leader election
        long proposedEpoch = epoch + 1;
        
        List<CompletableFuture<ElectionResponse>> responses = peers.stream()
            .map(peer -> peer.proposeLeader(new ElectionRequest(nodeId, proposedEpoch)))
            .collect(Collectors.toList());
        
        long maxEpoch = responses.stream()
            .mapToLong(response -> response.join().getEpoch())
            .max()
            .orElse(proposedEpoch);
        
        if (maxEpoch == proposedEpoch) {
            becomeLeader();
        }
    }
    
    public void broadcast(Transaction transaction) {
        if (state != ZabState.LEADING) return;
        
        // Assign transaction ID (zxid)
        long zxid = generateZxid();
        transaction.setZxid(zxid);
        
        // Send to all followers
        List<CompletableFuture<Acknowledgment>> acks = peers.stream()
            .map(peer -> peer.sendTransaction(transaction))
            .collect(Collectors.toList());
        
        // Wait for majority acknowledgment
        long ackCount = acks.stream()
            .mapToLong(future -> future.join().getZxid())
            .filter(ackZxid -> ackZxid == zxid)
            .count();
        
        if (ackCount >= getMajoritySize()) {
            // Transaction committed
            commitTransaction(transaction);
        }
    }
    
    private void becomeLeader() {
        state = ZabState.LEADING;
        epoch++;
        // Start synchronization phase
        synchronizeFollowers();
    }
}
```

## Viewstamped Replication (VR)

### Overview
VR is a state machine replication protocol that provides strong consistency guarantees.

### VR Components
- **Primary**: Handles client requests and coordinates replicas
- **Backup**: Maintain replicas of the primary's state
- **Client**: Sends requests to the primary

### Normal Operation
```java
public class VRReplica {
    
    private VRRole role = VRRole.BACKUP;
    private long viewNumber = 0;
    private long operationNumber = 0;
    private final List<VRReplica> replicas;
    
    public void handleClientRequest(ClientRequest request) {
        if (role == VRRole.PRIMARY) {
            // Assign operation number
            operationNumber++;
            request.setOperationNumber(operationNumber);
            
            // Send prepare messages to backups
            PrepareMessage prepareMsg = new PrepareMessage(viewNumber, request);
            List<CompletableFuture<PrepareResponse>> responses = replicas.stream()
                .filter(replica -> replica != this)
                .map(replica -> replica.receivePrepare(prepareMsg))
                .collect(Collectors.toList());
            
            // Wait for f+1 responses (where f is max failures)
            long successCount = responses.stream()
                .map(CompletableFuture::join)
                .filter(PrepareResponse::isPrepared)
                .count();
            
            if (successCount >= getQuorumSize()) {
                // Send commit to all replicas
                CommitMessage commitMsg = new CommitMessage(viewNumber, request);
                replicas.forEach(replica -> replica.receiveCommit(commitMsg));
                
                // Send reply to client
                sendReplyToClient(request);
            }
        } else {
            // Forward to primary
            primaryReplica.handleClientRequest(request);
        }
    }
    
    public PrepareResponse receivePrepare(PrepareMessage message) {
        if (message.getViewNumber() == viewNumber) {
            // Accept prepare
            return new PrepareResponse(true);
        }
        return new PrepareResponse(false);
    }
    
    public void receiveCommit(CommitMessage message) {
        // Apply operation to state machine
        applyOperation(message.getRequest());
    }
    
    private int getQuorumSize() {
        // For f failures, need 2f+1 replicas, quorum is f+1
        int totalReplicas = replicas.size() + 1; // Include self
        int maxFailures = (totalReplicas - 1) / 2;
        return maxFailures + 1;
    }
}
```

## Byzantine Fault Tolerance

### Problem
Byzantine faults occur when components fail arbitrarily, including sending incorrect or malicious messages.

### Practical Byzantine Fault Tolerance (PBFT)
```java
public class PBFTNode {
    
    private final int nodeId;
    private final List<PBFTNode> nodes;
    private long sequenceNumber = 0;
    private long viewNumber = 0;
    
    public void handleClientRequest(ClientRequest request) {
        if (isPrimary()) {
            // Pre-prepare phase
            sequenceNumber++;
            PrePrepareMessage prePrepare = new PrePrepareMessage(
                viewNumber, sequenceNumber, request.getDigest());
            
            broadcast(prePrepare);
        }
    }
    
    public void receivePrePrepare(PrePrepareMessage message) {
        if (isValidPrePrepare(message)) {
            // Prepare phase
            PrepareMessage prepare = new PrepareMessage(
                viewNumber, sequenceNumber, message.getDigest(), nodeId);
            
            broadcast(prepare);
        }
    }
    
    public void receivePrepare(PrepareMessage message) {
        // Collect prepares
        if (hasQuorumPrepares(message)) {
            // Commit phase
            CommitMessage commit = new CommitMessage(
                viewNumber, sequenceNumber, message.getDigest(), nodeId);
            
            broadcast(commit);
        }
    }
    
    public void receiveCommit(CommitMessage message) {
        // Collect commits
        if (hasQuorumCommits(message)) {
            // Execute request
            executeRequest(message.getRequest());
            
            // Reply to client
            sendReply(message.getRequest());
        }
    }
    
    private boolean isPrimary() {
        return (viewNumber % nodes.size()) == nodeId;
    }
    
    private boolean hasQuorumPrepares(PrepareMessage message) {
        // Need 2f+1 matching prepares (where f is max faulty nodes)
        return true; // Simplified
    }
}
```

## Consensus in Distributed Databases

### etcd Consensus
```java
@Configuration
public class EtcdConfig {
    
    @Bean
    public Client etcdClient() {
        return Client.builder()
            .endpoints("http://localhost:2379")
            .build();
    }
    
    @Bean
    public KV kvClient(Client client) {
        return client.getKVClient();
    }
}

@Service
public class DistributedLockService {
    
    @Autowired
    private KV kvClient;
    
    public boolean acquireLock(String lockKey, String ownerId, Duration ttl) {
        ByteSequence key = ByteSequence.from(lockKey.getBytes());
        ByteSequence value = ByteSequence.from(ownerId.getBytes());
        
        Lease lease = kvClient.getLeaseClient().grant(ttl.getSeconds()).get();
        
        Txn txn = Txn.newBuilder()
            .If(new Cmp(key, Cmp.Op.EQUAL, CmpTarget.version(0)))
            .Then(Op.put(key, value, PutOption.newBuilder().withLeaseId(lease.getID()).build()))
            .Else(Op.get(key, GetOption.DEFAULT))
            .build();
        
        CompletableFuture<TxnResponse> txnFuture = kvClient.txn(txn);
        TxnResponse response = txnFuture.get();
        
        return !response.getGetResponses().isEmpty(); // Lock acquired if key didn't exist
    }
    
    public void releaseLock(String lockKey) {
        kvClient.delete(ByteSequence.from(lockKey.getBytes()));
    }
}
```

### ZooKeeper Consensus
```java
@Service
public class ZooKeeperConsensusService {
    
    private ZooKeeper zk;
    
    public void createEphemeralNode(String path, byte[] data) throws Exception {
        // Ephemeral nodes are automatically deleted when session ends
        // Used for leader election and service discovery
        zk.create(path, data, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
    }
    
    public void electLeader(String electionPath) throws Exception {
        String myPath = zk.create(
            electionPath + "/candidate-", 
            getNodeId().getBytes(), 
            ZooDefs.Ids.OPEN_ACL_UNSAFE, 
            CreateMode.EPHEMERAL_SEQUENTIAL);
        
        List<String> candidates = zk.getChildren(electionPath, false);
        Collections.sort(candidates);
        
        if (myPath.equals(electionPath + "/" + candidates.get(0))) {
            // I am the leader
            becomeLeader();
        } else {
            // Watch the candidate ahead of me
            String predecessor = candidates.get(
                candidates.indexOf(myPath.substring(electionPath.length() + 1)) - 1);
            
            zk.exists(electionPath + "/" + predecessor, new LeaderWatcher());
        }
    }
    
    private class LeaderWatcher implements Watcher {
        @Override
        public void process(WatchedEvent event) {
            if (event.getType() == Event.EventType.NodeDeleted) {
                // Predecessor disappeared, try to become leader
                try {
                    electLeader(event.getPath().substring(0, event.getPath().lastIndexOf("/")));
                } catch (Exception e) {
                    logger.error("Failed to elect leader", e);
                }
            }
        }
    }
}
```

## Performance Considerations

### Throughput Optimization
```java
public class ConsensusThroughputOptimizer {
    
    // Batch multiple operations
    public void batchOperations(List<Operation> operations) {
        if (operations.size() == 1) {
            // Single operation
            proposeOperation(operations.get(0));
        } else {
            // Batch operations together
            OperationBatch batch = new OperationBatch(operations);
            proposeOperation(batch);
        }
    }
    
    // Parallel consensus for independent operations
    public void parallelConsensus(List<Operation> operations) {
        operations.stream()
            .collect(Collectors.groupingBy(Operation::getKey))
            .values()
            .parallelStream()
            .forEach(keyOperations -> {
                // Consensus per key group
                proposeOperationsForKey(keyOperations);
            });
    }
    
    // Read optimization - no consensus needed
    public Object readLocal(String key) {
        // Read from local replica (eventually consistent)
        return localStore.get(key);
    }
    
    public Object readQuorum(String key) {
        // Read from majority for strong consistency
        return quorumRead(key);
    }
}
```

### Latency Reduction
```java
public class ConsensusLatencyOptimizer {
    
    // Leader leases to reduce coordination
    public class LeaderLease {
        private long leaseExpiration;
        
        public boolean isValid() {
            return System.currentTimeMillis() < leaseExpiration;
        }
        
        public void renew() {
            leaseExpiration = System.currentTimeMillis() + LEASE_DURATION_MS;
        }
    }
    
    // Fast path for single replica
    public Object fastRead(String key) {
        if (isSingleReplicaDeployment()) {
            return localRead(key);
        }
        return quorumRead(key);
    }
    
    // Chain replication for faster writes
    public class ChainReplication {
        
        public void appendToChain(Operation op) {
            // Send to head of chain
            Replica head = getChainHead();
            head.receiveOperation(op);
        }
        
        public Object readFromTail() {
            // Read from tail (most up-to-date)
            Replica tail = getChainTail();
            return tail.getLatestValue();
        }
    }
}
```

## Monitoring Consensus

### Consensus Health Metrics
```java
@Service
public class ConsensusMetricsService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter consensusRounds = Counter.builder("consensus_rounds_total")
        .description("Total consensus rounds")
        .register(meterRegistry);
    
    private final Counter failedConsensus = Counter.builder("consensus_failures_total")
        .description("Total consensus failures")
        .register(meterRegistry);
    
    private final Histogram consensusLatency = Histogram.builder("consensus_latency_seconds")
        .description("Consensus operation latency")
        .register(meterRegistry);
    
    private final Gauge leaderChanges = Gauge.builder("leader_changes_total")
        .description("Total leader changes")
        .register(meterRegistry);
    
    public void recordConsensusRound(long latencyMs) {
        consensusRounds.increment();
        consensusLatency.observe(latencyMs / 1000.0);
    }
    
    public void recordConsensusFailure() {
        failedConsensus.increment();
    }
    
    public void recordLeaderChange() {
        // Update gauge
    }
}
```

## Choosing Consensus Algorithms

### Decision Criteria

#### Performance Requirements
```java
public class ConsensusAlgorithmSelector {
    
    public ConsensusAlgorithm selectAlgorithm(SystemRequirements reqs) {
        
        if (reqs.getExpectedThroughput() > 100000) {
            // High throughput needs
            return ConsensusAlgorithm.RAFT; // Better performance than Paxos
        }
        
        if (reqs.getNetworkLatency() > 100) {
            // High latency networks
            return ConsensusAlgorithm.RAFT; // More efficient than Paxos
        }
        
        if (reqs.isLargeCluster()) {
            // Large number of nodes
            return ConsensusAlgorithm.RAFT; // Scales better than Paxos
        }
        
        if (reqs.needsByzantineTolerance()) {
            // Faulty nodes possible
            return ConsensusAlgorithm.PBFT;
        }
        
        // Default choice
        return ConsensusAlgorithm.RAFT;
    }
}
```

### Algorithm Comparison

| Algorithm | Strengths | Weaknesses | Best Use Case |
|-----------|-----------|------------|---------------|
| **Paxos** | Proven, flexible | Complex implementation | Research, custom protocols |
| **Raft** | Easy to understand, good performance | Newer protocol | Production systems, Kubernetes |
| **Zab** | ZooKeeper integration, reliable | Tied to ZooKeeper | ZooKeeper, distributed coordination |
| **VR** | Strong consistency, simple | Less widely used | Academic, specialized systems |
| **PBFT** | Byzantine fault tolerance | Performance overhead | Blockchain, critical systems |

## Real-World Consensus Examples

### Kubernetes etcd
```yaml
# etcd cluster configuration
apiVersion: v1
kind: Pod
metadata:
  name: etcd
spec:
  containers:
  - name: etcd
    image: quay.io/coreos/etcd:v3.5.0
    command:
    - etcd
    - --name=$(ETCD_NAME)
    - --initial-advertise-peer-urls=http://$(ETCD_NAME).etcd:2380
    - --listen-peer-urls=http://0.0.0.0:2380
    - --listen-client-urls=http://0.0.0.0:2379
    - --advertise-client-urls=http://$(ETCD_NAME).etcd:2379
    - --initial-cluster=etcd-0=http://etcd-0.etcd:2380,etcd-1=http://etcd-1.etcd:2380,etcd-2=http://etcd-2.etcd:2380
    - --initial-cluster-state=new
    - --auto-compaction-retention=1
```

### Apache ZooKeeper Ensemble
```java
public class ZooKeeperEnsemble {
    
    public void createEnsemble(int numServers) {
        for (int i = 0; i < numServers; i++) {
            ZooKeeperServer server = new ZooKeeperServer(
                new File("data/zookeeper-" + i), 
                new File("logs/zookeeper-" + i), 
                2000);
            
            ServerCnxnFactory factory = ServerCnxnFactory.createFactory(
                new InetSocketAddress("localhost", 2181 + i), 60);
            
            factory.startup(server);
            
            // Configure server list in zoo.cfg
            configureServerList(numServers);
        }
    }
    
    private void configureServerList(int numServers) {
        // server.1=localhost:2888:3888
        // server.2=localhost:2889:3889
        // etc.
    }
}
```

### HashiCorp Consul
```java
@Service
public class ConsulConsensusService {
    
    @Autowired
    private ConsulClient consulClient;
    
    public boolean acquireLock(String lockKey, String ownerId) {
        Lock lock = consulClient.lock(
            LockOptions.builder()
                .key(lockKey)
                .session(Session.builder()
                    .name("service-lock")
                    .behavior(Session.Behavior.DELETE)
                    .ttl("10s")
                    .build())
                .build());
        
        return lock.acquire();
    }
    
    public void releaseLock(Lock lock) {
        lock.release();
    }
    
    public void registerService(String serviceId, String serviceName, String address, int port) {
        AgentServiceRegistration registration = AgentServiceRegistration.builder()
            .id(serviceId)
            .name(serviceName)
            .address(address)
            .port(port)
            .build();
        
        consulClient.agentServiceRegister(registration);
    }
}
```

## Conclusion

Consensus algorithms are essential for building reliable distributed systems. They enable coordination and agreement among distributed components despite failures and network issues.

**Key Takeaways:**
- **Paxos**: Proven but complex, foundation for many algorithms
- **Raft**: Easier to understand and implement than Paxos
- **Zab**: Reliable consensus for ZooKeeper and similar systems
- **Performance Trade-offs**: Different algorithms optimize for different scenarios
- **Fault Tolerance**: Choose based on expected failure modes
- **Monitoring**: Track consensus health and performance

Selecting the right consensus algorithm depends on your system's requirements for performance, reliability, and operational complexity.
