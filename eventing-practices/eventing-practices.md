# Lucent Services Event-Driven Architecture Practices

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Mandatory patterns for all Lucent Services microservices

## Overview

This document defines the **mandatory coding standards** for all services in the Lucent Services event-driven architecture. Following these patterns ensures world-class debugging, tracing, and operational excellence across the entire crypto trading platform.

## Setup Requirements

### 1. Install Infrastructure Package

Add to your microservice's `package.json`:

```json
{
  "dependencies": {
    "@lucent/infrastructure": "^1.0.0",
    "@nestjs/core": "^10.0.0",
    "@nestjs/platform-fastify": "^10.0.0",
    "@nestjs/common": "^10.0.0"
  }
}
```

Install in your microservice:
```bash
npm install @lucent/infrastructure
```

### 2. Environment Setup

Your microservice needs access to infrastructure services:

```yaml
# docker-compose.yml for your service
version: '3.8'
services:
  my-service:
    image: my-service:latest
    environment:
      - NATS_URL=nats://localhost:4222
      - KAFKA_BROKERS=localhost:9092
      - CLICKHOUSE_HOST=localhost:8123
      - KURRENTDB_CONNECTION=esdb://localhost:2113
    networks:
      - lucent-network

networks:
  lucent-network:
    external: true  # Connect to infrastructure network
```

## Service Architecture Requirements

### 1. Service Base Classes (MANDATORY)

All services **MUST** extend the appropriate base class:

```typescript
import { 
  CryptoTradingServiceBase, 
  UserServiceBase, 
  LucentServiceBase,
  BusinessContextBuilder,
  InfrastructureFactory 
} from '@lucent/infrastructure';

// Trading services
class YieldFarmingService extends CryptoTradingServiceBase {
  constructor() {
    const infrastructure = await InfrastructureFactory.createDefault();
    super(infrastructure, 'yield-farming');
  }
}

// User management services  
class UserService extends UserServiceBase {
  constructor() {
    const infrastructure = await InfrastructureFactory.createDefault();
    super(infrastructure, 'user-management');
  }
}

// Other services
class NotificationService extends LucentServiceBase {
  constructor() {
    const infrastructure = await InfrastructureFactory.createDefault();
    super(infrastructure, 'notifications');
  }
}
```

### 2. Business Operations (ENFORCED PATTERN)

**ALL business operations MUST use the enforced pattern:**

```typescript
class TradingService extends CryptoTradingServiceBase {
  async executeSwap(request: SwapRequest): Promise<SwapResult> {
    // REQUIRED: Use business context builder
    const businessContext = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('Trade', `trade-${request.tradeId}`)
      .withUser(request.userId)
      .withTradingPair(request.pair)
      .withExchange(request.exchange)
      .withAmount(request.amountUsd)
      .withWorkflow(`swap-workflow-${request.tradeId}`)
      .withPriority(request.amountUsd > 100000 ? 'high' : 'normal')
      .build();

    // ENFORCED: All operations go through executeBusinessOperation
    return this.executeBusinessOperation('execute_swap', businessContext, async (context) => {
      // Your business logic here
      const swapResult = await this.performSwap(request);
      
      // ENFORCED: Publish domain events through framework
      await this.publishDomainEvent(
        'SwapExecuted',
        `trade-${request.tradeId}`,
        swapResult,
        businessContext,
        context
      );

      return swapResult;
    });
  }
}
```

### 3. Domain Event Publishing (MANDATORY PATTERN)

```typescript
// WRONG: Never publish events directly
await infrastructure.eventStore.append(streamId, [event]); // ❌ NO BUSINESS CONTEXT

// CORRECT: Always use framework methods
await this.publishDomainEvent(
  'YieldOpportunityDetected',      // Event type
  `yield-analysis-${analysisId}`,  // Stream ID  
  {                                // Event data
    protocol: 'aave',
    apy: 8.5,
    tvl: 1000000,
    riskScore: 3.2
  },
  businessContext,                 // Business context (REQUIRED)
  requestContext                   // Request context (optional)
);
```

### 4. Event Handling (ENFORCED PATTERN)

```typescript
class PortfolioService extends CryptoTradingServiceBase {
  async handleTradeExecuted(event: TradeExecutedEvent): Promise<void> {
    // ENFORCED: Use handleDomainEvent wrapper
    return this.handleDomainEvent(event, async (event, context) => {
      // Your event handling logic here
      const portfolio = await this.updatePortfolio(event.data);
      
      // Publish resulting events through framework
      await this.publishDomainEvent(
        'PortfolioUpdated',
        `portfolio-${event.data.userId}`,
        portfolio,
        context.businessContext,
        context
      );
    });
  }
}
```

### 5. Cross-Service Communication (MANDATORY PATTERN)

```typescript
class YieldService extends CryptoTradingServiceBase {
  async requestRiskAssessment(yieldData: YieldData): Promise<RiskAssessment> {
    const businessContext = BusinessContextBuilder
      .forYieldFarming()
      .withAggregate('YieldAnalysis', `analysis-${yieldData.id}`)
      .withUser(yieldData.userId)
      .withProtocol(yieldData.protocol)
      .build();

    // ENFORCED: Use sendCommand for service communication
    return this.sendCommand<YieldData, RiskAssessment>(
      'risk-assessment',        // Target service
      'AssessYieldRisk',       // Command type
      yieldData,               // Command data
      businessContext,         // Business context (REQUIRED)
      15000                    // Timeout
    );
  }
}
```

### 6. Multi-Step Business Processes (ENFORCED PATTERN)

```typescript
class YieldOptimizationService extends CryptoTradingServiceBase {
  async optimizeYieldStrategy(request: OptimizeRequest): Promise<Strategy> {
    const businessContext = BusinessContextBuilder
      .forYieldFarming()
      .withAggregate('YieldStrategy', `strategy-${request.strategyId}`)
      .withUser(request.userId)
      .withAmount(request.portfolioValueUsd)
      .build();

    // ENFORCED: Multi-step processes use executeBusinessProcess
    return this.executeBusinessProcess('yield_optimization', businessContext, [
      {
        name: 'analyze_current_positions',
        execute: async (context) => {
          return this.analyzeCurrentPositions(request.userId);
        }
      },
      {
        name: 'identify_yield_opportunities', 
        execute: async (context, currentPositions) => {
          return this.findYieldOpportunities(currentPositions);
        }
      },
      {
        name: 'assess_risks',
        execute: async (context, opportunities) => {
          return this.assessRisks(opportunities);
        }
      },
      {
        name: 'generate_strategy',
        execute: async (context, riskAssessment) => {
          return this.generateOptimalStrategy(riskAssessment);
        }
      }
    ]);
  }
}
```

## OpenTelemetry Integration Benefits

### Automatic Tracing Attributes

Every operation automatically gets:

```json
{
  "business.domain": "crypto-trading",
  "business.bounded_context": "yield-farming", 
  "business.aggregate_type": "YieldStrategy",
  "business.aggregate_id": "strategy-abc123",
  "business.user_id": "user-456",
  "business.workflow_id": "yield-workflow-789",
  "business.process_name": "yield_optimization",
  "business.step_number": 3,
  "business.total_steps": 4,
  
  "crypto.trading_pair": "ETH-USDC",
  "crypto.exchange": "binance",
  "crypto.protocol": "aave",
  "crypto.amount_usd": 50000,
  "crypto.risk_level": "medium",
  "crypto.strategy_type": "yield-farming",
  
  "event.type": "YieldOpportunityDetected",
  "event.stream_id": "yield-analysis-abc123",
  "event.correlation_id": "corr-456-789",
  "event.causation_id": "event-123-456",
  
  "service.name": "yield-farming",
  "service.operation": "business_operation",
  
  "request.id": "req-789-012",
  "request.source": "yield-farming"
}
```

### Cross-Service Context Propagation

Business context automatically flows through:
```
HTTP Request → Service A → NATS Command → Service B → EventStore → Projection Update
    ↓              ↓             ↓             ↓           ↓              ↓
workflow-123 → workflow-123 → workflow-123 → workflow-123 → workflow-123 → workflow-123
user-456     → user-456     → user-456     → user-456     → user-456     → user-456
ETH-USDC     → ETH-USDC     → ETH-USDC     → ETH-USDC     → ETH-USDC     → ETH-USDC
```

## Event Naming Conventions (MANDATORY)

### Domain Events
```typescript
// Pattern: {AggregateType}{Action}{Result}
'UserCreated'              // User creation completed
'TradeExecuted'            // Trade execution completed  
'YieldOpportunityDetected' // Yield opportunity found
'RiskAssessmentCompleted'  // Risk analysis finished
'PortfolioRebalanced'      // Portfolio rebalancing done
'AlertTriggered'           // Alert condition met
```

### NATS Subjects
```typescript
// Pattern: lucent.events.domain.{domain}.{EventType}
'lucent.events.domain.crypto_trading.TradeExecuted'
'lucent.events.domain.yield_farming.YieldOpportunityDetected'  
'lucent.events.domain.user_management.UserCreated'

// Commands: lucent.commands.{service}.{CommandType}
'lucent.commands.risk_assessment.AssessYieldRisk'
'lucent.commands.trading.ExecuteSwap'
'lucent.commands.portfolio.Rebalance'
```

### EventStore Streams
```typescript
// Pattern: {aggregate-type}-{aggregate-id}
'user-123'                    // User aggregate events
'trade-abc123'               // Trade aggregate events
'portfolio-user-456'         // User portfolio events
'yield-analysis-789'         // Yield analysis events
'risk-assessment-xyz'        // Risk assessment events
```

## Debugging Capabilities

### Find All Events for User
```bash
# Jaeger query
service.name="*" AND business.user_id="user-123"
```

### Trace Complete Yield Farming Decision
```bash  
# Jaeger query
business.workflow_id="yield-workflow-789"
```

### Find All High-Risk Trades
```bash
# Jaeger query
crypto.risk_level="high" AND business.domain="crypto-trading"
```

### Track Multi-Service Process
```bash
# Jaeger query  
business.process_name="yield_optimization" AND business.workflow_id="workflow-123"
```

## Error Handling Requirements

### Structured Error Context
```typescript
try {
  await this.executeYieldStrategy(data);
} catch (error) {
  // ENFORCED: Errors automatically include business context
  this.logger.error('Yield strategy execution failed', {
    strategy_id: data.strategyId,
    user_id: data.userId,
    protocol: data.protocol,
    error: error.message,
    correlation_id: context.correlationId
  }, error);
  
  // Framework automatically adds error to trace with business context
  throw error; // Error bubbles up with full context
}
```

## Performance Monitoring

### Automatic SLA Tracking
```typescript
// Framework automatically tracks:
// - Business process completion times
// - Cross-service call latencies  
// - Event processing performance
// - Error rates by business domain
// - Resource utilization per workflow
```

## Development Workflow

### 1. Create Service
```typescript
// Extend appropriate base class
class MyService extends CryptoTradingServiceBase {
  // Framework provides all infrastructure
}
```

### 2. Define Operations  
```typescript
async myBusinessOperation(request: MyRequest): Promise<MyResult> {
  // Build business context
  const context = BusinessContextBuilder.forCryptoTrading()...;
  
  // Use enforced pattern
  return this.executeBusinessOperation('my_operation', context, async (ctx) => {
    // Your logic here
    return result;
  });
}
```

### 3. Handle Events
```typescript
async handleSomeEvent(event: SomeEvent): Promise<void> {
  // Use enforced pattern  
  return this.handleDomainEvent(event, async (event, context) => {
    // Your event handling logic
  });
}
```

### 4. Debug Issues
- Open Jaeger UI: http://localhost:16686
- Search by user ID, workflow ID, or business domain
- See complete end-to-end traces with business context
- Correlate technical failures to business impact

## Benefits

### For Developers
- ✅ **No tracing setup required** - automatic with base classes
- ✅ **Consistent patterns** - same approach across all services
- ✅ **Type safety** - compile-time validation of business context
- ✅ **Auto-completion** - IDE guides proper usage

### For Operations  
- ✅ **Perfect debugging** - trace any business process end-to-end
- ✅ **Business metrics** - SLAs tracked automatically
- ✅ **Error correlation** - link technical failures to business impact
- ✅ **Performance optimization** - identify bottlenecks by business process

### For Product
- ✅ **User journey tracking** - see complete user experience
- ✅ **Feature performance** - measure business process success rates
- ✅ **Issue resolution** - faster debugging means better user experience
- ✅ **Data insights** - rich business context for analytics

## Compliance

### MANDATORY Requirements
- ✅ All services MUST extend base classes
- ✅ All business operations MUST use business context
- ✅ All domain events MUST follow naming conventions  
- ✅ All cross-service calls MUST use framework methods
- ✅ All error handling MUST preserve business context

### Code Review Checklist
- [ ] Service extends appropriate base class?
- [ ] Business context provided for all operations?
- [ ] Domain events use proper naming convention?
- [ ] Cross-service calls use sendCommand pattern?
- [ ] Event handlers use handleDomainEvent wrapper?
- [ ] Error handling preserves business context?

## Examples

### Complete Trading Service Example
```typescript
class TradingService extends CryptoTradingServiceBase {
  async executeArbitrageStrategy(request: ArbitrageRequest): Promise<ArbitrageResult> {
    // 1. Build business context
    const context = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('ArbitrageTrade', request.tradeId)
      .withUser(request.userId)
      .withTradingPair(request.tradingPair)
      .withAmount(request.amountUsd)
      .withWorkflow(`arbitrage-${request.tradeId}`)
      .withMetadata('strategy', 'cross-exchange-arbitrage')
      .build();

    // 2. Execute with enforced pattern
    return this.executeBusinessOperation('arbitrage_execution', context, async (ctx) => {
      // 3. Multi-step process with automatic tracing
      return this.executeBusinessProcess('arbitrage_strategy', context, [
        {
          name: 'detect_opportunity',
          execute: async (ctx) => {
            const opportunity = await this.detectArbitrageOpportunity(request.tradingPair);
            
            await this.publishDomainEvent(
              'ArbitrageOpportunityDetected',
              `arbitrage-${request.tradeId}`,
              opportunity,
              context,
              ctx
            );
            
            return opportunity;
          }
        },
        {
          name: 'assess_risk',
          execute: async (ctx, opportunity) => {
            const riskAssessment = await this.sendCommand<any, RiskAssessment>(
              'risk-assessment',
              'AssessArbitrageRisk', 
              { opportunity, tradingPair: request.tradingPair },
              context
            );
            
            return riskAssessment;
          }
        },
        {
          name: 'execute_trades',
          execute: async (ctx, riskAssessment) => {
            if (riskAssessment.riskLevel === 'high') {
              throw new Error('Risk level too high for execution');
            }
            
            const result = await this.executeCrossExchangeTrades(request, riskAssessment);
            
            await this.publishDomainEvent(
              'ArbitrageTradeExecuted',
              `arbitrage-${request.tradeId}`,
              result,
              context,
              ctx
            );
            
            return result;
          }
        }
      ]);
    });
  }

  // Event handler example
  async handleMarketPriceUpdate(event: MarketPriceUpdatedEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      // Check for new arbitrage opportunities
      if (this.shouldCheckArbitrage(event.data)) {
        const opportunities = await this.scanForArbitrageOpportunities(event.data.tradingPair);
        
        for (const opportunity of opportunities) {
          await this.publishDomainEvent(
            'ArbitrageOpportunityDetected',
            `arbitrage-opportunity-${Date.now()}`,
            opportunity,
            context.businessContext,
            context
          );
        }
      }
    });
  }
}
```

### Complete Event Flow Example

```typescript
// 1. Market data arrives via Kafka → ClickHouse
// 2. Price update triggers arbitrage detection
class ArbitrageDetectionService extends CryptoTradingServiceBase {
  async detectOpportunities(priceUpdate: PriceUpdateEvent): Promise<void> {
    const context = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('ArbitrageAnalysis', `analysis-${Date.now()}`)
      .withTradingPair(priceUpdate.tradingPair)
      .withWorkflow(`arbitrage-detection-${Date.now()}`)
      .build();

    return this.executeBusinessOperation('detect_arbitrage', context, async (ctx) => {
      const opportunities = await this.findArbitrageOpportunities(priceUpdate);
      
      for (const opportunity of opportunities) {
        await this.publishDomainEvent(
          'ArbitrageOpportunityDetected', 
          `arbitrage-${opportunity.id}`,
          opportunity,
          context,
          ctx
        );
      }
    });
  }
}

// 3. Arbitrage opportunity triggers risk assessment
class RiskAssessmentService extends CryptoTradingServiceBase {
  async handleArbitrageOpportunity(event: ArbitrageOpportunityDetectedEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      const assessment = await this.assessArbitrageRisk(event.data);
      
      await this.publishDomainEvent(
        'ArbitrageRiskAssessed',
        `risk-${event.data.opportunityId}`,
        assessment,
        context.businessContext,
        context
      );
    });
  }
}

// 4. Risk assessment triggers trade execution
class TradingExecutionService extends CryptoTradingServiceBase {
  async handleArbitrageRiskAssessed(event: ArbitrageRiskAssessedEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      if (event.data.riskLevel === 'low') {
        const trades = await this.executeArbitrageTrades(event.data);
        
        await this.publishDomainEvent(
          'ArbitrageTradesExecuted',
          `trades-${event.data.opportunityId}`, 
          trades,
          context.businessContext,
          context
        );
      }
    });
  }
}
```

## Debugging Examples

### Find Complete User Journey
```bash
# In Jaeger UI, search:
business.user_id="user-123"

# Results show:
1. UserCreated → user-management service
2. PortfolioCreated → portfolio service  
3. YieldOpportunityDetected → yield-farming service
4. RiskAssessed → risk-assessment service
5. TradeExecuted → trading service
6. PortfolioUpdated → portfolio service
```

### Debug Failed Arbitrage Strategy
```bash
# Search by workflow:
business.workflow_id="arbitrage-workflow-456"

# See complete process:
Step 1: detect_opportunity ✅ 50ms
Step 2: assess_risk ✅ 200ms  
Step 3: execute_trades ❌ FAILED - "Insufficient liquidity"
  ↳ Error details: Exchange XYZ liquidity too low
  ↳ Business impact: $50,000 arbitrage opportunity missed
  ↳ Recommended action: Increase liquidity threshold
```

### Performance Analysis
```bash
# Find slow yield farming processes:
business.process_name="yield_optimization" AND operation.duration_ms>30000

# Results:
- 95% complete under 15 seconds ✅
- 4% take 15-30 seconds ⚠️ 
- 1% timeout after 30 seconds ❌ (investigate ClickHouse query performance)
```

## Summary

This opinionated framework **enforces world-class patterns** while providing **effortless debugging** for event-driven architecture. Every business process becomes traceable, every error includes business context, and every performance issue can be correlated to business impact.

**Compliance with these patterns is MANDATORY** for all Lucent Services to ensure operational excellence and debugging capabilities that surpass any major tech company.