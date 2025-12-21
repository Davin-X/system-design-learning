# Java Concurrency Patterns

Concurrency is essential for building high-performance, scalable Java applications that can handle multiple simultaneous operations efficiently. This guide covers essential concurrency patterns, thread safety techniques, and performance optimization strategies for Java backend development.

## Understanding Java Memory Model

### 1. Happens-Before Relationship

```java
public class HappensBeforeExample {
    
    private int counter = 0;
    private volatile boolean ready = false;
    
    // Thread 1: Writer
    public void writer() {
        counter = 42;        // Write operation
        ready = true;        // Volatile write - establishes happens-before
    }
    
    // Thread 2: Reader
    public void reader() {
        if (ready) {         // Volatile read - observes happens-before
            // Guaranteed to see counter = 42
            System.out.println(counter);
        }
    }
    
    // Alternative with synchronized
    public synchronized void synchronizedWriter() {
        counter = 42;
        ready = true;
        // Monitor release establishes happens-before
    }
    
    public synchronized void synchronizedReader() {
        if (ready) {
            // Guaranteed to see all writes from synchronizedWriter
            System.out.println(counter);
        }
    }
}
```

### 2. Atomic Operations

```java
public class AtomicOperations {
    
    // Not thread-safe
    private int nonAtomicCounter = 0;
    
    public void incrementNonAtomic() {
        nonAtomicCounter++; // Read-modify-write race condition
    }
    
    // Thread-safe with synchronized
    private int synchronizedCounter = 0;
    
    public synchronized void incrementSynchronized() {
        synchronizedCounter++;
    }
    
    // Thread-safe with AtomicInteger
    private AtomicInteger atomicCounter = new AtomicInteger(0);
    
    public void incrementAtomic() {
        atomicCounter.incrementAndGet(); // Atomic operation
    }
    
    // Complex atomic operations
    public void complexAtomicOperation() {
        // Compare and set
        int currentValue;
        int newValue;
        do {
            currentValue = atomicCounter.get();
            newValue = currentValue * 2;
        } while (!atomicCounter.compareAndSet(currentValue, newValue));
        
        // Atomic field updater for volatile fields
        AtomicIntegerFieldUpdater<AtomicOperations> updater = 
            AtomicIntegerFieldUpdater.newUpdater(AtomicOperations.class, "volatileField");
        
        updater.incrementAndGet(this);
    }
    
    private volatile int volatileField = 0;
}
```

## Thread Safety Patterns

### 1. Immutable Objects

```java
// Immutable class - thread-safe by design
@Value // Lombok annotation for immutable class
public class ImmutableUser {
    
    private final Long id;
    private final String username;
    private final String email;
    private final Instant createdAt;
    private final List<String> roles; // Defensive copy needed
    
    public ImmutableUser(Long id, String username, String email, Instant createdAt, List<String> roles) {
        this.id = id;
        this.username = Objects.requireNonNull(username);
        this.email = Objects.requireNonNull(email);
        this.createdAt = Objects.requireNonNull(createdAt);
        this.roles = List.copyOf(roles); // Defensive copy
    }
    
    // Only getters, no setters
    public Long getId() { return id; }
    public String getUsername() { return username; }
    public String getEmail() { return email; }
    public Instant getCreatedAt() { return createdAt; }
    public List<String> getRoles() { return roles; } // Returns immutable copy
    
    // Factory method for creating modified instances
    public ImmutableUser withUsername(String newUsername) {
        return new ImmutableUser(id, newUsername, email, createdAt, roles);
    }
    
    public ImmutableUser withRoles(List<String> newRoles) {
        return new ImmutableUser(id, username, email, createdAt, newRoles);
    }
}

// Builder pattern for complex immutable objects
public class UserBuilder {
    
    private Long id;
    private String username;
    private String email;
    private Instant createdAt;
    private List<String> roles = new ArrayList<>();
    
    public UserBuilder id(Long id) {
        this.id = id;
        return this;
    }
    
    public UserBuilder username(String username) {
        this.username = username;
        return this;
    }
    
    public UserBuilder email(String email) {
        this.email = email;
        return this;
    }
    
    public UserBuilder createdAt(Instant createdAt) {
        this.createdAt = createdAt;
        return this;
    }
    
    public UserBuilder addRole(String role) {
        this.roles.add(role);
        return this;
    }
    
    public ImmutableUser build() {
        return new ImmutableUser(id, username, email, createdAt, roles);
    }
}
```

### 2. Thread Confinement

```java
public class ThreadConfinementExamples {
    
    // Thread-local storage
    private static final ThreadLocal<UserContext> userContext = 
        new ThreadLocal<UserContext>() {
            @Override
            protected UserContext initialValue() {
                return new UserContext();
            }
        };
    
    public void processUserRequest() {
        // Set user context for this thread
        userContext.get().setUserId(getCurrentUserId());
        userContext.get().setRequestId(UUID.randomUUID().toString());
        
        try {
            // Process request - all methods can access userContext safely
            validateRequest();
            processBusinessLogic();
            sendResponse();
        } finally {
            // Clean up thread-local storage
            userContext.remove();
        }
    }
    
    private void validateRequest() {
        String userId = userContext.get().getUserId();
        // Validation logic using userId
    }
    
    private void processBusinessLogic() {
        String requestId = userContext.get().getRequestId();
        // Business logic using requestId
    }
    
    private void sendResponse() {
        UserContext context = userContext.get();
        // Send response using context
    }
    
    // Stack confinement (method-local variables)
    public void stackConfinementExample() {
        // These variables are confined to this method's stack
        List<String> localList = new ArrayList<>();
        Map<String, Object> localMap = new HashMap<>();
        
        // Safe to use without synchronization
        localList.add("item1");
        localMap.put("key1", "value1");
        
        // Pass to other methods (but don't store references)
        processLocalData(localList, localMap);
    }
    
    private void processLocalData(List<String> list, Map<String, Object> map) {
        // Safe because references are not escaped from stack confinement
        list.add("item2");
        map.put("key2", "value2");
    }
    
    public static class UserContext {
        private String userId;
        private String requestId;
        private Instant startTime = Instant.now();
        
        // Getters and setters
        public String getUserId() { return userId; }
        public void setUserId(String userId) { this.userId = userId; }
        
        public String getRequestId() { return requestId; }
        public void setRequestId(String requestId) { this.requestId = requestId; }
        
        public Instant getStartTime() { return startTime; }
    }
}
```

### 3. Guarded Blocks

```java
public class GuardedBlocks {
    
    // Condition-based synchronization
    private final Object lock = new Object();
    private boolean condition = false;
    private String sharedData;
    
    public void producer() throws InterruptedException {
        synchronized (lock) {
            while (condition) {
                lock.wait(); // Wait until condition becomes false
            }
            
            // Critical section
            sharedData = "Produced data: " + System.currentTimeMillis();
            condition = true;
            
            lock.notifyAll(); // Wake up waiting consumers
        }
    }
    
    public String consumer() throws InterruptedException {
        synchronized (lock) {
            while (!condition) {
                lock.wait(); // Wait until condition becomes true
            }
            
            // Critical section
            String data = sharedData;
            condition = false;
            
            lock.notifyAll(); // Wake up waiting producers
            
            return data;
        }
    }
    
    // Using Condition objects (more flexible than wait/notify)
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final Queue<String> queue = new LinkedList<>();
    private final int capacity = 10;
    
    public void put(String item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await(); // Wait until queue is not full
            }
            
            queue.add(item);
            notEmpty.signal(); // Signal that queue is not empty
        } finally {
            lock.unlock();
        }
    }
    
    public String take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await(); // Wait until queue is not empty
            }
            
            String item = queue.remove();
            notFull.signal(); // Signal that queue is not full
            
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

## Producer-Consumer Pattern

### 1. Blocking Queue Implementation

```java
public class ProducerConsumerExample {
    
    private final BlockingQueue<Task> taskQueue = new LinkedBlockingQueue<>(100);
    private final ExecutorService executor = Executors.newFixedThreadPool(10);
    private volatile boolean shutdown = false;
    
    public void start() {
        // Start consumer threads
        for (int i = 0; i < 5; i++) {
            executor.submit(this::consumeTasks);
        }
    }
    
    public void submitTask(Task task) throws InterruptedException {
        if (shutdown) {
            throw new IllegalStateException("Producer-Consumer is shut down");
        }
        
        taskQueue.put(task); // Blocks if queue is full
    }
    
    public void shutdown() {
        shutdown = true;
        executor.shutdown();
        
        try {
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
    
    private void consumeTasks() {
        while (!shutdown || !taskQueue.isEmpty()) {
            try {
                Task task = taskQueue.poll(1, TimeUnit.SECONDS); // Wait up to 1 second
                
                if (task != null) {
                    processTask(task);
                }
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            } catch (Exception e) {
                // Log error and continue processing
                System.err.println("Error processing task: " + e.getMessage());
            }
        }
    }
    
    private void processTask(Task task) {
        // Simulate task processing
        try {
            Thread.sleep(100); // Simulate work
            System.out.println("Processed task: " + task.getId());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    public static class Task {
        private final String id;
        private final String data;
        
        public Task(String id, String data) {
            this.id = id;
            this.data = data;
        }
        
        public String getId() { return id; }
        public String getData() { return data; }
    }
}
```

### 2. Disruptor Pattern

```java
// High-performance producer-consumer using LMAX Disruptor
public class DisruptorExample {
    
    private final Disruptor<TaskEvent> disruptor;
    private final RingBuffer<TaskEvent> ringBuffer;
    
    @SuppressWarnings("unchecked")
    public DisruptorExample() {
        // Create disruptor with ring buffer size (must be power of 2)
        disruptor = new Disruptor<>(
            TaskEvent::new,           // Event factory
            1024,                     // Ring buffer size
            DaemonThreadFactory.INSTANCE, // Thread factory
            ProducerType.MULTI,       // Allow multiple producers
            new BusySpinWaitStrategy() // Wait strategy for low latency
        );
        
        // Set up event handlers (consumers)
        disruptor.handleEventsWith(this::handleTask);
        
        // Start disruptor
        ringBuffer = disruptor.start();
    }
    
    public void publishTask(String taskId, String data) {
        // Get next sequence
        long sequence = ringBuffer.next();
        
        try {
            // Get event for sequence
            TaskEvent event = ringBuffer.get(sequence);
            
            // Populate event
            event.setTaskId(taskId);
            event.setData(data);
            event.setTimestamp(System.nanoTime());
            
        } finally {
            // Publish event
            ringBuffer.publish(sequence);
        }
    }
    
    private void handleTask(TaskEvent event, long sequence, boolean endOfBatch) {
        // Process event
        System.out.println("Processing task: " + event.getTaskId() + 
                          " at sequence: " + sequence);
        
        // Simulate processing time
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    public void shutdown() {
        disruptor.shutdown();
    }
    
    public static class TaskEvent {
        private String taskId;
        private String data;
        private long timestamp;
        
        // Default constructor required by Disruptor
        public TaskEvent() {}
        
        // Getters and setters
        public String getTaskId() { return taskId; }
        public void setTaskId(String taskId) { this.taskId = taskId; }
        
        public String getData() { return data; }
        public void setData(String data) { this.data = data; }
        
        public long getTimestamp() { return timestamp; }
        public void setTimestamp(long timestamp) { this.timestamp = timestamp; }
    }
}
```

## Thread Pool Patterns

### 1. Custom Thread Pool Configuration

```java
@Configuration
public class ThreadPoolConfig {
    
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        
        // Core pool size - minimum number of threads
        executor.setCorePoolSize(10);
        
        // Maximum pool size - maximum number of threads
        executor.setMaxPoolSize(50);
        
        // Queue capacity - size of queue for holding tasks
        executor.setQueueCapacity(100);
        
        // Thread name prefix for easier debugging
        executor.setThreadNamePrefix("task-executor-");
        
        // Keep alive time for idle threads
        executor.setKeepAliveSeconds(60);
        
        // Rejection policy when queue is full and max pool size reached
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        
        // Allow core threads to timeout
        executor.setAllowCoreThreadTimeOut(true);
        
        executor.initialize();
        return executor;
    }
    
    @Bean(name = "scheduledTaskExecutor")
    public ScheduledExecutorService scheduledTaskExecutor() {
        return Executors.newScheduledThreadPool(5, r -> {
            Thread t = new Thread(r);
            t.setName("scheduled-task-" + t.getId());
            t.setDaemon(true);
            return t;
        });
    }
    
    @Bean(name = "workStealingPool")
    public Executor workStealingPool() {
        // Work-stealing pool with daemon threads
        return Executors.newWorkStealingPool();
    }
}
```

### 2. Task Execution Patterns

```java
@Service
public class TaskExecutionService {
    
    @Autowired
    private Executor taskExecutor;
    
    @Autowired
    private ScheduledExecutorService scheduledExecutor;
    
    // Asynchronous task execution
    @Async("taskExecutor")
    public CompletableFuture<String> executeAsyncTask(String input) {
        return CompletableFuture.supplyAsync(() -> {
            // Simulate async work
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return "Interrupted";
            }
            
            return "Processed: " + input;
        }, taskExecutor);
    }
    
    // Batch task execution
    public List<CompletableFuture<String>> executeBatchTasks(List<String> inputs) {
        return inputs.stream()
            .map(this::executeAsyncTask)
            .collect(Collectors.toList());
    }
    
    // Wait for all tasks to complete
    public List<String> executeBatchAndWait(List<String> inputs) throws Exception {
        List<CompletableFuture<String>> futures = executeBatchTasks(inputs);
        
        // Wait for all to complete
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).get();
        
        // Collect results
        return futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
    }
    
    // Task with timeout
    public String executeTaskWithTimeout(String input, long timeout, TimeUnit unit) 
            throws Exception {
        
        CompletableFuture<String> future = executeAsyncTask(input);
        
        try {
            return future.get(timeout, unit);
        } catch (TimeoutException e) {
            future.cancel(true); // Cancel the task
            throw new RuntimeException("Task timed out", e);
        }
    }
    
    // Scheduled task execution
    public ScheduledFuture<?> scheduleTask(Runnable task, long delay, TimeUnit unit) {
        return scheduledExecutor.schedule(task, delay, unit);
    }
    
    // Periodic task execution
    public ScheduledFuture<?> schedulePeriodicTask(Runnable task, 
                                                  long initialDelay, 
                                                  long period, 
                                                  TimeUnit unit) {
        return scheduledExecutor.scheduleAtFixedRate(task, initialDelay, period, unit);
    }
    
    // Task composition
    public CompletableFuture<String> executeComplexTask(String input) {
        return executeAsyncTask(input)
            .thenApply(String::toUpperCase)
            .thenApply(s -> s + " - processed")
            .thenApply(s -> s + " at " + Instant.now())
            .exceptionally(throwable -> "Error: " + throwable.getMessage());
    }
    
    // Circuit breaker pattern for task execution
    private final CircuitBreaker circuitBreaker = new CircuitBreaker(5, Duration.ofMinutes(1));
    
    public String executeWithCircuitBreaker(String input) throws Exception {
        if (circuitBreaker.isOpen()) {
            throw new RuntimeException("Circuit breaker is open");
        }
        
        try {
            String result = executeAsyncTask(input).get(5, TimeUnit.SECONDS);
            circuitBreaker.recordSuccess();
            return result;
        } catch (Exception e) {
            circuitBreaker.recordFailure();
            throw e;
        }
    }
    
    public static class CircuitBreaker {
        private final int failureThreshold;
        private final Duration timeout;
        private int failureCount = 0;
        private Instant lastFailureTime;
        private boolean open = false;
        
        public CircuitBreaker(int failureThreshold, Duration timeout) {
            this.failureThreshold = failureThreshold;
            this.timeout = timeout;
        }
        
        public synchronized boolean isOpen() {
            if (open && lastFailureTime != null) {
                if (Duration.between(lastFailureTime, Instant.now()).compareTo(timeout) > 0) {
                    // Reset circuit breaker
                    open = false;
                    failureCount = 0;
                }
            }
            return open;
        }
        
        public synchronized void recordSuccess() {
            failureCount = 0;
            open = false;
        }
        
        public synchronized void recordFailure() {
            failureCount++;
            lastFailureTime = Instant.now();
            
            if (failureCount >= failureThreshold) {
                open = true;
            }
        }
    }
}
```

## Lock-Free Data Structures

### 1. Atomic Variables

```java
public class LockFreeCounter {
    
    private final AtomicLong counter = new AtomicLong(0);
    private final AtomicLong maxValue = new AtomicLong(0);
    private final AtomicReference<String> lastUpdatedBy = new AtomicReference<>();
    
    public void increment(String userId) {
        counter.incrementAndGet();
        
        // Update max value atomically
        maxValue.updateAndGet(current -> Math.max(current, counter.get()));
        
        // Update last updated by
        lastUpdatedBy.set(userId);
    }
    
    public long getCounter() {
        return counter.get();
    }
    
    public long getMaxValue() {
        return maxValue.get();
    }
    
    public String getLastUpdatedBy() {
        return lastUpdatedBy.get();
    }
    
    // Complex atomic operations
    public boolean incrementIfLessThan(long maxLimit) {
        long currentValue;
        do {
            currentValue = counter.get();
            if (currentValue >= maxLimit) {
                return false;
            }
        } while (!counter.compareAndSet(currentValue, currentValue + 1));
        
        return true;
    }
    
    // Atomic field updater for regular fields
    private volatile long volatileCounter = 0;
    private static final AtomicLongFieldUpdater<LockFreeCounter> counterUpdater =
        AtomicLongFieldUpdater.newUpdater(LockFreeCounter.class, "volatileCounter");
    
    public void incrementVolatileCounter() {
        counterUpdater.incrementAndGet(this);
    }
    
    public long getVolatileCounter() {
        return volatileCounter;
    }
}
```

### 2. Concurrent Collections

```java
public class ConcurrentCollectionsExample {
    
    // Thread-safe hash map
    private final ConcurrentHashMap<String, User> userCache = new ConcurrentHashMap<>();
    
    public User getUser(String userId) {
        return userCache.computeIfAbsent(userId, this::loadUserFromDatabase);
    }
    
    public void updateUser(User user) {
        userCache.put(user.getId(), user);
    }
    
    public void removeUser(String userId) {
        userCache.remove(userId);
    }
    
    // Bulk operations
    public void updateUsersBatch(Map<String, User> updates) {
        userCache.putAll(updates);
    }
    
    // Atomic operations
    public User getAndUpdateUser(String userId, UnaryOperator<User> updateFunction) {
        return userCache.compute(userId, (key, existingUser) -> {
            if (existingUser == null) {
                return loadUserFromDatabase(key);
            }
            return updateFunction.apply(existingUser);
        });
    }
    
    // Concurrent skip list map for ordered data
    private final ConcurrentSkipListMap<Long, Order> orderQueue = new ConcurrentSkipListMap<>();
    
    public void addOrder(Order order) {
        orderQueue.put(order.getPriority(), order);
    }
    
    public Order getNextOrder() {
        Map.Entry<Long, Order> first = orderQueue.pollFirstEntry();
        return first != null ? first.getValue() : null;
    }
    
    // Concurrent skip list set for unique ordered elements
    private final ConcurrentSkipListSet<String> activeSessions = new ConcurrentSkipListSet<>();
    
    public void addSession(String sessionId) {
        activeSessions.add(sessionId);
    }
    
    public void removeSession(String sessionId) {
        activeSessions.remove(sessionId);
    }
    
    public boolean hasActiveSession(String sessionId) {
        return activeSessions.contains(sessionId);
    }
    
    // Copy on write array list for read-heavy workloads
    private final CopyOnWriteArrayList<String> features = new CopyOnWriteArrayList<>();
    
    public List<String> getFeatures() {
        return features; // Returns immutable snapshot
    }
    
    public void addFeature(String feature) {
        features.add(feature); // Creates new copy internally
    }
    
    private User loadUserFromDatabase(String userId) {
        // Simulate database load
        return new User(userId, "User " + userId);
    }
    
    public static class User {
        private final String id;
        private final String name;
        
        public User(String id, String name) {
            this.id = id;
            this.name = name;
        }
        
        public String getId() { return id; }
        public String getName() { return name; }
    }
    
    public static class Order {
        private final String id;
        private final long priority;
        
        public Order(String id, long priority) {
            this.id = id;
            this.priority = priority;
        }
        
        public String getId() { return id; }
        public long getPriority() { return priority; }
    }
}
```

## Reactive Programming Patterns

### 1. Reactive Streams with Project Reactor

```java
@Service
public class ReactiveProcessingService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    // Basic reactive operations
    public Flux<User> getUsersReactive() {
        return Flux.fromIterable(userRepository.findAll())
            .doOnNext(user -> log.info("Processing user: {}", user.getId()))
            .doOnError(error -> log.error("Error processing users", error));
    }
    
    // Asynchronous data processing
    public Mono<UserDetails> getUserDetailsReactive(String userId) {
        return Mono.fromCallable(() -> userRepository.findById(userId))
            .flatMap(optionalUser -> optionalUser
                .map(Mono::just)
                .orElse(Mono.error(new UserNotFoundException(userId))))
            .flatMap(user -> {
                // Load additional user details asynchronously
                Mono<List<Order>> userOrders = getUserOrdersReactive(userId);
                Mono<UserPreferences> userPreferences = getUserPreferencesReactive(userId);
                
                return Mono.zip(userOrders, userPreferences)
                    .map(tuple -> new UserDetails(user, tuple.getT1(), tuple.getT2()));
            })
            .subscribeOn(Schedulers.boundedElastic()); // Run on async thread pool
    }
    
    public Mono<List<Order>> getUserOrdersReactive(String userId) {
        return Mono.fromCallable(() -> orderRepository.findByUserId(userId))
            .subscribeOn(Schedulers.boundedElastic());
    }
    
    public Mono<UserPreferences> getUserPreferencesReactive(String userId) {
        return Mono.fromCallable(() -> loadUserPreferences(userId))
            .subscribeOn(Schedulers.boundedElastic());
    }
    
    // Reactive batch processing
    public Flux<ProcessedOrder> processOrdersReactive(List<String> orderIds) {
        return Flux.fromIterable(orderIds)
            .flatMap(orderId -> processOrderReactive(orderId), 10) // Concurrency limit
            .onErrorContinue((error, orderId) -> {
                log.error("Error processing order: {}", orderId, error);
            });
    }
    
    private Mono<ProcessedOrder> processOrderReactive(String orderId) {
        return Mono.fromCallable(() -> {
                Order order = orderRepository.findById(orderId)
                    .orElseThrow(() -> new OrderNotFoundException(orderId));
                
                // Process order
                return new ProcessedOrder(orderId, "PROCESSED");
            })
            .subscribeOn(Schedulers.parallel())
            .timeout(Duration.ofSeconds(30)) // Timeout
            .retry(3); // Retry on failure
    }
    
    // Reactive database queries
    public Flux<User> searchUsersReactive(String query, int page, int size) {
        return Flux.defer(() -> {
                Pageable pageable = PageRequest.of(page, size);
                Page<User> userPage = userRepository.searchByUsernameOrFullName(query, pageable);
                return Flux.fromIterable(userPage.getContent());
            })
            .doOnNext(user -> log.debug("Found user: {}", user.getUsername()));
    }
    
    // Reactive caching
    private final Map<String, Mono<User>> userCache = new ConcurrentHashMap<>();
    
    public Mono<User> getUserCachedReactive(String userId) {
        return userCache.computeIfAbsent(userId, key -> 
            Mono.fromCallable(() -> userRepository.findById(key))
                .flatMap(optional -> optional
                    .map(Mono::just)
                    .orElse(Mono.error(new UserNotFoundException(key))))
                .cache(Duration.ofMinutes(10)) // Cache result for 10 minutes
        );
    }
    
    // Reactive error handling
    public Flux<User> getUsersWithErrorHandling() {
        return Flux.fromIterable(userRepository.findAll())
            .onErrorResume(throwable -> {
                log.error("Error loading users, returning empty list", throwable);
                return Flux.empty();
            })
            .onErrorContinue((throwable, user) -> {
                log.warn("Error processing user {}, skipping", user, throwable);
            });
    }
    
    // Reactive backpressure handling
    public Flux<String> processWithBackpressure(Flux<String> input) {
        return input
            .onBackpressureBuffer(1000) // Buffer up to 1000 items
            .flatMap(item -> processItemReactive(item), 5) // Max concurrency of 5
            .doOnNext(result -> log.debug("Processed item: {}", result));
    }
    
    private Mono<String> processItemReactive(String item) {
        return Mono.fromCallable(() -> {
                // Simulate processing
                Thread.sleep(100);
                return "Processed: " + item;
            })
            .subscribeOn(Schedulers.elastic());
    }
    
    private UserPreferences loadUserPreferences(String userId) {
        // Simulate loading preferences
        return new UserPreferences(userId, Arrays.asList("action", "comedy"));
    }
    
    // Data classes
    public static class UserDetails {
        private final User user;
        private final List<Order> orders;
        private final UserPreferences preferences;
        
        public UserDetails(User user, List<Order> orders, UserPreferences preferences) {
            this.user = user;
            this.orders = orders;
            this.preferences = preferences;
        }
        
        // Getters
    }
    
    public static class UserPreferences {
        private final String userId;
        private final List<String> genres;
        
        public UserPreferences(String userId, List<String> genres) {
            this.userId = userId;
            this.genres = genres;
        }
        
        // Getters
    }
    
    public static class ProcessedOrder {
        private final String orderId;
        private final String status;
        
        public ProcessedOrder(String orderId, String status) {
            this.orderId = orderId;
            this.status = status;
        }
        
        // Getters
    }
}
```

### 2. Reactive Web Controllers

```java
@RestController
@RequestMapping("/api/v1/reactive")
public class ReactiveUserController {
    
    @Autowired
    private ReactiveProcessingService processingService;
    
    @GetMapping("/users")
    public Flux<User> getUsers() {
        return processingService.getUsersReactive()
            .doOnNext(user -> log.info("Returning user: {}", user.getId()));
    }
    
    @GetMapping("/users/{userId}")
    public Mono<UserDetails> getUserDetails(@PathVariable String userId) {
        return processingService.getUserDetailsReactive(userId)
            .doOnNext(details -> log.info("Returning user details for: {}", userId));
    }
    
    @GetMapping("/users/search")
    public Flux<User> searchUsers(@RequestParam String query,
                                 @RequestParam(defaultValue = "0") int page,
                                 @RequestParam(defaultValue = "20") int size) {
        
        return processingService.searchUsersReactive(query, page, size);
    }
    
    @PostMapping("/orders/process")
    public Flux<ProcessedOrder> processOrders(@RequestBody List<String> orderIds) {
        return processingService.processOrdersReactive(orderIds)
            .doOnNext(order -> log.info("Processed order: {}", order.getOrderId()));
    }
    
    @PostMapping("/items/process")
    public Flux<String> processItems(@RequestBody Flux<String> items) {
        return processingService.processWithBackpressure(items);
    }
    
    // Error handling
    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Mono<ErrorResponse> handleUserNotFound(UserNotFoundException e) {
        return Mono.just(new ErrorResponse("USER_NOT_FOUND", e.getMessage()));
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public Mono<ErrorResponse> handleGenericError(Exception e) {
        log.error("Unexpected error", e);
        return Mono.just(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred"));
    }
    
    public static class ErrorResponse {
        private final String errorCode;
        private final String message;
        
        public ErrorResponse(String errorCode, String message) {
            this.errorCode = errorCode;
            this.message = message;
        }
        
        // Getters
    }
}
```

## Performance Monitoring

### 1. Thread Pool Metrics

```java
@Configuration
public class MetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCustomizer() {
        return registry -> {
            registry.config()
                .commonTags("application", "concurrent-app");
        };
    }
    
    @Bean
    public ExecutorService monitoredExecutorService(MeterRegistry registry) {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            10, 50, 60L, TimeUnit.SECONDS, new LinkedBlockingQueue<>(100),
            new NamedThreadFactory("monitored-executor")
        );
        
        // Add metrics
        Gauge.builder("executor.pool.size", executor, ThreadPoolExecutor::getPoolSize)
            .register(registry);
        
        Gauge.builder("executor.active.threads", executor, ThreadPoolExecutor::getActiveCount)
            .register(registry);
        
        Gauge.builder("executor.queue.size", executor, e -> e.getQueue().size())
            .register(registry);
        
        return executor;
    }
}

@Service
public class PerformanceMonitor {
    
    private final MeterRegistry registry;
    private final Map<String, Timer.Sample> activeTimers = new ConcurrentHashMap<>();
    
    public PerformanceMonitor(MeterRegistry registry) {
        this.registry = registry;
    }
    
    public void startTimer(String operationId) {
        Timer.Sample sample = Timer.start(registry);
        activeTimers.put(operationId, sample);
    }
    
    public void stopTimer(String operationId, String operationName) {
        Timer.Sample sample = activeTimers.remove(operationId);
        if (sample != null) {
            sample.stop(Timer.builder(operationName)
                .description("Time taken for " + operationName)
                .register(registry));
        }
    }
    
    public void recordConcurrentOperation(String operationName, Runnable operation) {
        Counter.builder(operationName + "_started")
            .description("Number of " + operationName + " operations started")
            .register(registry)
            .increment();
        
        long startTime = System.nanoTime();
        
        try {
            operation.run();
            
            Counter.builder(operationName + "_completed")
                .description("Number of " + operationName + " operations completed")
                .register(registry)
                .increment();
                
        } catch (Exception e) {
            Counter.builder(operationName + "_failed")
                .description("Number of " + operationName + " operations failed")
                .register(registry)
                .increment();
                
            throw e;
        } finally {
            long duration = System.nanoTime() - startTime;
            Timer.builder(operationName + "_duration")
                .description("Duration of " + operationName + " operations")
                .register(registry)
                .record(duration, TimeUnit.NANOSECONDS);
        }
    }
    
    // Monitor thread contention
    public void monitorThreadContention(String resourceName, Runnable operation) {
        long startTime = System.nanoTime();
        long startCpuTime = getCurrentThreadCpuTime();
        
        try {
            operation.run();
        } finally {
            long endTime = System.nanoTime();
            long endCpuTime = getCurrentThreadCpuTime();
            
            double wallClockTime = (endTime - startTime) / 1_000_000.0; // milliseconds
            double cpuTime = (endCpuTime - startCpuTime) / 1_000_000.0; // milliseconds
            
            // Calculate CPU utilization
            double cpuUtilization = cpuTime / wallClockTime;
            
            Gauge.builder("thread.cpu.utilization")
                .tag("resource", resourceName)
                .register(registry)
                .set(cpuUtilization);
        }
    }
    
    private long getCurrentThreadCpuTime() {
        try {
            return ((com.sun.management.ThreadMXBean) 
                ManagementFactory.getThreadMXBean()).getCurrentThreadCpuTime();
        } catch (Exception e) {
            return 0;
        }
    }
    
    // Monitor lock contention
    public void monitorLockContention(String lockName, Runnable operation) {
        long startTime = System.nanoTime();
        
        try {
            operation.run();
        } finally {
            long duration = System.nanoTime() - startTime;
            
            Timer.builder("lock.contention.duration")
                .tag("lock", lockName)
                .description("Time spent waiting for lock")
                .register(registry)
                .record(duration, TimeUnit.NANOSECONDS);
        }
    }
}
```

### 2. JVM Metrics

```java
@Configuration
public class JvmMetricsConfig {
    
    @Bean
    public JvmMemoryMetrics jvmMemoryMetrics(MeterRegistry registry) {
        return new JvmMemoryMetrics(registry);
    }
    
    @Bean
    public JvmGcMetrics jvmGcMetrics(MeterRegistry registry) {
        return new JvmGcMetrics(registry);
    }
    
    @Bean
    public JvmThreadMetrics jvmThreadMetrics(MeterRegistry registry) {
        return new JvmThreadMetrics(registry);
    }
}

@Component
public class ConcurrencyHealthIndicator implements HealthIndicator {
    
    @Autowired
    private ThreadPoolTaskExecutor taskExecutor;
    
    @Override
    public Health health() {
        ThreadPoolExecutor executor = (ThreadPoolExecutor) taskExecutor.getThreadPoolExecutor();
        
        int activeThreads = executor.getActiveCount();
        int poolSize = executor.getPoolSize();
        int queueSize = executor.getQueue().size();
        long completedTasks = executor.getCompletedTaskCount();
        
        // Check if thread pool is healthy
        boolean healthy = activeThreads < poolSize && queueSize < executor.getQueue().remainingCapacity();
        
        if (healthy) {
            return Health.up()
                .withDetail("activeThreads", activeThreads)
                .withDetail("poolSize", poolSize)
                .withDetail("queueSize", queueSize)
                .withDetail("completedTasks", completedTasks)
                .build();
        } else {
            return Health.down()
                .withDetail("activeThreads", activeThreads)
                .withDetail("poolSize", poolSize)
                .withDetail("queueSize", queueSize)
                .withDetail("error", "Thread pool is overloaded")
                .build();
        }
    }
}
```

## Best Practices

### 1. Thread Safety Guidelines

```java
public class ConcurrencyBestPractices {
    
    // ✅ DO: Use immutable objects where possible
    public final class ImmutableConfig {
        private final String host;
        private final int port;
        private final Map<String, String> properties;
        
        public ImmutableConfig(String host, int port, Map<String, String> properties) {
            this.host = host;
            this.port = port;
            this.properties = Map.copyOf(properties);
        }
        
        // Only getters, no setters
    }
    
    // ✅ DO: Prefer composition over inheritance for thread safety
    public class ThreadSafeList<T> {
        private final List<T> list = new ArrayList<>();
        private final Object lock = new Object();
        
        public void add(T item) {
            synchronized (lock) {
                list.add(item);
            }
        }
        
        public List<T> getSnapshot() {
            synchronized (lock) {
                return new ArrayList<>(list); // Defensive copy
            }
        }
    }
    
    // ❌ DON'T: Expose internal state
    public class BadExample {
        private List<String> items = new ArrayList<>();
        
        public List<String> getItems() {
            return items; // Exposes mutable internal state
        }
    }
    
    // ✅ DO: Use proper encapsulation
    public class GoodExample {
        private final List<String> items = new ArrayList<>();
        
        public synchronized void addItem(String item) {
            items.add(item);
        }
        
        public synchronized List<String> getItems() {
            return new ArrayList<>(items); // Return defensive copy
        }
    }
    
    // ✅ DO: Use thread-local variables for request-scoped data
    private static final ThreadLocal<UserContext> userContext = 
        ThreadLocal.withInitial(UserContext::new);
    
    public void processRequest() {
        try {
            // Set request-scoped data
            userContext.get().setUserId(getCurrentUserId());
            userContext.get().setRequestId(UUID.randomUUID().toString());
            
            // Process request
            doWork();
        } finally {
            userContext.remove(); // Always clean up
        }
    }
    
    // ✅ DO: Use appropriate concurrency utilities
    private final ConcurrentHashMap<String, Object> cache = new ConcurrentHashMap<>();
    private final AtomicInteger counter = new AtomicInteger(0);
    private final Semaphore semaphore = new Semaphore(10); // Limit concurrent access
    
    public Object getFromCache(String key) {
        return cache.computeIfAbsent(key, this::loadFromDatabase);
    }
    
    public void incrementCounter() {
        counter.incrementAndGet();
    }
    
    public void doLimitedWork() throws InterruptedException {
        semaphore.acquire();
        try {
            // Do work with limited concurrency
            performWork();
        } finally {
            semaphore.release();
        }
    }
    
    // ✅ DO: Handle interruptions properly
    public void interruptibleWork() throws InterruptedException {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                // Do work
                doUnitOfWork();
                
                // Check for interruption
                if (Thread.interrupted()) {
                    throw new InterruptedException();
                }
                
            } catch (InterruptedException e) {
                // Restore interruption status
                Thread.currentThread().interrupt();
                throw e;
            }
        }
    }
    
    // Helper methods (simplified)
    private void doWork() { /* implementation */ }
    private void performWork() { /* implementation */ }
    private void doUnitOfWork() { /* implementation */ }
    private Object loadFromDatabase(String key) { return new Object(); }
    private String getCurrentUserId() { return "user123"; }
    
    public static class UserContext {
        private String userId;
        private String requestId;
        
        // Getters and setters
    }
}
```

### 2. Performance Considerations

```java
public class PerformancePatterns {
    
    // ✅ DO: Use object pooling for expensive objects
    private static final ObjectPool<ExpensiveObject> objectPool = 
        new GenericObjectPool<>(new ExpensiveObjectFactory());
    
    public void usePooledObject() {
        ExpensiveObject obj = null;
        try {
            obj = objectPool.borrowObject();
            // Use object
            obj.doWork();
        } catch (Exception e) {
            // Handle error
        } finally {
            if (obj != null) {
                objectPool.returnObject(obj);
            }
        }
    }
    
    // ✅ DO: Batch operations to reduce contention
    public void batchDatabaseUpdates(List<User> users) {
        // Instead of updating one by one, batch them
        userRepository.saveAll(users);
    }
    
    // ✅ DO: Use striped locks for better concurrency
    private final Striped<Lock> stripedLocks = Striped.lock(64); // 64 stripes
    
    public void updateUserConcurrently(String userId, UserUpdate update) {
        Lock lock = stripedLocks.get(userId); // Get lock based on userId
        lock.lock();
        try {
            // Update user - only users with same lock stripe block each other
            updateUser(userId, update);
        } finally {
            lock.unlock();
        }
    }
    
    // ✅ DO: Use non-blocking algorithms when possible
    private final AtomicReference<Node> head = new AtomicReference<>();
    
    public boolean addToLockFreeList(String item) {
        Node newNode = new Node(item);
        
        while (true) {
            Node currentHead = head.get();
            newNode.next = currentHead;
            
            if (head.compareAndSet(currentHead, newNode)) {
                return true; // Successfully added
            }
            // Retry if another thread modified head
        }
    }
    
    // ✅ DO: Profile and optimize hotspots
    @Timed(value = "database.query.time", description = "Time taken for database queries")
    public List<User> findUsersWithComplexQuery(String query) {
        // Complex query - measure performance
        return userRepository.findUsersWithComplexQuery(query);
    }
    
    // ✅ DO: Use appropriate data structures for concurrent access
    private final ConcurrentHashMap<String, ConcurrentLinkedQueue<Event>> eventQueues = 
        new ConcurrentHashMap<>();
    
    public void addEvent(String userId, Event event) {
        eventQueues.computeIfAbsent(userId, k -> new ConcurrentLinkedQueue<>())
                  .offer(event);
    }
    
    public Event pollEvent(String userId) {
        ConcurrentLinkedQueue<Event> queue = eventQueues.get(userId);
        return queue != null ? queue.poll() : null;
    }
    
    // Helper classes (simplified)
    public static class ExpensiveObject {
        public void doWork() { /* expensive operation */ }
    }
    
    public static class ExpensiveObjectFactory extends BasePooledObjectFactory<ExpensiveObject> {
        @Override
        public ExpensiveObject create() { return new ExpensiveObject(); }
        
        @Override
        public PooledObject<ExpensiveObject> wrap(ExpensiveObject obj) {
            return new DefaultPooledObject<>(obj);
        }
    }
    
    public static class Node {
        final String value;
        volatile Node next;
        
        Node(String value) { this.value = value; }
    }
    
    public static class UserUpdate { /* user update data */ }
    public static class Event { /* event data */ }
    
    // Mock methods
    private void updateUser(String userId, UserUpdate update) { /* implementation */ }
}
```

## Conclusion

Java concurrency patterns are essential for building high-performance, scalable applications that can handle multiple simultaneous operations efficiently. Key takeaways include:

- **Thread Safety**: Use immutable objects, proper synchronization, and atomic operations
- **Performance**: Choose appropriate data structures, use connection pooling, and monitor performance
- **Patterns**: Apply producer-consumer, thread pool, and reactive programming patterns
- **Monitoring**: Implement comprehensive metrics and health checks
- **Best Practices**: Follow established guidelines for thread safety and performance

Effective concurrency programming requires understanding the Java Memory Model, choosing appropriate synchronization mechanisms, and continuously monitoring and optimizing performance. When applied correctly, these patterns enable applications to scale efficiently while maintaining correctness and reliability.
