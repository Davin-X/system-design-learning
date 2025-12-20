# System Design Learning Curriculum

This curriculum provides a structured 12-week learning path based on the comprehensive guide. Each week focuses on specific topics with learning objectives, resources, and practical exercises.

## Week 1: Foundations & Architecture Basics
**Topics**: System Design Intro (HLD/LLD), Requirements, Architectural Styles

**Learning Objectives**:
- Understand difference between HLD and LLD
- Identify functional vs non-functional requirements
- Compare monolithic vs microservices architectures
- Explain event-driven and serverless patterns

**Resources**:
- Guide sections: Basics
- Video: "System Design for Beginners" (freeCodeCamp)
- Article: "Monolithic vs Microservices" (Martin Fowler)

**Exercises**:
1. Write functional/non-functional requirements for a simple e-commerce app
2. Draw architecture diagrams for monolithic and microservices versions
3. Explain when to choose each architectural style

**Milestone**: Create architecture diagram for a familiar application

## Week 2: Scalability Fundamentals
**Topics**: Horizontal/Vertical Scaling, Bottlenecks

**Learning Objectives**:
- Explain scaling approaches and trade-offs
- Identify common scalability bottlenecks
- Understand scaling strategies for different application types

**Resources**:
- Guide section: Scalability
- Article: "Scalability Best Practices" (AWS)

**Exercises**:
1. Analyze scalability needs for different app types (social media vs banking)
2. Identify bottlenecks in a given system description
3. Propose scaling solutions for a growing application

**Milestone**: Write a scaling strategy for a hypothetical app

## Week 3: Databases Deep Dive
**Topics**: SQL/NoSQL, Replication, Sharding, Storage Types

**Learning Objectives**:
- Choose appropriate database for different use cases
- Understand replication and sharding concepts
- Explain different storage types and their uses

**Resources**:
- Guide section: Databases in Designing Systems
- Book: "Designing Data-Intensive Applications" Chapters 3-5

**Exercises**:
1. Design database schema for a social media platform (SQL vs NoSQL)
2. Explain sharding strategy for a large e-commerce database
3. Compare block, file, and object storage for different scenarios

**Milestone**: Design complete data architecture for a complex application

## Week 4: System Guarantees
**Topics**: CAP Theorem, Consistency, Availability, Reliability

**Learning Objectives**:
- Explain CAP theorem trade-offs
- Understand different consistency models
- Design for high availability and fault tolerance

**Resources**:
- Guide section: Consistency, Availability, Reliability & Maintainability
- Article: "CAP Theorem" (Wikipedia + examples)

**Exercises**:
1. Analyze CAP theorem implications for real systems (Netflix, banking)
2. Design fault tolerance mechanisms for a distributed system
3. Explain consistency patterns and their use cases

**Milestone**: Create availability and reliability plan for a critical system

## Week 5: Performance & Communication
**Topics**: Load Balancing, Latency/Throughput, Caching, APIs

**Learning Objectives**:
- Understand load balancing algorithms and consistent hashing
- Explain caching strategies and patterns
- Design API gateways and rate limiting

**Resources**:
- Guide sections: Load Balancing, Latency/Throughput and Caching, API Gateway
- Article: "Load Balancing Algorithms"

**Exercises**:
1. Implement simple load balancing algorithm in code
2. Design multi-level caching strategy for a high-traffic site
3. Create API gateway configuration with rate limiting

**Milestone**: Design complete request flow from client to database with all components

## Week 6: Advanced Communication & Infrastructure
**Topics**: Message Queues, Protocols, CDN, Proxies, WebSockets

**Learning Objectives**:
- Understand asynchronous communication patterns
- Explain network protocols and optimization
- Design content delivery and proxy architectures

**Resources**:
- Guide sections: API Gateway/Message Queues, Protocols/CDN/Proxies/WebSockets

**Exercises**:
1. Design message queue architecture for event-driven system
2. Explain CDN benefits and implementation
3. Create WebSocket implementation plan for real-time features

**Milestone**: Design communication architecture for a real-time application

## Week 7: Security, Testing & DevOps
**Topics**: Security Measures, Testing, CI/CD

**Learning Objectives**:
- Understand security principles and authentication/authorization
- Explain testing strategies and automation
- Design secure development lifecycle

**Resources**:
- Guide sections: Security Measures, Testing

**Exercises**:
1. Design authentication/authorization system
2. Create security threat model for an application
3. Set up CI/CD pipeline for a microservices application

**Milestone**: Create security and testing strategy document

## Week 8: Distributed Systems & Performance
**Topics**: Consensus Algorithms, Distributed Tracing, Cost Optimization

**Learning Objectives**:
- Understand distributed consensus and tracing
- Optimize for performance and cost
- Design distributed system communication

**Resources**:
- Guide sections: Distributed System Design, Cost & Performance Optimizations

**Exercises**:
1. Explain Paxos/Raft consensus algorithms
2. Design distributed tracing for microservices
3. Create cost optimization strategy for cloud deployment

**Milestone**: Design distributed system architecture with all considerations

## Week 9-10: Low Level Design Fundamentals
**Topics**: OOP, Design Principles, UML, Design Patterns

**Learning Objectives**:
- Master object-oriented programming concepts
- Apply SOLID principles and design patterns
- Create UML diagrams for system design

**Resources**:
- Guide section: Low Level Design(LLD)

**Exercises**:
1. Refactor code to follow SOLID principles
2. Identify and apply appropriate design patterns
3. Create class, sequence, and use case diagrams

**Milestone**: Design complete LLD for a system component

## Week 11: Interview Preparation - System Designs
**Topics**: URL Shortener, Dropbox, Twitter, Netflix, Uber, etc.

**Learning Objectives**:
- Practice end-to-end system design
- Handle scaling and trade-off discussions
- Communicate design decisions clearly

**Resources**:
- Guide section: Interview Questions & Answers

**Exercises**:
1. Design URL shortening service with all components
2. Create Twitter architecture with scaling considerations
3. Design Netflix-like streaming platform

**Milestone**: Successfully design 3-4 complex systems from scratch

## Week 12: Advanced Interview Topics & Review
**Topics**: Interview Tips, Common Concepts, Object-Oriented Design

**Learning Objectives**:
- Master interview approaches and tips
- Review all major concepts
- Practice object-oriented design questions

**Resources**:
- Guide sections: Tips for System Design interview, 6 Steps to Approach OOD

**Exercises**:
1. Practice mock interviews with timing
2. Review and strengthen weak areas
3. Create personal system design cheat sheet

**Milestone**: Confident in system design interviews and real-world application

## Daily Learning Structure
- **Morning**: Read theory from guide (1-2 hours)
- **Afternoon**: Watch videos/practice exercises (2-3 hours)
- **Evening**: Hands-on coding/design (1-2 hours)
- **Weekly**: Review progress, update repository

## Assessment Methods
- Self-assessment checklists for each week
- Practical design exercises
- Code implementations where applicable
- Mock interview practice

## Resources Throughout
- **Primary**: The comprehensive guide in this repo
- **Practice**: LeetCode system design problems
- **Videos**: Gaurav Sen, freeCodeCamp, Tech Dummies Narendra
- **Books**: DDIA, System Design Interview
- **Communities**: Reddit r/systems, LinkedIn groups

## Tracking Progress
- Update this curriculum with completion dates
- Add personal notes and challenges faced
- Share progress on GitHub for accountability

Start with Week 1 and progress systematically. Adjust timeline based on your pace and experience level.
