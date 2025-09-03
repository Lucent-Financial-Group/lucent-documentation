# Decorator Pattern for Function Registration

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Type-safe function discovery and sharding configuration

## Overview

Decorators in Lucent Services provide **declarative configuration** for pure functions, enabling automatic discovery, intelligent sharding, and resource optimization while preserving full TypeScript type safety.

## Decorator Types

### @EventHandler(eventType: string)

Declares which domain events a function can process.

```typescript
import { EventHandler, EnhancedFunctionContext } from '@lucent/infrastructure';
import { YieldOpportunityData, YieldStrategy } from './types';

@EventHandler('YieldOpportunityDetected')
export function calculateCompoundYield(
  context: EnhancedFunctionContext,
  data: YieldOpportunityData
): YieldStrategy {
  // Function automatically registered for 'YieldOpportunityDetected' events
  return {
    protocol: 'compound',
    allocation: calculateOptimalAllocation(data),
    confidence: assessConfidence(data)
  };
}

// Multiple event types for same function
@EventHandler('MarketDataUpdated')
@EventHandler('PriceAlertTriggered') 
export function analyzeMarketConditions(
  context: EnhancedFunctionContext,
  data: MarketData
): MarketAnalysis {
  // Handles both market data updates and price alerts
}
```

### @ShardBy(strategy: string, value?: string)

Configures intelligent routing strategy for load distribution.

```typescript
// Protocol-based sharding
@EventHandler('YieldOpportunityDetected')
@ShardBy('protocol', 'aave') // Only Aave events route to this function
export function calculateAaveYield(context, data): AaveYieldStrategy {
  // This function only runs on Aave-specialized shards
}

// Trading pair sharding
@EventHandler('ArbitrageOpportunityDetected')
@ShardBy('trading_pair') // Routes by hash(tradingPair) % shardCount  
export function detectCrossExchangeArbitrage(context, data): ArbitrageStrategy {
  // ETH pairs → Shard A, BTC pairs → Shard B, etc.
}

// Risk-based sharding
@EventHandler('TradeExecutionRequested')
@ShardBy('risk_level') // High-risk trades go to specialized shards
export function executeHighRiskTrade(context, data): TradeResult {
  // High-risk trades get dedicated high-performance nodes
}

// Amount-based sharding  
@EventHandler('PortfolioRebalanceRequested')
@ShardBy('amount_range') // Whale trades vs retail trades
export function rebalanceWhalePortfolio(context, data): RebalanceStrategy {
  // Large portfolios get dedicated resources
}

// Geographic sharding
@EventHandler('MarketDataReceived')
@ShardBy('geographic_region', 'us_east')
export function processUSMarketData(context, data): ProcessedData {
  // US market data processed on US East nodes for latency
}

// User-based sharding (sticky sessions)
@EventHandler('UserPreferenceUpdated') 
@ShardBy('user_id') // Same user always routes to same shard
export function updateUserStrategy(context, data): UserStrategy {
  // User data stays on same shard for consistency
}
```

### @ResourceRequirements(requirements)

Specifies computational requirements for load balancing.

```typescript
// Low resource function - can run on any shard
@EventHandler('UserNotificationRequested')
@ResourceRequirements({ 
  cpu: 'low',      // 10% CPU
  memory: '64MB',  // 64MB RAM
  priority: 'low', // Can be delayed
  timeout: 5000    // 5 second timeout
})
export function sendUserNotification(context, data): NotificationResult {
  // Lightweight function
}

// High resource function - needs powerful shards
@EventHandler('ComplexArbitrageAnalysis') 
@ResourceRequirements({
  cpu: 'critical',    // 90% CPU  
  memory: '2GB',      // 2GB RAM
  priority: 'critical', // Never delay
  timeout: 60000      // 1 minute timeout
})
export function analyzeComplexArbitrage(context, data): ComplexAnalysis {
  // CPU-intensive calculation requiring dedicated resources
}

// Medium resource function
@EventHandler('YieldOptimizationRequested')
@ResourceRequirements({
  cpu: 'medium',     // 30% CPU
  memory: '512MB',   // 512MB RAM  
  priority: 'high',  // Important but not critical
  timeout: 30000     // 30 second timeout
})
export function optimizeYieldStrategy(context, data): OptimizedStrategy {
  // Balanced resource usage
}
```

### @ShardFilter(predicate: string)

Advanced filtering for complex routing logic.

```typescript
// Custom routing logic using JavaScript expressions
@EventHandler('YieldOpportunityDetected')
@ShardFilter('data.apy > 15 && data.riskScore < 5') // Only high-yield, low-risk
export function calculatePremiumYield(context, data): PremiumStrategy {
  // Only processes exceptional yield opportunities
}

@EventHandler('TradeExecutionRequested')
@ShardFilter('data.amountUsd > 1000000 && context.businessContext.crypto.riskLevel === "high"')
export function executeInstitutionalTrade(context, data): InstitutionalTradeResult {
  // Only institutional-level high-value trades
}

@EventHandler('ArbitrageOpportunityDetected')
@ShardFilter('data.profitPotential > 0.05 && data.executionRisk < 0.2')
export function executeHighProfitArbitrage(context, data): ArbitrageExecution {
  // Only high-profit, low-risk arbitrage opportunities
}
```

### @Version(version: string)

Enables A/B testing and gradual rollouts.

```typescript
// Version A - conservative strategy
@EventHandler('YieldOpportunityDetected')
@Version('v1.0')
@ShardBy('protocol', 'aave')
export function calculateAaveYieldV1(context, data): YieldStrategy {
  // Conservative yield calculation
  return { allocation: 0.2, riskAdjustment: 0.8 };
}

// Version B - aggressive strategy  
@EventHandler('YieldOpportunityDetected')
@Version('v2.0')
@ShardBy('protocol', 'aave')
export function calculateAaveYieldV2(context, data): YieldStrategy {
  // More aggressive yield calculation
  return { allocation: 0.5, riskAdjustment: 0.6 };
}

// Function manager can route percentage of traffic to each version
// 90% → v1.0, 10% → v2.0 for testing
```

## Decorator Implementation

### Runtime Metadata Storage
```typescript
import 'reflect-metadata';

// Decorators store metadata for compile-time registry generation
export function EventHandler(eventType: string) {
  return function(target: any) {
    const existingEvents = Reflect.getMetadata('event-handlers', target) || [];
    Reflect.defineMetadata('event-handlers', [...existingEvents, eventType], target);
    Reflect.defineMetadata('function-name', target.name, target);
  };
}

export function ShardBy(strategy: string, value?: string) {
  return function(target: any) {
    Reflect.defineMetadata('shard-strategy', { type: strategy, value }, target);
  };
}

export function ResourceRequirements(requirements: {
  cpu: 'low' | 'medium' | 'high' | 'critical';
  memory: string;
  priority: 'low' | 'medium' | 'high' | 'critical';
  timeout: number;
}) {
  return function(target: any) {
    Reflect.defineMetadata('resource-requirements', requirements, target);
  };
}

export function ShardFilter(predicate: string) {
  return function(target: any) {
    Reflect.defineMetadata('shard-filter', predicate, target);
  };
}

export function Version(version: string) {
  return function(target: any) {
    Reflect.defineMetadata('version', version, target);
  };
}
```

## Compile-Time Registry Generation

### Build Process Integration
```typescript
// build-tools/generate-registry.ts
import * as ts from 'typescript';
import 'reflect-metadata';

/**
 * Scans source files for decorated functions and generates typed registry
 */
export class FunctionRegistryGenerator {
  generateRegistry(sourceFiles: string[]): string {
    const functions: FunctionDefinition[] = [];
    
    for (const file of sourceFiles) {
      const sourceFile = ts.createSourceFile(file, readFileSync(file, 'utf8'), ts.ScriptTarget.Latest);
      
      ts.forEachChild(sourceFile, (node) => {
        if (ts.isFunctionDeclaration(node) && this.hasEventHandlerDecorator(node)) {
          functions.push(this.extractFunctionDefinition(node));
        }
      });
    }
    
    return this.generateRegistryCode(functions);
  }
  
  private extractFunctionDefinition(node: ts.FunctionDeclaration): FunctionDefinition {
    const decorators = this.extractDecorators(node);
    
    return {
      name: node.name?.getText() || '',
      eventTypes: decorators.eventHandlers || [],
      shardingStrategy: decorators.shardBy || { type: 'round_robin' },
      resourceRequirements: decorators.resourceRequirements || defaultRequirements,
      inputType: this.extractInputType(node),
      outputType: this.extractOutputType(node),
      version: decorators.version || 'v1.0'
    };
  }
  
  private generateRegistryCode(functions: FunctionDefinition[]): string {
    return `
      // AUTO-GENERATED - DO NOT EDIT
      import { ${functions.map(f => f.name).join(', ')} } from './functions';
      
      export const TYPED_FUNCTION_REGISTRY = {
        ${functions.map(f => `
          ${f.name}: {
            name: '${f.name}',
            eventTypes: ${JSON.stringify(f.eventTypes)},
            shardingStrategy: ${JSON.stringify(f.shardingStrategy)},
            resourceRequirements: ${JSON.stringify(f.resourceRequirements)},
            version: '${f.version}',
            execute: ${f.name}
          }
        `).join(',')}
      } as const;
      
      // Type mapping for compile-time validation
      export type EventToFunctionMapping = {
        ${functions.map(f => 
          f.eventTypes.map(eventType => `
            '${eventType}': {
              input: ${f.inputType};
              output: ${f.outputType};
              functions: ['${f.name}'];
            };
          `).join('')
        ).join('')}
      };
    `;
  }
}
```

### TypeScript Compilation Integration
```json
{
  "scripts": {
    "prebuild": "node build-tools/generate-registry.js",
    "build": "tsc",
    "postbuild": "npm run validate-registry"
  }
}
```

## Usage Examples

### Simple Function Registration
```typescript
// src/domain/yield-functions.ts
@EventHandler('YieldOpportunityDetected')
@ShardBy('protocol', 'compound')
@ResourceRequirements({ cpu: 'medium', memory: '256MB', priority: 'high', timeout: 15000 })
export function calculateCompoundYield(
  context: EnhancedFunctionContext,
  data: YieldOpportunityData
): CompoundYieldStrategy {
  
  context.logger.info('Calculating Compound yield strategy', {
    apy: data.apy,
    protocol: data.protocol,
    shard: context.shard.id
  });
  
  // Pure calculation logic
  const strategy = {
    protocol: 'compound' as const,
    allocation: data.apy > 8 ? 0.4 : 0.2,
    confidence: calculateCompoundConfidence(data),
    compoundRate: data.apy / 365 // Daily compound rate
  };
  
  // Emit high-confidence opportunities
  if (strategy.confidence > 0.85) {
    context.emit('HighConfidenceCompoundYield', {
      strategy,
      timestamp: Date.now()
    });
  }
  
  context.addMetadata('compound.daily_rate', strategy.compoundRate);
  context.addMetadata('compound.confidence', strategy.confidence);
  
  return strategy;
}
```

### Complex Multi-Event Function
```typescript
// Handles multiple related events with different sharding
@EventHandler('ArbitrageOpportunityDetected')
@EventHandler('LiquidityUpdated')
@EventHandler('FeeStructureChanged')
@ShardBy('trading_pair')
@ResourceRequirements({ cpu: 'high', memory: '1GB', priority: 'critical', timeout: 45000 })
@Version('v2.1')
export function analyzeAdvancedArbitrage(
  context: EnhancedFunctionContext,
  data: ArbitrageOpportunityData | LiquidityData | FeeData
): AdvancedArbitrageStrategy {
  
  // Type guards for different event data
  if (isArbitrageData(data)) {
    return calculateArbitrageStrategy(context, data);
  } else if (isLiquidityData(data)) {
    return adjustForLiquidityChange(context, data);
  } else {
    return adjustForFeeChange(context, data);
  }
}
```

### Cross-Protocol Function
```typescript
// Function that works across multiple protocols
@EventHandler('CrossProtocolArbitrageDetected')
@ShardBy('amount_range') // Route by trade size, not protocol
@ShardFilter('data.protocols.includes("aave") && data.protocols.includes("compound")')
@ResourceRequirements({ cpu: 'critical', memory: '2GB', priority: 'critical', timeout: 60000 })
export function executeCrossProtocolArbitrage(
  context: EnhancedFunctionContext,
  data: CrossProtocolArbitrageData
): CrossProtocolStrategy {
  
  context.logger.info('Executing cross-protocol arbitrage', {
    protocols: data.protocols,
    estimated_profit: data.estimatedProfit,
    shard_specialization: context.shard.id
  });
  
  // Complex arbitrage calculation across multiple protocols
  const strategy = {
    sourceProtocol: data.protocols[0],
    targetProtocol: data.protocols[1], 
    executionSteps: planCrossProtocolExecution(data),
    expectedProfit: data.estimatedProfit,
    executionTime: calculateExecutionTime(data)
  };
  
  // Emit execution plan for other services
  context.emit('CrossProtocolExecutionPlanned', {
    strategy,
    requiredApprovals: ['risk-manager', 'compliance-checker'],
    urgency: strategy.executionTime < 300000 ? 'high' : 'normal'
  });
  
  return strategy;
}
```

## Registry Generation Output

### Generated Registry File
```typescript
// AUTO-GENERATED by build process
export const TYPED_FUNCTION_REGISTRY = {
  calculateAaveYield: {
    name: 'calculateAaveYield',
    eventTypes: ['YieldOpportunityDetected'],
    shardingStrategy: { type: 'protocol', value: 'aave' },
    resourceRequirements: { cpu: 'medium', memory: '256MB', priority: 'high', timeout: 15000 },
    version: 'v1.0',
    execute: calculateAaveYield
  },
  calculateCompoundYield: {
    name: 'calculateCompoundYield', 
    eventTypes: ['YieldOpportunityDetected'],
    shardingStrategy: { type: 'protocol', value: 'compound' },
    resourceRequirements: { cpu: 'medium', memory: '256MB', priority: 'high', timeout: 15000 },
    version: 'v1.0',
    execute: calculateCompoundYield
  },
  detectCrossExchangeArbitrage: {
    name: 'detectCrossExchangeArbitrage',
    eventTypes: ['ArbitrageOpportunityDetected'], 
    shardingStrategy: { type: 'trading_pair', value: null },
    resourceRequirements: { cpu: 'high', memory: '1GB', priority: 'critical', timeout: 45000 },
    version: 'v2.1',
    execute: detectCrossExchangeArbitrage
  }
} as const;

// Type mappings for compile-time validation
export type EventToFunctionMapping = {
  'YieldOpportunityDetected': {
    input: YieldOpportunityData;
    output: AaveYieldStrategy | CompoundYieldStrategy;
    functions: ['calculateAaveYield', 'calculateCompoundYield'];
  };
  'ArbitrageOpportunityDetected': {
    input: ArbitrageOpportunityData;
    output: ArbitrageStrategy;
    functions: ['detectCrossExchangeArbitrage'];
  };
};
```

## TypeScript Integration

### Strict Type Checking
```typescript
// Function manager gets full type safety
class YieldFunctionManager extends LucentServiceBase {
  async processYieldEvent(event: DomainEvent<YieldOpportunityData>): Promise<void> {
    //                                           ^^^^^^^^^^^^^^^^^ ✅ TypeScript validates event data type
    
    const functions = this.getFunctionsForEvent('YieldOpportunityDetected');
    //                                          ^^^^^^^^^^^^^^^^^^^^^^^^^ ✅ TypeScript knows this returns yield functions
    
    const selectedFunction = await this.selectOptimalFunction(functions);
    const result = selectedFunction.execute(context, event.data);
    //                                              ^^^^^^^^^^ ✅ TypeScript knows this is YieldOpportunityData
    //             ^^^^^^ ✅ TypeScript knows result is AaveYieldStrategy | CompoundYieldStrategy
  }
}
```

### IDE Integration
```typescript
// Developer gets full autocomplete and validation
@EventHandler('YieldOpportunityDetected')
export function myYieldFunction(
  context: EnhancedFunctionContext,
  data: YieldOpportunityData // ✅ IDE shows: apy, tvl, riskScore, etc.
): YieldStrategy {            // ✅ IDE validates return object
  
  // ✅ Full intellisense on context.logger, context.shard, etc.
  context.logger.debug('Processing yield', {
    apy: data.apy,           // ✅ IDE knows data.apy is number
    protocol: data.protocol // ✅ IDE knows data.protocol is string
  });
  
  return {
    protocol: data.protocol, // ✅ IDE validates this exists on YieldStrategy
    allocation: 0.3,         // ✅ IDE validates this is number
    confidence: 0.8          // ✅ IDE validates this exists
  };
}
```

## Build Process Integration

### Package.json Scripts
```json
{
  "scripts": {
    "generate-registry": "ts-node build-tools/generate-function-registry.ts",
    "prebuild": "npm run generate-registry",
    "build": "tsc",
    "validate-types": "tsc --noEmit",
    "dev": "npm run generate-registry && tsc --watch"
  }
}
```

### Registry Generation Tool
```typescript
// build-tools/generate-function-registry.ts
import { FunctionRegistryGenerator } from '@lucent/infrastructure/build-tools';

const generator = new FunctionRegistryGenerator();

// Scan all domain function files
const sourceFiles = glob.sync('src/domain/**/*.ts');
const registryCode = generator.generateRegistry(sourceFiles);

// Write typed registry file
writeFileSync('src/registry/function-registry.ts', registryCode);

console.log(`Generated registry with ${sourceFiles.length} functions`);
```

## Error Handling

### Decorator Validation
```typescript
// Compile-time validation of decorator usage
@EventHandler('InvalidEventType') // ❌ Build fails - event type not defined
export function invalidFunction(context, data) { }

@ShardBy('invalid_strategy')      // ❌ Build fails - strategy not supported
export function anotherInvalidFunction(context, data) { }

@ResourceRequirements({ cpu: 'invalid' }) // ❌ Build fails - invalid CPU level
export function resourceInvalidFunction(context, data) { }
```

### Runtime Validation
```typescript
// Registry validates function compatibility at startup
class FunctionManager {
  async validateRegistry(): Promise<void> {
    for (const [name, func] of Object.entries(TYPED_FUNCTION_REGISTRY)) {
      // Validate event types exist
      for (const eventType of func.eventTypes) {
        if (!VALID_EVENT_TYPES.includes(eventType)) {
          throw new Error(`Function ${name} references invalid event type: ${eventType}`);
        }
      }
      
      // Validate resource requirements
      if (!this.shardCanSupportFunction(func)) {
        throw new Error(`Shard cannot support function ${name} resource requirements`);
      }
    }
  }
}
```

## Benefits Summary

### For Developers
- ✅ **Declarative configuration**: Just add decorators, everything else automatic
- ✅ **Full TypeScript support**: Autocomplete, validation, refactoring
- ✅ **Zero boilerplate**: No manual registry management
- ✅ **Clear intent**: Decorators self-document function behavior

### For Operations
- ✅ **Automatic discovery**: Build process finds all functions
- ✅ **Load optimization**: Functions route to optimal shards automatically
- ✅ **Resource planning**: Resource requirements declared upfront
- ✅ **Version management**: A/B testing and gradual rollouts built-in

### For Architecture
- ✅ **Separation of concerns**: Business logic separate from infrastructure
- ✅ **Type safety**: Full compile-time validation despite dynamic execution
- ✅ **Scalability**: Functions distribute across shards automatically
- ✅ **Maintainability**: Functions are pure and independently testable

This decorator pattern enables **Netflix-style dynamic function distribution** while maintaining **Google-level type safety** and **clean architectural separation**.