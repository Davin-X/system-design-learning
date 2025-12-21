# Database Replication

Database replication is the process of copying and maintaining database objects across multiple database servers. It provides high availability, fault tolerance, and improved read performance by distributing data across multiple nodes.

## What is Database Replication?

Database replication creates and maintains multiple copies of the same database across different servers. Changes made to one database (master/primary) are automatically propagated to other databases (replicas/secondaries).

### Why Replication Matters
- **High Availability**: System continues working if one server fails
- **Fault Tolerance**: Automatic failover to replica servers
- **Read Scalability**: Distribute read queries across multiple servers
- **Disaster Recovery**: Data protection and quick recovery
- **Geographic Distribution**: Serve users from nearby data centers

## Replication Architectures

### Master-Slave Replication

#### Architecture
- **One Master**: Handles all write operations
- **Multiple Slaves**: Handle read operations only
- **Asynchronous Replication**: Changes propagate from master to slaves

#### How It Works
1. **Write Operation**: All writes go to master database
2. **Log Changes**: Master records changes in binary log
3. **Replication Stream**: Changes sent to slave servers
4. **Apply Changes**: Slaves apply changes to their copies
5. **Read Operations**: Reads distributed across master and slaves

#### Advantages
- **Simple Setup**: Easy to implement and understand
- **Read Scalability**: Scale reads by adding more slaves
- **Data Consistency**: Strong consistency for writes

#### Disadvantages
- **Single Point of Failure**: Master failure stops all writes
- **Replication Lag**: Slaves may have stale data
- **Write Bottleneck**: Master can become overloaded

#### Use Cases
- **Read-Heavy Applications**: Content websites, analytics
- **Reporting Systems**: Separate read workloads
- **Backup Systems**: Continuous data backup

### Master-Master Replication

#### Architecture
- **Multiple Masters**: All servers can accept writes
- **Bidirectional Replication**: Changes propagate between all masters
- **Conflict Resolution**: Handle conflicting writes

#### How It Works
1. **Write Anywhere**: Writes can go to any master
2. **Conflict Detection**: Identify conflicting changes
3. **Resolution Logic**: Apply conflict resolution rules
4. **Propagation**: Changes replicated to all other masters

#### Advantages
- **High Availability**: No single point of failure for writes
- **Load Distribution**: Writes distributed across masters
- **Geographic Distribution**: Masters in different regions

#### Disadvantages
- **Conflict Resolution**: Complex conflict handling
- **Data Consistency**: Potential for data conflicts
- **Network Overhead**: More complex replication topology

#### Use Cases
- **Global Applications**: Users in multiple regions
- **High Availability Systems**: Zero downtime requirements
- **Collaborative Systems**: Multiple writers updating data

### Multi-Master Replication

#### Architecture
- **Multiple Masters**: More than two master servers
- **Complex Topology**: Various replication relationships
- **Advanced Conflict Resolution**: Sophisticated conflict handling

#### Variants
- **Circular Replication**: Masters form a replication circle
- **Star Replication**: Central master with multiple spokes
- **Mesh Replication**: Complex interconnected topology

#### Use Cases
- **Large-Scale Systems**: Massive user bases
- **Global Enterprises**: Worldwide data distribution
- **Real-Time Systems**: Low-latency global access

## Replication Technologies

### MySQL Replication
```sql
-- Configure master
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-do-db=myapp

-- Configure slave
[mysqld]
server-id=2
relay-log=relay-bin

-- Start replication on slave
CHANGE MASTER TO
  MASTER_HOST='master-host',
  MASTER_USER='replication-user',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=0;

START SLAVE;
```

### PostgreSQL Replication
```sql
-- Configure streaming replication
-- Master: postgresql.conf
wal_level = replica
max_wal_senders = 3
wal_keep_segments = 64

-- Slave: recovery.conf
standby_mode = 'on'
primary_conninfo = 'host=master-host port=5432 user=replication-user password=password'
restore_command = 'cp /var/lib/postgresql/archive/%f %p'
```

### MongoDB Replication
```javascript
// Initialize replica set
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb0.example.com:27017" },
    { _id: 1, host: "mongodb1.example.com:27017" },
    { _id: 2, host: "mongodb2.example.com:27017", arbiterOnly: true }
  ]
});

// Check replica set status
rs.status();
```

## Replication Methods

### Synchronous Replication
- **Definition**: Master waits for all replicas to confirm receipt before committing
- **Consistency**: Strong consistency across all nodes
- **Performance**: Higher latency, lower throughput
- **Durability**: Guaranteed durability on multiple nodes

**Pros:**
- Zero data loss on master failure
- Strong consistency guarantees
- Immediate consistency across replicas

**Cons:**
- Higher write latency
- Reduced write throughput
- Network issues block writes

### Asynchronous Replication
- **Definition**: Master commits immediately, replicas updated later
- **Consistency**: Eventual consistency
- **Performance**: Low latency, high throughput
- **Durability**: Master durability only

**Pros:**
- Fast writes, no network blocking
- High write throughput
- Better performance for distant replicas

**Cons:**
- Potential data loss on master failure
- Replication lag causes stale reads
- Complex conflict resolution

### Semi-Synchronous Replication
- **Definition**: Master waits for at least one replica to confirm before committing
- **Consistency**: Balance between sync and async
- **Performance**: Moderate latency and throughput

**Pros:**
- Better durability than async
- Faster than full sync
- Configurable consistency level

**Cons:**
- Still some performance impact
- More complex than pure async

## Replication Lag and Consistency

### Replication Lag
Time difference between when data is written to master and when it appears on replica.

#### Causes of Replication Lag
- **Network Latency**: Distance between servers
- **High Write Volume**: Master overwhelmed with writes
- **Large Transactions**: Big transactions take time to replicate
- **Disk I/O**: Slow disk operations on replicas

#### Measuring Replication Lag
```sql
-- MySQL: Check slave status
SHOW SLAVE STATUS\G

-- PostgreSQL: Check replication lag
SELECT
  client_addr,
  state,
  sent_lsn,
  write_lsn,
  flush_lsn,
  replay_lsn
FROM pg_stat_replication;

-- MongoDB: Check replication lag
rs.printSlaveReplicationInfo();
```

### Consistency Models

#### Strong Consistency
- **Reads always return latest writes**
- **Requires synchronous replication**
- **Higher latency, lower availability**

#### Eventual Consistency
- **Reads may return stale data temporarily**
- **Asynchronous replication**
- **Lower latency, higher availability**

#### Read-Your-Writes Consistency
- **User always sees their own writes**
- **Common in social media platforms**
- **Balance of consistency and performance**

#### Monotonic Read Consistency
- **Once user sees a value, they never see an older value**
- **Prevents "going backwards" in time**

## Failover and High Availability

### Automatic Failover
- **Health Monitoring**: Continuous monitoring of master health
- **Failure Detection**: Automatic detection of master failure
- **Replica Promotion**: Promote healthy replica to master
- **Client Redirection**: Update client connections

### Manual Failover
- **Planned Maintenance**: Controlled master switch
- **Configuration Changes**: Update during maintenance windows
- **Version Upgrades**: Rolling upgrades with minimal downtime

### Split-Brain Scenario
- **Problem**: Multiple masters think they're active
- **Causes**: Network partitions, slow failure detection
- **Prevention**: Quorum-based decisions, fencing mechanisms

## Replication Topologies

### Single Master, Multiple Slaves
```
Master ←─── Slave 1
        ├─── Slave 2
        └─── Slave 3
```

**Use Case:** Read scaling, high availability

### Master-Master with Slaves
```
Master 1 ─── Master 2
    │            │
    └── Slave 1   └── Slave 2
```

**Use Case:** Geographic distribution, load balancing

### Cascade Replication
```
Master ←── Slave 1 ←── Slave 2
                   └── Slave 3
```

**Use Case:** Reduce master load, hierarchical distribution

### Ring Replication
```
Master 1 → Master 2 → Master 3 → Master 1
```

**Use Case:** Multi-master scenarios, complex topologies

## Monitoring and Management

### Key Metrics to Monitor

#### Replication Health
- **Replication Lag**: Time behind master
- **Replication Status**: Running, stopped, error
- **Network Connectivity**: Connection between servers

#### Performance Metrics
- **Replication Throughput**: Data replicated per second
- **Apply Lag**: Time to apply changes on replica
- **Disk Usage**: Space used by replication logs

#### Error Metrics
- **Replication Errors**: Failed replications
- **Connection Failures**: Network issues
- **Data Corruption**: Checksum failures

### Monitoring Tools
```bash
# MySQL monitoring
mysql> SHOW SLAVE STATUS;
mysql> SHOW PROCESSLIST;

# PostgreSQL monitoring
postgres=# SELECT * FROM pg_stat_replication;
postgres=# SELECT * FROM pg_stat_wal_receiver;

# MongoDB monitoring
> rs.status()
> db.serverStatus().repl
```

### Alerting
- **Replication Lag Thresholds**: Alert when lag exceeds limits
- **Replication Stopped**: Alert when replication fails
- **Disk Space**: Alert when running low on space
- **Network Issues**: Alert on connection failures

## Replication in System Design

### Read Scaling Architecture
```
┌─────────────────┐
│   Load Balancer │
└─────────────────┘
          │
    ┌─────┴─────┐
    │          │
┌───▼───┐  ┌───▼───┐
│ Master│  │ Slave│
│ (Write)│  │ (Read)│
└───────┘  └───────┘
           │
      ┌────▼────┐
      │  Slave  │
      │  (Read) │
      └─────────┘
```

### Multi-Region Architecture
```
┌─────────────────┐    ┌─────────────────┐
│   Region 1      │    │   Region 2      │
│ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │  Master     │◄┼────┼►│  Slave      │ │
│ │             │ │    │ │             │ │
│ └─────────────┘ │    │ └─────────────┘ │
│       │         │    │       │         │
│ ┌─────▼─────┐   │    │ ┌─────▼─────┐   │
│ │  Slave    │   │    │ │  Slave    │   │
│ │           │   │    │ │           │   │
│ └───────────┘   │    │ └───────────┘   │
└─────────────────┘    └─────────────────┘
```

### Disaster Recovery Architecture
```
┌─────────────────┐    ┌─────────────────┐
│ Production      │    │ Disaster Recovery│
│ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │  Master     │◄┼────┼►│  Slave      │ │
│ │             │ │    │ │             │ │
│ └─────────────┘ │    │ └─────────────┘ │
│       │         │    │                 │
│ ┌─────▼─────┐   │    │                 │
│ │  Slave    │   │    │                 │
│ │           │   │    │                 │
│ └───────────┘   │    │                 │
└─────────────────┘    └─────────────────┘
```

## Replication Challenges

### Network Issues
**Challenge:** Network partitions, latency, bandwidth limitations
**Solutions:**
- Compression of replication traffic
- Efficient serialization formats
- Connection pooling and reuse

### Large Datasets
**Challenge:** Initial sync of large databases
**Solutions:**
- Incremental backups
- Logical replication
- Parallel sync processes

### Schema Changes
**Challenge:** Applying DDL changes across replicas
**Solutions:**
- Rolling schema updates
- Schema versioning
- Lock-free schema changes

### Monitoring Complexity
**Challenge:** Tracking replication across many servers
**Solutions:**
- Centralized monitoring
- Automated alerting
- Replication dashboards

## Best Practices

### Setup and Configuration
- **Test Replication**: Always test failover scenarios
- **Monitor Lag**: Set up alerts for replication lag
- **Secure Connections**: Use SSL/TLS for replication traffic
- **Document Topology**: Keep replication architecture documented

### Performance Optimization
- **Optimize Network**: Use fast, reliable network connections
- **Tune Parameters**: Adjust replication parameters for your workload
- **Monitor Resources**: Ensure replicas have adequate resources
- **Load Balancing**: Distribute read load across replicas

### Maintenance and Operations
- **Regular Backups**: Backup master and replicas separately
- **Version Compatibility**: Keep all servers on compatible versions
- **Security Updates**: Apply security patches promptly
- **Capacity Planning**: Monitor growth and plan for scaling

### Troubleshooting
- **Log Analysis**: Monitor replication logs for errors
- **Network Diagnostics**: Check connectivity and bandwidth
- **Performance Profiling**: Identify bottlenecks in replication
- **Failover Testing**: Regularly test failover procedures

## Replication in Java Applications

### Spring Boot with Multiple Data Sources
```java
@Configuration
public class ReplicationConfig {
    
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public DataSource routingDataSource(
            @Qualifier("masterDataSource") DataSource master,
            @Qualifier("slaveDataSource") DataSource slave) {
        
        ReplicationRoutingDataSource routingDataSource = new ReplicationRoutingDataSource();
        
        Map<Object, Object> dataSources = new HashMap<>();
        dataSources.put("master", master);
        dataSources.put("slave", slave);
        
        routingDataSource.setTargetDataSources(dataSources);
        routingDataSource.setDefaultTargetDataSource(master);
        
        return routingDataSource;
    }
}
```

### Read/Write Splitting
```java
@Service
public class RoutingService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Transactional(readOnly = true)
    public List<User> getUsers() {
        // This will use slave database
        return jdbcTemplate.query("SELECT * FROM users", 
            (rs, rowNum) -> new User(rs.getString("name")));
    }
    
    @Transactional
    public void createUser(String name) {
        // This will use master database
        jdbcTemplate.update("INSERT INTO users (name) VALUES (?)", name);
    }
}
```

## Conclusion

Database replication is essential for building scalable, reliable, and high-performance systems. It provides redundancy, improves read performance, and ensures data availability across failures.

**Key Takeaways:**
- **Master-Slave**: Simple, good for read scaling
- **Master-Master**: Complex, good for write distribution
- **Monitor Lag**: Critical for consistency and performance
- **Test Failover**: Regularly test disaster recovery
- **Choose Topology**: Based on read/write patterns and requirements

Understanding replication enables you to design systems that can handle growth, failures, and geographic distribution effectively.
