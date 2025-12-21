# Asynchronous Processing Patterns

Asynchronous processing enables systems to handle operations without blocking, improving scalability, responsiveness, and resource utilization. This guide covers essential patterns for implementing robust asynchronous processing in distributed systems.

## What is Asynchronous Processing?

Asynchronous processing allows operations to be initiated and completed independently of the main execution flow. Instead of waiting for an operation to finish, the system continues processing and handles the result when it becomes available.

### Key Benefits
- **Non-blocking**: System remains responsive during long operations
- **Scalability**: Better resource utilization and throughput
- **Fault Tolerance**: Failures in async operations don't block main flow
- **User Experience**: Immediate feedback while processing continues in background

### Synchronous vs Asynchronous

#### Synchronous Processing
```java
public User createUserSync(CreateUserRequest request) {
    // Block until database operation completes
    User user = userRepository.save(new User(request.getName(), request.getEmail()));
    
    // Block until email is sent
    emailService.sendWelcomeEmail(user.getEmail());
    
    // Return only after all operations complete
    return user;
}
```

#### Asynchronous Processing
```java
public CompletableFuture<User> createUserAsync(CreateUserRequest request) {
    User user = new User(request.getName(), request.getEmail());
    
    return userRepository.saveAsync(user)
        .thenCompose(savedUser -> {
            // Send email asynchronously
            emailService.sendWelcomeEmailAsync(savedUser.getEmail());
            return CompletableFuture.completedFuture(savedUser);
        });
}
```

## Core Asynchronous Patterns

### 1. Callback Pattern

#### Basic Callbacks
```java
public interface AsyncCallback<T> {
    void onSuccess(T result);
    void onFailure(Throwable error);
}

public void processWithCallback(AsyncCallback<String> callback) {
    executor.submit(() -> {
        try {
            String result = performLongRunningOperation();
            callback.onSuccess(result);
        } catch (Exception e) {
            callback.onFailure(e);
        }
    });
}

// Usage
processWithCallback(new AsyncCallback<String>() {
    @Override
    public void onSuccess(String result) {
        System.out.println("Success: " + result);
    }
    
    @Override
    public void onFailure(Throwable error) {
        System.err.println("Error: " + error.getMessage());
    }
});
```

#### Callback Hell Problem
```java
// Nested callbacks become hard to read and maintain
service.doFirstOperation(request, new Callback<Result1>() {
    @Override
    public void onSuccess(Result1 result1) {
        service.doSecondOperation(result1, new Callback<Result2>() {
            @Override
            public void onSuccess(Result2 result2) {
                service.doThirdOperation(result2, new Callback<Result3>() {
                    @Override
                    public void onSuccess(Result3 result3) {
                        // Finally done!
                        processFinalResult(result3);
                    }
                    
                    @Override
                    public void onFailure(Throwable error) {
                        handleError(error);
                    }
                });
            }
            
            @Override
            public void onFailure(Throwable error) {
                handleError(error);
            }
        });
    }
    
    @Override
    public void onFailure(Throwable error) {
        handleError(error);
    }
});
```

### 2. Future/Promise Pattern

#### Java CompletableFuture
```java
@Service
public class AsyncUserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EmailService emailService;
    
    public CompletableFuture<User> createUserAsync(CreateUserRequest request) {
        // Start with user creation
        return CompletableFuture.supplyAsync(() -> {
            User user = new User(request.getName(), request.getEmail());
            return userRepository.save(user);
        })
        .thenCompose(user -> {
            // Send welcome email asynchronously
            CompletableFuture<Void> emailFuture = emailService.sendWelcomeEmailAsync(user.getEmail());
            
            // Return user immediately, email sending continues in background
            return CompletableFuture.completedFuture(user);
        })
        .exceptionally(throwable -> {
            logger.error("Failed to create user", throwable);
            throw new CompletionException(throwable);
        });
    }
    
    public CompletableFuture<UserDetails> getUserDetailsAsync(Long userId) {
        // Parallel execution of independent operations
        CompletableFuture<User> userFuture = userRepository.findByIdAsync(userId);
        CompletableFuture<List<Post>> postsFuture = postRepository.findRecentPostsByUserIdAsync(userId);
        CompletableFuture<List<Comment>> commentsFuture = commentRepository.findRecentCommentsByUserIdAsync(userId);
        
        return userFuture.thenCombine(postsFuture, (user, posts) -> 
            UserDetails.builder()
                .user(user)
                .recentPosts(posts)
                .build()
        )
        .thenCombine(commentsFuture, (userDetails, comments) -> 
            userDetails.toBuilder()
                .recentComments(comments)
                .build()
        );
    }
}
```

#### Error Handling with Futures
```java
public CompletableFuture<Order> processOrderAsync(OrderRequest request) {
    return validateOrderAsync(request)
        .thenCompose(validatedRequest -> reserveInventoryAsync(validatedRequest))
        .thenCompose(reservation -> processPaymentAsync(reservation))
        .thenCompose(payment -> createOrderAsync(payment))
        .thenApply(order -> {
            // Send confirmation asynchronously
            sendConfirmationAsync(order);
            return order;
        })
        .exceptionally(throwable -> {
            logger.error("Order processing failed", throwable);
            
            // Compensate for partial failures
            compensateFailedOrder(request);
            
            throw new CompletionException(throwable);
        });
}
```

### 3. Reactive Programming Pattern

#### Project Reactor (Spring WebFlux)
```java
@Service
public class ReactiveOrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    public Mono<Order> createOrderReactive(CreateOrderRequest request) {
        return Mono.fromCallable(() -> validateOrderRequest(request))
            .subscribeOn(Schedulers.boundedElastic())
            
            .flatMap(validatedRequest -> 
                inventoryService.reserveInventory(validatedRequest.getItems())
                    .timeout(Duration.ofSeconds(5))
                    .retry(2)
            )
            
            .flatMap(inventoryReservation -> 
                paymentService.processPayment(
                    request.getPaymentInfo(), 
                    request.getTotalAmount()
                )
                .timeout(Duration.ofSeconds(10))
            )
            
            .flatMap(paymentResult -> {
                Order order = new Order(request.getCustomerId(), request.getItems(), paymentResult);
                return orderRepository.saveReactive(order);
            })
            
            .doOnSuccess(order -> {
                // Send confirmation email asynchronously
                sendOrderConfirmationAsync(order);
                
                // Publish order created event
                eventPublisher.publish(new OrderCreatedEvent(order.getId()));
            })
            
            .doOnError(error -> {
                logger.error("Order creation failed", error);
                // Handle compensation logic
            });
    }
    
    public Flux<OrderSummary> getOrderSummaries(String customerId) {
        return orderRepository.findByCustomerIdReactive(customerId)
            .map(order -> OrderSummary.from(order))
            .sort((o1, o2) -> o2.getCreatedAt().compareTo(o1.getCreatedAt()))
            .take(10); // Limit to 10 most recent
    }
}
```

#### Backpressure Handling
```java
public Flux<ProcessedData> processDataStream(Flux<RawData> dataStream) {
    return dataStream
        .onBackpressureBuffer(1000) // Buffer up to 1000 items
        .flatMap(data -> processDataItem(data), 10) // Max concurrency of 10
        .onBackpressureDrop(droppedItem -> {
            logger.warn("Dropping data item due to backpressure: {}", droppedItem.getId());
        })
        .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
            .jitter(0.1)
            .doAfterRetry(retrySignal -> 
                logger.warn("Retrying after failure, attempt: {}", retrySignal.totalRetries())
            )
        );
}
```

### 4. Actor Model Pattern

#### Akka Actors Implementation
```java
public class OrderProcessor extends AbstractActor {
    
    @Override
    public Receive createReceive() {
        return receiveBuilder()
            .match(CreateOrderCommand.class, this::handleCreateOrder)
            .match(PaymentProcessedEvent.class, this::handlePaymentProcessed)
            .match(InventoryReservedEvent.class, this::handleInventoryReserved)
            .build();
    }
    
    private void handleCreateOrder(CreateOrderCommand command) {
        // Start order creation process
        Order order = new Order(command.getCustomerId(), command.getItems());
        
        // Send to inventory actor
        ActorRef inventoryActor = getContext().actorOf(InventoryActor.props());
        inventoryActor.tell(new ReserveInventoryCommand(order.getId(), order.getItems()), getSelf());
        
        // Send to payment actor
        ActorRef paymentActor = getContext().actorOf(PaymentActor.props());
        paymentActor.tell(new ProcessPaymentCommand(order.getId(), command.getPaymentInfo()), getSelf());
        
        // Store order state
        getContext().become(processingOrder(order));
    }
    
    private Receive processingOrder(Order order) {
        return receiveBuilder()
            .match(InventoryReservedEvent.class, event -> {
                order.setInventoryReserved(true);
                checkOrderCompletion(order);
            })
            .match(PaymentProcessedEvent.class, event -> {
                order.setPaymentProcessed(true);
                checkOrderCompletion(order);
            })
            .build();
    }
    
    private void checkOrderCompletion(Order order) {
        if (order.isInventoryReserved() && order.isPaymentProcessed()) {
            // Order is complete
            orderRepository.save(order);
            getContext().unbecome();
        }
    }
}
```

## Asynchronous Communication Patterns

### 1. Fire-and-Forget Pattern

#### One-way Communication
```java
@Service
public class EventPublisher {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publishUserEvent(UserEvent event) {
        // Fire and forget - don't wait for acknowledgment
        kafkaTemplate.send("user-events", event.getUserId(), event);
        
        // Continue immediately, event processing happens asynchronously
        logger.debug("Published user event for user {}", event.getUserId());
    }
}
```

#### Best for:
- Logging and analytics events
- Notification sending
- Cache invalidation
- Background cleanup tasks

### 2. Request-Reply Pattern

#### Synchronous Request with Async Processing
```java
@RestController
public class OrderController {
    
    @Autowired
    private AsyncOrderService orderService;
    
    @PostMapping("/orders")
    public CompletableFuture<ResponseEntity<OrderResponse>> createOrder(
            @RequestBody CreateOrderRequest request) {
        
        return orderService.createOrderAsync(request)
            .thenApply(order -> {
                // Return immediate response
                OrderResponse response = OrderResponse.from(order);
                return ResponseEntity.accepted()
                    .header("Location", "/orders/" + order.getId())
                    .body(response);
            })
            .exceptionally(throwable -> {
                logger.error("Order creation failed", throwable);
                return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body(new OrderResponse("Order creation failed"));
            });
    }
    
    @GetMapping("/orders/{orderId}")
    public ResponseEntity<OrderDetails> getOrder(@PathVariable String orderId) {
        // Check order status
        Order order = orderService.getOrder(orderId);
        return ResponseEntity.ok(OrderDetails.from(order));
    }
}
```

### 3. Saga Pattern for Distributed Transactions

#### Orchestration-based Saga
```java
@Service
public class OrderSagaOrchestrator {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private ShippingService shippingService;
    
    public CompletableFuture<Order> processOrderSaga(OrderRequest request) {
        Order order = createInitialOrder(request);
        
        return CompletableFuture.completedFuture(order)
            .thenCompose(this::reserveInventory)
            .thenCompose(this::processPayment)
            .thenCompose(this::arrangeShipping)
            .thenApply(finalOrder -> {
                finalOrder.setStatus(OrderStatus.COMPLETED);
                return orderRepository.save(finalOrder);
            })
            .exceptionally(throwable -> {
                // Compensate for partial failures
                compensateOrder(order);
                throw new CompletionException(throwable);
            });
    }
    
    private CompletableFuture<Order> reserveInventory(Order order) {
        return inventoryService.reserveInventory(order.getId(), order.getItems())
            .thenApply(reservation -> {
                order.setInventoryReservationId(reservation.getId());
                return order;
            });
    }
    
    private CompletableFuture<Order> processPayment(Order order) {
        return paymentService.processPayment(order.getPaymentInfo(), order.getTotalAmount())
            .thenApply(payment -> {
                order.setPaymentId(payment.getId());
                order.setStatus(OrderStatus.PAID);
                return order;
            });
    }
    
    private void compensateOrder(Order order) {
        // Rollback inventory reservation
        if (order.getInventoryReservationId() != null) {
            inventoryService.releaseInventory(order.getInventoryReservationId());
        }
        
        // Refund payment if processed
        if (order.getPaymentId() != null) {
            paymentService.refundPayment(order.getPaymentId());
        }
        
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
    }
}
```

#### Choreography-based Saga
```java
@Component
public class OrderEventHandlers {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Reserve inventory
        inventoryService.reserveInventory(event.getOrderId(), event.getItems())
            .thenAccept(reservation -> 
                eventPublisher.publish(new InventoryReservedEvent(event.getOrderId()))
            )
            .exceptionally(throwable -> {
                eventPublisher.publish(new OrderFailedEvent(event.getOrderId(), "Inventory unavailable"));
                return null;
            });
    }
    
    @EventListener
    public void handleInventoryReserved(InventoryReservedEvent event) {
        // Process payment after inventory is reserved
        Order order = orderRepository.findById(event.getOrderId());
        paymentService.processPayment(order.getPaymentInfo(), order.getTotalAmount())
            .thenAccept(payment -> 
                eventPublisher.publish(new PaymentProcessedEvent(event.getOrderId()))
            )
            .exceptionally(throwable -> {
                // Compensate: release inventory
                inventoryService.releaseInventory(event.getOrderId());
                eventPublisher.publish(new OrderFailedEvent(event.getOrderId(), "Payment failed"));
                return null;
            });
    }
}
```

## Error Handling and Resilience

### Circuit Breaker Pattern
```java
@Service
public class CircuitBreakerEmailService {
    
    private final CircuitBreaker circuitBreaker;
    
    public CircuitBreakerEmailService() {
        this.circuitBreaker = CircuitBreaker.ofDefaults("email-service");
    }
    
    public CompletableFuture<Void> sendEmailAsync(String to, String subject, String body) {
        return circuitBreaker.executeSupplier(() -> 
            CompletableFuture.runAsync(() -> {
                try {
                    emailProvider.sendEmail(to, subject, body);
                } catch (EmailException e) {
                    throw new CompletionException(e);
                }
            })
        );
    }
}
```

### Timeout and Cancellation
```java
public CompletableFuture<String> processWithTimeout(CompletableFuture<String> future, Duration timeout) {
    return future.orTimeout(timeout.toMillis(), TimeUnit.MILLISECONDS)
        .exceptionally(throwable -> {
            if (throwable instanceof TimeoutException) {
                logger.warn("Operation timed out after {}", timeout);
                return "TIMEOUT";
            }
            throw new CompletionException(throwable);
        });
}

public void cancelLongRunningOperation() {
    CompletableFuture<String> future = startLongRunningOperation();
    
    // Cancel after 30 seconds
    scheduler.schedule(() -> {
        if (!future.isDone()) {
            future.cancel(true);
            logger.info("Cancelled long running operation");
        }
    }, 30, TimeUnit.SECONDS);
}
```

### Retry and Recovery
```java
public CompletableFuture<String> processWithRetry(Supplier<CompletableFuture<String>> operation) {
    return operation.get()
        .exceptionally(throwable -> {
            logger.warn("Operation failed, retrying", throwable);
            // Simple retry logic
            try {
                Thread.sleep(1000); // Wait before retry
                return operation.get().join();
            } catch (Exception retryException) {
                logger.error("Retry also failed", retryException);
                throw new CompletionException(retryException);
            }
        });
}
```

## Performance Optimization

### Connection Pooling
```java
@Configuration
public class AsyncConfig {
    
    @Bean
    public WebClient webClient() {
        HttpClient httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 10000)
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(10))
                .addHandlerLast(new WriteTimeoutHandler(10))
            );
        
        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
    
    @Bean
    public ThreadPoolTaskExecutor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

### Batch Processing
```java
@Service
public class BatchProcessor {
    
    private final List<Order> batch = Collections.synchronizedList(new ArrayList<>());
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    
    public BatchProcessor() {
        // Process batch every 5 seconds
        scheduler.scheduleAtFixedRate(this::processBatch, 0, 5, TimeUnit.SECONDS);
    }
    
    public void addToBatch(Order order) {
        synchronized (batch) {
            batch.add(order);
            if (batch.size() >= 50) { // Process early if batch is large
                processBatch();
            }
        }
    }
    
    private void processBatch() {
        List<Order> ordersToProcess;
        synchronized (batch) {
            if (batch.isEmpty()) return;
            ordersToProcess = new ArrayList<>(batch);
            batch.clear();
        }
        
        // Process batch asynchronously
        CompletableFuture.runAsync(() -> 
            orderRepository.saveAll(ordersToProcess)
        ).exceptionally(throwable -> {
            logger.error("Batch processing failed", throwable);
            // Implement retry logic or dead letter queue
            return null;
        });
    }
}
```

## Monitoring and Observability

### Async Operation Metrics
```java
@Service
public class AsyncMetricsService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter asyncOperationsStarted = Counter.builder("async_operations_started_total")
        .description("Total async operations started")
        .register(meterRegistry);
    
    private final Counter asyncOperationsCompleted = Counter.builder("async_operations_completed_total")
        .description("Total async operations completed")
        .register(meterRegistry);
    
    private final Counter asyncOperationsFailed = Counter.builder("async_operations_failed_total")
        .description("Total async operations failed")
        .register(meterRegistry);
    
    private final Histogram asyncOperationDuration = Histogram.builder("async_operation_duration_seconds")
        .description("Async operation duration")
        .register(meterRegistry);
    
    public void recordAsyncOperationStarted(String operationType) {
        asyncOperationsStarted.increment();
    }
    
    public void recordAsyncOperationCompleted(String operationType, long durationMs) {
        asyncOperationsCompleted.increment();
        asyncOperationDuration.observe(durationMs / 1000.0);
    }
    
    public void recordAsyncOperationFailed(String operationType) {
        asyncOperationsFailed.increment();
    }
}
```

### Distributed Tracing
```java
@Configuration
public class TracingConfig {
    
    @Bean
    public CompletableFutureDecorator traceCompletableFuture() {
        return (future, spanName) -> {
            Span span = tracer.nextSpan().name(spanName);
            return future
                .whenComplete((result, throwable) -> {
                    if (throwable != null) {
                        span.setStatus(Status.INTERNAL_ERROR);
                        span.log(Map.of("error", throwable.getMessage()));
                    } else {
                        span.setStatus(Status.OK);
                    }
                    span.finish();
                });
        };
    }
}
```

## Testing Asynchronous Code

### Unit Testing with Async
```java
@SpringBootTest
public class AsyncServiceTest {
    
    @Autowired
    private AsyncUserService userService;
    
    @Test
    public void testCreateUserAsync() throws Exception {
        CreateUserRequest request = new CreateUserRequest("John Doe", "john@example.com");
        
        CompletableFuture<User> future = userService.createUserAsync(request);
        
        // Wait for completion with timeout
        User user = future.get(5, TimeUnit.SECONDS);
        
        assertNotNull(user);
        assertEquals("John Doe", user.getName());
        assertEquals("john@example.com", user.getEmail());
    }
    
    @Test
    public void testAsyncOperationFailure() {
        CreateUserRequest invalidRequest = new CreateUserRequest("", "invalid-email");
        
        CompletableFuture<User> future = userService.createUserAsync(invalidRequest);
        
        ExecutionException exception = assertThrows(ExecutionException.class, 
            () -> future.get(5, TimeUnit.SECONDS));
        
        assertTrue(exception.getCause() instanceof ValidationException);
    }
}
```

### Integration Testing
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class AsyncIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private KafkaTestConsumer kafkaConsumer;
    
    @Test
    public void testAsyncOrderProcessing() {
        // Create order
        OrderRequest request = createTestOrderRequest();
        ResponseEntity<OrderResponse> response = restTemplate.postForEntity(
            "/orders", request, OrderResponse.class);
        
        assertEquals(HttpStatus.ACCEPTED, response.getStatusCode());
        String orderId = extractOrderIdFromLocation(response.getHeaders().getLocation());
        
        // Wait for async processing to complete
        await().atMost(10, TimeUnit.SECONDS)
            .until(() -> orderRepository.findById(orderId).isPresent());
        
        // Verify events were published
        List<OrderEvent> events = kafkaConsumer.getConsumedEvents("order-events");
        assertTrue(events.stream().anyMatch(e -> e instanceof OrderCreatedEvent));
        assertTrue(events.stream().anyMatch(e -> e instanceof OrderProcessedEvent));
    }
}
```

## Best Practices

### 1. Error Handling
- **Always handle exceptions**: Use exceptionally() or handle() methods
- **Avoid blocking calls**: Don't call get() without timeouts
- **Graceful degradation**: Continue operation when async operations fail
- **Logging and monitoring**: Track async operation failures

### 2. Resource Management
- **Connection pooling**: Reuse connections for async operations
- **Thread pool sizing**: Configure appropriate thread pool sizes
- **Memory management**: Be aware of memory usage in async operations
- **Timeout configuration**: Set reasonable timeouts for all async operations

### 3. Performance Considerations
- **Batch operations**: Group multiple operations together
- **Parallel execution**: Use parallel streams or completable futures appropriately
- **Caching**: Cache results of expensive async operations
- **Circuit breakers**: Protect against cascading failures

### 4. Testing and Debugging
- **Async-aware testing**: Use appropriate testing frameworks for async code
- **Timeout handling**: Test timeout scenarios
- **Concurrency testing**: Test concurrent async operations
- **Tracing**: Implement distributed tracing for complex async flows

### 5. Design Principles
- **Single Responsibility**: Each async operation should have one clear purpose
- **Immutability**: Use immutable objects in async operations
- **Idempotency**: Make operations safe to retry
- **Observability**: Monitor and log async operations comprehensively

## Real-World Examples

### E-commerce Order Processing
```java
@Service
public class OrderProcessingWorkflow {
    
    public CompletableFuture<Order> processOrderAsync(OrderRequest request) {
        return CompletableFuture.supplyAsync(() -> validateOrder(request))
            .thenCompose(validatedOrder -> 
                CompletableFuture.allOf(
                    inventoryService.reserveInventoryAsync(validatedOrder),
                    fraudService.checkOrderAsync(validatedOrder),
                    taxService.calculateTaxAsync(validatedOrder)
                ).thenApply(v -> validatedOrder)
            )
            .thenCompose(order -> paymentService.processPaymentAsync(order))
            .thenCompose(order -> shippingService.arrangeShippingAsync(order))
            .thenApply(order -> {
                order.setStatus(OrderStatus.CONFIRMED);
                // Send confirmation email asynchronously
                emailService.sendOrderConfirmationAsync(order);
                return orderRepository.save(order);
            })
            .exceptionally(throwable -> {
                logger.error("Order processing failed", throwable);
                // Handle compensation
                handleOrderFailure(request);
                throw new CompletionException(throwable);
            });
    }
}
```

### Social Media Post Processing
```java
@Service
public class PostProcessingService {
    
    @Async
    public CompletableFuture<Post> processNewPost(Post post) {
        return CompletableFuture.completedFuture(post)
            // Validate content
            .thenApply(this::validateContent)
            // Extract hashtags and mentions
            .thenApply(this::extractMetadata)
            // Store in database
            .thenCompose(this::savePost)
            // Update user feed
            .thenCompose(this::updateUserFeed)
            // Send notifications
            .thenCompose(this::sendNotifications)
            // Index for search
            .thenApply(this::indexForSearch)
            .exceptionally(throwable -> {
                logger.error("Post processing failed for post {}", post.getId(), throwable);
                // Mark post as failed
                post.setStatus(PostStatus.FAILED);
                return post;
            });
    }
    
    private CompletableFuture<Post> savePost(Post post) {
        return CompletableFuture.supplyAsync(() -> postRepository.save(post));
    }
    
    private CompletableFuture<Post> updateUserFeed(Post post) {
        return CompletableFuture.runAsync(() -> 
            feedService.addToFollowersFeeds(post)
        ).thenApply(v -> post);
    }
    
    private CompletableFuture<Post> sendNotifications(Post post) {
        return CompletableFuture.runAsync(() -> 
            notificationService.notifyMentionedUsers(post)
        ).thenApply(v -> post);
    }
}
```

## Conclusion

Asynchronous processing patterns are essential for building scalable, responsive systems that can handle high loads and provide excellent user experiences. By leveraging futures, reactive programming, and proper error handling, applications can process operations efficiently without blocking.

**Key Takeaways:**
- **Non-blocking operations**: Keep systems responsive during heavy processing
- **Fault tolerance**: Handle failures gracefully in distributed systems
- **Scalability**: Process more requests with the same resources
- **Error handling**: Comprehensive error handling and recovery mechanisms
- **Monitoring**: Track async operations for performance and reliability

Effective asynchronous processing requires understanding the patterns, proper error handling, monitoring, and testing. Choose the right pattern based on your use case and implement proper safeguards for production reliability.
