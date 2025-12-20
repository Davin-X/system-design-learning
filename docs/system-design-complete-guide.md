# Complete System Design Learning Guide

This comprehensive guide covers all essential topics for system design, from basics to advanced concepts, including low-level design and interview preparation.

## Basics

### System Design Introduction - HLD & LLD
- **High Level Design (HLD)**: Overall system architecture, major components, data flow, technology stack
- **Low Level Design (LLD)**: Detailed design of individual components, classes, interfaces, algorithms

### Functional and Non Functional Requirements
- **Functional Requirements**: What the system should do (features, capabilities)
- **Non-Functional Requirements**: How the system should perform (scalability, reliability, security, performance)

### High Level Design
- System architecture overview
- Component identification and relationships
- Technology choices
- Data flow diagrams

### System Architectural Styles

#### Monolithic Architecture
- Single deployable unit containing all functionality
- Pros: Simple development, testing, deployment
- Cons: Scalability issues, tight coupling, technology lock-in

#### Microservices
- Decomposed into small, independent services
- Pros: Scalability, technology diversity, fault isolation
- Cons: Complexity, distributed system challenges

#### Monolithic vs Microservices Architecture
- When to choose monolithic: Small team, simple domain, MVP
- When to choose microservices: Large scale, complex domain, frequent deployments

#### Event-Driven Architecture
- Components communicate through events
- Loose coupling, asynchronous processing
- Examples: Message queues, pub/sub patterns

#### Serverless Architecture
- No server management, pay-per-use
- Functions as a Service (FaaS)
- Pros: Cost-effective, auto-scaling
- Cons: Cold starts, vendor lock-in

#### Stateful vs. Stateless Architecture
- **Stateless**: No client session data stored on server
- **Stateful**: Server maintains client session state
- Stateless preferred for scalability

#### Pub/Sub Architecture
- Publishers send messages to topics
- Subscribers receive messages from topics
- Decoupling of producers and consumers

## Scalability

### Horizontal and Vertical Scaling
- **Vertical Scaling**: Adding more resources to single machine (CPU, RAM, storage)
- **Horizontal Scaling**: Adding more machines/instances

### Which Scalability approach is right for our Application?
- Vertical: Simpler, good for databases with complex queries
- Horizontal: Better for web applications, microservices

### Primary Bottlenecks that Hurt the Scalability of an Application
- Database bottlenecks (slow queries, locks)
- Network latency
- Memory/CPU constraints
- Single points of failure

## Databases in Designing Systems

### Choosing a Database - SQL or NoSQL
- **SQL**: Structured data, complex queries, ACID transactions
- **NoSQL**: Flexible schemas, high scalability, eventual consistency
- Hybrid approaches

### File and Database Storage Systems
- File systems: Local storage, NAS, SAN
- Database systems: RDBMS, NoSQL, NewSQL

### Database Replication in System Design
- Master-slave replication
- Master-master replication
- Synchronous vs asynchronous replication
- Read replicas for scalability

### Database Sharding
- Horizontal partitioning of data across multiple servers
- Sharding keys and strategies
- Cross-shard queries challenges

### Block, Object, and File Storage
- **Block Storage**: Raw storage volumes (SAN, iSCSI)
- **File Storage**: Hierarchical file systems (NFS, SMB)
- **Object Storage**: Flat namespace with metadata (S3, GCS)

### Normalization Process in DBMS
- 1NF: Eliminate repeating groups
- 2NF: Remove partial dependencies
- 3NF: Remove transitive dependencies
- Higher normal forms for special cases

### SQL Query Optimization
- Indexing strategies
- Query execution plans
- Avoiding full table scans
- Join optimization

### Denormalization in Databases
- Adding redundant data to improve read performance
- Trade-offs: Update complexity vs read speed

### Intro to Redis
- In-memory data structure store
- Use cases: Caching, session storage, pub/sub
- Data types: Strings, hashes, lists, sets, sorted sets

## Consistency, Availability, Reliability & Maintainability

### Availability in System Design
- Percentage of uptime
- 99.9% (8.76 hours downtime/year) vs 99.99% (52.56 minutes)

### How to achieve High Availability?
- Redundancy, failover, load balancing
- Multi-region deployments
- Circuit breakers, retries

### Consistency in System Design
- Data consistency across distributed systems
- Strong vs eventual consistency

### Consistency pattern
- Read-after-write consistency
- Monotonic reads, consistent prefix
- Implementing consistency with versioning, timestamps

### CAP Theorem
- Consistency, Availability, Partition tolerance
- Can only achieve 2 out of 3 in distributed systems
- BASE vs ACID

### Reliability in System Design
- System performs correctly under expected conditions
- Mean Time Between Failures (MTBF)
- Mean Time To Recovery (MTTR)

### Fault Tolerance in System Design
- Graceful handling of failures
- Retry mechanisms, circuit breakers
- Bulkheads, timeouts

### Maintainability
- Code readability, modularity
- Automated testing, CI/CD
- Documentation, monitoring

## Load Balancing

### Concurrency and Parallelism
- Concurrency: Dealing with multiple tasks (interleaving)
- Parallelism: Executing multiple tasks simultaneously

### Load Balancer
- Distributes incoming traffic across multiple servers
- Types: Hardware, software (Nginx, HAProxy)

### Load Balancing Algorithms
- Round-robin, least connections, IP hash
- Weighted algorithms, least response time

### Consistent Hashing
- Distributes data across nodes
- Minimizes data movement when nodes are added/removed
- Used in distributed caches, databases

## Latency, Throughput and Caching

### Latency and Throughput
- **Latency**: Time for single operation
- **Throughput**: Operations per unit time
- Trade-offs in optimization

### Caching in System Design
- Reduces latency, increases throughput
- Cache hit ratio, cache invalidation
- Cache-aside, write-through, write-behind patterns

## API Gateway, Message Queues & Rate Limiting

### What is API Gateway
- Single entry point for APIs
- Authentication, rate limiting, routing, transformation

### Message Queues
- Asynchronous communication
- Buffering, decoupling producers and consumers
- Examples: Kafka, RabbitMQ, SQS

### Rate Limiting
- Controlling request rates
- Prevents abuse, ensures fair usage

### Rate Limiting Algorithm
- Token bucket, leaky bucket, fixed window, sliding window

## Protocols, CDN, Proxies & WebSockets

### Communication Protocols
- HTTP/HTTPS, TCP/UDP, WebSockets
- REST, GraphQL, gRPC

### Domain Name System
- Translates domain names to IP addresses
- Hierarchical structure

### DNS Caching
- Local DNS resolver caching
- TTL (Time To Live) for cache expiration

### Time to Live(TTL)
- Cache expiration time
- Balancing freshness vs performance

### Content Delivery Network(CDN)
- Distributed network of servers
- Caches static content closer to users
- Reduces latency, offloads origin servers

### Proxies in System Design
- Intermediaries between clients and servers

### Forward Proxy vs Reverse Proxy
- **Forward Proxy**: Client-side, hides client identity
- **Reverse Proxy**: Server-side, load balancing, caching

### Websockets
- Full-duplex communication over single TCP connection
- Real-time applications (chat, gaming, live updates)

## Testing

### Unit Testing
- Testing individual components in isolation
- Mocking dependencies

### Integration Testing
- Testing interactions between components
- End-to-end testing

### CI/CD Pipeline
- Continuous Integration: Automated building/testing
- Continuous Deployment: Automated deployment
- Tools: Jenkins, GitHub Actions, GitLab CI

## Security Measures

### Security Measures in System Design
- Defense in depth
- Least privilege principle
- Secure coding practices

### Authentication and Authorization
- **Authentication**: Verifying identity (OAuth, JWT)
- **Authorization**: Granting permissions (RBAC, ABAC)

### Secure Socket Layer (SSL) and Transport Layer Security (TLS)
- Encrypting data in transit
- Certificate authorities, handshake process

### Secure Software Developement Life Cycle (SSDLC)
- Security considerations throughout development
- Threat modeling, code reviews, penetration testing

### Data Backup and Disaster Recovery
- Regular backups, backup strategies
- Recovery Time Objective (RTO), Recovery Point Objective (RPO)

## Distributed System Design

### Consensus Algorithms in Distributed System
- Achieving agreement in distributed systems
- Paxos, Raft, Zab algorithms

### Distributed Tracing
- Tracking requests across microservices
- Tools: Jaeger, Zipkin

### Secure Communication in Distributed System
- Mutual TLS, service mesh (Istio)
- Secrets management

## Cost & Performance Optimizations

### Software Cost Estimation
- Estimating development effort and cost
- Function points, COCOMO model

### Performance Optimization Techniques
- Profiling, benchmarking
- Database optimization, caching, async processing
- Horizontal/vertical scaling

## Low Level Design(LLD)

## Core Concepts

### Object-Oriented Programing(OOP) Concepts
- Encapsulation, inheritance, polymorphism, abstraction
- SOLID principles

### Modularity and Interfaces
- Breaking down systems into modules
- Interface segregation

### What is Low Level Design or LLD
- Detailed design of classes, methods, data structures
- UML diagrams, design patterns

## Design Principles

### SOLID Principles
- **S**: Single Responsibility
- **O**: Open/Closed
- **L**: Liskov Substitution
- **I**: Interface Segregation
- **D**: Dependency Inversion

### DRY Principle
- Don't Repeat Yourself
- Code reusability

### KISS Principle
- Keep It Simple, Stupid
- Simplicity over complexity

### YAGNI Principle
- You Aren't Gonna Need It
- Avoid over-engineering

## UML

### Unified Modeling Language (UML)
- Standard for modeling software systems
- Class diagrams, sequence diagrams, use case diagrams

## Design Patterns

### Design Patterns
- Reusable solutions to common problems
- Creational, structural, behavioral patterns

### Singleton Pattern
- Ensures single instance of a class
- Global access point

### Factory Method
- Creates objects without specifying exact classes
- Subclass decides which class to instantiate

### Abstract Factory
- Creates families of related objects
- Interface for creating families of related objects

### Builder Pattern
- Constructs complex objects step by step
- Separates construction from representation

### Prototype Pattern
- Creates objects by copying existing objects
- Avoids costly creation

### Adapter Pattern
- Allows incompatible interfaces to work together
- Wrapper pattern

### Decorator Pattern
- Adds behavior to objects dynamically
- Alternative to subclassing

### Composite Pattern
- Treats individual objects and compositions uniformly
- Tree structure

### Proxy Pattern
- Provides placeholder for another object
- Controls access to original object

### Facade Pattern
- Provides unified interface to subsystem
- Simplifies complex interfaces

### Observer Pattern
- Defines dependency between objects
- Notifies observers of state changes

### Strategy Pattern
- Defines family of algorithms
- Encapsulates algorithms, makes them interchangeable

### Command Pattern
- Encapsulates request as object
- Allows parameterization of clients with queues, logging

### State Pattern
- Allows object to alter behavior when internal state changes
- State-specific behavior

### Template Method Pattern
- Defines skeleton of algorithm
- Subclasses override specific steps

## Interview Questions & Answers of System Design

### URL Shortening Service
- Requirements: Shorten URLs, redirect, analytics
- Components: API, database, cache, load balancer
- Design considerations: Collision handling, expiration

### Design Dropbox
- File storage and synchronization
- Components: Client, server, database, CDN
- Challenges: Conflict resolution, offline sync

### Design Twitter
- Microblogging platform
- Timeline generation, tweet storage, user relationships
- Scaling: Sharding, caching, message queues

### System Design Netflix – Complete Architecture
- Video streaming service
- CDN, microservices, recommendation engine
- Global distribution, adaptive bitrate streaming

### System Design of Uber App – Uber System Architecture
- Ride-hailing platform
- Matching algorithm, real-time tracking, payments
- Surge pricing, driver availability

### Design BookMyShow
- Movie ticket booking system
- Seat selection, payment processing, notifications
- Concurrency handling for seat booking

### Designing Facebook Messenger
- Real-time messaging
- WebSockets, message persistence, push notifications
- Scaling: Sharding, caching

### Designing Whatsapp Messenger
- End-to-end encrypted messaging
- Message delivery, group chats, media sharing
- Offline messaging, battery optimization

### Designing Instagram
- Photo/video sharing platform
- Feed generation, likes/comments, stories
- Image processing, content moderation

### Designing Airbnb
- Accommodation booking platform
- Search, booking, reviews, payments
- Recommendation engine, fraud detection

### System Designing of Airline Management System
- Flight booking, seat allocation, schedule management
- Integration with third-party systems
- High availability requirements

### Common Design Interview Questions
- Design a parking lot, elevator system, etc.
- Focus on requirements gathering, constraints, trade-offs

### Tips for System Design interview
- Clarify requirements
- Start with high-level design
- Discuss trade-offs
- Consider scale and constraints

### How to Crack System Design Round in Interviews?
- Practice common problems
- Learn design patterns and principles
- Study real-world architectures
- Communicate clearly

### 5 Tips to Crack Low-Level System Design Interviews
- Understand OOP and design principles
- Practice UML diagrams
- Know common design patterns
- Think about extensibility
- Consider edge cases

### 5 Common System Design Concepts for Interview Preparation
- Scalability, consistency, availability
- Caching, load balancing, databases
- Microservices, APIs, security

### 6 Steps To Approach Object-Oriented Design Questions in Interview
1. Understand requirements
2. Identify classes and relationships
3. Create class diagrams
4. Define interfaces
5. Consider design patterns
6. Discuss trade-offs
