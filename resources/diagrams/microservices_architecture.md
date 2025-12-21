# Microservices Architecture Patterns

This document provides visual representations and detailed explanations of microservices architectural patterns, decomposition strategies, and communication patterns.

## 🏗 Microservices Decomposition Patterns

### Domain-Driven Design (DDD) Decomposition

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              E-commerce System                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Bounded Contexts:                                                           │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────┐  │
│  │  Order Context  │ │ Product Catalog │ │   User Mgmt    │ │  Payment   │  │
│  │                 │ │                 │ │                 │ │             │  │
│  │ • Order Service │ │ • Product Svc   │ │ • User Service │ │ • Payment Svc│  │
│  │ • Cart Service  │ │ • Inventory Svc │ │ • Auth Service │ │ • Gateway   │  │
│  │ • Shipping Svc  │ └─────────────────┘ └─────────────────┘ └─────────────┘  │
│  └─────────────────┘                                                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Shared Kernel:                                                                │
│  ┌─────────────────┐ ┌─────────────────┐                                       │
│  │   Notification  │ │    Logging     │                                       │
│  │    Service      │ │   Service      │                                       │
│  └─────────────────┘ └─────────────────┘                                       │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Business Capability Decomposition

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          Business Capabilities                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────┐  │
│  │   Front-end     │ │   Back-end      │ │   Analytics     │ │   Admin     │  │
│  │   Experience    │ │   Processing    │ │   & Reporting   │ │   Portal    │  │
│  │                 │ │                 │ │                 │ │             │  │
│  │ • Web App       │ │ • Order Mgmt    │ │ • Data Pipeline │ │ • CMS        │  │
│  │ • Mobile App    │ │ • Inventory     │ │ • BI Reports    │ │ • Monitoring │  │
│  │ • API Gateway   │ │ • Fulfillment   │ └─────────────────┘ └─────────────┘  │
│  └─────────────────┘ └─────────────────┘                                       │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 🔄 Service Communication Patterns

### Synchronous Communication (REST/GraphQL)

```
┌─────────────┐    HTTP/REST    ┌─────────────────┐
│   Client    │◄────────────────►│   API Gateway  │
└─────────────┘                 └─────────────────┘
                                   │         │
                    ┌──────────────┼─────────┼──────────────┐
                    │              │         │              │
                    ▼              ▼         ▼              ▼
            ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
            │Order Service│ │Product Svc  │ │User Service │ │Payment Svc  │
            └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
                    ▲              ▲         ▲              ▲
                    │              │         │              │
                    └──────────────┼─────────┼──────────────┘
                                   │         │
                    ┌──────────────┼─────────┼──────────────┐
                    │              │         │              │
                    ▼              ▼         ▼              ▼
            ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
            │  Database   │ │  Cache      │ │  Search     │ │  Message   │
            │  (MySQL)    │ │  (Redis)    │ │  (ES)       │ │  Queue     │
            └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
```

### Asynchronous Communication (Event-Driven)

```
┌─────────────┐    HTTP/Events    ┌─────────────────┐
│   Client    │◄─────────────────►│   API Gateway  │
└─────────────┘                   └─────────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
            ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
            │Order Service│ │Product Svc  │ │User Service │
            │             │ │             │ │             │
            │  Producer   │ │  Consumer   │ │  Producer   │
            └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
                   │               │               │
                   └───────────────┼───────────────┘
                                   │
                    ┌──────────────┼───────────────┐
                    │              │              │
                    ▼              ▼              ▼
            ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
            │ Event Bus   │ │Event Processor│ │Event Store  │
            │  (Kafka)    │ │ (Stream App)  │ │ (Database)  │
            └─────────────┘ └─────────────┘ └─────────────┘
                                   │
                    ┌──────────────┼───────────────┐
                    │              │              │
                    ▼              ▼              ▼
            ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
            │Notification │ │  Analytics  │ │  Search     │
            │   Service   │ │   Service   │ │   Service   │
            └─────────────┘ └─────────────┘ └─────────────┘
```

## 🏛 Service Architecture Patterns

### API Gateway Pattern

```
┌─────────────┐
│   Clients   │  (Web, Mobile, APIs)
│             │
│ • Browsers  │
│ • Mobile Apps│
│ • Third-party│
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌─────────────────────┐
│ API Gateway │ ◄──► │   Service Registry │
│             │     │                     │
│ • Routing   │     │ • Service Discovery │
│ • Auth      │     │ • Load Balancing    │
│ • Rate Limit│     │ • Health Checks     │
│ • Caching   │     └─────────────────────┘
│ • Transform │
└──────┬──────┘
       │
       ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Order Svc   │ │ Product Svc │ │ User Svc    │
│             │ │             │ │             │
│ Port: 8081  │ │ Port: 8082  │ │ Port: 8083  │
└─────────────┘ └─────────────┘ └─────────────┘
```

### Database per Service Pattern

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Microservices                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────┐  │
│  │  Order Service  │ │ Product Service │ │  User Service  │ │Payment Svc  │  │
│  │                 │ │                 │ │                 │ │             │  │
│  │ • Order DB      │ │ • Product DB    │ │ • User DB       │ │ • Payment DB│  │
│  │ • Order Events  │ │ • Product Events│ │ • User Events   │ │ • Payment   │  │
│  │ • Order Cache   │ │ • Product Cache │ │ • User Cache    │ │ • Events    │  │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────┘  │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Private Databases:                                                            │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────┐  │
│  │   MySQL Order   │ │  MongoDB Prod   │ │ PostgreSQL User │ │   Redis     │  │
│  │   Database      │ │   Database      │ │   Database      │ │   Payment   │  │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────┘  │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Event-Driven Communication:                                                   │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐                   │
│  │  Event Bus      │ │ Event Processor │ │  Read Models    │                   │
│  │   (Kafka)       │ │   (Streams)     │ │   (Views)       │                   │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 🔄 Saga Pattern for Distributed Transactions

### Choreography-Based Saga

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│Order Service│────►│Inventory Svc│────►│Payment Svc  │
│             │     │             │     │             │
│ 1. Create   │     │ 2. Reserve  │     │ 3. Charge   │
│    Order    │     │   Inventory │     │   Customer  │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                ┌──────────┼──────────┐
                │         ▼          │
                │ ┌─────────────┐    │
                │ │Notification │    │
                │ │   Service   │    │
                │ │             │    │
                │ │4. Send Email│    │
                │ └─────────────┘    │
                └────────────────────┘

Failure Scenarios:
- If Payment fails → Inventory Service releases reserved items
- If Inventory fails → Order Service cancels order
- All services listen for failure events and perform compensation
```

### Orchestration-Based Saga

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            Saga Orchestrator                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────┐  │
│  │   Step 1:       │ │   Step 2:       │ │   Step 3:       │ │   Step 4:   │  │
│  │ Create Order    │ │ Reserve Inv     │ │ Process Payment │ │ Send Notif  │  │
│  │                 │ │                 │ │                 │ │             │  │
│  │ Order Service   │ │ Inventory Svc   │ │ Payment Service │ │ Notif Svc   │  │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────┘  │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Compensation Actions:                                                         │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐                   │
│  │ Cancel Order    │ │ Release Inv     │ │ Refund Payment  │                   │
│  │ (Step 1 Fail)   │ │ (Step 2 Fail)   │ │ (Step 3 Fail)   │                   │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 🎯 Service Mesh Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            Service Mesh                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Data Plane:                                                                    │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────┐  │
│  │   Sidecar       │ │   Sidecar       │ │   Sidecar       │ │   Sidecar   │  │
│  │  Proxy (Envoy)  │ │  Proxy (Envoy)  │ │  Proxy (Envoy)  │ │  Proxy      │  │
│  │                 │ │                 │ │                 │ │             │  │
│  │ • Service A     │ │ • Service B     │ │ • Service C     │ │ • Service D │  │
│  │ • mTLS          │ │ • mTLS          │ │ • mTLS          │ │ • mTLS      │  │
│  │ • Observability │ │ • Observability │ │ • Observability │ │ • Observ     │  │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────┘  │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Control Plane:                                                                 │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐                   │
│  │   Istio         │ │   Linkerd       │ │   Consul        │                   │
│  │   Control       │ │   Control       │ │   Connect       │                   │
│  │   Plane         │ │   Plane         │ │                 │                   │
│  │                 │ │                 │ │ • Service       │                   │
│  │ • Traffic Mgmt  │ │ • Traffic Mgmt  │ │   Discovery     │                   │
│  │ • Security      │ │ • Security      │ │ • Configuration │                   │
│  │ • Observability │ │ • Observability │ │ • Segmentation  │                   │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 🏢 Strangler Fig Pattern (Migration)

```
Legacy Monolith → Microservices Migration

Phase 1: Initial State
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            Legacy Monolith                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │    Monolithic Application                                              │   │
│  │                                                                         │   │
│  │  • User Management • Order Processing • Payment • Inventory • Search   │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘

Phase 2: Strangler Fig (UI Layer)
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            Legacy Monolith                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │    Monolithic Application                                              │   │
│  │                                                                         │   │
│  │  • User Management • Order Processing • Payment • Inventory • Search   │   │
│  │                                                                         │   │
│  └─────────────────┬───────────────────────────────────────────────────────┘   │
│                   │                                                           │
│                   ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │   API Gateway / UI Layer (New)                                         │   │
│  │                                                                         │   │
│  │  Routes to Monolith or New Services                                    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘

Phase 3: Extract Services
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      Hybrid Architecture                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────┐  │
│  │  User Service   │ │ Order Service   │ │ Legacy Monolith │ │ API Gateway │  │
│  │   (New)         │ │   (New)         │ │   (Reduced)     │ │             │  │
│  │                 │ │                 │ │                 │ │ • Routing    │  │
│  │ • DB per Svc    │ │ • DB per Svc    │ │ • Payment       │ │ • Load Bal  │  │
│  │ • Event Driven  │ │ • Event Driven  │ │ • Inventory     │ │ • Auth       │  │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────┘  │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Event-Driven Communication:                                                   │
│  ┌─────────────────┐ ┌─────────────────┐                                       │
│  │  Event Bus      │ │ Event Processor │                                       │
│  │   (Kafka)       │ │   (Streams)     │                                       │
│  └─────────────────┘ └─────────────────┘                                       │
└─────────────────────────────────────────────────────────────────────────────────┘

Phase 4: Complete Migration
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          Full Microservices                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────┐  │
│  │  User Service   │ │ Order Service   │ │Payment Service  │ │Inventory Svc│  │
│  │                 │ │                 │ │                 │ │             │  │
│  │ • DB per Svc    │ │ • DB per Svc    │ │ • DB per Svc    │ │ • DB per Svc │  │
│  │ • Event Driven  │ │ • Event Driven  │ │ • Event Driven  │ │ • Event     │  │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────┘  │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Shared Infrastructure:                                                         │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐                   │
│  │  API Gateway    │ │  Service Mesh   │ │  Event Bus      │                   │
│  │                 │ │                 │ │                 │                   │
│  │ • Routing       │ │ • Observability │ │ • Kafka         │                   │
│  │ • Security      │ │ • Traffic Mgmt  │ │ • Event Store   │                   │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 📋 Key Design Considerations

### Service Boundaries
- **Domain-driven design** for service identification
- **Business capability** alignment
- **Data ownership** and consistency
- **Team ownership** and autonomy

### Communication Patterns
- **Synchronous** for request-response (REST/gRPC)
- **Asynchronous** for event-driven (Kafka/RabbitMQ)
- **Hybrid** approaches for complex workflows

### Data Management
- **Database per service** for loose coupling
- **Event sourcing** for audit trails
- **CQRS** for read/write optimization
- **Saga pattern** for distributed transactions

### Cross-Cutting Concerns
- **API Gateway** for external traffic
- **Service Mesh** for internal communication
- **Centralized logging** and monitoring
- **Distributed tracing** for observability

### Deployment Strategies
- **Blue-Green** deployments
- **Canary** releases
- **Rolling updates** with zero downtime
- **Feature flags** for gradual rollouts

---

**Microservices architecture enables scalable, maintainable systems through decomposition and loose coupling.** 🔄

*Choose the right patterns for your organization's needs and constraints.*
