# I/O Shell Pattern (OOP at the Edges)

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Object-oriented infrastructure layer for external integrations

## Overview

The I/O Shell Pattern implements the **imperative shell** of our functional architecture using object-oriented base classes. This layer handles all side effects, external integrations, and infrastructure concerns while providing a clean interface to the functional core.

## Base Class Hierarchy

### LucentServiceBase (Foundation)
```typescript
import { LucentServiceBase, BusinessContextBuilder } from '@lucent/infrastructure';

/**
 * Base class provides infrastructure integration for all services
 */
export abstract class LucentServiceBase {
  protected logger: InfrastructureLogger;
  protected infrastructure: IInfrastructureProvider;

  constructor(infrastructure: IInfrastructureProvider, serviceName: string) {
    this.infrastructure = infrastructure;
    this.logger = LoggerFactory.getLogger(serviceName, 'service');
    this.setupInfrastructureIntegration();
  }

  // PROVIDES: Automatic business context tracing
  protected async executeBusinessOperation<T>(
    operationName: string,
    businessContext: CryptoBusinessContext,
    operation: (context: RequestContext) => Promise<T>
  ): Promise<T> {
    // Handles tracing, logging, error handling, metrics automatically
  }

  // PROVIDES: Type-safe event publishing 
  protected async publishDomainEvent<T>(
    eventType: string,
    streamId: string,
    eventData: T,
    businessContext: CryptoBusinessContext
  ): Promise<void> {
    // Handles EventStore persistence + NATS publishing automatically
  }

  // PROVIDES: Cross-service communication
  protected async sendCommand<TReq, TRes>(
    targetService: string,
    commandType: string,
    commandData: TReq,
    businessContext: CryptoBusinessContext
  ): Promise<TRes> {
    // Handles NATS request/response with full business context
  }

  // PROVIDES: Event handling infrastructure
  protected async handleDomainEvent<T>(
    event: DomainEvent<T>,
    handler: (event: DomainEvent<T>, context: EventProcessingContext) => Promise<void>
  ): Promise<void> {
    // Handles tracing, error recovery, business context extraction
  }
}
```

### CryptoTradingServiceBase (Domain-Specific)
```typescript
/**
 * Crypto trading specialized base class with domain-specific patterns
 */
export abstract class CryptoTradingServiceBase extends LucentServiceBase {
  
  // PROVIDES: Crypto-specific business operations
  protected async executeTrade<T>(
    tradeType: string,
    tradingPair: string,
    exchange: string,
    amountUsd: number,
    userId: string,
    operation: (context: RequestContext) => Promise<T>
  ): Promise<T> {
    // Automatically creates crypto business context
    const businessContext = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('Trade', `trade-${Date.now()}`)
      .withUser(userId)
      .withTradingPair(tradingPair)
      .withExchange(exchange)
      .withAmount(amountUsd)
      .withRiskLevel(this.determineRiskLevel(amountUsd))
      .build();

    return this.executeBusinessOperation(`execute_trade_${tradeType}`, businessContext, operation);
  }

  // PROVIDES: Yield farming operations
  protected async analyzeYieldOpportunity<T>(
    protocol: string,
    poolId: string,
    userId: string,
    operation: (context: RequestContext) => Promise<T>
  ): Promise<T> {
    // Crypto-specific context creation
    const businessContext = BusinessContextBuilder
      .forYieldFarming()
      .withAggregate('YieldAnalysis', `yield-${protocol}-${poolId}`)
      .withUser(userId)
      .withProtocol(protocol)
      .build();

    return this.executeBusinessOperation(`analyze_yield_${protocol}`, businessContext, operation);
  }

  // PROVIDES: Portfolio management operations  
  protected async executePortfolioOperation<T>(
    operationType: string,
    userId: string,
    portfolioData: any,
    operation: (context: RequestContext) => Promise<T>
  ): Promise<T> {
    const businessContext = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('Portfolio', `portfolio-${userId}`)
      .withUser(userId)
      .withAmount(portfolioData.totalValueUsd)
      .withRiskLevel(portfolioData.riskProfile)
      .build();

    return this.executeBusinessOperation(`portfolio_${operationType}`, businessContext, operation);
  }

  private determineRiskLevel(amountUsd: number): 'low' | 'medium' | 'high' {
    if (amountUsd > 1000000) return 'high';
    if (amountUsd > 100000) return 'medium'; 
    return 'low';
  }
}
```

### UserServiceBase (Domain-Specific)
```typescript
/**
 * User management specialized base class
 */
export abstract class UserServiceBase extends LucentServiceBase {
  
  // PROVIDES: User-specific operations
  protected async executeUserOperation<T>(
    operationType: string,
    userId: string,
    operation: (context: RequestContext) => Promise<T>
  ): Promise<T> {
    const businessContext = BusinessContextBuilder
      .forUserManagement()
      .withAggregate('User', userId)
      .withUser(userId)
      .build();

    return this.executeBusinessOperation(`user_${operationType}`, businessContext, operation);
  }

  // PROVIDES: Authentication operations
  protected async executeAuthOperation<T>(
    operationType: string,
    userId: string,
    securityContext: SecurityContext,
    operation: (context: RequestContext) => Promise<T>
  ): Promise<T> {
    const businessContext = BusinessContextBuilder
      .forUserManagement()
      .withAggregate('UserAuth', `auth-${userId}`)
      .withUser(userId)
      .withPriority('high') // Security operations are high priority
      .withMetadata('security_level', securityContext.level)
      .build();

    return this.executeBusinessOperation(`auth_${operationType}`, businessContext, operation);
  }
}
```

## I/O Operations Implementation

### External API Integration
```typescript
/**
 * Market data service handling external exchange APIs
 */
export class MarketDataService extends CryptoTradingServiceBase {
  
  /**
   * I/O Shell: Fetch data from external exchange API
   */
  async fetchBinancePriceData(tradingPair: string): Promise<void> {
    // Use crypto trading base class for automatic context
    return this.executeTrade(
      'market_data_fetch',
      tradingPair,
      'binance',
      0, // No monetary amount for data fetch
      'system',
      async (context) => {
        
        // 1. External API call (I/O)
        const rawData = await this.binanceAPI.getPriceData(tradingPair);
        
        // 2. Call pure function for data processing (functional core)
        const processedData = this.callPureFunction('processBinanceData', {
          rawData,
          tradingPair,
          timestamp: Date.now()
        }, context);
        
        // 3. Store in ClickHouse (I/O)
        await this.infrastructure.analyticsStore.insert('market_prices', [processedData]);
        
        // 4. Publish domain event (I/O)
        await this.publishDomainEvent(
          'MarketDataProcessed',
          `market-${tradingPair}`,
          processedData,
          context.businessContext,
          context
        );
        
        // 5. Send to Kafka for real-time processing (I/O)
        await this.infrastructure.streamProcessor.produce(
          `market.prices.binance.${tradingPair.toLowerCase()}`,
          [processedData]
        );
        
        return processedData;
      }
    );
  }

  /**
   * Call pure function from I/O shell
   */
  private callPureFunction<TIn, TOut>(
    functionName: string,
    input: TIn,
    context: RequestContext
  ): TOut {
    // Get function from registry with type safety
    const func = TYPED_FUNCTION_REGISTRY[functionName];
    
    // Create enhanced context for pure function
    const enhancedContext = this.createEnhancedContext(context);
    
    // Execute pure function with no side effects
    return func.execute(enhancedContext, input);
  }
}
```

### Database Operations
```typescript
/**
 * Portfolio service handling database persistence
 */
export class PortfolioService extends CryptoTradingServiceBase {
  
  /**
   * I/O Shell: Handle portfolio update event
   */
  async handleTradeExecuted(event: TradeExecutedEvent): Promise<void> {
    return this.handleDomainEvent(event, async (event, context) => {
      
      // 1. Read current portfolio from EventStore (I/O)
      const portfolioEvents = await this.infrastructure.eventStore.read<PortfolioEvent>(
        `portfolio-${event.data.userId}`
      );
      
      // 2. Call pure function to calculate new portfolio state (functional core)
      const updatedPortfolio = this.callPureFunction('calculatePortfolioUpdate', {
        currentEvents: portfolioEvents,
        newTrade: event.data,
        timestamp: Date.now()
      }, context);
      
      // 3. Validate portfolio constraints (pure function)
      const validation = this.callPureFunction('validatePortfolioConstraints', {
        portfolio: updatedPortfolio,
        userRiskProfile: await this.getUserRiskProfile(event.data.userId), // I/O
        regulatoryLimits: await this.getRegulatoryLimits(event.data.userId)  // I/O
      }, context);
      
      if (!validation.isValid) {
        // 4. Handle constraint violations (I/O)
        await this.publishDomainEvent(
          'PortfolioConstraintViolated',
          `portfolio-${event.data.userId}`,
          validation,
          context.businessContext,
          context
        );
        return;
      }
      
      // 5. Persist portfolio update (I/O)
      await this.publishDomainEvent(
        'PortfolioUpdated',
        `portfolio-${event.data.userId}`,
        updatedPortfolio,
        context.businessContext,
        context
      );
      
      // 6. Update real-time cache in Redis (I/O)
      await this.updatePortfolioCache(event.data.userId, updatedPortfolio);
      
      // 7. Send notifications if needed (I/O)
      if (updatedPortfolio.riskLevel !== validation.previousRiskLevel) {
        await this.sendCommand(
          'notification-service',
          'SendRiskLevelChangeNotification',
          {
            userId: event.data.userId,
            oldRisk: validation.previousRiskLevel,
            newRisk: updatedPortfolio.riskLevel
          },
          context.businessContext
        );
      }
    });
  }
  
  /**
   * I/O Shell: External service integration
   */
  private async getUserRiskProfile(userId: string): Promise<RiskProfile> {
    // External API call with resilience patterns from base class
    return this.executeUserOperation('get_risk_profile', userId, async (context) => {
      
      // Circuit breaker and retry logic from infrastructure
      const response = await this.httpClient.get(`/users/${userId}/risk-profile`);
      
      // Pure function for data transformation
      return this.callPureFunction('parseRiskProfile', response.data, context);
    });
  }
}
```

### Cross-Service Communication
```typescript
/**
 * Yield optimization service coordinating multiple services
 */
export class YieldOptimizationService extends CryptoTradingServiceBase {
  
  /**
   * I/O Shell: Orchestrate multi-service yield optimization
   */
  async optimizeUserYield(request: OptimizeYieldRequest): Promise<OptimizationResult> {
    return this.executePortfolioOperation(
      'yield_optimization',
      request.userId,
      request.portfolioData,
      async (context) => {
        
        // 1. Get current positions (I/O - EventStore)
        const currentPositions = await this.infrastructure.eventStore.read<PositionEvent>(
          `portfolio-${request.userId}`
        );
        
        // 2. Get market data (I/O - ClickHouse)  
        const marketData = await this.infrastructure.analyticsStore.query<MarketData>(
          'SELECT * FROM market_prices WHERE timestamp >= now() - INTERVAL 1 HOUR'
        );
        
        // 3. Get risk assessment (I/O - Cross-service call)
        const riskAssessment = await this.sendCommand<RiskRequest, RiskAssessment>(
          'risk-assessment-service',
          'AssessPortfolioRisk',
          {
            positions: currentPositions,
            marketConditions: marketData,
            userProfile: request.userProfile
          },
          context.businessContext
        );
        
        // 4. Call pure function for optimization (functional core)
        const optimizationInput = {
          currentPositions: this.callPureFunction('parsePositionEvents', currentPositions, context),
          marketData: this.callPureFunction('normalizeMarketData', marketData, context),
          riskConstraints: this.callPureFunction('parseRiskAssessment', riskAssessment, context),
          userPreferences: request.userPreferences
        };
        
        const optimization = this.callPureFunction('optimizeYieldAllocation', optimizationInput, context);
        
        // 5. Validate optimization result (pure function)
        const validation = this.callPureFunction('validateOptimization', {
          optimization,
          currentPositions: optimizationInput.currentPositions,
          riskLimits: optimizationInput.riskConstraints
        }, context);
        
        if (!validation.isValid) {
          throw new ValidationError('Optimization violates risk constraints', 'optimization', validation.violations);
        }
        
        // 6. Generate execution plan (pure function)
        const executionPlan = this.callPureFunction('generateExecutionPlan', {
          currentPositions: optimizationInput.currentPositions,
          targetPositions: optimization.targetPositions,
          marketConditions: optimizationInput.marketData
        }, context);
        
        // 7. Publish optimization result (I/O)
        await this.publishDomainEvent(
          'YieldOptimizationCompleted',
          `optimization-${request.userId}-${Date.now()}`,
          {
            optimization,
            executionPlan,
            expectedImprovement: optimization.expectedYieldImprovement
          },
          context.businessContext,
          context
        );
        
        // 8. Send execution commands to trading service (I/O)
        for (const step of executionPlan.steps) {
          await this.sendCommand(
            'trading-execution-service',
            'ExecuteTradeStep',
            step,
            context.businessContext
          );
        }
        
        return {
          optimization,
          executionPlan,
          status: 'initiated'
        };
      }
    );
  }
}
```

## Infrastructure Integration Patterns

### Event Store Integration
```typescript
export class EventSourcingService extends LucentServiceBase {
  
  /**
   * I/O Shell: Manage event streams with business context
   */
  async appendUserEvents(userId: string, events: UserEvent[]): Promise<void> {
    const businessContext = BusinessContextBuilder
      .forUserManagement()
      .withAggregate('User', userId)
      .withUser(userId)
      .build();
    
    return this.executeBusinessOperation('append_user_events', businessContext, async (context) => {
      
      // 1. Validate events using pure functions
      const validatedEvents = events.map(event => 
        this.callPureFunction('validateUserEvent', event, context)
      );
      
      // 2. Transform events using pure functions
      const domainEvents = validatedEvents.map(event =>
        this.callPureFunction('transformToUserDomainEvent', event, context)
      );
      
      // 3. Append to EventStore (I/O)
      await this.infrastructure.eventStore.append(
        `user-${userId}`,
        domainEvents,
        undefined,
        {
          correlationId: context.correlationId,
          traceContext: getCurrentSpan()?.spanContext()
        }
      );
      
      // 4. Publish to NATS for real-time processing (I/O)
      for (const event of domainEvents) {
        await this.infrastructure.messageBus.publish(
          `lucent.events.domain.users.${event.eventType}`,
          event,
          {
            correlationId: context.correlationId,
            traceContext: getCurrentSpan()?.spanContext()
          }
        );
      }
      
      // 5. Update projections (I/O)
      await this.updateUserProjections(userId, domainEvents);
    });
  }
}
```

### Real-Time Data Processing
```typescript
export class MarketDataProcessor extends CryptoTradingServiceBase {
  
  /**
   * I/O Shell: Process incoming market data streams
   */
  async processKafkaMarketStream(): Promise<void> {
    // Subscribe to Kafka stream (I/O)
    await this.infrastructure.streamProcessor.consume<RawMarketData>(
      'market.prices.all',
      async (rawData, traceContext) => {
        
        const businessContext = BusinessContextBuilder
          .forCryptoTrading()
          .withAggregate('MarketData', rawData.tradingPair)
          .withTradingPair(rawData.tradingPair)
          .withExchange(rawData.exchange)
          .build();
        
        return this.executeBusinessOperation('process_market_data', businessContext, async (context) => {
          
          // 1. Validate raw data (pure function)
          const validation = this.callPureFunction('validateMarketData', rawData, context);
          if (!validation.isValid) {
            this.logger.warn('Invalid market data received', validation.errors);
            return;
          }
          
          // 2. Normalize data format (pure function)  
          const normalizedData = this.callPureFunction('normalizeMarketData', rawData, context);
          
          // 3. Detect anomalies (pure function)
          const anomalyCheck = this.callPureFunction('detectMarketAnomalies', {
            current: normalizedData,
            historical: await this.getHistoricalData(rawData.tradingPair), // I/O
            threshold: 0.15 // 15% price deviation threshold
          }, context);
          
          if (anomalyCheck.hasAnomaly) {
            // 4. Handle anomaly (I/O)
            await this.publishDomainEvent(
              'MarketAnomalyDetected',
              `anomaly-${rawData.tradingPair}-${Date.now()}`,
              anomalyCheck,
              context.businessContext,
              context
            );
          }
          
          // 5. Store processed data (I/O)
          await this.infrastructure.analyticsStore.insert('market_prices', [normalizedData]);
          
          // 6. Check for arbitrage opportunities (pure function)
          const arbitrageCheck = this.callPureFunction('checkArbitrageOpportunity', {
            newPrice: normalizedData,
            exchangePrices: await this.getExchangePrices(rawData.tradingPair), // I/O
            transactionCosts: await this.getTransactionCosts(rawData.exchange)   // I/O
          }, context);
          
          if (arbitrageCheck.opportunityExists) {
            // 7. Emit arbitrage opportunity (I/O)
            await this.publishDomainEvent(
              'ArbitrageOpportunityDetected',
              `arbitrage-${rawData.tradingPair}-${Date.now()}`,
              arbitrageCheck.opportunity,
              context.businessContext,
              context
            );
          }
        });
      }
    );
  }
}
```

### HTTP Controller Integration
```typescript
/**
 * NestJS controller using I/O shell patterns
 */
@Controller('yield')
export class YieldController {
  constructor(
    private yieldService: YieldOptimizationService,
    private functionManager: YieldFunctionManager
  ) {}
  
  @Post('optimize/:userId')
  @UseGuards(AuthGuard)
  async optimizeYield(
    @Param('userId') userId: string,
    @Body() request: OptimizeYieldRequest,
    @RequestContext() httpContext: HttpRequestContext
  ): Promise<OptimizationResult> {
    
    // HTTP handling (I/O shell) delegates to service layer
    return this.yieldService.optimizeUserYield({
      ...request,
      userId,
      correlationId: httpContext.correlationId,
      source: 'http-api'
    });
    
    // Service layer uses base class patterns
    // Base class provides business context, tracing, logging
    // Pure functions handle calculations
    // I/O shell handles persistence and external calls
  }
  
  @Post('analyze/:protocol')
  async analyzeProtocolYield(
    @Param('protocol') protocol: string,
    @Body() analysisRequest: YieldAnalysisRequest
  ): Promise<AnalysisResult> {
    
    // Direct function manager call for compute-intensive operations
    const event: DomainEvent<YieldOpportunityData> = {
      eventId: `analysis-${Date.now()}`,
      eventType: 'YieldOpportunityDetected',
      streamId: `yield-analysis-${protocol}`,
      timestamp: new Date().toISOString(),
      data: {
        protocol,
        apy: analysisRequest.currentApy,
        tvl: analysisRequest.totalValueLocked,
        riskScore: analysisRequest.riskScore,
        historicalPerformance: analysisRequest.historicalData
      },
      metadata: {
        source: 'http-controller',
        version: '1.0.0',
        userId: analysisRequest.userId
      }
    };
    
    // Route to function manager for load-balanced execution
    await this.functionManager.processIncomingEvent(event);
    
    // Return immediate response while processing continues asynchronously
    return {
      status: 'processing',
      analysisId: event.eventId,
      estimatedCompletion: Date.now() + 30000 // 30 seconds
    };
  }
}
```

## Resource Management

### Connection Lifecycle
```typescript
export abstract class ResourceManagedService extends LucentServiceBase {
  
  /**
   * I/O Shell: Manage external service connections
   */
  protected async withExternalService<T>(
    serviceName: string,
    operation: (client: any) => Promise<T>
  ): Promise<T> {
    
    // Use base class circuit breaker and retry logic
    return this.circuitBreaker.execute(async () => {
      
      // Get connection from pool (infrastructure)
      const connection = await this.connectionPool.acquire(serviceName);
      
      try {
        // Execute operation with connection
        return await operation(connection.client);
        
      } catch (error: any) {
        // Base class error handling with business context
        this.logger.error('External service operation failed', {
          service: serviceName,
          error: error.message
        }, error);
        throw error;
        
      } finally {
        // Always release connection
        await this.connectionPool.release(connection);
      }
    });
  }
  
  /**
   * I/O Shell: Batch operations for performance
   */
  protected async executeBatchOperation<T, R>(
    operationName: string,
    items: T[],
    batchSize: number,
    processor: (batch: T[], context: RequestContext) => Promise<R[]>
  ): Promise<R[]> {
    
    const businessContext = BusinessContextBuilder
      .forCryptoTrading()
      .withAggregate('BatchOperation', `batch-${Date.now()}`)
      .withMetadata('batch_size', batchSize)
      .withMetadata('total_items', items.length)
      .build();
    
    return this.executeBusinessOperation(`batch_${operationName}`, businessContext, async (context) => {
      
      const batches: T[][] = [];
      for (let i = 0; i < items.length; i += batchSize) {
        batches.push(items.slice(i, i + batchSize));
      }
      
      const results: R[] = [];
      
      for (let i = 0; i < batches.length; i++) {
        const batch = batches[i];
        
        this.logger.debug('Processing batch', {
          batch_index: i,
          batch_size: batch.length,
          total_batches: batches.length
        });
        
        // Process batch with context
        const batchResults = await processor(batch, context);
        results.push(...batchResults);
        
        // Rate limiting between batches
        if (i < batches.length - 1) {
          await this.sleep(100); // 100ms between batches
        }
      }
      
      this.logger.info('Batch operation completed', {
        operation: operationName,
        total_items: items.length,
        total_results: results.length,
        batches_processed: batches.length
      });
      
      return results;
    });
  }
}
```

## Error Handling and Recovery

### Resilience Patterns
```typescript
export class ResilientFunctionManager extends CryptoTradingServiceBase {
  
  /**
   * I/O Shell: Execute function with comprehensive error handling
   */
  async executeWithResilience<TIn, TOut>(
    func: TypedFunction<TIn, TOut>,
    event: DomainEvent<TIn>,
    maxRetries = 3
  ): Promise<TOut> {
    
    let attempt = 0;
    let lastError: Error;
    
    while (attempt < maxRetries) {
      try {
        // Use base class circuit breaker
        return await this.circuitBreaker.execute(async () => {
          
          // Apply rate limiting from base class
          await this.rateLimiter.acquire();
          
          // Execute function with chaos engineering
          return this.chaosMonkey.maybeInjectChaos(async () => {
            return func.execute(this.createEnhancedContext(event), event.data);
          }, 'function-execution');
        });
        
      } catch (error: any) {
        lastError = error;
        attempt++;
        
        this.logger.warn('Function execution failed, retrying', {
          function_name: func.name,
          attempt,
          max_retries: maxRetries,
          error: error.message,
          event_type: event.eventType
        });
        
        // Exponential backoff
        const backoffMs = Math.pow(2, attempt) * 1000;
        await this.sleep(backoffMs);
      }
    }
    
    // All retries exhausted
    this.logger.error('Function execution failed after all retries', {
      function_name: func.name,
      attempts: attempt,
      final_error: lastError.message,
      event_type: event.eventType
    }, lastError);
    
    // Publish failure event for monitoring
    await this.publishDomainEvent(
      'FunctionExecutionFailed',
      `function-failure-${func.name}-${Date.now()}`,
      {
        functionName: func.name,
        eventType: event.eventType,
        attempts: attempt,
        error: lastError.message
      },
      this.createFailureBusinessContext(func, event)
    );
    
    throw lastError;
  }
}
```

### State Management
```typescript
export class StatefulFunctionManager extends LucentServiceBase {
  private functionState: Map<string, FunctionState> = new Map();
  
  /**
   * I/O Shell: Manage function state across executions
   */
  async executeStatefulFunction<TIn, TOut>(
    func: StatefulFunction<TIn, TOut>,
    event: DomainEvent<TIn>
  ): Promise<TOut> {
    
    return this.executeBusinessOperation('execute_stateful_function', businessContext, async (context) => {
      
      // 1. Load previous state (I/O)
      const previousState = await this.loadFunctionState(func.name, event.streamId);
      
      // 2. Execute pure function with state
      const result = func.execute(
        this.createEnhancedContext(context),
        event.data,
        previousState // Previous state as input
      );
      
      // 3. Extract new state from result
      const newState = this.callPureFunction('extractFunctionState', result, context);
      
      // 4. Persist new state (I/O)
      await this.saveFunctionState(func.name, event.streamId, newState);
      
      // 5. Publish state change event if significant
      const stateChange = this.callPureFunction('calculateStateChange', {
        previous: previousState,
        current: newState
      }, context);
      
      if (stateChange.isSignificant) {
        await this.publishDomainEvent(
          'FunctionStateChanged',
          `state-${func.name}-${event.streamId}`,
          stateChange,
          context.businessContext,
          context
        );
      }
      
      return result.output; // Return business result, not state
    });
  }
  
  private async loadFunctionState(functionName: string, streamId: string): Promise<any> {
    // Load from EventStore or Redis depending on state type
    const stateEvents = await this.infrastructure.eventStore.read(
      `function-state-${functionName}-${streamId}`
    );
    
    // Reduce events to current state using pure function
    return this.callPureFunction('reduceFunctionState', stateEvents, null);
  }
}
```

## Integration with Pure Functions

### Function Context Bridge
```typescript
export class FunctionContextBridge {
  
  /**
   * Create enhanced context that bridges I/O shell to pure functions
   */
  createEnhancedContext(
    serviceBase: LucentServiceBase,
    requestContext: RequestContext
  ): EnhancedFunctionContext {
    
    return {
      correlationId: requestContext.correlationId,
      businessContext: requestContext.businessContext,
      logger: serviceBase.logger.child('pure-function'),
      shard: {
        id: serviceBase.shardId,
        loadFactor: serviceBase.loadMonitor.getCurrentLoad(),
        cpuUsage: serviceBase.resourceMonitor.getCpuUsage(),
        memoryUsage: serviceBase.resourceMonitor.getMemoryUsage(),
        availableShards: serviceBase.availableShards
      },
      metrics: serviceBase.metricsCollector.getFunctionMetrics(),
      
      // Pure functions can emit events through I/O shell
      emit: async (eventType: string, data: any) => {
        await serviceBase.publishDomainEvent(
          eventType,
          `pure-function-emission-${Date.now()}`,
          data,
          requestContext.businessContext,
          requestContext
        );
      },
      
      // Pure functions can add tracing metadata
      addMetadata: (key: string, value: any) => {
        getCurrentSpan()?.setAttributes({ [`function.${key}`]: JSON.stringify(value) });
      },
      
      // Pure functions can access cached data (read-only)
      getCache: async (key: string) => {
        return serviceBase.cacheManager.get(key);
      },
      
      // Pure functions can request external data (triggers I/O)
      requestData: async (dataType: string, params: any) => {
        return serviceBase.dataProvider.fetchData(dataType, params);
      }
    };
  }
}
```

### Pure Function Execution Wrapper
```typescript
export class PureFunctionExecutor extends LucentServiceBase {
  
  /**
   * Execute pure function with full I/O shell integration
   */
  async executePureFunction<TIn, TOut>(
    functionName: string,
    input: TIn,
    businessContext: CryptoBusinessContext
  ): Promise<TOut> {
    
    return this.executeBusinessOperation(`pure_function_${functionName}`, businessContext, async (context) => {
      
      // 1. Get function from registry
      const func = this.functionRegistry.getFunction<TIn, TOut>(functionName);
      if (!func) {
        throw new ProviderOperationError(`Function ${functionName} not found`, 'function-registry', 'get');
      }
      
      // 2. Check resource requirements
      const hasResources = await this.checkResourceAvailability(func.resourceRequirements);
      if (!hasResources) {
        throw new ResourceExhaustionError(
          `Insufficient resources for function ${functionName}`,
          'cpu_memory',
          { required: func.resourceRequirements, available: await this.getAvailableResources() }
        );
      }
      
      // 3. Create enhanced context
      const enhancedContext = this.createEnhancedContext(context);
      
      // 4. Execute with monitoring
      const startTime = Date.now();
      let result: TOut;
      
      try {
        result = func.execute(enhancedContext, input);
        
        const executionTime = Date.now() - startTime;
        
        // 5. Record success metrics
        await this.recordFunctionSuccess(functionName, executionTime, result);
        
        return result;
        
      } catch (error: any) {
        const executionTime = Date.now() - startTime;
        
        // 6. Record error metrics  
        await this.recordFunctionError(functionName, executionTime, error);
        
        throw error;
      }
    });
  }
}
```

## Benefits of I/O Shell Pattern

### Clean Separation
- **I/O operations**: Handled by base classes with enterprise patterns
- **Pure calculations**: Isolated in pure functions with no side effects  
- **Infrastructure concerns**: Abstracted away from business logic
- **Cross-cutting concerns**: Tracing, logging, monitoring provided automatically

### Enterprise Integration
- **Resilience patterns**: Circuit breakers, rate limiting, bulkheads from base classes
- **Observability**: Distributed tracing and structured logging built-in
- **Service communication**: Type-safe cross-service calls via base classes
- **Event sourcing**: Domain event management with full business context

### Operational Excellence
- **Load balancing**: Functions distributed optimally across shards
- **Resource management**: CPU/memory requirements enforced automatically  
- **Error recovery**: Retry logic and fallback strategies built-in
- **Performance monitoring**: Function execution metrics collected automatically

The I/O shell pattern provides **enterprise-grade infrastructure** while keeping your **functional core completely pure** and enabling **dynamic load-balanced function distribution** across shards.