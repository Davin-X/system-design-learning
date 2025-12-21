# Service Discovery

Service discovery enables automatic detection and location of services in distributed systems. It allows services to find and communicate with each other without hard-coded addresses, supporting dynamic scaling and fault tolerance. This guide covers service discovery patterns, implementations, and best practices.

## What is Service Discovery?

Service discovery is the automatic detection of devices and services offered by these devices on a computer network. In distributed systems, service discovery allows services to locate each other dynamically, enabling loose coupling and scalability.

### Key Concepts
- **Service Registry**: Database of available service instances and their locations
- **Service Registration**: Process of adding a service instance to the registry
- **Service Deregistration**: Process of removing a service instance from the registry
- **Service Lookup**: Process of finding service instances from the registry
- **Health Checks**: Monitoring service instances to ensure they are healthy

### Service Discovery Patterns

#### Client-Side Discovery
Clients query the service registry directly to find available service instances.

**Advantages:**
- Clients have full control over load balancing decisions
- No single point of failure at the load balancer level
- Can implement sophisticated load balancing algorithms

**Disadvantages:**
- Tight coupling between clients and service registry
- Clients must implement service discovery logic
- Registry communication increases client complexity

#### Server-Side Discovery
Clients send requests to a load balancer/router that queries the service registry.

**Advantages:**
- Service discovery logic centralized in load balancer
- Clients remain simple and unaware of service discovery
- Easier to implement and maintain

**Disadvantages:**
- Load balancer becomes a potential single point of failure
- Additional network hop for each request
- Load balancer must be highly available

## Service Discovery Implementations

### Netflix Eureka

#### Server Implementation
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}

# application.yml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

#### Client Implementation
```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
public class ServiceClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceClientApplication.class, args);
    }
}

# application.yml
spring:
  application:
    name: user-service

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    preferIpAddress: true

# Feign client for service communication
@FeignClient(name = "order-service")
public interface OrderServiceClient {
    
    @GetMapping("/orders/{userId}")
    List<Order> getUserOrders(@PathVariable("userId") Long userId);
    
    @PostMapping("/orders")
    Order createOrder(@RequestBody CreateOrderRequest request);
}
```

### Apache ZooKeeper

#### Service Registration
```java
@Service
public class ZooKeeperServiceRegistry {
    
    private ZooKeeper zk;
    private final String registryPath = "/services";
    
    public ZooKeeperServiceRegistry(String connectionString) throws Exception {
        this.zk = new ZooKeeper(connectionString, 3000, null);
        
        // Create registry root if it doesn't exist
        if (zk.exists(registryPath, false) == null) {
            zk.create(registryPath, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }
    }
    
    public void registerService(String serviceName, String serviceId, String address, int port) throws Exception {
        String servicePath = registryPath + "/" + serviceName;
        String instancePath = servicePath + "/" + serviceId;
        
        // Create service directory if it doesn't exist
        if (zk.exists(servicePath, false) == null) {
            zk.create(servicePath, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }
        
        // Register service instance
        ServiceInstance instance = new ServiceInstance(serviceId, address, port, System.currentTimeMillis());
        byte[] data = serialize(instance);
        
        zk.create(instancePath, data, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
        
        logger.info("Registered service {} at {}", serviceName, instancePath);
    }
    
    public void unregisterService(String serviceName, String serviceId) throws Exception {
        String instancePath = registryPath + "/" + serviceName + "/" + serviceId;
        zk.delete(instancePath, -1);
        
        logger.info("Unregistered service {} from {}", serviceName, instancePath);
    }
    
    public List<ServiceInstance> discoverServices(String serviceName) throws Exception {
        String servicePath = registryPath + "/" + serviceName;
        
        if (zk.exists(servicePath, false) == null) {
            return Collections.emptyList();
        }
        
        List<String> instanceIds = zk.getChildren(servicePath, false);
        List<ServiceInstance> instances = new ArrayList<>();
        
        for (String instanceId : instanceIds) {
            String instancePath = servicePath + "/" + instanceId;
            byte[] data = zk.getData(instancePath, false, null);
            ServiceInstance instance = deserialize(data);
            instances.add(instance);
        }
        
        return instances;
    }
    
    public void watchService(String serviceName, Watcher watcher) throws Exception {
        String servicePath = registryPath + "/" + serviceName;
        zk.getChildren(servicePath, watcher);
    }
    
    private byte[] serialize(ServiceInstance instance) throws Exception {
        // JSON serialization
        return objectMapper.writeValueAsBytes(instance);
    }
    
    private ServiceInstance deserialize(byte[] data) throws Exception {
        return objectMapper.readValue(data, ServiceInstance.class);
    }
}
```

#### Service Discovery
```java
@Service
public class ZooKeeperServiceDiscovery {
    
    @Autowired
    private LoadBalancer loadBalancer;
    
    @Autowired
    private ZooKeeperServiceRegistry registry;
    
    private final Map<String, List<ServiceInstance>> serviceCache = new ConcurrentHashMap<>();
    private final Map<String, Watcher> watchers = new ConcurrentHashMap<>();
    
    public ServiceInstance discoverService(String serviceName) {
        List<ServiceInstance> instances = serviceCache.get(serviceName);
        
        if (instances == null || instances.isEmpty()) {
            // Cache miss - fetch from registry
            try {
                instances = registry.discoverServices(serviceName);
                serviceCache.put(serviceName, instances);
                
                // Set up watcher for service changes
                setupWatcher(serviceName);
                
            } catch (Exception e) {
                logger.error("Failed to discover service {}", serviceName, e);
                return null;
            }
        }
        
        // Filter healthy instances
        List<ServiceInstance> healthyInstances = instances.stream()
            .filter(this::isHealthy)
            .collect(Collectors.toList());
        
        if (healthyInstances.isEmpty()) {
            return null;
        }
        
        // Use load balancer to select instance
        return loadBalancer.selectInstance(healthyInstances);
    }
    
    private void setupWatcher(String serviceName) {
        if (watchers.containsKey(serviceName)) {
            return; // Watcher already set up
        }
        
        Watcher watcher = new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == EventType.NodeChildrenChanged) {
                    // Service instances changed - refresh cache
                    try {
                        List<ServiceInstance> updatedInstances = registry.discoverServices(serviceName);
                        serviceCache.put(serviceName, updatedInstances);
                        
                        // Re-setup watcher
                        registry.watchService(serviceName, this);
                        
                        logger.info("Updated service cache for {}", serviceName);
                        
                    } catch (Exception e) {
                        logger.error("Failed to refresh service cache for {}", serviceName, e);
                    }
                }
            }
        };
        
        try {
            registry.watchService(serviceName, watcher);
            watchers.put(serviceName, watcher);
        } catch (Exception e) {
            logger.error("Failed to setup watcher for service {}", serviceName, e);
        }
    }
    
    private boolean isHealthy(ServiceInstance instance) {
        // Implement health check logic
        // Could check last heartbeat, response time, error rate, etc.
        long timeSinceLastSeen = System.currentTimeMillis() - instance.getLastSeen();
        return timeSinceLastSeen < 30000; // 30 seconds
    }
}
```

### etcd

#### Service Registration with etcd
```java
@Service
public class EtcdServiceRegistry {
    
    private Client etcdClient;
    private final String registryPrefix = "/services";
    
    public EtcdServiceRegistry(String[] endpoints) {
        this.etcdClient = Client.builder()
            .endpoints(endpoints)
            .build();
    }
    
    public void registerService(String serviceName, String instanceId, 
                               String address, int port) throws Exception {
        
        String key = registryPrefix + "/" + serviceName + "/" + instanceId;
        
        ServiceInstance instance = new ServiceInstance(instanceId, address, port, System.currentTimeMillis());
        String value = objectMapper.writeValueAsString(instance);
        
        // Register with lease for automatic cleanup
        long leaseId = createLease(30); // 30 second lease
        
        etcdClient.getKVClient().put(
            ByteSequence.from(key.getBytes()),
            ByteSequence.from(value.getBytes()),
            PutOption.newBuilder().withLeaseId(leaseId).build()
        );
        
        // Keep lease alive
        keepLeaseAlive(leaseId);
        
        logger.info("Registered service {} instance {}", serviceName, instanceId);
    }
    
    public void unregisterService(String serviceName, String instanceId) throws Exception {
        String key = registryPrefix + "/" + serviceName + "/" + instanceId;
        
        etcdClient.getKVClient().delete(ByteSequence.from(key.getBytes()));
        
        logger.info("Unregistered service {} instance {}", serviceName, instanceId);
    }
    
    public List<ServiceInstance> discoverServices(String serviceName) throws Exception {
        String prefix = registryPrefix + "/" + serviceName + "/";
        
        GetResponse response = etcdClient.getKVClient().get(
            ByteSequence.from(prefix.getBytes()),
            GetOption.newBuilder().withPrefix(ByteSequence.from(prefix.getBytes())).build()
        );
        
        List<ServiceInstance> instances = new ArrayList<>();
        for (KeyValue kv : response.getKvs()) {
            String value = kv.getValue().toString();
            ServiceInstance instance = objectMapper.readValue(value, ServiceInstance.class);
            instances.add(instance);
        }
        
        return instances;
    }
    
    public void watchService(String serviceName, Consumer<WatchEvent> eventHandler) {
        String prefix = registryPrefix + "/" + serviceName + "/";
        
        Watch watch = etcdClient.getWatchClient();
        watch.watch(
            ByteSequence.from(prefix.getBytes()),
            WatchOption.newBuilder().withPrefix(ByteSequence.from(prefix.getBytes())).build(),
            new Watch.Listener() {
                @Override
                public void onNext(WatchResponse response) {
                    response.getEvents().forEach(event -> {
                        String key = event.getKeyValue().getKey().toString();
                        String instanceId = key.substring(key.lastIndexOf('/') + 1);
                        
                        eventHandler.accept(new WatchEvent(
                            serviceName, 
                            instanceId,
                            event.getEventType()
                        ));
                    });
                }
                
                @Override
                public void onError(Throwable throwable) {
                    logger.error("Watch error for service {}", serviceName, throwable);
                }
                
                @Override
                public void onCompleted() {
                    logger.info("Watch completed for service {}", serviceName);
                }
            }
        );
    }
    
    private long createLease(long ttlSeconds) throws Exception {
        Lease lease = etcdClient.getLeaseClient().grant(ttlSeconds);
        return lease.getID();
    }
    
    private void keepLeaseAlive(long leaseId) {
        Executors.newSingleThreadScheduledExecutor()
            .scheduleAtFixedRate(() -> {
                try {
                    etcdClient.getLeaseClient().keepAliveOnce(leaseId);
                } catch (Exception e) {
                    logger.warn("Failed to keep lease alive: {}", leaseId);
                }
            }, 10, 10, TimeUnit.SECONDS); // Keep alive every 10 seconds
    }
}
```

### Consul

#### Service Registration with Consul
```java
@Service
public class ConsulServiceRegistry {
    
    @Autowired
    private ConsulClient consulClient;
    
    public void registerService(String serviceId, String serviceName, 
                               String address, int port, String healthCheckUrl) {
        
        AgentServiceRegistration registration = AgentServiceRegistration.builder()
            .id(serviceId)
            .name(serviceName)
            .address(address)
            .port(port)
            .check(AgentServiceCheck.builder()
                .http(healthCheckUrl)
                .interval("10s")
                .timeout("5s")
                .deregisterCriticalServiceAfter("30s")
                .build())
            .build();
        
        consulClient.agentServiceRegister(registration);
        
        logger.info("Registered service {} with Consul", serviceName);
    }
    
    public void deregisterService(String serviceId) {
        consulClient.agentServiceDeregister(serviceId);
        
        logger.info("Deregistered service {} from Consul", serviceId);
    }
    
    public List<ServiceInstance> discoverServices(String serviceName) {
        HealthServicesRequest request = HealthServicesRequest.newBuilder()
            .setServiceName(serviceName)
            .setPassing(true) // Only healthy services
            .build();
        
        HealthServicesResponse response = consulClient.getHealthServices(request);
        
        return response.getValue().stream()
            .map(service -> new ServiceInstance(
                service.getService().getId(),
                service.getService().getAddress(),
                service.getService().getPort(),
                System.currentTimeMillis()
            ))
            .collect(Collectors.toList());
    }
    
    public void watchServices(String serviceName, Consumer<ServiceChangeEvent> callback) {
        HealthServicesRequest request = HealthServicesRequest.newBuilder()
            .setServiceName(serviceName)
            .build();
        
        consulClient.getHealthServices(request, new ServiceCallback(callback));
    }
    
    private static class ServiceCallback implements ConsulResponseCallback<HealthServicesResponse> {
        
        private final Consumer<ServiceChangeEvent> callback;
        private List<ServiceHealth> lastServices = new ArrayList<>();
        
        public ServiceCallback(Consumer<ServiceChangeEvent> callback) {
            this.callback = callback;
        }
        
        @Override
        public void onComplete(HealthServicesResponse response) {
            List<ServiceHealth> currentServices = response.getValue();
            
            // Detect changes
            List<ServiceInstance> added = findAddedServices(lastServices, currentServices);
            List<ServiceInstance> removed = findRemovedServices(lastServices, currentServices);
            
            if (!added.isEmpty() || !removed.isEmpty()) {
                callback.accept(new ServiceChangeEvent(added, removed));
            }
            
            lastServices = new ArrayList<>(currentServices);
        }
        
        @Override
        public void onFailure(Throwable throwable) {
            logger.error("Service watch failed", throwable);
        }
        
        private List<ServiceInstance> findAddedServices(List<ServiceHealth> oldList, List<ServiceHealth> newList) {
            return newList.stream()
                .filter(newService -> !containsService(oldList, newService))
                .map(service -> new ServiceInstance(
                    service.getService().getId(),
                    service.getService().getAddress(),
                    service.getService().getPort(),
                    System.currentTimeMillis()
                ))
                .collect(Collectors.toList());
        }
        
        private List<ServiceInstance> findRemovedServices(List<ServiceHealth> oldList, List<ServiceHealth> newList) {
            return oldList.stream()
                .filter(oldService -> !containsService(newList, oldService))
                .map(service -> new ServiceInstance(
                    service.getService().getId(),
                    service.getService().getAddress(),
                    service.getService().getPort(),
                    System.currentTimeMillis()
                ))
                .collect(Collectors.toList());
        }
        
        private boolean containsService(List<ServiceHealth> services, ServiceHealth target) {
            return services.stream()
                .anyMatch(service -> service.getService().getId().equals(target.getService().getId()));
        }
    }
}
```

## Load Balancing Strategies

### Round Robin
```java
public class RoundRobinLoadBalancer {
    
    private final AtomicInteger nextIndex = new AtomicInteger(0);
    
    public ServiceInstance selectInstance(List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            return null;
        }
        
        int index = nextIndex.getAndIncrement() % instances.size();
        return instances.get(index);
    }
}
```

### Weighted Round Robin
```java
public class WeightedRoundRobinLoadBalancer {
    
    private final Map<String, Integer> weights = new ConcurrentHashMap<>();
    private final Map<String, AtomicInteger> currentWeights = new ConcurrentHashMap<>();
    
    public ServiceInstance selectInstance(List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            return null;
        }
        
        ServiceInstance selected = null;
        int totalWeight = 0;
        int maxWeight = -1;
        
        for (ServiceInstance instance : instances) {
            String instanceId = instance.getId();
            int weight = weights.getOrDefault(instanceId, 1);
            AtomicInteger currentWeight = currentWeights.computeIfAbsent(instanceId, k -> new AtomicInteger(0));
            
            currentWeight.addAndGet(weight);
            totalWeight += weight;
            
            if (selected == null || currentWeight.get() > maxWeight) {
                selected = instance;
                maxWeight = currentWeight.get();
            }
        }
        
        // Reset current weights periodically
        if (totalWeight >= 1000) { // Reset threshold
            currentWeights.values().forEach(weight -> weight.set(0));
        }
        
        return selected;
    }
    
    public void setWeight(String instanceId, int weight) {
        weights.put(instanceId, weight);
    }
}
```

### Least Connections
```java
public class LeastConnectionsLoadBalancer {
    
    private final Map<String, AtomicInteger> activeConnections = new ConcurrentHashMap<>();
    
    public ServiceInstance selectInstance(List<ServiceInstance> instances) {
        return instances.stream()
            .min(Comparator.comparing(instance -> 
                activeConnections.computeIfAbsent(instance.getId(), k -> new AtomicInteger(0)).get()))
            .orElse(null);
    }
    
    public void incrementConnections(String instanceId) {
        activeConnections.computeIfAbsent(instanceId, k -> new AtomicInteger(0)).incrementAndGet();
    }
    
    public void decrementConnections(String instanceId) {
        AtomicInteger connections = activeConnections.get(instanceId);
        if (connections != null) {
            connections.decrementAndGet();
        }
    }
}
```

### Random Selection
```java
public class RandomLoadBalancer {
    
    private final Random random = new Random();
    
    public ServiceInstance selectInstance(List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            return null;
        }
        
        int index = random.nextInt(instances.size());
        return instances.get(index);
    }
}
```

## Health Checking

### Passive Health Checks
```java
@Service
public class PassiveHealthChecker {
    
    @Autowired
    private ServiceRegistry registry;
    
    private final Map<String, InstanceHealth> healthStatus = new ConcurrentHashMap<>();
    
    public void recordRequest(String instanceId, boolean success, long responseTime) {
        InstanceHealth health = healthStatus.computeIfAbsent(instanceId, 
            k -> new InstanceHealth(instanceId));
        
        health.recordRequest(success, responseTime);
        
        // Update instance health in registry
        if (!health.isHealthy()) {
            registry.markInstanceUnhealthy(instanceId);
        } else if (health.isHealthy() && health.wasPreviouslyUnhealthy()) {
            registry.markInstanceHealthy(instanceId);
        }
    }
    
    public boolean isHealthy(String instanceId) {
        InstanceHealth health = healthStatus.get(instanceId);
        return health != null && health.isHealthy();
    }
    
    private static class InstanceHealth {
        
        private static final int FAILURE_THRESHOLD = 5;
        private static final int SUCCESS_THRESHOLD = 2;
        private static final long TIMEOUT_THRESHOLD = 5000; // 5 seconds
        
        private final String instanceId;
        private int consecutiveFailures = 0;
        private int consecutiveSuccesses = 0;
        private boolean currentlyHealthy = true;
        
        public InstanceHealth(String instanceId) {
            this.instanceId = instanceId;
        }
        
        public void recordRequest(boolean success, long responseTime) {
            if (!success || responseTime > TIMEOUT_THRESHOLD) {
                consecutiveFailures++;
                consecutiveSuccesses = 0;
                
                if (consecutiveFailures >= FAILURE_THRESHOLD) {
                    currentlyHealthy = false;
                }
            } else {
                consecutiveSuccesses++;
                consecutiveFailures = 0;
                
                if (consecutiveSuccesses >= SUCCESS_THRESHOLD) {
                    currentlyHealthy = true;
                }
            }
        }
        
        public boolean isHealthy() {
            return currentlyHealthy;
        }
        
        public boolean wasPreviouslyUnhealthy() {
            return !currentlyHealthy && consecutiveSuccesses >= SUCCESS_THRESHOLD;
        }
    }
}
```

### Active Health Checks
```java
@Service
public class ActiveHealthChecker {
    
    @Autowired
    private ServiceRegistry registry;
    
    @Autowired
    private WebClient webClient;
    
    @Scheduled(fixedRate = 30000) // Check every 30 seconds
    public void performHealthChecks() {
        List<ServiceInstance> allInstances = registry.getAllInstances();
        
        for (ServiceInstance instance : allInstances) {
            checkInstanceHealth(instance);
        }
    }
    
    private void checkInstanceHealth(ServiceInstance instance) {
        String healthUrl = "http://" + instance.getAddress() + ":" + instance.getPort() + "/health";
        
        webClient.get()
            .uri(healthUrl)
            .retrieve()
            .bodyToMono(String.class)
            .timeout(Duration.ofSeconds(5))
            .doOnSuccess(response -> {
                // Health check passed
                registry.markInstanceHealthy(instance.getId());
            })
            .doOnError(error -> {
                // Health check failed
                logger.warn("Health check failed for instance {}: {}", instance.getId(), error.getMessage());
                registry.markInstanceUnhealthy(instance.getId());
            })
            .subscribe(); // Fire and forget
    }
}
```

## Service Discovery Patterns

### Sidecar Pattern
```java
// Service with sidecar proxy
public class ServiceWithSidecar {
    
    // Service only needs to know sidecar location
    private static final String SIDECAR_HOST = "localhost";
    private static final int SIDECAR_PORT = 15000;
    
    public String callService(String serviceName, String request) {
        // Route through sidecar proxy
        return webClient.post()
            .uri("http://" + SIDECAR_HOST + ":" + SIDECAR_PORT + "/call/" + serviceName)
            .bodyValue(request)
            .retrieve()
            .bodyToMono(String.class)
            .block();
    }
}

// Sidecar proxy handles service discovery
@Service
public class SidecarProxy {
    
    @Autowired
    private ServiceDiscovery discovery;
    
    @Autowired
    private LoadBalancer loadBalancer;
    
    @PostMapping("/call/{serviceName}")
    public Mono<String> callService(@PathVariable String serviceName, @RequestBody String request) {
        // Discover service instances
        List<ServiceInstance> instances = discovery.discoverService(serviceName);
        
        if (instances.isEmpty()) {
            return Mono.error(new ServiceUnavailableException("No instances available"));
        }
        
        // Select instance using load balancer
        ServiceInstance selectedInstance = loadBalancer.selectInstance(instances);
        
        // Forward request
        return webClient.post()
            .uri("http://" + selectedInstance.getAddress() + ":" + selectedInstance.getPort() + "/api")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

### Service Mesh Pattern
```java
@Configuration
public class ServiceMeshConfig {
    
    @Bean
    public ServiceDiscovery serviceDiscovery() {
        // Centralized service discovery
        return new ConsulServiceDiscovery();
    }
    
    @Bean
    public LoadBalancer loadBalancer() {
        // Advanced load balancing
        return new WeightedLoadBalancer();
    }
    
    @Bean
    public CircuitBreaker circuitBreaker() {
        // Circuit breaker for resilience
        return new Resilience4JCircuitBreaker();
    }
    
    @Bean
    public RetryPolicy retryPolicy() {
        // Retry policy for fault tolerance
        return new ExponentialBackoffRetry();
    }
    
    @Bean
    public ServiceClient serviceClient(ServiceDiscovery discovery, 
                                      LoadBalancer loadBalancer,
                                      CircuitBreaker circuitBreaker,
                                      RetryPolicy retryPolicy) {
        
        return new MeshServiceClient(discovery, loadBalancer, circuitBreaker, retryPolicy);
    }
}

public class MeshServiceClient {
    
    private final ServiceDiscovery discovery;
    private final LoadBalancer loadBalancer;
    private final CircuitBreaker circuitBreaker;
    private final RetryPolicy retryPolicy;
    
    public MeshServiceClient(ServiceDiscovery discovery, LoadBalancer loadBalancer,
                           CircuitBreaker circuitBreaker, RetryPolicy retryPolicy) {
        this.discovery = discovery;
        this.loadBalancer = loadBalancer;
        this.circuitBreaker = circuitBreaker;
        this.retryPolicy = retryPolicy;
    }
    
    public <T> T callService(String serviceName, Supplier<T> operation) throws Exception {
        return retryPolicy.execute(() -> 
            circuitBreaker.execute(() -> {
                // Discover service instances
                List<ServiceInstance> instances = discovery.discoverService(serviceName);
                
                if (instances.isEmpty()) {
                    throw new ServiceUnavailableException("No instances available");
                }
                
                // Select instance
                ServiceInstance instance = loadBalancer.selectInstance(instances);
                
                // Execute operation against selected instance
                return operation.get();
            })
        );
    }
}
```

## Monitoring Service Discovery

### Discovery Metrics
```java
@Service
public class ServiceDiscoveryMetrics {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter serviceRegistrations = Counter.builder("service_discovery_registrations_total")
        .description("Total service registrations")
        .register(meterRegistry);
    
    private final Counter serviceDeregistrations = Counter.builder("service_discovery_deregistrations_total")
        .description("Total service deregistrations")
        .register(meterRegistry);
    
    private final Counter serviceLookups = Counter.builder("service_discovery_lookups_total")
        .description("Total service lookups")
        .register(meterRegistry);
    
    private final Counter serviceLookupFailures = Counter.builder("service_discovery_lookup_failures_total")
        .description("Total service lookup failures")
        .register(meterRegistry);
    
    private final Gauge registeredServices = Gauge.builder("service_discovery_registered_services")
        .description("Number of registered services")
        .register(meterRegistry);
    
    private final Histogram discoveryLatency = Histogram.builder("service_discovery_latency")
        .description("Service discovery latency")
        .register(meterRegistry);
    
    public void recordServiceRegistration() {
        serviceRegistrations.increment();
    }
    
    public void recordServiceDeregistration() {
        serviceDeregistrations.increment();
    }
    
    public void recordServiceLookup(long latencyMs) {
        serviceLookups.increment();
        discoveryLatency.observe(latencyMs / 1000.0);
    }
    
    public void recordServiceLookupFailure() {
        serviceLookupFailures.increment();
    }
}
```

## Best Practices

### Service Registration
- **Unique Instance IDs**: Use unique identifiers for each service instance
- **Health Checks**: Implement proper health check endpoints
- **Metadata**: Include version, environment, and capability information
- **TTL**: Use appropriate time-to-live for registrations

### Service Discovery
- **Caching**: Cache discovery results to reduce registry load
- **Fallbacks**: Implement fallback mechanisms for discovery failures
- **Load Balancing**: Use appropriate load balancing strategies
- **Monitoring**: Track discovery performance and failures

### Operational Considerations
- **High Availability**: Ensure registry is highly available
- **Scalability**: Choose registry that scales with your needs
- **Security**: Secure service registration and discovery
- **Network Awareness**: Consider network topology in service placement

## Conclusion

Service discovery is fundamental to building scalable, resilient distributed systems. By implementing proper service registration, discovery, and load balancing, systems can automatically adapt to changing topologies and maintain reliable communication between services.

**Key Takeaways:**
- **Client vs Server-Side**: Choose discovery pattern based on requirements
- **Service Registry**: Use reliable, scalable registry (Eureka, ZooKeeper, etcd, Consul)
- **Health Checks**: Implement both passive and active health monitoring
- **Load Balancing**: Use appropriate load balancing for your use case
- **Monitoring**: Track discovery metrics and performance

Effective service discovery enables loose coupling, automatic scaling, and fault tolerance in distributed systems.
