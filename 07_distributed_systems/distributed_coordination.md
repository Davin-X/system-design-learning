# Distributed Coordination

Distributed coordination ensures that multiple processes or services work together harmoniously in a distributed system. It involves managing shared resources, coordinating actions, and maintaining system-wide consistency. This guide covers distributed coordination patterns, algorithms, and implementations.

## What is Distributed Coordination?

Distributed coordination involves synchronizing the actions of multiple distributed processes to achieve a common goal. It ensures that different parts of a distributed system can work together effectively despite network delays, failures, and concurrent operations.

### Key Concepts
- **Coordination**: Managing dependencies and timing between distributed components
- **Synchronization**: Ensuring processes agree on shared state and timing
- **Consensus**: Achieving agreement on decisions among distributed processes
- **Atomicity**: Ensuring operations complete entirely or not at all
- **Isolation**: Preventing interference between concurrent operations

### Coordination Challenges
- **Network Partitions**: Communication failures between nodes
- **Node Failures**: Individual process or machine failures
- **Clock Skew**: Different time perceptions across nodes
- **Race Conditions**: Concurrent access to shared resources
- **Deadlocks**: Circular waiting dependencies

## Coordination Primitives

### 1. Locks and Semaphores

#### Distributed Locks
```java
public interface DistributedLock {
    boolean tryLock(String resourceId, Duration timeout) throws Exception;
    void unlock(String resourceId) throws Exception;
    boolean isLocked(String resourceId);
}

// Redis-based distributed lock
@Service
public class RedisDistributedLock implements DistributedLock {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final String LOCK_PREFIX = "lock:";
    private static final String OWNER_PREFIX = "owner:";
    
    @Override
    public boolean tryLock(String resourceId, Duration timeout) throws Exception {
        String lockKey = LOCK_PREFIX + resourceId;
        String ownerKey = OWNER_PREFIX + resourceId;
        String ownerId = getCurrentOwnerId();
        
        // Use Redis SET with NX and PX for atomic lock acquisition
        Boolean acquired = redisTemplate.opsForValue().setIfAbsent(
            lockKey, 
            ownerId, 
            timeout
        );
        
        if (Boolean.TRUE.equals(acquired)) {
            // Store owner information
            redisTemplate.opsForValue().set(ownerKey, ownerId, timeout);
            return true;
        }
        
        return false;
    }
    
    @Override
    public void unlock(String resourceId) throws Exception {
        String lockKey = LOCK_PREFIX + resourceId;
        String ownerKey = OWNER_PREFIX + resourceId;
        String currentOwner = getCurrentOwnerId();
        
        // Use Lua script for atomic unlock (check-and-delete)
        String script = 
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    redis.call('del', KEYS[1]); " +
            "    redis.call('del', KEYS[2]); " +
            "    return 1; " +
            "else " +
            "    return 0; " +
            "end";
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Arrays.asList(lockKey, ownerKey),
            currentOwner
        );
        
        if (result == null || result != 1) {
            throw new IllegalStateException("Failed to unlock - not the owner");
        }
    }
    
    @Override
    public boolean isLocked(String resourceId) {
        return redisTemplate.hasKey(LOCK_PREFIX + resourceId);
    }
    
    private String getCurrentOwnerId() {
        // Use thread ID or process ID as owner identifier
        return "process-" + ManagementFactory.getRuntimeMXBean().getName() + 
               "-thread-" + Thread.currentThread().getId();
    }
}
```

#### Read-Write Locks
```java
public class ReadWriteDistributedLock {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final String READ_LOCK_PREFIX = "readlock:";
    private static final String WRITE_LOCK_PREFIX = "writelock:";
    private static final String READ_COUNT_PREFIX = "readcount:";
    
    public boolean tryReadLock(String resourceId, Duration timeout) {
        String readLockKey = READ_LOCK_PREFIX + resourceId;
        String writeLockKey = WRITE_LOCK_PREFIX + resourceId;
        String readCountKey = READ_COUNT_PREFIX + resourceId;
        String ownerId = getCurrentOwnerId();
        
        // Check if write lock exists
        if (redisTemplate.hasKey(writeLockKey)) {
            return false; // Write lock held, cannot acquire read lock
        }
        
        // Acquire read lock and increment count
        Boolean readLockAcquired = redisTemplate.opsForValue().setIfAbsent(
            readLockKey, ownerId, timeout);
        
        if (Boolean.TRUE.equals(readLockAcquired)) {
            redisTemplate.opsForValue().increment(readCountKey);
            redisTemplate.expire(readCountKey, timeout);
            return true;
        }
        
        return false;
    }
    
    public boolean tryWriteLock(String resourceId, Duration timeout) {
        String readLockKey = READ_LOCK_PREFIX + resourceId;
        String writeLockKey = WRITE_LOCK_PREFIX + resourceId;
        String readCountKey = READ_COUNT_PREFIX + resourceId;
        String ownerId = getCurrentOwnerId();
        
        // Check if any read locks exist
        Long readCount = (Long) redisTemplate.opsForValue().get(readCountKey);
        if (readCount != null && readCount > 0) {
            return false; // Read locks held, cannot acquire write lock
        }
        
        // Acquire write lock
        Boolean writeLockAcquired = redisTemplate.opsForValue().setIfAbsent(
            writeLockKey, ownerId, timeout);
        
        return Boolean.TRUE.equals(writeLockAcquired);
    }
    
    public void releaseReadLock(String resourceId) {
        String readLockKey = READ_LOCK_PREFIX + resourceId;
        String readCountKey = READ_COUNT_PREFIX + resourceId;
        
        Long newCount = redisTemplate.opsForValue().decrement(readCountKey);
        if (newCount <= 0) {
            redisTemplate.delete(readLockKey);
            redisTemplate.delete(readCountKey);
        }
    }
    
    public void releaseWriteLock(String resourceId) {
        redisTemplate.delete(WRITE_LOCK_PREFIX + resourceId);
    }
}
```

### 2. Barriers

#### Synchronization Barriers
```java
public class DistributedBarrier {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private final String barrierKey;
    private final int totalParticipants;
    
    public DistributedBarrier(String barrierId, int totalParticipants) {
        this.barrierKey = "barrier:" + barrierId;
        this.totalParticipants = totalParticipants;
    }
    
    public boolean await(Duration timeout) throws Exception {
        String participantKey = barrierKey + ":participant:" + getParticipantId();
        
        // Register participant
        redisTemplate.opsForValue().set(participantKey, "waiting", timeout);
        
        // Count total participants
        Set<String> participants = redisTemplate.keys(barrierKey + ":participant:*");
        if (participants.size() >= totalParticipants) {
            // All participants arrived - signal barrier release
            redisTemplate.opsForValue().set(barrierKey + ":released", "true");
            return true;
        }
        
        // Wait for barrier release
        Instant startTime = Instant.now();
        while (Duration.between(startTime, Instant.now()).compareTo(timeout) < 0) {
            String released = (String) redisTemplate.opsForValue().get(barrierKey + ":released");
            if ("true".equals(released)) {
                return true;
            }
            Thread.sleep(100); // Poll interval
        }
        
        // Timeout - cleanup
        redisTemplate.delete(participantKey);
        return false;
    }
    
    private String getParticipantId() {
        return ManagementFactory.getRuntimeMXBean().getName() + 
               "-" + Thread.currentThread().getId();
    }
}
```

### 3. CountDown Latches

#### Distributed CountDown Latch
```java
public class DistributedCountDownLatch {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private final String latchKey;
    private final int initialCount;
    
    public DistributedCountDownLatch(String latchId, int initialCount) {
        this.latchKey = "latch:" + latchId;
        this.initialCount = initialCount;
        
        // Initialize if not exists
        redisTemplate.opsForValue().setIfAbsent(latchKey, String.valueOf(initialCount));
    }
    
    public void countDown() {
        Long newCount = redisTemplate.opsForValue().decrement(latchKey);
        
        if (newCount == 0) {
            // Latch reached zero - notify waiting threads
            redisTemplate.convertAndSend("latch-channel:" + latchKey, "zero");
        }
    }
    
    public boolean await(Duration timeout) throws Exception {
        Instant startTime = Instant.now();
        
        while (Duration.between(startTime, Instant.now()).compareTo(timeout) < 0) {
            String countStr = (String) redisTemplate.opsForValue().get(latchKey);
            int count = Integer.parseInt(countStr);
            
            if (count <= 0) {
                return true; // Latch already at zero
            }
            
            // Wait for notification
            Thread.sleep(100);
        }
        
        return false; // Timeout
    }
    
    public int getCount() {
        String countStr = (String) redisTemplate.opsForValue().get(latchKey);
        return Integer.parseInt(countStr);
    }
}
```

## Coordination Algorithms

### 1. Leader Election

#### Bully Algorithm
```java
public class BullyLeaderElection {
    
    private final int processId;
    private final List<Integer> allProcesses;
    private volatile Integer currentLeader = null;
    
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
        
        logger.info("Process {} starting election", processId);
        
        // Send election messages to higher processes
        List<Integer> higherProcesses = allProcesses.stream()
            .filter(id -> id > processId)
            .collect(Collectors.toList());
        
        boolean higherProcessResponded = false;
        
        for (int higherProcess : higherProcesses) {
            try {
                boolean alive = sendElectionMessage(higherProcess);
                if (alive) {
                    higherProcessResponded = true;
                    break;
                }
            } catch (Exception e) {
                logger.warn("Process {} is not responding", higherProcess);
            }
        }
        
        if (!higherProcessResponded) {
            // No higher process responded - become leader
            becomeLeader();
        }
    }
    
    private void becomeLeader() {
        currentLeader = processId;
        logger.info("Process {} became leader", processId);
        
        // Announce leadership to all processes
        for (int process : allProcesses) {
            if (process != processId) {
                sendCoordinatorMessage(process);
            }
        }
    }
    
    public void handleElectionMessage(int fromProcess) {
        // Send OK message back
        sendOkMessage(fromProcess);
        
        // Start own election if not already leader
        if (currentLeader == null || currentLeader != processId) {
            startElection();
        }
    }
    
    public void handleCoordinatorMessage(int leaderId) {
        currentLeader = leaderId;
        logger.info("Process {} acknowledges leader {}", processId, leaderId);
    }
    
    // Message sending methods (simplified)
    private boolean sendElectionMessage(int targetProcess) {
        // Implementation would send actual network message
        return true; // Assume success for demo
    }
    
    private void sendOkMessage(int targetProcess) {
        // Send OK response
    }
    
    private void sendCoordinatorMessage(int targetProcess) {
        // Send coordinator announcement
    }
}
```

#### Ring Algorithm
```java
public class RingLeaderElection {
    
    private final int processId;
    private final List<Integer> ringProcesses;
    private volatile Integer currentLeader = null;
    private boolean participating = false;
    
    public RingLeaderElection(int processId, List<Integer> processes) {
        this.processId = processId;
        this.ringProcesses = new ArrayList<>(processes);
        // Sort to create ring order
        Collections.sort(this.ringProcesses);
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
        
        logger.info("Process {} became coordinator", processId);
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
}
```

### 2. Distributed Transactions

#### Two-Phase Commit (2PC)
```java
public class TwoPhaseCommitCoordinator {
    
    private final List<TransactionParticipant> participants;
    private final Map<String, Transaction> activeTransactions = new ConcurrentHashMap<>();
    
    public TransactionResult commit(Transaction transaction) {
        String transactionId = transaction.getId();
        activeTransactions.put(transactionId, transaction);
        
        try {
            // Phase 1: Prepare
            PrepareResult prepareResult = preparePhase(transaction);
            
            if (!prepareResult.isAllPrepared()) {
                // Prepare failed - abort transaction
                abortPhase(transaction);
                return TransactionResult.ABORTED;
            }
            
            // Phase 2: Commit
            CommitResult commitResult = commitPhase(transaction);
            
            if (commitResult.isAllCommitted()) {
                return TransactionResult.COMMITTED;
            } else {
                // Some participants failed to commit
                return TransactionResult.INCONSISTENT;
            }
            
        } catch (Exception e) {
            logger.error("Transaction {} failed", transactionId, e);
            abortPhase(transaction);
            return TransactionResult.ABORTED;
        } finally {
            activeTransactions.remove(transactionId);
        }
    }
    
    private PrepareResult preparePhase(Transaction transaction) {
        List<CompletableFuture<PrepareResponse>> futures = participants.stream()
            .map(participant -> participant.prepare(transaction))
            .collect(Collectors.toList());
        
        List<PrepareResponse> responses = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        
        boolean allPrepared = responses.stream()
            .allMatch(PrepareResponse::isPrepared);
        
        return new PrepareResult(allPrepared, responses);
    }
    
    private CommitResult commitPhase(Transaction transaction) {
        List<CompletableFuture<CommitResponse>> futures = participants.stream()
            .map(participant -> participant.commit(transaction))
            .collect(Collectors.toList());
        
        List<CommitResponse> responses = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        
        boolean allCommitted = responses.stream()
            .allMatch(CommitResponse::isCommitted);
        
        return new CommitResult(allCommitted, responses);
    }
    
    private void abortPhase(Transaction transaction) {
        participants.forEach(participant -> {
            try {
                participant.abort(transaction);
            } catch (Exception e) {
                logger.warn("Failed to abort transaction on participant", e);
            }
        });
    }
    
    // Recovery for coordinator failures
    public void recover() {
        // Check status of active transactions
        for (Transaction transaction : activeTransactions.values()) {
            // Query participants for transaction status
            // and complete accordingly
        }
    }
}
```

#### Saga Pattern
```java
public class SagaOrchestrator {
    
    private final List<SagaStep> sagaSteps;
    private final Map<String, SagaExecution> activeSagas = new ConcurrentHashMap<>();
    
    public SagaResult executeSaga(String sagaId, Object initialData) {
        SagaExecution execution = new SagaExecution(sagaId, sagaSteps.size());
        activeSagas.put(sagaId, execution);
        
        Object currentData = initialData;
        
        try {
            // Execute saga steps
            for (int i = 0; i < sagaSteps.size(); i++) {
                SagaStep step = sagaSteps.get(i);
                
                try {
                    currentData = step.execute(currentData);
                    execution.recordStepSuccess(i);
                    
                } catch (Exception e) {
                    logger.error("Saga step {} failed", i, e);
                    execution.recordStepFailure(i, e);
                    
                    // Compensate previous steps
                    compensateSaga(execution, i, currentData);
                    return SagaResult.failed("Step " + i + " failed: " + e.getMessage());
                }
            }
            
            execution.markCompleted();
            return SagaResult.success(currentData);
            
        } finally {
            activeSagas.remove(sagaId);
        }
    }
    
    private void compensateSaga(SagaExecution execution, int failedStepIndex, Object data) {
        // Compensate steps in reverse order
        for (int i = failedStepIndex - 1; i >= 0; i--) {
            if (execution.wasStepSuccessful(i)) {
                try {
                    sagaSteps.get(i).compensate(data);
                    execution.recordCompensationSuccess(i);
                } catch (Exception e) {
                    logger.error("Compensation failed for step {}", i, e);
                    execution.recordCompensationFailure(i, e);
                }
            }
        }
    }
    
    // Recovery for failed sagas
    public void recoverFailedSagas() {
        for (SagaExecution execution : activeSagas.values()) {
            if (execution.isFailed()) {
                // Retry or compensate based on failure type
                retryOrCompensate(execution);
            }
        }
    }
}
```

## Coordination Services

### Apache ZooKeeper

#### Service Implementation
```java
@Service
public class ZooKeeperCoordinator {
    
    private ZooKeeper zk;
    private final String connectionString;
    
    public ZooKeeperCoordinator(String connectionString) {
        this.connectionString = connectionString;
        connect();
    }
    
    private void connect() {
        try {
            zk = new ZooKeeper(connectionString, 3000, new Watcher() {
                @Override
                public void process(WatchedEvent event) {
                    // Handle connection events
                    if (event.getState() == KeeperState.SyncConnected) {
                        logger.info("Connected to ZooKeeper");
                    }
                }
            });
        } catch (Exception e) {
            throw new RuntimeException("Failed to connect to ZooKeeper", e);
        }
    }
    
    // Leader Election
    public void electLeader(String electionPath, LeaderCallback callback) {
        try {
            String myPath = zk.create(
                electionPath + "/candidate-", 
                getNodeId().getBytes(), 
                ZooDefs.Ids.OPEN_ACL_UNSAFE, 
                CreateMode.EPHEMERAL_SEQUENTIAL);
            
            List<String> candidates = zk.getChildren(electionPath, false);
            Collections.sort(candidates);
            
            if (myPath.equals(electionPath + "/" + candidates.get(0))) {
                callback.onElectedLeader();
            } else {
                // Watch predecessor
                String predecessor = candidates.get(
                    candidates.indexOf(myPath.substring(electionPath.length() + 1)) - 1);
                
                zk.exists(electionPath + "/" + predecessor, new Watcher() {
                    @Override
                    public void process(WatchedEvent event) {
                        if (event.getType() == EventType.NodeDeleted) {
                            electLeader(electionPath, callback); // Try again
                        }
                    }
                });
            }
            
        } catch (Exception e) {
            logger.error("Leader election failed", e);
        }
    }
    
    // Distributed Lock
    public boolean acquireLock(String lockPath, Duration timeout) throws Exception {
        String lockNode = zk.create(
            lockPath + "/lock-", 
            getNodeId().getBytes(), 
            ZooDefs.Ids.OPEN_ACL_UNSAFE, 
            CreateMode.EPHEMERAL_SEQUENTIAL);
        
        return tryAcquireLock(lockPath, lockNode, timeout);
    }
    
    private boolean tryAcquireLock(String lockPath, String lockNode, Duration timeout) 
            throws Exception {
        
        List<String> children = zk.getChildren(lockPath, false);
        Collections.sort(children);
        
        if (lockNode.equals(lockPath + "/" + children.get(0))) {
            // This node has the lock
            return true;
        }
        
        // Watch the node before us
        int myIndex = children.indexOf(lockNode.substring(lockPath.length() + 1));
        String predecessor = children.get(myIndex - 1);
        
        final CountDownLatch latch = new CountDownLatch(1);
        
        Stat stat = zk.exists(lockPath + "/" + predecessor, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == EventType.NodeDeleted) {
                    latch.countDown();
                }
            }
        });
        
        if (stat != null) {
            // Wait for predecessor to release lock
            boolean acquired = latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS);
            if (acquired) {
                return tryAcquireLock(lockPath, lockNode, timeout);
            }
        } else {
            // Predecessor doesn't exist, try again
            return tryAcquireLock(lockPath, lockNode, timeout);
        }
        
        return false;
    }
    
    public void releaseLock(String lockPath) throws Exception {
        zk.delete(lockPath + "/lock-" + getNodeId(), -1);
    }
    
    // Configuration Management
    public void setConfig(String configPath, String configData) throws Exception {
        if (zk.exists(configPath, false) != null) {
            zk.setData(configPath, configData.getBytes(), -1);
        } else {
            zk.create(configPath, configData.getBytes(), 
                     ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }
    }
    
    public String getConfig(String configPath, Watcher watcher) throws Exception {
        byte[] data = zk.getData(configPath, watcher, null);
        return new String(data);
    }
    
    private String getNodeId() {
        return ManagementFactory.getRuntimeMXBean().getName();
    }
}
```

### etcd

#### Service Implementation
```java
@Service
public class EtcdCoordinator {
    
    private Client etcdClient;
    private final String[] endpoints;
    
    public EtcdCoordinator(String[] endpoints) {
        this.endpoints = endpoints;
        connect();
    }
    
    private void connect() {
        etcdClient = Client.builder()
            .endpoints(endpoints)
            .build();
    }
    
    // Leader Election
    public void campaignForLeadership(String electionName, LeaderCallback callback) {
        Election election = etcdClient.getElectionClient();
        
        election.campaign(ByteSequence.from(electionName.getBytes()), 
                         ByteSequence.from(getNodeId().getBytes()))
            .thenAccept(leaderKey -> {
                logger.info("Elected leader for {}", electionName);
                callback.onElectedLeader();
                
                // Keep leadership alive
                keepLeadershipAlive(leaderKey, electionName);
            });
    }
    
    private void keepLeadershipAlive(Lease leaderLease, String electionName) {
        // Renew lease periodically to maintain leadership
        Executors.newSingleThreadScheduledExecutor()
            .scheduleAtFixedRate(() -> {
                try {
                    etcdClient.getLeaseClient().keepAliveOnce(leaderLease.getID());
                } catch (Exception e) {
                    logger.warn("Failed to renew leadership lease", e);
                }
            }, 0, 10, TimeUnit.SECONDS);
    }
    
    // Distributed Lock
    public boolean acquireLock(String lockName, Duration timeout) {
        Lock lock = etcdClient.getLockClient().lock(
            ByteSequence.from(lockName.getBytes()),
            LockOption.newBuilder()
                .withLeaseId(createLease(timeout))
                .build());
        
        return lock != null;
    }
    
    public void releaseLock(Lock lock) {
        etcdClient.getLockClient().unlock(lock.getKey());
    }
    
    private long createLease(Duration ttl) {
        try {
            Lease lease = etcdClient.getLeaseClient().grant(ttl.getSeconds());
            return lease.getID();
        } catch (Exception e) {
            throw new RuntimeException("Failed to create lease", e);
        }
    }
    
    // Key-Value Operations
    public void put(String key, String value) {
        etcdClient.getKVClient().put(
            ByteSequence.from(key.getBytes()),
            ByteSequence.from(value.getBytes())
        );
    }
    
    public String get(String key) {
        try {
            GetResponse response = etcdClient.getKVClient().get(
                ByteSequence.from(key.getBytes()));
            
            if (!response.getKvs().isEmpty()) {
                return response.getKvs().get(0).getValue().toString();
            }
            return null;
        } catch (Exception e) {
            logger.error("Failed to get key {}", key, e);
            return null;
        }
    }
    
    // Watch for changes
    public void watch(String key, Consumer<WatchEvent> eventHandler) {
        Watch watch = etcdClient.getWatchClient();
        
        watch.watch(ByteSequence.from(key.getBytes()), 
            WatchOption.newBuilder().build(),
            new Watch.Listener() {
                @Override
                public void onNext(WatchResponse response) {
                    response.getEvents().forEach(event -> {
                        eventHandler.accept(new WatchEvent(
                            event.getKeyValue().getKey().toString(),
                            event.getKeyValue().getValue().toString(),
                            event.getEventType()
                        ));
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
    
    private String getNodeId() {
        return ManagementFactory.getRuntimeMXBean().getName();
    }
}
```

## Coordination Patterns

### 1. Master-Worker Pattern

#### Implementation
```java
public class MasterWorkerCoordinator {
    
    @Autowired
    private TaskQueue taskQueue;
    
    @Autowired
    private ResultAggregator resultAggregator;
    
    @Autowired
    private WorkerRegistry workerRegistry;
    
    public void submitJob(Job job) {
        // Split job into tasks
        List<Task> tasks = splitJobIntoTasks(job);
        
        // Submit tasks to queue
        for (Task task : tasks) {
            taskQueue.submitTask(task);
        }
        
        // Wait for results
        List<TaskResult> results = collectResults(tasks, job.getTimeout());
        
        // Aggregate final result
        JobResult finalResult = resultAggregator.aggregate(results);
        
        // Notify job completion
        notifyJobCompletion(job, finalResult);
    }
    
    public void registerWorker(WorkerInfo workerInfo) {
        workerRegistry.register(workerInfo);
        
        // Start worker monitoring
        monitorWorker(workerInfo);
    }
    
    private void monitorWorker(WorkerInfo workerInfo) {
        Executors.newSingleThreadScheduledExecutor()
            .scheduleAtFixedRate(() -> {
                if (!workerRegistry.isAlive(workerInfo)) {
                    logger.warn("Worker {} is not responding", workerInfo.getId());
                    
                    // Reassign tasks from failed worker
                    reassignTasksFromWorker(workerInfo);
                    
                    // Remove from registry
                    workerRegistry.unregister(workerInfo);
                }
            }, 30, 30, TimeUnit.SECONDS);
    }
    
    private void reassignTasksFromWorker(WorkerInfo workerInfo) {
        List<Task> assignedTasks = taskQueue.getTasksForWorker(workerInfo.getId());
        
        for (Task task : assignedTasks) {
            // Mark task as available for reassignment
            taskQueue.requeueTask(task);
        }
    }
}
```

### 2. Publish-Subscribe Pattern

#### Distributed Event Bus
```java
public class DistributedEventBus {
    
    @Autowired
    private MessageBroker broker;
    
    @Autowired
    private SubscriberRegistry subscriberRegistry;
    
    public void publish(String topic, Object event) {
        // Get subscribers for topic
        List<Subscriber> subscribers = subscriberRegistry.getSubscribers(topic);
        
        // Publish to each subscriber
        for (Subscriber subscriber : subscribers) {
            try {
                broker.sendMessage(subscriber.getQueueName(), event);
            } catch (Exception e) {
                logger.warn("Failed to send message to subscriber {}", 
                           subscriber.getId(), e);
            }
        }
    }
    
    public void subscribe(String topic, Subscriber subscriber) {
        subscriberRegistry.addSubscriber(topic, subscriber);
        
        // Start consumer for subscriber
        startConsumer(subscriber);
    }
    
    public void unsubscribe(String topic, Subscriber subscriber) {
        subscriberRegistry.removeSubscriber(topic, subscriber);
        
        // Stop consumer for subscriber
        stopConsumer(subscriber);
    }
    
    private void startConsumer(Subscriber subscriber) {
        // Start message consumer for subscriber's queue
        broker.startConsumer(subscriber.getQueueName(), message -> {
            try {
                // Deserialize and deliver event
                Object event = deserializeEvent(message);
                subscriber.handleEvent(event);
            } catch (Exception e) {
                logger.error("Subscriber {} failed to handle event", 
                           subscriber.getId(), e);
            }
        });
    }
}
```

## Monitoring and Observability

### Coordination Metrics
```java
@Service
public class CoordinationMetrics {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter lockAcquisitions = Counter.builder("coordination_locks_acquired_total")
        .description("Total locks acquired")
        .register(meterRegistry);
    
    private final Counter lockAcquisitionFailures = Counter.builder("coordination_lock_failures_total")
        .description("Total lock acquisition failures")
        .register(meterRegistry);
    
    private final Histogram lockHoldTime = Histogram.builder("coordination_lock_hold_time")
        .description("Time locks are held")
        .register(meterRegistry);
    
    private final Counter leaderElections = Counter.builder("coordination_leader_elections_total")
        .description("Total leader elections")
        .register(meterRegistry);
    
    private final Gauge currentLeader = Gauge.builder("coordination_current_leader")
        .description("Current leader process ID")
        .register(meterRegistry);
    
    private final Counter transactionCommits = Counter.builder("coordination_transactions_committed_total")
        .description("Total transactions committed")
        .register(meterRegistry);
    
    private final Counter transactionAborts = Counter.builder("coordination_transactions_aborted_total")
        .description("Total transactions aborted")
        .register(meterRegistry);
    
    public void recordLockAcquired() {
        lockAcquisitions.increment();
    }
    
    public void recordLockAcquisitionFailure() {
        lockAcquisitionFailures.increment();
    }
    
    public void recordLockHoldTime(long holdTimeMs) {
        lockHoldTime.observe(holdTimeMs / 1000.0);
    }
    
    public void recordLeaderElection() {
        leaderElections.increment();
    }
    
    public void recordTransactionCommitted() {
        transactionCommits.increment();
    }
    
    public void recordTransactionAborted() {
        transactionAborts.increment();
    }
}
```

### Coordination Health Checks
```java
@Service
public class CoordinationHealthCheck implements HealthIndicator {
    
    @Autowired
    private CoordinationMetrics metrics;
    
    @Autowired
    private CoordinationService coordinationService;
    
    @Override
    public Health health() {
        try {
            // Check if coordination service is operational
            if (!coordinationService.isOperational()) {
                return Health.down()
                    .withDetail("coordination", "service not operational")
                    .build();
            }
            
            // Check leader election health
            if (!coordinationService.hasActiveLeader()) {
                return Health.down()
                    .withDetail("leader", "no active leader")
                    .build();
            }
            
            // Check lock service health
            if (!coordinationService.canAcquireLocks()) {
                return Health.down()
                    .withDetail("locks", "cannot acquire locks")
                    .build();
            }
            
            // Check transaction success rate
            double transactionSuccessRate = metrics.getTransactionSuccessRate();
            if (transactionSuccessRate < 0.95) { // Less than 95% success
                return Health.down()
                    .withDetail("transactions", 
                        String.format("success rate: %.2f%%", transactionSuccessRate * 100))
                    .build();
            }
            
            return Health.up()
                .withDetail("leader_id", coordinationService.getCurrentLeaderId())
                .withDetail("active_locks", coordinationService.getActiveLockCount())
                .withDetail("transaction_success_rate", 
                    String.format("%.2f%%", transactionSuccessRate * 100))
                .build();
                
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

## Conclusion

Distributed coordination is essential for building reliable, scalable distributed systems. By implementing proper coordination primitives and algorithms, systems can maintain consistency, avoid race conditions, and coordinate complex distributed operations.

**Key Takeaways:**
- **Coordination Primitives**: Locks, barriers, and latches for basic coordination
- **Leader Election**: Ensuring single leader for critical operations
- **Distributed Transactions**: Maintaining consistency across services
- **Coordination Services**: ZooKeeper, etcd for robust coordination
- **Monitoring**: Track coordination health and performance

Effective distributed coordination requires understanding the trade-offs between consistency, availability, and performance, and choosing the right coordination mechanisms for each use case.
