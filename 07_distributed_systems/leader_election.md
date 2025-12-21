# Leader Election

Leader election is the process of designating a single process as the leader among a group of processes in a distributed system. The leader coordinates activities, makes decisions, and handles critical operations. This guide covers leader election algorithms, implementations, and patterns.

## What is Leader Election?

Leader election is a fundamental problem in distributed systems where a group of processes must choose one process to act as the coordinator or leader. The leader handles tasks like coordinating writes, managing state, and making system-wide decisions.

### Key Concepts
- **Leader**: Process responsible for coordination and critical decisions
- **Follower**: Processes that respond to leader requests
- **Candidate**: Process attempting to become leader during election
- **Term/Epoch**: Logical timestamp for leadership periods
- **Quorum**: Minimum number of processes that must agree

### Leader Election Challenges
- **Network Partitions**: Communication failures between processes
- **Process Failures**: Leader or follower crashes
- **Split Brain**: Multiple leaders emerge due to partitions
- **Clock Skew**: Different time perceptions across processes

## Leader Election Algorithms

### Bully Algorithm

#### Overview
Processes are ordered by IDs. Higher ID processes can bully lower ID processes into submission.

#### Implementation
```java
public class BullyLeaderElection {
    
    private final int processId;
    private final List<Integer> allProcesses;
    private volatile Integer currentLeader = null;
    private final ExecutorService executor = Executors.newCachedThreadPool();
    
    public BullyLeaderElection(int processId, List<Integer> allProcesses) {
        this.processId = processId;
        this.allProcesses = new ArrayList<>(allProcesses);
        Collections.sort(this.allProcesses, Collections.reverseOrder()); // Higher IDs first
    }
    
    public void startElection() {
        if (currentLeader != null && currentLeader > processId) {
            // Higher process exists, don't start election
            return;
        }
        
        logger.info("Process {} starting bully election", processId);
        
        // Send election messages to higher processes
        List<Integer> higherProcesses = allProcesses.stream()
            .filter(id -> id > processId)
            .collect(Collectors.toList());
        
        List<CompletableFuture<ElectionResponse>> responses = new ArrayList<>();
        
        for (int higherProcess : higherProcesses) {
            CompletableFuture<ElectionResponse> response = CompletableFuture.supplyAsync(() -> {
                try {
                    return sendElectionMessage(higherProcess);
                } catch (Exception e) {
                    return new ElectionResponse(false, -1);
                }
            }, executor);
            responses.add(response);
        }
        
        // Wait for all responses with timeout
        boolean higherProcessResponded = responses.stream()
            .map(future -> {
                try {
                    return future.get(5, TimeUnit.SECONDS);
                } catch (Exception e) {
                    return new ElectionResponse(false, -1);
                }
            })
            .anyMatch(ElectionResponse::isAlive);
        
        if (!higherProcessResponded) {
            // No higher process responded - become leader
            becomeLeader();
        }
    }
    
    private void becomeLeader() {
        currentLeader = processId;
        logger.info("Process {} became leader (bully)", processId);
        
        // Announce leadership
        announceLeadership();
        
        // Start leader duties
        startLeaderResponsibilities();
    }
    
    private void announceLeadership() {
        for (int process : allProcesses) {
            if (process != processId) {
                try {
                    sendCoordinatorMessage(process);
                } catch (Exception e) {
                    logger.warn("Failed to announce leadership to {}", process);
                }
            }
        }
    }
    
    public void handleCoordinatorMessage(int leaderId) {
        currentLeader = leaderId;
        logger.info("Process {} acknowledges coordinator {}", processId, leaderId);
        
        if (leaderId != processId) {
            // Stop being leader if we were
            stopLeaderResponsibilities();
        }
    }
    
    public void handleElectionMessage(int fromProcess) {
        // Send OK response back
        try {
            sendOkMessage(fromProcess);
        } catch (Exception e) {
            logger.warn("Failed to send OK to {}", fromProcess);
        }
        
        // Start own election if not already leader
        if (currentLeader == null || currentLeader != processId) {
            startElection();
        }
    }
    
    // Placeholder methods for messaging
    private ElectionResponse sendElectionMessage(int targetProcess) {
        // Implementation would send actual message
        return new ElectionResponse(true, targetProcess);
    }
    
    private void sendOkMessage(int targetProcess) {
        // Send OK response
    }
    
    private void sendCoordinatorMessage(int targetProcess) {
        // Send coordinator announcement
    }
    
    private void startLeaderResponsibilities() {
        // Start heartbeat, coordination, etc.
    }
    
    private void stopLeaderResponsibilities() {
        // Stop leader duties
    }
    
    public Integer getCurrentLeader() {
        return currentLeader;
    }
}
```

### Ring Algorithm

#### Overview
Processes are arranged in a logical ring. Election messages circulate around the ring.

#### Implementation
```java
public class RingLeaderElection {
    
    private final int processId;
    private final List<Integer> ringProcesses;
    private volatile Integer currentLeader = null;
    private boolean participating = false;
    private final ExecutorService executor = Executors.newCachedThreadPool();
    
    public RingLeaderElection(int processId, List<Integer> processes) {
        this.processId = processId;
        this.ringProcesses = new ArrayList<>(processes);
        Collections.sort(this.ringProcesses); // Create ring order
    }
    
    public void startElection() {
        if (participating) return; // Already participating
        
        participating = true;
        logger.info("Process {} starting ring election", processId);
        
        // Send election message to next process in ring
        int nextProcess = getNextProcess();
        sendElectionMessage(nextProcess, Arrays.asList(processId));
    }
    
    public void handleElectionMessage(List<Integer> participantIds) {
        if (participating) {
            // Already participating, forward message
            int nextProcess = getNextProcess();
            sendElectionMessage(nextProcess, participantIds);
            return;
        }
        
        participating = true;
        List<Integer> newParticipants = new ArrayList<>(participantIds);
        newParticipants.add(processId);
        
        if (isCoordinator(newParticipants)) {
            // This process has highest ID - become coordinator
            becomeCoordinator(newParticipants);
        } else {
            // Forward to next process
            int nextProcess = getNextProcess();
            sendElectionMessage(nextProcess, newParticipants);
        }
    }
    
    public void handleCoordinatorMessage(int leaderId) {
        currentLeader = leaderId;
        participating = false;
        logger.info("Process {} acknowledges coordinator {}", processId, leaderId);
    }
    
    private void becomeCoordinator(List<Integer> participants) {
        currentLeader = processId;
        participating = false;
        
        // Send coordinator message to all processes
        for (int process : participants) {
            if (process != processId) {
                sendCoordinatorMessage(process);
            }
        }
        
        logger.info("Process {} became coordinator (ring)", processId);
        
        // Start leader responsibilities
        startLeaderResponsibilities();
    }
    
    private boolean isCoordinator(List<Integer> participants) {
        int maxId = participants.stream().max(Integer::compare).orElse(-1);
        return maxId == processId;
    }
    
    private int getNextProcess() {
        int currentIndex = ringProcesses.indexOf(processId);
        int nextIndex = (currentIndex + 1) % ringProcesses.size();
        return ringProcesses.get(nextIndex);
    }
    
    private void startLeaderResponsibilities() {
        // Start coordination tasks
    }
    
    // Message sending methods (simplified)
    private void sendElectionMessage(int targetProcess, List<Integer> participantIds) {
        // Send election message with participant list
    }
    
    private void sendCoordinatorMessage(int targetProcess) {
        // Send coordinator announcement
    }
    
    public Integer getCurrentLeader() {
        return currentLeader;
    }
}
```

## Raft Consensus Algorithm

### Overview
Raft provides strong leader election guarantees with clear separation of leader, follower, and candidate roles.

### Leader Election in Raft
```java
public class RaftLeaderElection {
    
    private RaftState state = RaftState.FOLLOWER;
    private int currentTerm = 0;
    private Integer votedFor = null;
    private final List<RaftNode> peers;
    private final int nodeId;
    private final ExecutorService executor = Executors.newCachedThreadPool();
    
    // Election timeout management
    private long electionTimeoutMs = 5000; // 5 seconds base
    private long lastHeartbeat = System.currentTimeMillis();
    private final Random random = new Random();
    
    public RaftLeaderElection(int nodeId, List<RaftNode> peers) {
        this.nodeId = nodeId;
        this.peers = new ArrayList<>(peers);
        
        // Start election timer
        startElectionTimer();
    }
    
    private void startElectionTimer() {
        executor.scheduleAtFixedRate(this::checkElectionTimeout, 
            getRandomizedTimeout(), getRandomizedTimeout(), TimeUnit.MILLISECONDS);
    }
    
    private long getRandomizedTimeout() {
        // Add randomization to prevent split votes
        return electionTimeoutMs + random.nextInt(1000);
    }
    
    private void checkElectionTimeout() {
        long timeSinceLastHeartbeat = System.currentTimeMillis() - lastHeartbeat;
        
        if (timeSinceLastHeartbeat > electionTimeoutMs && state != RaftState.LEADER) {
            startElection();
        }
    }
    
    public void startElection() {
        synchronized (this) {
            if (state == RaftState.LEADER) return;
            
            state = RaftState.CANDIDATE;
            currentTerm++;
            votedFor = nodeId;
            int votes = 1; // Vote for self
            
            logger.info("Node {} starting election for term {}", nodeId, currentTerm);
            
            // Send vote requests to all peers
            List<CompletableFuture<VoteResponse>> voteFutures = peers.stream()
                .map(peer -> peer.requestVote(new VoteRequest(currentTerm, nodeId)))
                .collect(Collectors.toList());
            
            // Count votes with timeout
            for (CompletableFuture<VoteResponse> future : voteFutures) {
                try {
                    VoteResponse response = future.get(2, TimeUnit.SECONDS);
                    if (response.isGranted()) {
                        votes++;
                    }
                } catch (Exception e) {
                    // Vote request failed or timed out
                    logger.debug("Vote request failed for term {}", currentTerm);
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
        logger.info("Node {} became leader for term {}", nodeId, currentTerm);
        
        // Start sending heartbeats
        startHeartbeatTimer();
        
        // Initialize leader state
        initializeLeaderState();
    }
    
    private void becomeFollower() {
        state = RaftState.FOLLOWER;
        votedFor = null;
        logger.info("Node {} became follower for term {}", nodeId, currentTerm);
    }
    
    private void startHeartbeatTimer() {
        executor.scheduleAtFixedRate(this::sendHeartbeats, 0, 1000, TimeUnit.MILLISECONDS);
    }
    
    private void sendHeartbeats() {
        if (state != RaftState.LEADER) return;
        
        // Send heartbeats to all peers
        AppendEntriesRequest heartbeat = new AppendEntriesRequest(
            currentTerm, nodeId, 0, 0, Collections.emptyList());
        
        peers.forEach(peer -> {
            executor.submit(() -> {
                try {
                    peer.appendEntries(heartbeat);
                } catch (Exception e) {
                    logger.debug("Heartbeat failed to peer {}", peer.getId());
                }
            });
        });
    }
    
    public void receiveHeartbeat(int leaderTerm, int leaderId) {
        if (leaderTerm >= currentTerm) {
            currentTerm = leaderTerm;
            lastHeartbeat = System.currentTimeMillis();
            
            if (state != RaftState.FOLLOWER) {
                becomeFollower();
            }
            
            // Acknowledge leadership
            logger.debug("Received heartbeat from leader {} in term {}", leaderId, leaderTerm);
        }
    }
    
    public VoteResponse handleVoteRequest(VoteRequest request) {
        if (request.getTerm() < currentTerm) {
            return new VoteResponse(false);
        }
        
        if (request.getTerm() > currentTerm) {
            currentTerm = request.getTerm();
            votedFor = null;
            becomeFollower();
        }
        
        // Vote for candidate if we haven't voted or already voted for them
        if (votedFor == null || votedFor == request.getCandidateId()) {
            votedFor = request.getCandidateId();
            return new VoteResponse(true);
        }
        
        return new VoteResponse(false);
    }
    
    private void initializeLeaderState() {
        // Initialize nextIndex and matchIndex for all peers
        // Start log replication
    }
    
    public RaftState getState() {
        return state;
    }
    
    public int getCurrentTerm() {
        return currentTerm;
    }
}
```

## ZooKeeper Leader Election

### Using Ephemeral Sequential Nodes
```java
@Service
public class ZooKeeperLeaderElection {
    
    private ZooKeeper zk;
    private final String electionPath;
    private final String nodeId;
    private volatile boolean isLeader = false;
    private String leaderNodePath;
    
    public ZooKeeperLeaderElection(ZooKeeper zk, String electionPath, String nodeId) {
        this.zk = zk;
        this.electionPath = electionPath;
        this.nodeId = nodeId;
    }
    
    public void startElection() throws Exception {
        // Create election znode if it doesn't exist
        if (zk.exists(electionPath, false) == null) {
            zk.create(electionPath, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }
        
        // Create ephemeral sequential node
        String nodePath = zk.create(
            electionPath + "/candidate-", 
            nodeId.getBytes(), 
            ZooDefs.Ids.OPEN_ACL_UNSAFE, 
            CreateMode.EPHEMERAL_SEQUENTIAL);
        
        leaderNodePath = nodePath;
        
        // Check if we are the leader
        checkLeadership();
        
        // Watch for changes
        setupWatcher();
    }
    
    private void checkLeadership() throws Exception {
        List<String> children = zk.getChildren(electionPath, false);
        Collections.sort(children);
        
        String myNode = leaderNodePath.substring(leaderNodePath.lastIndexOf('/') + 1);
        int myIndex = children.indexOf(myNode);
        
        if (myIndex == 0) {
            // We are the leader
            becomeLeader();
        } else {
            // Watch the node before us
            String predecessor = children.get(myIndex - 1);
            watchPredecessor(electionPath + "/" + predecessor);
        }
    }
    
    private void watchPredecessor(String predecessorPath) throws Exception {
        if (zk.exists(predecessorPath, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == EventType.NodeDeleted) {
                    // Predecessor disappeared, check if we became leader
                    try {
                        checkLeadership();
                    } catch (Exception e) {
                        logger.error("Failed to check leadership", e);
                    }
                }
            }
        }) == null) {
            // Predecessor doesn't exist, we might be leader
            checkLeadership();
        }
    }
    
    private void becomeLeader() {
        isLeader = true;
        logger.info("Node {} became leader", nodeId);
        
        // Start leader responsibilities
        startLeaderDuties();
    }
    
    private void setupWatcher() throws Exception {
        zk.getChildren(electionPath, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == EventType.NodeChildrenChanged) {
                    try {
                        checkLeadership();
                    } catch (Exception e) {
                        logger.error("Failed to check leadership on children change", e);
                    }
                }
            }
        });
    }
    
    private void startLeaderDuties() {
        // Implement leader-specific logic
        // Coordinate operations, manage state, etc.
    }
    
    public boolean isLeader() {
        return isLeader;
    }
    
    public void stop() {
        if (leaderNodePath != null) {
            try {
                zk.delete(leaderNodePath, -1);
            } catch (Exception e) {
                logger.warn("Failed to delete leader node", e);
            }
        }
    }
}
```

## etcd Leader Election

### Using Leases and Keys
```java
@Service
public class EtcdLeaderElection {
    
    private Client etcdClient;
    private final String electionKey;
    private final String nodeId;
    private volatile boolean isLeader = false;
    private long leaseId;
    
    public EtcdLeaderElection(Client etcdClient, String electionKey, String nodeId) {
        this.etcdClient = etcdClient;
        this.electionKey = electionKey;
        this.nodeId = nodeId;
    }
    
    public void startElection() throws Exception {
        // Create lease
        Lease lease = etcdClient.getLeaseClient().grant(10); // 10 second lease
        leaseId = lease.getID();
        
        // Try to acquire leadership
        ByteSequence key = ByteSequence.from(electionKey.getBytes());
        ByteSequence value = ByteSequence.from(nodeId.getBytes());
        
        Txn txn = Txn.newBuilder()
            .If(new Cmp(key, Cmp.Op.EQUAL, CmpTarget.version(0)))
            .Then(Op.put(key, value, PutOption.newBuilder().withLeaseId(leaseId).build()))
            .Else(Op.get(key))
            .build();
        
        TxnResponse response = etcdClient.getKVClient().txn(txn).get();
        
        if (!response.getGetResponses().isEmpty()) {
            // Key exists, we didn't get leadership
            watchForLeadershipChanges();
        } else {
            // We got leadership
            becomeLeader();
        }
        
        // Keep lease alive
        keepLeaseAlive();
    }
    
    private void becomeLeader() {
        isLeader = true;
        logger.info("Node {} became leader", nodeId);
        
        // Start leader duties
        startLeaderResponsibilities();
    }
    
    private void watchForLeadershipChanges() {
        ByteSequence key = ByteSequence.from(electionKey.getBytes());
        
        Watch watch = etcdClient.getWatchClient();
        watch.watch(key, WatchOption.DEFAULT, new Watch.Listener() {
            @Override
            public void onNext(WatchResponse response) {
                // Check if we can become leader
                response.getEvents().forEach(event -> {
                    if (event.getEventType() == EventType.DELETE) {
                        // Leadership key was deleted, try to become leader
                        try {
                            startElection();
                        } catch (Exception e) {
                            logger.error("Failed to start election after watch event", e);
                        }
                    }
                });
            }
            
            @Override
            public void onError(Throwable throwable) {
                logger.error("Watch error", throwable);
            }
            
            @Override
            public void onCompleted() {
                logger.info("Watch completed");
            }
        });
    }
    
    private void keepLeaseAlive() {
        Executors.newSingleThreadScheduledExecutor()
            .scheduleAtFixedRate(() -> {
                try {
                    etcdClient.getLeaseClient().keepAliveOnce(leaseId);
                } catch (Exception e) {
                    logger.warn("Failed to keep lease alive");
                    isLeader = false;
                }
            }, 5, 5, TimeUnit.SECONDS);
    }
    
    private void startLeaderResponsibilities() {
        // Implement leader-specific operations
    }
    
    public boolean isLeader() {
        return isLeader;
    }
    
    public void stop() {
        if (leaseId != 0) {
            try {
                etcdClient.getLeaseClient().revoke(leaseId);
            } catch (Exception e) {
                logger.warn("Failed to revoke lease", e);
            }
        }
    }
}
```

## Leader Election Patterns

### Master-Slave Pattern
```java
public class MasterSlaveCoordinator {
    
    @Autowired
    private LeaderElection leaderElection;
    
    @Autowired
    private TaskQueue taskQueue;
    
    @Autowired
    private WorkerRegistry workerRegistry;
    
    private final ExecutorService executor = Executors.newCachedThreadPool();
    
    public void initialize() {
        leaderElection.addLeadershipListener(new LeadershipListener() {
            @Override
            public void onLeadershipGained() {
                startMasterResponsibilities();
            }
            
            @Override
            public void onLeadershipLost() {
                stopMasterResponsibilities();
            }
        });
        
        // Start election
        leaderElection.startElection();
    }
    
    private void startMasterResponsibilities() {
        logger.info("Starting master responsibilities");
        
        // Start task distribution
        startTaskDistribution();
        
        // Start worker monitoring
        startWorkerMonitoring();
        
        // Start health checks
        startHealthChecks();
    }
    
    private void stopMasterResponsibilities() {
        logger.info("Stopping master responsibilities");
        
        // Stop all master activities
        stopTaskDistribution();
        stopWorkerMonitoring();
        stopHealthChecks();
    }
    
    private void startTaskDistribution() {
        executor.submit(() -> {
            while (leaderElection.isLeader()) {
                try {
                    Task task = taskQueue.take();
                    Worker worker = selectWorker();
                    
                    if (worker != null) {
                        assignTaskToWorker(task, worker);
                    } else {
                        // No workers available, put task back
                        taskQueue.putBack(task);
                        Thread.sleep(1000);
                    }
                    
                } catch (Exception e) {
                    logger.error("Error in task distribution", e);
                }
            }
        });
    }
    
    private Worker selectWorker() {
        List<Worker> availableWorkers = workerRegistry.getAvailableWorkers();
        return availableWorkers.isEmpty() ? null : 
               availableWorkers.get(ThreadLocalRandom.current().nextInt(availableWorkers.size()));
    }
    
    private void assignTaskToWorker(Task task, Worker worker) {
        // Assign task to worker
        worker.assignTask(task);
    }
    
    private void startWorkerMonitoring() {
        executor.submit(() -> {
            while (leaderElection.isLeader()) {
                try {
                    List<Worker> workers = workerRegistry.getAllWorkers();
                    for (Worker worker : workers) {
                        if (!worker.isHealthy()) {
                            handleUnhealthyWorker(worker);
                        }
                    }
                    
                    Thread.sleep(5000); // Check every 5 seconds
                    
                } catch (Exception e) {
                    logger.error("Error in worker monitoring", e);
                }
            }
        });
    }
    
    private void handleUnhealthyWorker(Worker worker) {
        logger.warn("Worker {} is unhealthy", worker.getId());
        
        // Reassign tasks from unhealthy worker
        List<Task> assignedTasks = worker.getAssignedTasks();
        for (Task task : assignedTasks) {
            taskQueue.putBack(task);
        }
        
        // Remove worker from registry
        workerRegistry.removeWorker(worker);
    }
    
    private void startHealthChecks() {
        // Start periodic health checks
    }
    
    private void stopTaskDistribution() {
        // Stop task distribution
    }
    
    private void stopWorkerMonitoring() {
        // Stop worker monitoring
    }
    
    private void stopHealthChecks() {
        // Stop health checks
    }
}
```

### Leader-Lease Pattern
```java
public class LeaderLeaseManager {
    
    @Autowired
    private LeaderElection leaderElection;
    
    private long leaseDurationMs = 30000; // 30 seconds
    private long leaseRenewalIntervalMs = 10000; // 10 seconds
    private volatile long leaseExpirationTime = 0;
    private final ScheduledExecutorService leaseExecutor = Executors.newScheduledThreadPool(1);
    
    public void initialize() {
        leaderElection.addLeadershipListener(new LeadershipListener() {
            @Override
            public void onLeadershipGained() {
                acquireLease();
            }
            
            @Override
            public void onLeadershipLost() {
                releaseLease();
            }
        });
    }
    
    private void acquireLease() {
        leaseExpirationTime = System.currentTimeMillis() + leaseDurationMs;
        logger.info("Acquired leader lease, expires at {}", leaseExpirationTime);
        
        // Start lease renewal
        startLeaseRenewal();
    }
    
    private void startLeaseRenewal() {
        leaseExecutor.scheduleAtFixedRate(() -> {
            if (leaderElection.isLeader() && isLeaseExpiring()) {
                renewLease();
            }
        }, leaseRenewalIntervalMs, leaseRenewalIntervalMs, TimeUnit.MILLISECONDS);
    }
    
    private boolean isLeaseExpiring() {
        long timeUntilExpiration = leaseExpirationTime - System.currentTimeMillis();
        return timeUntilExpiration < (leaseDurationMs / 2); // Renew when half expired
    }
    
    private void renewLease() {
        if (!leaderElection.isLeader()) {
            return; // Not leader anymore
        }
        
        try {
            // Attempt to renew lease (e.g., update timestamp in shared storage)
            long newExpirationTime = System.currentTimeMillis() + leaseDurationMs;
            
            if (updateLeaseInStorage(newExpirationTime)) {
                leaseExpirationTime = newExpirationTime;
                logger.debug("Renewed leader lease");
            } else {
                // Failed to renew, relinquish leadership
                logger.warn("Failed to renew lease, relinquishing leadership");
                leaderElection.relinquishLeadership();
            }
            
        } catch (Exception e) {
            logger.error("Error renewing lease", e);
            leaderElection.relinquishLeadership();
        }
    }
    
    private boolean updateLeaseInStorage(long newExpirationTime) {
        // Update lease in shared storage (ZooKeeper, etcd, etc.)
        // Return true if successful
        return true; // Simplified
    }
    
    private void releaseLease() {
        leaseExpirationTime = 0;
        logger.info("Released leader lease");
        
        // Stop lease renewal
        leaseExecutor.shutdown();
    }
    
    public boolean hasValidLease() {
        return leaderElection.isLeader() && 
               System.currentTimeMillis() < leaseExpirationTime;
    }
    
    public long getLeaseExpirationTime() {
        return leaseExpirationTime;
    }
}
```

## Monitoring Leader Election

### Election Metrics
```java
@Service
public class LeaderElectionMetrics {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter electionsStarted = Counter.builder("leader_elections_started_total")
        .description("Total leader elections started")
        .register(meterRegistry);
    
    private final Counter electionsWon = Counter.builder("leader_elections_won_total")
        .description("Total leader elections won")
        .register(meterRegistry);
    
    private final Counter leadershipChanges = Counter.builder("leadership_changes_total")
        .description("Total leadership changes")
        .register(meterRegistry);
    
    private final Gauge currentLeader = Gauge.builder("current_leader_id")
        .description("Current leader process ID")
        .register(meterRegistry);
    
    private final Histogram electionDuration = Histogram.builder("election_duration_seconds")
        .description("Time taken for leader election")
        .register(meterRegistry);
    
    private final Histogram leadershipDuration = Histogram.builder("leadership_duration_seconds")
        .description("Duration of leadership periods")
        .register(meterRegistry);
    
    public void recordElectionStarted() {
        electionsStarted.increment();
    }
    
    public void recordElectionWon(long durationSeconds) {
        electionsWon.increment();
        electionDuration.observe(durationSeconds);
    }
    
    public void recordLeadershipChange(int newLeaderId, long previousLeadershipDurationSeconds) {
        leadershipChanges.increment();
        leadershipDuration.observe(previousLeadershipDurationSeconds);
        // Update current leader gauge
    }
}
```

## Best Practices

### Election Design
- **Uniqueness**: Ensure only one leader at a time
- **Liveness**: Eventually elect a leader when needed
- **Safety**: Never elect multiple leaders simultaneously
- **Fault Tolerance**: Continue working despite process failures

### Implementation Guidelines
- **Timeouts**: Use appropriate timeouts for elections and heartbeats
- **Randomization**: Add randomization to prevent split votes
- **Logging**: Comprehensive logging for debugging election issues
- **Monitoring**: Track election metrics and leadership stability

### Operational Considerations
- **Network Partitions**: Handle network splits gracefully
- **Process Restarts**: Handle leader and follower restarts
- **Load Balancing**: Distribute leadership duties appropriately
- **Failover**: Quick failover when leader fails

## Conclusion

Leader election is crucial for coordinating activities in distributed systems. Different algorithms (Bully, Ring, Raft) provide various trade-offs in terms of simplicity, performance, and fault tolerance. Choose the algorithm that best fits your system's requirements and constraints.

**Key Takeaways:**
- **Bully Algorithm**: Simple but requires ordered process IDs
- **Ring Algorithm**: Deterministic but can be slow in large rings
- **Raft**: Production-ready with strong guarantees
- **ZooKeeper/etcd**: Practical implementations using coordination services

Effective leader election ensures system coordination while maintaining fault tolerance and performance.
