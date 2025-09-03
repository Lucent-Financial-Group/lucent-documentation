# ADR-002: Market Data Streaming with Apache Kafka

**Status:** Proposed  
**Date:** 2025-09-02  
**Context:** Lucent Services Real-Time Market Data Processing

## Context

Lucent Services requires high-throughput, low-latency processing of cryptocurrency market data to enable:

- Real-time yield farming opportunity detection
- Portfolio rebalancing triggers
- Price alert and notification systems
- Historical market data analysis for AI training
- Cross-exchange arbitrage opportunity identification

Market data characteristics:
- High volume (millions of price updates per day)
- Low latency requirements (<100ms end-to-end)
- Multiple data sources (exchanges, DeFi protocols, oracles)
- Varying update frequencies (tick-by-tick to periodic snapshots)
- Need for data replay and historical analysis

## Decision

We will use **Apache Kafka** as our market data streaming platform for external market data ingestion and **ClickHouse** for analytics queries, implementing a comprehensive real-time data processing pipeline with the following hybrid architecture:

### Hybrid Architecture Overview

```
External APIs → Kafka → ClickHouse (market data pipeline)
Internal Services ↔ NATS JetStream (service communication)
Business Events → NATS → EventStore (domain events)
AI Agents → NATS subjects (real-time coordination)
```

**Rationale for Hybrid Approach:**
- **Kafka**: Proven high-throughput streaming for financial market data with mature ClickHouse integration
- **NATS JetStream**: Superior operational simplicity for internal service communication and domain events
- **ClickHouse**: Optimal for market data analytics with native Kafka engine support
- **EventStore**: Event sourcing for business domain events via NATS integration

### Kafka Cluster Architecture

**Topic Strategy:**
```
market.prices.{exchange}.{pair}     # e.g., market.prices.binance.eth-usdc
market.orderbook.{exchange}.{pair}  # e.g., market.orderbook.uniswap.eth-usdc
market.trades.{exchange}.{pair}     # e.g., market.trades.coinbase.btc-usd
market.yields.{protocol}.{pool}     # e.g., market.yields.aave.eth-lending
market.liquidations.{protocol}      # e.g., market.liquidations.compound
market.governance.{protocol}        # e.g., market.governance.makerdao
```

**Partitioning Strategy:**
- Partition by trading pair for price data
- Partition by protocol for DeFi data
- Custom partitioning for cross-shard analysis
- Time-based partitioning for historical queries

**Message Schema:**
```json
{
  "header": {
    "messageId": "uuid",
    "timestamp": "2025-09-02T10:30:00.123Z",
    "source": "binance-websocket",
    "messageType": "price_update",
    "version": "1.0"
  },
  "data": {
    "exchange": "binance",
    "pair": "ETH-USDC",
    "price": 2847.52,
    "volume24h": 1234567.89,
    "change24h": 2.45,
    "bid": 2847.40,
    "ask": 2847.60,
    "spread": 0.20,
    "liquidity": {
      "bidSize": 100.5,
      "askSize": 85.3
    }
  },
  "metadata": {
    "latencyMs": 15,
    "confidenceScore": 0.98,
    "dataQuality": "high"
  }
}
```

### Data Ingestion Pipeline

**Market Data Connectors:**
- **CEX Connectors:** Binance, Coinbase, Kraken WebSocket feeds
- **DEX Connectors:** Uniswap, SushiSwap, PancakeSwap event logs
- **Oracle Connectors:** Chainlink, Band Protocol price feeds
- **Lending Connectors:** Aave, Compound yield rate monitoring
- **Cross-chain Connectors:** Layer 2 and bridge monitoring

**Data Validation & Enrichment:**
```typescript
interface MarketDataProcessor {
  validate(message: MarketMessage): ValidationResult;
  enrich(message: MarketMessage): EnrichedMarketMessage;
  deduplicate(messages: MarketMessage[]): MarketMessage[];
}

class PriceDataProcessor implements MarketDataProcessor {
  validate(message: PriceMessage): ValidationResult {
    // Price sanity checks, outlier detection
    // Volume validation, spread analysis
    // Timestamp drift detection
  }
  
  enrich(message: PriceMessage): EnrichedPriceMessage {
    // Add moving averages, volatility metrics
    // Cross-exchange spread calculations
    // Liquidity depth analysis
  }
}
```

**Stream Processing Topology:**
1. **Raw Ingestion:** Direct connector → Kafka topic
2. **Validation:** Kafka Streams for data quality checks
3. **Enrichment:** Add derived metrics and cross-references
4. **Normalization:** Standardize formats across exchanges
5. **Distribution:** Route to consuming applications

### Real-Time Processing Framework

**Kafka Streams Applications:**

**Yield Opportunity Detector:**
```java
StreamsBuilder builder = new StreamsBuilder();

KStream<String, YieldData> yieldStream = builder
    .stream("market.yields.*")
    .filter((key, value) -> value.apy > 5.0)
    .groupByKey()
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .aggregate(YieldOpportunity::new, YieldOpportunity::accumulate)
    .toStream()
    .filter((key, opportunity) -> opportunity.isViable());

yieldStream.to("yield.opportunities", Produced.with(Serdes.String(), yieldSerde));
```

**Price Alert Engine:**
```java
KStream<String, PriceData> priceStream = builder.stream("market.prices.*");
KStream<String, Alert> alerts = priceStream
    .join(userAlertsTable, (price, alert) -> {
        if (price.current >= alert.targetPrice) {
            return new Alert(alert.userId, price, AlertType.PRICE_TARGET_HIT);
        }
        return null;
    })
    .filter((key, alert) -> alert != null);

alerts.to("user.alerts");
```

**Arbitrage Detection:**
```java
KStream<String, PriceData> crossExchangeStream = priceStream
    .groupBy((key, price) -> price.pair)
    .windowedBy(TimeWindows.of(Duration.ofSeconds(10)))
    .aggregate(
        CrossExchangePrice::new,
        (key, price, aggregate) -> aggregate.addExchange(price),
        Materialized.with(Serdes.String(), crossExchangeSerde)
    )
    .toStream()
    .mapValues(CrossExchangePrice::findArbitrageOpportunities)
    .flatMapValues(opportunities -> opportunities);

crossExchangeStream.to("arbitrage.opportunities");
```

### Integration with ClickHouse Analytics

**Kafka-to-ClickHouse Pipeline:**
```typescript
// ClickHouse table with Kafka engine
CREATE TABLE market_data_stream (
    timestamp DateTime64(3),
    exchange String,
    pair String,
    price Float64,
    volume Float64,
    bid Float64,
    ask Float64,
    metadata String
) ENGINE = Kafka('localhost:9092', 'market.prices.all', 'clickhouse_group', 'JSONEachRow');

// Materialized view for real-time aggregation
CREATE MATERIALIZED VIEW market_data_mv TO market_data AS
SELECT 
    toStartOfMinute(timestamp) as minute,
    exchange,
    pair,
    avg(price) as avg_price,
    sum(volume) as total_volume,
    max(price) as high,
    min(price) as low
FROM market_data_stream
GROUP BY minute, exchange, pair;
```

**Analytics Query Examples:**
```sql
-- Top 20 coins by volume today
SELECT pair, sum(total_volume) as volume 
FROM market_data 
WHERE toDate(minute) = today() 
GROUP BY pair 
ORDER BY volume DESC 
LIMIT 20;

-- Real-time yield opportunities
SELECT protocol, pool, apy, tvl
FROM yield_data 
WHERE apy > 5.0 AND tvl > 1000000
ORDER BY apy DESC;
```

### Integration with Domain Events via NATS

**Market-to-Domain Event Bridge:**
- Kafka Consumer listening to market topics
- Transform market events into domain commands
- Publish to NATS for EventStore persistence
- Correlation between market triggers and business actions

**Event Flow Example:**
```
1. Kafka: market.yields.aave.eth → High yield detected (8.5% APY)
2. Stream Processor: Yield opportunity analysis
3. Bridge Service: Transform to domain command
4. NATS: yield.opportunity.detected → EventStore domain event
5. Event Handler: Calculate position sizing
6. EventStore: PositionRecommendationEvent  
7. NATS: Notify trading engine and AI agents
```

## Technical Implementation

### Kafka Configuration

**Broker Configuration:**
```properties
# Performance tuning
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# Reliability
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# Retention for market data
log.retention.hours=168  # 7 days for hot data
log.retention.bytes=107374182400  # 100GB per partition
```

**Producer Configuration:**
```properties
# Low latency for real-time feeds
acks=1
retries=3
batch.size=16384
linger.ms=1
compression.type=snappy

# Idempotent producers for data integrity
enable.idempotence=true
max.in.flight.requests.per.connection=1
```

**Consumer Configuration:**
```properties
# Real-time processing
fetch.min.bytes=1
fetch.max.wait.ms=10
max.partition.fetch.bytes=1048576

# Offset management
auto.offset.reset=latest
enable.auto.commit=false  # Manual commit for exactly-once
```

### Schema Management

**Confluent Schema Registry:**
- Avro schemas for market data messages
- Schema evolution compatibility rules
- Version management and backward compatibility
- Schema validation at producer/consumer level

**Schema Evolution Strategy:**
- Backward compatible changes only
- Optional fields for new data points
- Schema versioning in message headers
- Graceful degradation for unknown fields

### Monitoring & Operations

**Kafka Monitoring:**
- JMX metrics for broker health
- Consumer lag monitoring
- Topic partition balance
- Message throughput and latency metrics

**Data Quality Monitoring:**
- Message validation failure rates
- Data freshness and staleness alerts
- Cross-exchange price deviation detection
- Missing data source monitoring

**Performance Metrics:**
- End-to-end latency (exchange → application)
- Message processing throughput
- Consumer group rebalancing frequency
- Disk utilization and growth trends

## Consequences

### Benefits

**High Throughput & Low Latency:**
- Handle millions of market updates per day
- Sub-100ms processing latency for critical paths
- Horizontal scaling through partition addition
- Efficient batch processing for analytics

**Data Reliability:**
- Guaranteed message delivery with replication
- Exactly-once processing semantics
- Message ordering within partitions
- Fault tolerance through cluster redundancy

**Ecosystem Integration:**
- Rich connector ecosystem for data sources
- Stream processing with Kafka Streams
- Integration with analytics platforms
- Support for multiple consumer patterns

**Operational Flexibility:**
- Topic-based data organization
- Independent scaling of producers/consumers
- Time-based data retention policies
- Schema evolution capabilities

### Challenges

**Operational Complexity:**
- Kafka cluster management and tuning
- Topic partition rebalancing
- Schema registry maintenance
- Cross-region replication setup

**Data Consistency:**
- Eventually consistent across partitions
- Potential message duplication scenarios
- Clock synchronization across data sources
- Handling late-arriving data

**Cost Considerations:**
- Storage costs for high-volume data retention
- Network bandwidth for replication
- Operational overhead of cluster management
- Schema registry licensing costs

### Risk Mitigations

- Start with single-node Kafka for MVP
- Implement comprehensive monitoring from day one
- Use managed Kafka services for production (Confluent Cloud/MSK)
- Establish data retention and archival policies early
- Create runbooks for common operational scenarios

## Integration Points

### With ClickHouse (Analytics):**
- Native Kafka engine for real-time data ingestion
- Materialized views for aggregated market metrics  
- Sub-second query performance for complex analytics
- Historical data analysis and trend detection

### With EventStore via NATS (Domain Events):**
- Market event bridge service transforms Kafka → NATS → EventStore
- Command transformation pipeline for business actions
- Event correlation and causation tracking via NATS subjects
- Business rule execution triggers through event sourcing

### With NATS JetStream (Service Communication):**
- Real-time market alerts to trading services
- AI agent coordination and market signal distribution
- Service-to-service communication for internal commands
- Hierarchical subject routing for multi-layer architecture

### With AI Agents:**
- Real-time market data streams via Kafka consumers
- Historical data analysis through ClickHouse queries
- Feature engineering pipelines consuming market streams
- Model prediction publishing via NATS subjects

## Implementation Phases

### Phase 1: Core Infrastructure
- Single-broker Kafka setup
- Basic price data ingestion
- Simple stream processing applications
- Integration with existing monitoring

### Phase 2: Production Scaling  
- Multi-broker cluster deployment
- Schema registry implementation
- Advanced stream processing topologies
- Cross-region replication

### Phase 3: Advanced Analytics
- Real-time ML feature computation
- Complex event pattern detection
- Multi-exchange arbitrage detection
- Predictive analytics pipelines

## References

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Kafka Streams Developer Guide](https://docs.confluent.io/platform/current/streams/developer-guide/)
- [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/)
- [Building Event-Driven Microservices](https://www.oreilly.com/library/view/building-event-driven-microservices/9781492057888/)