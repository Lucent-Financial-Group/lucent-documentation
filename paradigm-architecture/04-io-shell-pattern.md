# I/O Shell Pattern (OOP at the Edges)

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Object-oriented infrastructure layer for external integrations

## Overview

The I/O Shell Pattern implements the **imperative shell** of our functional architecture using clean, professional service classes. This layer handles all side effects, external integrations, and infrastructure concerns while providing a clean interface to the functional core.

## Service Class Architecture

### Single Professional Service Classes

```typescript
import { CryptoTradingServiceBase, BusinessContextBuilder } from 'lucent-infrastructure';

/**
 * Portfolio service handles all portfolio-related I/O operations
 */
export class PortfolioService extends CryptoTradingServiceBase {
  constructor() {
    const infrastructure = await InfrastructureFactory.createDefault();
    super(infrastructure, 'portfolio-service');
  }

  /**
   * Handle trade execution events with complete I/O coordination
   */
  async handleTradeExecuted(event: TradeExecutedEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      // 1. Read current state from EventStore (I/O)
      const portfolioEvents = await this.infrastructure.eventStore.read(`portfolio-${event.data.userId}`);
      const userRiskProfile = await this.getUserRiskProfile(event.data.userId);
      const regulatoryLimits = await this.getRegulatoryLimits(event.data.userId);
      
      // 2. Send calculation request to Function Manager (I/O)
      const portfolioUpdate = await this.sendCommand(
        'function-manager',
        'ExecuteFunction',
        {
          functionType: 'calculatePortfolioUpdate',
          eventType: 'PortfolioUpdateRequested',
          data: {
            currentEvents: portfolioEvents,
            newTrade: event.data,
            userRiskProfile: userRiskProfile,
            regulatoryLimits: regulatoryLimits,
            timestamp: Date.now()
          }
        },
        context.businessContext
      );

      if (!portfolioUpdate.success) {
        await this.publishDomainEvent('PortfolioUpdateFailed', `portfolio-${event.data.userId}`, portfolioUpdate.error, context.businessContext, context);
        return;
      }
      
      // 3. Store result in EventStore (I/O)
      await this.publishDomainEvent(
        'PortfolioUpdated',
        `portfolio-${event.data.userId}`,
        portfolioUpdate.result,
        context.businessContext,
        context
      );
      
      // 4. Update Redis projection (I/O)
      await this.updatePortfolioProjection(event.data.userId, portfolioUpdate.result);
      
      // 5. Send notifications if needed (I/O)
      if (portfolioUpdate.result.riskLevelChanged) {
        await this.sendCommand(
          'notification-service',
          'SendRiskAlert',
          {
            userId: event.data.userId,
            oldRisk: portfolioUpdate.result.previousRiskLevel,
            newRisk: portfolioUpdate.result.newRiskLevel
          },
          context.businessContext
        );
      }
    });
  }

  /**
   * External API integration with resilience (built into base class)
   */
  private async getUserRiskProfile(userId: string): Promise<RiskProfile> {
    return this.executeUserOperation('get_risk_profile', userId, async (context) => {
      
      // Base class provides circuit breaker, retry logic, monitoring
      const response = await this.httpClient.get(`/users/${userId}/risk-profile`);
      
      // Transform data using Function Manager
      const riskProfile = await this.sendCommand(
        'function-manager',
        'ExecuteFunction',
        {
          functionType: 'parseRiskProfile',
          eventType: 'RiskProfileParseRequested',
          data: response.data
        },
        context.businessContext
      );
      
      return riskProfile.result;
    });
  }

  /**
   * Update portfolio cache with domain-specific abstraction
   */
  private async updatePortfolioProjection(userId: string, portfolio: Portfolio): Promise<void> {
    await this.updatePortfolioCache({
      userId,
      positions: portfolio.positions,
      totalValue: portfolio.totalValue,
      lastUpdated: Date.now(),
      version: portfolio.version
    });
  }

  /**
   * Domain-specific portfolio cache operations
   */
  private async updatePortfolioCache(update: PortfolioUpdate): Promise<void> {
    const pipeline = this.infrastructure.cacheStore.pipeline();
    
    // Abstract Redis implementation behind clean domain methods
    pipeline
      .hset(this.getPositionsKey(update.userId), 'data', JSON.stringify(update.positions))
      .hset(this.getMetadataKey(update.userId), 'totalValue', update.totalValue.toString())
      .hset(this.getMetadataKey(update.userId), 'lastUpdated', update.lastUpdated.toString())
      .hset(this.getMetadataKey(update.userId), 'version', update.version.toString())
      .zadd('portfolio-rankings', update.totalValue, update.userId);

    await pipeline.exec();
    
    this.logger.debug('Portfolio cache updated', {
      user_id: update.userId,
      total_value: update.totalValue,
      position_count: Object.keys(update.positions).length
    });
  }

  async getPortfolioFromCache(userId: string): Promise<Portfolio | null> {
    const [positions, metadata] = await Promise.all([
      this.infrastructure.cacheStore.hget(this.getPositionsKey(userId), 'data'),
      this.infrastructure.cacheStore.hgetall(this.getMetadataKey(userId))
    ]);

    if (!positions || !metadata.totalValue) {
      return null;
    }

    return {
      userId,
      positions: JSON.parse(positions),
      totalValue: parseFloat(metadata.totalValue),
      lastUpdated: new Date(parseInt(metadata.lastUpdated)),
      version: parseInt(metadata.version)
    };
  }

  async addToPortfolioRanking(userId: string, totalValue: number): Promise<void> {
    await this.infrastructure.cacheStore.zadd('portfolio-rankings', totalValue, userId);
  }

  async getPortfolioRanking(userId: string): Promise<number | null> {
    const rank = await this.infrastructure.cacheStore.zrevrank('portfolio-rankings', userId);
    return rank !== null ? rank + 1 : null; // Convert to 1-based ranking
  }

  // Clean key generation methods
  private getPositionsKey(userId: string): string {
    return `portfolio:${userId}:positions`;
  }

  private getMetadataKey(userId: string): string {
    return `portfolio:${userId}:metadata`;
  }

  /**
   * Batch operations for high-frequency updates
   */
  async processTradeBatch(trades: TradeExecutedEvent[]): Promise<void> {
    const businessContext = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('TradeBatch', `batch-${Date.now()}`)
      .withMetadata('batch_size', trades.length)
      .build();

    return this.executeBusinessOperation('process_trade_batch', businessContext, async (context) => {
      
      // Process all trades in parallel through Function Manager
      const batchRequest = {
        functionType: 'processTradeBatch',
        eventType: 'TradeBatchRequested',
        data: {
          trades: trades.map(t => t.data),
          batchId: context.correlationId,
          timestamp: Date.now()
        }
      };

      const batchResult = await this.sendCommand('function-manager', 'ExecuteFunction', batchRequest, context.businessContext);
      
      if (batchResult.success) {
        // Update Redis projections in batch
        const pipeline = this.infrastructure.cacheStore.pipeline();
        
        for (const result of batchResult.result.portfolioUpdates) {
          pipeline
            .hset(`portfolio:${result.userId}:positions`, 'data', JSON.stringify(result.positions))
            .hset(`portfolio:${result.userId}:metadata`, 'totalValue', result.totalValue.toString())
            .zadd('portfolio-rankings', result.totalValue, result.userId);
        }

        await pipeline.exec();
      }
    });
  }

  /**
   * Recovery operations (built into service, not separate class)
   */
  async recoverPortfolioProjection(userId: string): Promise<void> {
    return this.executeUserOperation('recover_portfolio', userId, async (context) => {
      
      // Get complete event history
      const allEvents = await this.infrastructure.eventStore.read(`user-${userId}`);
      
      // Send recovery request to Function Manager
      const recoveryResult = await this.sendCommand(
        'function-manager',
        'ExecuteFunction',
        {
          functionType: 'rebuildPortfolioFromEvents',
          eventType: 'PortfolioRecoveryRequested',
          data: {
            userId,
            events: allEvents,
            recoveryTimestamp: Date.now()
          }
        },
        context.businessContext
      );

      if (recoveryResult.success) {
        // Store rebuilt projection
        await this.updatePortfolioProjection(userId, recoveryResult.result.portfolio);
        
        // Mark recovery complete
        await this.infrastructure.cacheStore.hset(
          `portfolio:${userId}:metadata`,
          'recovered_at',
          Date.now().toString()
        );
      }
    });
  }
}

```

**Note**: The complete Function Manager implementation is shown in **03-function-manager.md** with both domain event processing and service request handling capabilities.

## Professional Class Naming

### Clean Service Classes
- ✅ `PortfolioService` (not `ResilientPortfolioService` or `EnhancedPortfolioService`)
- ✅ `TradingService` (not `AdvancedTradingService`)  
- ✅ `RiskService` (not `StatefulRiskService`)
- ✅ `AnalyticsService` (not `HighFrequencyAnalyticsService`)

### Single Function Manager
- ✅ `FunctionManager` (not `GenericFunctionManager`, `LoadBalancedFunctionManager`, etc.)
- **Handles everything**: Load balancing, state management, context bridging, projections
- **No separate classes** for each capability

### No Unnecessary Abstractions
- ❌ Remove: `FunctionContextBridge`, `ResilientProjectionService`, `StatefulFunctionManager`
- ✅ **All capabilities built into base classes** and Function Manager
- ✅ **Professional naming** without adjectives

## Key Principles

### One Class Per Business Responsibility
- **PortfolioService**: All portfolio-related I/O
- **TradingService**: All trading-related I/O
- **FunctionManager**: All function orchestration
- **No specialized variants** with confusing names

### Capabilities Built In (Not Separate)
- **Resilience**: Built into base classes (circuit breakers, retry logic)
- **State management**: Built into Function Manager
- **Context creation**: Built into Function Manager  
- **Load balancing**: Built into Function Manager
- **Projection updates**: Built into Function Manager

This clean approach eliminates confusion and provides **professional, maintainable code** with clear responsibilities.