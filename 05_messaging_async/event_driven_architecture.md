# Event-Driven Architecture

Event-driven architecture (EDA) is a design pattern where system components communicate through events rather than direct method calls. Events represent significant changes in state that other components can react to. This approach enables loose coupling, scalability, and real-time responsiveness in distributed systems.

## What is Event-Driven Architecture?

Event-driven architecture is a software architecture pattern where system components communicate by producing and consuming events. An event is a notification that something significant has happened in the system.

### Key Concepts
- **Event**: A notification of a state change or action
- **Event Producer/Publisher**: Component that generates events
- **Event Consumer/Subscriber**: Component that reacts to events
- **Event Broker**: Middleware that routes events between producers and consumers
- **Event Store**: Repository for storing events for auditing and replay

### Why Event-Driven Architecture Matters
- **Loose Coupling**: Components don't need to know about each other
- **Scalability**: Components can scale independently
- **Responsiveness**: Real-time reactions to system changes
- **Auditability**: Complete history of system activities
- **Flexibility**: Easy to add new functionality by subscribing to events

## Event Types

### Domain Events
Events that represent business-significant occurrences within a domain.

```java
public class OrderPlacedEvent {
    private final String orderId;
    private final String customerId;
    private final BigDecimal totalAmount;
    private final List<OrderItem> items;
    private final Instant occurredAt;
    
    // Constructor, getters...
}

public class PaymentProcessedEvent {
    private final String orderId;
    private final String paymentId;
    private final PaymentStatus status;
    private final BigDecimal amount;
    private final Instant occurredAt;
    
    // Constructor, getters...
}

public class OrderShippedEvent {
    private final String orderId;
    private final String trackingNumber;
    private final Instant shippedAt;
    
    // Constructor, getters...
}
```

### System Events
Technical events related to system operations and infrastructure.

```java
public class UserLoggedInEvent {
    private final String userId;
    private final String sessionId;
    private final String ipAddress;
    private final Instant occurredAt;
}

public class DatabaseConnectionFailedEvent {
    private final String databaseUrl;
    private final String errorMessage;
    private final Instant occurredAt;
}

public class ServiceHealthChangedEvent {
    private final String serviceName;
    private final HealthStatus previousStatus;
    private final HealthStatus currentStatus;
    private final Instant occurredAt;
}
```

### Integration Events
Events used for communication between different bounded contexts or services.

```java
public class CustomerProfileUpdatedEvent {
    private final String customerId;
    private final String updatedBy;
    private final Map<String, Object> changes;
    private final Instant occurredAt;
}

public class ProductInventoryChangedEvent {
    private final String productId;
    private final int previousStock;
    private final int newStock;
    private final String reason;
    private final Instant occurredAt;
}
```

## Event-Driven Patterns

### Event Sourcing
Store state changes as a sequence of events rather than current state.

```java
@Entity
public class Order {
    @Id
    private String id;
    private String customerId;
    private OrderStatus status;
    private BigDecimal totalAmount;
    
    // Event sourcing: rebuild state from events
    public static Order fromEvents(List<OrderEvent> events) {
        Order order = null;
        for (OrderEvent event : events) {
            if (event instanceof OrderCreatedEvent) {
                OrderCreatedEvent created = (OrderCreatedEvent) event;
                order = new Order(created.getOrderId(), created.getCustomerId());
            } else if (event instanceof OrderPaidEvent) {
                order.setStatus(OrderStatus.PAID);
            } else if (event instanceof OrderShippedEvent) {
                order.setStatus(OrderStatus.SHIPPED);
            }
            // Handle other events...
        }
        return order;
    }
}
```

### CQRS (Command Query Responsibility Segregation)
Separate read and write operations using different models.

```java
// Write Model (Commands)
@Service
public class OrderCommandService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    @Transactional
    public Order createOrder(CreateOrderCommand command) {
        Order order = new Order(command.getCustomerId(), command.getItems());
        orderRepository.save(order);
        
        // Publish event
        OrderCreatedEvent event = new OrderCreatedEvent(order.getId(), order.getCustomerId());
        eventPublisher.publish(event);
        
        return order;
    }
}

// Read Model (Queries)
@Service
public class OrderQueryService {
    
    @Autowired
    private OrderReadRepository orderReadRepository;
    
    public OrderSummary getOrderSummary(String orderId) {
        return orderReadRepository.findOrderSummaryById(orderId);
    }
    
    public List<OrderSummary> getOrdersByCustomer(String customerId) {
        return orderReadRepository.findOrderSummariesByCustomer(customerId);
    }
}
```

### Event Storming
Collaborative process to identify domain events and build event-driven systems.

### Saga Pattern
Manage distributed transactions using events and compensation.

```java
@Service
public class OrderSagaOrchestrator {
    
    @Autowired
    private EventPublisher eventPublisher;
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            // Step 1: Reserve inventory
            inventoryService.reserveInventory(event.getOrderId(), event.getItems());
            
            // Step 2: Process payment
            paymentService.processPayment(event.getOrderId(), event.getTotalAmount());
            
            // Step 3: Confirm order
            orderService.confirmOrder(event.getOrderId());
            
        } catch (Exception e) {
            // Compensate: Cancel order and release resources
            orderService.cancelOrder(event.getOrderId());
            inventoryService.releaseInventory(event.getOrderId());
            
            // Publish compensation event
            OrderFailedEvent failedEvent = new OrderFailedEvent(event.getOrderId(), e.getMessage());
            eventPublisher.publish(failedEvent);
        }
    }
}
```

## Event Processing Styles

### Synchronous Event Processing
Events processed immediately when they occur.

**Advantages:**
- Immediate consistency
- Simple error handling
- Predictable behavior

**Disadvantages:**
- Blocking operations
- Tight coupling
- Performance impact

**Use Cases:**
- Simple business rules
- Immediate user feedback required
- Critical path operations

### Asynchronous Event Processing
Events queued for later processing.

**Advantages:**
- Non-blocking operations
- Loose coupling
- Better scalability

**Disadvantages:**
- Eventual consistency
- Complex error handling
- Harder debugging

**Use Cases:**
- Background processing
- High-volume operations
- Integration with external systems

### Stream Processing
Continuous processing of event streams.

**Advantages:**
- Real-time analytics
- Complex event processing
- State management

**Disadvantages:**
- Complex infrastructure
- State management challenges
- Higher operational complexity

**Use Cases:**
- Real-time dashboards
- Fraud detection
- IoT data processing

## Event Infrastructure

### Event Brokers

#### Apache Kafka
Distributed streaming platform for event-driven architectures.

```java
@Configuration
public class KafkaEventConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

#### RabbitMQ
Message broker with advanced routing capabilities.

```java
@Configuration
public class RabbitMQEventConfig {
    
    @Bean
    public Queue eventQueue() {
        return new Queue("domain-events", true);
    }
    
    @Bean
    public TopicExchange eventExchange() {
        return new TopicExchange("domain-events-exchange");
    }
    
    @Bean
    public Binding binding(Queue queue, TopicExchange exchange) {
        return BindingBuilder.bind(queue).to(exchange).with("order.#");
    }
}
```

### Event Stores

#### Event Sourcing Database
Specialized database for storing events.

```java
@Repository
public class EventStoreRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public void saveEvent(DomainEvent event) {
        jdbcTemplate.update(
            "INSERT INTO events (aggregate_id, event_type, event_data, occurred_at) VALUES (?, ?, ?, ?)",
            event.getAggregateId(),
            event.getClass().getSimpleName(),
            serializeEvent(event),
            event.getOccurredAt()
        );
    }
    
    public List<DomainEvent> getEventsForAggregate(String aggregateId) {
        return jdbcTemplate.query(
            "SELECT * FROM events WHERE aggregate_id = ? ORDER BY occurred_at",
            (rs, rowNum) -> deserializeEvent(rs.getString("event_type"), rs.getString("event_data")),
            aggregateId
        );
    }
}
```

### Event Serialization

#### JSON Serialization
```java
@Component
public class JsonEventSerializer implements EventSerializer {
    
    private final ObjectMapper objectMapper;
    
    public JsonEventSerializer() {
        this.objectMapper = new ObjectMapper();
        this.objectMapper.registerModule(new JavaTimeModule());
    }
    
    @Override
    public String serialize(DomainEvent event) {
        try {
            return objectMapper.writeValueAsString(event);
        } catch (JsonProcessingException e) {
            throw new EventSerializationException("Failed to serialize event", e);
        }
    }
    
    @Override
    public <T extends DomainEvent> T deserialize(String eventType, String eventData) {
        try {
            Class<?> eventClass = Class.forName("com.example.events." + eventType);
            return (T) objectMapper.readValue(eventData, eventClass);
        } catch (Exception e) {
            throw new EventSerializationException("Failed to deserialize event", e);
        }
    }
}
```

## Event-Driven Microservices

### Service Communication
```java
@Service
public class OrderService {
    
    @Autowired
    private EventPublisher eventPublisher;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // Create order
        Order order = new Order(request.getCustomerId(), request.getItems());
        orderRepository.save(order);
        
        // Publish domain event
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotalAmount(),
            Instant.now()
        );
        eventPublisher.publish("orders", event);
        
        return order;
    }
}
```

### Event Subscribers
```java
@Component
public class OrderEventHandlers {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private NotificationService notificationService;
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Reserve inventory
        inventoryService.reserveItems(event.getOrderId(), event.getItems());
        
        // Send confirmation email
        notificationService.sendOrderConfirmation(event.getCustomerId(), event.getOrderId());
    }
    
    @EventListener
    public void handleOrderPaid(OrderPaidEvent event) {
        // Update inventory
        inventoryService.confirmReservation(event.getOrderId());
        
        // Send shipping notification
        notificationService.sendShippingNotification(event.getCustomerId(), event.getOrderId());
    }
}
```

### Cross-Service Events
```java
@Service
public class CustomerEventPublisher {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publishCustomerUpdated(Customer customer, Map<String, Object> changes) {
        CustomerUpdatedEvent event = new CustomerUpdatedEvent(
            customer.getId(),
            changes,
            Instant.now()
        );
        
        // Publish to shared topic for cross-service communication
        kafkaTemplate.send("customer-events", customer.getId(), event);
    }
}
```

## Event Versioning and Evolution

### Event Schema Evolution
```java
// Version 1
public class OrderCreatedEventV1 {
    private String orderId;
    private String customerId;
    private BigDecimal totalAmount;
}

// Version 2 - Added shipping address
public class OrderCreatedEventV2 {
    private String orderId;
    private String customerId;
    private BigDecimal totalAmount;
    private Address shippingAddress;
}

// Version handling
public class OrderCreatedEventHandler {
    
    @EventListener
    public void handleOrderCreated(DomainEvent rawEvent) {
        if (rawEvent instanceof OrderCreatedEventV1) {
            handleV1((OrderCreatedEventV1) rawEvent);
        } else if (rawEvent instanceof OrderCreatedEventV2) {
            handleV2((OrderCreatedEventV2) rawEvent);
        } else {
            throw new UnsupportedEventVersionException(rawEvent.getClass());
        }
    }
    
    private void handleV1(OrderCreatedEventV1 event) {
        // Handle version 1 logic
        processOrder(event.getOrderId(), event.getCustomerId(), event.getTotalAmount(), null);
    }
    
    private void handleV2(OrderCreatedEventV2 event) {
        // Handle version 2 logic
        processOrder(event.getOrderId(), event.getCustomerId(), 
                    event.getTotalAmount(), event.getShippingAddress());
    }
}
```

### Upcasting Events
Transform old event versions to current versions.

```java
public class EventUpcaster {
    
    public DomainEvent upcast(DomainEvent event) {
        if (event instanceof OrderCreatedEventV1) {
            OrderCreatedEventV1 v1 = (OrderCreatedEventV1) event;
            return new OrderCreatedEventV2(
                v1.getOrderId(),
                v1.getCustomerId(),
                v1.getTotalAmount(),
                null // Default shipping address
            );
        }
        return event;
    }
}
```

## Event Reliability and Delivery

### At-Least-Once Delivery
Ensure events are delivered at least once, possibly with duplicates.

```java
@Service
public class ReliableEventPublisher {
    
    @Autowired
    private EventRepository eventRepository;
    
    @Autowired
    private MessageBroker broker;
    
    @Transactional
    public void publishEvent(DomainEvent event) {
        // Store event first
        EventEntity storedEvent = new EventEntity(event, EventStatus.PENDING);
        eventRepository.save(storedEvent);
        
        try {
            // Publish to broker
            broker.publish(event);
            
            // Mark as published
            storedEvent.setStatus(EventStatus.PUBLISHED);
            eventRepository.save(storedEvent);
            
        } catch (Exception e) {
            // Event will be retried by background process
            logger.error("Failed to publish event {}", event.getId(), e);
        }
    }
}
```

### Exactly-Once Delivery
Ensure events are delivered exactly once (challenging in distributed systems).

```java
@Service
public class IdempotentEventProcessor {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private EventProcessor processor;
    
    public void processEvent(DomainEvent event) {
        String processedKey = "processed:" + event.getClass().getSimpleName() + ":" + event.getId();
        
        // Check if already processed
        Boolean alreadyProcessed = redisTemplate.hasKey(processedKey);
        if (Boolean.TRUE.equals(alreadyProcessed)) {
            logger.info("Event {} already processed", event.getId());
            return;
        }
        
        try {
            // Process event
            processor.process(event);
            
            // Mark as processed (with TTL)
            redisTemplate.opsForValue().set(processedKey, "true", Duration.ofDays(7));
            
        } catch (Exception e) {
            logger.error("Failed to process event {}", event.getId(), e);
            // Event will be retried
        }
    }
}
```

## Event Monitoring and Observability

### Event Metrics
```java
@Service
public class EventMetricsService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter eventsPublished = Counter.builder("events_published_total")
        .description("Total events published")
        .tag("event_type", "")
        .register(meterRegistry);
    
    private final Counter eventsConsumed = Counter.builder("events_consumed_total")
        .description("Total events consumed")
        .tag("consumer", "")
        .register(meterRegistry);
    
    private final Counter eventsFailed = Counter.builder("events_failed_total")
        .description("Total event processing failures")
        .register(meterRegistry);
    
    private final Histogram eventProcessingTime = Histogram.builder("event_processing_duration")
        .description("Event processing time")
        .register(meterRegistry);
    
    public void recordEventPublished(String eventType) {
        eventsPublished.withTag("event_type", eventType).increment();
    }
    
    public void recordEventConsumed(String consumer, long processingTimeMs) {
        eventsConsumed.withTag("consumer", consumer).increment();
        eventProcessingTime.observe(processingTimeMs);
    }
    
    public void recordEventFailed(String eventType) {
        eventsFailed.withTag("event_type", eventType).increment();
    }
}
```

### Event Tracing
```java
@Configuration
public class EventTracingConfig {
    
    @Bean
    public EventPublisher tracedEventPublisher(EventPublisher delegate) {
        return new TracedEventPublisher(delegate);
    }
}

public class TracedEventPublisher implements EventPublisher {
    
    private final EventPublisher delegate;
    
    public TracedEventPublisher(EventPublisher delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public void publish(String topic, DomainEvent event) {
        Span span = tracer.nextSpan().name("publish-event")
            .tag("event.type", event.getClass().getSimpleName())
            .tag("event.id", event.getId())
            .tag("topic", topic);
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            delegate.publish(topic, event);
            span.setStatus(Status.OK);
        } catch (Exception e) {
            span.setStatus(Status.INTERNAL_ERROR);
            span.log(Map.of("error", e.getMessage()));
            throw e;
        } finally {
            span.finish();
        }
    }
}
```

## Event-Driven Best Practices

### Event Design
- **Descriptive Names**: Use past tense (OrderCreated, PaymentProcessed)
- **Immutable Data**: Events should be immutable
- **Business Context**: Include relevant business information
- **Versioning**: Plan for event schema evolution
- **Size Considerations**: Keep events reasonably sized

### Publisher Guidelines
- **Fire and Forget**: Don't wait for event processing
- **Idempotent Publishing**: Safe to publish same event multiple times
- **Correlation IDs**: Include for request tracing
- **Metadata**: Add timestamps, source information
- **Error Handling**: Handle publishing failures gracefully

### Consumer Guidelines
- **Idempotent Processing**: Handle duplicate events safely
- **Independent Processing**: Don't depend on processing order
- **Error Isolation**: Failures shouldn't affect other events
- **Monitoring**: Track processing success/failure rates
- **Scalability**: Support multiple consumer instances

### Infrastructure Considerations
- **Event Ordering**: Don't assume event ordering unless guaranteed
- **Eventual Consistency**: Design for eventual consistency
- **Backpressure**: Handle slow consumers appropriately
- **Security**: Authenticate and authorize event access

## Common Event-Driven Challenges

### Event Ordering
**Problem:** Events may arrive out of order
**Solutions:**
- Include sequence numbers or timestamps
- Use event sourcing with version checking
- Design consumers to handle out-of-order events

### Event Duplication
**Problem:** Same event processed multiple times
**Solutions:**
- Idempotent event processing
- Deduplication using event IDs
- Optimistic locking in consumers

### Event Schema Evolution
**Problem:** Changing event structures breaks consumers
**Solutions:**
- Version events explicitly
- Use upcasters for backward compatibility
- Provide migration paths for old events

### Debugging Complex Event Flows
**Problem:** Hard to trace event chains across services
**Solutions:**
- Distributed tracing with correlation IDs
- Event logging and monitoring
- Event replay capabilities for debugging

## Real-World Event-Driven Examples

### E-commerce Platform
```java
@Service
public class OrderEventOrchestrator {
    
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Parallel processing of order components
        CompletableFuture<Void> inventoryCheck = CompletableFuture.runAsync(() -> 
            inventoryService.checkAvailability(event.getItems()));
        
        CompletableFuture<Void> fraudCheck = CompletableFuture.runAsync(() -> 
            fraudService.checkOrder(event.getOrderId(), event.getCustomerId()));
        
        CompletableFuture<Void> paymentAuth = CompletableFuture.runAsync(() -> 
            paymentService.authorizePayment(event.getOrderId(), event.getTotalAmount()));
        
        // When all checks complete
        CompletableFuture.allOf(inventoryCheck, fraudCheck, paymentAuth)
            .thenRun(() -> {
                if (allChecksPassed()) {
                    eventPublisher.publish(new OrderApprovedEvent(event.getOrderId()));
                } else {
                    eventPublisher.publish(new OrderRejectedEvent(event.getOrderId(), getFailureReason()));
                }
            });
    }
}
```

### IoT Sensor Network
```java
@Service
public class SensorDataProcessor {
    
    @Autowired
    private AnomalyDetector anomalyDetector;
    
    @Autowired
    private AlertService alertService;
    
    @EventListener
    public void handleSensorReading(SensorReadingEvent event) {
        // Process reading
        SensorReading reading = event.getReading();
        
        // Check for anomalies
        if (anomalyDetector.isAnomalous(reading)) {
            alertService.sendAlert(new AnomalyAlert(
                reading.getSensorId(),
                reading.getValue(),
                reading.getTimestamp()
            ));
        }
        
        // Update aggregates
        aggregateService.updateHourlyAggregate(reading);
        
        // Trigger rules
        ruleEngine.evaluateRules(reading);
    }
}
```

### Financial Trading System
```java
@Service
public class TradingEventProcessor {
    
    @EventListener
    public void handleOrderSubmitted(OrderSubmittedEvent event) {
        // Validate order
        validationService.validateOrder(event.getOrder());
        
        // Check risk limits
        riskService.checkRiskLimits(event.getOrder());
        
        // Execute order
        executionService.executeOrder(event.getOrder());
        
        // Publish execution event
        eventPublisher.publish(new OrderExecutedEvent(
            event.getOrderId(),
            executionResult,
            Instant.now()
        ));
    }
    
    @EventListener
    public void handleOrderExecuted(OrderExecutedEvent event) {
        // Update positions
        positionService.updatePositions(event.getExecutionResult());
        
        // Send confirmations
        notificationService.sendTradeConfirmation(event.getOrderId());
        
        // Update analytics
        analyticsService.recordTrade(event.getExecutionResult());
    }
}
```

## Conclusion

Event-driven architecture provides a powerful way to build loosely coupled, scalable systems that can react to changes in real-time. By using events as the primary communication mechanism, systems become more flexible, maintainable, and responsive.

**Key Takeaways:**
- **Loose Coupling**: Components communicate through events, not direct calls
- **Scalability**: Components can scale independently based on event load
- **Flexibility**: Easy to add new functionality by subscribing to events
- **Auditability**: Complete event history for debugging and analytics
- **Real-time**: Immediate reactions to system state changes

Successful event-driven architecture requires careful event design, reliable infrastructure, and proper monitoring. Events should be well-defined, immutable, and contain all necessary context for consumers to process them effectively.
