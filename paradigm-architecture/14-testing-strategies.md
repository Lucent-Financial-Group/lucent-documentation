# Testing Strategies for Paradigm Architecture

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Comprehensive testing approaches for all architectural patterns

## Overview

Testing strategies for our paradigm architecture must cover **pure functions**, **Function Manager**, **CQRS patterns**, **cache repositories**, **I/O scheduling**, and **cross-pattern integration**. This document provides complete testing methodologies for all components.

## Testing Strategy by Pattern

### 1. Pure Function Testing (Easiest)

```typescript
/**
 * Pure functions are easiest to test - no mocks needed
 */
describe('Pure Function Testing', () => {
  
  describe('calculateYieldStrategy', () => {
    it('should calculate conservative strategy for low APY', () => {
      // Arrange
      const context = createMockEnhancedContext();
      const data: YieldOpportunityData = {
        protocol: 'aave',
        apy: 0.05, // 5% APY
        tvl: 1000000,
        riskScore: 7, // High risk
        historicalPerformance: [0.04, 0.05, 0.06, 0.05, 0.04]
      };
      
      // Act
      const result = calculateYieldStrategy(context, data);
      
      // Assert
      expect(result.allocation).toBeLessThan(0.2); // Conservative allocation
      expect(result.protocol).toBe('aave');
      expect(result.confidence).toBeGreaterThan(0);
      
      // Verify context usage (observability without side effects)
      expect(context.addMetadata).toHaveBeenCalledWith('yield.risk_adjustment', expect.any(Number));
      expect(context.logger.debug).toHaveBeenCalledWith(expect.stringContaining('Calculating yield strategy'), expect.any(Object));
    });

    it('should calculate aggressive strategy for high APY low risk', () => {
      const context = createMockEnhancedContext();
      const data: YieldOpportunityData = {
        protocol: 'compound',
        apy: 0.25, // 25% APY
        tvl: 10000000,
        riskScore: 2, // Low risk
        historicalPerformance: [0.24, 0.25, 0.26, 0.25, 0.24]
      };
      
      const result = calculateYieldStrategy(context, data);
      
      expect(result.allocation).toBeGreaterThan(0.3); // Aggressive allocation
      expect(result.confidence).toBeGreaterThan(0.8);
      expect(result.exitStrategy).toBe('gradual');
    });
  });
});

function createMockEnhancedContext(): EnhancedFunctionContext {
  return {
    correlationId: 'test-correlation',
    businessContext: createMockBusinessContext(),
    logger: {
      debug: jest.fn(),
      info: jest.fn(),
      warn: jest.fn(),
      error: jest.fn()
    } as any,
    shard: {
      id: 'test-shard',
      loadFactor: 0.3,
      cpuUsage: 0.2,
      memoryUsage: 0.4,
      availableShards: ['test-shard']
    },
    metrics: {
      executionCount: 10,
      avgExecutionTime: 150,
      errorRate: 0.02
    },
    emit: jest.fn(),
    addMetadata: jest.fn()
  };
}
```

### 2. Function Manager Testing (Mock Heavy)

```typescript
/**
 * Function Manager testing requires comprehensive mocking
 */
describe('Function Manager Integration', () => {
  let functionManager: FunctionManager;
  let mockInfrastructure: MockInfrastructureProvider;

  beforeEach(async () => {
    mockInfrastructure = new MockInfrastructureProvider();
    functionManager = new FunctionManager('test-shard', ['test-shard', 'backup-shard']);
    await functionManager.initialize();
  });

  describe('function execution', () => {
    it('should route events to optimal functions', async () => {
      const yieldEvent = createDomainEvent('YieldOpportunityDetected', {
        protocol: 'aave',
        apy: 0.125,
        tvl: 5000000
      });

      // Mock function registry
      mockFunctionRegistry.set('calculateYieldStrategy', {
        name: 'calculateYieldStrategy',
        eventTypes: ['YieldOpportunityDetected'],
        shardingStrategy: { type: 'protocol', value: 'aave' },
        resourceRequirements: { cpu: 'medium', memory: '256MB', priority: 'high' },
        execute: jest.fn().mockReturnValue({ allocation: 0.3, confidence: 0.85 })
      });

      await functionManager.processIncomingEvent(yieldEvent);

      expect(mockFunctionRegistry.get('calculateYieldStrategy').execute).toHaveBeenCalledWith(
        expect.objectContaining({
          correlationId: expect.any(String),
          businessContext: expect.any(Object)
        }),
        yieldEvent.data
      );
    });

    it('should handle function execution failures with retry', async () => {
      const failingFunction = {
        name: 'failingFunction',
        eventTypes: ['TestEvent'],
        execute: jest.fn()
          .mockRejectedValueOnce(new Error('First attempt failed'))
          .mockRejectedValueOnce(new Error('Second attempt failed'))
          .mockResolvedValueOnce({ success: true })
      };

      mockFunctionRegistry.set('failingFunction', failingFunction);

      const testEvent = createDomainEvent('TestEvent', { test: true });
      
      // Should retry and eventually succeed
      await expect(functionManager.processIncomingEvent(testEvent)).resolves.not.toThrow();
      expect(failingFunction.execute).toHaveBeenCalledTimes(3);
    });
  });
});
```

### 3. CQRS Testing (Service Integration)

```typescript
/**
 * CQRS testing focuses on command/query separation and validation
 */
describe('CQRS Pattern Testing', () => {
  let tradingService: TradingService;
  let mockCommandDispatcher: jest.Mocked<CommandDispatcher>;
  let mockQueryDispatcher: jest.Mocked<QueryDispatcher>;

  beforeEach(() => {
    mockCommandDispatcher = createMockCommandDispatcher();
    mockQueryDispatcher = createMockQueryDispatcher();
    tradingService = new TradingService(mockCommandDispatcher, mockQueryDispatcher);
  });

  describe('command execution', () => {
    it('should execute trade command with validation', async () => {
      const tradeRequest = {
        tradeId: 'trade-123',
        userId: 'user-456',
        pair: 'ETH-USD',
        side: 'buy',
        amount: 1.5,
        amountUsd: 4275
      };

      mockCommandDispatcher.executeCommand.mockResolvedValue({
        success: true,
        result: { tradeId: 'trade-123', status: 'executed' },
        events: [createTradeExecutedEvent(tradeRequest)],
        executionTime: 150,
        commandId: 'cmd-123'
      });

      const result = await tradingService.executeTrade(tradeRequest);

      expect(result.success).toBe(true);
      expect(mockCommandDispatcher.executeCommand).toHaveBeenCalledWith(
        'ExecuteTradeCommand',
        expect.objectContaining(tradeRequest),
        expect.any(Object)
      );
    });

    it('should reject invalid trade commands', async () => {
      const invalidRequest = {
        tradeId: 'invalid-trade',
        userId: 'user-456',
        pair: 'INVALID-PAIR',
        amount: -1, // Invalid amount
        amountUsd: -100
      };

      mockCommandDispatcher.executeCommand.mockResolvedValue({
        success: false,
        error: { code: 'VALIDATION_FAILED', message: 'Invalid trade amount', retryable: false },
        events: [],
        executionTime: 5,
        commandId: 'cmd-invalid'
      });

      const result = await tradingService.executeTrade(invalidRequest);

      expect(result.success).toBe(false);
      expect(result.error?.code).toBe('VALIDATION_FAILED');
    });
  });

  describe('query execution', () => {
    it('should return fast portfolio queries from cache', async () => {
      const portfolioQuery = { userId: 'user-789' };

      mockQueryDispatcher.executeQuery.mockResolvedValue({
        success: true,
        data: {
          userId: 'user-789',
          positions: { 'ETH-USD': 2.5 },
          totalValue: 7126.25
        },
        executionTime: 2,
        queryId: 'query-123',
        fromCache: true,
        cacheAge: 30000
      });

      const result = await tradingService.getPortfolio('user-789');

      expect(result.success).toBe(true);
      expect(result.executionTime).toBeLessThan(10); // Very fast
      expect(result.fromCache).toBe(true);
    });
  });
});
```

### 4. Cache Repository Testing

```typescript
/**
 * Cache repository testing focuses on abstraction correctness
 */
describe('Cache Repository Testing', () => {
  let portfolioCache: PortfolioCacheRepository;
  let mockCacheStore: MockCacheStore;

  beforeEach(() => {
    mockCacheStore = new MockCacheStore();
    portfolioCache = new PortfolioCacheRepository(mockCacheStore, createMockLogger());
  });

  describe('portfolio operations', () => {
    it('should update portfolio with clean abstraction', async () => {
      const portfolio: Portfolio = {
        userId: 'user-123',
        positions: { 'ETH-USD': 2.5, 'BTC-USD': 0.1 },
        totalValue: 75000,
        riskLevel: 'medium',
        lastUpdated: new Date(),
        version: 1
      };

      await portfolioCache.updatePortfolio(portfolio);

      // Verify abstracted operations (not raw Redis commands)
      const cachedPortfolio = await portfolioCache.getPortfolio('user-123');
      expect(cachedPortfolio).toEqual(portfolio);
    });

    it('should handle portfolio rankings correctly', async () => {
      const portfolios = [
        { userId: 'whale-1', totalValue: 1000000 },
        { userId: 'whale-2', totalValue: 750000 },
        { userId: 'retail-1', totalValue: 25000 }
      ];

      for (const p of portfolios) {
        await portfolioCache.updatePortfolio({ ...p, positions: {}, riskLevel: 'medium', lastUpdated: new Date(), version: 1 });
      }

      const topPortfolios = await portfolioCache.getTopPortfolios(3);
      
      expect(topPortfolios[0].userId).toBe('whale-1'); // Highest value first
      expect(topPortfolios[0].rank).toBe(1);
      expect(topPortfolios[1].userId).toBe('whale-2');
      expect(topPortfolios[2].userId).toBe('retail-1');
    });
  });
});
```

### 5. I/O Scheduling Testing

```typescript
/**
 * I/O scheduling testing requires time and event mocking
 */
describe('I/O Scheduling Testing', () => {
  let scheduleManager: IOScheduleManager;
  let mockInfrastructure: MockInfrastructureProvider;

  beforeEach(async () => {
    mockInfrastructure = new MockInfrastructureProvider();
    scheduleManager = new IOScheduleManager();
    await scheduleManager.initialize();
  });

  describe('time-based scheduling', () => {
    it('should execute scheduled operations at correct intervals', async () => {
      const mockScheduleFunction = jest.fn().mockResolvedValue({
        success: true,
        executedOperations: 1
      });

      // Register short interval for testing
      await scheduleManager.registerTimeSchedule(
        'test-schedule',
        '*/5 * * * * *', // Every 5 seconds
        mockScheduleFunction,
        { maxExecutions: 2 }
      );

      // Wait for executions
      await new Promise(resolve => setTimeout(resolve, 12000)); // 12 seconds

      expect(mockScheduleFunction).toHaveBeenCalledTimes(2);
    });
  });

  describe('event-based scheduling', () => {
    it('should trigger on specific events with delays', async () => {
      const mockEventFunction = jest.fn().mockResolvedValue({
        success: true,
        completed: true
      });

      await scheduleManager.registerEventSchedule(
        'event-test',
        'TradeExecuted',
        mockEventFunction,
        { delayMs: 1000, maxRetries: 3 }
      );

      // Publish trigger event
      await mockInfrastructure.messageBus.publish('lucent.events.domain.trading.TradeExecuted', {
        tradeId: 'test-trade',
        userId: 'user-123'
      });

      // Wait for delayed execution
      await new Promise(resolve => setTimeout(resolve, 1500));

      expect(mockEventFunction).toHaveBeenCalledTimes(1);
    });
  });
});
```

## Integration Testing Best Practices

### Test Environment Setup
- ✅ **Mock Infrastructure**: Use `MockInfrastructureProvider` for isolated testing
- ✅ **Test Containers**: Use real Docker services for integration testing
- ✅ **Seed Data**: Consistent test data across all test scenarios
- ✅ **Cleanup**: Proper cleanup after each test to prevent interference

### Performance Testing
- ✅ **Load testing**: Verify patterns handle expected throughput
- ✅ **Latency testing**: Ensure performance targets are met
- ✅ **Stress testing**: Verify graceful degradation under load
- ✅ **Memory testing**: Check for leaks in long-running operations

### Error Scenario Testing
- ✅ **Network failures**: Test resilience patterns activation
- ✅ **Timeout handling**: Verify proper timeout and retry behavior
- ✅ **Data corruption**: Test recovery and consistency mechanisms
- ✅ **Resource exhaustion**: Test bulkhead and circuit breaker patterns

## Mock Utilities

### Function Registry Mocking
```typescript
/**
 * Mock function registry for Function Manager testing
 */
export class MockFunctionRegistry {
  private functions: Map<string, FunctionRegistryEntry> = new Map();

  registerFunction(name: string, config: Partial<FunctionRegistryEntry>): void {
    this.functions.set(name, {
      name,
      eventTypes: config.eventTypes || [],
      shardingStrategy: config.shardingStrategy || { type: 'round_robin' },
      resourceRequirements: config.resourceRequirements || { cpu: 'low', memory: '128MB', priority: 'normal' },
      execute: config.execute || jest.fn().mockResolvedValue({}),
      ...config
    } as FunctionRegistryEntry);
  }

  getFunctionsForEvent(eventType: string): FunctionRegistryEntry[] {
    return Array.from(this.functions.values())
      .filter(func => func.eventTypes.includes(eventType));
  }
}
```

### CQRS Handler Mocking
```typescript
/**
 * Mock CQRS dispatchers for service testing
 */
export function createMockCommandDispatcher(): jest.Mocked<CommandDispatcher> {
  return {
    executeCommand: jest.fn(),
    registerHandler: jest.fn()
  } as any;
}

export function createMockQueryDispatcher(): jest.Mocked<QueryDispatcher> {
  return {
    executeQuery: jest.fn(),
    registerHandler: jest.fn()
  } as any;
}
```

### Cache Repository Testing
```typescript
/**
 * Test cache repositories with behavior verification
 */
describe('Cache Repository Behavior', () => {
  
  it('should maintain data consistency across operations', async () => {
    const repository = new PortfolioCacheRepository(mockCacheStore, mockLogger);
    
    // Test data consistency
    const portfolio = createTestPortfolio();
    await repository.updatePortfolio(portfolio);
    
    const retrieved = await repository.getPortfolio(portfolio.userId);
    expect(retrieved).toEqual(portfolio);
    
    // Test ranking consistency
    const topPortfolios = await repository.getTopPortfolios(10);
    expect(topPortfolios.some(p => p.userId === portfolio.userId)).toBe(true);
  });

  it('should handle concurrent updates correctly', async () => {
    const repository = new PortfolioCacheRepository(mockCacheStore, mockLogger);
    
    // Simulate concurrent portfolio updates
    const updates = Array.from({ length: 10 }, (_, i) => 
      repository.updatePortfolio(createTestPortfolio(`user-${i}`))
    );
    
    await Promise.all(updates);
    
    // Verify all updates completed
    const allPortfolios = await Promise.all(
      Array.from({ length: 10 }, (_, i) => repository.getPortfolio(`user-${i}`))
    );
    
    expect(allPortfolios.every(p => p !== null)).toBe(true);
  });
});
```

## Testing Guidelines Summary

### Testing Hierarchy (Easiest to Hardest)
1. **Pure Functions**: No mocks, fast execution, high confidence
2. **Cache Repositories**: Simple mocking, behavior verification
3. **CQRS Handlers**: Command/query mocking, validation testing
4. **Function Manager**: Complex mocking, integration scenarios
5. **Cross-Pattern Integration**: Full system testing, comprehensive scenarios

### Testing Principles
- ✅ **Test behavior, not implementation**: Focus on what, not how
- ✅ **Mock at service boundaries**: Mock infrastructure, not internal methods
- ✅ **Use real data structures**: Realistic test data for better confidence
- ✅ **Test error scenarios**: Verify resilience patterns activation
- ✅ **Performance validation**: Ensure patterns meet performance requirements

This testing strategy ensures **comprehensive coverage** of all architectural patterns while maintaining **fast execution** and **high confidence** in the crypto trading platform.