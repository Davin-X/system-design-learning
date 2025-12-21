# Distributed Locking

Distributed locking ensures mutual exclusion across multiple processes or nodes in a distributed system. It prevents race conditions and ensures data consistency when multiple processes need to access shared resources. This guide covers distributed locking patterns, implementations, and best practices.

## What is Distributed Locking?

Distributed locking provides mutual exclusion across different processes or nodes in a distributed system. Unlike single-process locks (synchronized, ReentrantLock), distributed locks work across process boundaries and survive process restarts.

### Key Concepts
- **Mutual Exclusion**: Only one process holds the lock at a time
- **Visibility**: Lock state is visible across all processes
- **Atomicity**: Lock operations are atomic
- **Fault Tolerance**: Locks survive partial system failures

### Distributed Locking Challenges
- **Network Delays**: Lock acquisition/release may be delayed
- **Process Failures**: Locks must be released when processes die
- **Clock Skew**: Time-based locks may be unreliable
- **Split Brain**: Network partitions may cause multiple locks

## Lock Implementation Patterns

### 1. Redis-Based Distributed Locks

#### Basic Redis Lock
```java
@Service
public class RedisDistributedLock {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final String LOCK_PREFIX = "lock:";
    private final String lockValue;
    
    public RedisDistributedLock() {
        // Use unique identifier for this instance
        this.lockValue = UUID.randomUUID().toString();
    }
    
    public boolean tryLock(String resourceKey, Duration timeout) {
        String lockKey = LOCK_PREFIX + resourceKey;
        Long result = redisTemplate.opsForValue().setIfAbsent(
            lockKey, 
            lockValue, 
            timeout
        );
        
        return Boolean.TRUE.equals(result);
    }
    
    public boolean unlock(String resourceKey) {
        String lockKey = LOCK_PREFIX + resourceKey;
        
        // Use Lua script to ensure atomic check-and-delete
        String script = 
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(lockKey),
            lockValue
        );
        
        return result != null && result > 0;
    }
    
    public boolean isLocked(String resourceKey) {
        String lockKey = LOCK_PREFIX + resourceKey;
        String currentValue = (String) redisTemplate.opsForValue().get(lockKey);
        return currentValue != null;
    }
    
    public boolean isHeldByCurrentInstance(String resourceKey) {
        String lockKey = LOCK_PREFIX + resourceKey;
        String currentValue = (String) redisTemplate.opsForValue().get(lockKey);
        return lockValue.equals(currentValue);
    }
}
```

#### Redlock Algorithm
```java
public class RedlockDistributedLock {
    
    private final List<RedisTemplate<String, Object>> redisInstances;
    private final int quorumSize;
    private final Duration lockValidityTime;
    private final String lockValue;
    
    public RedlockDistributedLock(List<RedisTemplate<String, Object>> redisInstances, 
                                 Duration lockValidityTime) {
        this.redisInstances = redisInstances;
        this.quorumSize = (redisInstances.size() / 2) + 1; // Majority
        this.lockValidityTime = lockValidityTime;
        this.lockValue = UUID.randomUUID().toString();
    }
    
    public boolean tryLock(String resourceKey, Duration acquireTimeout) {
        long startTime = System.currentTimeMillis();
        String lockKey = "redlock:" + resourceKey;
        
        while (System.currentTimeMillis() - startTime < acquireTimeout.toMillis()) {
            int lockedInstances = 0;
            
            for (RedisTemplate<String, Object> redis : redisInstances) {
                try {
                    Boolean acquired = redis.opsForValue().setIfAbsent(
                        lockKey, lockValue, lockValidityTime);
                    
                    if (Boolean.TRUE.equals(acquired)) {
                        lockedInstances++;
                    }
                } catch (Exception e) {
                    // Redis instance unavailable
                    logger.warn("Redis instance unavailable", e);
                }
            }
            
            // Check if we have quorum
            if (lockedInstances >= quorumSize) {
                return true;
            } else {
                // Release locks we acquired
                unlock(resourceKey);
                
                // Wait before retry
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return false;
                }
            }
        }
        
        return false;
    }
    
    public void unlock(String resourceKey) {
        String lockKey = "redlock:" + resourceKey;
        
        for (RedisTemplate<String, Object> redis : redisInstances) {
            try {
                String script = 
                    "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                    "    return redis.call('del', KEYS[1]) " +
                    "else " +
                    "    return 0 " +
                    "end";
                
                redis.execute(new DefaultRedisScript<>(script, Long.class),
                    Collections.singletonList(lockKey), lockValue);
            } catch (Exception e) {
                logger.warn("Failed to release lock on Redis instance", e);
            }
        }
    }
}
```

### 2. ZooKeeper-Based Distributed Locks

#### Sequential Node Locks
```java
@Service
public class ZooKeeperDistributedLock {
    
    private ZooKeeper zk;
    private final String locksPath;
    private final String nodeId;
    
    public ZooKeeperDistributedLock(ZooKeeper zk, String locksPath, String nodeId) {
        this.zk = zk;
        this.locksPath = locksPath;
        this.nodeId = nodeId;
    }
    
    public boolean tryLock(String resourceKey, Duration timeout) throws Exception {
        String resourcePath = locksPath + "/" + resourceKey;
        
        // Create resource lock directory if it doesn't exist
        if (zk.exists(resourcePath, false) == null) {
            zk.create(resourcePath, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }
        
        // Create sequential ephemeral node
        String lockNode = zk.create(
            resourcePath + "/lock-", 
            nodeId.getBytes(), 
            ZooDefs.Ids.OPEN_ACL_UNSAFE, 
            CreateMode.EPHEMERAL_SEQUENTIAL);
        
        // Check if we got the lock
        return tryAcquireLock(resourcePath, lockNode, timeout);
    }
    
    private boolean tryAcquireLock(String resourcePath, String lockNode, Duration timeout) 
            throws Exception {
        
        List<String> children = zk.getChildren(resourcePath, false);
        Collections.sort(children);
        
        String myNode = lockNode.substring(lockNode.lastIndexOf('/') + 1);
        int myIndex = children.indexOf(myNode);
        
        if (myIndex == 0) {
            // We have the lock
            return true;
        } else {
            // Watch the node before us
            String predecessor = children.get(myIndex - 1);
            return watchPredecessor(resourcePath + "/" + predecessor, timeout);
        }
    }
    
    private boolean watchPredecessor(String predecessorPath, Duration timeout) throws Exception {
        final CountDownLatch latch = new CountDownLatch(1);
        
        Stat stat = zk.exists(predecessorPath, event -> {
            if (event.getType() == EventType.NodeDeleted) {
                latch.countDown();
            }
        });
        
        if (stat == null) {
            // Predecessor doesn't exist, we got the lock
            return true;
        }
        
        // Wait for predecessor to release lock or timeout
        boolean acquired = latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS);
        
        if (acquired) {
            // Predecessor released, try to acquire again
            String resourcePath = predecessorPath.substring(0, predecessorPath.lastIndexOf('/'));
            String myLockNode = getMyLockNode(resourcePath);
            return tryAcquireLock(resourcePath, myLockNode, Duration.ofMillis(100));
        }
        
        return false;
    }
    
    public void unlock(String resourceKey) throws Exception {
        String resourcePath = locksPath + "/" + resourceKey;
        String myLockNode = getMyLockNode(resourcePath);
        
        if (myLockNode != null) {
            zk.delete(myLockNode, -1);
        }
    }
    
    private String getMyLockNode(String resourcePath) throws Exception {
        List<String> children = zk.getChildren(resourcePath, false);
        
        for (String child : children) {
            String childPath = resourcePath + "/" + child;
            byte[] data = zk.getData(childPath, false, null);
            String ownerId = new String(data);
            
            if (nodeId.equals(ownerId)) {
                return childPath;
            }
        }
        
        return null;
    }
}
```

### 3. etcd-Based Distributed Locks

#### Lease-Based Locks
```java
@Service
public class EtcdDistributedLock {
    
    private Client etcdClient;
    private final String locksPrefix;
    private final String nodeId;
    private final Map<String, Long> activeLeases = new ConcurrentHashMap<>();
    
    public EtcdDistributedLock(Client etcdClient, String locksPrefix, String nodeId) {
        this.etcdClient = etcdClient;
        this.locksPrefix = locksPrefix;
        this.nodeId = nodeId;
    }
    
    public boolean tryLock(String resourceKey, Duration timeout) throws Exception {
        String lockKey = locksPrefix + "/" + resourceKey;
        
        // Create lease
        Lease lease = etcdClient.getLeaseClient().grant(timeout.getSeconds() + 5); // Extra time
        long leaseId = lease.getID();
        
        // Try to acquire lock
        ByteSequence key = ByteSequence.from(lockKey.getBytes());
        ByteSequence value = ByteSequence.from((nodeId + ":" + leaseId).getBytes());
        
        Txn txn = Txn.newBuilder()
            .If(new Cmp(key, Cmp.Op.EQUAL, CmpTarget.version(0)))
            .Then(Op.put(key, value, PutOption.newBuilder().withLeaseId(leaseId).build()))
            .Else(Op.get(key))
            .build();
        
        CompletableFuture<TxnResponse> future = etcdClient.getKVClient().txn(txn);
        TxnResponse response = future.get(timeout.toMillis(), TimeUnit.MILLISECONDS);
        
        if (response.getGetResponses().isEmpty()) {
            // Successfully acquired lock
            activeLeases.put(resourceKey, leaseId);
            keepLeaseAlive(leaseId);
            return true;
        } else {
            // Lock already held, revoke lease
            etcdClient.getLeaseClient().revoke(leaseId);
            return false;
        }
    }
    
    public void unlock(String resourceKey) {
        Long leaseId = activeLeases.remove(resourceKey);
        if (leaseId != null) {
            try {
                etcdClient.getLeaseClient().revoke(leaseId);
            } catch (Exception e) {
                logger.warn("Failed to revoke lease for lock {}", resourceKey, e);
            }
        }
    }
    
    private void keepLeaseAlive(long leaseId) {
        Executors.newSingleThreadScheduledExecutor()
            .scheduleAtFixedRate(() -> {
                try {
                    etcdClient.getLeaseClient().keepAliveOnce(leaseId);
                } catch (Exception e) {
                    logger.warn("Failed to keep lease alive: {}", leaseId);
                }
            }, 2, 2, TimeUnit.SECONDS); // Keep alive every 2 seconds
    }
}
```

## Lock Management Patterns

### 1. Lock Hierarchy Pattern

#### Nested Locks with Deadlock Prevention
```java
public class LockHierarchyManager {
    
    private final DistributedLockService lockService;
    private final ThreadLocal<Deque<String>> lockStack = new ThreadLocal<>();
    
    public LockHierarchyManager(DistributedLockService lockService) {
        this.lockService = lockService;
    }
    
    public boolean acquireLocks(List<String> resourceKeys, Duration timeout) throws Exception {
        // Sort resources to prevent deadlocks
        List<String> sortedKeys = resourceKeys.stream()
            .sorted()
            .collect(Collectors.toList());
        
        Deque<String> acquiredLocks = new ArrayDeque<>();
        long startTime = System.currentTimeMillis();
        
        try {
            for (String resourceKey : sortedKeys) {
                long remainingTimeout = timeout.toMillis() - 
                    (System.currentTimeMillis() - startTime);
                
                if (remainingTimeout <= 0) {
                    throw new TimeoutException("Lock acquisition timed out");
                }
                
                boolean acquired = lockService.tryLock(resourceKey, 
                    Duration.ofMillis(remainingTimeout));
                
                if (!acquired) {
                    throw new LockAcquisitionException("Failed to acquire lock for " + resourceKey);
                }
                
                acquiredLocks.push(resourceKey);
                getLockStack().push(resourceKey);
            }
            
            return true;
            
        } catch (Exception e) {
            // Release locks in reverse order
            while (!acquiredLocks.isEmpty()) {
                String resourceKey = acquiredLocks.pop();
                try {
                    lockService.unlock(resourceKey);
                } catch (Exception unlockException) {
                    logger.error("Failed to release lock {}", resourceKey, unlockException);
                }
            }
            throw e;
        }
    }
    
    public void releaseLocks() {
        Deque<String> stack = getLockStack();
        while (!stack.isEmpty()) {
            String resourceKey = stack.pop();
            try {
                lockService.unlock(resourceKey);
            } catch (Exception e) {
                logger.error("Failed to release lock {}", resourceKey, e);
            }
        }
    }
    
    private Deque<String> getLockStack() {
        Deque<String> stack = lockStack.get();
        if (stack == null) {
            stack = new ArrayDeque<>();
            lockStack.set(stack);
        }
        return stack;
    }
}
```

### 2. Lock Timeout and Renewal

#### Automatic Lock Renewal
```java
public class LockRenewalManager {
    
    private final DistributedLockService lockService;
    private final ScheduledExecutorService renewalExecutor = Executors.newScheduledThreadPool(5);
    private final Map<String, ScheduledFuture<?>> renewalTasks = new ConcurrentHashMap<>();
    
    public boolean acquireLockWithRenewal(String resourceKey, Duration initialTimeout, 
                                         Duration renewalInterval) throws Exception {
        
        boolean acquired = lockService.tryLock(resourceKey, initialTimeout);
        
        if (acquired) {
            // Schedule renewal
            ScheduledFuture<?> renewalTask = renewalExecutor.scheduleAtFixedRate(
                () -> renewLock(resourceKey, initialTimeout),
                renewalInterval.toMillis(),
                renewalInterval.toMillis(),
                TimeUnit.MILLISECONDS
            );
            
            renewalTasks.put(resourceKey, renewalTask);
        }
        
        return acquired;
    }
    
    public void releaseLock(String resourceKey) {
        // Cancel renewal task
        ScheduledFuture<?> task = renewalTasks.remove(resourceKey);
        if (task != null) {
            task.cancel(false);
        }
        
        // Release lock
        try {
            lockService.unlock(resourceKey);
        } catch (Exception e) {
            logger.error("Failed to release lock {}", resourceKey, e);
        }
    }
    
    private void renewLock(String resourceKey, Duration extension) {
        try {
            // Try to extend lock
            boolean renewed = lockService.extendLock(resourceKey, extension);
            
            if (!renewed) {
                logger.warn("Failed to renew lock {}, may lose lock soon", resourceKey);
            }
        } catch (Exception e) {
            logger.error("Error renewing lock {}", resourceKey, e);
        }
    }
}
```

### 3. Read-Write Locks

#### Distributed Read-Write Locks
```java
public class DistributedReadWriteLock {
    
    private final DistributedLockService lockService;
    private final String readLockPrefix = "readlock:";
    private final String writeLockPrefix = "writelock:";
    private final AtomicInteger readLockCount = new AtomicInteger(0);
    
    public DistributedReadWriteLock(DistributedLockService lockService) {
        this.lockService = lockService;
    }
    
    public boolean tryReadLock(String resourceKey, Duration timeout) throws Exception {
        String readLockKey = readLockPrefix + resourceKey;
        String writeLockKey = writeLockPrefix + resourceKey;
        
        // Check if write lock is held
        if (lockService.isLocked(writeLockKey)) {
            return false;
        }
        
        // Acquire read lock
        boolean acquired = lockService.tryLock(readLockKey, timeout);
        
        if (acquired) {
            readLockCount.incrementAndGet();
        }
        
        return acquired;
    }
    
    public boolean tryWriteLock(String resourceKey, Duration timeout) throws Exception {
        String readLockKey = readLockPrefix + resourceKey;
        String writeLockKey = writeLockPrefix + resourceKey;
        
        // Check if any read locks are held
        if (readLockCount.get() > 0) {
            return false;
        }
        
        // Acquire write lock
        return lockService.tryLock(writeLockKey, timeout);
    }
    
    public void releaseReadLock(String resourceKey) throws Exception {
        String readLockKey = readLockPrefix + resourceKey;
        
        lockService.unlock(readLockKey);
        readLockCount.decrementAndGet();
    }
    
    public void releaseWriteLock(String resourceKey) throws Exception {
        String writeLockKey = writeLockPrefix + resourceKey;
        
        lockService.unlock(writeLockKey);
    }
}
```

## Lock Performance Optimization

### 1. Lock Granularity

#### Fine-Grained vs Coarse-Grained Locking
```java
public class LockGranularityExample {
    
    @Autowired
    private DistributedLockService lockService;
    
    // Coarse-grained locking (locks entire user)
    public void updateUserCoarseGrained(Long userId, UserUpdate update) throws Exception {
        String lockKey = "user:" + userId;
        
        lockService.tryLock(lockKey, Duration.ofSeconds(30));
        try {
            // Update all user data
            updateUserProfile(userId, update.getProfile());
            updateUserPreferences(userId, update.getPreferences());
            updateUserSettings(userId, update.getSettings());
        } finally {
            lockService.unlock(lockKey);
        }
    }
    
    // Fine-grained locking (locks specific resources)
    public void updateUserFineGrained(Long userId, UserUpdate update) throws Exception {
        List<String> lockKeys = Arrays.asList(
            "user:" + userId + ":profile",
            "user:" + userId + ":preferences", 
            "user:" + userId + ":settings"
        );
        
        // Acquire locks in sorted order to prevent deadlocks
        lockKeys.sort(String::compareTo);
        
        for (String lockKey : lockKeys) {
            lockService.tryLock(lockKey, Duration.ofSeconds(30));
        }
        
        try {
            // Update specific parts concurrently if possible
            CompletableFuture<Void> profileFuture = CompletableFuture.runAsync(() -> 
                updateUserProfile(userId, update.getProfile()));
            
            CompletableFuture<Void> preferencesFuture = CompletableFuture.runAsync(() -> 
                updateUserPreferences(userId, update.getPreferences()));
            
            CompletableFuture<Void> settingsFuture = CompletableFuture.runAsync(() -> 
                updateUserSettings(userId, update.getSettings()));
            
            // Wait for all updates to complete
            CompletableFuture.allOf(profileFuture, preferencesFuture, settingsFuture).get();
            
        } finally {
            // Release locks in reverse order
            Collections.reverse(lockKeys);
            for (String lockKey : lockKeys) {
                lockService.unlock(lockKey);
            }
        }
    }
}
```

### 2. Lock Contention Reduction

#### Optimistic Locking with Versioning
```java
@Entity
public class VersionedEntity {
    
    @Id
    private Long id;
    
    @Version
    private Long version;
    
    // Other fields...
}

@Service
public class OptimisticLockingService {
    
    @Autowired
    private VersionedEntityRepository repository;
    
    public void updateEntityOptimistically(Long entityId, EntityUpdate update) {
        int maxRetries = 3;
        
        for (int attempt = 0; attempt < maxRetries; attempt++) {
            try {
                VersionedEntity entity = repository.findById(entityId)
                    .orElseThrow(() -> new EntityNotFoundException("Entity not found"));
                
                // Apply update
                applyUpdate(entity, update);
                
                // Save with version check
                repository.save(entity);
                
                return; // Success
                
            } catch (OptimisticLockingFailureException e) {
                if (attempt == maxRetries - 1) {
                    throw new ConcurrentUpdateException("Entity was modified by another transaction", e);
                }
                
                // Retry after brief delay
                try {
                    Thread.sleep(100 * (attempt + 1));
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException(ie);
                }
            }
        }
    }
}
```

## Monitoring Distributed Locks

### Lock Metrics
```java
@Service
public class DistributedLockMetrics {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter lockAcquisitions = Counter.builder("distributed_locks_acquired_total")
        .description("Total distributed locks acquired")
        .register(meterRegistry);
    
    private final Counter lockAcquisitionFailures = Counter.builder("distributed_lock_failures_total")
        .description("Total distributed lock acquisition failures")
        .register(meterRegistry);
    
    private final Histogram lockHoldTime = Histogram.builder("distributed_lock_hold_time")
        .description("Time distributed locks are held")
        .register(meterRegistry);
    
    private final Gauge activeLocks = Gauge.builder("distributed_locks_active")
        .description("Number of active distributed locks")
        .register(meterRegistry);
    
    private final Histogram lockWaitTime = Histogram.builder("distributed_lock_wait_time")
        .description("Time spent waiting for distributed locks")
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
    
    public void recordLockWaitTime(long waitTimeMs) {
        lockWaitTime.observe(waitTimeMs / 1000.0);
    }
}
```

### Lock Health Checks
```java
@Service
public class LockHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DistributedLockService lockService;
    
    @Autowired
    private DistributedLockMetrics metrics;
    
    @Override
    public Health health() {
        try {
            // Test lock acquisition and release
            String testLockKey = "health-check-lock-" + System.currentTimeMillis();
            
            boolean acquired = lockService.tryLock(testLockKey, Duration.ofSeconds(5));
            if (!acquired) {
                return Health.down()
                    .withDetail("lock-acquisition", "failed")
                    .build();
            }
            
            boolean released = lockService.unlock(testLockKey);
            if (!released) {
                return Health.down()
                    .withDetail("lock-release", "failed")
                    .build();
            }
            
            // Check lock metrics
            double failureRate = metrics.getLockFailureRate();
            if (failureRate > 0.05) { // More than 5% failures
                return Health.down()
                    .withDetail("failure-rate", String.format("%.2f%%", failureRate * 100))
                    .build();
            }
            
            double avgWaitTime = metrics.getAverageLockWaitTime();
            if (avgWaitTime > 1000) { // More than 1 second average wait
                return Health.down()
                    .withDetail("avg-wait-time", String.format("%.2fms", avgWaitTime))
                    .build();
            }
            
            return Health.up()
                .withDetail("failure-rate", String.format("%.2f%%", failureRate * 100))
                .withDetail("avg-wait-time", String.format("%.2fms", avgWaitTime))
                .build();
                
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

## Best Practices

### Lock Design
- **Minimal Lock Scope**: Hold locks for shortest time possible
- **Avoid Nested Locks**: Prevent deadlock scenarios
- **Use Timeouts**: Always specify lock timeouts
- **Graceful Degradation**: Handle lock failures gracefully

### Lock Implementation
- **Idempotent Operations**: Make operations safe to retry
- **Atomic Operations**: Use atomic lock acquisition and release
- **Lease Management**: Use leases to handle process failures
- **Monitoring**: Track lock usage and contention

### Operational Considerations
- **Lock Contention**: Monitor and optimize high-contention locks
- **Deadlock Detection**: Implement deadlock detection and resolution
- **Backup Strategies**: Have fallback mechanisms when locks fail
- **Testing**: Thoroughly test locking behavior under failure scenarios

## Real-World Examples

### Database Migration Locking
```java
@Service
public class DatabaseMigrationService {
    
    @Autowired
    private DistributedLockService lockService;
    
    @Autowired
    private MigrationRepository migrationRepository;
    
    public void runMigration(String migrationId, MigrationScript script) throws Exception {
        String lockKey = "migration:" + migrationId;
        
        // Acquire distributed lock to prevent concurrent migrations
        boolean acquired = lockService.tryLock(lockKey, Duration.ofMinutes(30));
        
        if (!acquired) {
            throw new MigrationException("Another migration is already running for " + migrationId);
        }
        
        try {
            logger.info("Starting migration {}", migrationId);
            
            // Check if migration already completed
            if (migrationRepository.isMigrationCompleted(migrationId)) {
                logger.info("Migration {} already completed", migrationId);
                return;
            }
            
            // Run migration
            script.execute();
            
            // Mark as completed
            migrationRepository.markMigrationCompleted(migrationId);
            
            logger.info("Migration {} completed successfully", migrationId);
            
        } finally {
            lockService.unlock(lockKey);
        }
    }
}
```

### Job Scheduling with Distributed Locks
```java
@Service
public class DistributedJobScheduler {
    
    @Autowired
    private DistributedLockService lockService;
    
    @Autowired
    private JobRepository jobRepository;
    
    @Scheduled(fixedRate = 60000) // Check every minute
    public void scheduleJobs() {
        List<Job> pendingJobs = jobRepository.findPendingJobs();
        
        for (Job job : pendingJobs) {
            String lockKey = "job:" + job.getId();
            
            try {
                // Try to acquire lock for this job
                boolean acquired = lockService.tryLock(lockKey, Duration.ofSeconds(30));
                
                if (acquired) {
                    try {
                        // Double-check job status
                        Job currentJob = jobRepository.findById(job.getId());
                        if (currentJob.getStatus() == JobStatus.PENDING) {
                            // Execute job
                            executeJob(currentJob);
                            
                            // Update status
                            currentJob.setStatus(JobStatus.COMPLETED);
                            jobRepository.save(currentJob);
                        }
                    } finally {
                        lockService.unlock(lockKey);
                    }
                }
                
            } catch (Exception e) {
                logger.error("Failed to schedule job {}", job.getId(), e);
            }
        }
    }
    
    private void executeJob(Job job) {
        // Execute job logic
        logger.info("Executing job {}", job.getId());
        // ... job execution code ...
    }
}
```

## Conclusion

Distributed locking is essential for maintaining data consistency and preventing race conditions in distributed systems. By implementing proper locking mechanisms and following best practices, systems can safely coordinate access to shared resources across multiple processes and nodes.

**Key Takeaways:**
- **Choose Right Implementation**: Redis, ZooKeeper, or etcd based on requirements
- **Handle Failures**: Implement timeouts, retries, and lease mechanisms
- **Monitor Performance**: Track lock contention and acquisition times
- **Avoid Deadlocks**: Use proper lock ordering and timeouts
- **Test Thoroughly**: Test locking behavior under various failure scenarios

Effective distributed locking ensures system reliability while maintaining performance and fault tolerance.
