# Detailed System Design Explanations

This document provides in-depth explanations for key system design concepts.

## High Level Design (HLD) vs Low Level Design (LLD)

### High Level Design (HLD)
HLD is the architectural blueprint of the system. It answers "what" and "how" at a macro level:

**Key Components of HLD:**
1. System Architecture Diagram: Visual representation of major components
2. Component Identification: Breaking down into logical modules
3. Data Flow: How data moves between components
4. Technology Stack: Choosing languages, frameworks, databases
5. Deployment Architecture: How components are deployed
6. Scalability Considerations: How system handles growth
7. Security Overview: High-level security measures

### Low Level Design (LLD)
LLD focuses on implementation details within components. It answers "how" at a micro level:

**Key Components of LLD:**
1. Class Diagrams: Detailed class structures and relationships
2. Database Schema: Tables, relationships, indexes
3. API Specifications: REST endpoints, request/response formats
4. Algorithm Design: Specific algorithms for operations
5. Data Structures: Choosing appropriate structures
6. Interface Design: Contracts between components
7. Error Handling: Exception handling and recovery

## Functional vs Non-Functional Requirements

### Functional Requirements
Specify what the system should do - features and capabilities.

**Examples:**
- User can create account with email and password
- System allows posting text and images
- Users receive notifications for followed posts

### Non-Functional Requirements
Specify how the system should perform - quality attributes.

**Categories:**
- Performance: Response time < 200ms, throughput > 1000 req/sec
- Scalability: Support 1M concurrent users
- Availability: 99.9% uptime
- Security: Data encrypted, GDPR compliance
- Reliability: MTBF > 30 days

## CAP Theorem

States that in distributed systems, you can only guarantee 2 out of 3 properties:
- Consistency: All nodes see same data simultaneously
- Availability: System operational despite failures  
- Partition Tolerance: System continues despite network failures

**CP Systems**: Prioritize consistency (banking)
**AP Systems**: Prioritize availability (social media)
**CA Systems**: Rare, assume no partitions

## Load Balancing Algorithms

- **Round Robin**: Sequential distribution, simple but ignores load
- **Least Connections**: Route to server with fewest connections
- **IP Hash**: Same IP always goes to same server (session persistence)
- **Weighted**: Different capacities get proportional load

## Caching Strategies

- **Cache-aside**: Check cache first, then DB if miss
- **Write-through**: Write to cache and DB simultaneously
- **Write-behind**: Write to cache first, DB asynchronously

## Database Sharding

Horizontal partitioning of data across servers for scalability.

**Strategies:**
- Range-based: Data by value ranges
- Hash-based: Hash key determines server
- Directory-based: Lookup table maps keys to servers

## SOLID Principles

- **S**: Single Responsibility - one reason to change
- **O**: Open/Closed - open for extension, closed for modification  
- **L**: Liskov Substitution - subtypes substitutable for base types
- **I**: Interface Segregation - no forced dependencies on unused interfaces
- **D**: Dependency Inversion - depend on abstractions, not concretions

## Design Patterns

### Creational Patterns
- **Singleton**: One instance, global access
- **Factory Method**: Interface for creating objects, subclasses decide
- **Abstract Factory**: Creates families of related objects

### Structural Patterns  
- **Adapter**: Allows incompatible interfaces to work together
- **Decorator**: Adds responsibilities dynamically
- **Proxy**: Provides placeholder for another object

### Behavioral Patterns
- **Observer**: One-to-many dependency, automatic notifications
- **Strategy**: Family of algorithms, interchangeable
- **Command**: Encapsulates request as object

## URL Shortening Service Design

**Requirements:**
- Shorten URLs, redirect, analytics
- High availability, low latency, scalability

**Architecture:**
- API Gateway (rate limiting, auth)
- URL Service (generate codes, store mappings)
- Redirect Service (handle redirects, analytics)
- Database + Cache (Redis)
- Load Balancer

**Key Decisions:**
- Base62 encoding for short codes
- Database sharding by hash
- Cache for fast lookups
- Async analytics processing
