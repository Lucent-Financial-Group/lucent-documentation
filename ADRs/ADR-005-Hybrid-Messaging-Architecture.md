# ADR-005: Hybrid Messaging Architecture

**Status:** Proposed  
**Date:** 2025-09-03  
**Context:** Lucent Services Unified Event-Driven Architecture

## Context

Based on architectural decisions in ADR-002 (Market Data Streaming) and ADR-003 (Message Bus Architecture), Lucent Services implements a hybrid messaging approach that combines the strengths of multiple messaging systems for optimal performance and operational simplicity.

## Decision

We implement a **Hybrid Messaging Architecture** using specialized messaging systems for different use cases:

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Lucent Services Architecture                  │
│                                                                 │
│  External APIs → Kafka → ClickHouse (market data pipeline)     │
│  Services ↔ NATS JetStream (internal communication)            │
│  Domain Events → NATS → EventStore (business events)           │
│  AI Agents → NATS subjects (real-time coordination)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Messaging System Responsibilities

#### **Apache Kafka: External Market Data Pipeline**

**Use Cases:**
- External exchange price feeds (Binance, Coinbase, etc.)
- DeFi protocol yield data (Aave, Compound, Curve)
- High-frequency trading signals
- Market analytics and historical data

**Rationale:**
- **High throughput**: Proven capability for millions of financial messages per day
- **ClickHouse integration**: Native Kafka engine provides seamless analytics
- **Ecosystem maturity**: Rich connector ecosystem for external data sources
- **Data replay**: Essential for backtesting and historical analysis

#### **NATS JetStream: Internal Service Communication**

**Use Cases:**
- Service-to-service command/response patterns
- Domain event publishing to EventStore
- AI agent coordination and task distribution
- Hierarchical event propagation (edge→cluster→site→global)
- Internal health monitoring and configuration

**Rationale:**
- **Operational simplicity**: Single binary, minimal infrastructure complexity
- **Exactly-once delivery**: Critical for financial domain events
- **Request/reply patterns**: Built-in RPC without complex routing
- **Hierarchical subjects**: Perfect match for decentralized architecture
- **Performance**: Sub-millisecond latency for internal operations

#### **EventStore: Domain Event Persistence**

**Use Cases:**
- Business domain events (user actions, trades, calculations)
- Event sourcing and audit trails
- Projections for read models
- Event replay for debugging and recovery

**Integration:** Receives events via NATS subjects for guaranteed delivery

#### **ClickHouse: Market Data Analytics**

**Use Cases:**
- Real-time market data queries ("top 20 coins by volume today")
- Historical trend analysis and backtesting
- Cross-exchange arbitrage detection
- Performance metrics and reporting

**Integration:** Consumes directly from Kafka using native Kafka engine

### Data Flow Architecture

#### **Market Data Flow:**
```
Exchange APIs → Market Data Service → Kafka Topics → ClickHouse Tables
                                   ↓
                              Stream Processors → NATS → EventStore
                                   ↓                ↓
                             AI Agent Analysis → Business Events
```

#### **Business Event Flow:**
```
User Actions → Services → NATS Commands → Business Logic
                              ↓
                        NATS Domain Events → EventStore → Projections
                              ↓
                        AI Agent Coordination → Real-time Processing
```

#### **Integration Bridge Flow:**
```
Kafka Market Events → Bridge Service → NATS Domain Events → EventStore
NATS Commands → External Service → Kafka Market Data → Analytics
```

## Technical Implementation

### Message Routing Strategy

#### **Subject/Topic Naming Convention:**

**Kafka Topics (Market Data):**
```
market.prices.{exchange}.{pair}
market.yields.{protocol}.{pool}  
market.orderbook.{exchange}.{pair}
market.liquidations.{protocol}
```

**NATS Subjects (Internal):**
```
lucent.commands.{service}.{action}        # Commands
lucent.events.domain.{entity}.{event}     # Domain events
lucent.agents.{agent_type}.{task}         # AI coordination
lucent.hierarchy.{level}.{identifier}     # Hierarchical events
```

### Integration Patterns

#### **Kafka-to-NATS Bridge:**
```typescript
class MarketDataBridge {
  constructor(
    private kafkaConsumer: KafkaConsumer,
    private natsClient: NatsConnection,
    private jetstream: JetStreamClient
  ) {}
  
  async start(): Promise<void> {
    // Consume yield opportunities from Kafka
    this.kafkaConsumer.subscribe(['yield.opportunities'], {
      eachMessage: async ({ topic, message }) => {
        const yieldData = JSON.parse(message.value.toString());
        
        // Transform to domain event
        const domainEvent: YieldOpportunityEvent = {
          messageId: generateUUID(),
          messageType: 'YieldOpportunityDetected',
          subject: 'lucent.events.domain.yield.opportunity_detected',
          payload: yieldData,
          metadata: {
            source: 'kafka-bridge',
            kafkaTopic: topic,
            kafkaOffset: message.offset
          }
        };
        
        // Publish to NATS → EventStore
        await this.jetstream.publish(
          domainEvent.subject,
          JSON.stringify(domainEvent),
          { msgID: domainEvent.messageId }
        );
      }
    });
  }
}
```

#### **NATS-to-EventStore Integration:**
```typescript
class EventStoreBridge {
  constructor(
    private natsClient: NatsConnection,
    private eventStore: EventStoreDBClient
  ) {}
  
  async start(): Promise<void> {
    // Subscribe to all domain events
    const subscription = this.natsClient.subscribe('lucent.events.domain.*');
    
    for await (const message of subscription) {
      try {
        const event = JSON.parse(message.data.toString());
        
        // Store in EventStore
        await this.eventStore.appendToStream(
          this.deriveStreamId(event),
          event,
          { expectedRevision: BACKWARDS }
        );
        
        console.log(`Stored event ${event.messageId} in EventStore`);
      } catch (error) {
        console.error('Failed to store event in EventStore:', error);
        // Handle retry logic or dead letter processing
      }
    }
  }
  
  private deriveStreamId(event: DomainEvent): string {
    // Create stream ID based on event type and entity
    return `${event.messageType}-${event.payload.entityId || 'global'}`;
  }
}
```

### Configuration Management

#### **NATS JetStream Setup:**
```typescript
// jetstream-config.ts
export const JETSTREAM_CONFIG = {
  streams: [
    {
      name: 'DOMAIN_EVENTS',
      subjects: ['lucent.events.domain.*'],
      retention: RetentionPolicy.Interest,
      replicas: 3,
      max_age: nanos(365 * 24 * 60 * 60 * 1000000000), // 1 year
    },
    {
      name: 'COMMANDS',
      subjects: ['lucent.commands.*'],
      retention: RetentionPolicy.Limits,
      max_age: nanos(1 * 60 * 60 * 1000000000), // 1 hour
      max_msgs: 100000,
    },
    {
      name: 'AGENT_COORDINATION', 
      subjects: ['lucent.agents.*'],
      retention: RetentionPolicy.WorkQueue,
      replicas: 3,
      max_age: nanos(24 * 60 * 60 * 1000000000), // 24 hours
    }
  ],
  
  consumers: [
    {
      stream_name: 'DOMAIN_EVENTS',
      name: 'eventstore-bridge',
      ack_policy: AckPolicy.Explicit,
      deliver_policy: DeliverPolicy.All,
      max_deliver: 3
    }
  ]
};
```

#### **Kafka-ClickHouse Integration:**
```sql
-- ClickHouse table consuming from Kafka
CREATE TABLE market_data_kafka (
    timestamp DateTime64(3),
    exchange String,
    pair String,
    price Float64,
    volume Float64,
    bid Float64,
    ask Float64
) ENGINE = Kafka('localhost:9092', 'market.prices.all', 'clickhouse_consumer', 'JSONEachRow')
SETTINGS kafka_commit_every_batch_consumed = 1;

-- Materialized view for real-time aggregation
CREATE MATERIALIZED VIEW market_data_realtime TO market_data AS
SELECT 
    toStartOfMinute(timestamp) as minute,
    exchange,
    pair,
    avg(price) as avg_price,
    sum(volume) as volume,
    max(price) as high,
    min(price) as low
FROM market_data_kafka
GROUP BY minute, exchange, pair;
```

## Benefits and Trade-offs

### Benefits

#### **System Optimization:**
- **Right tool for the job**: Each messaging system optimized for its specific use case
- **Performance**: High-throughput Kafka for market data, low-latency NATS for commands
- **Operational efficiency**: NATS simplicity for internal ops, Kafka power for external data

#### **Scalability:**
- **Independent scaling**: Market data pipeline scales separately from internal services
- **Resource optimization**: Different performance characteristics don't interfere
- **Load distribution**: Distribute high-frequency market data vs. business logic efficiently

#### **Reliability:**
- **Fault isolation**: Market data issues don't affect internal service communication
- **Exactly-once semantics**: NATS JetStream guarantees for critical business events
- **Data durability**: EventStore persistence with Kafka replay capabilities

### Trade-offs

#### **Complexity:**
- **Multiple systems**: More infrastructure to manage and monitor
- **Integration overhead**: Bridge services add complexity and failure points
- **Learning curve**: Team needs expertise in multiple messaging paradigms

#### **Operational Considerations:**
- **Monitoring**: Need to monitor Kafka, NATS, EventStore, and ClickHouse separately
- **Debugging**: Event flows span multiple systems, requiring distributed tracing
- **Configuration management**: Different configuration patterns for each system

### Risk Mitigations

#### **Infrastructure:**
- **Docker Compose**: Unified development environment for all systems
- **Health checks**: Comprehensive monitoring for all message flow components
- **Circuit breakers**: Prevent cascade failures between systems

#### **Operational:**
- **Comprehensive logging**: Correlation IDs across all systems
- **Monitoring dashboards**: Unified view of message flows
- **Runbooks**: Documented procedures for common scenarios

## Implementation Phases

### Phase 1: Core Infrastructure (Week 1-2)
- Set up Kafka, NATS, EventStore, ClickHouse in Docker Compose
- Basic bridge services between systems
- Health monitoring and basic observability

### Phase 2: Service Integration (Week 3-4)
- Implement service clients for each messaging system
- Create standardized message schemas and routing
- Test end-to-end message flows

### Phase 3: Advanced Features (Week 5-8)
- Implement advanced JetStream features (work queues, hierarchical subjects)
- Optimize ClickHouse queries and materialized views
- Add comprehensive monitoring and alerting

### Phase 4: Production Hardening (Week 9-12)
- Clustering and high availability setup
- Performance tuning and optimization  
- Disaster recovery and backup procedures

## Monitoring and Observability

### Key Metrics

#### **Message Flow Metrics:**
- **Throughput**: Messages per second through each system
- **Latency**: End-to-end processing times across system boundaries
- **Error rates**: Failed messages, retries, dead letters

#### **System Health:**
- **Kafka**: Consumer lag, partition distribution, broker health
- **NATS**: Connection status, JetStream storage, consumer health
- **EventStore**: Stream position, projection processing
- **ClickHouse**: Query performance, ingestion rates

### Distributed Tracing

All messages carry correlation IDs that flow through:
```
Kafka Message → Bridge Service → NATS Event → EventStore → Projections
     ↓              ↓              ↓           ↓            ↓
  Trace ID    →  Trace ID   →   Trace ID  →  Trace ID  →  Trace ID
```

## References

- [NATS JetStream Documentation](https://docs.nats.io/nats-concepts/jetstream)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [ClickHouse Kafka Engine](https://clickhouse.com/docs/engines/table-engines/integrations/kafka)
- [EventStoreDB Documentation](https://developers.eventstore.com/)
- [Microservices Message Patterns](https://microservices.io/patterns/communication-style/messaging.html)