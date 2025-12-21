# Metrics and Alerting

Metrics and alerting provide quantitative monitoring and proactive notification systems for distributed applications. Metrics track system performance and health indicators, while alerting notifies teams of issues requiring attention. This guide covers metrics collection, alerting strategies, and implementation patterns for building robust monitoring systems.

## What are Metrics and Alerting?

Metrics are numerical measurements of system performance, health, and behavior collected over time. Alerting uses metrics to detect anomalies and notify teams when predefined conditions are met, enabling proactive issue resolution.

### Metrics Types
- **System Metrics**: CPU, memory, disk, network usage
- **Application Metrics**: Response times, error rates, throughput
- **Business Metrics**: User registrations, orders placed, revenue
- **Custom Metrics**: Domain-specific measurements

### Alerting Components
- **Alert Rules**: Conditions that trigger alerts
- **Alert Channels**: Methods to deliver notifications (email, Slack, PagerDuty)
- **Alert Severity**: Critical, warning, info levels
- **Alert Lifecycle**: Generation, routing, acknowledgment, resolution

## Metrics Collection Frameworks

### Micrometer (Spring Boot Integration)

#### Basic Metrics Configuration
```java
@Configuration
public class MetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCustomizer() {
        return registry -> {
            // Add common tags to all metrics
            registry.config()
                .commonTags("application", "ecommerce-service")
                .commonTags("environment", getEnvironment())
                .commonTags("instance", getInstanceId());
        };
    }
    
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
    
    private String getEnvironment() {
        return System.getProperty("spring.profiles.active", "default");
    }
    
    private String getInstanceId() {
        return System.getenv("HOSTNAME") != null ? 
               System.getenv("HOSTNAME") : "unknown";
    }
}

@Service
public class OrderMetricsService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter ordersCreated = Counter.builder("orders_created_total")
        .description("Total number of orders created")
        .register(meterRegistry);
    
    private final Counter ordersFailed = Counter.builder("orders_failed_total")
        .description("Total number of failed order attempts")
        .register(meterRegistry);
    
    private final Histogram orderValue = Histogram.builder("order_value_dollars")
        .description("Distribution of order values")
        .register(meterRegistry);
    
    private final Gauge activeOrders = Gauge.builder("orders_active")
        .description("Number of currently active orders")
        .register(meterRegistry);
    
    public void recordOrderCreated(Order order) {
        ordersCreated.increment();
        orderValue.observe(order.getTotalAmount().doubleValue());
        
        // Update active orders gauge
        // In real implementation, you'd track this properly
    }
    
    public void recordOrderFailed(Order order, Exception e) {
        ordersFailed.increment();
        
        // Record error details
        meterRegistry.counter("orders_failed_by_reason", 
            "reason", e.getClass().getSimpleName()).increment();
    }
}
```

#### Custom Metrics Implementation
```java
@Service
public class CustomMetricsService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    public void recordPaymentProcessingTime(long durationMs, boolean success) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            // Payment processing logic
            processPayment();
            
            sample.stop(Timer.builder("payment_processing_duration")
                .description("Payment processing duration")
                .tag("result", "success")
                .register(meterRegistry));
                
        } catch (Exception e) {
            sample.stop(Timer.builder("payment_processing_duration")
                .description("Payment processing duration")
                .tag("result", "failure")
                .tag("error_type", e.getClass().getSimpleName())
                .register(meterRegistry));
                
            throw e;
        }
    }
    
    public void updateInventoryLevel(String productId, int currentStock, int reservedStock) {
        // Gauge for current stock level
        Gauge.builder("inventory_stock_level", () -> currentStock)
            .description("Current stock level for product")
            .tag("product_id", productId)
            .register(meterRegistry);
            
        // Gauge for reserved stock
        Gauge.builder("inventory_reserved_stock", () -> reservedStock)
            .description("Reserved stock for product")
            .tag("product_id", productId)
            .register(meterRegistry);
            
        // Calculate available stock
        int availableStock = currentStock - reservedStock;
        Gauge.builder("inventory_available_stock", () -> availableStock)
            .description("Available stock for product")
            .tag("product_id", productId)
            .register(meterRegistry);
    }
    
    public void recordUserSession(String userId, long sessionDurationMinutes) {
        // Counter for active sessions
        meterRegistry.counter("user_sessions_active").increment();
        
        // Timer for session duration
        Timer.builder("user_session_duration")
            .description("User session duration")
            .tag("user_id", userId)
            .register(meterRegistry)
            .record(Duration.ofMinutes(sessionDurationMinutes));
            
        // Decrement active sessions when session ends
        meterRegistry.counter("user_sessions_active").increment(-1);
    }
    
    private void processPayment() {
        // Simulate payment processing
        try {
            Thread.sleep(ThreadLocalRandom.current().nextInt(100, 1000));
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### Prometheus Client Integration

#### Direct Prometheus Metrics
```java
@Configuration
public class PrometheusMetricsConfig {
    
    @Bean
    public CollectorRegistry collectorRegistry() {
        return CollectorRegistry.defaultRegistry;
    }
    
    @Bean
    public Counter ordersProcessed() {
        return Counter.build()
            .name("orders_processed_total")
            .help("Total number of processed orders")
            .labelNames("status", "service")
            .register();
    }
    
    @Bean
    public Histogram requestLatency() {
        return Histogram.build()
            .name("http_request_duration_seconds")
            .help("HTTP request duration in seconds")
            .labelNames("method", "endpoint", "status")
            .register();
    }
    
    @Bean
    public Gauge activeConnections() {
        return Gauge.build()
            .name("active_connections")
            .help("Number of active connections")
            .register();
    }
}

@Service
public class PrometheusMetricsService {
    
    @Autowired
    private Counter ordersProcessed;
    
    @Autowired
    private Histogram requestLatency;
    
    @Autowired
    private Gauge activeConnections;
    
    public void recordOrderProcessed(String status) {
        ordersProcessed.labels(status, "order-service").inc();
    }
    
    public void recordHttpRequest(String method, String endpoint, String status, double durationSeconds) {
        requestLatency.labels(method, endpoint, status).observe(durationSeconds);
    }
    
    public void updateActiveConnections(int count) {
        activeConnections.set(count);
    }
    
    public void incrementActiveConnections() {
        activeConnections.inc();
    }
    
    public void decrementActiveConnections() {
        activeConnections.dec();
    }
}
```

#### Custom Prometheus Collectors
```java
@Service
public class DatabaseMetricsCollector extends Collector {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    public List<MetricFamilySamples> collect() {
        List<MetricFamilySamples> samples = new ArrayList<>();
        
        try {
            // Collect connection pool metrics
            if (dataSource instanceof HikariDataSource) {
                HikariDataSource hikari = (HikariDataSource) dataSource;
                HikariPoolMXBean poolBean = hikari.getHikariPoolMXBean();
                
                // Active connections
                samples.add(new MetricFamilySamples.Sample(
                    "db_connections_active", 
                    Collections.emptyList(), 
                    poolBean.getActiveConnections()));
                    
                // Idle connections
                samples.add(new MetricFamilySamples.Sample(
                    "db_connections_idle", 
                    Collections.emptyList(), 
                    poolBean.getIdleConnections()));
                    
                // Total connections
                samples.add(new MetricFamilySamples.Sample(
                    "db_connections_total", 
                    Collections.emptyList(), 
                    poolBean.getTotalConnections()));
                    
                // Threads awaiting connection
                samples.add(new MetricFamilySamples.Sample(
                    "db_connections_pending", 
                    Collections.emptyList(), 
                    poolBean.getThreadsAwaitingConnection()));
            }
            
        } catch (Exception e) {
            logger.error("Failed to collect database metrics", e);
        }
        
        return samples;
    }
}
```

## Alerting Systems

### Prometheus Alert Manager

#### Alert Rules Configuration
```yaml
groups:
  - name: system_alerts
    rules:
      # CPU usage alerts
      - alert: HighCpuUsage
        expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 5m
        labels:
          severity: critical
          team: infrastructure
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value }}%"
          dashboard: "https://grafana.example.com/d/cpu-usage"

      # Memory usage alerts
      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 10m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value }}%"

      # Disk space alerts
      - alert: LowDiskSpace
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 10
        for: 5m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk space available: {{ $value }}%"

  - name: application_alerts
    rules:
      # HTTP error rate alerts
      - alert: HighHttpErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High HTTP error rate on {{ $labels.service }}"
          description: "HTTP error rate is {{ $value | humanizePercentage }}"

      # Response time alerts
      - alert: SlowResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Slow response time on {{ $labels.service }}"
          description: "95th percentile response time is {{ $value }}s"

      # Database connection alerts
      - alert: HighDatabaseConnections
        expr: db_connections_active > 80
        for: 3m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "High database connection usage"
          description: "Active database connections: {{ $value }}"

  - name: business_alerts
    rules:
      # Order processing alerts
      - alert: OrderProcessingFailed
        expr: rate(orders_failed_total[5m]) > 5
        for: 2m
        labels:
          severity: critical
          team: business
        annotations:
          summary: "High order processing failure rate"
          description: "Order failure rate: {{ $value }} per second"

      # Payment processing alerts
      - alert: PaymentProcessingSlow
        expr: histogram_quantile(0.95, rate(payment_processing_duration_bucket[5m])) > 30
        for: 3m
        labels:
          severity: warning
          team: payments
        annotations:
          summary: "Slow payment processing"
          description: "95th percentile payment processing time: {{ $value }}s"
```

#### Alert Manager Configuration
```yaml
global:
  smtp_smarthost: 'smtp.company.com:587'
  smtp_from: 'alertmanager@company.com'
  smtp_auth_username: 'alertmanager'
  smtp_auth_password: 'password'

route:
  group_by: ['alertname', 'team']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 4h
  receiver: 'team-routing'
  routes:
  - match:
      team: infrastructure
    receiver: 'infrastructure-pager'
    group_wait: 5s
    repeat_interval: 30m
    
  - match:
      team: backend
    receiver: 'backend-slack'
    
  - match:
      team: database
    receiver: 'database-email'
    
  - match:
      severity: critical
    receiver: 'critical-pager'
    group_wait: 5s
    repeat_interval: 5m

receivers:
- name: 'infrastructure-pager'
  pagerduty_configs:
  - service_key: 'infrastructure-service-key'
    
- name: 'backend-slack'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/.../.../...'
    channel: '#backend-alerts'
    title: '{{ .GroupLabels.alertname }}'
    text: '{{ .CommonAnnotations.description }}'
    
- name: 'database-email'
  email_configs:
  - to: 'database-team@company.com'
    subject: '{{ .GroupLabels.alertname }}'
    body: '{{ .CommonAnnotations.description }}'
    
- name: 'critical-pager'
  pagerduty_configs:
  - service_key: 'critical-service-key'
    
- name: 'team-routing'
  email_configs:
  - to: 'oncall@company.com'
```

### Custom Alerting Service

#### Alert Generation and Management
```java
@Service
public class AlertingService {
    
    @Autowired
    private AlertRepository alertRepository;
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private AlertDeduplicationService deduplicationService;
    
    private final Map<String, AlertRule> alertRules = new ConcurrentHashMap<>();
    
    @PostConstruct
    public void initializeAlertRules() {
        // System alerts
        alertRules.put("high_cpu", new AlertRule("high_cpu", 
            metrics -> metrics.getCpuUsage() > 90, Severity.CRITICAL));
            
        alertRules.put("high_memory", new AlertRule("high_memory", 
            metrics -> metrics.getMemoryUsage() > 85, Severity.WARNING));
            
        // Application alerts
        alertRules.put("high_error_rate", new AlertRule("high_error_rate", 
            metrics -> metrics.getErrorRate() > 0.05, Severity.CRITICAL));
            
        alertRules.put("slow_response_time", new AlertRule("slow_response_time", 
            metrics -> metrics.getP95ResponseTime() > 2000, Severity.WARNING));
            
        // Business alerts
        alertRules.put("low_order_conversion", new AlertRule("low_order_conversion", 
            metrics -> metrics.getOrderConversionRate() < 0.02, Severity.WARNING));
    }
    
    @Scheduled(fixedRate = 30000) // Check every 30 seconds
    public void evaluateAlerts() {
        SystemMetrics currentMetrics = collectCurrentMetrics();
        
        for (AlertRule rule : alertRules.values()) {
            boolean conditionMet = rule.getCondition().test(currentMetrics);
            
            if (conditionMet) {
                createOrUpdateAlert(rule, currentMetrics);
            } else {
                resolveAlertIfExists(rule.getName());
            }
        }
    }
    
    private void createOrUpdateAlert(AlertRule rule, SystemMetrics metrics) {
        String alertKey = rule.getName();
        
        // Check for existing alert
        Alert existingAlert = alertRepository.findByKey(alertKey);
        
        if (existingAlert == null) {
            // Create new alert
            Alert alert = new Alert();
            alert.setKey(alertKey);
            alert.setName(rule.getName());
            alert.setSeverity(rule.getSeverity());
            alert.setStatus(AlertStatus.FIRING);
            alert.setDescription(generateDescription(rule, metrics));
            alert.setCreatedAt(Instant.now());
            alert.setUpdatedAt(Instant.now());
            
            alertRepository.save(alert);
            
            // Send notification
            notificationService.sendAlert(alert);
            
        } else if (existingAlert.getStatus() == AlertStatus.RESOLVED) {
            // Re-fire existing alert
            existingAlert.setStatus(AlertStatus.FIRING);
            existingAlert.setUpdatedAt(Instant.now());
            existingAlert.setResolvedAt(null);
            
            alertRepository.save(existingAlert);
            
            // Send notification for re-fired alert
            notificationService.sendAlert(existingAlert);
        }
    }
    
    private void resolveAlertIfExists(String alertKey) {
        Alert alert = alertRepository.findByKey(alertKey);
        
        if (alert != null && alert.getStatus() == AlertStatus.FIRING) {
            alert.setStatus(AlertStatus.RESOLVED);
            alert.setResolvedAt(Instant.now());
            alert.setUpdatedAt(Instant.now());
            
            alertRepository.save(alert);
            
            // Send resolution notification
            notificationService.sendAlertResolution(alert);
        }
    }
    
    private String generateDescription(AlertRule rule, SystemMetrics metrics) {
        switch (rule.getName()) {
            case "high_cpu":
                return String.format("CPU usage is %.2f%%", metrics.getCpuUsage());
            case "high_memory":
                return String.format("Memory usage is %.2f%%", metrics.getMemoryUsage());
            case "high_error_rate":
                return String.format("Error rate is %.2f%%", metrics.getErrorRate() * 100);
            case "slow_response_time":
                return String.format("P95 response time is %.2fms", metrics.getP95ResponseTime());
            default:
                return "Alert condition met";
        }
    }
    
    private SystemMetrics collectCurrentMetrics() {
        // Collect current system metrics
        // This would integrate with your metrics collection system
        return new SystemMetrics();
    }
}
```

#### Alert Deduplication and Grouping
```java
@Service
public class AlertDeduplicationService {
    
    private final Map<String, AlertGroup> alertGroups = new ConcurrentHashMap<>();
    private final ScheduledExecutorService cleanupExecutor = Executors.newScheduledThreadPool(1);
    
    public AlertDeduplicationService() {
        // Clean up old alert groups periodically
        cleanupExecutor.scheduleAtFixedRate(this::cleanupOldGroups, 1, 1, TimeUnit.HOURS);
    }
    
    public AlertGroup deduplicateAlert(Alert alert) {
        String groupKey = generateGroupKey(alert);
        
        return alertGroups.compute(groupKey, (key, existingGroup) -> {
            if (existingGroup == null) {
                // Create new group
                AlertGroup newGroup = new AlertGroup();
                newGroup.setKey(groupKey);
                newGroup.setAlerts(new ArrayList<>());
                newGroup.setCreatedAt(Instant.now());
                newGroup.addAlert(alert);
                return newGroup;
            } else {
                // Add to existing group
                existingGroup.addAlert(alert);
                existingGroup.setUpdatedAt(Instant.now());
                return existingGroup;
            }
        });
    }
    
    private String generateGroupKey(Alert alert) {
        // Group alerts by type and affected component
        return alert.getName() + ":" + extractComponentFromAlert(alert);
    }
    
    private String extractComponentFromAlert(Alert alert) {
        // Extract component information from alert labels or description
        // This would depend on your alert structure
        return "default-component";
    }
    
    private void cleanupOldGroups() {
        Instant cutoffTime = Instant.now().minus(Duration.ofHours(24));
        
        alertGroups.entrySet().removeIf(entry -> {
            AlertGroup group = entry.getValue();
            return group.getUpdatedAt().isBefore(cutoffTime) && 
                   group.getAlerts().stream().allMatch(alert -> alert.getStatus() == AlertStatus.RESOLVED);
        });
    }
    
    public static class AlertGroup {
        private String key;
        private List<Alert> alerts;
        private Instant createdAt;
        private Instant updatedAt;
        
        // Getters and setters
        
        public void addAlert(Alert alert) {
            if (alerts == null) {
                alerts = new ArrayList<>();
            }
            alerts.add(alert);
        }
        
        public int getAlertCount() {
            return alerts != null ? alerts.size() : 0;
        }
        
        public AlertSeverity getMaxSeverity() {
            return alerts.stream()
                .map(Alert::getSeverity)
                .max(Comparator.comparingInt(AlertSeverity::getLevel))
                .orElse(AlertSeverity.INFO);
        }
    }
}
```

## Alert Notification Channels

### Email Notifications
```java
@Service
public class EmailAlertNotifier implements AlertNotifier {
    
    @Autowired
    private JavaMailSender mailSender;
    
    @Autowired
    private TemplateEngine templateEngine;
    
    @Override
    public void sendAlert(Alert alert) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            
            helper.setTo(getRecipientsForAlert(alert));
            helper.setSubject(generateSubject(alert));
            helper.setText(generateBody(alert), true);
            
            mailSender.send(message);
            
            logger.info("Sent email alert for {}", alert.getName());
            
        } catch (Exception e) {
            logger.error("Failed to send email alert for {}", alert.getName(), e);
        }
    }
    
    @Override
    public void sendAlertResolution(Alert alert) {
        // Send resolution notification
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            
            helper.setTo(getRecipientsForAlert(alert));
            helper.setSubject("RESOLVED: " + generateSubject(alert));
            helper.setText(generateResolutionBody(alert), true);
            
            mailSender.send(message);
            
        } catch (Exception e) {
            logger.error("Failed to send resolution email for {}", alert.getName(), e);
        }
    }
    
    private String[] getRecipientsForAlert(Alert alert) {
        // Determine recipients based on alert type and severity
        switch (alert.getSeverity()) {
            case CRITICAL:
                return new String[]{"critical-alerts@company.com", "oncall@company.com"};
            case WARNING:
                return new String[]{"team-alerts@company.com"};
            default:
                return new String[]{"monitoring@company.com"};
        }
    }
    
    private String generateSubject(Alert alert) {
        return String.format("[%s] %s", alert.getSeverity(), alert.getName());
    }
    
    private String generateBody(Alert alert) {
        Context context = new Context();
        context.setVariable("alert", alert);
        return templateEngine.process("alert-email-template", context);
    }
    
    private String generateResolutionBody(Alert alert) {
        Context context = new Context();
        context.setVariable("alert", alert);
        return templateEngine.process("alert-resolution-email-template", context);
    }
}
```

### Slack Notifications
```java
@Service
public class SlackAlertNotifier implements AlertNotifier {
    
    @Value("${slack.webhook.url}")
    private String slackWebhookUrl;
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Override
    public void sendAlert(Alert alert) {
        SlackMessage message = createSlackMessage(alert);
        
        try {
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            
            HttpEntity<SlackMessage> request = new HttpEntity<>(message, headers);
            
            ResponseEntity<String> response = restTemplate.postForEntity(
                slackWebhookUrl, request, String.class);
                
            if (response.getStatusCode().is2xxSuccessful()) {
                logger.info("Sent Slack alert for {}", alert.getName());
            } else {
                logger.error("Failed to send Slack alert: {}", response.getBody());
            }
            
        } catch (Exception e) {
            logger.error("Failed to send Slack alert for {}", alert.getName(), e);
        }
    }
    
    @Override
    public void sendAlertResolution(Alert alert) {
        SlackMessage message = createResolutionMessage(alert);
        
        try {
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            
            HttpEntity<SlackMessage> request = new HttpEntity<>(message, headers);
            
            restTemplate.postForEntity(slackWebhookUrl, request, String.class);
            
        } catch (Exception e) {
            logger.error("Failed to send Slack resolution for {}", alert.getName(), e);
        }
    }
    
    private SlackMessage createSlackMessage(Alert alert) {
        SlackMessage message = new SlackMessage();
        message.setChannel(getChannelForAlert(alert));
        
        SlackAttachment attachment = new SlackAttachment();
        attachment.setTitle(alert.getName());
        attachment.setText(alert.getDescription());
        attachment.setColor(getColorForSeverity(alert.getSeverity()));
        
        List<SlackField> fields = new ArrayList<>();
        fields.add(new SlackField("Severity", alert.getSeverity().toString(), true));
        fields.add(new SlackField("Status", alert.getStatus().toString(), true));
        fields.add(new SlackField("Time", alert.getCreatedAt().toString(), false));
        
        attachment.setFields(fields);
        message.setAttachments(Collections.singletonList(attachment));
        
        return message;
    }
    
    private SlackMessage createResolutionMessage(Alert alert) {
        SlackMessage message = new SlackMessage();
        message.setChannel(getChannelForAlert(alert));
        message.setText(String.format("✅ *RESOLVED*: %s", alert.getName()));
        
        return message;
    }
    
    private String getChannelForAlert(Alert alert) {
        switch (alert.getSeverity()) {
            case CRITICAL:
                return "#critical-alerts";
            case WARNING:
                return "#warnings";
            default:
                return "#general-alerts";
        }
    }
    
    private String getColorForSeverity(AlertSeverity severity) {
        switch (severity) {
            case CRITICAL:
                return "danger";  // Red
            case WARNING:
                return "warning"; // Yellow
            default:
                return "good";    // Green
        }
    }
    
    public static class SlackMessage {
        private String channel;
        private String text;
        private List<SlackAttachment> attachments;
        
        // Getters and setters
    }
    
    public static class SlackAttachment {
        private String title;
        private String text;
        private String color;
        private List<SlackField> fields;
        
        // Getters and setters
    }
    
    public static class SlackField {
        private String title;
        private String value;
        private boolean shortField;
        
        public SlackField(String title, String value, boolean shortField) {
            this.title = title;
            this.value = value;
            this.shortField = shortField;
        }
        
        // Getters and setters
    }
}
```

## Metrics Aggregation and Analysis

### Time-Series Analysis
```java
@Service
public class MetricsAnalyzer {
    
    @Autowired
    private MetricsRepository metricsRepository;
    
    public MetricsAnalysis analyzeMetrics(String metricName, Duration timeWindow) {
        Instant endTime = Instant.now();
        Instant startTime = endTime.minus(timeWindow);
        
        List<MetricDataPoint> dataPoints = metricsRepository.getMetrics(metricName, startTime, endTime);
        
        MetricsAnalysis analysis = new MetricsAnalysis();
        analysis.setMetricName(metricName);
        analysis.setTimeWindow(timeWindow);
        analysis.setDataPoints(dataPoints);
        
        // Calculate basic statistics
        analysis.setMin(calculateMin(dataPoints));
        analysis.setMax(calculateMax(dataPoints));
        analysis.setAverage(calculateAverage(dataPoints));
        analysis.setMedian(calculateMedian(dataPoints));
        analysis.setP95(calculatePercentile(dataPoints, 95));
        analysis.setP99(calculatePercentile(dataPoints, 99));
        
        // Detect anomalies
        analysis.setAnomalies(detectAnomalies(dataPoints));
        
        // Calculate trends
        analysis.setTrend(calculateTrend(dataPoints));
        
        return analysis;
    }
    
    private double calculateMin(List<MetricDataPoint> dataPoints) {
        return dataPoints.stream()
            .mapToDouble(MetricDataPoint::getValue)
            .min()
            .orElse(0.0);
    }
    
    private double calculateMax(List<MetricDataPoint> dataPoints) {
        return dataPoints.stream()
            .mapToDouble(MetricDataPoint::getValue)
            .max()
            .orElse(0.0);
    }
    
    private double calculateAverage(List<MetricDataPoint> dataPoints) {
        return dataPoints.stream()
            .mapToDouble(MetricDataPoint::getValue)
            .average()
            .orElse(0.0);
    }
    
    private double calculateMedian(List<MetricDataPoint> dataPoints) {
        List<Double> values = dataPoints.stream()
            .map(MetricDataPoint::getValue)
            .sorted()
            .collect(Collectors.toList());
            
        int size = values.size();
        if (size == 0) return 0.0;
        if (size % 2 == 0) {
            return (values.get(size / 2 - 1) + values.get(size / 2)) / 2.0;
        } else {
            return values.get(size / 2);
        }
    }
    
    private double calculatePercentile(List<MetricDataPoint> dataPoints, double percentile) {
        List<Double> values = dataPoints.stream()
            .map(MetricDataPoint::getValue)
            .sorted()
            .collect(Collectors.toList());
            
        if (values.isEmpty()) return 0.0;
        
        int index = (int) Math.ceil((percentile / 100.0) * values.size()) - 1;
        return values.get(Math.max(0, Math.min(index, values.size() - 1)));
    }
    
    private List<Anomaly> detectAnomalies(List<MetricDataPoint> dataPoints) {
        List<Anomaly> anomalies = new ArrayList<>();
        
        if (dataPoints.size() < 10) return anomalies;
        
        // Simple anomaly detection based on standard deviation
        double mean = calculateAverage(dataPoints);
        double stdDev = calculateStandardDeviation(dataPoints, mean);
        
        for (MetricDataPoint point : dataPoints) {
            double deviation = Math.abs(point.getValue() - mean);
            if (deviation > 3 * stdDev) { // 3-sigma rule
                anomalies.add(new Anomaly(point.getTimestamp(), point.getValue(), 
                                        deviation / stdDev));
            }
        }
        
        return anomalies;
    }
    
    private double calculateStandardDeviation(List<MetricDataPoint> dataPoints, double mean) {
        double variance = dataPoints.stream()
            .mapToDouble(point -> Math.pow(point.getValue() - mean, 2))
            .average()
            .orElse(0.0);
            
        return Math.sqrt(variance);
    }
    
    private Trend calculateTrend(List<MetricDataPoint> dataPoints) {
        if (dataPoints.size() < 2) return Trend.STABLE;
        
        // Simple linear regression to detect trend
        int n = dataPoints.size();
        double sumX = 0, sumY = 0, sumXY = 0, sumXX = 0;
        
        for (int i = 0; i < n; i++) {
            double x = i; // Time index
            double y = dataPoints.get(i).getValue();
            
            sumX += x;
            sumY += y;
            sumXY += x * y;
            sumXX += x * x;
        }
        
        double slope = (n * sumXY - sumX * sumY) / (n * sumXX - sumX * sumX);
        
        if (slope > 0.1) return Trend.INCREASING;
        if (slope < -0.1) return Trend.DECREASING;
        return Trend.STABLE;
    }
}
```

## Best Practices

### Metrics Collection
- **Define Clear Metrics**: Establish what to measure and why
- **Use Appropriate Granularity**: Balance detail with performance impact
- **Standardize Naming**: Use consistent metric names and labels
- **Consider Cardinality**: Avoid high-cardinality metrics that cause storage issues

### Alerting Strategy
- **Avoid Alert Fatigue**: Only alert on actionable issues
- **Progressive Escalation**: Start with warnings, escalate to critical
- **Include Context**: Provide enough information for quick diagnosis
- **Test Alerts**: Regularly test alerting systems

### Operational Excellence
- **Monitor Monitoring**: Track the health of your monitoring systems
- **Automate Responses**: Implement automatic remediation where possible
- **Document Procedures**: Create runbooks for common alert responses
- **Continuous Improvement**: Regularly review and improve monitoring coverage

### Security Considerations
- **Access Control**: Restrict access to sensitive metrics
- **Data Encryption**: Encrypt metrics data in transit and at rest
- **Audit Logging**: Track access to monitoring systems
- **Compliance**: Ensure monitoring complies with data regulations

## Real-World Examples

### E-commerce Metrics and Alerts
```java
@Service
public class EcommerceMetricsService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter productViews = Counter.builder("product_views_total")
        .description("Total product views")
        .tag("category", "")
        .register(meterRegistry);
        
    private final Counter purchases = Counter.builder("purchases_total")
        .description("Total purchases")
        .tag("payment_method", "")
        .register(meterRegistry);
        
    private final Histogram orderValue = Histogram.builder("order_value_dollars")
        .description("Order value distribution")
        .register(meterRegistry);
        
    private final Timer checkoutTime = Timer.builder("checkout_duration")
        .description("Checkout process duration")
        .register(meterRegistry);
    
    public void recordProductView(String productId, String category) {
        productViews.withTag("category", category).increment();
    }
    
    public void recordPurchase(String orderId, BigDecimal amount, String paymentMethod) {
        purchases.withTag("payment_method", paymentMethod).increment();
        orderValue.observe(amount.doubleValue());
        
        // Business metric
        meterRegistry.counter("revenue_total", "currency", "USD").increment(amount.doubleValue());
    }
    
    @Timed(value = "checkout_process", percentiles = {0.5, 0.95, 0.99})
    public void processCheckout(CheckoutRequest request) {
        // Checkout logic
        // The @Timed annotation will automatically record timing metrics
    }
}

// Alert rules for e-commerce
// In alertmanager.yml
groups:
  - name: ecommerce_alerts
    rules:
      - alert: LowConversionRate
        expr: rate(purchases_total[1h]) / rate(product_views_total[1h]) < 0.02
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Low conversion rate detected"
          description: "Conversion rate is below 2%"

      - alert: SlowCheckout
        expr: histogram_quantile(0.95, rate(checkout_duration_bucket[5m])) > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow checkout process"
          description: "95th percentile checkout time is {{ $value }}s"

      - alert: PaymentFailures
        expr: rate(purchases_total{payment_method="credit_card"}[5m]) / rate(payment_attempts_total{payment_method="credit_card"}[5m]) < 0.95
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High payment failure rate"
          description: "Credit card payment success rate is below 95%"
```

### API Gateway Metrics
```java
@Service
public class ApiGatewayMetricsService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter apiRequests = Counter.builder("api_requests_total")
        .description("Total API requests")
        .tag("service", "")
        .tag("endpoint", "")
        .tag("method", "")
        .tag("status", "")
        .register(meterRegistry);
        
    private final Timer apiLatency = Timer.builder("api_request_duration")
        .description("API request duration")
        .tag("service", "")
        .tag("endpoint", "")
        .register(meterRegistry);
        
    private final Counter rateLimitHits = Counter.builder("rate_limit_hits_total")
        .description("Total rate limit hits")
        .tag("service", "")
        .tag("client", "")
        .register(meterRegistry);
    
    public void recordApiRequest(String service, String endpoint, String method, 
                               String status, long durationMs) {
                               
        apiRequests.withTag("service", service)
                  .withTag("endpoint", endpoint)
                  .withTag("method", method)
                  .withTag("status", status)
                  .increment();
                  
        apiLatency.withTag("service", service)
                 .withTag("endpoint", endpoint)
                 .record(Duration.ofMillis(durationMs));
    }
    
    public void recordRateLimitHit(String service, String clientId) {
        rateLimitHits.withTag("service", service)
                    .withTag("client", clientId)
                    .increment();
    }
}

// API Gateway alerts
groups:
  - name: api_gateway_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(api_requests_total{status=~"5.."}[5m]) / rate(api_requests_total[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High API error rate"
          description: "API error rate is {{ $value | humanizePercentage }}"

      - alert: SlowApiResponse
        expr: histogram_quantile(0.95, rate(api_request_duration_bucket[5m])) > 5
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "Slow API responses"
          description: "95th percentile API response time is {{ $value }}s"

      - alert: RateLimitExceeded
        expr: rate(rate_limit_hits_total[1m]) > 10
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High rate limit hits"
          description: "Rate limit being hit {{ $value }} times per minute"
```

## Conclusion

Metrics and alerting are essential for maintaining reliable, performant distributed systems. By collecting comprehensive metrics, defining appropriate alert rules, and implementing effective notification channels, teams can proactively identify and resolve issues before they impact users.

**Key Takeaways:**
- **Comprehensive Coverage**: Monitor system, application, and business metrics
- **Actionable Alerts**: Design alerts that enable quick diagnosis and resolution
- **Multiple Channels**: Use appropriate notification methods for different alert types
- **Continuous Monitoring**: Regularly review and improve monitoring effectiveness

Effective metrics and alerting enable data-driven decision making and ensure system reliability and performance.
