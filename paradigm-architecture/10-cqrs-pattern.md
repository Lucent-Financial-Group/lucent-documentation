# CQRS Pattern Integration

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Command Query Responsibility Segregation with pure function architecture

## Overview

The CQRS (Command Query Responsibility Segregation) pattern **separates write operations (commands) from read operations (queries)** to optimize for different performance characteristics. This integrates seamlessly with our pure function architecture to provide both **synchronous command handling** and **asynchronous event processing**.

## CQRS Architecture Integration

### Complete Architecture with CQRS
```
┌─────────────────────────────────────────────────────────────────┐
│                    HTTP/API Layer                                │
├─────────────────────────────────────────────────────────────────┤
│               Command Side (Writes)    │    Query Side (Reads)   │
│  Commands → Command Handlers → Events  │  Queries → Query Cache  │
├─────────────────────────────────────────────────────────────────┤
│                 Function Manager (Pure Functions)               │
│  - Event Processing      - Load Balancing     - Function Exec   │
├─────────────────────────────────────────────────────────────────┤
│                    Infrastructure Layer                         │
│  EventStore (Events) │ Redis (Projections) │ ClickHouse (Analytics) │
└─────────────────────────────────────────────────────────────────┘
```

## Command Side (Synchronous Writes)

### 1. Command Handlers
```typescript
/**
 * Pure function command handler for trade execution
 */
@CommandHandler('ExecuteTradeCommand')
@ResourceRequirements({ cpu: 'high', memory: '512MB', priority: 'critical', timeout: 10000 })
export function handleExecuteTradeCommand(
  context: CommandContext,
  command: ExecuteTradeCommand
): Promise<CommandResult<TradeExecutionResult>> {
  
  context.logger.info('Executing trade command', {
    command_id: context.commandId,
    user_id: command.userId,
    trading_pair: command.pair,
    amount_usd: command.amountUsd
  });

  // Synchronous validation (must pass before any events)
  if (command.amountUsd < 10) {
    return Promise.resolve({
      success: false,
      error: {
        code: 'AMOUNT_TOO_SMALL',
        message: 'Trade amount must be at least $10',
        retryable: false
      },
      events: [],
      executionTime: 0,
      commandId: context.commandId
    });
  }

  if (command.amountUsd > command.maxPositionSize) {
    return Promise.resolve({
      success: false,
      error: {
        code: 'POSITION_LIMIT_EXCEEDED',
        message: 'Trade exceeds maximum position size',
        retryable: false
      },
      events: [],
      executionTime: 0,
      commandId: context.commandId
    });
  }

  // Pure business logic execution
  const tradeExecution = {
    tradeId: command.tradeId,
    userId: command.userId,
    pair: command.pair,
    side: command.side,
    amount: command.amount,
    price: command.price,
    amountUsd: command.amountUsd,
    executedAt: new Date().toISOString(),
    status: 'executed'
  };

  // Calculate trade impact
  const portfolioImpact = calculatePortfolioImpact(tradeExecution);
  const riskImpact = calculateRiskImpact(tradeExecution, context.businessContext);

  // Generate domain events for async processing
  const events: DomainEvent[] = [
    {
      eventId: `trade-executed-${command.tradeId}`,
      eventType: 'TradeExecuted',
      streamId: `trade-${command.tradeId}`,
      timestamp: new Date().toISOString(),
      data: tradeExecution,
      metadata: {
        source: 'trade-command-handler',
        version: '1.0.0',
        commandId: context.commandId,
        correlationId: context.correlationId
      }
    }
  ];

  if (portfolioImpact.significantChange) {
    events.push({
      eventId: `portfolio-impact-${command.tradeId}`,
      eventType: 'PortfolioImpacted',
      streamId: `portfolio-${command.userId}`,
      timestamp: new Date().toISOString(),
      data: portfolioImpact,
      metadata: {
        source: 'trade-command-handler',
        version: '1.0.0',
        commandId: context.commandId
      }
    });
  }

  context.addMetadata('trade.execution_status', 'executed');
  context.addMetadata('trade.portfolio_impact', portfolioImpact.changePercent);
  context.addMetadata('trade.risk_impact', riskImpact.newRiskLevel);

  return Promise.resolve({
    success: true,
    result: tradeExecution,
    events: events,
    executionTime: Date.now() - context.startTime.getTime(),
    commandId: context.commandId
  });
}

/**
 * Pure function command handler for portfolio rebalancing
 */
@CommandHandler('RebalancePortfolioCommand')
@ResourceRequirements({ cpu: 'critical', memory: '1GB', priority: 'high', timeout: 30000 })
export function handleRebalancePortfolioCommand(
  context: CommandContext,
  command: RebalancePortfolioCommand
): Promise<CommandResult<RebalanceResult>> {
  
  context.logger.info('Executing portfolio rebalance command', {
    command_id: context.commandId,
    user_id: command.userId,
    target_allocations: Object.keys(command.targetAllocations).length,
    rebalance_reason: command.reason
  });

  // Synchronous validation
  const totalAllocation = Object.values(command.targetAllocations).reduce((sum, alloc) => sum + alloc, 0);
  if (Math.abs(totalAllocation - 1.0) > 0.001) { // Must sum to 100%
    return Promise.resolve({
      success: false,
      error: {
        code: 'INVALID_ALLOCATION',
        message: `Target allocations sum to ${totalAllocation * 100}%, must equal 100%`,
        retryable: false
      },
      events: [],
      executionTime: 0,
      commandId: context.commandId
    });
  }

  // Calculate rebalancing plan
  const rebalancePlan = calculateRebalancePlan(
    command.currentPositions,
    command.targetAllocations,
    command.marketPrices
  );

  const rebalanceResult = {
    rebalanceId: `rebal-${command.userId}-${Date.now()}`,
    userId: command.userId,
    oldPositions: command.currentPositions,
    newPositions: rebalancePlan.targetPositions,
    tradingSteps: rebalancePlan.executionSteps,
    estimatedCost: rebalancePlan.estimatedCost,
    estimatedTime: rebalancePlan.estimatedTime,
    reason: command.reason
  };

  // Generate events for execution
  const events: DomainEvent[] = [
    {
      eventId: `rebalance-planned-${rebalanceResult.rebalanceId}`,
      eventType: 'PortfolioRebalancePlanned',
      streamId: `portfolio-${command.userId}`,
      timestamp: new Date().toISOString(),
      data: rebalanceResult,
      metadata: {
        source: 'rebalance-command-handler',
        version: '1.0.0',
        commandId: context.commandId
      }
    }
  ];

  context.addMetadata('rebalance.steps_count', rebalancePlan.executionSteps.length);
  context.addMetadata('rebalance.estimated_cost', rebalancePlan.estimatedCost);

  return Promise.resolve({
    success: true,
    result: rebalanceResult,
    events: events,
    executionTime: Date.now() - context.startTime.getTime(),
    commandId: context.commandId
  });
}
```

## Query Side (Fast Reads)

### 1. Query Handlers
```typescript
/**
 * Pure function query handler for portfolio lookup
 */
@QueryHandler('GetPortfolioQuery')
@CachePolicy({ ttlSeconds: 300, enabled: true, staleWhileRevalidate: true })
export function handleGetPortfolioQuery(
  context: QueryContext,
  query: GetPortfolioQuery
): Promise<QueryResult<Portfolio>> {
  
  context.logger.debug('Executing portfolio query', {
    query_id: context.queryId,
    user_id: query.userId,
    include_history: query.includeHistory
  });

  // Pure data transformation (no I/O in query function)
  const portfolioQuery = {
    userId: query.userId,
    includePositions: query.includePositions !== false,
    includeMetrics: query.includeMetrics !== false,
    includeHistory: query.includeHistory === true,
    timeRange: query.timeRange || { from: Date.now() - 86400000, to: Date.now() }
  };

  context.addMetadata('query.include_history', portfolioQuery.includeHistory);
  context.addMetadata('query.time_range_hours', (portfolioQuery.timeRange.to - portfolioQuery.timeRange.from) / 3600000);

  // Query execution will be handled by Query Dispatcher (I/O)
  return Promise.resolve({
    success: true,
    data: {
      query: portfolioQuery,
      requiresRedisLookup: true,
      cacheKey: `portfolio:${query.userId}`,
      projectionKeys: [
        `portfolio:${query.userId}:positions`,
        `portfolio:${query.userId}:metadata`
      ]
    } as any, // Query dispatcher will replace with actual data
    executionTime: 1,
    queryId: context.queryId,
    fromCache: false
  });
}

/**
 * Pure function query handler for market analytics
 */
@QueryHandler('GetTopCoinsQuery')
@CachePolicy({ ttlSeconds: 60, enabled: true, staleWhileRevalidate: true })
export function handleGetTopCoinsQuery(
  context: QueryContext,
  query: GetTopCoinsQuery
): Promise<QueryResult<TopCoinsResult[]>> {
  
  context.logger.debug('Executing top coins query', {
    query_id: context.queryId,
    limit: query.limit,
    sort_by: query.sortBy,
    time_window: query.timeWindow
  });

  // Pure query logic
  const analyticsQuery = {
    sortBy: query.sortBy || 'volume',
    limit: Math.min(query.limit || 20, 100), // Max 100 results
    timeWindow: query.timeWindow || '24h',
    filters: {
      minVolume: query.minVolume || 0,
      excludeStableCoins: query.excludeStableCoins !== false
    }
  };

  context.addMetadata('query.sort_by', analyticsQuery.sortBy);
  context.addMetadata('query.limit', analyticsQuery.limit);
  context.addMetadata('query.time_window', analyticsQuery.timeWindow);

  return Promise.resolve({
    success: true,
    data: {
      query: analyticsQuery,
      requiresClickHouseLookup: true,
      cacheKey: `top-coins:${analyticsQuery.sortBy}:${analyticsQuery.limit}:${analyticsQuery.timeWindow}`,
      sqlQuery: `
        SELECT pair, sum(volume) as total_volume, avg(price) as avg_price
        FROM market_prices 
        WHERE timestamp >= now() - INTERVAL ${analyticsQuery.timeWindow}
        GROUP BY pair 
        ORDER BY total_volume DESC 
        LIMIT ${analyticsQuery.limit}
      `
    } as any,
    executionTime: 1,
    queryId: context.queryId,
    fromCache: false
  });
}
```

## Saga Pattern (Long-Running Processes)

### 1. Saga Handlers
```typescript
/**
 * Pure function saga for yield farming workflow
 */
@SagaHandler('YieldFarmingWorkflow')
export function handleYieldFarmingWorkflow(
  context: SagaContext,
  saga: YieldFarmingSaga
): Promise<SagaResult> {
  
  context.logger.info('Starting yield farming workflow saga', {
    saga_id: context.sagaId,
    user_id: saga.userId,
    target_protocols: saga.targetProtocols.length,
    total_amount: saga.totalAmountUsd
  });

  // Define workflow steps
  const steps: SagaStep[] = [
    {
      stepName: 'analyze_opportunities',
      stepNumber: 1,
      timeout: 15000,
      retryPolicy: { maxRetries: 3, backoffMs: 1000, retryableErrors: ['TimeoutError'] },
      execute: async (stepContext: SagaStepContext, data: any): Promise<SagaStepResult> => {
        
        // Pure function: analyze yield opportunities
        const opportunities = analyzeYieldOpportunities(data.protocols, data.marketData);
        
        return {
          success: true,
          data: { opportunities },
          events: [{
            eventId: `yield-analysis-${stepContext.sagaId}-${stepContext.stepNumber}`,
            eventType: 'YieldOpportunitiesAnalyzed',
            streamId: `yield-workflow-${stepContext.sagaId}`,
            timestamp: new Date().toISOString(),
            data: { opportunities, analysisId: stepContext.sagaId },
            metadata: { source: 'yield-farming-saga', version: '1.0.0' }
          }],
          continueFlow: opportunities.length > 0,
          compensationData: { analysisId: stepContext.sagaId }
        };
      },
      compensationAction: {
        actionType: 'cleanup_analysis',
        stepNumber: 1,
        compensationData: {},
        execute: async (context: SagaContext) => {
          context.logger.info('Cleaning up yield analysis', { saga_id: context.sagaId });
        }
      }
    },
    {
      stepName: 'assess_risks',
      stepNumber: 2,
      timeout: 10000,
      retryPolicy: { maxRetries: 2, backoffMs: 2000, retryableErrors: ['ConnectionError'] },
      execute: async (stepContext: SagaStepContext, data: any): Promise<SagaStepResult> => {
        
        const opportunities = stepContext.previousStepResults['analyze_opportunities'].opportunities;
        
        // Pure function: assess risks for each opportunity
        const riskAssessments = opportunities.map((opp: any) => 
          assessYieldRisk(opp, data.userRiskProfile, data.marketConditions)
        );

        const acceptableRisks = riskAssessments.filter(risk => risk.acceptable);

        return {
          success: true,
          data: { riskAssessments: acceptableRisks },
          events: [{
            eventId: `risk-assessed-${stepContext.sagaId}-${stepContext.stepNumber}`,
            eventType: 'YieldRisksAssessed',
            streamId: `yield-workflow-${stepContext.sagaId}`,
            timestamp: new Date().toISOString(),
            data: { assessments: acceptableRisks },
            metadata: { source: 'yield-farming-saga', version: '1.0.0' }
          }],
          continueFlow: acceptableRisks.length > 0,
          compensationData: { assessmentId: stepContext.sagaId }
        };
      }
    },
    {
      stepName: 'execute_positions',
      stepNumber: 3,
      timeout: 30000,
      retryPolicy: { maxRetries: 1, backoffMs: 5000, retryableErrors: ['ExecutionError'] },
      execute: async (stepContext: SagaStepContext, data: any): Promise<SagaStepResult> => {
        
        const riskAssessments = stepContext.previousStepResults['assess_risks'].riskAssessments;
        
        // Pure function: create execution plan
        const executionPlan = createYieldExecutionPlan(
          riskAssessments,
          data.availableCapital,
          data.diversificationRules
        );

        return {
          success: true,
          data: { executionPlan },
          events: [{
            eventId: `execution-planned-${stepContext.sagaId}-${stepContext.stepNumber}`,
            eventType: 'YieldExecutionPlanned',
            streamId: `yield-workflow-${stepContext.sagaId}`,
            timestamp: new Date().toISOString(),
            data: executionPlan,
            metadata: { source: 'yield-farming-saga', version: '1.0.0' }
          }],
          continueFlow: true,
          compensationData: { executionPlanId: executionPlan.planId }
        };
      },
      compensationAction: {
        actionType: 'cancel_execution_plan',
        stepNumber: 3,
        compensationData: {},
        execute: async (context: SagaContext) => {
          // Would cancel any pending executions
          context.logger.warn('Canceling yield execution plan', { saga_id: context.sagaId });
        }
      }
    }
  ];

  // Execute saga with compensation capability
  return Promise.resolve({
    success: true,
    sagaId: context.sagaId,
    finalState: {
      sagaId: context.sagaId,
      status: 'completed',
      currentStep: steps.length,
      totalSteps: steps.length,
      stepResults: {}, // Would be populated during execution
      compensationActions: steps.map(s => s.compensationAction).filter(Boolean) as CompensationAction[],
      createdAt: new Date(),
      updatedAt: new Date(),
      completedAt: new Date()
    },
    events: [], // Events from all steps combined
    compensationExecuted: false
  });
}
```

## Service Integration with CQRS

### 1. Service Classes Handle Commands and Queries
```typescript
/**
 * Trading service integrates CQRS with function manager
 */
class TradingService extends CryptoTradingServiceBase {
  private commandDispatcher: CommandDispatcher;
  private queryDispatcher: QueryDispatcher;
  
  constructor() {
    const infrastructure = await InfrastructureFactory.createDefault();
    super(infrastructure, 'trading-service');
    
    this.commandDispatcher = new CommandDispatcher('trading-service');
    this.queryDispatcher = new QueryDispatcher('trading-service', infrastructure.cacheStore);
    
    this.registerHandlers();
  }

  /**
   * Handle HTTP command (synchronous)
   */
  async executeTrade(request: ExecuteTradeRequest): Promise<TradeExecutionResponse> {
    const businessContext = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('Trade', request.tradeId)
      .withUser(request.userId)
      .withTradingPair(request.pair)
      .withExchange(request.exchange)
      .withAmount(request.amountUsd)
      .build();

    return this.executeBusinessOperation('execute_trade_command', businessContext, async (context) => {
      
      // CQRS Command: Synchronous execution with immediate response
      const commandResult = await this.commandDispatcher.executeCommand<ExecuteTradeCommand, TradeExecutionResult>(
        'ExecuteTradeCommand',
        {
          tradeId: request.tradeId,
          userId: request.userId,
          pair: request.pair,
          side: request.side,
          amount: request.amount,
          price: request.price,
          amountUsd: request.amountUsd,
          maxPositionSize: await this.getMaxPositionSize(request.userId) // I/O
        },
        businessContext,
        10000 // 10 second timeout
      );

      if (commandResult.success) {
        // Command succeeded - events will be processed asynchronously
        
        // Store events in EventStore
        for (const event of commandResult.events) {
          await this.infrastructure.eventStore.append(event.streamId, [event]);
        }

        // Send events to Function Manager for async processing (projections, notifications, etc.)
        for (const event of commandResult.events) {
          await this.sendCommand(
            'function-manager',
            'ProcessEvent',
            event,
            businessContext
          );
        }

        return {
          success: true,
          tradeId: request.tradeId,
          executionTime: commandResult.executionTime,
          result: commandResult.result
        };
      } else {
        return {
          success: false,
          error: commandResult.error,
          tradeId: request.tradeId
        };
      }
    });
  }

  /**
   * Handle HTTP query (fast read)
   */
  async getPortfolio(userId: string, options?: PortfolioQueryOptions): Promise<PortfolioQueryResponse> {
    const businessContext = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('Portfolio', userId)
      .withUser(userId)
      .build();

    return this.executeBusinessOperation('get_portfolio_query', businessContext, async (context) => {
      
      // CQRS Query: Fast read from Redis projections
      const queryResult = await this.queryDispatcher.executeQuery<GetPortfolioQuery, Portfolio>(
        'GetPortfolioQuery',
        {
          userId,
          includePositions: options?.includePositions !== false,
          includeMetrics: options?.includeMetrics !== false,
          includeHistory: options?.includeHistory === true,
          timeRange: options?.timeRange
        },
        businessContext,
        5000 // 5 second timeout
      );

      return {
        success: queryResult.success,
        portfolio: queryResult.data,
        fromCache: queryResult.fromCache,
        cacheAge: queryResult.cacheAge,
        executionTime: queryResult.executionTime
      };
    });
  }

  private registerHandlers(): void {
    // Commands would be registered with command dispatcher
    // Queries would be registered with query dispatcher
    // Actual registration happens through reflection/discovery
  }
}
```

## CQRS with Function Manager Integration

### 1. Command Flow
```
HTTP POST → Service.executeCommand() → CommandDispatcher.execute()
                                           ↓
                                    Pure Function Command Handler
                                           ↓
                                    CommandResult + Events
                                           ↓
                                    EventStore.append() + Function Manager
                                           ↓
                                    Async Event Processing (Redis projections)
```

### 2. Query Flow  
```
HTTP GET → Service.executeQuery() → QueryDispatcher.execute()
                                        ↓
                                 Check Redis Cache
                                        ↓
                                 Pure Function Query Handler
                                        ↓
                                 QueryResult (sub-millisecond)
```

### 3. Saga Flow
```
HTTP POST → Service.startSaga() → SagaOrchestrator.start()
                                      ↓
                              Execute Step 1 (Command)
                                      ↓ (wait for completion)
                              Execute Step 2 (Command)
                                      ↓ (wait for completion)
                              Execute Step 3 (Command)
                                      ↓
                              Saga Complete (or compensate on failure)
```

## Benefits of CQRS Integration

### Performance Optimization
- ✅ **Commands**: Optimized for consistency and validation
- ✅ **Queries**: Optimized for speed and caching
- ✅ **Sagas**: Optimized for long-running coordination
- ✅ **Events**: Optimized for async processing and projections

### Scalability
- ✅ **Command side**: Can scale based on write load
- ✅ **Query side**: Can scale based on read load
- ✅ **Function Manager**: Can scale based on computation load
- ✅ **Independent scaling**: Each side scales independently

### Consistency Models
- ✅ **Commands**: Strong consistency (synchronous validation)
- ✅ **Queries**: Eventual consistency (Redis projections)
- ✅ **Sagas**: Process consistency (coordinated workflows)
- ✅ **Events**: Eventual consistency (async processing)

## CQRS-Function Manager Integration Patterns

### How CQRS and Function Manager Work Together

#### 1. Commands Trigger Function Manager Operations
```typescript
/**
 * CQRS Command Handler triggers Function Manager execution
 */
@CommandHandler('ExecuteYieldStrategyCommand')
export function handleExecuteYieldStrategyCommand(
  context: CommandContext,
  command: ExecuteYieldStrategyCommand
): Promise<CommandResult<YieldStrategyResult>> {
  
  // Synchronous validation
  if (command.amountUsd < 1000) {
    return Promise.resolve({
      success: false,
      error: { code: 'AMOUNT_TOO_LOW', message: 'Minimum $1000 required', retryable: false },
      events: [],
      executionTime: 0,
      commandId: context.commandId
    });
  }

  // Command succeeds immediately, but triggers async Function Manager processing
  const events: DomainEvent[] = [{
    eventId: `yield-strategy-requested-${command.strategyId}`,
    eventType: 'YieldStrategyExecutionRequested',
    streamId: `yield-strategy-${command.strategyId}`,
    timestamp: new Date().toISOString(),
    data: {
      strategyId: command.strategyId,
      userId: command.userId,
      protocols: command.targetProtocols,
      amountUsd: command.amountUsd,
      riskTolerance: command.riskTolerance
    },
    metadata: {
      source: 'yield-strategy-command',
      version: '1.0.0',
      commandId: context.commandId
    }
  }];

  // Command completes immediately, Function Manager processes async
  return Promise.resolve({
    success: true,
    result: {
      strategyId: command.strategyId,
      status: 'queued_for_processing',
      estimatedCompletion: Date.now() + 30000 // 30 seconds
    },
    events: events, // Events will be processed by Function Manager
    executionTime: 5,
    commandId: context.commandId
  });
}
```

#### 2. Sagas Coordinate with Function Manager
```typescript
/**
 * Saga Step that waits for Function Manager completion
 */
{
  stepName: 'execute_yield_analysis',
  stepNumber: 2,
  timeout: 30000,
  retryPolicy: { maxRetries: 3, backoffMs: 5000, retryableErrors: ['TimeoutError'] },
  execute: async (stepContext: SagaStepContext, data: any): Promise<SagaStepResult> => {
    
    // Send analysis request to Function Manager
    const analysisRequest = await stepContext.sendCommand(
      'function-manager',
      'ExecuteFunction',
      {
        functionType: 'analyzeYieldOpportunities',
        eventType: 'YieldAnalysisRequested',
        data: {
          protocols: data.targetProtocols,
          amountUsd: data.totalAmountUsd,
          riskProfile: data.userRiskProfile
        }
      }
    );

    if (!analysisRequest.success) {
      return {
        success: false,
        error: 'Yield analysis failed',
        events: [],
        continueFlow: false
      };
    }

    // Analysis completed by Function Manager
    return {
      success: true,
      data: { 
        analysisResults: analysisRequest.result,
        recommendedAllocations: analysisRequest.result.recommendations
      },
      events: [{
        eventId: `yield-analysis-completed-${stepContext.sagaId}`,
        eventType: 'YieldAnalysisCompleted',
        streamId: `yield-saga-${stepContext.sagaId}`,
        timestamp: new Date().toISOString(),
        data: analysisRequest.result,
        metadata: { source: 'yield-farming-saga', sagaId: stepContext.sagaId }
      }],
      continueFlow: true,
      compensationData: { analysisId: analysisRequest.result.analysisId }
    };
  }
}
```

#### 3. Query Optimization with Function Manager
```typescript
/**
 * Query that leverages Function Manager computations
 */
@QueryHandler('GetOptimizedPortfolioQuery')
export function handleGetOptimizedPortfolioQuery(
  context: QueryContext,
  query: GetOptimizedPortfolioQuery
): Promise<QueryResult<OptimizedPortfolio>> {
  
  // Fast query that requests Function Manager optimization
  const optimizationRequest = {
    userId: query.userId,
    optimizationType: query.optimizationType || 'yield-focused',
    constraints: query.constraints,
    requestTimestamp: Date.now()
  };

  // Query immediately returns cached result if available
  const cachedOptimization = context.getCachedResult(`portfolio-optimization:${query.userId}`);
  
  if (cachedOptimization && (Date.now() - cachedOptimization.timestamp) < 300000) { // 5 min cache
    return Promise.resolve({
      success: true,
      data: cachedOptimization.optimization,
      executionTime: 1,
      queryId: context.queryId,
      fromCache: true,
      cacheAge: Date.now() - cachedOptimization.timestamp
    });
  }

  // Cache miss - trigger Function Manager computation (but return quickly)
  context.triggerAsyncComputation('function-manager', 'OptimizePortfolio', optimizationRequest);
  
  // Return current best-effort result immediately
  return Promise.resolve({
    success: true,
    data: {
      userId: query.userId,
      currentOptimization: context.getCurrentPortfolio(query.userId),
      optimizationStatus: 'computing',
      estimatedCompletion: Date.now() + 15000
    } as any,
    executionTime: 2,
    queryId: context.queryId,
    fromCache: false
  });
}
```

## Cross-Pattern Integration Benefits

### Synchronous + Asynchronous Coordination
- ✅ **Commands**: Immediate validation and response
- ✅ **Function Manager**: Async pure function processing
- ✅ **Queries**: Fast reads from computed projections
- ✅ **Sagas**: Long-running coordination with Function Manager
- ✅ **I/O Scheduling**: Triggered operations and external confirmations

### Performance Optimization
- ✅ **Command latency**: <10ms for validation and queuing
- ✅ **Query latency**: <1ms from Redis projections
- ✅ **Function execution**: Load-balanced across optimal shards
- ✅ **Saga coordination**: Efficient multi-step workflows
- ✅ **Scheduled I/O**: Optimal timing for blockchain and external operations

This CQRS integration provides the **synchronous command handling** you need while maintaining our **pure function architecture** and **async event processing** capabilities.