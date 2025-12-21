# Monitoring

Monitoring is the continuous observation and tracking of system performance, health, and behavior. It provides insights into system operations, enables proactive issue detection, and supports data-driven decision making. This guide covers monitoring concepts, tools, and implementation patterns for building observable systems.

## What is Monitoring?

Monitoring involves collecting, analyzing, and visualizing data about system performance and health. It answers questions about system state, performance trends, and potential issues before they become critical problems.

### Key Concepts
- **Metrics**: Quantitative measurements of system performance
- **Logs**: Structured or unstructured records of system events
- **Traces**: End-to-end request flow through distributed systems
- **Alerts**: Notifications when system conditions breach thresholds
- **Dashboards**: Visual representations of system health and performance

### Monitoring Goals
- **Detect Issues**: Identify problems before they impact users
- **Understand Performance**: Track response times, throughput, and resource usage
- **Capacity Planning**: Monitor resource usage trends for scaling decisions
- **Root Cause Analysis**: Gather data to debug and resolve issues
- **Compliance**: Ensure systems meet operational requirements

## Monitoring Types

### 1. Infrastructure Monitoring

#### System Metrics
Track underlying infrastructure performance and health.

```java
@Service
public class SystemMetricsCollector {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Gauge cpuUsage = Gauge.builder("system_cpu_usage_percent")
        .description("Current CPU usage percentage")
        .register(meterRegistry);
    
    private final Gauge memoryUsage = Gauge.builder("system_memory_usage_bytes")
        .description("Current memory usage in bytes")
        .register(meterRegistry);
    
    private final Gauge diskUsage = Gauge.builder("system_disk_usage_bytes")
        .description("Current disk usage in bytes")
        .register(meterRegistry);
    
    private final Counter networkPacketsIn = Counter.builder("system_network_packets_received_total")
        .description("Total network packets received")
        .register(meterRegistry);
    
    private final Counter networkPacketsOut = Counter.builder("system_network_packets_sent_total")
        .description("Total network packets sent")
        .register(meterRegistry);
    
    @Scheduled(fixedRate = 30000) // Collect every 30 seconds
    public void collectSystemMetrics() {
        try {
            OperatingSystemMXBean osBean = ManagementFactory.getOperatingSystemMXBean();
            
            // CPU usage (simplified - actual implementation would use more precise methods)
            double cpuLoad = osBean.getSystemLoadAverage();
            if (cpuLoad >= 0) {
                // Update gauge with current value
                // Note: In real implementation, you'd use a callback-based approach
            }
            
            // Memory usage
            Runtime runtime = Runtime.getRuntime();
            long totalMemory = runtime.totalMemory();
            long freeMemory = runtime.freeMemory();
            long usedMemory = totalMemory - freeMemory;
            
            // Disk usage
            File root = new File("/");
            long totalSpace = root.getTotalSpace();
            long freeSpace = root.getFreeSpace();
            long usedSpace = totalSpace - freeSpace;
            
            // Network stats would require platform-specific implementations
            // This is simplified for demonstration
            
        } catch (Exception e) {
            logger.error("Failed to collect system metrics", e);
        }
    }
}
```

#### Container Metrics
Monitor containerized applications and orchestration platforms.

```java
@Service
public class ContainerMetricsCollector {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Gauge containerCpuUsage = Gauge.builder("container_cpu_usage_percent")
        .description("Container CPU usage percentage")
        .tag("container", "")
        .register(meterRegistry);
    
    private final Gauge containerMemoryUsage = Gauge.builder("container_memory_usage_bytes")
        .description("Container memory usage in bytes")
        .tag("container", "")
        .register(meterRegistry);
    
    private final Counter containerRestarts = Counter.builder("container_restarts_total")
        .description("Total container restarts")
        .tag("container", "")
        .register(meterRegistry);
    
    @Scheduled(fixedRate = 15000) // Collect every 15 seconds
    public void collectContainerMetrics() {
        try {
            // In Kubernetes environment, use Kubernetes API
            // This is simplified for demonstration
            
            String containerId = getCurrentContainerId();
            String containerName = getCurrentContainerName();
            
            // CPU usage
            double cpuUsage = getContainerCpuUsage(containerId);
            
            // Memory usage  
            long memoryUsage = getContainerMemoryUsage(containerId);
            
            // Container status
            boolean isHealthy = checkContainerHealth(containerId);
            
            // Update metrics with tags
            // In real implementation, you'd need to handle gauge updates properly
            
        } catch (Exception e) {
            logger.error("Failed to collect container metrics", e);
        }
    }
    
    private String getCurrentContainerId() {
        // Read from environment variables or cgroup files
        return System.getenv("HOSTNAME"); // Simplified
    }
    
    private String getCurrentContainerName() {
        return System.getenv("CONTAINER_NAME"); // Simplified
    }
    
    private double getContainerCpuUsage(String containerId) {
        // Would read from /sys/fs/cgroup/cpu/... in Linux containers
        return 0.0; // Simplified
    }
    
    private long getContainerMemoryUsage(String containerId) {
        // Would read from /sys/fs/cgroup/memory/... in Linux containers
        return 0L; // Simplified
    }
    
    private boolean checkContainerHealth(String containerId) {
        // Implement health check logic
        return true; // Simplified
    }
}
```

### 2. Application Monitoring

#### Business Metrics
Track application-specific business logic and user behavior.

```java
@Service
public class BusinessMetricsCollector {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter userRegistrations = Counter.builder("business_user_registrations_total")
        .description("Total user registrations")
        .register(meterRegistry);
    
    private final Counter ordersPlaced = Counter.builder("business_orders_placed_total")
        .description("Total orders placed")
        .register(meterRegistry);
    
    private final Histogram orderValue = Histogram.builder("business_order_value_dollars")
        .description("Order value distribution")
        .register(meterRegistry);
    
    private final Counter apiRequests = Counter.builder("business_api_requests_total")
        .description("Total API requests")
        .tag("endpoint", "")
        .tag("method", "")
        .register(meterRegistry);
    
    private final Timer requestLatency = Timer.builder("business_request_latency")
        .description("Request latency distribution")
        .tag("endpoint", "")
        .register(meterRegistry);
    
    public void recordUserRegistration(String source) {
        userRegistrations.increment();
    }
    
    public void recordOrderPlaced(double orderValue, String currency) {
        ordersPlaced.increment();
        this.orderValue.observe(orderValue);
    }
    
    public void recordApiRequest(String endpoint, String method, long latencyMs) {
        apiRequests.withTag("endpoint", endpoint)
                  .withTag("method", method)
                  .increment();
        
        requestLatency.withTag("endpoint", endpoint)
                     .record(Duration.ofMillis(latencyMs));
    }
}
```

#### Performance Metrics
Monitor application performance and resource usage.

```java
@Service
public class PerformanceMetricsCollector {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Timer httpRequestTimer = Timer.builder("http_request_duration")
        .description("HTTP request duration")
        .tag("method", "")
        .tag("uri", "")
        .tag("status", "")
        .register(meterRegistry);
    
    private final Counter httpRequestsTotal = Counter.builder("http_requests_total")
        .description("Total HTTP requests")
        .tag("method", "")
        .tag("uri", "")
        .tag("status", "")
        .register(meterRegistry);
    
    private final Gauge activeConnections = Gauge.builder("active_connections")
        .description("Number of active connections")
        .register(meterRegistry);
    
    private final Histogram databaseQueryTime = Histogram.builder("database_query_duration")
        .description("Database query duration")
        .tag("table", "")
        .tag("operation", "")
        .register(meterRegistry);
    
    public void recordHttpRequest(String method, String uri, int status, long durationMs) {
        httpRequestTimer.withTag("method", method)
                       .withTag("uri", uri)
                       .withTag("status", String.valueOf(status))
                       .record(Duration.ofMillis(durationMs));
        
        httpRequestsTotal.withTag("method", method)
                        .withTag("uri", uri)
                        .withTag("status", String.valueOf(status))
                        .increment();
    }
    
    public void recordDatabaseQuery(String table, String operation, long durationMs) {
        databaseQueryTime.withTag("table", table)
                        .withTag("operation", operation)
                        .observe(durationMs / 1000.0);
    }
}
```

### 3. Service Mesh Monitoring

#### Istio Metrics
Monitor service mesh traffic and performance.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-service
spec:
  hosts:
  - api.external.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 75
    - destination:
        host: reviews
        subset: v2
      weight: 25
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

#### Service Mesh Metrics Collection
```java
@Configuration
public class ServiceMeshMetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCustomizer() {
        return registry -> {
            // Add common tags for service mesh identification
            registry.config()
                .commonTags("service_mesh", "istio")
                .commonTags("namespace", getCurrentNamespace())
                .commonTags("service", getCurrentServiceName());
        };
    }
    
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
    
    private String getCurrentNamespace() {
        return System.getenv("NAMESPACE") != null ? 
               System.getenv("NAMESPACE") : "default";
    }
    
    private String getCurrentServiceName() {
        return System.getenv("SERVICE_NAME") != null ? 
               System.getenv("SERVICE_NAME") : "unknown";
    }
}

@Service
public class ServiceMeshMetricsCollector {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter serviceRequests = Counter.builder("service_requests_total")
        .description("Total service requests")
        .tag("source_service", "")
        .tag("target_service", "")
        .register(meterRegistry);
    
    private final Histogram serviceLatency = Histogram.builder("service_latency")
        .description("Service call latency")
        .tag("source_service", "")
        .tag("target_service", "")
        .register(meterRegistry);
    
    private final Counter circuitBreakerOpens = Counter.builder("circuit_breaker_opens_total")
        .description("Circuit breaker open events")
        .tag("service", "")
        .register(meterRegistry);
    
    @Timed(value = "service_call", percentiles = {0.5, 0.95, 0.99})
    public <T> T callService(String targetService, Supplier<T> serviceCall) {
        String sourceService = getCurrentServiceName();
        
        serviceRequests.withTag("source_service", sourceService)
                      .withTag("target_service", targetService)
                      .increment();
        
        long startTime = System.nanoTime();
        try {
            T result = serviceCall.get();
            
            long latencyNs = System.nanoTime() - startTime;
            serviceLatency.withTag("source_service", sourceService)
                         .withTag("target_service", targetService)
                         .observe(latencyNs / 1_000_000_000.0);
            
            return result;
            
        } catch (Exception e) {
            // Record error metrics
            throw e;
        }
    }
    
    public void recordCircuitBreakerOpen(String service) {
        circuitBreakerOpens.withTag("service", service).increment();
    }
    
    private String getCurrentServiceName() {
        return System.getenv("SERVICE_NAME") != null ? 
               System.getenv("SERVICE_NAME") : "unknown";
    }
}
```

## Monitoring Tools and Frameworks

### 1. Prometheus

#### Prometheus Configuration
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:8080']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'blackbox-exporter'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://example.com
        - https://another-example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115
```

#### Spring Boot Prometheus Integration
```java
@Configuration
public class PrometheusMetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> prometheusMetricsCustomizer() {
        return registry -> {
            // Configure common tags
            registry.config()
                .commonTags("application", "my-service")
                .commonTags("instance", getInstanceId());
        };
    }
    
    @Bean
    public ServletRegistrationBean<MetricsServlet> metricsServlet() {
        MetricsServlet metricsServlet = new MetricsServlet();
        ServletRegistrationBean<MetricsServlet> servletRegistrationBean = 
            new ServletRegistrationBean<>(metricsServlet, "/metrics");
        return servletRegistrationBean;
    }
    
    private String getInstanceId() {
        return System.getenv("HOSTNAME") != null ? 
               System.getenv("HOSTNAME") : "unknown";
    }
}

@RestController
public class MetricsController {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @GetMapping("/metrics")
    public String getMetrics() {
        // Return Prometheus-formatted metrics
        StringBuilder metrics = new StringBuilder();
        
        // Add custom metrics
        metrics.append("# HELP custom_metric Custom metric example\n");
        metrics.append("# TYPE custom_metric gauge\n");
        metrics.append("custom_metric 42\n");
        
        return metrics.toString();
    }
}
```

### 2. Grafana

#### Dashboard Configuration
```json
{
  "dashboard": {
    "title": "System Overview",
    "tags": ["system", "overview"],
    "timezone": "browser",
    "panels": [
      {
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "100 - (avg by (instance) (irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "{{instance}}"
          }
        ]
      },
      {
        "title": "Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100",
            "legendFormat": "{{instance}}"
          }
        ]
      },
      {
        "title": "HTTP Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{uri}}"
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "30s"
  }
}
```

### 3. ELK Stack (Elasticsearch, Logstash, Kibana)

#### Logstash Configuration
```conf
input {
  file {
    path => "/var/log/application.log"
    start_position => "beginning"
  }
  
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{DATA:logger} - %{GREEDYDATA:message}" }
  }
  
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
  }
  
  mutate {
    remove_field => [ "timestamp" ]
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "application-logs-%{+YYYY.MM.dd}"
  }
  
  stdout {
    codec => rubydebug
  }
}
```

#### Kibana Visualization
```json
{
  "title": "Error Rate Over Time",
  "type": "line",
  "params": {
    "type": "line",
    "grid": {
      "categoryLines": false
    },
    "categoryAxes": [
      {
        "id": "CategoryAxis-1",
        "type": "category",
        "position": "bottom",
        "show": true,
        "style": {},
        "scale": {
          "type": "linear"
        },
        "labels": {
          "show": true,
          "truncate": 100
        },
        "title": {}
      }
    ],
    "valueAxes": [
      {
        "id": "ValueAxis-1",
        "name": "LeftAxis-1",
        "type": "value",
        "position": "left",
        "show": true,
        "style": {},
        "scale": {
          "type": "linear",
          "mode": "normal"
        },
        "labels": {
          "show": true,
          "rotate": 0,
          "filter": false,
          "truncate": 100
        },
        "title": {
          "text": "Error Count"
        }
      }
    ],
    "seriesParams": [
      {
        "show": true,
        "type": "line",
        "mode": "normal",
        "data": {
          "label": "Count",
          "id": "1"
        },
        "valueAxis": "ValueAxis-1",
        "drawLinesBetweenPoints": true,
        "showCircles": true
      }
    ],
    "addTooltip": true,
    "addLegend": true,
    "legendPosition": "right",
    "times": [],
    "addTimeMarker": false,
    "labels": {},
    "thresholdLine": {
      "show": false,
      "value": 10,
      "width": 1,
      "style": "full",
      "color": "#E24D42"
    }
  },
  "aggs": [
    {
      "id": "1",
      "enabled": true,
      "type": "date_histogram",
      "schema": "segment",
      "params": {
        "field": "@timestamp",
        "interval": "auto",
        "customInterval": "2h",
        "min_doc_count": 1,
        "extended_bounds": {},
        "customLabel": "Time"
      }
    },
    {
      "id": "2",
      "enabled": true,
      "type": "count",
      "schema": "metric",
      "params": {
        "customLabel": "Error Count"
      }
    }
  ]
}
```

## Monitoring Best Practices

### 1. Metrics Collection

#### Metric Naming Conventions
```java
public class MetricsNamingConvention {
    
    public static final String METRIC_PREFIX = "myapp";
    
    // Counter metrics
    public static final String USER_REGISTRATIONS = METRIC_PREFIX + "_user_registrations_total";
    public static final String ORDERS_PLACED = METRIC_PREFIX + "_orders_placed_total";
    public static final String API_REQUESTS = METRIC_PREFIX + "_api_requests_total";
    
    // Gauge metrics
    public static final String ACTIVE_CONNECTIONS = METRIC_PREFIX + "_active_connections";
    public static final String QUEUE_SIZE = METRIC_PREFIX + "_queue_size";
    public static final String CACHE_SIZE = METRIC_PREFIX + "_cache_size";
    
    // Histogram/Timer metrics
    public static final String REQUEST_LATENCY = METRIC_PREFIX + "_request_latency_seconds";
    public static final String DATABASE_QUERY_TIME = METRIC_PREFIX + "_db_query_duration_seconds";
    public static final String EXTERNAL_API_LATENCY = METRIC_PREFIX + "_external_api_latency_seconds";
    
    // Labels/Tags
    public static final String LABEL_METHOD = "method";
    public static final String LABEL_ENDPOINT = "endpoint";
    public static final String LABEL_STATUS = "status";
    public static final String LABEL_SERVICE = "service";
    public static final String LABEL_OPERATION = "operation";
}
```

#### Metric Collection Strategy
```java
@Service
public class MetricsCollectionStrategy {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    // Business metrics - always collect
    private final Counter businessTransactions = Counter.builder("business_transactions_total")
        .description("Business transactions completed")
        .register(meterRegistry);
    
    // Performance metrics - sample based on load
    private final Timer performanceTimer = Timer.builder("performance_operation_duration")
        .description("Performance-critical operation duration")
        .publishPercentiles(0.5, 0.95, 0.99)
        .register(meterRegistry);
    
    // Debug metrics - only in debug mode
    private final Counter debugCounter = Counter.builder("debug_events_total")
        .description("Debug events (only in debug mode)")
        .register(meterRegistry);
    
    public void recordBusinessTransaction(String transactionType) {
        businessTransactions.increment();
    }
    
    @Timed(value = "performance_operation", percentiles = {0.5, 0.95, 0.99})
    public void performOperation() {
        // Operation that needs performance monitoring
    }
    
    public void recordDebugEvent() {
        if (isDebugMode()) {
            debugCounter.increment();
        }
    }
    
    private boolean isDebugMode() {
        return "true".equals(System.getProperty("debug.metrics.enabled"));
    }
}
```

### 2. Alerting Strategy

#### Alert Rules Configuration
```yaml
groups:
  - name: system_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value }}%"

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value }}%"

      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "Service {{ $labels.job }} has been down for more than 1 minute"

  - name: application_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: SlowResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow response time on {{ $labels.instance }}"
          description: "95th percentile response time is {{ $value }}s"
```

#### Alert Manager Configuration
```yaml
global:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager'
  smtp_auth_password: 'password'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'team-email'
  routes:
  - match:
      severity: critical
    receiver: 'critical-pager'
    group_wait: 5s
    repeat_interval: 5m

receivers:
- name: 'team-email'
  email_configs:
  - to: 'team@example.com'
    send_resolved: true

- name: 'critical-pager'
  pagerduty_configs:
  - service_key: 'your-pagerduty-service-key'
    send_resolved: true
```

### 3. Dashboard Design

#### Key Dashboard Principles
```java
@Service
public class DashboardDesignPrinciples {
    
    // 1. Focus on Key Metrics
    public List<String> getKeyMetrics() {
        return Arrays.asList(
            "Request Rate",
            "Error Rate", 
            "Response Time (P95)",
            "CPU Usage",
            "Memory Usage",
            "Disk I/O",
            "Network I/O"
        );
    }
    
    // 2. Use Appropriate Visualizations
    public Map<String, String> getVisualizationGuidelines() {
        return Map.of(
            "time-series", "Line charts for trends over time",
            "distribution", "Histograms for latency distributions",
            "comparison", "Bar charts for comparing values",
            "status", "Status indicators for health checks",
            "ratio", "Gauge charts for utilization percentages"
        );
    }
    
    // 3. Implement Alert Thresholds
    public Map<String, Threshold> getAlertThresholds() {
        return Map.of(
            "cpu_usage", new Threshold(80, 90),      // Warning at 80%, Critical at 90%
            "memory_usage", new Threshold(85, 95),   // Warning at 85%, Critical at 95%
            "error_rate", new Threshold(0.05, 0.10), // Warning at 5%, Critical at 10%
            "response_time_p95", new Threshold(1000, 2000) // Warning at 1s, Critical at 2s
        );
    }
    
    // 4. Color Coding Standards
    public Map<String, String> getColorStandards() {
        return Map.of(
            "healthy", "#00FF00",      // Green
            "warning", "#FFFF00",      // Yellow
            "critical", "#FF0000",     // Red
            "unknown", "#808080"       // Gray
        );
    }
    
    public static class Threshold {
        private final double warning;
        private final double critical;
        
        public Threshold(double warning, double critical) {
            this.warning = warning;
            this.critical = critical;
        }
        
        public double getWarning() { return warning; }
        public double getCritical() { return critical; }
    }
}
```

## Monitoring in Production

### 1. Monitoring Architecture

#### Distributed Monitoring Setup
```yaml
# docker-compose.yml for monitoring stack
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'

volumes:
  prometheus_data:
  grafana_data:
```

### 2. Monitoring as Code

#### Infrastructure as Code for Monitoring
```java
@Configuration
public class MonitoringAsCodeConfig {
    
    @Bean
    public PrometheusRuleManager prometheusRuleManager() {
        return new PrometheusRuleManager();
    }
    
    @Bean
    public GrafanaDashboardManager grafanaDashboardManager() {
        return new GrafanaDashboardManager();
    }
    
    @Bean
    public AlertManager alertManager() {
        return new AlertManager();
    }
}

@Service
public class PrometheusRuleManager {
    
    @Autowired
    private PrometheusAdminClient prometheusClient;
    
    @PostConstruct
    public void deployRules() {
        // Deploy alerting rules
        String rules = loadRulesFromClasspath("/monitoring/rules.yml");
        prometheusClient.updateRules(rules);
        
        // Deploy recording rules
        String recordingRules = loadRecordingRules();
        prometheusClient.updateRecordingRules(recordingRules);
    }
    
    @Scheduled(fixedRate = 300000) // Check every 5 minutes
    public void validateRules() {
        // Validate that rules are correctly deployed
        List<String> deployedRules = prometheusClient.getRules();
        List<String> expectedRules = loadExpectedRules();
        
        if (!deployedRules.equals(expectedRules)) {
            logger.warn("Prometheus rules are out of sync, redeploying");
            deployRules();
        }
    }
}

@Service
public class GrafanaDashboardManager {
    
    @Autowired
    private GrafanaAdminClient grafanaClient;
    
    @PostConstruct
    public void deployDashboards() {
        // Deploy system dashboards
        deployDashboard("/monitoring/dashboards/system-overview.json");
        deployDashboard("/monitoring/dashboards/application-metrics.json");
        deployDashboard("/monitoring/dashboards/business-metrics.json");
    }
    
    private void deployDashboard(String dashboardPath) {
        String dashboardJson = loadResourceAsString(dashboardPath);
        grafanaClient.createOrUpdateDashboard(dashboardJson);
    }
}
```

### 3. Monitoring Security

#### Secure Monitoring Setup
```java
@Configuration
public class SecureMonitoringConfig {
    
    @Bean
    public WebSecurityConfigurerAdapter monitoringSecurity() {
        return new WebSecurityConfigurerAdapter() {
            @Override
            protected void configure(HttpSecurity http) throws Exception {
                http
                    .antMatcher("/actuator/**")
                    .authorizeRequests()
                    .antMatchers("/actuator/health", "/actuator/info").permitAll()
                    .antMatchers("/actuator/**").hasRole("ADMIN")
                    .and()
                    .httpBasic();
            }
        };
    }
    
    @Bean
    public MeterFilter meterFilter() {
        return MeterFilter.deny(id -> {
            // Don't expose sensitive metrics
            String name = id.getName();
            return name.contains("password") || 
                   name.contains("secret") || 
                   name.contains("key");
        });
    }
}

@Service
public class MonitoringAccessControl {
    
    public boolean canAccessMetrics(String userId, String metricName) {
        // Implement access control logic
        if (metricName.startsWith("business_")) {
            return hasBusinessMetricsAccess(userId);
        }
        
        if (metricName.startsWith("system_")) {
            return hasSystemMetricsAccess(userId);
        }
        
        return hasGeneralMetricsAccess(userId);
    }
    
    public boolean canAccessDashboard(String userId, String dashboardId) {
        // Implement dashboard access control
        return true; // Simplified
    }
    
    private boolean hasBusinessMetricsAccess(String userId) {
        // Check user permissions
        return true; // Simplified
    }
    
    private boolean hasSystemMetricsAccess(String userId) {
        // Check user permissions
        return true; // Simplified
    }
    
    private boolean hasGeneralMetricsAccess(String userId) {
        // Check user permissions
        return true; // Simplified
    }
}
```

## Conclusion

Monitoring is essential for maintaining reliable, performant systems. By implementing comprehensive monitoring strategies, collecting appropriate metrics, and setting up effective alerting, teams can proactively identify and resolve issues before they impact users.

**Key Takeaways:**
- **Multiple Monitoring Layers**: Infrastructure, application, and business metrics
- **Appropriate Tools**: Prometheus for metrics, Grafana for visualization, ELK for logs
- **Alerting Strategy**: Define clear alert rules and escalation procedures
- **Security**: Implement access controls for monitoring data
- **Continuous Improvement**: Regularly review and improve monitoring coverage

Effective monitoring enables data-driven decision making and ensures system reliability and performance.
