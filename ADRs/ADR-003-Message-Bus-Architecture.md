# ADR-003: Message Bus Architecture with NATS JetStream

**Status:** Updated  
**Date:** 2025-09-03  
**Context:** Lucent Services Inter-Service Communication and Domain Events

## Context

Lucent Services requires a robust message bus for orchestrating communication between distributed services in our event-driven architecture. Based on the hybrid architecture decision (ADR-002), we use NATS JetStream for internal communication while Kafka handles external market data. The system needs to handle:

- Synchronous command/response patterns for trading operations
- Asynchronous job processing for AI model training and analysis
- Domain event publishing to EventStore
- Real-time AI agent coordination
- Hierarchical event propagation (edge → cluster → site → global)
- Reliable message delivery with exactly-once semantics

Key requirements:
- Sub-millisecond response times for internal commands
- Guaranteed message delivery with exactly-once semantics
- Subject-based routing for hierarchical event propagation
- Request/reply patterns for synchronous operations
- Operational simplicity with minimal infrastructure complexity

## Decision

We will use **NATS JetStream** as our primary message bus, implementing a comprehensive messaging architecture with the following patterns:

### Rationale for NATS JetStream over RabbitMQ:

- **Operational Simplicity**: Single binary, no external dependencies (vs RabbitMQ + Erlang + management complexity)
- **Exactly-Once Delivery**: Built-in JetStream guarantees (critical for financial operations)
- **Hierarchical Subjects**: Perfect match for our edge→cluster→global architecture pattern
- **Performance**: Lower latency and higher throughput than RabbitMQ
- **Cloud-Native**: Built for distributed systems from the ground up
- **Request/Reply**: Native RPC patterns without complex routing setup

### Subject and Stream Architecture

**Subject Strategy (Hierarchical):**
```
# Command/Response Subjects
lucent.commands.trading.execute          # Direct command routing
lucent.commands.portfolio.rebalance      # Portfolio operations
lucent.commands.analytics.calculate      # AI/Analytics commands

# Domain Events (to EventStore)
lucent.events.domain.user.created        # Business domain events
lucent.events.domain.trade.executed      # Trading domain events
lucent.events.domain.yield.detected      # Yield opportunity events

# Service Communication
lucent.services.{service}.health         # Health check subjects
lucent.services.{service}.config         # Configuration updates
lucent.services.{service}.alerts         # Service alerts

# AI Agent Coordination
lucent.agents.yield.analysis             # Yield farming analysis
lucent.agents.risk.assessment            # Risk analysis
lucent.agents.execution.optimization     # Trade execution optimization

# Hierarchical Event Propagation
lucent.hierarchy.node.{nodeId}           # Individual node events
lucent.hierarchy.cluster.{clusterId}     # Cluster-level events  
lucent.hierarchy.site.{siteId}           # Site-level events
lucent.hierarchy.global                  # Global events
```

**Stream Configuration:**
```
# JetStream streams for different event types
DOMAIN_EVENTS: subjects=["lucent.events.domain.*"], retention=interest, replicas=3
COMMANDS: subjects=["lucent.commands.*"], retention=limits, max_age=1h
AGENT_COORDINATION: subjects=["lucent.agents.*"], retention=limits, max_age=24h
HIERARCHY: subjects=["lucent.hierarchy.*"], retention=workqueue, replicas=3

```

**Message Schema (NATS Format):**
```json
{
  "messageId": "uuid",
  "correlationId": "uuid", 
  "causationId": "uuid",
  "timestamp": "2025-09-02T10:30:00.123Z",
  "messageType": "ExecuteTradeCommand | PositionUpdatedEvent | YieldCalculationJob",
  "version": "1.0",
  "subject": "lucent.commands.trading.execute",
  "replyTo": "lucent.responses.trading.{requestId}",
  "sender": {
    "serviceId": "trading-service-001",
    "instanceId": "ts-001-pod-abc123",
    "version": "1.2.3",
    "nodeId": "edge-node-001"
  },
  "jetstream": {
    "streamName": "COMMANDS",
    "deliveryPolicy": "all",
    "ackPolicy": "explicit",
    "maxDeliveries": 3
  },
  "payload": {
    "command": "execute_trade",
    "data": {
      "userId": "user-123",
      "pair": "ETH-USDC", 
      "side": "buy",
      "amount": 1.5,
      "orderType": "market",
      "slippage": 0.5
    }
  },
  "metadata": {
    "hierarchy_level": "node | cluster | site | global",
    "security": {
      "signature": "cryptographic-signature", 
      "permissions": ["trade.execute", "portfolio.read"],
      "identity": "spiffe://lucent.io/service/trading"
    }
  }
}
```

### Message Patterns Implementation

**Command Pattern (Request/Reply):**
```typescript
class TradingCommandHandler {
  constructor(private nats: NatsConnection, private jetstream: JetStreamManager) {}
  
  async executeTradeCommand(message: TradeCommand): Promise<TradeResult> {
    const correlationId = message.correlationId;
    
    // NATS request/reply pattern (built-in RPC)
    const response = await this.nats.request(
      'lucent.commands.trading.execute',
      JSON.stringify(message),
      { timeout: 30000 } // 30 second timeout
    );
    
    return JSON.parse(response.data.toString()) as TradeResult;
  }
  
  // Service handler for incoming commands
  async handleTradeExecution() {
    const subscription = this.nats.subscribe('lucent.commands.trading.execute');
    
    for await (const message of subscription) {
      try {
        const command = JSON.parse(message.data.toString()) as TradeCommand;
        const result = await this.processTradeCommand(command);
        
        // Reply to request
        message.respond(JSON.stringify(result));
      } catch (error) {
        message.respond(JSON.stringify({ error: error.message }));
      }
    }
  }
}
```

**Event Pattern (JetStream Publish/Subscribe):**
```typescript
class PortfolioEventPublisher {
  constructor(private jetstream: JetStreamClient) {}
  
  async publishBalanceUpdate(userId: string, balance: Balance): Promise<void> {
    const event: BalanceUpdatedEvent = {
      messageId: generateUUID(),
      correlationId: this.currentCorrelationId,
      messageType: 'BalanceUpdatedEvent',
      timestamp: new Date().toISOString(),
      subject: 'lucent.events.domain.portfolio.balance_updated',
      payload: {
        userId,
        balance,
        previousBalance: this.getPreviousBalance(userId)
      }
    };
    
    // JetStream publish with exactly-once delivery guarantee
    await this.jetstream.publish(
      'lucent.events.domain.portfolio.balance_updated',
      JSON.stringify(event),
      {
        msgID: event.messageId, // Deduplication
        headers: {
          'X-Correlation-ID': event.correlationId,
          'X-Event-Type': event.messageType
        }
      }
    );
  }
  
  // Subscribe to balance updates
  async subscribeToBalanceUpdates() {
    const consumer = await this.jetstream.consumers.get('DOMAIN_EVENTS', 'balance-processor');
    const subscription = consumer.consume({
      max_messages: 100,
      expires: 30000
    });
    
    for await (const message of subscription) {
      try {
        const event = JSON.parse(message.data.toString()) as BalanceUpdatedEvent;
        await this.processBalanceUpdate(event);
        message.ack(); // Acknowledge successful processing
      } catch (error) {
        console.error('Failed to process balance update:', error);
        message.nack(); // Negative acknowledgment for retry
      }
    }
  }
}
```

**Work Queue Pattern (JetStream Work Queue):**
```typescript
class AIModelTrainingQueue {
  constructor(private jetstream: JetStreamClient) {}
  
  async queueModelTraining(request: ModelTrainingRequest): Promise<void> {
    const job: ModelTrainingJob = {
      messageId: generateUUID(),
      messageType: 'ModelTrainingJob',
      subject: `lucent.agents.training.${request.modelType}`,
      payload: {
        modelType: request.modelType,
        trainingData: request.datasetId,
        hyperparameters: request.config,
        priority: request.priority || 'normal'
      },
      metadata: {
        estimated_duration_minutes: 180,
        required_gpu_memory: '8GB',
        max_retries: 2
      }
    };
    
    // Publish to work queue stream
    await this.jetstream.publish(
      `lucent.agents.training.${request.modelType}`,
      JSON.stringify(job),
      {
        msgID: job.messageId,
        headers: {
          'X-Priority': job.payload.priority,
          'X-Job-Type': job.messageType
        }
      }
    );
  }
  
  // Worker consuming from the training queue
  async startTrainingWorker() {
    const consumer = await this.jetstream.consumers.get('AGENT_COORDINATION', 'training-worker');
    const subscription = consumer.consume({
      max_messages: 1, // Process one job at a time
      expires: 3600000 // 1 hour timeout for long-running training
    });
    
    for await (const message of subscription) {
      try {
        const job = JSON.parse(message.data.toString()) as ModelTrainingJob;
        await this.processTrainingJob(job);
        message.ack();
      } catch (error) {
        console.error('Training job failed:', error);
        message.nack(3000); // Retry after 3 seconds
      }
    }
  }
}
```

### Service Integration Patterns

**Kafka-to-RabbitMQ Bridge:**
```typescript
class MarketDataBridge {
  constructor(
    private kafkaConsumer: KafkaConsumer,
    private rabbitPublisher: RabbitMQPublisher
  ) {}
  
  async start(): Promise<void> {
    // Consume yield opportunities from Kafka
    this.kafkaConsumer.subscribe(['yield.opportunities'], {
      eachMessage: async ({ topic, partition, message }) => {
        const yieldOpportunity = JSON.parse(message.value.toString());
        
        // Transform to command message
        const command: YieldAnalysisCommand = {
          messageId: generateUUID(),
          messageType: 'YieldAnalysisCommand',
          payload: {
            protocol: yieldOpportunity.protocol,
            yield: yieldOpportunity.apy,
            confidence: yieldOpportunity.confidence
          }
        };
        
        // Publish to RabbitMQ for AI agent processing
        await this.rabbitPublisher.publish({
          exchange: 'lucent.commands.direct',
          routingKey: 'ai.agents.yield_analysis',
          message: command
        });
      }
    });
  }
}
```

**KurrentDB Integration:**
```typescript
class DomainEventBridge {
  async handleServiceEvent(message: ServiceMessage): Promise<void> {
    // Transform service message to domain event
    const domainEvent = this.transformToDomainEvent(message);
    
    // Store in KurrentDB for event sourcing
    await this.eventStore.appendToStream(
      domainEvent.streamId,
      domainEvent.expectedVersion,
      [domainEvent]
    );
    
    // Acknowledge RabbitMQ message
    await message.ack();
  }
  
  private transformToDomainEvent(message: ServiceMessage): DomainEvent {
    return {
      eventId: generateUUID(),
      eventType: this.mapMessageTypeToDomainEvent(message.messageType),
      streamId: this.deriveStreamId(message),
      data: message.payload,
      metadata: {
        causationId: message.messageId,
        correlationId: message.correlationId,
        source: 'rabbitmq-bridge'
      }
    };
  }
}
```

## Technical Implementation

### RabbitMQ Configuration

**Cluster Setup:**
```yaml
# High Availability Configuration
cluster_formation.peer_discovery_backend: rabbit_peer_discovery_k8s
cluster_formation.k8s.host: kubernetes.default.svc.cluster.local
cluster_formation.k8s.address_type: hostname

# Performance Tuning
vm_memory_high_watermark.relative: 0.6
disk_free_limit.relative: 1.0
collect_statistics_interval: 10000

# Message Store
queue_master_locator: min-masters
ha_mode: all
ha_sync_mode: automatic
```

**Virtual Host Organization:**
```bash
# Environment-based virtual hosts
/lucent/development
/lucent/staging  
/lucent/production

# Feature-based virtual hosts (within environment)
/lucent/production/trading
/lucent/production/analytics
/lucent/production/notifications
```

**Security Configuration:**
```yaml
# Authentication & Authorization
auth_backends:
  - rabbit_auth_backend_internal
  - rabbit_auth_backend_ldap

# SSL/TLS Configuration  
ssl_listeners.default: 5671
ssl_options.cacertfile: /etc/ssl/certs/ca-certificates.crt
ssl_options.certfile: /etc/ssl/certs/server.crt
ssl_options.keyfile: /etc/ssl/private/server.key
ssl_options.verify: verify_peer
ssl_options.fail_if_no_peer_cert: true
```

### Connection Management

**Connection Pooling:**
```typescript
class RabbitMQConnectionManager {
  private pools: Map<string, ConnectionPool> = new Map();
  
  async getConnection(vhost: string = '/'): Promise<Connection> {
    if (!this.pools.has(vhost)) {
      this.pools.set(vhost, new ConnectionPool({
        url: `amqp://${this.config.host}:${this.config.port}${vhost}`,
        poolSize: 10,
        reconnectAttempts: 5,
        reconnectDelay: 1000
      }));
    }
    
    return this.pools.get(vhost)!.acquire();
  }
  
  async withChannel<T>(
    vhost: string, 
    operation: (channel: Channel) => Promise<T>
  ): Promise<T> {
    const connection = await this.getConnection(vhost);
    const channel = await connection.createChannel();
    
    try {
      return await operation(channel);
    } finally {
      await channel.close();
      await connection.release();
    }
  }
}
```

**Circuit Breaker Pattern:**
```typescript
class RabbitMQCircuitBreaker {
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
  private failures = 0;
  private lastFailure?: Date;
  
  async publish(message: Message): Promise<void> {
    if (this.state === 'OPEN' && this.shouldAttemptReset()) {
      this.state = 'HALF_OPEN';
    }
    
    if (this.state === 'OPEN') {
      throw new Error('Circuit breaker is OPEN');
    }
    
    try {
      await this.rabbitMQ.publish(message);
      this.onSuccess();
    } catch (error) {
      this.onFailure(error);
      throw error;
    }
  }
  
  private onSuccess(): void {
    this.failures = 0;
    this.state = 'CLOSED';
  }
  
  private onFailure(error: Error): void {
    this.failures++;
    this.lastFailure = new Date();
    
    if (this.failures >= this.config.failureThreshold) {
      this.state = 'OPEN';
    }
  }
}
```

### Dead Letter Handling

**Dead Letter Exchange Setup:**
```typescript
class DeadLetterHandler {
  async setupDeadLetterHandling(): Promise<void> {
    // Main exchange and queue
    await this.channel.assertExchange('lucent.commands.direct', 'direct');
    await this.channel.assertQueue('trading.commands.execute_order', {
      deadLetterExchange: 'lucent.deadletter.direct',
      deadLetterRoutingKey: 'trading.commands.execute_order.failed',
      messageTtl: 300000, // 5 minutes
      arguments: {
        'x-max-retries': 3
      }
    });
    
    // Dead letter exchange and queue
    await this.channel.assertExchange('lucent.deadletter.direct', 'direct');
    await this.channel.assertQueue('trading.commands.execute_order.failed');
    
    // Retry queue with delayed redelivery
    await this.channel.assertQueue('trading.commands.execute_order.retry', {
      deadLetterExchange: 'lucent.commands.direct',
      deadLetterRoutingKey: 'trading.commands.execute_order',
      messageTtl: 60000, // 1 minute delay
      arguments: {
        'x-dead-letter-ttl': 60000
      }
    });
  }
  
  async handleFailedMessage(message: FailedMessage): Promise<void> {
    const retryCount = message.properties.headers?.retryCount || 0;
    const maxRetries = message.properties.headers?.maxRetries || 3;
    
    if (retryCount < maxRetries) {
      // Retry with exponential backoff
      const delay = Math.pow(2, retryCount) * 1000;
      await this.scheduleRetry(message, delay, retryCount + 1);
    } else {
      // Send to permanent dead letter for manual investigation
      await this.sendToPermanentDeadLetter(message);
      await this.notifyOperationsTeam(message);
    }
  }
}
```

## Monitoring & Observability

**Health Checks:**
```typescript
class RabbitMQHealthCheck {
  async checkHealth(): Promise<HealthStatus> {
    const checks = await Promise.allSettled([
      this.checkConnection(),
      this.checkQueueDepths(),
      this.checkConsumerCounts(),
      this.checkMemoryUsage()
    ]);
    
    return {
      status: checks.every(c => c.status === 'fulfilled') ? 'healthy' : 'unhealthy',
      details: checks.map(c => c.status === 'fulfilled' ? c.value : c.reason),
      timestamp: new Date().toISOString()
    };
  }
  
  private async checkQueueDepths(): Promise<QueueDepthCheck> {
    const criticalQueues = ['trading.commands.execute_order', 'alerts.high'];
    const depths = await Promise.all(
      criticalQueues.map(q => this.management.getQueueInfo(q))
    );
    
    const issues = depths.filter(d => d.messages > 1000);
    
    return {
      status: issues.length === 0 ? 'ok' : 'warning',
      queues: depths,
      issues: issues
    };
  }
}
```

**Metrics Collection:**
```typescript
class RabbitMQMetrics {
  private prometheus = require('prom-client');
  
  private messagesSent = new this.prometheus.Counter({
    name: 'rabbitmq_messages_sent_total',
    help: 'Total messages sent',
    labelNames: ['exchange', 'routing_key', 'service']
  });
  
  private messagesReceived = new this.prometheus.Counter({
    name: 'rabbitmq_messages_received_total', 
    help: 'Total messages received',
    labelNames: ['queue', 'service', 'status']
  });
  
  private messageLatency = new this.prometheus.Histogram({
    name: 'rabbitmq_message_processing_duration_seconds',
    help: 'Message processing duration',
    labelNames: ['message_type', 'service'],
    buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5]
  });
  
  recordMessageSent(exchange: string, routingKey: string, service: string): void {
    this.messagesSent.inc({ exchange, routing_key: routingKey, service });
  }
  
  recordMessageReceived(queue: string, service: string, status: 'success' | 'error'): void {
    this.messagesReceived.inc({ queue, service, status });
  }
  
  recordMessageLatency(messageType: string, service: string, duration: number): void {
    this.messageLatency.observe({ message_type: messageType, service }, duration);
  }
}
```

## Consequences

### Benefits

**Reliability & Durability:**
- Guaranteed message delivery with acknowledgments
- Dead letter handling for failed messages
- Message persistence across broker restarts
- Cluster redundancy for high availability

**Flexible Messaging Patterns:**
- Support for multiple communication patterns
- Topic-based routing for event notifications
- Request/response for synchronous operations
- Work queues for background processing

**Operational Excellence:**
- Rich management and monitoring interfaces
- Plugin ecosystem for extensions
- Detailed metrics and logging
- Administrative tooling

**Performance & Scalability:**
- High throughput message processing
- Horizontal scaling through clustering
- Load balancing across consumers
- Configurable QoS and prefetch

### Challenges

**Operational Complexity:**
- Cluster management and maintenance
- Queue and exchange configuration
- Memory and disk usage monitoring
- Network partition handling

**Message Ordering:**
- No global ordering across queues
- Ordering only within single queue
- Complex routing for ordered processing
- Consumer coordination challenges

**Resource Management:**
- Memory usage with large queues
- Disk space for persistent messages
- Network bandwidth for clustering
- CPU overhead for message routing

### Risk Mitigations

- Use managed RabbitMQ services where available (CloudAMQP, AWS MQ)
- Implement comprehensive monitoring and alerting
- Create disaster recovery procedures
- Establish queue size limits and TTLs
- Use circuit breakers for external dependencies

## Integration Architecture

### Service Communication Flow:
```
1. Kafka Market Data → RabbitMQ Commands → AI Agents
2. AI Agent Decisions → RabbitMQ Events → KurrentDB Domain Events
3. Domain Events → RabbitMQ Notifications → External Systems
4. External Systems → RabbitMQ Commands → Internal Services
```

### Message Routing Strategy:
- **Commands:** Direct exchange for point-to-point communication
- **Events:** Topic exchange for publish/subscribe patterns
- **Jobs:** Direct exchange with work queue distribution
- **Notifications:** Fanout exchange for broadcast messaging

## Implementation Phases

### Phase 1: Core Messaging
- Single RabbitMQ node setup
- Basic exchange and queue topology
- Command/response patterns
- Simple event publishing

### Phase 2: Reliability & Scale
- RabbitMQ cluster deployment
- Dead letter handling implementation
- Circuit breakers and retry logic
- Comprehensive monitoring

### Phase 3: Advanced Integration
- Kafka-RabbitMQ bridge services
- KurrentDB event integration
- AI agent communication protocols
- Cross-region message replication

## References

- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
- [RabbitMQ Best Practices](https://www.cloudamqp.com/blog/part1-rabbitmq-best-practices.html)
- [Microservices Messaging Patterns](https://microservices.io/patterns/communication-style/messaging.html)