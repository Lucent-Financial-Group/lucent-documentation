# Service Read Models with Redis

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Service-specific computed projections and caching patterns

## Overview

Service Read Models provide **fast, service-specific views** of data computed from domain events. Each service maintains its own Redis-based read models that are updated asynchronously as events arrive, enabling **sub-millisecond queries** while maintaining **eventual consistency** with the event store.

## Architecture Pattern

### Event-Driven Read Model Updates
```
Domain Events → EventStore → Service Event Handlers → Redis Read Models
                    ↓
             Source of Truth         Fast Query Layer
```

### Service Isolation
```
Portfolio Service: portfolio:*, portfolio-history:*
Risk Service:     user-risk:*, user-exposures-*, risk-alerts
Analytics Service: latest-prices, volume-24h, price-changes-24h
Function Manager: function-load:*, shard-health, active-functions:*
```

## Redis Provider Usage

### Pure Function Cache Operations
```typescript
import { EnhancedFunctionContext } from 'lucent-infrastructure';

/**
 * Pure function for updating portfolio read model
 */
@EventHandler('TradeExecuted')
@ShardBy('user_id')
export function updatePortfolioProjection(
  context: EnhancedFunctionContext,
  event: TradeExecutedEventData
): PortfolioUpdateResult {
  
  context.logger.debug('Updating portfolio projection', {
    user_id: event.userId,
    trade_id: event.tradeId,
    pair: event.pair,
    side: event.side,
    amount: event.amount
  });

  // Pure calculation logic
  const positionChange = event.side === 'buy' ? event.amount : -event.amount;
  const valueChange = event.side === 'buy' ? event.amountUsd : -event.amountUsd;
  
  // Calculate new portfolio state (pure function)
  const portfolioUpdate = {
    userId: event.userId,
    positionUpdates: {
      [event.pair]: positionChange
    },
    valueChange: valueChange,
    lastTradeId: event.tradeId,
    updatedAt: Date.now()
  };

  // Context provides cache update capabilities
  context.emit('PortfolioProjectionUpdated', portfolioUpdate);
  
  context.addMetadata('portfolio.position_change', positionChange);
  context.addMetadata('portfolio.value_change', valueChange);
  
  return portfolioUpdate;
}

/**
 * Function Manager handles I/O and coordinates pure functions
 */
class PortfolioFunctionManager extends LucentServiceBase {
  constructor() {
    super(infrastructure, 'portfolio-function-manager');
  }

  async processTradeEvent(event: DomainEvent<TradeExecutedEventData>): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      // 1. Execute pure function for portfolio calculation
      const functionContext = this.createEnhancedContext(context);
      const portfolioUpdate = updatePortfolioProjection(functionContext, event.data);
      
      // 2. Handle I/O - update Redis read model
      await this.updateRedisReadModel(portfolioUpdate);
      
      // 3. Handle I/O - publish domain event if needed
      if (this.significantPortfolioChange(portfolioUpdate)) {
        await this.publishDomainEvent(
          'PortfolioSignificantlyUpdated',
          `portfolio-${portfolioUpdate.userId}`,
          portfolioUpdate,
          context.businessContext,
          context
        );
      }
    });
  }

  private async updateRedisReadModel(update: PortfolioUpdateResult): Promise<void> {
    // I/O Operation: Update Redis with computed projection
    const pipeline = this.infrastructure.cacheStore.pipeline();
    
    // Get current positions
    const currentPositions = await this.infrastructure.cacheStore.hget(
      `portfolio:${update.userId}:positions`,
      'data'
    );
    
    const positions = currentPositions ? JSON.parse(currentPositions) : {};
    
    // Apply position updates
    for (const [pair, change] of Object.entries(update.positionUpdates)) {
      const current = positions[pair] || 0;
      const newAmount = current + change;
      
      if (Math.abs(newAmount) < 0.000001) {
        delete positions[pair];
      } else {
        positions[pair] = newAmount;
      }
    }

    // Update cache with clean abstraction
    await this.updatePortfolioProjection({
      userId: update.userId,
      positions: positions,
      totalValue: update.valueChange,
      lastTradeId: update.lastTradeId,
      lastUpdated: update.updatedAt
    });
  }
}

  async getPortfolio(userId: string): Promise<Portfolio | null> {
    // Fast read from Redis
    const portfolioData = await this.infrastructure.cacheStore.hgetall(`portfolio:${userId}`);
    
    if (Object.keys(portfolioData).length === 0) {
      return null; // Not cached
    }

    return {
      userId,
      positions: JSON.parse(portfolioData.positions),
      totalValue: parseFloat(portfolioData.totalValue),
      riskLevel: portfolioData.riskLevel as RiskLevel,
      lastUpdated: new Date(portfolioData.lastUpdated)
    };
  }
}
```

### Advanced Redis Data Structures

#### Hash Maps for Complex Objects
```typescript
// Portfolio positions with metadata
await this.infrastructure.cacheStore.hmset(`portfolio:${userId}`, {
  'ETH-USD': '2.5',
  'BTC-USD': '0.1',
  'totalValue': '75000',
  'riskScore': '3.2',
  'lastRebalanced': new Date().toISOString()
});

// Fast field access
const ethPosition = await this.infrastructure.cacheStore.hget(`portfolio:${userId}`, 'ETH-USD');
const totalValue = await this.infrastructure.cacheStore.hget(`portfolio:${userId}`, 'totalValue');
```

#### Sorted Sets for Rankings and Leaderboards
```typescript
// Clean domain-specific ranking operations
await this.addToVolumeLeaderboard(userId, dailyVolume);
await this.addToStrategyPerformance(strategyId, roi);

// Get top performers with clean abstraction
const topTraders = await this.getTopVolumeTraders(10);
const userRank = await this.getUserVolumeRank(userId);

/**
 * Clean volume ranking operations
 */
private async addToVolumeLeaderboard(userId: string, volume: number): Promise<void> {
  await this.infrastructure.cacheStore.zadd('daily-volume-leaders', volume, userId);
}

private async getTopVolumeTraders(limit: number): Promise<VolumeRanking[]> {
  const topUsers = await this.infrastructure.cacheStore.zrevrange('daily-volume-leaders', 0, limit - 1);
  
  return topUsers.map((userId, index) => ({
    userId,
    rank: index + 1,
    volume: 0 // Would get actual volume from zscore
  }));
}

private async getUserVolumeRank(userId: string): Promise<number | null> {
  const rank = await this.infrastructure.cacheStore.zrevrank('daily-volume-leaders', userId);
  return rank !== null ? rank + 1 : null;
}
```

#### Sets for Unique Collections
```typescript
// Active users
await this.infrastructure.cacheStore.sadd('active-users-today', userId);

// Users with alerts
await this.infrastructure.cacheStore.sadd('users-with-alerts', userId);

// Check if user is active
const isActive = await this.infrastructure.cacheStore.sismember('active-users-today', userId);
```

## Service-Specific Read Model Patterns

### 1. Portfolio Service Read Models

#### Portfolio State Management
```typescript
class PortfolioService extends CryptoTradingServiceBase {
  
  async handleTradeExecuted(event: TradeExecutedEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      // 1. Update portfolio positions
      await this.updatePortfolioPositions(event.data.userId, event.data);
      
      // 2. Recalculate portfolio value
      const newValue = await this.recalculatePortfolioValue(event.data.userId);
      
      // 3. Update portfolio metrics
      await this.updatePortfolioMetrics(event.data.userId, newValue);
      
      // 4. Check rebalancing triggers
      await this.checkRebalancingTriggers(event.data.userId);
    });
  }

  private async updatePortfolioPositions(userId: string, trade: TradeData): Promise<void> {
    // Get current position
    const currentPosition = await this.infrastructure.cacheStore.hget(
      `portfolio:${userId}:positions`, 
      trade.pair
    );
    
    const current = parseFloat(currentPosition || '0');
    const newPosition = trade.side === 'buy' 
      ? current + trade.amount 
      : current - trade.amount;

    if (Math.abs(newPosition) < 0.000001) {
      // Remove zero positions
      await this.infrastructure.cacheStore.hdel(`portfolio:${userId}:positions`, trade.pair);
    } else {
      // Update position
      await this.infrastructure.cacheStore.hset(
        `portfolio:${userId}:positions`, 
        trade.pair, 
        newPosition.toString()
      );
    }

    // Track position history for analytics
    await this.infrastructure.cacheStore.zadd(
      `portfolio:${userId}:history`,
      Date.now(),
      JSON.stringify({ pair: trade.pair, position: newPosition, timestamp: Date.now() })
    );

    // Keep only last 1000 history entries
    await this.infrastructure.cacheStore.zremrangebyrank(`portfolio:${userId}:history`, 0, -1001);
  }

  async getPortfolioSummary(userId: string): Promise<PortfolioSummary> {
    // Parallel Redis queries for fast response
    const [positions, metadata, history] = await Promise.all([
      this.infrastructure.cacheStore.hgetall(`portfolio:${userId}:positions`),
      this.infrastructure.cacheStore.hgetall(`portfolio:${userId}:metadata`), 
      this.infrastructure.cacheStore.zrevrange(`portfolio:${userId}:history`, 0, 9) // Last 10 entries
    ]);

    return {
      userId,
      positions: this.parsePositions(positions),
      totalValue: parseFloat(metadata.totalValue || '0'),
      riskScore: parseFloat(metadata.riskScore || '0'),
      lastUpdated: new Date(metadata.lastUpdated || Date.now()),
      recentHistory: history.map(h => JSON.parse(h))
    };
  }
}
```

### 2. Risk Service Read Models

#### Risk Assessment and Monitoring
```typescript
class RiskService extends CryptoTradingServiceBase {
  
  async handleTradeExecuted(event: TradeExecutedEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      // Update user risk metrics
      await this.updateUserRiskProfile(event.data.userId, event.data);
      
      // Update global risk indicators
      await this.updateGlobalRiskMetrics(event.data);
      
      // Check risk thresholds
      const newRiskLevel = await this.assessRiskLevel(event.data.userId);
      
      if (newRiskLevel !== await this.getCurrentRiskLevel(event.data.userId)) {
        await this.publishRiskLevelChange(event.data.userId, newRiskLevel, context);
      }
    });
  }

  private async updateUserRiskProfile(userId: string, trade: TradeData): Promise<void> {
    const pipeline = this.infrastructure.cacheStore.pipeline();
    
    // Update exposure tracking
    pipeline.zincrby('user-daily-exposure', trade.amountUsd, userId);
    pipeline.zincrby('user-total-exposure', trade.amountUsd, userId);
    
    // Update trade frequency (for risk scoring)
    pipeline.incr(`user-trade-count:${userId}:${this.getDateKey()}`);
    
    // Update risk metrics with clean abstraction
    const riskScore = await this.calculateRiskScore(userId, trade);
    
    await this.updateUserRiskMetrics({
      userId,
      newRiskScore: riskScore,
      largestTrade: trade.amountUsd,
      lastUpdated: Date.now()
    });
  }

  async getHighRiskUsers(): Promise<UserRiskProfile[]> {
    // Get users with high exposure
    const highExposureUsers = await this.infrastructure.cacheStore.zrevrangebyscore(
      'user-daily-exposure', 
      '+inf', 
      '100000', // Above $100K daily
      { LIMIT: { offset: 0, count: 50 } }
    );

    const riskProfiles: UserRiskProfile[] = [];

    for (const userId of highExposureUsers) {
      const riskData = await this.infrastructure.cacheStore.hgetall(`user-risk:${userId}`);
      const exposure = await this.infrastructure.cacheStore.zscore('user-daily-exposure', userId);
      
      riskProfiles.push({
        userId,
        dailyExposure: exposure || 0,
        riskScore: parseFloat(riskData.riskScore || '0'),
        largestTrade: parseFloat(riskData.largestTrade || '0'),
        lastUpdated: new Date(parseInt(riskData.lastUpdated || '0'))
      });
    }

    return riskProfiles.sort((a, b) => b.riskScore - a.riskScore);
  }
}
```

### 3. Analytics Service Read Models

#### Real-Time Market Analytics
```typescript
class AnalyticsService extends CryptoTradingServiceBase {
  
  async handlePriceUpdate(event: PriceUpdateEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      // Update real-time price tracking
      await this.updatePriceMetrics(event.data);
      
      // Update volume rankings
      await this.updateVolumeRankings(event.data);
      
      // Detect and cache arbitrage opportunities
      await this.detectAndCacheArbitrageOpportunities(event.data);
    });
  }

  private async updatePriceMetrics(priceData: PriceData): Promise<void> {
    const pipeline = this.infrastructure.cacheStore.pipeline();
    
    // Latest price per exchange
    pipeline.hset(
      'latest-prices',
      `${priceData.exchange}:${priceData.pair}`,
      JSON.stringify({
        price: priceData.price,
        timestamp: priceData.timestamp,
        volume: priceData.volume
      })
    );

    // Price change calculation
    const previousPrice = await this.getPreviousPrice(priceData.exchange, priceData.pair);
    if (previousPrice) {
      const changePercent = ((priceData.price - previousPrice) / previousPrice) * 100;
      pipeline.hset('price-changes-1h', `${priceData.exchange}:${priceData.pair}`, changePercent.toString());
    }

    // Volume tracking for the hour
    pipeline.zincrby(`volume-1h:${this.getHourKey()}`, priceData.volume, priceData.pair);
    
    await pipeline.exec();
  }

  async getMarketOverview(): Promise<MarketOverview> {
    // Parallel queries for dashboard data
    const [topGainers, topLosers, volumeLeaders, latestPrices] = await Promise.all([
      this.getTopMovers('price-changes-1h', 'gainers', 10),
      this.getTopMovers('price-changes-1h', 'losers', 10),
      this.infrastructure.cacheStore.zrevrange(`volume-1h:${this.getHourKey()}`, 0, 19),
      this.infrastructure.cacheStore.hgetall('latest-prices')
    ]);

    return {
      topGainers,
      topLosers,
      volumeLeaders: volumeLeaders.map((pair, index) => ({ pair, rank: index + 1 })),
      totalPairs: Object.keys(latestPrices).length,
      lastUpdated: new Date()
    };
  }

  private async detectAndCacheArbitrageOpportunities(priceData: PriceData): Promise<void> {
    // Get prices from all exchanges for this pair
    const allPrices = await this.infrastructure.cacheStore.hgetall('latest-prices');
    const pairPrices: ExchangePrice[] = [];

    Object.entries(allPrices).forEach(([exchangePair, data]) => {
      const [exchange, pair] = exchangePair.split(':');
      if (pair === priceData.pair) {
        const parsed = JSON.parse(data);
        pairPrices.push({ exchange, price: parsed.price, timestamp: parsed.timestamp });
      }
    });

    if (pairPrices.length >= 2) {
      // Find price spreads
      pairPrices.sort((a, b) => a.price - b.price);
      const cheapest = pairPrices[0];
      const mostExpensive = pairPrices[pairPrices.length - 1];
      
      const spread = (mostExpensive.price - cheapest.price) / cheapest.price;
      
      if (spread > 0.005) { // 0.5% minimum arbitrage opportunity
        const arbitrageOpportunity = {
          pair: priceData.pair,
          buyExchange: cheapest.exchange,
          sellExchange: mostExpensive.exchange,
          spread,
          profitPotential: spread * 0.8, // Account for fees
          timestamp: Date.now()
        };

        // Cache arbitrage opportunity with short TTL
        await this.infrastructure.cacheStore.set(
          `arbitrage:${priceData.pair}:${Date.now()}`,
          JSON.stringify(arbitrageOpportunity),
          300 // 5 minutes
        );

        // Add to sorted set for ranking
        await this.infrastructure.cacheStore.zadd(
          'arbitrage-opportunities',
          spread,
          `${priceData.pair}:${cheapest.exchange}:${mostExpensive.exchange}`
        );
      }
    }
  }
}
```

## Service Read Model Patterns

### 1. Portfolio Service Patterns

#### Position Tracking
```typescript
class PortfolioService extends CryptoTradingServiceBase {
  
  /**
   * Handle trade execution and update portfolio read model
   */
  async handleTradeExecuted(event: TradeExecutedEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      const trade = event.data;
      
      // Atomic portfolio update using Redis transaction
      const multi = this.infrastructure.cacheStore.multi();
      
      // Get current position
      const currentPosition = await this.infrastructure.cacheStore.hget(
        `portfolio:${trade.userId}:positions`, 
        trade.pair
      );
      
      const current = parseFloat(currentPosition || '0');
      const newPosition = trade.side === 'buy' 
        ? current + trade.amount 
        : current - trade.amount;

      // Update position
      if (Math.abs(newPosition) < 0.000001) {
        multi.hdel(`portfolio:${trade.userId}:positions`, trade.pair);
      } else {
        multi.hset(`portfolio:${trade.userId}:positions`, trade.pair, newPosition.toString());
      }

      // Update portfolio metadata
      const newTotalValue = await this.calculateNewPortfolioValue(trade.userId, trade);
      multi
        .hset(`portfolio:${trade.userId}:metadata`, 'totalValue', newTotalValue.toString())
        .hset(`portfolio:${trade.userId}:metadata`, 'lastTradeTime', trade.executedAt)
        .hset(`portfolio:${trade.userId}:metadata`, 'tradeCount', 'INCREMENT') // Redis increment
        .hset(`portfolio:${trade.userId}:metadata`, 'lastUpdated', Date.now().toString());

      // Execute atomically
      await multi.exec();

      // Update global rankings
      await this.infrastructure.cacheStore.zadd('portfolio-values', newTotalValue, trade.userId);

      this.logger.info('Portfolio read model updated', {
        user_id: trade.userId,
        pair: trade.pair,
        new_position: newPosition,
        new_total_value: newTotalValue
      });
    });
  }

  /**
   * Fast portfolio queries
   */
  async getPortfolioSummary(userId: string): Promise<PortfolioSummary> {
    const [positions, metadata] = await Promise.all([
      this.infrastructure.cacheStore.hgetall(`portfolio:${userId}:positions`),
      this.infrastructure.cacheStore.hgetall(`portfolio:${userId}:metadata`)
    ]);

    return {
      userId,
      positions: Object.fromEntries(
        Object.entries(positions).map(([pair, amount]) => [pair, parseFloat(amount)])
      ),
      totalValue: parseFloat(metadata.totalValue || '0'),
      lastTradeTime: metadata.lastTradeTime,
      tradeCount: parseInt(metadata.tradeCount || '0'),
      lastUpdated: new Date(parseInt(metadata.lastUpdated || '0'))
    };
  }

  /**
   * Portfolio analytics queries
   */
  async getTopPortfolios(limit: number = 10): Promise<PortfolioRanking[]> {
    const topUsers = await this.infrastructure.cacheStore.zrevrange('portfolio-values', 0, limit - 1);
    
    const rankings: PortfolioRanking[] = [];
    
    for (const userId of topUsers) {
      const value = await this.infrastructure.cacheStore.zscore('portfolio-values', userId);
      const summary = await this.getPortfolioSummary(userId);
      
      rankings.push({
        userId,
        portfolioValue: value || 0,
        positionCount: Object.keys(summary.positions).length,
        rank: rankings.length + 1
      });
    }

    return rankings;
  }
}
```

### 2. Risk Service Patterns

#### Risk Monitoring and Alerting
```typescript
class RiskService extends CryptoTradingServiceBase {
  
  async handleTradeExecuted(event: TradeExecutedEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      const trade = event.data;
      
      // Update risk metrics
      await this.updateRiskMetrics(trade.userId, trade);
      
      // Check risk thresholds
      const currentRisk = await this.calculateCurrentRisk(trade.userId);
      
      if (currentRisk.level === 'high') {
        await this.triggerRiskAlert(trade.userId, currentRisk, context);
      }
    });
  }

  private async updateRiskMetrics(userId: string, trade: TradeData): Promise<void> {
    const pipeline = this.infrastructure.cacheStore.pipeline();
    
    // Daily trading volume (for risk calculation)
    const today = this.getTodayKey();
    pipeline.zincrby(`daily-volume:${today}`, trade.amountUsd, userId);
    
    // Position concentration risk
    pipeline.hincrby(`user-risk:${userId}`, `exposure:${trade.pair}`, trade.amountUsd);
    
    // Trade frequency risk
    pipeline.incr(`trade-frequency:${userId}:${today}`);
    pipeline.expire(`trade-frequency:${userId}:${today}`, 86400); // 24 hours
    
    // Update risk score components
    const dailyVolume = await this.infrastructure.cacheStore.zscore(`daily-volume:${today}`, userId) || 0;
    const tradeFreq = await this.infrastructure.cacheStore.get(`trade-frequency:${userId}:${today}`) || '0';
    
    const riskScore = this.calculateRiskScore({
      dailyVolume: dailyVolume + trade.amountUsd,
      tradeFrequency: parseInt(tradeFreq) + 1,
      largestSingleTrade: trade.amountUsd,
      portfolioConcentration: await this.getPortfolioConcentration(userId)
    });

    pipeline.hset(`user-risk:${userId}`, 'riskScore', riskScore.toString());
    pipeline.hset(`user-risk:${userId}`, 'lastCalculated', Date.now().toString());

    await pipeline.exec();
  }

  async getRiskDashboard(): Promise<RiskDashboard> {
    const today = this.getTodayKey();
    
    // Parallel queries for risk dashboard
    const [highRiskUsers, dailyVolumeLeaders, activeAlerts] = await Promise.all([
      this.infrastructure.cacheStore.zrevrangebyscore('user-risk-scores', '+inf', '7', { limit: 20 }),
      this.infrastructure.cacheStore.zrevrange(`daily-volume:${today}`, 0, 9),
      this.infrastructure.cacheStore.smembers('active-risk-alerts')
    ]);

    return {
      highRiskUsers: highRiskUsers.map((userId, index) => ({ userId, rank: index + 1 })),
      volumeLeaders: dailyVolumeLeaders.map((userId, index) => ({ userId, rank: index + 1 })),
      activeAlerts: activeAlerts.length,
      lastUpdated: new Date()
    };
  }
}
```

### 3. Analytics Service Patterns

#### Market Data Analytics
```typescript
class AnalyticsService extends CryptoTradingServiceBase {
  
  async handleMarketDataUpdate(event: MarketDataEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      const data = event.data;
      
      // Update real-time market metrics
      await this.updateMarketMetrics(data);
      
      // Update exchange performance tracking
      await this.updateExchangePerformance(data);
      
      // Update trading pair popularity
      await this.updateTradingPairMetrics(data);
    });
  }

  private async updateMarketMetrics(data: MarketData): Promise<void> {
    const pipeline = this.infrastructure.cacheStore.pipeline();
    
    // Latest price tracking
    pipeline.hset('latest-prices', `${data.exchange}:${data.pair}`, JSON.stringify({
      price: data.price,
      timestamp: data.timestamp,
      volume: data.volume,
      spread: data.spread || 0
    }));

    // 24-hour volume rankings
    pipeline.zincrby('volume-24h', data.volume, data.pair);
    pipeline.expire('volume-24h', 86400); // Reset daily
    
    // Exchange liquidity tracking
    pipeline.zincrby('exchange-liquidity', data.volume, data.exchange);
    
    // Volatility tracking (simplified)
    const volatilityKey = `volatility:${data.pair}:${this.getHourKey()}`;
    pipeline.zadd(volatilityKey, data.price, data.timestamp.toString());
    pipeline.expire(volatilityKey, 3600); // Keep hourly data

    await pipeline.exec();
  }

  async getMarketSummary(): Promise<MarketSummary> {
    // Real-time market overview from Redis
    const [
      topVolumeCoins,
      topExchanges, 
      latestPrices,
      totalVolume24h
    ] = await Promise.all([
      this.infrastructure.cacheStore.zrevrange('volume-24h', 0, 19), // Top 20
      this.infrastructure.cacheStore.zrevrange('exchange-liquidity', 0, 9), // Top 10
      this.infrastructure.cacheStore.hgetall('latest-prices'),
      this.getTotalVolume24h()
    ]);

    return {
      topCoins: topVolumeCoins.map((pair, index) => ({ 
        pair, 
        rank: index + 1,
        volume24h: 0 // Would get from zscore
      })),
      topExchanges: topExchanges.map((exchange, index) => ({ 
        exchange, 
        rank: index + 1 
      })),
      totalPairs: Object.keys(latestPrices).length,
      totalVolume24h,
      lastUpdated: new Date()
    };
  }
}
```

## Read Model Lifecycle Management

### Cache Warming Strategies
```typescript
class ReadModelManager extends CryptoTradingServiceBase {
  
  /**
   * Warm cache with essential data on startup
   */
  async warmCache(): Promise<void> {
    return this.executeBusinessOperation('warm_cache', this.createSystemContext(), async (context) => {
      
      this.logger.info('Starting cache warming process');

      // 1. Load active user portfolios
      await this.warmUserPortfolios();
      
      // 2. Load current market data  
      await this.warmMarketData();
      
      // 3. Load risk profiles for active users
      await this.warmRiskProfiles();

      this.logger.info('Cache warming completed');
    });
  }

  private async warmUserPortfolios(): Promise<void> {
    // Get active users from EventStore
    const activeUsers = await this.getActiveUsersFromEvents();
    
    for (const userId of activeUsers) {
      // Rebuild portfolio from events
      const portfolio = await this.rebuildPortfolioFromEvents(userId);
      
      // Cache portfolio
      await this.infrastructure.cacheStore.hmset(`portfolio:${userId}`, {
        'positions': JSON.stringify(portfolio.positions),
        'totalValue': portfolio.totalValue.toString(),
        'lastUpdated': Date.now().toString()
      });
    }
  }
}
```

### Cache Invalidation Patterns
```typescript
class CacheInvalidationService extends CryptoTradingServiceBase {
  
  /**
   * Intelligent cache invalidation based on data dependencies
   */
  async handleEventStoreProjectionUpdate(event: ProjectionUpdatedEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      const affectedKeys = this.determineAffectedCacheKeys(event.data);
      
      // Batch invalidation
      const pipeline = this.infrastructure.cacheStore.pipeline();
      
      for (const key of affectedKeys) {
        pipeline.del(key);
      }
      
      await pipeline.exec();

      this.logger.info('Cache invalidation completed', {
        projection: event.data.projectionName,
        invalidated_keys: affectedKeys.length
      });
    });
  }

  private determineAffectedCacheKeys(projectionData: any): string[] {
    const keys: string[] = [];
    
    switch (projectionData.projectionName) {
      case 'user-portfolio':
        keys.push(`portfolio:${projectionData.userId}`);
        keys.push(`portfolio:${projectionData.userId}:positions`);
        keys.push(`portfolio:${projectionData.userId}:metadata`);
        break;
        
      case 'market-analytics':
        keys.push('latest-prices');
        keys.push('volume-24h');
        keys.push('price-changes-1h');
        break;
        
      case 'risk-assessment':
        keys.push(`user-risk:${projectionData.userId}`);
        keys.push('user-risk-scores');
        break;
    }
    
    return keys;
  }
}
```

## Performance Optimization

### Pipeline Operations for Batch Updates
```typescript
class HighFrequencyUpdateService extends CryptoTradingServiceBase {
  
  /**
   * Handle high-frequency market data with batched Redis operations
   */
  async processBatchMarketData(events: MarketDataEvent[]): Promise<void> {
    const businessContext = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('MarketDataBatch', `batch-${Date.now()}`)
      .build();

    return this.executeBusinessOperation('process_batch_market_data', businessContext, async (context) => {
      
      // Use pipeline for efficient batching
      const pipeline = this.infrastructure.cacheStore.pipeline();
      
      for (const event of events) {
        const data = event.data;
        
        // Batch all updates
        pipeline
          .hset('latest-prices', `${data.exchange}:${data.pair}`, JSON.stringify(data))
          .zincrby('volume-1h', data.volume, data.pair)
          .zadd('price-updates', Date.now(), `${data.exchange}:${data.pair}:${data.price}`);
      }

      // Execute all operations at once
      const results = await pipeline.exec();
      
      this.logger.info('Batch market data processed', {
        events_processed: events.length,
        operations_executed: results.length,
        execution_time_ms: Date.now() - context.startTime.getTime()
      });
    });
  }
}
```

## Read Model Consistency

### Event-Driven Consistency
```typescript
class ReadModelConsistencyManager extends CryptoTradingServiceBase {
  
  /**
   * Ensure read model consistency with event store
   */
  async verifyReadModelConsistency(userId: string): Promise<ConsistencyReport> {
    return this.executeBusinessOperation('verify_consistency', this.createUserContext(userId), async (context) => {
      
      // 1. Get current read model from Redis
      const cachedPortfolio = await this.getPortfolioFromCache(userId);
      
      // 2. Rebuild from EventStore
      const eventSourcedPortfolio = await this.rebuildFromEventStore(userId);
      
      // 3. Compare for consistency
      const inconsistencies = this.comparePortfolios(cachedPortfolio, eventSourcedPortfolio);
      
      if (inconsistencies.length > 0) {
        this.logger.warn('Read model inconsistency detected', {
          user_id: userId,
          inconsistency_count: inconsistencies.length,
          inconsistencies
        });

        // Auto-repair: Update Redis with event store data
        await this.repairReadModel(userId, eventSourcedPortfolio);
      }

      return {
        userId,
        isConsistent: inconsistencies.length === 0,
        inconsistencies,
        lastChecked: new Date(),
        autoRepaired: inconsistencies.length > 0
      };
    });
  }
}
```

## Benefits

### Performance
- ✅ **Sub-millisecond queries**: Portfolio lookups in <1ms
- ✅ **Batch operations**: Pipeline/Multi for atomic updates
- ✅ **Memory efficiency**: TTL-based expiration management
- ✅ **Network optimization**: Connection pooling with resilience

### Service Isolation
- ✅ **Independent read models**: Each service manages its own cache keys
- ✅ **No cross-service dependencies**: Services don't share cache data
- ✅ **Service-specific optimizations**: Redis data structures per use case
- ✅ **Failure isolation**: One service's cache issues don't affect others

### Data Consistency
- ✅ **Event-driven updates**: Read models stay consistent with domain events
- ✅ **Atomic operations**: Redis transactions for complex updates
- ✅ **Eventual consistency**: Read models catch up with event store
- ✅ **Consistency verification**: Tools to detect and repair inconsistencies

### Observability
- ✅ **Full tracing**: Every cache operation traced with business context
- ✅ **Performance monitoring**: Cache hit rates, operation latencies
- ✅ **Cache analytics**: Usage patterns, memory utilization
- ✅ **Business metrics**: Read model update frequencies, consistency checks

## EventStore Projection Recovery

### Redis Failure Recovery Pattern

When Redis goes offline or read models become inconsistent, services can **rebuild projections from EventStore** using pure functions:

```typescript
/**
 * Pure function to rebuild portfolio from event history
 */
@EventHandler('PortfolioRecoveryRequested')
@ShardBy('user_id')
export function rebuildPortfolioFromEvents(
  context: EnhancedFunctionContext,
  data: PortfolioRecoveryData
): RebuiltPortfolio {
  
  context.logger.info('Rebuilding portfolio from EventStore', {
    user_id: data.userId,
    event_count: data.events.length,
    from_version: data.fromVersion
  });

  // Pure computation - replay all events
  let portfolio: PortfolioState = {
    userId: data.userId,
    positions: {},
    totalValue: 0,
    tradeCount: 0,
    lastUpdated: new Date(0)
  };

  for (const event of data.events) {
    // Apply each event to rebuild state
    portfolio = applyEventToPortfolio(portfolio, event, context);
  }

  // Calculate derived metrics
  portfolio.riskLevel = calculateRiskLevel(portfolio.totalValue);
  portfolio.diversificationScore = calculateDiversification(portfolio.positions);
  portfolio.lastUpdated = new Date();

  context.addMetadata('recovery.events_processed', data.events.length);
  context.addMetadata('recovery.final_position_count', Object.keys(portfolio.positions).length);
  context.addMetadata('recovery.final_total_value', portfolio.totalValue);

  return {
    portfolio,
    recovery: {
      eventsProcessed: data.events.length,
      recoveredAt: Date.now(),
      recoveryDuration: Date.now() - data.startTime,
      dataConsistent: true
    }
  };
}

/**
 * Pure helper function to apply single event to portfolio state
 */
function applyEventToPortfolio(
  portfolio: PortfolioState,
  event: DomainEvent,
  context: EnhancedFunctionContext
): PortfolioState {
  
  const updated = { ...portfolio };

  switch (event.eventType) {
    case 'TradeExecuted':
      const trade = event.data as TradeExecutedEventData;
      const positionChange = trade.side === 'buy' ? trade.amount : -trade.amount;
      
      updated.positions[trade.pair] = (updated.positions[trade.pair] || 0) + positionChange;
      updated.totalValue += trade.side === 'buy' ? trade.amountUsd : -trade.amountUsd;
      updated.tradeCount++;
      
      // Remove zero positions
      if (Math.abs(updated.positions[trade.pair]) < 0.000001) {
        delete updated.positions[trade.pair];
      }
      break;

    case 'PortfolioRebalanced':
      const rebalance = event.data as RebalanceEventData;
      updated.positions = { ...rebalance.newPositions };
      updated.totalValue = rebalance.newTotalValue;
      break;

    case 'PositionClosed':
      const closure = event.data as PositionClosureEventData;
      delete updated.positions[closure.pair];
      updated.totalValue -= closure.closureValue;
      break;
  }

  return updated;
}

/**
 * Function Manager coordinates recovery process
 */
class PortfolioRecoveryManager extends LucentServiceBase {
  
  async recoverPortfolioReadModel(userId: string): Promise<void> {
    const businessContext = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('PortfolioRecovery', `recovery-${userId}`)
      .withUser(userId)
      .withPriority('high')
      .build();

    return this.executeBusinessOperation('recover_portfolio', businessContext, async (context) => {
      
      this.logger.info('Starting portfolio recovery from EventStore', {
        user_id: userId,
        correlation_id: context.correlationId
      });

      // 1. Read all portfolio events from EventStore (I/O)
      const portfolioEvents = await this.infrastructure.eventStore.read(`portfolio-${userId}`);
      const tradeEvents = await this.infrastructure.eventStore.read(`user-${userId}`);
      
      const allEvents = [...portfolioEvents, ...tradeEvents]
        .sort((a, b) => new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime());

      // 2. Execute pure function to rebuild state
      const recoveryData: PortfolioRecoveryData = {
        userId,
        events: allEvents,
        fromVersion: 0,
        startTime: Date.now()
      };

      const functionContext = this.createEnhancedContext(context);
      const rebuilt = rebuildPortfolioFromEvents(functionContext, recoveryData);

      // 3. Store rebuilt projection in Redis (I/O)
      await this.storeRebuiltPortfolio(rebuilt.portfolio);

      // 4. Mark recovery complete
      await this.infrastructure.cacheStore.hset(
        `portfolio:${userId}:metadata`,
        'recovery_completed',
        Date.now().toString()
      );

      this.logger.info('Portfolio recovery completed', {
        user_id: userId,
        events_processed: rebuilt.recovery.eventsProcessed,
        recovery_duration_ms: rebuilt.recovery.recoveryDuration,
        final_total_value: rebuilt.portfolio.totalValue
      });
    });
  }

  private async storeRebuiltPortfolio(portfolio: PortfolioState): Promise<void> {
    // Store complete rebuilt state atomically
    const multi = this.infrastructure.cacheStore.multi();
    
    multi
      .hset(`portfolio:${portfolio.userId}:positions`, 'data', JSON.stringify(portfolio.positions))
      .hset(`portfolio:${portfolio.userId}:metadata`, 'totalValue', portfolio.totalValue.toString())
      .hset(`portfolio:${portfolio.userId}:metadata`, 'tradeCount', portfolio.tradeCount.toString())
      .hset(`portfolio:${portfolio.userId}:metadata`, 'riskLevel', portfolio.riskLevel)
      .hset(`portfolio:${portfolio.userId}:metadata`, 'lastUpdated', portfolio.lastUpdated.toISOString())
      .hset(`portfolio:${portfolio.userId}:metadata`, 'status', 'recovered');

    await multi.exec();
  }
}
```

### Automated Recovery Strategies

#### 1. Health Check-Triggered Recovery
```typescript
/**
 * Pure function to detect inconsistencies
 */
@EventHandler('HealthCheckFailed')
@ShardBy('service_name')
export function detectProjectionInconsistencies(
  context: EnhancedFunctionContext,
  data: HealthCheckData
): InconsistencyReport {
  
  context.logger.info('Detecting projection inconsistencies', {
    service: data.serviceName,
    last_health_check: data.lastHealthCheck
  });

  const inconsistencies: Inconsistency[] = [];
  
  // Check for stale read models
  if (data.redisLastUpdated < data.eventStoreLastEvent - 300000) { // 5 minutes stale
    inconsistencies.push({
      type: 'stale_projection',
      severity: 'warning',
      message: 'Redis projection is more than 5 minutes behind EventStore',
      affectedKeys: data.affectedKeys
    });
  }

  // Check for missing read models
  if (data.expectedReadModels > data.actualReadModels) {
    inconsistencies.push({
      type: 'missing_projections',
      severity: 'error',
      message: `Missing ${data.expectedReadModels - data.actualReadModels} read models`,
      affectedKeys: data.missingKeys
    });
  }

  return {
    serviceName: data.serviceName,
    inconsistencies,
    recoveryRequired: inconsistencies.some(i => i.severity === 'error'),
    recoveryPriority: inconsistencies.length > 5 ? 'high' : 'normal',
    estimatedRecoveryTime: inconsistencies.length * 30000 // 30s per inconsistency
  };
}

/**
 * Function Manager handles automated recovery
 */
class ProjectionHealthManager extends LucentServiceBase {
  
  async handleHealthCheckFailed(event: DomainEvent<HealthCheckData>): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      // 1. Execute pure function to detect issues
      const functionContext = this.createEnhancedContext(context);
      const inconsistencyReport = detectProjectionInconsistencies(functionContext, event.data);
      
      if (inconsistencyReport.recoveryRequired) {
        // 2. Trigger recovery for affected services
        await this.triggerProjectionRecovery(inconsistencyReport, context);
      }
    });
  }

  private async triggerProjectionRecovery(
    report: InconsistencyReport,
    context: RequestContext
  ): Promise<void> {
    
    this.logger.warn('Triggering projection recovery', {
      service: report.serviceName,
      inconsistencies: report.inconsistencies.length,
      priority: report.recoveryPriority
    });

    // Send recovery command to affected service
    await this.sendCommand(
      `${report.serviceName}-function-manager`,
      'RecoverProjections',
      {
        inconsistencies: report.inconsistencies,
        priority: report.recoveryPriority,
        estimatedDuration: report.estimatedRecoveryTime
      },
      context.businessContext
    );

    // Track recovery initiation
    await this.infrastructure.cacheStore.zadd(
      'active-recoveries',
      Date.now(),
      `${report.serviceName}:${context.correlationId}`
    );
  }
}
```

#### 2. Cold Start Recovery
```typescript
/**
 * Pure function to initialize empty service projections
 */
@EventHandler('ServiceStartup')
@ShardBy('service_name')
export function initializeServiceProjections(
  context: EnhancedFunctionContext,
  data: ServiceStartupData
): ServiceInitialization {
  
  context.logger.info('Initializing service projections on startup', {
    service: data.serviceName,
    expected_projections: data.expectedProjections.length
  });

  // Pure logic to determine what needs to be rebuilt
  const initializationPlan = {
    serviceName: data.serviceName,
    projectionsToRebuild: data.expectedProjections.filter(proj => 
      !data.existingProjections.includes(proj)
    ),
    usersToRecover: data.activeUsers,
    estimatedDuration: data.expectedProjections.length * 10000, // 10s per projection
    priority: data.serviceName.includes('trading') ? 'critical' : 'normal'
  };

  context.addMetadata('initialization.projections_needed', initializationPlan.projectionsToRebuild.length);
  context.addMetadata('initialization.users_to_recover', initializationPlan.usersToRecover.length);

  return initializationPlan;
}

/**
 * Function Manager handles cold start scenario
 */
class ColdStartRecoveryManager extends LucentServiceBase {
  
  async handleServiceStartup(event: DomainEvent<ServiceStartupData>): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      // 1. Determine what needs rebuilding (pure function)
      const functionContext = this.createEnhancedContext(context);
      const initPlan = initializeServiceProjections(functionContext, event.data);
      
      if (initPlan.projectionsToRebuild.length === 0) {
        this.logger.info('No projection recovery needed', {
          service: initPlan.serviceName
        });
        return;
      }

      // 2. Execute recovery for each missing projection
      await this.recoverMissingProjections(initPlan, context);
    });
  }

  private async recoverMissingProjections(
    plan: ServiceInitialization,
    context: RequestContext
  ): Promise<void> {
    
    this.logger.info('Starting cold start projection recovery', {
      service: plan.serviceName,
      projections_to_rebuild: plan.projectionsToRebuild.length,
      users_to_recover: plan.usersToRecover.length
    });

    for (const userId of plan.usersToRecover) {
      // Read user's complete event history from EventStore
      const userEvents = await this.getAllUserEvents(userId);
      
      // Rebuild each projection type for this user
      for (const projectionType of plan.projectionsToRebuild) {
        await this.rebuildUserProjection(userId, projectionType, userEvents);
      }
    }

    // Mark recovery complete
    await this.infrastructure.cacheStore.hset(
      `service:${plan.serviceName}:status`,
      'cold_start_recovery_completed',
      Date.now().toString()
    );
  }

  private async getAllUserEvents(userId: string): Promise<DomainEvent[]> {
    // Get events from all relevant streams
    const [userEvents, portfolioEvents, tradeEvents] = await Promise.all([
      this.infrastructure.eventStore.read(`user-${userId}`),
      this.infrastructure.eventStore.read(`portfolio-${userId}`),
      this.infrastructure.eventStore.read(`trades-${userId}`)
    ]);

    // Combine and sort chronologically
    const allEvents = [...userEvents, ...portfolioEvents, ...tradeEvents];
    return allEvents.sort((a, b) => 
      new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime()
    );
  }

  private async rebuildUserProjection(
    userId: string,
    projectionType: string,
    events: DomainEvent[]
  ): Promise<void> {
    
    switch (projectionType) {
      case 'portfolio':
        await this.rebuildPortfolioProjection(userId, events);
        break;
      case 'risk':
        await this.rebuildRiskProjection(userId, events);
        break;
      case 'analytics':
        await this.rebuildAnalyticsProjection(userId, events);
        break;
    }
  }
}
```

#### 3. Incremental Recovery
```typescript
/**
 * Pure function to compute incremental updates
 */
@EventHandler('ProjectionOutOfSync')
@ShardBy('user_id')
export function computeIncrementalUpdate(
  context: EnhancedFunctionContext,
  data: IncrementalRecoveryData
): IncrementalUpdate {
  
  context.logger.info('Computing incremental projection update', {
    user_id: data.userId,
    missing_events: data.missingEvents.length,
    last_known_version: data.lastKnownVersion
  });

  // Pure computation of what changed
  let incrementalState = { ...data.currentRedisState };

  // Apply only the missing events
  for (const event of data.missingEvents) {
    incrementalState = applyEventToState(incrementalState, event);
  }

  // Calculate what actually changed
  const changes = calculateStateDifferences(data.currentRedisState, incrementalState);

  return {
    userId: data.userId,
    changes,
    newState: incrementalState,
    eventsApplied: data.missingEvents.length,
    consistencyRestored: true
  };
}

/**
 * Function Manager handles incremental recovery
 */
class IncrementalRecoveryManager extends LucentServiceBase {
  
  async handleProjectionSync(userId: string, lastKnownVersion: number): Promise<void> {
    const businessContext = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('ProjectionSync', `sync-${userId}`)
      .withUser(userId)
      .build();

    return this.executeBusinessOperation('incremental_recovery', businessContext, async (context) => {
      
      // 1. Get current Redis state
      const currentRedisState = await this.getCurrentRedisState(userId);
      
      // 2. Get missing events from EventStore
      const missingEvents = await this.infrastructure.eventStore.read(
        `user-${userId}`,
        lastKnownVersion + 1 // Start from next version
      );

      if (missingEvents.length === 0) {
        this.logger.debug('No incremental recovery needed', { user_id: userId });
        return;
      }

      // 3. Execute pure function to compute updates
      const recoveryData: IncrementalRecoveryData = {
        userId,
        missingEvents,
        currentRedisState,
        lastKnownVersion
      };

      const functionContext = this.createEnhancedContext(context);
      const incrementalUpdate = computeIncrementalUpdate(functionContext, recoveryData);

      // 4. Apply incremental changes to Redis
      await this.applyIncrementalChanges(incrementalUpdate);

      this.logger.info('Incremental recovery completed', {
        user_id: userId,
        events_applied: incrementalUpdate.eventsApplied,
        changes_applied: Object.keys(incrementalUpdate.changes).length
      });
    });
  }

  private async applyIncrementalChanges(update: IncrementalUpdate): Promise<void> {
    const pipeline = this.infrastructure.cacheStore.pipeline();

    // Apply only the changed fields
    for (const [key, change] of Object.entries(update.changes)) {
      switch (change.type) {
        case 'field_update':
          pipeline.hset(`portfolio:${update.userId}:${change.section}`, change.field, change.newValue);
          break;
        case 'position_update':
          pipeline.hset(`portfolio:${update.userId}:positions`, change.pair, change.amount.toString());
          break;
        case 'ranking_update':
          pipeline.zadd(change.rankingKey, change.score, update.userId);
          break;
      }
    }

    await pipeline.exec();
  }
}
```

### Recovery Triggers and Strategies

#### Automatic Recovery Triggers
```typescript
/**
 * Monitor projection health and trigger recovery
 */
class ProjectionHealthMonitor extends LucentServiceBase {
  
  async monitorProjectionHealth(): Promise<void> {
    const businessContext = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('ProjectionHealth', 'global-monitor')
      .build();

    return this.executeBusinessOperation('monitor_projections', businessContext, async (context) => {
      
      // Check all services for projection health
      const services = ['portfolio', 'risk', 'analytics', 'trading'];
      
      for (const serviceName of services) {
        await this.checkServiceProjectionHealth(serviceName, context);
      }
    });
  }

  private async checkServiceProjectionHealth(
    serviceName: string, 
    context: RequestContext
  ): Promise<void> {
    
    // Get last EventStore event timestamp
    const lastEventTime = await this.getLastEventTimestamp(serviceName);
    
    // Get last Redis update timestamp  
    const lastRedisUpdate = await this.getLastRedisUpdate(serviceName);

    const lagMs = lastEventTime - lastRedisUpdate;

    if (lagMs > 300000) { // 5 minutes behind
      this.logger.warn('Projection lag detected', {
        service: serviceName,
        lag_ms: lagMs,
        last_event_time: new Date(lastEventTime),
        last_redis_update: new Date(lastRedisUpdate)
      });

      // Trigger incremental recovery
      await this.sendCommand(
        `${serviceName}-function-manager`,
        'TriggerIncrementalRecovery',
        { lagMs, reason: 'health_check_lag' },
        context.businessContext
      );
    }
  }
}
```

#### Manual Recovery Commands
```typescript
/**
 * Administrative recovery functions
 */
@EventHandler('AdminRecoveryCommand')
@ShardBy('admin_command')
export function processRecoveryCommand(
  context: EnhancedFunctionContext,
  command: AdminRecoveryCommand
): RecoveryPlan {
  
  context.logger.info('Processing admin recovery command', {
    command_type: command.type,
    target_service: command.targetService,
    scope: command.scope
  });

  let recoveryPlan: RecoveryPlan;

  switch (command.type) {
    case 'rebuild_all':
      recoveryPlan = {
        strategy: 'full_rebuild',
        affectedServices: [command.targetService],
        estimatedDuration: 3600000, // 1 hour
        dataLoss: false,
        requiresDowntime: false
      };
      break;

    case 'rebuild_user':
      recoveryPlan = {
        strategy: 'user_rebuild',
        affectedUsers: [command.userId!],
        estimatedDuration: 30000, // 30 seconds
        dataLoss: false,
        requiresDowntime: false
      };
      break;

    case 'resync_incremental':
      recoveryPlan = {
        strategy: 'incremental_sync',
        affectedServices: [command.targetService],
        estimatedDuration: 60000, // 1 minute
        dataLoss: false,
        requiresDowntime: false
      };
      break;

    default:
      throw new Error(`Unknown recovery command: ${command.type}`);
  }

  context.addMetadata('recovery.strategy', recoveryPlan.strategy);
  context.addMetadata('recovery.estimated_duration', recoveryPlan.estimatedDuration);

  return recoveryPlan;
}
```

### Recovery Best Practices

#### 1. Zero-Downtime Recovery
- **Read models remain available** during recovery (serve potentially stale data)
- **New events continue** to be processed normally
- **Recovery happens in background** without blocking operations
- **Atomic updates** when recovery completes

#### 2. Consistency Verification
```typescript
/**
 * Pure function to verify projection consistency
 */
@EventHandler('ConsistencyCheckRequested')
export function verifyProjectionConsistency(
  context: EnhancedFunctionContext,
  data: ConsistencyCheckData
): ConsistencyReport {
  
  // Compare EventStore-derived state with Redis state
  const eventStoreState = computeStateFromEvents(data.events);
  const redisState = parseRedisState(data.redisData);
  
  const differences = findStateDifferences(eventStoreState, redisState);
  
  return {
    isConsistent: differences.length === 0,
    differences,
    lastEventProcessed: data.events[data.events.length - 1]?.eventNumber || 0,
    redisLastUpdate: redisState.lastUpdated,
    consistencyScore: 1.0 - (differences.length / Object.keys(eventStoreState).length)
  };
}
```

#### 3. Performance-Optimized Recovery
- **Batch event processing** for large histories
- **Parallel recovery** across multiple users
- **Progressive loading** for massive datasets
- **Resource-aware** recovery scheduling

This recovery pattern ensures **zero data loss** and **automatic healing** when Redis projections become inconsistent with the EventStore source of truth.