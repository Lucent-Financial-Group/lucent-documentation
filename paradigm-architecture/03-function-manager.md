# Function Manager Pattern

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Generic function orchestration with load-balanced pure function execution

## Overview

The **Generic Function Manager** serves as the **imperative shell** that coordinates I/O operations and orchestrates execution of **any** pure functions across distributed shards. It uses the compile-time generated function registry and decorator metadata to provide domain-agnostic load balancing and routing.

## Generic Function Manager Architecture

### Core Responsibilities
1. **Event Processing**: Receive and route domain events of any type
2. **Function Discovery**: Use generated registry to find compatible functions  
3. **Load Balancing**: Select optimal function based on current shard load
4. **Generic Routing**: Route events using decorator metadata (no domain-specific logic)
5. **Result Handling**: Process function outputs and emit resulting events
6. **Performance Monitoring**: Track function execution across all domains
7. **Error Recovery**: Handle function failures generically

### Single Generic Implementation

```typescript
import { LucentServiceBase, BusinessContextBuilder } from 'lucent-infrastructure';

/**
 * Function Manager orchestrates ALL pure functions across ANY domain
 */
export class FunctionManager extends LucentServiceBase {
  constructor(
    private shardId: string,
    private availableShards: string[]
  ) {
    const infrastructure = InfrastructureFactory.createDefault();
    super(infrastructure, `function-manager-${shardId}`);
    
    this.setupGenericEventSubscriptions();
    this.initializeGenericLoadBalancing();
    this.registerGenericHealthChecks();
  }
  
  /**
   * Process domain events from NATS subscriptions
   */
  async processIncomingEvent<T>(event: DomainEvent<T>): Promise<void> {
    const businessContext = this.extractBusinessContextFromEvent(event);
    
    return this.handleDomainEvent(event, async (event, context) => {
      
      // 1. Determine target shard using decorator metadata (generic)
      const targetShard = await this.determineTargetShardGeneric(event);
      
      if (targetShard !== this.shardId) {
        return this.forwardEventToShard(targetShard, event, context);
      }
      
      // 2. Get compatible functions from generated registry (generic)
      const availableFunctions = this.getFunctionsForEvent(event.eventType);
      
      if (availableFunctions.length === 0) {
        this.logger.warn('No functions available for event type', {
          event_type: event.eventType,
          shard_id: this.shardId,
          available_shards: this.availableShards
        });
        return;
      }
      
      // 3. Select optimal function using generic load balancing
      const selectedFunction = await this.selectOptimalFunctionGeneric(availableFunctions);
      
      // 4. Execute pure function with enhanced context (generic)
      const functionContext = this.createEnhancedFunctionContext(context);
      const result = selectedFunction.execute(functionContext, event.data);
      
      // 5. Handle function result generically
      await this.handleFunctionResultGeneric(selectedFunction, result, context);
    });
  }

  /**
   * Generic function selection based on decorator metadata only
   */
  private async selectOptimalFunctionGeneric(functions: FunctionRegistryEntry[]): Promise<FunctionRegistryEntry> {
    if (functions.length === 1) {
      return functions[0];
    }

    // Generic scoring algorithm (no domain-specific logic)
    const functionScores = await Promise.all(
      functions.map(async func => ({
        function: func,
        loadScore: await this.getFunctionLoad(func.name),
        resourceScore: await this.getResourceAvailability(func.resourceRequirements),
        priorityScore: this.getPriorityScore(func.resourceRequirements.priority),
        shardAffinityScore: await this.getShardAffinity(func.shardingStrategy)
      }))
    );

    // Calculate composite scores (domain-agnostic)
    const scoredFunctions = functionScores.map(fs => ({
      ...fs,
      compositeScore: 
        (fs.loadScore * 0.3) +         // Current function load
        (fs.resourceScore * 0.25) +    // Resource availability
        (fs.priorityScore * 0.25) +    // Business priority from @ResourceRequirements
        (fs.shardAffinityScore * 0.2)  // Shard affinity from @ShardBy
    }));

    const selected = scoredFunctions.sort((a, b) => b.compositeScore - a.compositeScore)[0];
    
    this.logger.info('Function selected by generic load balancing', {
      function_name: selected.function.name,
      load_score: selected.loadScore,
      resource_score: selected.resourceScore,
      priority_score: selected.priorityScore,
      affinity_score: selected.shardAffinityScore,
      composite_score: selected.compositeScore,
      shard_id: this.shardId
    });

    return selected.function;
  }

  /**
   * Generic shard routing using decorator metadata
   */
  private async determineTargetShardGeneric(event: DomainEvent): Promise<string> {
    const functions = this.getFunctionsForEvent(event.eventType);
    
    for (const func of functions) {
      const strategy = func.shardingStrategy;
      
      // Generic routing based on @ShardBy decorators (no domain logic)
      const targetShard = await this.routeByStrategy(strategy, event.data);
      if (targetShard) {
        return targetShard;
      }
    }
    
    // Fallback to load-balanced routing
    return this.getLeastLoadedShard();
  }

  /**
   * Generic strategy routing (works for any data type)
   */
  private async routeByStrategy(strategy: ShardingStrategy, eventData: any): Promise<string | null> {
    switch (strategy.type) {
      case 'round_robin':
        return this.getNextShardRoundRobin();
        
      case 'hash_based':
        const hashKey = this.extractHashKey(eventData, strategy.hashField);
        return this.getShardByHash(hashKey);
        
      case 'load_balanced':
        return this.getLeastLoadedShard();
        
      case 'sticky_session':
        const sessionKey = this.extractSessionKey(eventData, strategy.sessionField);
        return this.getShardBySession(sessionKey);
        
      default:
        return null; // Unknown strategy
    }
  }

  /**
   * Handle function results generically
   */
  private async handleFunctionResultGeneric(
    func: FunctionRegistryEntry,
    result: any,
    context: EventProcessingContext
  ): Promise<void> {
    
    if (!result) return;

    // Generic result handling (no domain-specific logic)
    
    // 1. Store function result as domain event
    await this.publishDomainEvent(
      `${func.name}Completed`,
      `function-result-${context.correlationId}`,
      result,
      context.businessContext,
      context
    );

    // 2. Update generic performance metrics
    await this.recordFunctionExecution(func.name, context);
    
    // 3. Update shard load metrics
    await this.updateShardLoadMetrics(func.name);
    
    // 4. Handle function-specific emissions (if any)
    if (result.additionalEvents) {
      for (const additionalEvent of result.additionalEvents) {
        await this.publishDomainEvent(
          additionalEvent.eventType,
          additionalEvent.streamId,
          additionalEvent.data,
          context.businessContext,
          context
        );
      }
    }
  }

  /**
   * Handle function execution requests from service classes
   */
  async handleExecuteFunction(request: FunctionExecutionRequest): Promise<void> {
    return this.handleDomainEvent(request, async (request, context) => {
      
      // Same logic as processIncomingEvent but for service requests
      const targetShard = await this.determineTargetShardGeneric({ eventType: request.data.eventType, data: request.data.data });
      
      if (targetShard !== this.shardId) {
        return this.forwardToShard(targetShard, request);
      }

      const functions = this.getFunctionsForEvent(request.data.eventType);
      const selectedFunction = await this.selectOptimalFunctionGeneric(functions);
      
      const functionContext = this.createEnhancedFunctionContext(context);
      const result = selectedFunction.execute(functionContext, request.data.data);
      
      await this.handleFunctionResultGeneric(selectedFunction, result, context);
      
      // Send response back to requesting service
      await this.sendResponse(request.requestId, {
        success: true,
        result: result,
        functionName: selectedFunction.name,
        shardId: this.shardId
      });
    });
  }
}