# Observability Best Practices

Observability enables understanding system behavior through logs, metrics, and traces. This guide covers best practices for implementing comprehensive observability in distributed systems, including instrumentation strategies, data collection patterns, and operational procedures.

## Observability Fundamentals

### The Three Pillars
- **Logs**: Structured records of discrete events
- **Metrics**: Numerical measurements of system state over time
- **Traces**: End-to-end request flows through distributed systems

### Observability Goals
- **Understand System Behavior**: Know what your system is doing at any time
- **Rapid Issue Diagnosis**: Quickly identify and resolve problems
- **Performance Optimization**: Identify bottlenecks and optimization opportunities
- **Capacity Planning**: Make data-driven scaling decisions
- **Business Insights**: Understand user behavior and system usage patterns

## Instrumentation Strategies

### 1. Structured Logging

#### Log Format Standards
```java
@Configuration
public class StructuredLoggingConfig {
    
    @Bean
    public LogstashTcpSocketAppender logstashAppender() {
        LogstashTcpSocketAppender appender = new LogstashTcpSocketAppender();
        appender.setHost("logstash");
        appender.setPort(5044);
        
        // Configure encoder for structured output
        LogstashEncoder encoder = new LogstashEncoder();
        encoder.setCustomFields("{\"service\":\"order-service\",\"version\":\"1.0.0\"}");
        appender.setEncoder(encoder);
        
        appender.start();
        return appender;
    }
}

@Service
public class StructuredLogger {
    
    private static final Logger logger = LoggerFactory.getLogger(StructuredLogger.class);
    
    public void logBusinessEvent(String eventType, String entityId, Map<String, Object> context) {
        logger.info("Business event occurred", 
            StructuredArguments.keyValue("eventType", eventType),
            StructuredArguments.keyValue("entityId", entityId),
            StructuredArguments.keyValue("context", context),
            StructuredArguments.keyValue("timestamp", Instant.now()),
            StructuredArguments.keyValue("correlationId", getCorrelationId()));
    }
    
    public void logPerformanceMetric(String operation, long durationMs, boolean success) {
        logger.info("Performance metric recorded",
            StructuredArguments.keyValue("operation", operation),
            StructuredArguments.keyValue("durationMs", durationMs),
            StructuredArguments.keyValue("success", success),
            StructuredArguments.keyValue("timestamp", Instant.now()));
    }
    
    public void logError(String operation, Exception error, Map<String, Object> context) {
        logger.error("Operation failed",
            StructuredArguments.keyValue("operation", operation),
            StructuredArguments.keyValue("error", error.getMessage()),
            StructuredArguments.keyValue("errorType", error.getClass().getSimpleName()),
            StructuredArguments.keyValue("context", context),
            StructuredArguments.keyValue("stackTrace", getStackTrace(error)),
            error);
    }
    
    private String getCorrelationId() {
        // Extract from MDC or thread-local context
        return MDC.get("correlationId");
    }
    
    private String getStackTrace(Exception error) {
        StringWriter sw = new StringWriter();
        error.printStackTrace(new PrintWriter(sw));
        return sw.toString();
    }
}
```

#### Log Levels and Context
```java
@Service
public class ContextAwareLogger {
    
    private static final Logger logger = LoggerFactory.getLogger(ContextAwareLogger.class);
    
    public void logWithContext(String level, String message, Object... args) {
        // Build context map
        Map<String, Object> context = new HashMap<>();
        context.put("timestamp", Instant.now());
        context.put("correlationId", MDC.get("correlationId"));
        context.put("userId", MDC.get("userId"));
        context.put("requestId", MDC.get("requestId"));
        context.put("service", getServiceName());
        context.put("version", getServiceVersion());
        context.put("environment", getEnvironment());
        
        // Add context to MDC for structured logging
        context.forEach((key, value) -> {
            if (value != null) {
                MDC.put(key, value.toString());
            }
        });
        
        try {
            switch (level.toUpperCase()) {
                case "TRACE":
                    logger.trace(message, args);
                    break;
                case "DEBUG":
                    logger.debug(message, args);
                    break;
                case "INFO":
                    logger.info(message, args);
                    break;
                case "WARN":
                    logger.warn(message, args);
                    break;
                case "ERROR":
                    logger.error(message, args);
                    break;
                default:
                    logger.info(message, args);
            }
        } finally {
            // Clean up MDC
            context.keySet().forEach(MDC::remove);
        }
    }
    
    private String getServiceName() {
        return System.getProperty("spring.application.name", "unknown-service");
    }
    
    private String getServiceVersion() {
        return System.getProperty("app.version", "unknown");
    }
    
    private String getEnvironment() {
        return System.getProperty("spring.profiles.active", "default");
    }
}
```

### 2. Metrics Instrumentation

#### Key Metrics to Track
```java
@Service
public class ComprehensiveMetricsService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    // Infrastructure Metrics
    private final Gauge systemCpuUsage = Gauge.builder("system_cpu_usage_percent")
        .description("Current CPU usage percentage")
        .register(meterRegistry);
        
    private final Gauge systemMemoryUsage = Gauge.builder("system_memory_usage_bytes")
        .description("Current memory usage in bytes")
        .register(meterRegistry);
        
    private final Gauge jvmHeapUsage = Gauge.builder("jvm_heap_usage_bytes")
        .description("JVM heap memory usage")
        .register(meterRegistry);
    
    // Application Metrics
    private final Counter httpRequests = Counter.builder("http_requests_total")
        .description("Total HTTP requests")
        .tag("method", "")
        .tag("endpoint", "")
        .tag("status", "")
        .register(meterRegistry);
        
    private final Timer httpRequestDuration = Timer.builder("http_request_duration")
        .description("HTTP request duration")
        .tag("method", "")
        .tag("endpoint", "")
        .register(meterRegistry);
        
    private final Gauge activeConnections = Gauge.builder("active_connections")
        .description("Number of active connections")
        .register(meterRegistry);
    
    // Business Metrics
    private final Counter businessTransactions = Counter.builder("business_transactions_total")
        .description("Total business transactions")
        .tag("type", "")
        .register(meterRegistry);
        
    private final Histogram transactionValue = Histogram.builder("transaction_value")
        .description("Transaction value distribution")
        .register(meterRegistry);
    
    // Error Metrics
    private final Counter applicationErrors = Counter.builder("application_errors_total")
        .description("Total application errors")
        .tag("type", "")
        .register(meterRegistry);
        
    // Performance Metrics
    private final Timer databaseQueryTime = Timer.builder("database_query_duration")
        .description("Database query duration")
        .tag("table", "")
        .tag("operation", "")
        .register(meterRegistry);
        
    private final Timer externalApiCallTime = Timer.builder("external_api_duration")
        .description("External API call duration")
        .tag("service", "")
        .tag("endpoint", "")
        .register(meterRegistry);
    
    // Custom Business Metrics
    private final Counter userRegistrations = Counter.builder("user_registrations_total")
        .description("Total user registrations")
        .tag("source", "")
        .register(meterRegistry);
        
    private final Counter orderCompletions = Counter.builder("order_completions_total")
        .description("Total completed orders")
        .tag("payment_method", "")
        .register(meterRegistry);
    
    public void recordHttpRequest(String method, String endpoint, int status, long durationMs) {
        httpRequests.withTag("method", method)
                   .withTag("endpoint", endpoint)
                   .withTag("status", String.valueOf(status))
                   .increment();
                   
        httpRequestDuration.withTag("method", method)
                          .withTag("endpoint", endpoint)
                          .record(Duration.ofMillis(durationMs));
    }
    
    public void recordBusinessTransaction(String type, double value) {
        businessTransactions.withTag("type", type).increment();
        transactionValue.observe(value);
    }
    
    public void recordDatabaseQuery(String table, String operation, long durationMs) {
        databaseQueryTime.withTag("table", table)
                        .withTag("operation", operation)
                        .record(Duration.ofMillis(durationMs));
    }
    
    public void recordExternalApiCall(String service, String endpoint, long durationMs, boolean success) {
        externalApiCallTime.withTag("service", service)
                          .withTag("endpoint", endpoint)
                          .record(Duration.ofMillis(durationMs));
                          
        if (!success) {
            applicationErrors.withTag("type", "external_api_failure").increment();
        }
    }
    
    public void recordUserRegistration(String source) {
        userRegistrations.withTag("source", source).increment();
    }
    
    public void recordOrderCompletion(String paymentMethod, double orderValue) {
        orderCompletions.withTag("payment_method", paymentMethod).increment();
        transactionValue.observe(orderValue);
    }
    
    public void recordApplicationError(String errorType, Exception e) {
        applicationErrors.withTag("type", errorType).increment();
        
        // Log structured error
        logger.error("Application error occurred",
            StructuredArguments.keyValue("errorType", errorType),
            StructuredArguments.keyValue("errorMessage", e.getMessage()),
            StructuredArguments.keyValue("stackTrace", getStackTrace(e)),
            e);
    }
    
    private String getStackTrace(Exception e) {
        StringWriter sw = new StringWriter();
        e.printStackTrace(new PrintWriter(sw));
        return sw.toString();
    }
}
```

### 3. Distributed Tracing

#### Comprehensive Tracing Strategy
```java
@Configuration
public class ComprehensiveTracingConfig {
    
    @Bean
    public Tracer tracer() {
        // Configure OpenTelemetry tracer with sampling
        return OpenTelemetrySdk.builder()
            .setTracerProvider(
                SdkTracerProvider.builder()
                    .addSpanProcessor(
                        BatchSpanProcessor.builder(
                            JaegerGrpcSpanExporter.builder()
                                .setEndpoint("http://jaeger:4317")
                                .build()
                        ).build()
                    )
                    .setSampler(new AdaptiveSampler()) // Custom sampling strategy
                    .build()
            )
            .build()
            .getTracer("comprehensive-service");
    }
    
    @Bean
    public GlobalTracer globalTracer(Tracer tracer) {
        GlobalTracer.register(tracer);
        return GlobalTracer.get();
    }
}

@Service
public class ComprehensiveTracingService {
    
    @Autowired
    private Tracer tracer;
    
    public <T> T executeWithTracing(String operationName, Supplier<T> operation) {
        Span span = tracer.spanBuilder(operationName)
            .setAttribute("service.name", getServiceName())
            .setAttribute("service.version", getServiceVersion())
            .setAttribute("operation.type", "business")
            .startSpan();
            
        // Add resource attributes
        span.setAttribute("resource.cpu", getCpuUsage());
        span.setAttribute("resource.memory", getMemoryUsage());
        
        try (Scope scope = tracer.scopeManager().activate(span)) {
            // Add business context
            addBusinessContext(span);
            
            T result = operation.get();
            
            // Add result information
            span.setAttribute("operation.success", true);
            if (result != null) {
                span.setAttribute("result.type", result.getClass().getSimpleName());
            }
            
            return result;
            
        } catch (Exception e) {
            span.setAttribute("operation.success", false);
            span.setAttribute("error.type", e.getClass().getSimpleName());
            span.setAttribute("error.message", e.getMessage());
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
    
    public Span createChildSpan(String spanName, Span parentSpan) {
        return tracer.spanBuilder(spanName)
            .setParent(parentSpan.getContext())
            .setAttribute("span.type", "child")
            .startSpan();
    }
    
    public void addTagsToSpan(Span span, Map<String, String> tags) {
        tags.forEach(span::setAttribute);
    }
    
    public void addEventToSpan(Span span, String eventName, Map<String, Object> attributes) {
        span.addEvent(eventName, attributes);
    }
    
    private void addBusinessContext(Span span) {
        // Add business-relevant context
        span.setAttribute("business.tenant", getCurrentTenant());
        span.setAttribute("business.user", getCurrentUserId());
        span.setAttribute("business.session", getCurrentSessionId());
    }
    
    private String getServiceName() {
        return System.getProperty("spring.application.name", "unknown");
    }
    
    private String getServiceVersion() {
        return System.getProperty("app.version", "1.0.0");
    }
    
    private double getCpuUsage() {
        // Get current CPU usage
        return 0.0; // Implementation would read from MXBean
    }
    
    private long getMemoryUsage() {
        // Get current memory usage
        Runtime runtime = Runtime.getRuntime();
        return runtime.totalMemory() - runtime.freeMemory();
    }
    
    private String getCurrentTenant() {
        return MDC.get("tenantId");
    }
    
    private String getCurrentUserId() {
        return MDC.get("userId");
    }
    
    private String getCurrentSessionId() {
        return MDC.get("sessionId");
    }
}
```

## Data Collection and Storage

### 1. Centralized Logging Architecture

#### ELK Stack with Best Practices
```yaml
# docker-compose.yml for ELK stack
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
      - xpack.monitoring.enabled=false
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:7.10.0
    ports:
      - "5044:5044"
      - "9600:9600"
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:7.10.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  elasticsearch-data:
```

#### Logstash Configuration with Filtering
```conf
input {
  beats {
    port => 5044
    ssl => false
  }
  
  tcp {
    port => 5045
    codec => json
  }
}

filter {
  # Parse JSON logs
  json {
    source => "message"
    remove_field => ["message"]
  }
  
  # Parse timestamp
  date {
    match => ["timestamp", "ISO8601", "yyyy-MM-dd HH:mm:ss.SSS"]
    target => "@timestamp"
  }
  
  # Add geo information for IP addresses
  if [client_ip] {
    geoip {
      source => "client_ip"
      target => "geoip"
    }
  }
  
  # Parse user agent
  if [user_agent] {
    useragent {
      source => "user_agent"
      target => "user_agent"
    }
  }
  
  # Anonymize sensitive data
  mutate {
    # Remove or mask sensitive fields
    remove_field => ["password", "token", "secret"]
    
    # Mask email domains
    if [user_email] {
      gsub => [
        "user_email", "@.*$", "@***.***"
      ]
    }
    
    # Mask credit card numbers
    if [credit_card] {
      gsub => [
        "credit_card", "\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b", "****-****-****-****"
      ]
    }
  }
  
  # Add service metadata
  mutate {
    add_field => {
      "service.name" => "%{[fields][service]}"
      "service.version" => "%{[fields][version]}"
      "environment" => "%{[fields][environment]}"
    }
  }
  
  # Drop debug logs in production
  if [environment] == "production" and [level] == "DEBUG" {
    drop {}
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    document_type => "_doc"
    
    # Bulk indexing for performance
    flush_size => 1000
    idle_flush_time => 10
  }
  
  # Archive error logs
  if [level] == "ERROR" or [level] == "FATAL" {
    file {
      path => "/var/log/archive/%{+YYYY-MM-dd}/errors.log"
      codec => json
    }
  }
  
  # Send critical alerts
  if [level] == "FATAL" {
    email {
      to => "alerts@company.com"
      subject => "FATAL Error in %{service.name}"
      body => "A fatal error occurred: %{message}"
    }
  }
}
```

### 2. Metrics Storage and Aggregation

#### Prometheus with Long-term Storage
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'

# Remote write for long-term storage
remote_write:
  - url: "http://victoriametrics:8428/api/v1/write"
    remote_timeout: 30s
    write_relabel_configs:
      - source_labels: [__name__]
        regex: '.*'
        action: keep

# Remote read for querying historical data
remote_read:
  - url: "http://victoriametrics:8428/api/v1/read"
    remote_timeout: 30s

scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s
    scrape_timeout: 5s
    
    # Service discovery
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - production
    
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: application
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
        
    # Metric relabeling for efficiency
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'jvm_.*|http_.*|business_.*'
        action: keep
      - source_labels: [__name__]
        regex: 'debug_.*'
        action: drop
```

#### Metrics Aggregation Strategy
```java
@Service
public class MetricsAggregationService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    // Pre-aggregated metrics for common queries
    private final Counter hourlyRequests = Counter.builder("requests_per_hour")
        .description("Requests aggregated per hour")
        .register(meterRegistry);
        
    private final Histogram responseTimePercentiles = Histogram.builder("response_time_percentiles")
        .description("Response time percentiles")
        .register(meterRegistry);
    
    @Scheduled(fixedRate = 3600000) // Every hour
    public void aggregateHourlyMetrics() {
        // Aggregate metrics from the past hour
        Instant now = Instant.now();
        Instant oneHourAgo = now.minus(Duration.ofHours(1));
        
        // Aggregate request counts by endpoint
        Map<String, Long> requestsByEndpoint = getRequestsByEndpoint(oneHourAgo, now);
        requestsByEndpoint.forEach((endpoint, count) -> 
            Counter.builder("endpoint_requests_hourly")
                .tag("endpoint", endpoint)
                .register(meterRegistry)
                .increment(count));
        
        // Update hourly counter
        long totalRequests = requestsByEndpoint.values().stream().mapToLong(Long::longValue).sum();
        hourlyRequests.increment(totalRequests);
        
        // Calculate and store percentiles
        List<Double> responseTimes = getResponseTimes(oneHourAgo, now);
        if (!responseTimes.isEmpty()) {
            Collections.sort(responseTimes);
            
            double p50 = calculatePercentile(responseTimes, 50);
            double p95 = calculatePercentile(responseTimes, 95);
            double p99 = calculatePercentile(responseTimes, 99);
            
            // Store as gauges (updated periodically)
            Gauge.builder("response_time_p50", () -> p50)
                .register(meterRegistry);
            Gauge.builder("response_time_p95", () -> p95)
                .register(meterRegistry);
            Gauge.builder("response_time_p99", () -> p99)
                .register(meterRegistry);
        }
    }
    
    @Scheduled(fixedRate = 300000) // Every 5 minutes
    public void aggregateRecentMetrics() {
        // Aggregate metrics from the past 5 minutes for real-time monitoring
        Instant now = Instant.now();
        Instant fiveMinutesAgo = now.minus(Duration.ofMinutes(5));
        
        // Error rate aggregation
        long recentErrors = getErrorCount(fiveMinutesAgo, now);
        long recentRequests = getRequestCount(fiveMinutesAgo, now);
        
        double errorRate = recentRequests > 0 ? (double) recentErrors / recentRequests : 0.0;
        
        Gauge.builder("error_rate_5m", () -> errorRate)
            .register(meterRegistry);
            
        // Throughput aggregation
        double requestsPerSecond = recentRequests / 300.0; // 5 minutes = 300 seconds
        
        Gauge.builder("throughput_rps", () -> requestsPerSecond)
            .register(meterRegistry);
    }
    
    private Map<String, Long> getRequestsByEndpoint(Instant start, Instant end) {
        // Query metrics storage for aggregated data
        // This would integrate with your metrics backend
        return new HashMap<>();
    }
    
    private List<Double> getResponseTimes(Instant start, Instant end) {
        // Query response time metrics
        return new ArrayList<>();
    }
    
    private long getErrorCount(Instant start, Instant end) {
        // Query error count metrics
        return 0;
    }
    
    private long getRequestCount(Instant start, Instant end) {
        // Query request count metrics
        return 0;
    }
    
    private double calculatePercentile(List<Double> values, double percentile) {
        if (values.isEmpty()) return 0.0;
        
        int index = (int) Math.ceil((percentile / 100.0) * values.size()) - 1;
        return values.get(Math.max(0, Math.min(index, values.size() - 1)));
    }
}
```

## Alerting and Incident Response

### 1. Alert Design Principles

#### Alert Quality Framework
```java
@Service
public class AlertQualityService {
    
    @Autowired
    private AlertRepository alertRepository;
    
    @Autowired
    private MetricsService metricsService;
    
    // Alert quality metrics
    private final Counter falsePositives = Counter.builder("alerts_false_positives_total")
        .description("Total false positive alerts")
        .register(meterRegistry);
        
    private final Counter truePositives = Counter.builder("alerts_true_positives_total")
        .description("Total true positive alerts")
        .register(meterRegistry);
        
    private final Histogram alertResponseTime = Histogram.builder("alert_response_time")
        .description("Time to respond to alerts")
        .register(meterRegistry);
    
    public AlertQualityScore evaluateAlertQuality(Alert alert) {
        AlertQualityScore score = new AlertQualityScore();
        
        // Check if alert was actionable
        score.setActionable(wasAlertActionable(alert));
        
        // Check if alert was timely
        score.setTimely(wasAlertTimely(alert));
        
        // Check if alert provided sufficient context
        score.setInformative(wasAlertInformative(alert));
        
        // Calculate overall quality score
        score.setOverallScore(calculateOverallScore(score));
        
        // Update quality metrics
        if (score.isActionable()) {
            truePositives.increment();
        } else {
            falsePositives.increment();
        }
        
        return score;
    }
    
    public void recordAlertResponse(Alert alert, Duration responseTime) {
        alertResponseTime.observe(responseTime.toMillis() / 1000.0);
        
        // Update alert with response time
        alert.setResponseTime(responseTime);
        alertRepository.save(alert);
    }
    
    private boolean wasAlertActionable(Alert alert) {
        // Check if alert led to actual issue resolution
        // This would analyze incident response data
        return true; // Simplified
    }
    
    private boolean wasAlertTimely(Alert alert) {
        // Check if alert fired before issue became critical
        Duration timeToCritical = getTimeToCritical(alert);
        return timeToCritical.compareTo(Duration.ofMinutes(5)) > 0;
    }
    
    private boolean wasAlertInformative(Alert alert) {
        // Check if alert provided sufficient diagnostic information
        return alert.getDescription() != null && 
               alert.getDescription().length() > 50 &&
               alert.getContext() != null;
    }
    
    private double calculateOverallScore(AlertQualityScore score) {
        double score = 0.0;
        if (score.isActionable()) score += 0.4;
        if (score.isTimely()) score += 0.3;
        if (score.isInformative()) score += 0.3;
        return score;
    }
    
    private Duration getTimeToCritical(Alert alert) {
        // Calculate time from alert to critical impact
        return Duration.ofMinutes(10); // Simplified
    }
    
    public static class AlertQualityScore {
        private boolean actionable;
        private boolean timely;
        private boolean informative;
        private double overallScore;
        
        // Getters and setters
    }
}
```

### 2. Runbook Automation

#### Automated Incident Response
```java
@Service
public class AutomatedIncidentResponse {
    
    @Autowired
    private AlertService alertService;
    
    @Autowired
    private SystemControlService systemControlService;
    
    @Autowired
    private NotificationService notificationService;
    
    private final Map<String, IncidentResponsePlan> responsePlans = new HashMap<>();
    
    @PostConstruct
    public void initializeResponsePlans() {
        // High CPU usage response
        responsePlans.put("high_cpu", new IncidentResponsePlan(
            Arrays.asList(
                new ScaleOutAction(),
                new RestartHighCpuInstancesAction(),
                new NotifyEngineeringAction("High CPU usage detected")
            ),
            Duration.ofMinutes(5) // Auto-resolve after 5 minutes
        ));
        
        // Memory leak response
        responsePlans.put("memory_leak", new IncidentResponsePlan(
            Arrays.asList(
                new RestartLeakingInstancesAction(),
                new EnableVerboseGcLoggingAction(),
                new NotifyEngineeringAction("Memory leak detected")
            ),
            null // Manual resolution required
        ));
        
        // Service down response
        responsePlans.put("service_down", new IncidentResponsePlan(
            Arrays.asList(
                new AttemptServiceRestartAction(),
                new SwitchToBackupAction(),
                new NotifyOnCallAction("Service is down")
            ),
            Duration.ofMinutes(2)
        ));
    }
    
    public void handleAlert(Alert alert) {
        IncidentResponsePlan plan = responsePlans.get(alert.getName());
        
        if (plan != null) {
            executeResponsePlan(plan, alert);
        } else {
            // No automated response - notify on-call
            notificationService.notifyOnCall(alert);
        }
    }
    
    private void executeResponsePlan(IncidentResponsePlan plan, Alert alert) {
        logger.info("Executing automated response plan for alert: {}", alert.getName());
        
        for (Action action : plan.getActions()) {
            try {
                action.execute(alert);
                logger.info("Executed action: {}", action.getDescription());
            } catch (Exception e) {
                logger.error("Failed to execute action {}: {}", action.getDescription(), e.getMessage());
                
                // If automated action fails, escalate to manual
                notificationService.notifyOnCall(alert, "Automated response failed: " + e.getMessage());
                return;
            }
        }
        
        // Schedule auto-resolution if configured
        if (plan.getAutoResolveAfter() != null) {
            scheduleAutoResolution(alert, plan.getAutoResolveAfter());
        }
    }
    
    private void scheduleAutoResolution(Alert alert, Duration delay) {
        Executors.newSingleThreadScheduledExecutor().schedule(() -> {
            // Check if alert is still active
            if (alertService.isAlertActive(alert)) {
                // Verify system is healthy
                if (systemControlService.isSystemHealthy()) {
                    alertService.resolveAlert(alert, "Auto-resolved after automated response");
                    logger.info("Auto-resolved alert: {}", alert.getName());
                } else {
                    // System still unhealthy - escalate
                    notificationService.notifyOnCall(alert, "System still unhealthy after automated response");
                }
            }
        }, delay.toMillis(), TimeUnit.MILLISECONDS);
    }
    
    // Action implementations
    public interface Action {
        void execute(Alert alert) throws Exception;
        String getDescription();
    }
    
    public static class ScaleOutAction implements Action {
        @Override
        public void execute(Alert alert) {
            // Implement scaling logic
            systemControlService.scaleOutService(alert.getService());
        }
        
        @Override
        public String getDescription() {
            return "Scale out service instances";
        }
    }
    
    public static class RestartHighCpuInstancesAction implements Action {
        @Override
        public void execute(Alert alert) {
            // Identify and restart high CPU instances
            List<String> highCpuInstances = systemControlService.getHighCpuInstances();
            for (String instance : highCpuInstances) {
                systemControlService.restartInstance(instance);
            }
        }
        
        @Override
        public String getDescription() {
            return "Restart high CPU instances";
        }
    }
    
    public static class NotifyEngineeringAction implements Action {
        private final String message;
        
        public NotifyEngineeringAction(String message) {
            this.message = message;
        }
        
        @Override
        public void execute(Alert alert) {
            notificationService.notifyEngineering(message, alert);
        }
        
        @Override
        public String getDescription() {
            return "Notify engineering team";
        }
    }
    
    public static class IncidentResponsePlan {
        private final List<Action> actions;
        private final Duration autoResolveAfter;
        
        public IncidentResponsePlan(List<Action> actions, Duration autoResolveAfter) {
            this.actions = actions;
            this.autoResolveAfter = autoResolveAfter;
        }
        
        public List<Action> getActions() { return actions; }
        public Duration getAutoResolveAfter() { return autoResolveAfter; }
    }
}
```

## Operational Excellence

### 1. Observability Maturity Model

#### Level 1: Basic Monitoring
- Basic infrastructure metrics
- Reactive alerting
- Manual log analysis
- Limited visibility

#### Level 2: Reactive Observability
- Comprehensive metrics collection
- Structured logging
- Alert-driven incident response
- Basic tracing

#### Level 3: Proactive Observability
- Automated anomaly detection
- Predictive alerting
- Service mesh integration
- Advanced analytics

#### Level 4: Predictive Observability
- Machine learning-based anomaly detection
- Automated incident response
- Chaos engineering integration
- Business impact correlation

#### Level 5: Perfect Observability
- Complete system understanding
- Zero-touch operations
- Predictive maintenance
- Business outcome optimization

### 2. Continuous Improvement

#### Observability Health Checks
```java
@Service
public class ObservabilityHealthCheck {
    
    @Autowired
    private MetricsService metricsService;
    
    @Autowired
    private LoggingService loggingService;
    
    @Autowired
    private TracingService tracingService;
    
    @Scheduled(fixedRate = 3600000) // Hourly
    public void performObservabilityHealthCheck() {
        ObservabilityHealthReport report = new ObservabilityHealthReport();
        
        // Check metrics coverage
        report.setMetricsCoverage(assessMetricsCoverage());
        
        // Check logging quality
        report.setLoggingQuality(assessLoggingQuality());
        
        // Check tracing coverage
        report.setTracingCoverage(assessTracingCoverage());
        
        // Check alerting effectiveness
        report.setAlertingEffectiveness(assessAlertingEffectiveness());
        
        // Calculate overall health score
        report.setOverallHealthScore(calculateOverallHealthScore(report));
        
        // Generate recommendations
        report.setRecommendations(generateRecommendations(report));
        
        // Store report
        saveHealthReport(report);
        
        // Alert if health is poor
        if (report.getOverallHealthScore() < 0.7) {
            alertService.sendAlert("Observability Health Degraded", 
                                 "Observability health score: " + report.getOverallHealthScore());
        }
    }
    
    private double assessMetricsCoverage() {
        // Check what percentage of services have metrics
        int servicesWithMetrics = metricsService.getServicesWithMetrics();
        int totalServices = serviceRegistry.getTotalServices();
        
        return totalServices > 0 ? (double) servicesWithMetrics / totalServices : 0.0;
    }
    
    private double assessLoggingQuality() {
        // Check log structure, correlation IDs, etc.
        double structuredLogsRatio = loggingService.getStructuredLogsRatio();
        double correlationIdCoverage = loggingService.getCorrelationIdCoverage();
        
        return (structuredLogsRatio + correlationIdCoverage) / 2.0;
    }
    
    private double assessTracingCoverage() {
        // Check tracing instrumentation coverage
        double tracedServicesRatio = tracingService.getTracedServicesRatio();
        double traceSamplingRate = tracingService.getEffectiveSamplingRate();
        
        return (tracedServicesRatio + traceSamplingRate) / 2.0;
    }
    
    private double assessAlertingEffectiveness() {
        // Check alert quality metrics
        double truePositiveRate = alertingService.getTruePositiveRate();
        double averageResponseTime = alertingService.getAverageResponseTime();
        
        // Normalize response time (lower is better)
        double responseTimeScore = Math.max(0, 1.0 - (averageResponseTime / 3600.0)); // 1 hour baseline
        
        return (truePositiveRate + responseTimeScore) / 2.0;
    }
    
    private double calculateOverallHealthScore(ObservabilityHealthReport report) {
        return (report.getMetricsCoverage() + 
                report.getLoggingQuality() + 
                report.getTracingCoverage() + 
                report.getAlertingEffectiveness()) / 4.0;
    }
    
    private List<String> generateRecommendations(ObservabilityHealthReport report) {
        List<String> recommendations = new ArrayList<>();
        
        if (report.getMetricsCoverage() < 0.8) {
            recommendations.add("Improve metrics coverage - add metrics to services lacking instrumentation");
        }
        
        if (report.getLoggingQuality() < 0.7) {
            recommendations.add("Improve logging quality - implement structured logging and correlation IDs");
        }
        
        if (report.getTracingCoverage() < 0.6) {
            recommendations.add("Increase tracing coverage - instrument more services and operations");
        }
        
        if (report.getAlertingEffectiveness() < 0.8) {
            recommendations.add("Improve alerting effectiveness - reduce false positives and improve response times");
        }
        
        return recommendations;
    }
    
    public static class ObservabilityHealthReport {
        private double metricsCoverage;
        private double loggingQuality;
        private double tracingCoverage;
        private double alertingEffectiveness;
        private double overallHealthScore;
        private List<String> recommendations;
        
        // Getters and setters
    }
}
```

## Best Practices Summary

### 1. Instrumentation
- **Comprehensive Coverage**: Instrument all services and key operations
- **Consistent Naming**: Use standardized metric and log naming conventions
- **Context Propagation**: Ensure correlation IDs and trace context flow through requests
- **Performance Awareness**: Balance observability with application performance

### 2. Data Management
- **Structured Data**: Use structured logging and consistent metric tagging
- **Retention Policies**: Define appropriate data retention periods
- **Cost Optimization**: Sample and aggregate data to control storage costs
- **Data Quality**: Validate and clean observability data

### 3. Alerting
- **Actionable Alerts**: Only alert on issues requiring human intervention
- **Progressive Escalation**: Start with warnings, escalate based on severity
- **Context-Rich**: Include sufficient information for quick diagnosis
- **Automation**: Automate responses where possible

### 4. Operations
- **Monitoring as Code**: Define monitoring configuration in code
- **Continuous Deployment**: Deploy monitoring changes with application changes
- **Team Ownership**: Assign responsibility for monitoring quality
- **Regular Reviews**: Regularly assess and improve observability practices

### 5. Culture
- **Observability-First**: Design systems with observability in mind
- **Blame-Free**: Use observability for learning, not blame assignment
- **Data-Driven**: Base decisions on observability data
- **Continuous Learning**: Regularly improve observability practices

## Conclusion

Observability best practices provide a framework for building systems that are understandable, debuggable, and maintainable. By implementing comprehensive instrumentation, effective data collection, and proactive monitoring, teams can achieve high reliability and rapid issue resolution.

**Key Takeaways:**
- **Instrumentation First**: Build observability into systems from the start
- **Quality Over Quantity**: Focus on high-quality, actionable observability data
- **Automation**: Automate monitoring, alerting, and incident response
- **Continuous Improvement**: Regularly assess and enhance observability practices
- **Culture Matters**: Foster a culture of observability and data-driven decision making

Effective observability enables teams to understand system behavior, quickly diagnose issues, and continuously improve system reliability and performance.
