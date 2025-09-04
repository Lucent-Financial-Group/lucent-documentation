# I/O Event Scheduling Framework

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Scheduled and event-driven I/O execution management

## Overview

The I/O Event Scheduling Framework provides **sophisticated scheduling capabilities** for I/O operations that need to execute based on time, events, conditions, or external confirmations. This is critical for crypto trading systems that must handle blockchain confirmations, periodic rebalancing, market data synchronization, and cleanup operations.

## I/O Scheduling Architecture

### Core Scheduling Patterns

#### 1. Time-Based Scheduling
```typescript
/**
 * Schedule I/O operations based on time intervals
 */
@IOSchedule('portfolio-rebalance')
@CronSchedule('0 0 * * *') // Daily at midnight
@RequiresCondition('market-hours-closed')
export async function schedulePortfolioRebalance(
  context: IOScheduleContext
): Promise<ScheduleResult> {
  
  context.logger.info('Executing scheduled portfolio rebalance', {
    schedule_id: context.scheduleId,
    execution_time: context.scheduledTime,
    condition_met: context.conditionsMet
  });

  // Get all active portfolios that need rebalancing
  const portfolios = await context.infrastructure.cacheStore.smembers('portfolios-requiring-rebalance');
  
  const rebalanceResults = [];
  
  for (const userId of portfolios) {
    try {
      // Execute synchronous CQRS command for immediate validation
      const rebalanceResult = await context.executeCommand(
        'ExecuteRebalanceCommand',
        {
          userId,
          reason: 'scheduled_daily_rebalance',
          triggeredAt: Date.now()
        }
      );
      
      rebalanceResults.push({ userId, success: rebalanceResult.success });
      
    } catch (error: any) {
      rebalanceResults.push({ userId, success: false, error: error.message });
    }
  }

  // Update schedule execution metrics
  await context.infrastructure.cacheStore.hset(
    'schedule-execution-metrics',
    'last-portfolio-rebalance',
    JSON.stringify({
      executedAt: Date.now(),
      portfoliosProcessed: portfolios.length,
      successCount: rebalanceResults.filter(r => r.success).length,
      failureCount: rebalanceResults.filter(r => !r.success).length
    })
  );

  return {
    success: true,
    executedOperations: rebalanceResults.length,
    results: rebalanceResults,
    nextScheduledExecution: context.getNextExecution(),
    metadata: {
      scheduleType: 'cron',
      conditionsChecked: context.conditionsMet
    }
  };
}
```

#### 2. Event-Based Scheduling  
```typescript
/**
 * Schedule I/O operations after specific events occur
 */
@IOSchedule('blockchain-confirmation-check')
@EventTrigger('TradeSubmittedToBlockchain')
@DelayAfterEvent('5 minutes') // Wait 5 minutes after trade submission
@MaxRetries(10)
export async function scheduleBlockchainConfirmationCheck(
  context: IOScheduleContext,
  triggerEvent: TradeSubmittedEvent
): Promise<ScheduleResult> {
  
  context.logger.info('Checking blockchain confirmation', {
    trade_id: triggerEvent.tradeId,
    blockchain: triggerEvent.blockchain,
    tx_hash: triggerEvent.txHash,
    delay_elapsed: Date.now() - triggerEvent.submittedAt
  });

  // Check blockchain for transaction confirmation
  const confirmationStatus = await context.callExternalAPI(
    'blockchain-api',
    'getTransactionStatus',
    { txHash: triggerEvent.txHash }
  );

  if (confirmationStatus.confirmed) {
    // Trade confirmed - update systems
    await context.sendCommand(
      'trading-service',
      'ConfirmTradeExecution',
      {
        tradeId: triggerEvent.tradeId,
        confirmationHash: confirmationStatus.confirmationHash,
        blockNumber: confirmationStatus.blockNumber,
        confirmations: confirmationStatus.confirmations
      }
    );

    // Remove from pending confirmations
    await context.infrastructure.cacheStore.srem('pending-blockchain-confirmations', triggerEvent.tradeId);

    return {
      success: true,
      completed: true,
      result: confirmationStatus,
      reschedule: false
    };
    
  } else if (confirmationStatus.failed) {
    // Trade failed - handle failure
    await context.sendCommand(
      'trading-service',
      'HandleTradeFailure',
      {
        tradeId: triggerEvent.tradeId,
        failureReason: confirmationStatus.failureReason
      }
    );

    return {
      success: true,
      completed: true,
      result: { status: 'failed', reason: confirmationStatus.failureReason },
      reschedule: false
    };
    
  } else {
    // Still pending - reschedule for later
    return {
      success: true,
      completed: false,
      reschedule: true,
      rescheduleDelay: 60000, // Check again in 1 minute
      metadata: {
        confirmations: confirmationStatus.confirmations,
        requiredConfirmations: confirmationStatus.requiredConfirmations
      }
    };
  }
}
```

#### 3. Conditional Scheduling
```typescript
/**
 * Schedule I/O operations based on complex conditions
 */
@IOSchedule('market-volatility-response')
@ConditionCheck('checkVolatilityThreshold')
@ExecuteWhen('volatility > 0.05 && trading_volume > 1000000')
@CooldownPeriod('15 minutes')
export async function scheduleVolatilityResponse(
  context: IOScheduleContext,
  conditionData: VolatilityCondition
): Promise<ScheduleResult> {
  
  context.logger.info('Executing volatility response procedure', {
    volatility_level: conditionData.volatility,
    trading_volume: conditionData.tradingVolume,
    affected_pairs: conditionData.affectedPairs.length,
    cooldown_remaining: context.cooldownRemaining
  });

  const responseActions = [];

  // 1. Increase monitoring frequency
  await context.sendCommand(
    'market-data-service',
    'IncreaseMonitoringFrequency',
    {
      pairs: conditionData.affectedPairs,
      newFrequency: '1 second',
      duration: '30 minutes'
    }
  );
  responseActions.push('increased_monitoring');

  // 2. Tighten risk limits
  const riskUpdate = await context.sendCommand(
    'risk-service',
    'UpdateRiskLimits',
    {
      reason: 'high_volatility',
      newLimits: {
        maxPositionSize: conditionData.currentLimits.maxPositionSize * 0.5,
        maxDailyVolume: conditionData.currentLimits.maxDailyVolume * 0.7
      },
      duration: '1 hour'
    }
  );
  responseActions.push('tightened_risk_limits');

  // 3. Execute protective rebalancing if needed
  if (conditionData.volatility > 0.1) { // Extreme volatility
    const protectiveRebalance = await context.sendCommand(
      'portfolio-service',
      'ExecuteProtectiveRebalancing',
      {
        reason: 'extreme_volatility',
        targetRiskReduction: 0.3,
        affectedPairs: conditionData.affectedPairs
      }
    );
    responseActions.push('protective_rebalancing');
  }

  // Store volatility response metrics
  await context.infrastructure.cacheStore.zadd(
    'volatility-responses',
    conditionData.volatility,
    `${Date.now()}:${conditionData.affectedPairs.join(',')}`
  );

  return {
    success: true,
    executedActions: responseActions,
    cooldownActivated: true,
    cooldownDuration: 900000, // 15 minutes
    metadata: {
      volatilityLevel: conditionData.volatility,
      actionsExecuted: responseActions.length
    }
  };
}
```

#### 4. External Event Waiting
```typescript
/**
 * Schedule I/O operations that wait for external confirmations
 */
@IOSchedule('yield-position-confirmation')
@WaitForEvent('YieldPositionConfirmed')
@Timeout('10 minutes')
@FallbackAction('handlePositionTimeout')
export async function scheduleYieldPositionConfirmation(
  context: IOScheduleContext,
  waitingFor: YieldPositionWaitData
): Promise<ScheduleResult> {
  
  context.logger.info('Waiting for yield position confirmation', {
    position_id: waitingFor.positionId,
    protocol: waitingFor.protocol,
    user_id: waitingFor.userId,
    wait_started: waitingFor.waitStartTime,
    elapsed_ms: Date.now() - waitingFor.waitStartTime
  });

  // Check if confirmation event has arrived
  const confirmationEvent = await context.checkForEvent(
    'YieldPositionConfirmed',
    { positionId: waitingFor.positionId }
  );

  if (confirmationEvent) {
    // Position confirmed - execute synchronous finalization command
    const finalizationResult = await context.executeCommand(
      'FinalizeYieldPositionCommand',
      {
        positionId: waitingFor.positionId,
        confirmationData: confirmationEvent.data,
        finalizedAt: Date.now()
      }
    );
    
    if (!finalizationResult.success) {
      throw new Error(`Position finalization failed: ${finalizationResult.error.message}`);
    }

    // Update position status in Redis
    await context.infrastructure.cacheStore.hset(
      `yield-position:${waitingFor.positionId}`,
      'status',
      'confirmed'
    );

    return {
      success: true,
      completed: true,
      result: confirmationEvent.data,
      waitDuration: Date.now() - waitingFor.waitStartTime
    };

  } else if (Date.now() - waitingFor.waitStartTime > context.timeout) {
    // Timeout - execute fallback
    await context.executeFallback('handlePositionTimeout', {
      positionId: waitingFor.positionId,
      timeoutReason: 'confirmation_not_received'
    });

    return {
      success: true,
      completed: true,
      timedOut: true,
      fallbackExecuted: true
    };

  } else {
    // Still waiting - reschedule
    return {
      success: true,
      completed: false,
      reschedule: true,
      rescheduleDelay: 30000, // Check again in 30 seconds
      stillWaiting: true
    };
  }
}
```

## Complete IOScheduleManager Implementation

### Professional Schedule Orchestrator
```typescript
/**
 * IOScheduleManager handles all scheduled I/O operations
 */
export class IOScheduleManager extends LucentServiceBase {
  private activeSchedules: Map<string, ScheduleState> = new Map();
  private cronJobs: Map<string, NodeJS.Timeout> = new Map();
  private eventWaiters: Map<string, EventWaiter> = new Map();
  private conditionCheckers: Map<string, NodeJS.Timeout> = new Map();

  constructor() {
    const infrastructure = await InfrastructureFactory.createDefault();
    super(infrastructure, 'io-schedule-manager');
    
    this.initializeScheduleRegistry();
    this.setupEventListeners();
    this.startScheduleMonitoring();
    this.registerShutdownCleanup();
  }

  /**
   * Initialize schedule registry from decorated functions
   */
  private initializeScheduleRegistry(): void {
    // Discover decorated I/O schedule functions
    const schedules = this.discoverScheduleFunctions();
    
    for (const schedule of schedules) {
      this.registerScheduleFromMetadata(schedule);
    }
    
    this.logger.info('Schedule registry initialized', {
      total_schedules: schedules.length,
      time_based: schedules.filter(s => s.type === 'time-based').length,
      event_based: schedules.filter(s => s.type === 'event-based').length,
      conditional: schedules.filter(s => s.type === 'conditional').length
    });
  }

  /**
   * Setup event listeners for event-based schedules
   */
  private setupEventListeners(): void {
    // Subscribe to all domain events for event-triggered schedules
    this.infrastructure.messageBus.subscribe('lucent.events.domain.*', async (event, traceContext) => {
      await this.processEventTriggers(event, traceContext);
    });
  }

  /**
   * Start monitoring for scheduled operations
   */
  private startScheduleMonitoring(): void {
    // Monitor schedule health every minute
    setInterval(async () => {
      await this.monitorScheduleHealth();
    }, 60000);

    // Cleanup expired schedules every hour
    setInterval(async () => {
      await this.cleanupExpiredSchedules();
    }, 3600000);
  }

  /**
   * Process event triggers for event-based schedules
   */
  private async processEventTriggers(event: any, traceContext: any): Promise<void> {
    const eventType = event.eventType || event.type;
    
    // Find schedules triggered by this event type
    const triggeredSchedules = Array.from(this.activeSchedules.values())
      .filter(schedule => 
        schedule.type === 'event-based' && 
        schedule.triggerEventType === eventType
      );

    for (const schedule of triggeredSchedules) {
      if (schedule.delayMs) {
        // Schedule delayed execution
        setTimeout(async () => {
          await this.executeSchedule(schedule.scheduleId, { triggerEvent: event });
        }, schedule.delayMs);
      } else {
        // Execute immediately
        await this.executeSchedule(schedule.scheduleId, { triggerEvent: event });
      }
    }
  }

  /**
   * Execute scheduled operation
   */
  private async executeSchedule(scheduleId: string, triggerData?: any): Promise<void> {
    const schedule = this.activeSchedules.get(scheduleId);
    if (!schedule) return;

    const businessContext = this.createScheduleBusinessContext(scheduleId);
    
    return this.executeBusinessOperation(`execute_schedule_${scheduleId}`, businessContext, async (context) => {
      
      this.logger.info('Executing I/O schedule', {
        schedule_id: scheduleId,
        schedule_type: schedule.type,
        function_name: schedule.functionName,
        trigger_data: !!triggerData
      });

      // Create schedule context
      const scheduleContext: IOScheduleContext = {
        scheduleId,
        scheduledTime: new Date(),
        correlationId: context.correlationId,
        businessContext: context.businessContext,
        infrastructure: this.infrastructure,
        logger: this.logger.child(`schedule.${scheduleId}`),
        triggerInfo: triggerData || { type: 'scheduled' },
        conditionsMet: true,
        
        // Async cross-service messaging via NATS
        sendCommand: async (service: string, command: string, data: any) => {
          return this.sendCommand(service, command, data, context.businessContext);
        },
        
        // Synchronous CQRS command execution
        executeCommand: async (commandType: string, commandData: any) => {
          return this.commandDispatcher.executeCommand(commandType, commandData, context.businessContext);
        },
        
        // Fast CQRS query execution  
        executeQuery: async (queryType: string, queryData: any) => {
          return this.queryDispatcher.executeQuery(queryType, queryData, context.businessContext);
        },
        
        callExternalAPI: async (service: string, method: string, params: any) => {
          return this.callExternalService(service, method, params);
        },
        
        checkForEvent: async (eventType: string, criteria: any) => {
          return this.checkEventExists(eventType, criteria);
        },
        
        executeFallback: async (fallbackName: string, data: any) => {
          return this.executeFallbackAction(fallbackName, data);
        },
        
        getNextExecution: () => {
          return this.calculateNextExecution(scheduleId);
        }
      };

      try {
        // Execute schedule function
        const result = await schedule.ioFunction(scheduleContext, triggerData);
        
        // Handle schedule result
        await this.handleScheduleResult(scheduleId, result, context);
        
      } catch (error: any) {
        this.logger.error('Schedule execution failed', {
          schedule_id: scheduleId,
          error: error.message
        }, error);
        
        // Handle schedule failure
        await this.handleScheduleFailure(scheduleId, error, context);
      }
    });
  }

  /**
   * Monitor schedule health and performance
   */
  private async monitorScheduleHealth(): Promise<void> {
    for (const [scheduleId, schedule] of this.activeSchedules) {
      // Check if schedule is stuck or overdue
      const lastExecution = await this.getLastExecutionTime(scheduleId);
      const now = Date.now();
      
      if (schedule.type === 'time-based' && now - lastExecution > schedule.expectedInterval * 2) {
        this.logger.warn('Schedule appears stuck', {
          schedule_id: scheduleId,
          last_execution: new Date(lastExecution),
          expected_interval: schedule.expectedInterval,
          overdue_by: now - lastExecution - schedule.expectedInterval
        });
        
        // Attempt to restart stuck schedule
        await this.restartSchedule(scheduleId);
      }
    }
  }

  /**
   * Cleanup and shutdown handling
   */
  private registerShutdownCleanup(): void {
    process.on('SIGINT', async () => {
      this.logger.info('Gracefully shutting down I/O Schedule Manager');
      
      // Clear all intervals
      for (const interval of this.cronJobs.values()) {
        clearTimeout(interval);
      }
      
      for (const interval of this.conditionCheckers.values()) {
        clearInterval(interval);
      }
      
      this.logger.info('All schedules stopped');
    });
  }
}

  /**
   * Register time-based schedule
   */
  async registerTimeSchedule(
    scheduleId: string,
    cronPattern: string,
    ioFunction: IOScheduleFunction,
    options: ScheduleOptions = {}
  ): Promise<void> {
    
    this.logger.info('Registering time-based I/O schedule', {
      schedule_id: scheduleId,
      cron_pattern: cronPattern,
      function_name: ioFunction.name,
      options
    });

    const cronJob = this.createCronJob(cronPattern, async () => {
      await this.executeScheduledIO(scheduleId, ioFunction, { type: 'cron', pattern: cronPattern });
    });

    this.cronJobs.set(scheduleId, cronJob);
    
    // Store schedule metadata
    await this.infrastructure.cacheStore.hset(
      'active-schedules',
      scheduleId,
      JSON.stringify({
        type: 'time-based',
        cronPattern,
        functionName: ioFunction.name,
        registeredAt: Date.now(),
        options
      })
    );
  }

  /**
   * Register event-based schedule
   */
  async registerEventSchedule(
    scheduleId: string,
    triggerEventType: string,
    ioFunction: IOScheduleFunction,
    options: EventScheduleOptions = {}
  ): Promise<void> {
    
    this.logger.info('Registering event-based I/O schedule', {
      schedule_id: scheduleId,
      trigger_event: triggerEventType,
      function_name: ioFunction.name,
      delay_ms: options.delayMs,
      max_retries: options.maxRetries
    });

    // Subscribe to trigger events
    await this.infrastructure.messageBus.subscribe(
      `lucent.events.domain.*.${triggerEventType}`,
      async (event, traceContext) => {
        
        if (options.delayMs) {
          // Schedule delayed execution
          setTimeout(async () => {
            await this.executeScheduledIO(scheduleId, ioFunction, { 
              type: 'event-triggered', 
              triggerEvent: event,
              delayElapsed: options.delayMs 
            });
          }, options.delayMs);
        } else {
          // Execute immediately
          await this.executeScheduledIO(scheduleId, ioFunction, { 
            type: 'event-triggered', 
            triggerEvent: event 
          });
        }
      }
    );
  }

  /**
   * Register conditional schedule
   */
  async registerConditionalSchedule(
    scheduleId: string,
    conditionCheck: ConditionCheckFunction,
    ioFunction: IOScheduleFunction,
    options: ConditionalScheduleOptions = {}
  ): Promise<void> {
    
    const checkInterval = options.checkIntervalMs || 60000; // 1 minute default
    
    this.logger.info('Registering conditional I/O schedule', {
      schedule_id: scheduleId,
      function_name: ioFunction.name,
      check_interval_ms: checkInterval,
      cooldown_ms: options.cooldownMs
    });

    // Start condition checking loop
    const conditionChecker = setInterval(async () => {
      
      try {
        const conditionMet = await conditionCheck(this.createConditionContext());
        
        if (conditionMet.shouldExecute) {
          // Check cooldown
          const lastExecution = await this.getLastExecutionTime(scheduleId);
          const cooldownRemaining = options.cooldownMs ? 
            Math.max(0, (lastExecution + options.cooldownMs) - Date.now()) : 0;

          if (cooldownRemaining === 0) {
            await this.executeScheduledIO(scheduleId, ioFunction, { 
              type: 'condition-triggered', 
              conditionData: conditionMet.data 
            });
          } else {
            this.logger.debug('Condition met but cooldown active', {
              schedule_id: scheduleId,
              cooldown_remaining_ms: cooldownRemaining
            });
          }
        }
        
      } catch (error: any) {
        this.logger.error('Condition check failed', {
          schedule_id: scheduleId,
          error: error.message
        });
      }
    }, checkInterval);

    this.activeSchedules.set(scheduleId, { 
      type: 'conditional',
      interval: conditionChecker,
      options 
    });
  }

  /**
   * Register external event waiter
   */
  async registerEventWaiter(
    waiterId: string,
    eventToWaitFor: string,
    ioFunction: IOScheduleFunction,
    options: EventWaiterOptions = {}
  ): Promise<void> {
    
    const timeout = options.timeoutMs || 600000; // 10 minutes default
    
    this.logger.info('Registering external event waiter', {
      waiter_id: waiterId,
      waiting_for: eventToWaitFor,
      function_name: ioFunction.name,
      timeout_ms: timeout
    });

    const waiter: EventWaiter = {
      waiterId,
      eventToWaitFor,
      ioFunction,
      startTime: Date.now(),
      timeout,
      options
    };

    this.eventWaiters.set(waiterId, waiter);

    // Set timeout for fallback execution
    setTimeout(async () => {
      const stillWaiting = this.eventWaiters.has(waiterId);
      
      if (stillWaiting) {
        this.logger.warn('Event waiter timed out', {
          waiter_id: waiterId,
          waiting_for: eventToWaitFor,
          waited_ms: timeout
        });

        // Execute fallback or cleanup
        if (options.fallbackFunction) {
          await options.fallbackFunction(this.createTimeoutContext(waiter));
        }

        this.eventWaiters.delete(waiterId);
      }
    }, timeout);
  }

  /**
   * Execute scheduled I/O operation with full context
   */
  private async executeScheduledIO(
    scheduleId: string,
    ioFunction: IOScheduleFunction,
    triggerInfo: ScheduleTriggerInfo
  ): Promise<void> {
    
    const businessContext = this.createScheduleBusinessContext(scheduleId, triggerInfo);
    
    return this.executeBusinessOperation(`schedule_${scheduleId}`, businessContext, async (context) => {
      
      this.logger.info('Executing scheduled I/O operation', {
        schedule_id: scheduleId,
        function_name: ioFunction.name,
        trigger_type: triggerInfo.type,
        execution_time: Date.now()
      });

      // Create I/O schedule context
      const scheduleContext: IOScheduleContext = {
        scheduleId,
        scheduledTime: new Date(),
        correlationId: context.correlationId,
        businessContext: context.businessContext,
        infrastructure: this.infrastructure,
        logger: this.logger.child(`schedule.${scheduleId}`),
        triggerInfo,
        conditionsMet: triggerInfo.type === 'condition-triggered',
        
        // I/O operation helpers
        sendCommand: async (service: string, command: string, data: any) => {
          return this.sendCommand(service, command, data, context.businessContext);
        },
        
        callExternalAPI: async (service: string, method: string, params: any) => {
          return this.callExternalService(service, method, params, context);
        },
        
        checkForEvent: async (eventType: string, criteria: any) => {
          return this.checkEventOccurred(eventType, criteria);
        },
        
        executeFallback: async (fallbackName: string, data: any) => {
          return this.executeFallbackAction(fallbackName, data, context);
        },
        
        getNextExecution: () => {
          return this.calculateNextExecution(scheduleId);
        }
      };

      // Execute I/O function
      const result = await ioFunction(scheduleContext, triggerInfo.data);
      
      // Handle scheduling result
      await this.handleScheduleResult(scheduleId, result, context);
      
      // Update execution metrics
      await this.recordScheduleExecution(scheduleId, result, context);
    });
  }

  /**
   * Handle schedule execution results
   */
  private async handleScheduleResult(
    scheduleId: string,
    result: ScheduleResult,
    context: RequestContext
  ): Promise<void> {
    
    if (result.reschedule) {
      // Reschedule for later execution
      setTimeout(async () => {
        const scheduleData = await this.getScheduleData(scheduleId);
        if (scheduleData) {
          await this.executeScheduledIO(scheduleId, scheduleData.ioFunction, {
            type: 'rescheduled',
            originalTrigger: result.metadata
          });
        }
      }, result.rescheduleDelay || 60000);
    }

    if (result.cooldownActivated) {
      // Activate cooldown period
      await this.infrastructure.cacheStore.set(
        `schedule-cooldown:${scheduleId}`,
        Date.now().toString(),
        Math.floor((result.cooldownDuration || 0) / 1000)
      );
    }

    // Publish schedule completion event
    await this.publishDomainEvent(
      'IOScheduleCompleted',
      `schedule-${scheduleId}`,
      {
        scheduleId,
        result,
        executedAt: Date.now()
      },
      context.businessContext,
      context
    );
  }
}
```

## Communication Pattern Clarification

### Async vs Sync Operation Methods

I/O Schedule functions have access to **three distinct communication patterns**:

#### 1. Async Cross-Service Messaging (`sendCommand`)
```typescript
// Async messaging via NATS - fire and forget or request/reply
await context.sendCommand('notification-service', 'SendAlert', alertData);
await context.sendCommand('analytics-service', 'UpdateMetrics', metricsData);

// Use for: Notifications, analytics updates, non-critical operations
// Characteristics: Eventually consistent, resilient to service downtime
```

#### 2. Synchronous CQRS Commands (`executeCommand`) 
```typescript
// Synchronous command execution with immediate validation and response
const tradeResult = await context.executeCommand('ExecuteTradeCommand', {
  userId: 'user-123',
  amount: 50000,
  pair: 'ETH-USD'
});

if (!tradeResult.success) {
  throw new Error(`Trade failed: ${tradeResult.error.message}`);
}

// Use for: Critical operations requiring immediate feedback
// Characteristics: <10ms response, strong consistency, immediate validation
```

#### 3. Fast CQRS Queries (`executeQuery`)
```typescript
// Sub-millisecond queries from Redis projections
const portfolio = await context.executeQuery('GetPortfolioQuery', {
  userId: 'user-123',
  includePositions: true
});

const riskMetrics = await context.executeQuery('GetRiskMetricsQuery', {
  userId: 'user-123'
});

// Use for: Data retrieval, dashboard updates, real-time displays  
// Characteristics: <1ms response, eventually consistent, cached
```

### When to Use Each Pattern

| Operation Type | Method | Use Case | Response Time | Consistency |
|---|---|---|---|---|
| **Notifications** | `sendCommand` | Alert users, update dashboards | Eventually | Eventual |
| **Critical Trades** | `executeCommand` | Execute trades, validate limits | <10ms | Strong |
| **Data Retrieval** | `executeQuery` | Get portfolios, check balances | <1ms | Eventual |
| **External APIs** | `callExternalAPI` | Blockchain, exchange APIs | Variable | External |

## Schedule Context and Interfaces

### I/O Schedule Context
```typescript
/**
 * Context provided to I/O schedule functions
 */
export interface IOScheduleContext {
  readonly scheduleId: string;
  readonly scheduledTime: Date;
  readonly correlationId: string;
  readonly businessContext: BusinessContext;
  readonly infrastructure: IInfrastructureProvider;
  readonly logger: InfrastructureLogger;
  readonly triggerInfo: ScheduleTriggerInfo;
  readonly conditionsMet: boolean;
  
  // Async cross-service messaging (via NATS)
  sendCommand(service: string, command: string, data: any): Promise<any>;
  
  // Synchronous CQRS operations
  executeCommand(commandType: string, commandData: any): Promise<CommandResult>;
  executeQuery(queryType: string, queryData: any): Promise<QueryResult>;
  
  // External integrations
  callExternalAPI(service: string, method: string, params: any): Promise<any>;
  checkForEvent(eventType: string, criteria: any): Promise<any>;
  executeFallback(fallbackName: string, data: any): Promise<void>;
  getNextExecution(): Date;
}

/**
 * I/O Schedule function interface
 */
export interface IOScheduleFunction {
  (context: IOScheduleContext, data?: any): Promise<ScheduleResult>;
}

/**
 * Schedule execution result
 */
export interface ScheduleResult {
  readonly success: boolean;
  readonly completed?: boolean;
  readonly reschedule?: boolean;
  readonly rescheduleDelay?: number;
  readonly cooldownActivated?: boolean;
  readonly cooldownDuration?: number;
  readonly timedOut?: boolean;
  readonly fallbackExecuted?: boolean;
  readonly executedOperations?: number;
  readonly results?: any[];
  readonly metadata?: Record<string, any>;
}
```

## Common I/O Scheduling Patterns

### 1. Market Data Synchronization
```typescript
@IOSchedule('sync-market-data')
@CronSchedule('*/30 * * * * *') // Every 30 seconds
@RequiresCondition('trading-hours-active')
export async function syncMarketData(context: IOScheduleContext): Promise<ScheduleResult> {
  
  // Sync data from multiple exchanges
  const exchanges = ['binance', 'coinbase', 'kraken', 'huobi'];
  const syncResults = [];

  for (const exchange of exchanges) {
    const syncResult = await context.sendCommand(
      'market-data-service',
      'SyncExchangeData',
      { exchange, timestamp: Date.now() }
    );
    syncResults.push({ exchange, success: syncResult.success });
  }

  return {
    success: true,
    executedOperations: exchanges.length,
    results: syncResults,
    nextScheduledExecution: context.getNextExecution()
  };
}
```

### 2. Cleanup Operations
```typescript
@IOSchedule('cleanup-expired-data')
@EventTrigger('DataRetentionPolicyUpdated')
@DelayAfterEvent('1 hour')
export async function cleanupExpiredData(
  context: IOScheduleContext,
  triggerEvent: DataRetentionEvent
): Promise<ScheduleResult> {
  
  const cleanupOperations = [];

  // Clean up old Redis cache entries
  const expiredKeys = await context.infrastructure.cacheStore.scan('*:expired:*');
  if (expiredKeys.length > 0) {
    await context.infrastructure.cacheStore.del(...expiredKeys);
    cleanupOperations.push(`redis_cleanup:${expiredKeys.length}_keys`);
  }

  // Clean up old EventStore projections
  await context.sendCommand(
    'function-manager',
    'ExecuteFunction',
    {
      functionType: 'cleanupOldProjections',
      eventType: 'ProjectionCleanupRequested',
      data: { retentionPolicy: triggerEvent.newPolicy }
    }
  );
  cleanupOperations.push('eventstore_projection_cleanup');

  // Clean up old analytics data in ClickHouse
  await context.callExternalAPI(
    'clickhouse',
    'executeCleanupQuery',
    { retentionDays: triggerEvent.newPolicy.retentionDays }
  );
  cleanupOperations.push('clickhouse_cleanup');

  return {
    success: true,
    executedOperations: cleanupOperations.length,
    results: cleanupOperations
  };
}
```

### 3. Health Check Orchestration
```typescript
@IOSchedule('system-health-check')
@CronSchedule('0 */5 * * * *') // Every 5 minutes
@Priority('high')
export async function systemHealthCheck(context: IOScheduleContext): Promise<ScheduleResult> {
  
  const healthChecks = [];

  // Check all infrastructure providers
  const providers = ['nats', 'kafka', 'clickhouse', 'kurrentdb', 'redis'];
  
  for (const provider of providers) {
    const healthResult = await context.sendCommand(
      provider + '-service',
      'ExecuteHealthCheck',
      { timestamp: Date.now() }
    );
    
    healthChecks.push({
      provider,
      healthy: healthResult.success,
      latency: healthResult.latency,
      lastChecked: Date.now()
    });
  }

  // Check function manager shards
  const shardHealth = await context.callExternalAPI(
    'function-manager',
    'getShardHealth',
    { includeMetrics: true }
  );
  
  healthChecks.push({
    component: 'function-shards',
    healthy: shardHealth.overallHealth > 0.8,
    details: shardHealth
  });

  // Store health metrics
  await context.infrastructure.cacheStore.hset(
    'system-health',
    'last-check',
    JSON.stringify({
      timestamp: Date.now(),
      overallHealth: healthChecks.every(h => h.healthy),
      components: healthChecks
    })
  );

  return {
    success: true,
    executedOperations: healthChecks.length,
    results: healthChecks,
    metadata: {
      overallHealth: healthChecks.every(h => h.healthy),
      unhealthyComponents: healthChecks.filter(h => !h.healthy).length
    }
  };
}
```

## Benefits of I/O Scheduling Framework

### Operational Excellence
- ✅ **Automated operations**: Critical tasks execute reliably
- ✅ **Event-driven triggers**: React to business events automatically  
- ✅ **Condition monitoring**: Execute when specific conditions are met
- ✅ **External confirmations**: Wait for blockchain/API confirmations
- ✅ **Fallback handling**: Graceful handling of timeouts and failures

### Crypto Trading Specific
- ✅ **Blockchain confirmations**: Wait for transaction confirmations
- ✅ **Market condition responses**: React to volatility automatically
- ✅ **Periodic rebalancing**: Execute portfolio rebalancing on schedule
- ✅ **Data synchronization**: Keep market data fresh across exchanges
- ✅ **Risk monitoring**: Continuous risk threshold checking

### Integration with Pure Architecture
- ✅ **I/O operations only**: No business logic in schedulers
- ✅ **Function Manager integration**: Can trigger pure function execution
- ✅ **Service coordination**: Orchestrates multiple services
- ✅ **Event-driven**: Publishes domain events for results
- ✅ **Professional naming**: Clean, single-responsibility classes

This I/O Event Scheduling Framework completes the infrastructure by providing **sophisticated scheduling capabilities** for all the I/O operations that crypto trading systems require.