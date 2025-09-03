# Function Manager Pattern

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Imperative shell coordination with load-balanced pure function execution

## Overview

Function Managers serve as the **imperative shell** that coordinates I/O operations, manages infrastructure concerns, and orchestrates execution of pure functions across distributed shards. They extend Lucent base classes to inherit enterprise-grade infrastructure capabilities while maintaining clean separation from business logic.

## Function Manager Architecture

### Core Responsibilities
1. **Event Processing**: Receive and route domain events
2. **Function Selection**: Choose optimal function based on load and resources  
3. **Shard Coordination**: Route events to appropriate shards
4. **Context Management**: Provide enhanced context to pure functions
5. **Result Handling**: Process function outputs and emit resulting events
6. **Load Balancing**: Monitor and optimize function distribution
7. **Error Recovery**: Handle function failures and implement retry logic

### Base Class Integration
```typescript
import { LucentServiceBase, CryptoTradingServiceBase, BusinessContextBuilder } from '@lucent/infrastructure';

/**
 * Generic function manager for any business domain
 */
export class GenericFunctionManager extends LucentServiceBase {
  constructor(
    private shardId: string,
    private availableShards: string[],
    private functionRegistry: FunctionRegistry
  ) {
    const infrastructure = InfrastructureFactory.createDefault();
    super(infrastructure, `function-manager-${shardId}`);
    
    this.setupEventSubscriptions();
    this.initializeLoadBalancing();
    this.registerHealthChecks();
  }
  
  /**
   * Process incoming domain event with full type safety
   */
  async processIncomingEvent<T>(event: DomainEvent<T>): Promise<void> {
    // Extract business context from event (base class provides this)
    const businessContext = this.extractBusinessContextFromEvent(event);
    
    // Use base class event handling pattern
    return this.handleDomainEvent(event, async (event, context) => {
      
      // 1. Determine if this shard should handle this event
      const targetShard = await this.determineTargetShard(event);
      
      if (targetShard !== this.shardId) {
        return this.forwardToOptimalShard(targetShard, event, context);
      }
      
      // 2. Select optimal function for execution
      const availableFunctions = this.getCompatibleFunctions(event.eventType);
      const selectedFunction = await this.selectFunctionByLoad(availableFunctions);
      
      // 3. Execute function with enhanced context
      await this.executeFunctionWithContext(selectedFunction, event, context);
    });
  }
}

/**
 * Specialized function manager for crypto trading domain
 */
export class CryptoFunctionManager extends CryptoTradingServiceBase {
  private protocolShards: Map<string, string> = new Map();
  private riskShards: Map<string, string> = new Map();
  
  constructor(shardId: string, specializations: string[]) {
    const infrastructure = InfrastructureFactory.createDefault();
    super(infrastructure, `crypto-function-manager-${shardId}`);
    
    this.configureShardSpecializations(specializations);
    this.setupCryptoEventSubscriptions();
  }
  
  /**
   * Process yield farming events with crypto-specific logic
   */
  async processYieldFarmingEvent(event: YieldOpportunityDetectedEvent): Promise<void> {
    // Use crypto trading base class patterns
    return this.analyzeYieldOpportunity(
      event.data.protocol,
      event.data.poolId,
      event.metadata.userId || 'system',
      async (context) => {
        
        // Route to protocol-specialized shard if needed
        const protocolShard = this.protocolShards.get(event.data.protocol);
        if (protocolShard && protocolShard !== this.shardId) {
          return this.delegateToProtocolShard(protocolShard, event, context);
        }
        
        // Execute locally with load balancing
        const functions = this.getCryptoFunctionsForEvent('YieldOpportunityDetected');
        const selectedFunction = await this.selectByCryptoLoad(functions, event.data);
        
        const result = await this.executeWithCryptoContext(selectedFunction, event, context);
        
        // Publish crypto-specific result events
        await this.publishCryptoEvent('YieldStrategyCalculated', result, context);
        
        return result;
      }
    );
  }
  
  /**
   * Crypto-specific load balancing considering trading pair affinity
   */
  private async selectByCryptoLoad<TIn, TOut>(
    functions: CryptoFunction<TIn, TOut>[],
    eventData: any
  ): Promise<CryptoFunction<TIn, TOut>> {
    
    const tradingPair = eventData.tradingPair || eventData.pair;
    const riskLevel = eventData.riskLevel || 'medium';
    const amountUsd = eventData.amountUsd || 0;
    
    // Weight functions by crypto-specific factors
    const weightedFunctions = await Promise.all(
      functions.map(async (func) => ({
        function: func,
        load: await this.getCryptoFunctionLoad(func.name),
        pairAffinity: this.calculatePairAffinity(func, tradingPair),
        riskSuitability: this.calculateRiskSuitability(func, riskLevel),
        amountSuitability: this.calculateAmountSuitability(func, amountUsd),
        shardOptimization: await this.getShardOptimization(func)
      }))
    );
    
    // Calculate composite score
    const scoredFunctions = weightedFunctions.map(wf => ({
      ...wf,
      score: this.calculateCryptoScore(wf)
    }));
    
    // Select highest scoring function
    const selected = scoredFunctions.sort((a, b) => b.score - a.score)[0];
    
    this.logger.info('Crypto function selected', {
      function_name: selected.function.name,
      load: selected.load,
      pair_affinity: selected.pairAffinity,
      risk_suitability: selected.riskSuitability,
      composite_score: selected.score,
      trading_pair: tradingPair,
      risk_level: riskLevel
    });
    
    return selected.function;
  }
}
```

## Load Balancing Implementation

### Shard Load Monitoring
```typescript
export class ShardLoadMonitor {
  private shardMetrics: Map<string, ShardMetrics> = new Map();
  
  constructor(private messageBus: IMessageBus) {
    this.setupMetricsCollection();
  }
  
  async getCurrentShardLoad(shardId: string): Promise<number> {
    const metrics = this.shardMetrics.get(shardId);
    if (!metrics) return 0.5; // Default medium load
    
    // Composite load score
    const cpuLoad = metrics.cpuUsage / 100;
    const memoryLoad = metrics.memoryUsage / metrics.totalMemory;
    const functionLoad = metrics.activeFunctions / metrics.maxFunctions;
    const queueLoad = metrics.queueDepth / metrics.maxQueueDepth;
    
    return (cpuLoad * 0.3) + (memoryLoad * 0.3) + (functionLoad * 0.25) + (queueLoad * 0.15);
  }
  
  async recordFunctionExecution(
    shardId: string,
    functionName: string, 
    executionTime: number,
    resourceUsage: ResourceUsage
  ): Promise<void> {
    
    // Update function-specific metrics
    const functionMetrics = await this.getFunctionMetrics(shardId, functionName);
    functionMetrics.totalExecutions++;
    functionMetrics.totalExecutionTime += executionTime;
    functionMetrics.avgExecutionTime = functionMetrics.totalExecutionTime / functionMetrics.totalExecutions;
    functionMetrics.lastExecuted = new Date();
    
    // Update shard-level metrics
    const shardMetrics = this.shardMetrics.get(shardId)!;
    shardMetrics.cpuUsage = resourceUsage.cpuUsage;
    shardMetrics.memoryUsage = resourceUsage.memoryUsage;
    shardMetrics.activeFunctions = this.countActiveFunctions(shardId);
    shardMetrics.queueDepth = await this.getQueueDepth(shardId);
    
    // Publish metrics for cross-shard visibility
    await this.messageBus.publish(`shard.metrics.${shardId}`, {
      shardId,
      load: await this.getCurrentShardLoad(shardId),
      functionMetrics: functionMetrics,
      timestamp: Date.now()
    });
  }
}
```

### Dynamic Function Routing
```typescript
export class FunctionRouter {
  constructor(
    private loadMonitor: ShardLoadMonitor,
    private shardTopology: ShardTopology
  ) {}
  
  async routeEventToOptimalShard(event: DomainEvent): Promise<string> {
    // Get all shards that can handle this event type
    const capableShards = await this.getShardsForEventType(event.eventType);
    
    // Apply sharding strategy filters
    const filteredShards = await this.applyShardingFilters(event, capableShards);
    
    // Select optimal shard based on current load
    const shardLoads = await Promise.all(
      filteredShards.map(async (shardId) => ({
        shardId,
        load: await this.loadMonitor.getCurrentShardLoad(shardId),
        affinity: await this.calculateShardAffinity(shardId, event),
        capacity: await this.getAvailableCapacity(shardId)
      }))
    );
    
    // Filter out overloaded shards
    const availableShards = shardLoads.filter(shard => 
      shard.load < 0.9 && shard.capacity > this.getRequiredCapacity(event)
    );
    
    if (availableShards.length === 0) {
      throw new Error(`No available shards for event ${event.eventType}`);
    }
    
    // Select shard with best composite score
    const scoredShards = availableShards.map(shard => ({
      ...shard,
      score: this.calculateShardScore(shard)
    }));
    
    const optimalShard = scoredShards.sort((a, b) => b.score - a.score)[0];
    
    return optimalShard.shardId;
  }
  
  private calculateShardScore(shard: {
    shardId: string;
    load: number;
    affinity: number;
    capacity: number;
  }): number {
    // Lower load is better (invert)
    const loadScore = 1 - shard.load;
    // Higher affinity is better
    const affinityScore = shard.affinity;
    // Higher available capacity is better
    const capacityScore = shard.capacity;
    
    return (loadScore * 0.4) + (affinityScore * 0.35) + (capacityScore * 0.25);
  }
}
```

## Integration with Base Classes

### Event Subscription Setup
```typescript
export class YieldFunctionManager extends CryptoTradingServiceBase {
  async initialize(): Promise<void> {
    // Base class provides infrastructure initialization
    await super.initialize();
    
    // Subscribe to relevant events using base class messaging
    await this.subscribeToYieldEvents();
    await this.subscribeToMarketEvents();
    await this.subscribeToRiskEvents();
  }
  
  private async subscribeToYieldEvents(): Promise<void> {
    // Use base class NATS integration
    await this.infrastructure.messageBus.subscribe<YieldOpportunityData>(
      'lucent.events.domain.yield_farming.YieldOpportunityDetected',
      async (event, traceContext) => {
        await this.processYieldEvent(event, traceContext);
      }
    );
  }
  
  private async processYieldEvent(
    event: YieldOpportunityData, 
    traceContext: TraceContext
  ): Promise<void> {
    
    const domainEvent: DomainEvent<YieldOpportunityData> = {
      eventId: 'generated-id',
      eventType: 'YieldOpportunityDetected',
      streamId: 'yield-stream',
      timestamp: new Date().toISOString(),
      data: event,
      metadata: { source: 'yield-manager', version: '1.0.0' }
    };
    
    // Process using base class patterns
    await this.processIncomingEvent(domainEvent);
  }
}
```

### Health Check Integration
```typescript
export class FunctionManager extends LucentServiceBase {
  constructor(shardId: string) {
    super(infrastructure, `function-manager-${shardId}`);
    
    // Register shard-specific health checks
    this.registerShardHealthChecks();
    this.setupFunctionHealthMonitoring();
  }
  
  private registerShardHealthChecks(): void {
    // Use base class health monitoring
    this.healthMonitor.registerCheck({
      name: 'function-execution-health',
      interval: 30000,
      timeout: 10000,
      retries: 3,
      check: async () => {
        const executionMetrics = await this.getFunctionExecutionMetrics();
        
        return {
          healthy: executionMetrics.errorRate < 0.05, // Under 5% error rate
          latency: executionMetrics.avgExecutionTime,
          message: `Function execution health: ${executionMetrics.errorRate * 100}% error rate`,
          details: {
            total_executions: executionMetrics.totalExecutions,
            avg_execution_time: executionMetrics.avgExecutionTime,
            active_functions: executionMetrics.activeFunctions,
            shard_load: await this.loadMonitor.getCurrentShardLoad(this.shardId)
          },
          timestamp: new Date()
        };
      }
    });
  }
}
```

### Cross-Shard Communication
```typescript
export class DistributedFunctionManager extends CryptoTradingServiceBase {
  /**
   * Execute function across multiple shards for complex analysis
   */
  async executeDistributedAnalysis(request: ComplexAnalysisRequest): Promise<AnalysisResult> {
    const businessContext = BusinessContextBuilder
      .forYieldFarming()
      .withAggregate('DistributedAnalysis', `analysis-${request.id}`)
      .withUser(request.userId)
      .withWorkflow(`distributed-analysis-${request.id}`)
      .build();
    
    return this.executeBusinessOperation('distributed_analysis', businessContext, async (context) => {
      
      // 1. Break down analysis into shard-specific tasks
      const shardTasks = this.createShardTasks(request);
      
      // 2. Execute tasks on optimal shards in parallel
      const shardResults = await Promise.all(
        shardTasks.map(async (task) => {
          const targetShard = await this.selectOptimalShardForTask(task);
          
          // Use base class cross-service communication
          return this.sendCommand<ShardTask, ShardResult>(
            `function-manager-${targetShard}`,
            'ExecuteShardTask',
            task,
            businessContext,
            task.timeoutMs
          );
        })
      );
      
      // 3. Aggregate results using pure function
      const aggregationFunction = this.functionRegistry.getFunction('aggregateAnalysisResults');
      const finalResult = aggregationFunction.execute(
        this.createAggregationContext(context),
        shardResults
      );
      
      // 4. Publish consolidated result
      await this.publishDomainEvent(
        'DistributedAnalysisCompleted',
        `analysis-${request.id}`,
        finalResult,
        businessContext,
        context
      );
      
      return finalResult;
    });
  }
  
  private createShardTasks(request: ComplexAnalysisRequest): ShardTask[] {
    return [
      {
        type: 'yield_calculation',
        protocols: ['aave', 'compound'],
        targetShard: 'protocol-analysis',
        timeoutMs: 30000,
        data: request.yieldData
      },
      {
        type: 'arbitrage_detection', 
        tradingPairs: request.tradingPairs,
        targetShard: 'arbitrage-analysis',
        timeoutMs: 15000,
        data: request.arbitrageData
      },
      {
        type: 'risk_assessment',
        portfolioValue: request.portfolioValue,
        targetShard: 'risk-analysis', 
        timeoutMs: 45000,
        data: request.riskData
      }
    ];
  }
}
```

## Function Execution Pipeline

### 1. Event Reception
```typescript
class FunctionManager extends LucentServiceBase {
  /**
   * Central event processing pipeline
   */
  async processEvent<T>(event: DomainEvent<T>): Promise<void> {
    // Use base class tracing and error handling
    return withSpan(`function-manager.process.${event.eventType}`, async (span) => {
      
      this.logger.info('Processing event in function manager', {
        event_type: event.eventType,
        shard_id: this.shardId,
        correlation_id: event.metadata.correlationId
      });
      
      try {
        // 1. Validate event structure
        this.validateEventStructure(event);
        
        // 2. Check shard routing
        const shouldProcess = await this.shouldProcessLocally(event);
        if (!shouldProcess) {
          return this.routeToOptimalShard(event);
        }
        
        // 3. Select and execute function
        await this.selectAndExecuteFunction(event);
        
      } catch (error: any) {
        // Base class provides structured error handling
        this.logger.error('Function processing failed', {
          event_type: event.eventType,
          shard_id: this.shardId,
          error: error.message
        }, error);
        
        // Implement retry logic
        await this.handleFunctionError(event, error);
        throw error;
      }
    });
  }
}
```

### 2. Function Selection
```typescript
export class IntelligentFunctionSelector {
  async selectOptimalFunction<TIn, TOut>(
    eventType: string,
    availableFunctions: TypedFunction<TIn, TOut>[],
    eventData: TIn,
    shardCapacity: ShardCapacity
  ): Promise<TypedFunction<TIn, TOut>> {
    
    // 1. Filter functions by resource availability
    const viableFunctions = availableFunctions.filter(func => 
      this.canExecuteFunction(func, shardCapacity)
    );
    
    if (viableFunctions.length === 0) {
      throw new ResourceExhaustionError(
        'No functions available with sufficient resources',
        'function_selection',
        { eventType, requiredResources: availableFunctions.map(f => f.resourceRequirements) }
      );
    }
    
    // 2. Score functions by multiple criteria
    const scoredFunctions = await Promise.all(
      viableFunctions.map(async (func) => ({
        function: func,
        loadScore: await this.calculateLoadScore(func),
        affinityScore: this.calculateAffinityScore(func, eventData),
        performanceScore: await this.calculatePerformanceScore(func),
        priorityScore: this.calculatePriorityScore(func),
        compositeScore: 0
      }))
    );
    
    // 3. Calculate composite scores
    scoredFunctions.forEach(sf => {
      sf.compositeScore = 
        (sf.loadScore * 0.3) +       // Prefer less loaded functions
        (sf.affinityScore * 0.25) +   // Prefer functions with data affinity
        (sf.performanceScore * 0.25) + // Prefer faster functions
        (sf.priorityScore * 0.2);     // Consider business priority
    });
    
    // 4. Select highest scoring function
    const selected = scoredFunctions.sort((a, b) => b.compositeScore - a.compositeScore)[0];
    
    return selected.function;
  }
  
  private calculateAffinityScore<TIn>(func: TypedFunction<TIn, any>, eventData: TIn): number {
    // Crypto-specific affinity scoring
    const shardingStrategy = func.shardingStrategy;
    
    switch (shardingStrategy.type) {
      case 'protocol':
        const protocol = (eventData as any).protocol;
        return protocol === shardingStrategy.value ? 1.0 : 0.0;
        
      case 'trading_pair':
        const pair = (eventData as any).tradingPair;
        // Functions that previously processed this pair get higher affinity
        return this.getPairAffinityScore(func.name, pair);
        
      case 'risk_level':
        const riskLevel = (eventData as any).riskLevel;
        return this.getRiskAffinityScore(func.name, riskLevel);
        
      default:
        return 0.5; // Neutral affinity
    }
  }
}
```

### 3. Function Execution with Context
```typescript
export class FunctionExecutor extends LucentServiceBase {
  async executeWithEnhancedContext<TIn, TOut>(
    func: TypedFunction<TIn, TOut>,
    event: DomainEvent<TIn>,
    requestContext: RequestContext
  ): Promise<TOut> {
    
    // Create enhanced context for pure function
    const functionContext: EnhancedFunctionContext = {
      correlationId: requestContext.correlationId,
      businessContext: requestContext.businessContext,
      logger: this.logger.child(`function.${func.name}`),
      shard: await this.getShardInfo(),
      metrics: await this.getFunctionMetrics(func.name),
      emit: async (eventType: string, data: any) => {
        // Pure functions can emit events through context
        await this.publishDomainEvent(
          eventType,
          `${func.name}-emission-${requestContext.correlationId}`,
          data,
          requestContext.businessContext,
          requestContext
        );
      },
      addMetadata: (key: string, value: any) => {
        // Add metadata to current trace span
        getCurrentSpan()?.setAttributes({ [`function.${key}`]: JSON.stringify(value) });
      }
    };
    
    // Execute pure function with full tracing
    return withSpan(`function.execute.${func.name}`, async (span) => {
      
      span.setAttributes({
        'function.name': func.name,
        'function.version': func.version,
        'function.shard': this.shardId,
        'function.event_type': event.eventType,
        'function.resource_cpu': func.resourceRequirements.cpu,
        'function.resource_memory': func.resourceRequirements.memory
      });
      
      const startTime = Date.now();
      
      try {
        // Execute pure function
        const result = func.execute(functionContext, event.data);
        
        const executionTime = Date.now() - startTime;
        
        // Record performance metrics
        await this.recordFunctionPerformance(func.name, executionTime, true);
        
        span.setAttributes({
          'function.execution_time_ms': executionTime,
          'function.success': true
        });
        
        this.logger.info('Function executed successfully', {
          function_name: func.name,
          execution_time_ms: executionTime,
          event_type: event.eventType,
          correlation_id: requestContext.correlationId
        });
        
        return result;
        
      } catch (error: any) {
        const executionTime = Date.now() - startTime;
        
        // Record error metrics
        await this.recordFunctionPerformance(func.name, executionTime, false);
        
        span.recordException(error);
        span.setAttributes({
          'function.execution_time_ms': executionTime,
          'function.success': false,
          'function.error_type': error.constructor.name
        });
        
        this.logger.error('Function execution failed', {
          function_name: func.name,
          execution_time_ms: executionTime,
          error: error.message,
          event_type: event.eventType,
          correlation_id: requestContext.correlationId
        }, error);
        
        throw error;
      }
    }, requestContext.traceContext);
  }
}
```

## Deployment and Scaling

### Shard Configuration
```typescript
// config/shard-topology.ts
export const SHARD_TOPOLOGY = {
  'shard-aave-primary': {
    specializations: ['aave', 'yield-farming'],
    resourceProfile: 'medium-cpu-high-memory',
    maxFunctions: 50,
    functions: ['calculateAaveYield', 'optimizeAaveStrategy', 'assessAaveRisk']
  },
  'shard-arbitrage-primary': {
    specializations: ['arbitrage', 'cross-exchange'],
    resourceProfile: 'high-cpu-medium-memory',
    maxFunctions: 30,
    functions: ['detectArbitrage', 'executeArbitrageStrategy', 'optimizeArbitrageExecution']
  },
  'shard-whale-trades': {
    specializations: ['high-value-trading'],
    resourceProfile: 'critical-cpu-high-memory',
    maxFunctions: 20,
    functions: ['executeWhaleStrategy', 'assessInstitutionalRisk', 'optimizeWhalePortfolio']
  }
};
```

### Dynamic Scaling
```typescript
export class FunctionManagerScaler extends LucentServiceBase {
  async monitorAndScale(): Promise<void> {
    const metrics = await this.getAllShardMetrics();
    
    for (const [shardId, metric] of metrics) {
      if (metric.load > 0.85) {
        // Shard overloaded - redistribute functions
        await this.redistributeFunctions(shardId);
      }
      
      if (metric.load < 0.2 && this.canConsolidateShard(shardId)) {
        // Shard underutilized - consolidate functions
        await this.consolidateShard(shardId);
      }
    }
  }
  
  private async redistributeFunctions(overloadedShard: string): Promise<void> {
    const functions = await this.getShardFunctions(overloadedShard);
    const candidateShards = await this.getAvailableShards();
    
    // Move non-critical functions to less loaded shards
    for (const func of functions) {
      if (func.resourceRequirements.priority !== 'critical') {
        const targetShard = await this.selectLeastLoadedShard(candidateShards);
        await this.migrateFunctionToShard(func.name, overloadedShard, targetShard);
      }
    }
  }
}
```

This function manager pattern provides the **perfect bridge** between your pure functional core and the infrastructure base classes, enabling dynamic load balancing while maintaining clean architecture and type safety.