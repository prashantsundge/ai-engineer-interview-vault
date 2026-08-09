## System Design

### Introduction

### What is System Design?

### How to approach System Design?

### Performance vs Scalability

### Latency vs Throughput

### Availability vs Consistency

### CAP Theorem
- AP - Availability + Partition Tolerance
- CP - Consistency + Partition Tolerance

### Consistency Patterns
- Weak Consistency
- Eventual Consistency
- Strong Consistency

### Availability Patterns
- Fail-Over
- Active - Active
- Active - Passive

### Replication
- Master - Slave
- Master - Master

### Availability in Numbers
- 99.9% Availability - three 9s
- 99.99% Availability - four 9s

### Availability in Parallel vs Sequence

### Background Jobs

### Event-Driven

### Schedule Driven

### Returning Results

### Domain Name System

### Content Delivery Networks
- Push CDNs
- Pull CDNs

### Load Balancers
- LB vs Reverse Proxy
- Load Balancing Algorithms
- Layer 7 Load Balancing
- Layer 4 Load Balancing

### Horizontal Scaling

### Application Layer

### Microservices

### Service Discovery

### Databases
- SQL vs NoSQL
- Replication
- Sharding
- Federation
- Denormalization
- SQL Tuning
- RDBMS
- Key-Value Store
- Document Store
- Wide Column Store
- Graph Databases
- NoSQL

### Caching
- Refresh Ahead
- Write-behind
- Write-through
- Cache Aside

### Strategies
- Client Caching
- CDN Caching
- Web Server Caching
- Database Caching
- Application Caching

### Asynchronism

### Back Pressure

### Task Queues

### Message Queues

### Types of Caching

### Idempotent Operations

### Communication
- HTTP
- TCP
- UDP
- RPC
- REST
- gRPC
- GraphQL

### Performance Antipatterns
- Busy Database
- Busy Frontend
- Chatty I/O
- Extraneous Fetching
- Improper Instantiation
- Monolithic Persistence
- No Caching
- Noisy Neighbor
- Retry Storm
- Synchronous I/O

### Monitoring
- Health Monitoring
- Availability Monitoring
- Performance Monitoring
- Security Monitoring
- Usage Monitoring
- Instrumentation
- Visualization & Alerts

The design patterns given in this section are of varying importance. You don't need to master all of them. Simply get an overview of each and this will give you some insight into designing scalable systems.

### Cloud Design Patterns
#### Messaging
- Sequential Convoy
- Scheduling Agent Supervisor
- Queue-based Load Leveling
- Publisher/Subscriber
- Priority Queue
- Pipes and Filters
- Competing Consumers
- Choreography
- Claim Check
- Async Request Reply

#### Data Management
- Valet Key
- Static Content Hosting
- Sharding
- Materialized View
- Index Table
- Event Sourcing
- CQRS
- Cache-Aside

### Design & Implementation
- Strangler Fig
- Static Content Hosting
- Sidecar
- Pipes & Filters
- Leader Election
- Gateway Routing
- Gateway Offloading
- Gateway Aggregation
- External Config Store
- Compute Resource Consolidation
- CQRS
- Backends for Frontend
- Anti-Corruption Layer
- Ambassador

### Reliability Patterns
- Availability
- Deployment Stamps
- Geodes
- Health Endpoint Monitoring
- Queue-Based Load Leveling
- Throttling
- High Availability
- Bulkhead
- Circuit Breaker
- Resiliency
- Compensating Transaction
- Leader Election
- Retry
- Scheduler Agent Supervisor

### Security
- Federated Identity
- Gatekeeper
- Valet Key

Visit the following relevant tracks to learn more:
- Backend
- Software Architect
- DevOps
- Other Roadmaps
- Backend Roadmap
- DevOps Roadmap
- Software Design and Architecture

Find the detailed version of this roadmap along with other similar roadmaps at [roadmap.sh](https://roadmap.sh).