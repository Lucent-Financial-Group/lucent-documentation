# Debugging Event-Driven Function Architecture

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Professional debugging methodology for distributed pure functions

## Overview

Debugging distributed event-driven systems requires leveraging our enhanced tracing, business context, and function registry. All debugging capabilities are **built into existing classes** (Function Manager, Service Classes) rather than separate specialized debuggers.

## Distributed Tracing Debugging

### 1. Business Process Tracing via Function Manager
```typescript
/**
 * Function Manager includes built-in debugging capabilities
 */
export class FunctionManager extends LucentServiceBase {
  
  /**
   * Debug function performance (built into Function Manager)
   */
  async debugFunctionPerformance(functionName: string): Promise<PerformanceDebugReport> {
    
    return this.executeBusinessOperation('debug_function_performance', this.createSystemContext(), async (context) => {
      
      this.logger.info('Analyzing function performance across shards', {
        function_name: functionName,
        analysis_started: Date.now()
      });

      // Collect performance data from current shard
      const localMetrics = await this.getFunctionMetrics(functionName);
      
      // Query other shards for their metrics
      const otherShardMetrics = await Promise.all(
        this.availableShards
          .filter(shard => shard !== this.shardId)
          .map(async shard => {
            try {
              const metrics = await this.sendCommand(
                `function-manager-${shard}`,
                'GetFunctionMetrics',
                { functionName },
                context.businessContext
              );
              return { shard, metrics: metrics.result };
            } catch (error: any) {
              return { shard, metrics: null, error: error.message };
            }
          })
      );

      // Analyze performance patterns
      const allMetrics = [
        { shard: this.shardId, metrics: localMetrics },
        ...otherShardMetrics
      ];

      const performanceAnalysis = this.analyzePerformanceMetrics(allMetrics);
      
      const report: PerformanceDebugReport = {
        functionName,
        analysisTimestamp: Date.now(),
        shardMetrics: allMetrics,
        analysis: performanceAnalysis,
        recommendations: this.generatePerformanceRecommendations(performanceAnalysis)
      };
      
      // Store debug report for monitoring
      await this.infrastructure.cacheStore.set(
        `debug-report:${functionName}`,
        JSON.stringify(report),
        3600 // 1 hour TTL
      );

      return report;
    });
  }

  /**
   * Debug shard load distribution (built into Function Manager)
   */
  async debugShardDistribution(): Promise<ShardDebugReport> {
    
    return this.executeBusinessOperation('debug_shard_distribution', this.createSystemContext(), async (context) => {
      
      // Analyze function distribution across shards
      const functionDistribution = new Map<string, number>();
      
      for (const functionName of Object.keys(TYPED_FUNCTION_REGISTRY)) {
        const metrics = await this.getFunctionMetrics(functionName);
        functionDistribution.set(functionName, metrics.executionCount);
      }

      // Check for hot spots and cold spots
      const totalExecutions = Array.from(functionDistribution.values()).reduce((sum, count) => sum + count, 0);
      const avgExecutions = totalExecutions / functionDistribution.size;
      
      const hotFunctions = Array.from(functionDistribution.entries())
        .filter(([_, count]) => count > avgExecutions * 2)
        .map(([name, count]) => ({ name, count, factor: count / avgExecutions }));

      const coldFunctions = Array.from(functionDistribution.entries())
        .filter(([_, count]) => count < avgExecutions * 0.5)
        .map(([name, count]) => ({ name, count, factor: count / avgExecutions }));

      return {
        shardId: this.shardId,
        totalFunctions: functionDistribution.size,
        totalExecutions,
        averageExecutions: avgExecutions,
        hotFunctions,
        coldFunctions,
        distributionScore: this.calculateDistributionScore(hotFunctions, coldFunctions),
        recommendations: this.generateDistributionRecommendations(hotFunctions, coldFunctions)
      };
    });
  }
}
```

### 2. Service-Level Debugging

```typescript
/**
 * Service classes include debugging methods (not separate debugger classes)
 */
export class PortfolioService extends CryptoTradingServiceBase {
  
  /**
   * Debug portfolio projection consistency
   */
  async debugPortfolioConsistency(userId: string): Promise<ConsistencyDebugReport> {
    
    return this.executeUserOperation('debug_portfolio_consistency', userId, async (context) => {
      
      // Get current Redis projection
      const redisPortfolio = await this.infrastructure.cacheStore.hgetall(`portfolio:${userId}`);
      
      // Rebuild from EventStore
      const portfolioEvents = await this.infrastructure.eventStore.read(`portfolio-${userId}`);
      
      // Ask Function Manager to rebuild portfolio
      const rebuiltPortfolio = await this.sendCommand(
        'function-manager',
        'ExecuteFunction',
        {
          functionType: 'rebuildPortfolioFromEvents',
          eventType: 'PortfolioRebuildRequested',
          data: { userId, events: portfolioEvents }
        },
        context.businessContext
      );

      // Compare for consistency
      const inconsistencies = this.comparePortfolios(redisPortfolio, rebuiltPortfolio.result);
      
      const report = {
        userId,
        redisPortfolio,
        rebuiltPortfolio: rebuiltPortfolio.result,
        isConsistent: inconsistencies.length === 0,
        inconsistencies,
        checkedAt: Date.now()
      };

      if (inconsistencies.length > 0) {
        this.logger.warn('Portfolio inconsistency detected', {
          user_id: userId,
          inconsistency_count: inconsistencies.length,
          auto_repair_recommended: true
        });

        // Auto-repair if configured
        if (this.config.autoRepairProjections) {
          await this.updatePortfolioProjection(userId, rebuiltPortfolio.result);
          report.autoRepaired = true;
        }
      }

      return report;
    });
  }

  private comparePortfolios(redis: any, rebuilt: any): string[] {
    const inconsistencies: string[] = [];
    
    if (redis.totalValue !== rebuilt.totalValue?.toString()) {
      inconsistencies.push(`Total value mismatch: Redis=${redis.totalValue}, Rebuilt=${rebuilt.totalValue}`);
    }

    const redisPositions = JSON.parse(redis.positions || '{}');
    const rebuiltPositions = rebuilt.positions || {};
    
    for (const [pair, amount] of Object.entries(rebuiltPositions)) {
      if (redisPositions[pair]?.toString() !== amount?.toString()) {
        inconsistencies.push(`Position mismatch for ${pair}: Redis=${redisPositions[pair]}, Rebuilt=${amount}`);
      }
    }

    return inconsistencies;
  }
}
```

## Debug Query Patterns

### Jaeger Query Examples for Business Debugging

```bash
# Find all events for a specific user across all services
business.user_id="user-123"

# Trace complete yield farming decision process  
business.workflow_id="yield-workflow-789" 

# Find all high-risk trading operations
crypto.risk_level="high" AND business.domain="crypto-trading"

# Debug function load balancing decisions
function.shard_id="shard-aave" AND event.type="YieldOpportunityDetected"

# Find performance bottlenecks in specific functions
function.name="calculateYieldStrategy" AND function.execution_time_ms>5000

# Trace multi-service workflows
business.process_name="arbitrage_execution" AND crypto.trading_pair="ETH-USDC"
```

### Built-in Debugging Commands

Services and Function Manager include debugging methods accessible via:

```typescript
// Portfolio Service debugging
await portfolioService.debugPortfolioConsistency('user-123');
await portfolioService.debugProjectionHealth();

// Function Manager debugging  
await functionManager.debugFunctionPerformance('calculateYieldStrategy');
await functionManager.debugShardDistribution();
await functionManager.debugLoadBalancing();

// All debugging methods return structured reports with:
// - Performance metrics
// - Inconsistency detection
// - Auto-repair capabilities
// - Actionable recommendations
```

## Error Correlation and Analysis

### Built-in Error Analysis

```typescript
/**
 * Function Manager includes error correlation (not separate analyzer)
 */
export class FunctionManager extends LucentServiceBase {
  
  /**
   * Analyze function execution errors
   */
  async analyzeExecutionErrors(timeWindow: number = 3600000): Promise<ErrorAnalysisReport> {
    
    return this.executeBusinessOperation('analyze_execution_errors', this.createSystemContext(), async (context) => {
      
      // Get recent error events from EventStore
      const errorEvents = await this.infrastructure.eventStore.read('function-errors', Date.now() - timeWindow);
      
      // Group errors by pattern
      const errorPatterns = new Map<string, ErrorPattern>();
      
      for (const errorEvent of errorEvents) {
        const pattern = this.classifyErrorPattern(errorEvent);
        
        if (!errorPatterns.has(pattern.type)) {
          errorPatterns.set(pattern.type, {
            type: pattern.type,
            count: 0,
            functions: new Set(),
            shards: new Set(),
            firstSeen: Date.now(),
            lastSeen: 0
          });
        }

        const existing = errorPatterns.get(pattern.type)!;
        existing.count++;
        existing.functions.add(errorEvent.data.functionName);
        existing.shards.add(errorEvent.data.shardId);
        existing.lastSeen = Math.max(existing.lastSeen, errorEvent.data.timestamp);
      }

      // Generate recommendations
      const recommendations = Array.from(errorPatterns.values())
        .filter(pattern => pattern.count > 5) // Significant patterns
        .map(pattern => this.generateErrorRecommendation(pattern));

      return {
        timeWindow,
        totalErrors: errorEvents.length,
        errorPatterns: Array.from(errorPatterns.values()),
        recommendations,
        analysisTimestamp: Date.now()
      };
    });
  }
}
```

## Debugging Best Practices

### Professional Debugging Approach
- ✅ **Built-in debugging**: Methods integrated into existing classes
- ✅ **Structured reports**: Consistent debugging output format  
- ✅ **Auto-repair capabilities**: Fix issues automatically when possible
- ✅ **Actionable recommendations**: Clear next steps for issue resolution
- ✅ **Professional naming**: `debugPerformance()`, not `FunctionPerformanceDebugger`

### No Separate Debugger Classes
- ❌ **Removed**: `FunctionPerformanceDebugger`, `CrossShardErrorAnalyzer`, `LoadBalancingDebugger`
- ✅ **Consolidated**: All debugging built into `FunctionManager` and service classes
- ✅ **Clean interfaces**: Simple method calls, not complex class hierarchies

### Debugging Integration
- ✅ **OpenTelemetry**: Every operation automatically traced
- ✅ **Business context**: Every debug operation includes business metadata
- ✅ **Correlation tracking**: Debug operations linked to business processes
- ✅ **Performance monitoring**: Built-in metrics for all functions

This professional debugging approach provides **comprehensive debugging capabilities** without the complexity of specialized debugger classes.