# Cache Repository Patterns

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Standardized cache abstraction patterns for service read models

## Overview

Cache Repository Patterns provide **standardized abstractions** for service-specific Redis operations, eliminating raw Redis commands from business logic and ensuring **consistent caching patterns** across all services.

## Standardized Cache Repository Architecture

### Base Cache Repository
```typescript
/**
 * Base cache repository with common patterns
 */
export abstract class CacheRepository<T> {
  protected keyPrefix: string;
  
  constructor(
    protected cacheStore: ICacheStore,
    keyPrefix: string,
    protected logger: InfrastructureLogger
  ) {
    this.keyPrefix = keyPrefix;
  }

  /**
   * Generic entity caching with TTL
   */
  protected async cacheEntity(id: string, entity: T, ttlSeconds?: number): Promise<void> {
    const key = this.buildEntityKey(id);
    const serialized = this.serializeEntity(entity);
    
    await this.cacheStore.set(key, serialized, ttlSeconds);
    
    this.logger.debug('Entity cached', {
      key_prefix: this.keyPrefix,
      entity_id: id,
      ttl_seconds: ttlSeconds
    });
  }

  /**
   * Generic entity retrieval with deserialization
   */
  protected async getEntity(id: string): Promise<T | null> {
    const key = this.buildEntityKey(id);
    const cached = await this.cacheStore.get(key);
    
    if (!cached) return null;
    
    return this.deserializeEntity(cached);
  }

  /**
   * Generic hash-based entity storage
   */
  protected async updateEntityFields(id: string, fields: Record<string, any>): Promise<void> {
    const key = this.buildEntityKey(id);
    const serializedFields: Record<string, string> = {};
    
    for (const [field, value] of Object.entries(fields)) {
      serializedFields[field] = this.serializeField(value);
    }
    
    await this.cacheStore.hmset(key, serializedFields);
  }

  /**
   * Generic entity field retrieval
   */
  protected async getEntityFields(id: string): Promise<Record<string, any>> {
    const key = this.buildEntityKey(id);
    const fields = await this.cacheStore.hgetall(key);
    
    const deserialized: Record<string, any> = {};
    for (const [field, value] of Object.entries(fields)) {
      deserialized[field] = this.deserializeField(field, value);
    }
    
    return deserialized;
  }

  /**
   * Generic ranking operations
   */
  protected async addToRanking(rankingKey: string, score: number, member: string): Promise<void> {
    await this.cacheStore.zadd(this.buildRankingKey(rankingKey), score, member);
  }

  protected async getTopRanked(rankingKey: string, limit: number): Promise<RankingEntry[]> {
    const members = await this.cacheStore.zrevrange(this.buildRankingKey(rankingKey), 0, limit - 1);
    
    const rankings: RankingEntry[] = [];
    for (const member of members) {
      const score = await this.cacheStore.zscore(this.buildRankingKey(rankingKey), member);
      rankings.push({ member, score: score || 0, rank: rankings.length + 1 });
    }
    
    return rankings;
  }

  // Key generation methods
  protected buildEntityKey(id: string): string {
    return `${this.keyPrefix}:${id}`;
  }

  protected buildRankingKey(rankingType: string): string {
    return `${this.keyPrefix}-rankings:${rankingType}`;
  }

  // Serialization methods (override in subclasses)
  protected abstract serializeEntity(entity: T): string;
  protected abstract deserializeEntity(data: string): T;
  
  protected serializeField(value: any): string {
    if (typeof value === 'object') {
      return JSON.stringify(value);
    }
    return String(value);
  }

  protected deserializeField(field: string, value: string): any {
    // Try to parse as JSON first
    try {
      return JSON.parse(value);
    } catch {
      // Return as string if not JSON
      return value;
    }
  }
}

interface RankingEntry {
  member: string;
  score: number;
  rank: number;
}
```

### Portfolio Cache Repository
```typescript
/**
 * Portfolio-specific cache repository
 */
export class PortfolioCacheRepository extends CacheRepository<Portfolio> {
  
  constructor(cacheStore: ICacheStore, logger: InfrastructureLogger) {
    super(cacheStore, 'portfolio', logger);
  }

  /**
   * Update portfolio with clean domain abstraction
   */
  async updatePortfolio(portfolio: Portfolio): Promise<void> {
    // Use base class hash operations
    await this.updateEntityFields(portfolio.userId, {
      'totalValue': portfolio.totalValue,
      'positionCount': Object.keys(portfolio.positions).length,
      'riskLevel': portfolio.riskLevel,
      'lastUpdated': Date.now(),
      'version': portfolio.version
    });

    // Store positions separately
    await this.updatePositions(portfolio.userId, portfolio.positions);
    
    // Update rankings
    await this.addToRanking('total-value', portfolio.totalValue, portfolio.userId);
    
    this.logger.info('Portfolio cache updated', {
      user_id: portfolio.userId,
      total_value: portfolio.totalValue,
      position_count: Object.keys(portfolio.positions).length
    });
  }

  async getPortfolio(userId: string): Promise<Portfolio | null> {
    const [metadata, positions] = await Promise.all([
      this.getEntityFields(userId),
      this.getPositions(userId)
    ]);

    if (!metadata.totalValue) return null;

    return {
      userId,
      positions,
      totalValue: parseFloat(metadata.totalValue),
      riskLevel: metadata.riskLevel,
      lastUpdated: new Date(parseInt(metadata.lastUpdated)),
      version: parseInt(metadata.version)
    };
  }

  async getTopPortfolios(limit: number = 10): Promise<PortfolioRanking[]> {
    const rankings = await this.getTopRanked('total-value', limit);
    
    return rankings.map(ranking => ({
      userId: ranking.member,
      totalValue: ranking.score,
      rank: ranking.rank
    }));
  }

  private async updatePositions(userId: string, positions: Record<string, number>): Promise<void> {
    const positionsKey = `${this.keyPrefix}:${userId}:positions`;
    const serializedPositions: Record<string, string> = {};
    
    for (const [pair, amount] of Object.entries(positions)) {
      if (Math.abs(amount) > 0.000001) {
        serializedPositions[pair] = amount.toString();
      }
    }
    
    // Clear existing positions and set new ones
    await this.cacheStore.del(positionsKey);
    if (Object.keys(serializedPositions).length > 0) {
      await this.cacheStore.hmset(positionsKey, serializedPositions);
    }
  }

  private async getPositions(userId: string): Promise<Record<string, number>> {
    const positionsKey = `${this.keyPrefix}:${userId}:positions`;
    const positions = await this.cacheStore.hgetall(positionsKey);
    
    const result: Record<string, number> = {};
    for (const [pair, amount] of Object.entries(positions)) {
      result[pair] = parseFloat(amount);
    }
    
    return result;
  }

  protected serializeEntity(portfolio: Portfolio): string {
    return JSON.stringify(portfolio);
  }

  protected deserializeEntity(data: string): Portfolio {
    return JSON.parse(data);
  }
}
```

### Risk Cache Repository
```typescript
/**
 * Risk-specific cache repository
 */
export class RiskCacheRepository extends CacheRepository<RiskMetrics> {
  
  constructor(cacheStore: ICacheStore, logger: InfrastructureLogger) {
    super(cacheStore, 'risk', logger);
  }

  async updateUserRisk(userId: string, risk: RiskMetrics): Promise<void> {
    // Update user risk data
    await this.updateEntityFields(userId, {
      'riskScore': risk.riskScore,
      'riskLevel': risk.riskLevel,
      'dailyVolume': risk.dailyVolume,
      'largestTrade': risk.largestTrade,
      'lastUpdated': Date.now()
    });

    // Update risk rankings
    await this.addToRanking('risk-score', risk.riskScore, userId);
    await this.addToRanking('daily-volume', risk.dailyVolume, userId);
    
    // Update risk category sets
    await this.updateRiskCategory(userId, risk.riskLevel);
  }

  async getUserRisk(userId: string): Promise<RiskMetrics | null> {
    const fields = await this.getEntityFields(userId);
    
    if (!fields.riskScore) return null;

    return {
      userId,
      riskScore: parseFloat(fields.riskScore),
      riskLevel: fields.riskLevel,
      dailyVolume: parseFloat(fields.dailyVolume || '0'),
      largestTrade: parseFloat(fields.largestTrade || '0'),
      lastUpdated: new Date(parseInt(fields.lastUpdated))
    };
  }

  async getHighRiskUsers(limit: number = 20): Promise<RiskRanking[]> {
    const rankings = await this.getTopRanked('risk-score', limit);
    
    return rankings.map(ranking => ({
      userId: ranking.member,
      riskScore: ranking.score,
      rank: ranking.rank
    }));
  }

  private async updateRiskCategory(userId: string, riskLevel: string): Promise<void> {
    // Remove from other categories
    const categories = ['low', 'medium', 'high'];
    for (const category of categories) {
      if (category !== riskLevel) {
        await this.cacheStore.srem(`risk-category:${category}`, userId);
      }
    }
    
    // Add to correct category
    await this.cacheStore.sadd(`risk-category:${riskLevel}`, userId);
  }

  protected serializeEntity(risk: RiskMetrics): string {
    return JSON.stringify(risk);
  }

  protected deserializeEntity(data: string): RiskMetrics {
    return JSON.parse(data);
  }
}
```

## Service Integration with Cache Repositories

### Clean Service Implementation
```typescript
/**
 * Service uses cache repository for clean abstractions
 */
export class PortfolioService extends CryptoTradingServiceBase {
  private portfolioCache: PortfolioCacheRepository;
  
  constructor() {
    super(infrastructure, 'portfolio-service');
    this.portfolioCache = new PortfolioCacheRepository(
      this.infrastructure.cacheStore, 
      this.logger
    );
  }

  async handleTradeExecuted(event: TradeExecutedEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      // Get current portfolio state
      const currentPortfolio = await this.portfolioCache.getPortfolio(event.data.userId);
      
      // Send calculation to Function Manager
      const portfolioUpdate = await this.sendCommand(
        'function-manager',
        'ExecuteFunction',
        {
          functionType: 'updatePortfolioProjection',
          eventType: 'PortfolioUpdateRequested',
          data: {
            currentPortfolio,
            trade: event.data,
            timestamp: Date.now()
          }
        },
        context.businessContext
      );

      if (portfolioUpdate.success) {
        // Update cache with clean abstraction
        await this.portfolioCache.updatePortfolio(portfolioUpdate.result);
        
        // Publish domain event
        await this.publishDomainEvent(
          'PortfolioProjectionUpdated',
          `portfolio-${event.data.userId}`,
          portfolioUpdate.result,
          context.businessContext,
          context
        );
      }
    });
  }

  /**
   * Fast portfolio queries using cache repository
   */
  async getPortfolioSummary(userId: string): Promise<PortfolioSummary | null> {
    const portfolio = await this.portfolioCache.getPortfolio(userId);
    if (!portfolio) return null;

    const ranking = await this.portfolioCache.getTopPortfolios(100);
    const userRank = ranking.find(r => r.userId === userId)?.rank || null;

    return {
      ...portfolio,
      ranking: userRank,
      percentile: userRank ? (1 - (userRank / ranking.length)) * 100 : null
    };
  }
}
```

## Benefits of Cache Repository Pattern

### Clean Abstraction
- ✅ **Domain-specific methods**: `updatePortfolio()`, `getUserRisk()`, `addToLeaderboard()`
- ✅ **Hidden implementation**: Redis details abstracted away
- ✅ **Type safety**: Full TypeScript validation
- ✅ **Testability**: Easy to mock repository methods

### Consistency
- ✅ **Standardized patterns**: All services use same caching approach
- ✅ **Key management**: Consistent key naming across services  
- ✅ **Serialization**: Unified approach to data storage
- ✅ **Error handling**: Consistent cache operation error handling

### Maintainability  
- ✅ **Single responsibility**: Each repository handles one domain
- ✅ **Easy to change**: Update caching strategy in one place
- ✅ **Professional naming**: Clear, business-focused method names
- ✅ **No leaky abstractions**: Business logic doesn't know about Redis

This cache repository pattern ensures **clean, maintainable, and testable** caching operations across all crypto trading services.