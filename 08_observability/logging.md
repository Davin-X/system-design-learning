# Logging

Logging is the process of recording events, messages, and data during application execution to provide visibility into system behavior. Effective logging enables debugging, monitoring, auditing, and troubleshooting of distributed systems. This guide covers logging patterns, frameworks, and best practices for building observable applications.

## What is Logging?

Logging involves capturing and storing information about application execution, errors, and significant events. In distributed systems, logging becomes critical for understanding system behavior across multiple services and components.

### Logging Levels
- **TRACE**: Finest-grained information for debugging
- **DEBUG**: Detailed information for debugging
- **INFO**: General information about application operation
- **WARN**: Potentially harmful situations
- **ERROR**: Error conditions that might still allow the application to continue
- **FATAL**: Severe errors that cause premature application termination

### Log Components
- **Timestamp**: When the log entry was created
- **Level**: Severity of the log message
- **Logger Name**: Source of the log message
- **Message**: The actual log content
- **Context**: Additional metadata (thread, correlation ID, etc.)

## Logging Frameworks

### SLF4J with Logback

#### Configuration
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    
    <!-- Console appender for development -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- File appender for production -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- Async appender for performance -->
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
        <discardingThreshold>20</discardingThreshold>
        <queueSize>512</queueSize>
        <appender-ref ref="FILE" />
    </appender>
    
    <!-- Root logger -->
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="ASYNC" />
    </root>
    
    <!-- Package-specific loggers -->
    <logger name="com.example.service" level="DEBUG" />
    <logger name="org.springframework.security" level="WARN" />
    
</configuration>
```

#### Usage in Code
```java
@Service
public class OrderService {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderService.class);
    
    @Autowired
    private OrderRepository orderRepository;
    
    public Order createOrder(CreateOrderRequest request) {
        logger.info("Creating order for user {} with {} items", 
                   request.getUserId(), request.getItems().size());
        
        try {
            Order order = new Order(request.getUserId(), request.getItems(), Instant.now());
            Order savedOrder = orderRepository.save(order);
            
            logger.info("Order created successfully with ID {}", savedOrder.getId());
            return savedOrder;
            
        } catch (Exception e) {
            logger.error("Failed to create order for user {}", request.getUserId(), e);
            throw e;
        }
    }
    
    public Order getOrder(Long orderId) {
        logger.debug("Retrieving order with ID {}", orderId);
        
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        logger.debug("Order retrieved: {}", order);
        return order;
    }
}
```

### Structured Logging with JSON

#### JSON Log Format
```java
@Configuration
public class JsonLoggingConfig {
    
    @Bean
    public LoggingEventCompositeJsonEncoder jsonEncoder() {
        LoggingEventCompositeJsonEncoder encoder = new LoggingEventCompositeJsonEncoder();
        
        // Add custom fields
        encoder.addInclude("timestamp", null, null);
        encoder.addInclude("level", null, null);
        encoder.addInclude("logger", null, null);
        encoder.addInclude("message", null, null);
        encoder.addInclude("exception", null, null);
        encoder.addInclude("thread", null, null);
        
        // Add MDC fields
        encoder.addInclude("correlationId", null, "correlationId");
        encoder.addInclude("userId", null, "userId");
        encoder.addInclude("requestId", null, "requestId");
        
        return encoder;
    }
}

@Service
public class StructuredLogger {
    
    private static final Logger logger = LoggerFactory.getLogger(StructuredLogger.class);
    
    public void logUserAction(String userId, String action, Map<String, Object> details) {
        MDC.put("userId", userId);
        
        logger.info("User action performed", 
            StructuredArguments.keyValue("action", action),
            StructuredArguments.keyValue("details", details));
        
        MDC.remove("userId");
    }
    
    public void logBusinessEvent(String eventType, String entityId, Object data) {
        logger.info("Business event occurred",
            StructuredArguments.keyValue("eventType", eventType),
            StructuredArguments.keyValue("entityId", entityId),
            StructuredArguments.keyValue("data", data));
    }
    
    public void logPerformanceMetric(String operation, long durationMs, boolean success) {
        logger.info("Performance metric recorded",
            StructuredArguments.keyValue("operation", operation),
            StructuredArguments.keyValue("durationMs", durationMs),
            StructuredArguments.keyValue("success", success));
    }
}
```

## Correlation IDs and Request Tracing

### Correlation ID Filter
```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    
    private static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    private static final String CORRELATION_ID_KEY = "correlationId";
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                  HttpServletResponse response, 
                                  FilterChain filterChain) throws ServletException, IOException {
        
        String correlationId = request.getHeader(CORRELATION_ID_HEADER);
        
        if (correlationId == null || correlationId.isEmpty()) {
            correlationId = UUID.randomUUID().toString();
        }
        
        // Set in MDC for logging
        MDC.put(CORRELATION_ID_KEY, correlationId);
        
        // Add to response header
        response.setHeader(CORRELATION_ID_HEADER, correlationId);
        
        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(CORRELATION_ID_KEY);
        }
    }
}
```

### Request Context
```java
@Component
public class RequestContext {
    
    private static final ThreadLocal<RequestContextData> context = new ThreadLocal<>();
    
    public static void setContext(String correlationId, String userId, String requestId) {
        RequestContextData data = new RequestContextData(correlationId, userId, requestId);
        context.set(data);
    }
    
    public static RequestContextData getContext() {
        return context.get();
    }
    
    public static void clear() {
        context.remove();
    }
    
    public static class RequestContextData {
        private final String correlationId;
        private final String userId;
        private final String requestId;
        
        public RequestContextData(String correlationId, String userId, String requestId) {
            this.correlationId = correlationId;
            this.userId = userId;
            this.requestId = requestId;
        }
        
        // Getters
        public String getCorrelationId() { return correlationId; }
        public String getUserId() { return userId; }
        public String getRequestId() { return requestId; }
    }
}

@Service
public class ContextAwareLogger {
    
    private static final Logger logger = LoggerFactory.getLogger(ContextAwareLogger.class);
    
    public void logWithContext(String message, Object... args) {
        RequestContext.RequestContextData context = RequestContext.getContext();
        
        if (context != null) {
            MDC.put("correlationId", context.getCorrelationId());
            MDC.put("userId", context.getUserId());
            MDC.put("requestId", context.getRequestId());
        }
        
        try {
            logger.info(message, args);
        } finally {
            MDC.clear();
        }
    }
}
```

## Log Aggregation and Storage

### ELK Stack (Elasticsearch, Logstash, Kibana)

#### Logstash Configuration
```conf
input {
  beats {
    port => 5044
  }
  
  file {
    path => "/var/log/application/*.log"
    start_position => "beginning"
  }
}

filter {
  # Parse JSON logs
  json {
    source => "message"
  }
  
  # Extract timestamp
  date {
    match => [ "timestamp", "yyyy-MM-dd HH:mm:ss.SSS" ]
    target => "@timestamp"
  }
  
  # Add geo information
  geoip {
    source => "client_ip"
    target => "geoip"
  }
  
  # Anonymize sensitive data
  mutate {
    gsub => [
      "message", "(password|token|key)\s*[:=]\s*\S+", '\1:=***'
    ]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "application-logs-%{+YYYY.MM.dd}"
    document_type => "log"
  }
  
  # Archive old logs
  if [log_level] == "ERROR" or [log_level] == "WARN" {
    file {
      path => "/var/log/archive/%{+YYYY-MM-dd}/errors.log"
    }
  }
}
```

#### Filebeat Configuration
```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/application/*.log
  fields:
    service: user-service
    environment: production
  fields_under_root: true

output.logstash:
  hosts: ["logstash:5044"]
```

### Centralized Logging Architecture
```java
@Configuration
public class CentralizedLoggingConfig {
    
    @Bean
    public LogstashTcpSocketAppender logstashAppender() {
        LogstashTcpSocketAppender appender = new LogstashTcpSocketAppender();
        appender.setHost("logstash-host");
        appender.setPort(5044);
        appender.setReconnectionDelayMillis(5000);
        
        // Add custom fields
        appender.addCustomField("service", "order-service");
        appender.addCustomField("version", getApplicationVersion());
        
        appender.start();
        return appender;
    }
    
    @Bean
    public LoggerContext loggerContext(LogstashTcpSocketAppender logstashAppender) {
        LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();
        Logger rootLogger = context.getLogger(Logger.ROOT_LOGGER_NAME);
        
        rootLogger.addAppender(logstashAppender);
        return context;
    }
    
    private String getApplicationVersion() {
        return System.getProperty("app.version", "unknown");
    }
}
```

## Log Levels and Filtering

### Dynamic Log Level Management
```java
@RestController
@RequestMapping("/admin/logging")
public class LoggingController {
    
    @Autowired
    private LoggerContext loggerContext;
    
    @GetMapping("/levels")
    public Map<String, String> getLogLevels() {
        Map<String, String> levels = new HashMap<>();
        
        loggerContext.getLoggerList().forEach(logger -> {
            levels.put(logger.getName(), logger.getLevel().toString());
        });
        
        return levels;
    }
    
    @PostMapping("/levels/{loggerName}")
    public ResponseEntity<String> setLogLevel(@PathVariable String loggerName, 
                                            @RequestParam String level) {
        
        Logger logger = loggerContext.getLogger(loggerName);
        Level newLevel = Level.toLevel(level.toUpperCase());
        
        logger.setLevel(newLevel);
        
        return ResponseEntity.ok("Log level for " + loggerName + " set to " + newLevel);
    }
    
    @DeleteMapping("/levels/{loggerName}")
    public ResponseEntity<String> resetLogLevel(@PathVariable String loggerName) {
        Logger logger = loggerContext.getLogger(loggerName);
        logger.setLevel(null); // Reset to inherit from parent
        
        return ResponseEntity.ok("Log level for " + loggerName + " reset");
    }
}
```

### Environment-Based Logging
```java
@Configuration
public class EnvironmentLoggingConfig {
    
    @Bean
    @Profile("development")
    public LoggerContext developmentLogging() {
        LoggerContext context = configureBaseLogging();
        
        // Development-specific settings
        Logger rootLogger = context.getLogger(Logger.ROOT_LOGGER_NAME);
        rootLogger.setLevel(Level.DEBUG);
        
        // Enable debug logging for application packages
        context.getLogger("com.example").setLevel(Level.DEBUG);
        
        return context;
    }
    
    @Bean
    @Profile("production")
    public LoggerContext productionLogging() {
        LoggerContext context = configureBaseLogging();
        
        // Production-specific settings
        Logger rootLogger = context.getLogger(Logger.ROOT_LOGGER_NAME);
        rootLogger.setLevel(Level.INFO);
        
        // Reduce noise from third-party libraries
        context.getLogger("org.springframework").setLevel(Level.WARN);
        context.getLogger("com.zaxxer.hikari").setLevel(Level.WARN);
        
        return context;
    }
    
    private LoggerContext configureBaseLogging() {
        LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();
        
        // Configure console and file appenders
        ConsoleAppender consoleAppender = createConsoleAppender(context);
        RollingFileAppender fileAppender = createFileAppender(context);
        
        Logger rootLogger = context.getLogger(Logger.ROOT_LOGGER_NAME);
        rootLogger.addAppender(consoleAppender);
        rootLogger.addAppender(fileAppender);
        
        return context;
    }
    
    private ConsoleAppender createConsoleAppender(LoggerContext context) {
        ConsoleAppender appender = new ConsoleAppender();
        appender.setContext(context);
        appender.setEncoder(createEncoder(context));
        appender.start();
        return appender;
    }
    
    private RollingFileAppender createFileAppender(LoggerContext context) {
        RollingFileAppender appender = new RollingFileAppender();
        appender.setContext(context);
        appender.setFile("logs/application.log");
        
        TimeBasedRollingPolicy rollingPolicy = new TimeBasedRollingPolicy();
        rollingPolicy.setFileNamePattern("logs/application.%d{yyyy-MM-dd}.%i.log");
        rollingPolicy.setMaxHistory(30);
        rollingPolicy.setParent(appender);
        rollingPolicy.start();
        
        appender.setRollingPolicy(rollingPolicy);
        appender.setEncoder(createEncoder(context));
        appender.start();
        
        return appender;
    }
    
    private Encoder createEncoder(LoggerContext context) {
        PatternLayoutEncoder encoder = new PatternLayoutEncoder();
        encoder.setContext(context);
        encoder.setPattern("%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n");
        encoder.start();
        return encoder;
    }
}
```

## Security and Compliance Logging

### Audit Logging
```java
@Service
public class AuditLogger {
    
    private static final Logger auditLogger = LoggerFactory.getLogger("AUDIT");
    
    @Async
    public void logUserLogin(String userId, String ipAddress, boolean success) {
        auditLogger.info("User login attempt - userId: {}, ip: {}, success: {}", 
                        userId, ipAddress, success);
    }
    
    @Async
    public void logDataAccess(String userId, String resource, String action, String result) {
        auditLogger.info("Data access - userId: {}, resource: {}, action: {}, result: {}", 
                        userId, resource, action, result);
    }
    
    @Async
    public void logSecurityEvent(String eventType, String userId, Map<String, Object> details) {
        auditLogger.warn("Security event - type: {}, userId: {}, details: {}", 
                        eventType, userId, details);
    }
    
    @Async
    public void logAdminAction(String adminId, String action, Map<String, Object> details) {
        auditLogger.info("Admin action - adminId: {}, action: {}, details: {}", 
                        adminId, action, details);
    }
}
```

### Sensitive Data Masking
```java
@Component
public class SensitiveDataMaskingFilter {
    
    private static final Pattern CREDIT_CARD_PATTERN = 
        Pattern.compile("\\b\\d{4}[ -]?\\d{4}[ -]?\\d{4}[ -]?\\d{4}\\b");
    
    private static final Pattern SSN_PATTERN = 
        Pattern.compile("\\b\\d{3}[ -]?\\d{2}[ -]?\\d{4}\\b");
    
    private static final Pattern EMAIL_PATTERN = 
        Pattern.compile("\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b");
    
    public String maskSensitiveData(String message) {
        if (message == null) {
            return null;
        }
        
        String masked = message;
        masked = CREDIT_CARD_PATTERN.matcher(masked).replaceAll("XXXX-XXXX-XXXX-XXXX");
        masked = SSN_PATTERN.matcher(masked).replaceAll("XXX-XX-XXXX");
        masked = EMAIL_PATTERN.matcher(masked).replaceAll("***@***.***");
        
        return masked;
    }
}

@Aspect
@Component
public class LoggingAspect {
    
    @Autowired
    private SensitiveDataMaskingFilter maskingFilter;
    
    @Around("@within(org.springframework.web.bind.annotation.RestController)")
    public Object logControllerMethods(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        
        logger.info("Entering {}.{} with arguments: {}", 
                   className, methodName, maskArguments(joinPoint.getArgs()));
        
        long startTime = System.currentTimeMillis();
        
        try {
            Object result = joinPoint.proceed();
            
            long executionTime = System.currentTimeMillis() - startTime;
            
            logger.info("Exiting {}.{} with result: {} (execution time: {}ms)", 
                       className, methodName, maskResult(result), executionTime);
            
            return result;
            
        } catch (Exception e) {
            logger.error("Exception in {}.{}: {}", className, methodName, e.getMessage(), e);
            throw e;
        }
    }
    
    private String maskArguments(Object[] args) {
        // Mask sensitive data in method arguments
        return maskingFilter.maskSensitiveData(Arrays.toString(args));
    }
    
    private String maskResult(Object result) {
        // Mask sensitive data in method results
        return maskingFilter.maskSensitiveData(result != null ? result.toString() : "null");
    }
}
```

## Performance Considerations

### Asynchronous Logging
```java
@Configuration
public class AsyncLoggingConfig {
    
    @Bean
    public AsyncAppender asyncAppender() {
        AsyncAppender asyncAppender = new AsyncAppender();
        asyncAppender.setName("ASYNC");
        
        // Set the appender to append to
        asyncAppender.addAppender(createFileAppender());
        
        // Configuration
        asyncAppender.setQueueSize(512);
        asyncAppender.setDiscardingThreshold(20); // Discard less important logs if queue is full
        asyncAppender.setIncludeCallerData(false); // Improve performance
        
        asyncAppender.start();
        return asyncAppender;
    }
    
    private FileAppender createFileAppender() {
        FileAppender fileAppender = new FileAppender();
        fileAppender.setName("FILE");
        fileAppender.setFile("logs/application.log");
        fileAppender.setAppend(true);
        
        PatternLayout layout = new PatternLayout();
        layout.setPattern("%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n");
        layout.activateOptions();
        
        fileAppender.setLayout(layout);
        fileAppender.activateOptions();
        
        return fileAppender;
    }
}
```

### Log Sampling
```java
@Service
public class LogSampler {
    
    private final Map<String, Sampler> samplers = new ConcurrentHashMap<>();
    
    public boolean shouldLog(String loggerName, Level level) {
        Sampler sampler = samplers.computeIfAbsent(loggerName, this::createSampler);
        return sampler.shouldSample();
    }
    
    private Sampler createSampler(String loggerName) {
        // Sample DEBUG logs at 10%, INFO at 50%, etc.
        if (loggerName.contains("debug")) {
            return new PercentageSampler(10);
        } else if (loggerName.contains("performance")) {
            return new PercentageSampler(25);
        } else {
            return new AlwaysSampler(); // Don't sample important logs
        }
    }
    
    public interface Sampler {
        boolean shouldSample();
    }
    
    public static class PercentageSampler implements Sampler {
        private final int percentage;
        private final Random random = new Random();
        
        public PercentageSampler(int percentage) {
            this.percentage = percentage;
        }
        
        @Override
        public boolean shouldSample() {
            return random.nextInt(100) < percentage;
        }
    }
    
    public static class AlwaysSampler implements Sampler {
        @Override
        public boolean shouldSample() {
            return true;
        }
    }
}
```

## Log Analysis and Monitoring

### Log Metrics
```java
@Service
public class LogMetricsCollector {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter errorLogs = Counter.builder("logs_errors_total")
        .description("Total error log entries")
        .register(meterRegistry);
    
    private final Counter warnLogs = Counter.builder("logs_warnings_total")
        .description("Total warning log entries")
        .register(meterRegistry);
    
    private final Histogram logProcessingTime = Histogram.builder("log_processing_time")
        .description("Time spent processing logs")
        .register(meterRegistry);
    
    public void recordLogEvent(Level level, long processingTimeMs) {
        switch (level.toInt()) {
            case Level.ERROR_INT:
                errorLogs.increment();
                break;
            case Level.WARN_INT:
                warnLogs.increment();
                break;
        }
        
        logProcessingTime.observe(processingTimeMs / 1000.0);
    }
}
```

### Log Alerts
```yaml
groups:
  - name: logging_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(logs_errors_total[5m]) > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} errors per second"

      - alert: LogVolumeSpike
        expr: rate(log_lines_total[5m]) > 1000
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Log volume spike detected"
          description: "Log rate is {{ $value }} lines per second"

      - alert: LogProcessingDelay
        expr: histogram_quantile(0.95, rate(log_processing_time_bucket[5m])) > 1
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "Log processing delay"
          description: "95th percentile log processing time is {{ $value }}s"
```

## Best Practices

### Logging Guidelines
- **Be Concise**: Log only necessary information
- **Use Appropriate Levels**: Choose the right log level for each message
- **Include Context**: Add correlation IDs, user IDs, and relevant metadata
- **Avoid Sensitive Data**: Never log passwords, tokens, or personal information
- **Structure Logs**: Use structured logging for better analysis

### Operational Practices
- **Centralize Logs**: Use log aggregation systems like ELK stack
- **Set Retention Policies**: Define how long to keep different types of logs
- **Monitor Log Health**: Track log volume, error rates, and processing delays
- **Implement Log Rotation**: Prevent logs from filling up disk space

### Development Practices
- **Consistent Formatting**: Use consistent log message formats across the application
- **Parameterized Logging**: Use parameterized logging to prevent log injection
- **Test Logging**: Ensure logging works correctly in different environments
- **Document Log Messages**: Document what different log messages mean

## Real-World Logging Examples

### E-commerce Order Processing
```java
@Service
public class OrderProcessingLogger {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderProcessingLogger.class);
    
    @Autowired
    private AuditLogger auditLogger;
    
    public void logOrderLifecycle(Order order, OrderEvent event) {
        MDC.put("orderId", order.getId().toString());
        MDC.put("userId", order.getUserId().toString());
        
        try {
            switch (event.getType()) {
                case CREATED:
                    logger.info("Order created - amount: {}, items: {}", 
                               order.getTotalAmount(), order.getItems().size());
                    auditLogger.logBusinessEvent("ORDER_CREATED", order.getId().toString(), 
                                               Map.of("amount", order.getTotalAmount()));
                    break;
                    
                case PAYMENT_PROCESSED:
                    logger.info("Payment processed for order - paymentId: {}", 
                               event.getPaymentId());
                    break;
                    
                case SHIPPED:
                    logger.info("Order shipped - tracking: {}", event.getTrackingNumber());
                    break;
                    
                case DELIVERED:
                    logger.info("Order delivered successfully");
                    break;
                    
                case CANCELLED:
                    logger.warn("Order cancelled - reason: {}", event.getCancellationReason());
                    break;
            }
        } finally {
            MDC.clear();
        }
    }
}
```

### API Request Logging
```java
@Aspect
@Component
public class ApiLoggingAspect {
    
    private static final Logger logger = LoggerFactory.getLogger(ApiLoggingAspect.class);
    
    @Around("@within(org.springframework.web.bind.annotation.RestController)")
    public Object logApiRequests(ProceedingJoinPoint joinPoint) throws Throwable {
        HttpServletRequest request = ((ServletRequestAttributes) 
            RequestContextHolder.currentRequestAttributes()).getRequest();
        
        String method = request.getMethod();
        String uri = request.getRequestURI();
        String clientIp = getClientIpAddress(request);
        
        long startTime = System.currentTimeMillis();
        
        try {
            Object result = joinPoint.proceed();
            
            long duration = System.currentTimeMillis() - startTime;
            
            logger.info("API Request - method: {}, uri: {}, clientIp: {}, duration: {}ms, status: 200", 
                       method, uri, clientIp, duration);
            
            return result;
            
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            
            logger.error("API Request Failed - method: {}, uri: {}, clientIp: {}, duration: {}ms, error: {}", 
                        method, uri, clientIp, duration, e.getMessage());
            
            throw e;
        }
    }
    
    private String getClientIpAddress(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        
        String xRealIp = request.getHeader("X-Real-IP");
        if (xRealIp != null && !xRealIp.isEmpty()) {
            return xRealIp;
        }
        
        return request.getRemoteAddr();
    }
}
```

## Conclusion

Effective logging is crucial for maintaining, debugging, and monitoring distributed systems. By implementing structured logging, correlation IDs, centralized log aggregation, and appropriate log levels, teams can gain valuable insights into system behavior and quickly resolve issues.

**Key Takeaways:**
- **Structured Logging**: Use JSON format for better analysis and searchability
- **Correlation IDs**: Track requests across service boundaries
- **Centralized Aggregation**: Use ELK stack for log collection and analysis
- **Security**: Mask sensitive data and implement audit logging
- **Performance**: Use asynchronous logging and sampling to maintain performance

Proper logging practices enable better observability, faster troubleshooting, and improved system reliability.
