# Distributed Tracing

Distributed tracing tracks requests as they flow through distributed systems, providing end-to-end visibility into application performance and behavior. It enables debugging, performance analysis, and root cause identification across service boundaries. This guide covers distributed tracing concepts, implementations, and best practices.

## What is Distributed Tracing?

Distributed tracing follows a request's journey through multiple services in a distributed system. Each service adds timing and metadata information, creating a complete trace that shows the request's path, latency at each step, and any errors encountered.

### Key Concepts
- **Trace**: Complete journey of a request through the system
- **Span**: Individual operation within a trace (database query, HTTP call, etc.)
- **Trace ID**: Unique identifier for the entire request
- **Span ID**: Unique identifier for each operation
- **Parent Span ID**: Links spans in the call hierarchy
- **Baggage**: Context data propagated across service boundaries

### Tracing Components
- **Instrumentation**: Code that creates and manages spans
- **Collector**: Receives and processes trace data
- **Storage**: Persists trace data for analysis
- **UI/Dashboard**: Visualizes traces for debugging and monitoring

## Tracing Standards and Protocols

### OpenTracing (Now OpenTelemetry)
```java
@Configuration
public class OpenTelemetryConfig {
    
    @Bean
    public Tracer tracer() {
        // Configure OpenTelemetry tracer
        OpenTelemetrySdk openTelemetry = OpenTelemetrySdk.builder()
            .setTracerProvider(
                SdkTracerProvider.builder()
                    .addSpanProcessor(BatchSpanProcessor.builder(
                        OtlpGrpcSpanExporter.builder()
                            .setEndpoint("http://jaeger:4317")
                            .build()
                    ).build())
                    .build()
            )
            .build();
        
        return openTelemetry.getTracer("order-service");
    }
}

@Service
public class OrderService {
    
    @Autowired
    private Tracer tracer;
    
    @Autowired
    private UserServiceClient userService;
    
    @Autowired
    private InventoryService inventoryService;
    
    public Order createOrder(CreateOrderRequest request) {
        Span span = tracer.spanBuilder("createOrder")
            .setAttribute("userId", request.getUserId().toString())
            .setAttribute("itemCount", request.getItems().size())
            .startSpan();
        
        try (Tracer.SpanInScope scope = span.makeCurrent()) {
            // Validate user
            Span userSpan = tracer.spanBuilder("validateUser")
                .setAttribute("userId", request.getUserId().toString())
                .startSpan();
            
            try (Tracer.SpanInScope userScope = userSpan.makeCurrent()) {
                User user = userService.getUserById(request.getUserId());
                userSpan.setAttribute("userFound", user != null);
                
                if (user == null) {
                    userSpan.addEvent("User not found", Map.of("userId", request.getUserId().toString()));
                    userSpan.setStatus(StatusCode.ERROR, "User not found");
                    throw new UserNotFoundException(request.getUserId());
                }
            } finally {
                userSpan.end();
            }
            
            // Check inventory
            Span inventorySpan = tracer.spanBuilder("checkInventory")
                .setAttribute("itemCount", request.getItems().size())
                .startSpan();
            
            try (Tracer.SpanInScope inventoryScope = inventorySpan.makeCurrent()) {
                for (OrderItem item : request.getItems()) {
                    boolean available = inventoryService.checkAvailability(item.getProductId(), item.getQuantity());
                    inventorySpan.addEvent("Checked item availability", Map.of(
                        "productId", item.getProductId().toString(),
                        "quantity", item.getQuantity(),
                        "available", available
                    ));
                    
                    if (!available) {
                        inventorySpan.setStatus(StatusCode.ERROR, "Insufficient inventory");
                        throw new InsufficientInventoryException(item.getProductId());
                    }
                }
            } finally {
                inventorySpan.end();
            }
            
            // Create order
            Span dbSpan = tracer.spanBuilder("saveOrder")
                .setAttribute("operation", "INSERT")
                .setAttribute("table", "orders")
                .startSpan();
            
            Order order = null;
            try (Tracer.SpanInScope dbScope = dbSpan.makeCurrent()) {
                order = new Order(request.getUserId(), request.getItems(), Instant.now());
                order = orderRepository.save(order);
                dbSpan.setAttribute("orderId", order.getId().toString());
            } finally {
                dbSpan.end();
            }
            
            span.setAttribute("orderId", order.getId().toString());
            span.addEvent("Order created successfully");
            
            return order;
            
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### Jaeger Implementation
```java
@Configuration
public class JaegerConfig {
    
    @Bean
    public Tracer jaegerTracer() {
        Configuration.SamplerConfiguration samplerConfig = Configuration.SamplerConfiguration.fromEnv()
            .withType(ConstSampler.TYPE)
            .withParam(1);
        
        Configuration.ReporterConfiguration reporterConfig = Configuration.ReporterConfiguration.fromEnv()
            .withLogSpans(true)
            .withSender(Configuration.SenderConfiguration.fromEnv()
                .withAgentHost("jaeger-agent")
                .withAgentPort(6831));
        
        Configuration config = new Configuration("order-service")
            .withSampler(samplerConfig)
            .withReporter(reporterConfig);
        
        return config.getTracer();
    }
}

@Service
public class JaegerTracedService {
    
    @Autowired
    private Tracer tracer;
    
    public void processPayment(String orderId, BigDecimal amount) {
        Span span = tracer.buildSpan("processPayment")
            .withTag("orderId", orderId)
            .withTag("amount", amount.toString())
            .start();
        
        try (Scope scope = tracer.scopeManager().activate(span)) {
            // Simulate payment processing
            span.log("Starting payment processing");
            
            // Call external payment service
            Span externalSpan = tracer.buildSpan("callPaymentProvider")
                .withTag("provider", "stripe")
                .start();
            
            try (Scope externalScope = tracer.scopeManager().activate(externalSpan)) {
                // External API call
                externalSpan.log("Calling payment provider API");
                Thread.sleep(100); // Simulate network delay
                
                externalSpan.log("Payment processed successfully");
            } finally {
                externalSpan.finish();
            }
            
            span.log("Payment processing completed");
            
        } catch (Exception e) {
            span.log("Payment processing failed");
            span.setTag("error", true);
            Tags.ERROR.set(span, true);
            throw e;
        } finally {
            span.finish();
        }
    }
}
```

## Instrumentation Patterns

### Manual Instrumentation
```java
@Service
public class ManualInstrumentationService {
    
    @Autowired
    private Tracer tracer;
    
    @Autowired
    private HttpClient httpClient;
    
    public String callExternalService(String url) {
        Span span = tracer.buildSpan("callExternalService")
            .withTag("http.method", "GET")
            .withTag("http.url", url)
            .start();
        
        try (Scope scope = tracer.scopeManager().activate(span)) {
            // Set span tags
            span.setTag("component", "http-client");
            span.setTag("span.kind", "client");
            
            // Inject trace context into HTTP headers
            Headers headers = new Headers();
            tracer.inject(span.context(), Format.Builtin.HTTP_HEADERS, new TextMapAdapter(headers));
            
            // Make HTTP call
            span.log("Sending HTTP request");
            long startTime = System.currentTimeMillis();
            
            HttpResponse response = httpClient.get(url, headers);
            
            long duration = System.currentTimeMillis() - startTime;
            span.setTag("http.status_code", response.getStatusCode());
            span.setTag("duration_ms", duration);
            
            span.log("Received HTTP response");
            
            return response.getBody();
            
        } catch (Exception e) {
            span.setTag("error", true);
            span.log(Map.of("event", "error", "error.object", e));
            throw e;
        } finally {
            span.finish();
        }
    }
}
```

### Automatic Instrumentation
```java
@Configuration
public class AutoInstrumentationConfig {
    
    @Bean
    public GlobalTracer globalTracer(Tracer tracer) {
        GlobalTracer.register(tracer);
        return GlobalTracer.get();
    }
    
    // Spring Boot Actuator integration
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}

@RestController
public class TracedController {
    
    @Autowired
    private OrderService orderService;
    
    @GetMapping("/orders/{orderId}")
    @Timed(value = "order.get", percentiles = {0.5, 0.95, 0.99})
    public Order getOrder(@PathVariable Long orderId) {
        // This method will be automatically traced due to @Timed annotation
        // and any spans created within will be properly nested
        return orderService.getOrder(orderId);
    }
    
    @PostMapping("/orders")
    @Timed(value = "order.create", percentiles = {0.5, 0.95, 0.99})
    public Order createOrder(@RequestBody CreateOrderRequest request) {
        return orderService.createOrder(request);
    }
}
```

### Context Propagation
```java
@Component
public class TraceContextPropagator {
    
    private static final String TRACE_ID_HEADER = "X-Trace-Id";
    private static final String SPAN_ID_HEADER = "X-Span-Id";
    private static final String BAGGAGE_HEADER_PREFIX = "X-Baggage-";
    
    public void injectTraceContext(SpanContext context, Map<String, String> headers) {
        if (context != null) {
            headers.put(TRACE_ID_HEADER, context.getTraceId());
            headers.put(SPAN_ID_HEADER, context.getSpanId());
            
            // Inject baggage items
            context.baggageItems().forEach((key, value) -> 
                headers.put(BAGGAGE_HEADER_PREFIX + key, value));
        }
    }
    
    public SpanContext extractTraceContext(Map<String, String> headers) {
        String traceId = headers.get(TRACE_ID_HEADER);
        String spanId = headers.get(SPAN_ID_HEADER);
        
        if (traceId != null && spanId != null) {
            // Extract baggage items
            Map<String, String> baggage = headers.entrySet().stream()
                .filter(entry -> entry.getKey().startsWith(BAGGAGE_HEADER_PREFIX))
                .collect(Collectors.toMap(
                    entry -> entry.getKey().substring(BAGGAGE_HEADER_PREFIX.length()),
                    Map.Entry::getValue
                ));
            
            return new SpanContext(traceId, spanId, baggage);
        }
        
        return null;
    }
}

@Service
public class HttpClientWithTracing {
    
    @Autowired
    private TraceContextPropagator propagator;
    
    @Autowired
    private Tracer tracer;
    
    public <T> T executeWithTracing(String url, Supplier<T> operation) {
        Span span = tracer.buildSpan("http-client-call")
            .withTag("http.url", url)
            .withTag("component", "http-client")
            .start();
        
        try (Scope scope = tracer.scopeManager().activate(span)) {
            // Inject trace context into request
            Map<String, String> headers = new HashMap<>();
            propagator.injectTraceContext(span.context(), headers);
            
            // Execute operation with headers
            span.log("Executing HTTP request");
            T result = operation.get();
            span.log("HTTP request completed");
            
            return result;
            
        } catch (Exception e) {
            span.setTag("error", true);
            span.log(Map.of("event", "error", "error.object", e));
            throw e;
        } finally {
            span.finish();
        }
    }
}
```

## Sampling Strategies

### Probability Sampling
```java
@Configuration
public class SamplingConfig {
    
    @Bean
    public Sampler probabilitySampler() {
        return Sampler.probability(0.1); // Sample 10% of traces
    }
    
    @Bean
    public Sampler adaptiveSampler() {
        return new AdaptiveSampler();
    }
}

public class AdaptiveSampler implements Sampler {
    
    private final AtomicLong requestCount = new AtomicLong(0);
    private final AtomicLong sampledCount = new AtomicLong(0);
    private volatile double samplingRate = 0.1; // Start with 10%
    
    @Override
    public SamplingDecision shouldSample(SpanContext parentContext, String traceId, 
                                       String spanName, SpanKind spanKind, 
                                       Map<String, Object> tags, List<Link> links) {
        
        long totalRequests = requestCount.incrementAndGet();
        
        // Adjust sampling rate based on system load
        double targetRate = calculateTargetSamplingRate();
        
        if (Math.abs(targetRate - samplingRate) > 0.01) {
            samplingRate = targetRate;
        }
        
        boolean sample = ThreadLocalRandom.current().nextDouble() < samplingRate;
        
        if (sample) {
            sampledCount.incrementAndGet();
        }
        
        return new SamplingDecision(sample, Map.of("sampling.rate", String.valueOf(samplingRate)));
    }
    
    private double calculateTargetSamplingRate() {
        // Simple adaptive logic - reduce sampling under high load
        double systemLoad = getSystemLoad();
        
        if (systemLoad > 0.8) { // High load
            return 0.05; // Reduce to 5%
        } else if (systemLoad > 0.6) { // Medium load
            return 0.1;  // 10%
        } else {
            return 0.2;  // 20% for normal load
        }
    }
    
    private double getSystemLoad() {
        // Get system load (simplified)
        OperatingSystemMXBean osBean = ManagementFactory.getOperatingSystemMXBean();
        return osBean.getSystemLoadAverage() / osBean.getAvailableProcessors();
    }
}
```

### Tail-Based Sampling
```java
@Service
public class TailBasedSampler {
    
    private final Map<String, TraceBuffer> activeTraces = new ConcurrentHashMap<>();
    private final ScheduledExecutorService cleanupExecutor = Executors.newScheduledThreadPool(1);
    
    public TailBasedSampler() {
        // Clean up old traces periodically
        cleanupExecutor.scheduleAtFixedRate(this::cleanupOldTraces, 1, 1, TimeUnit.MINUTES);
    }
    
    public SamplingDecision shouldSample(String traceId, Span span) {
        TraceBuffer buffer = activeTraces.computeIfAbsent(traceId, k -> new TraceBuffer());
        buffer.addSpan(span);
        
        // Always sample initially, make decision later
        return SamplingDecision.SAMPLE;
    }
    
    public void finalizeTrace(String traceId) {
        TraceBuffer buffer = activeTraces.remove(traceId);
        
        if (buffer != null) {
            boolean shouldKeep = shouldKeepTrace(buffer);
            
            if (!shouldKeep) {
                // Drop the trace
                buffer.drop();
            }
        }
    }
    
    private boolean shouldKeepTrace(TraceBuffer buffer) {
        // Keep traces with errors
        if (buffer.hasErrors()) {
            return true;
        }
        
        // Keep traces with high latency (tail sampling)
        if (buffer.getTotalDuration() > 5000) { // 5 seconds
            return true;
        }
        
        // Keep traces with specific tags
        if (buffer.hasTag("important", "true")) {
            return true;
        }
        
        // Sample based on probability for normal traces
        return ThreadLocalRandom.current().nextDouble() < 0.1; // 10%
    }
    
    private void cleanupOldTraces() {
        long cutoffTime = System.currentTimeMillis() - TimeUnit.MINUTES.toMillis(5);
        
        activeTraces.entrySet().removeIf(entry -> 
            entry.getValue().getLastActivityTime() < cutoffTime);
    }
    
    public static class TraceBuffer {
        private final List<Span> spans = new ArrayList<>();
        private long lastActivityTime = System.currentTimeMillis();
        
        public void addSpan(Span span) {
            spans.add(span);
            lastActivityTime = System.currentTimeMillis();
        }
        
        public boolean hasErrors() {
            return spans.stream().anyMatch(span -> span.tags().containsKey("error"));
        }
        
        public long getTotalDuration() {
            if (spans.isEmpty()) return 0;
            
            long startTime = spans.stream()
                .mapToLong(Span::startTime)
                .min()
                .orElse(0);
                
            long endTime = spans.stream()
                .mapToLong(span -> span.startTime() + span.duration())
                .max()
                .orElse(0);
                
            return endTime - startTime;
        }
        
        public boolean hasTag(String key, String value) {
            return spans.stream()
                .anyMatch(span -> value.equals(span.tags().get(key)));
        }
        
        public long getLastActivityTime() {
            return lastActivityTime;
        }
        
        public void drop() {
            // Mark spans for dropping
            spans.forEach(span -> span.setTag("dropped", "true"));
        }
    }
}
```

## Tracing Storage and Analysis

### Trace Storage Backends

#### Elasticsearch Storage
```java
@Configuration
public class ElasticsearchTraceStorage {
    
    @Bean
    public SpanProcessor elasticsearchSpanProcessor() {
        return BatchSpanProcessor.builder(
            ElasticsearchSpanExporter.builder()
                .setEndpoint("http://elasticsearch:9200")
                .setIndex("traces")
                .build()
        ).build();
    }
}

@Service
public class TraceQueryService {
    
    @Autowired
    private RestHighLevelClient elasticsearchClient;
    
    public List<Trace> findTracesByService(String serviceName, Instant startTime, Instant endTime) {
        SearchRequest searchRequest = new SearchRequest("traces");
        
        BoolQueryBuilder queryBuilder = QueryBuilders.boolQuery()
            .must(QueryBuilders.termQuery("service.name", serviceName))
            .must(QueryBuilders.rangeQuery("timestamp")
                .gte(startTime.toEpochMilli())
                .lte(endTime.toEpochMilli()));
        
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder()
            .query(queryBuilder)
            .size(1000)
            .sort("timestamp", SortOrder.DESC);
        
        searchRequest.source(sourceBuilder);
        
        try {
            SearchResponse response = elasticsearchClient.search(searchRequest, RequestOptions.DEFAULT);
            return parseTraceResults(response);
        } catch (IOException e) {
            logger.error("Failed to query traces", e);
            return Collections.emptyList();
        }
    }
    
    public Trace getTraceById(String traceId) {
        GetRequest getRequest = new GetRequest("traces", traceId);
        
        try {
            GetResponse response = elasticsearchClient.get(getRequest, RequestOptions.DEFAULT);
            return parseTrace(response);
        } catch (IOException e) {
            logger.error("Failed to get trace {}", traceId, e);
            return null;
        }
    }
    
    private List<Trace> parseTraceResults(SearchResponse response) {
        return Arrays.stream(response.getHits().getHits())
            .map(hit -> parseTrace(hit))
            .collect(Collectors.toList());
    }
    
    private Trace parseTrace(GetResponse response) {
        // Parse trace from Elasticsearch document
        return null; // Implementation would parse JSON
    }
}
```

### Trace Analysis and Visualization

#### Performance Analysis
```java
@Service
public class TracePerformanceAnalyzer {
    
    @Autowired
    private TraceQueryService traceQueryService;
    
    public PerformanceReport analyzeServicePerformance(String serviceName, Duration timeWindow) {
        Instant endTime = Instant.now();
        Instant startTime = endTime.minus(timeWindow);
        
        List<Trace> traces = traceQueryService.findTracesByService(serviceName, startTime, endTime);
        
        PerformanceReport report = new PerformanceReport();
        report.setServiceName(serviceName);
        report.setTimeWindow(timeWindow);
        report.setTotalRequests(traces.size());
        
        // Calculate response time percentiles
        List<Long> responseTimes = traces.stream()
            .mapToLong(Trace::getTotalDuration)
            .boxed()
            .collect(Collectors.toList());
        
        Collections.sort(responseTimes);
        
        report.setP50ResponseTime(calculatePercentile(responseTimes, 50));
        report.setP95ResponseTime(calculatePercentile(responseTimes, 95));
        report.setP99ResponseTime(calculatePercentile(responseTimes, 99));
        
        // Calculate error rate
        long errorCount = traces.stream()
            .mapToLong(trace -> trace.getSpans().stream()
                .anyMatch(span -> span.hasError()) ? 1 : 0)
            .sum();
        
        report.setErrorRate((double) errorCount / traces.size());
        
        // Identify slow traces
        List<Trace> slowTraces = traces.stream()
            .filter(trace -> trace.getTotalDuration() > report.getP95ResponseTime())
            .collect(Collectors.toList());
        
        report.setSlowTraces(slowTraces);
        
        return report;
    }
    
    private long calculatePercentile(List<Long> sortedValues, double percentile) {
        if (sortedValues.isEmpty()) return 0;
        
        int index = (int) Math.ceil((percentile / 100.0) * sortedValues.size()) - 1;
        return sortedValues.get(Math.max(0, Math.min(index, sortedValues.size() - 1)));
    }
    
    public List<String> identifyBottlenecks(String serviceName, Duration timeWindow) {
        List<String> bottlenecks = new ArrayList<>();
        
        PerformanceReport report = analyzeServicePerformance(serviceName, timeWindow);
        
        // Check for high P95 response time
        if (report.getP95ResponseTime() > 2000) { // 2 seconds
            bottlenecks.add("High response time (P95: " + report.getP95ResponseTime() + "ms)");
        }
        
        // Check for high error rate
        if (report.getErrorRate() > 0.05) { // 5%
            bottlenecks.add("High error rate (" + String.format("%.2f%%", report.getErrorRate() * 100) + ")");
        }
        
        // Analyze slow traces for patterns
        Map<String, Long> operationCounts = report.getSlowTraces().stream()
            .flatMap(trace -> trace.getSpans().stream())
            .collect(Collectors.groupingBy(
                Span::getOperationName,
                Collectors.counting()
            ));
        
        operationCounts.entrySet().stream()
            .filter(entry -> entry.getValue() > report.getSlowTraces().size() * 0.5) // In >50% of slow traces
            .forEach(entry -> bottlenecks.add("Frequent slow operation: " + entry.getKey()));
        
        return bottlenecks;
    }
}
```

#### Error Analysis
```java
@Service
public class TraceErrorAnalyzer {
    
    @Autowired
    private TraceQueryService traceQueryService;
    
    public ErrorAnalysisReport analyzeErrors(String serviceName, Duration timeWindow) {
        Instant endTime = Instant.now();
        Instant startTime = endTime.minus(timeWindow);
        
        List<Trace> traces = traceQueryService.findTracesByService(serviceName, startTime, endTime);
        
        ErrorAnalysisReport report = new ErrorAnalysisReport();
        report.setServiceName(serviceName);
        report.setTimeWindow(timeWindow);
        
        // Group errors by type
        Map<String, List<Trace>> errorsByType = traces.stream()
            .filter(trace -> trace.getSpans().stream().anyMatch(Span::hasError))
            .collect(Collectors.groupingBy(this::getErrorType));
        
        report.setErrorsByType(errorsByType);
        
        // Calculate error rate over time
        Map<Instant, Double> errorRateOverTime = calculateErrorRateOverTime(traces, timeWindow);
        report.setErrorRateOverTime(errorRateOverTime);
        
        // Find common error patterns
        List<ErrorPattern> patterns = identifyErrorPatterns(errorsByType);
        report.setErrorPatterns(patterns);
        
        return report;
    }
    
    private String getErrorType(Trace trace) {
        return trace.getSpans().stream()
            .filter(Span::hasError)
            .findFirst()
            .map(span -> span.getTags().get("error.type"))
            .orElse("Unknown");
    }
    
    private Map<Instant, Double> calculateErrorRateOverTime(List<Trace> traces, Duration timeWindow) {
        // Group traces by time buckets
        Map<Instant, List<Trace>> tracesByTime = traces.stream()
            .collect(Collectors.groupingBy(trace -> {
                // Round to nearest minute
                long timestamp = trace.getStartTime().toEpochMilli();
                long rounded = (timestamp / 60000) * 60000;
                return Instant.ofEpochMilli(rounded);
            }));
        
        return tracesByTime.entrySet().stream()
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                entry -> {
                    long totalTraces = entry.getValue().size();
                    long errorTraces = entry.getValue().stream()
                        .mapToLong(trace -> trace.hasError() ? 1 : 0)
                        .sum();
                    return totalTraces > 0 ? (double) errorTraces / totalTraces : 0.0;
                }
            ));
    }
    
    private List<ErrorPattern> identifyErrorPatterns(Map<String, List<Trace>> errorsByType) {
        List<ErrorPattern> patterns = new ArrayList<>();
        
        for (Map.Entry<String, List<Trace>> entry : errorsByType.entrySet()) {
            String errorType = entry.getKey();
            List<Trace> errorTraces = entry.getValue();
            
            // Find common span sequence before error
            List<String> commonSequence = findCommonSpanSequence(errorTraces);
            
            if (!commonSequence.isEmpty()) {
                ErrorPattern pattern = new ErrorPattern();
                pattern.setErrorType(errorType);
                pattern.setFrequency(errorTraces.size());
                pattern.setCommonSequence(commonSequence);
                patterns.add(pattern);
            }
        }
        
        return patterns;
    }
    
    private List<String> findCommonSpanSequence(List<Trace> traces) {
        if (traces.isEmpty()) return Collections.emptyList();
        
        // Simplified: return span names from first trace
        return traces.get(0).getSpans().stream()
            .map(Span::getOperationName)
            .collect(Collectors.toList());
    }
}
```

## Best Practices

### Instrumentation Guidelines
- **Instrument Key Operations**: Focus on database calls, external API calls, and business logic
- **Add Relevant Tags**: Include operation names, parameters, and results
- **Handle Errors**: Always record exceptions and error conditions
- **Avoid Overhead**: Use sampling and efficient instrumentation

### Context Propagation
- **Use Standard Headers**: Follow W3C Trace Context or similar standards
- **Propagate Baggage**: Include business context across service boundaries
- **Handle Missing Context**: Generate new trace IDs when context is missing
- **Validate Context**: Ensure trace IDs are valid and properly formatted

### Sampling Decisions
- **Start Conservative**: Begin with lower sampling rates and increase as needed
- **Sample Errors**: Always sample traces with errors
- **Consider Performance**: Balance observability with system performance
- **Use Adaptive Sampling**: Adjust rates based on system load and requirements

### Analysis and Alerting
- **Monitor Key Metrics**: Track P95 latency, error rates, and throughput
- **Set Up Alerts**: Alert on performance degradation and error spikes
- **Analyze Patterns**: Look for common failure modes and bottlenecks
- **Continuous Improvement**: Use insights to optimize system performance

## Real-World Tracing Examples

### E-commerce Order Flow
```java
@Service
public class OrderTracingService {
    
    @Autowired
    private Tracer tracer;
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private ShippingService shippingService;
    
    public Order processOrder(CreateOrderRequest request) {
        Span orderSpan = tracer.buildSpan("processOrder")
            .withTag("userId", request.getUserId().toString())
            .withTag("itemCount", request.getItems().size())
            .start();
        
        Order order = null;
        
        try (Scope scope = tracer.scopeManager().activate(orderSpan)) {
            
            // Step 1: Validate user
            Span userSpan = tracer.buildSpan("validateUser").start();
            try (Scope userScope = tracer.scopeManager().activate(userSpan)) {
                User user = userService.getUserById(request.getUserId());
                userSpan.setTag("userExists", user != null);
            } finally {
                userSpan.finish();
            }
            
            // Step 2: Reserve inventory
            Span inventorySpan = tracer.buildSpan("reserveInventory").start();
            try (Scope inventoryScope = tracer.scopeManager().activate(inventorySpan)) {
                inventoryService.reserveInventory(request.getItems());
                inventorySpan.setTag("reservationSuccess", true);
            } finally {
                inventorySpan.finish();
            }
            
            // Step 3: Process payment
            Span paymentSpan = tracer.buildSpan("processPayment").start();
            try (Scope paymentScope = tracer.scopeManager().activate(paymentSpan)) {
                PaymentResult payment = paymentService.processPayment(request.getTotalAmount());
                paymentSpan.setTag("paymentSuccess", payment.isSuccessful());
                paymentSpan.setTag("paymentId", payment.getId());
            } finally {
                paymentSpan.finish();
            }
            
            // Step 4: Create order
            Span createSpan = tracer.buildSpan("createOrder").start();
            try (Scope createScope = tracer.scopeManager().activate(createSpan)) {
                order = new Order(request.getUserId(), request.getItems(), Instant.now());
                order = orderRepository.save(order);
                createSpan.setTag("orderId", order.getId().toString());
            } finally {
                createSpan.finish();
            }
            
            // Step 5: Schedule shipping (async)
            Span shippingSpan = tracer.buildSpan("scheduleShipping").start();
            try (Scope shippingScope = tracer.scopeManager().activate(shippingSpan)) {
                shippingService.scheduleShipping(order.getId());
                shippingSpan.setTag("shippingScheduled", true);
            } finally {
                shippingSpan.finish();
            }
            
            orderSpan.setTag("orderId", order.getId().toString());
            orderSpan.setTag("processingSuccess", true);
            
            return order;
            
        } catch (Exception e) {
            orderSpan.setTag("error", true);
            orderSpan.log(Map.of("event", "order_processing_failed", "error", e.getMessage()));
            throw e;
        } finally {
            orderSpan.finish();
        }
    }
}
```

### Microservices Communication
```java
@Service
public class ServiceCommunicationTracer {
    
    @Autowired
    private Tracer tracer;
    
    @Autowired
    private RestTemplate restTemplate;
    
    public <T> T callService(String serviceName, String endpoint, Class<T> responseType) {
        Span span = tracer.buildSpan("callService")
            .withTag("service", serviceName)
            .withTag("endpoint", endpoint)
            .withTag("span.kind", "client")
            .start();
        
        try (Scope scope = tracer.scopeManager().activate(span)) {
            
            // Inject trace context into headers
            HttpHeaders headers = new HttpHeaders();
            tracer.inject(span.context(), Format.Builtin.HTTP_HEADERS, 
                         new HttpHeadersCarrier(headers));
            
            // Create request
            HttpEntity<?> requestEntity = new HttpEntity<>(headers);
            
            span.log("Sending request to " + serviceName);
            long startTime = System.currentTimeMillis();
            
            // Make the call
            ResponseEntity<T> response = restTemplate.exchange(
                "http://" + serviceName + endpoint,
                HttpMethod.GET,
                requestEntity,
                responseType
            );
            
            long duration = System.currentTimeMillis() - startTime;
            span.setTag("http.status_code", response.getStatusCodeValue());
            span.setTag("duration_ms", duration);
            
            span.log("Received response from " + serviceName);
            
            return response.getBody();
            
        } catch (Exception e) {
            span.setTag("error", true);
            span.log(Map.of("event", "service_call_failed", "error", e.getMessage()));
            throw e;
        } finally {
            span.finish();
        }
    }
    
    // Carrier for injecting headers
    public static class HttpHeadersCarrier implements TextMap {
        
        private final HttpHeaders headers;
        
        public HttpHeadersCarrier(HttpHeaders headers) {
            this.headers = headers;
        }
        
        @Override
        public Iterator<Map.Entry<String, String>> iterator() {
            throw new UnsupportedOperationException("iterator not supported");
        }
        
        @Override
        public void put(String key, String value) {
            headers.add(key, value);
        }
    }
}
```

## Conclusion

Distributed tracing is essential for understanding and debugging complex distributed systems. By implementing proper instrumentation, context propagation, and analysis tools, teams can gain deep insights into system behavior, quickly identify performance bottlenecks, and resolve issues across service boundaries.

**Key Takeaways:**
- **Instrumentation**: Instrument key operations and propagate context
- **Sampling**: Use appropriate sampling strategies to balance observability and performance
- **Storage**: Choose scalable storage solutions for trace data
- **Analysis**: Implement automated analysis for performance and error patterns
- **Standards**: Follow OpenTelemetry and other standards for interoperability

Effective distributed tracing enables better system observability, faster debugging, and improved reliability in distributed architectures.
