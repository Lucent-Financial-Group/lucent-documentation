# ADR-001: Event Sourcing Architecture with KurrentDB

**Status:** Proposed  
**Date:** 2025-09-02  
**Context:** Lucent Services Decentralized Event-Driven Architecture

## Context

We are implementing a decentralized event sourcing architecture for Lucent Services that "turns the database inside out," using event streams as the sole source of truth across distributed nodes. This architecture must support:

- Parallel agentic processing for AI-driven yield farming analysis
- Real-time crypto market event processing and recalculation triggers
- Offline-first operation with eventual consistency
- Cryptographic identity and zero-trust security
- Multi-layer decentralization (edge → cluster → site → global)

## Decision

We will use **KurrentDB (EventStoreDB)** as our primary event store for domain events, implementing a comprehensive event sourcing pattern with the following characteristics:

### Event Store Architecture

**KurrentDB as Domain Event Store:**
- All business domain events (user actions, system state changes, AI agent decisions) stored in KurrentDB
- Append-only immutable event streams with strong consistency guarantees
- Built-in projections for materialized views and read models
- Native support for event versioning and stream metadata
- Persistent subscriptions for real-time event processing

**Event Schema Design:**
```json
{
  "eventId": "uuid",
  "eventType": "YieldOpportunityDetected | PositionCalculated | AgentDecisionMade",
  "streamId": "user-123 | agent-456 | market-eth-usdc",
  "eventNumber": 42,
  "timestamp": "2025-09-02T10:30:00Z",
  "metadata": {
    "causationId": "uuid",
    "correlationId": "uuid", 
    "agentId": "crypto-yield-agent-001",
    "signature": "cryptographic-signature"
  },
  "data": {
    "protocol": "aave",
    "yield": 8.5,
    "confidence": 0.92,
    "action": "enter_position"
  }
}
```

### Multi-Layer Event Propagation

**Hierarchical Event Streams:**
1. **Node Level:** Local KurrentDB instance per edge device/server
2. **Cluster Level:** KurrentDB cluster aggregating node streams
3. **Site Level:** Regional KurrentDB collecting cluster events
4. **Global Level:** Master KurrentDB with worldwide event consolidation

**Stream Categories:**
- `domain-events`: Business logic events (user actions, calculations)
- `agent-events`: AI agent decisions and processing results  
- `system-events`: Infrastructure and operational events
- `audit-events`: Security and compliance tracking

### Event Processing Pipeline

**Pure Event Handlers:**
- Deterministic functions taking events as input, producing new events as output
- No side effects within handlers (functional core, imperative shell pattern)
- Language-agnostic interface for multi-runtime support
- Handler registry mapping event types to processing functions

**Projection Management:**
- Real-time materialized views updated via KurrentDB projections
- Current state caches for fast reads
- Snapshot strategies for large aggregates
- Time-travel debugging capabilities

## Technical Implementation

### KurrentDB Configuration

**Clustering Setup:**
```yaml
EventStore:
  ClusterSize: 3
  NodePriority: 1
  HttpPort: 2113
  TcpPort: 1113
  EnableAtomPubOverHttp: true
  Projections:
    RunProjections: All
    ProjectionThreads: 4
```

**Security Configuration:**
- TLS encryption for all connections
- Certificate-based authentication
- Role-based access control for streams
- Event signing with cryptographic identities

**Stream Policies:**
- Retention policies per stream category
- Backup and archival strategies
- Cross-region replication configuration

### Event Handler Framework

**Handler Registration:**
```typescript
interface EventHandler<T> {
  eventType: string;
  handle(event: DomainEvent<T>): Promise<DomainEvent[]>;
}

class YieldOpportunityHandler implements EventHandler<YieldData> {
  async handle(event: DomainEvent<YieldData>): Promise<DomainEvent[]> {
    // Pure business logic - no side effects
    const analysis = analyzeYield(event.data);
    
    if (analysis.isViable) {
      return [new PositionRecommendationEvent(analysis)];
    }
    
    return [new OpportunityRejectedEvent(analysis.reason)];
  }
}
```

**Event Processing Engine:**
- Persistent subscriptions to relevant streams
- Concurrent processing with ordering guarantees per aggregate
- Retry policies and dead letter queues
- Metrics and observability integration

### Integration Points

**With Market Data (Kafka):**
- Market events trigger domain event recalculations
- Bridge service consuming Kafka topics, producing KurrentDB events
- Correlation between market updates and portfolio recalculations

**With Message Bus (RabbitMQ):**
- Command/response patterns for synchronous operations
- Event notifications to external systems
- Integration events for bounded context communication

## Consequences

### Benefits

**Auditability & Compliance:**
- Complete audit trail of all business decisions
- Regulatory compliance through immutable event history
- AI decision transparency and explainability

**Scalability & Performance:**
- Horizontal scaling through stream partitioning
- Read scaling via multiple projection instances
- Write scaling through stream sharding

**Resilience & Recovery:**
- Point-in-time recovery to any previous state
- Disaster recovery through event replay
- Graceful degradation during network partitions

**AI Integration:**
- Rich event history for machine learning training
- Real-time event streams for AI agent decision making
- Parallel processing of multiple agent strategies

### Challenges

**Complexity:**
- Event schema evolution and versioning
- Projection consistency and synchronization
- Distributed transaction coordination across layers

**Operational Overhead:**
- KurrentDB cluster management and monitoring
- Event store sizing and performance tuning
- Backup and archival procedures

**Development Learning Curve:**
- Event sourcing patterns and best practices
- Eventual consistency reasoning
- Event handler testing and debugging

### Risk Mitigations

- Start with single-node KurrentDB for MVP
- Implement comprehensive monitoring and alerting
- Establish event schema governance processes
- Create event sourcing training materials for team

## Implementation Plan

### Phase 1: Foundation
- Single KurrentDB node setup
- Basic event handler framework
- Core domain events (user actions, calculations)
- Simple projections for current state

### Phase 2: Scaling
- KurrentDB clustering and high availability
- Multi-layer event propagation
- Advanced projections and read models
- Performance optimization

### Phase 3: Integration
- Kafka market data bridge
- RabbitMQ command/event integration
- AI agent event processing pipeline
- Global event stream consolidation

## References

- [EventStoreDB Documentation](https://developers.eventstore.com/)
- [Event Sourcing Pattern](https://microservices.io/patterns/data/event-sourcing.html)
- [CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Turning the Database Inside Out - Martin Kleppmann](https://www.confluent.io/blog/turning-the-database-inside-out-with-apache-samza/)