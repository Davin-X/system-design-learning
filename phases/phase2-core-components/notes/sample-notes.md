# Phase 2 Notes - Core Components

## Date: [Today's Date]

## Resources Completed:
- [ ] DDIA Chapters 3-7
- [ ] Grokking the System Design Interview (Educative)
- [ ] Gaurav Sen's System Design playlist

## Key Concepts Learned:

### Databases
#### SQL Databases:
- ACID properties: Atomicity, Consistency, Isolation, Durability
- Normalization: 1NF, 2NF, 3NF to reduce redundancy
- Indexing: B-trees, hash indexes
- Joins, transactions

#### NoSQL Databases:
- Types: Document (MongoDB), Key-Value (Redis), Column-family (Cassandra), Graph
- BASE: Basically Available, Soft state, Eventual consistency
- When to use: High write loads, flexible schemas, big data

#### Sharding & Replication:
- Sharding: Horizontal partitioning, hash-based, range-based
- Replication: Master-slave, master-master, leader-follower
- Consistency models: Strong, eventual, causal

### Caching
- Types: In-memory (Redis, Memcached), CDN, Browser cache
- Strategies: Cache-aside, write-through, write-behind
- Eviction policies: LRU, LFU, TTL
- Cache invalidation problems: Stale data

### Load Balancing
- Algorithms: Round-robin, least connections, IP hash, weighted
- Hardware vs Software load balancers
- Reverse proxies: Nginx, HAProxy
- Health checks, session persistence

### Message Queues
- Purpose: Decoupling, async processing, buffering
- Examples: Kafka (pub-sub), RabbitMQ (message broker)
- Concepts: Producers, consumers, topics, partitions
- Use cases: Event-driven architecture, logging

### Other Components
- CDN: Content Delivery Networks for static assets
- DNS: Domain Name System, caching, load balancing
- APIs: REST (stateless), GraphQL (flexible queries)

## Java-Specific Notes:
- Spring Boot with JPA/Hibernate for SQL
- Spring Cache with Redis
- Spring Cloud for microservices/load balancing

## Questions:
- 

## Practice:
- Design e-commerce system diagram
- Implement simple cache in Java

## Next Steps:
- Complete Grokking course
- Move to Phase 3
