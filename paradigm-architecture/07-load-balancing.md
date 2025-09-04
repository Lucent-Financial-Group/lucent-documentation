# Load Balancing and Dynamic Sharding

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Generic function distribution through decorator-driven routing

## Overview

The Load Balancing system uses **decorator metadata** and **generic algorithms** built into the Function Manager to distribute pure functions across shards optimally. All routing decisions are made based on `@ShardBy`, `@ResourceRequirements`, and runtime load metrics - **no domain-specific logic** in the infrastructure layer.

## Decorator-Driven Load Balancing

### Function Routing via Decorators

Functions declare their own routing preferences using decorators:

```typescript
// Protocol-specific routing
@EventHandler('YieldOpportunityDetected')
@ShardBy('protocol', 'aave')
@ResourceRequirements({ cpu: 'medium', memory: '256MB', priority: 'high', timeout: 15000 })
export function calculateAaveYield(context, data): AaveYieldStrategy {
  // Function automatically routes to Aave-specialized shards
}

// Trading pair distribution
@EventHandler('ArbitrageOpportunityDetected')
@ShardBy('trading_pair')
@ResourceRequirements({ cpu: 'high', memory: '1GB', priority: 'critical', timeout: 30000 })
export function detectCrossExchangeArbitrage(context, data): ArbitrageStrategy {
  // Function automatically distributes by hash(tradingPair) % shardCount
}

// Risk-based routing
@EventHandler('HighRiskTradeRequested')
@ShardBy('risk_level', 'high')
@ResourceRequirements({ cpu: 'critical', memory: '2GB', priority: 'critical', timeout: 45000 })
export function processHighRiskTrade(context, data): RiskTradeResult {
  // Function automatically routes to high-risk specialized shards
}
```

### Generic Load Balancing in Function Manager

The **Function Manager uses generic algorithms** based on decorator metadata:

```typescript
/**
 * Function Manager handles ALL load balancing generically
 */
export class FunctionManager extends LucentServiceBase {
  
  /**
   * Generic function selection algorithm
   */
  private async selectOptimalFunction(functions: FunctionRegistryEntry[]): Promise<FunctionRegistryEntry> {
    
    // Generic scoring based on decorator metadata only
    const functionScores = await Promise.all(
      functions.map(async func => ({
        function: func,
        loadScore: await this.getCurrentFunctionLoad(func.name),
        resourceScore: await this.checkResourceAvailability(func.resourceRequirements),
        priorityScore: this.calculatePriorityScore(func.resourceRequirements.priority),
        shardAffinityScore: await this.calculateShardAffinity(func.shardingStrategy)
      }))
    );

    // Generic composite scoring
    const scoredFunctions = functionScores.map(fs => ({
      ...fs,
      compositeScore: 
        (fs.loadScore * 0.3) +         // Current function load
        (fs.resourceScore * 0.25) +    // Resource availability  
        (fs.priorityScore * 0.25) +    // Priority from @ResourceRequirements
        (fs.shardAffinityScore * 0.2)  // Affinity from @ShardBy
    }));

    return scoredFunctions.sort((a, b) => b.compositeScore - a.compositeScore)[0].function;
  }

  /**
   * Generic shard routing based on decorator metadata
   */
  private async determineTargetShard(event: DomainEvent): Promise<string> {
    const functions = this.getFunctionsForEvent(event.eventType);
    
    for (const func of functions) {
      const strategy = func.shardingStrategy;
      
      // Generic routing algorithm
      switch (strategy.type) {
        case 'protocol':
          return this.getShardForProtocol(event.data.protocol, strategy.value);
          
        case 'trading_pair':
          const pair = event.data.tradingPair || event.data.pair;
          return this.getShardByHash(pair);
          
        case 'user_id':
          return this.getShardByHash(event.data.userId);
          
        case 'risk_level':
          return this.getShardForRiskLevel(event.data.riskLevel || 'medium');
          
        case 'amount_range':
          const amount = event.data.amountUsd || 0;
          return this.getShardForAmountRange(amount);
          
        case 'round_robin':
          return this.getNextShardRoundRobin();
          
        default:
          return this.getLeastLoadedShard();
      }
    }
    
    return this.shardId; // Default to current shard
  }

  /**
   * Generic shard mapping helpers
   */
  private getShardForProtocol(protocol: string, specificValue?: string): string {
    if (specificValue && protocol === specificValue) {
      return `shard-${protocol}`;
    }
    return `shard-${protocol}` || this.getLeastLoadedShard();
  }

  private getShardByHash(key: string): string {
    const hash = this.calculateHash(key);
    const shardIndex = hash % this.availableShards.length;
    return this.availableShards[shardIndex];
  }

  private getShardForRiskLevel(riskLevel: string): string {
    const riskShards = {
      'low': 'shard-retail',
      'medium': 'shard-institutional', 
      'high': 'shard-whale'
    };
    return riskShards[riskLevel as keyof typeof riskShards] || this.getLeastLoadedShard();
  }

  private getShardForAmountRange(amountUsd: number): string {
    if (amountUsd > 1000000) return 'shard-whale';
    if (amountUsd > 100000) return 'shard-institutional';
    return 'shard-retail';
  }
}
```

## Benefits of Decorator-Driven Load Balancing

### Simplicity
- ✅ **Functions declare routing**: `@ShardBy('protocol')` handles all protocol routing
- ✅ **No specialized classes**: All logic in generic Function Manager
- ✅ **Consistent patterns**: Same load balancing for all domains

### Flexibility
- ✅ **Easy to modify**: Change function routing by changing decorator
- ✅ **A/B testing**: Route different versions to different shards
- ✅ **Hot deployment**: Update routing without infrastructure changes

### Maintainability
- ✅ **Single codebase**: All load balancing in Function Manager
- ✅ **No crypto-specific logic**: Infrastructure works for any domain
- ✅ **Generic algorithms**: Reusable across all function types

## Performance Optimization

### Generic Performance Monitoring
```typescript
/**
 * Performance monitoring built into Function Manager (not separate class)
 */
export class FunctionManager extends LucentServiceBase {
  
  /**
   * Monitor function performance generically
   */
  async monitorFunctionPerformance(): Promise<void> {
    return this.executeBusinessOperation('monitor_performance', this.createSystemContext(), async (context) => {
      
      // Get all active functions from registry
      const allFunctions = Object.keys(TYPED_FUNCTION_REGISTRY);
      
      for (const functionName of allFunctions) {
        const metrics = await this.getFunctionMetrics(functionName);
        
        // Generic performance analysis
        if (metrics.avgExecutionTime > metrics.expectedTime * 1.5) {
          this.logger.warn('Function performance degraded', {
            function_name: functionName,
            avg_time: metrics.avgExecutionTime,
            expected_time: metrics.expectedTime,
            degradation_factor: metrics.avgExecutionTime / metrics.expectedTime
          });
          
          // Generic remediation
          await this.redistributeFunctionLoad(functionName);
        }
      }
    });
  }

  /**
   * Generic function load redistribution
   */
  private async redistributeFunctionLoad(functionName: string): Promise<void> {
    // Move function to less loaded shards generically
    const currentShardLoad = await this.getCurrentShardLoad();
    const alternativeShards = await this.getLessLoadedShards(currentShardLoad);
    
    if (alternativeShards.length > 0) {
      const targetShard = alternativeShards[0];
      
      this.logger.info('Redistributing function load', {
        function_name: functionName,
        from_shard: this.shardId,
        to_shard: targetShard,
        reason: 'performance_degradation'
      });
      
      // Update shard preferences for this function
      await this.updateShardPreferences(functionName, targetShard);
    }
  }
}
```

This approach eliminates **all specialized load balancing classes** and consolidates everything into the **generic Function Manager** using **decorator metadata for routing decisions**.