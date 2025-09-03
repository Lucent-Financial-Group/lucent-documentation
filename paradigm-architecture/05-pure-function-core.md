# Pure Function Core Pattern

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Functional programming core with enhanced context integration

## Overview

The Pure Function Core contains all business logic as **side-effect-free functions** that are deterministic, easily testable, and can be distributed across shards. These functions receive enhanced context that provides infrastructure capabilities without violating functional purity.

## Pure Function Design Principles

### 1. No Side Effects
```typescript
// ✅ PURE: Only transforms input to output
export function calculateYieldStrategy(
  context: EnhancedFunctionContext,
  data: YieldOpportunityData
): YieldStrategy {
  
  // Pure calculation logic
  const riskAdjustedApy = data.apy * (1 - (data.riskScore / 10));
  const optimalAllocation = Math.min(riskAdjustedApy / 20, 0.5);
  
  // Context provides observability without side effects
  context.logger.debug('Yield calculation completed', {
    original_apy: data.apy,
    risk_adjusted_apy: riskAdjustedApy,
    allocation: optimalAllocation
  });
  
  context.addMetadata('calculation.risk_adjustment', data.riskScore / 10);
  
  return {
    protocol: data.protocol,
    allocation: optimalAllocation,
    confidence: calculateConfidenceScore(data),
    exitStrategy: optimalAllocation > 0.3 ? 'gradual' : 'immediate'
  };
}

// ❌ IMPURE: Has side effects
export function calculateYieldStrategyBad(data: YieldOpportunityData): YieldStrategy {
  // Side effect - violates purity
  console.log('Calculating yield'); // ❌ Logging side effect
  
  // Side effect - violates purity  
  database.insert('calculations', data); // ❌ Database side effect
  
  return strategy;
}
```

### 2. Deterministic Results
```typescript
// ✅ DETERMINISTIC: Same input always produces same output
export function assessArbitrageRisk(
  context: EnhancedFunctionContext,
  data: ArbitrageOpportunityData
): ArbitrageRiskAssessment {
  
  // All inputs are parameters or derived from parameters
  const profitMargin = (data.sellPrice - data.buyPrice) / data.buyPrice;
  const liquidityRisk = calculateLiquidityRisk(data.liquidity);
  const executionRisk = calculateExecutionRisk(data.timeWindow);
  const exchangeRisk = calculateExchangeRisk(data.exchanges);
  
  // Deterministic calculation
  const overallRisk = (liquidityRisk * 0.4) + (executionRisk * 0.3) + (exchangeRisk * 0.3);
  
  context.addMetadata('risk.liquidity', liquidityRisk);
  context.addMetadata('risk.execution', executionRisk); 
  context.addMetadata('risk.exchange', exchangeRisk);
  
  return {
    overallRisk,
    profitMargin,
    recommendation: overallRisk < 0.3 && profitMargin > 0.02 ? 'execute' : 'skip',
    maxPositionSize: calculateMaxPosition(overallRisk, data.availableLiquidity),
    timeToExecution: calculateOptimalTiming(data)
  };
}

// ❌ NON-DETERMINISTIC: Uses external state or random values
export function assessRiskBad(data: ArbitrageOpportunityData): RiskAssessment {
  // Non-deterministic - violates purity
  const currentTime = Date.now(); // ❌ External time state
  
  // Non-deterministic - violates purity
  const randomFactor = Math.random(); // ❌ Random value
  
  // Non-deterministic - violates purity
  const marketCondition = getCurrentMarketCondition(); // ❌ External API call
  
  return assessment;
}
```

### 3. Enhanced Context Usage
```typescript
export function optimizePortfolioAllocation(
  context: EnhancedFunctionContext,
  data: PortfolioOptimizationData
): OptimizationStrategy {
  
  context.logger.info('Starting portfolio optimization', {
    current_positions: data.positions.length,
    target_allocation: data.targetAllocation,
    risk_tolerance: data.riskTolerance
  });
  
  // Pure calculation using Modern Portfolio Theory
  const covarianceMatrix = calculateCovarianceMatrix(data.historicalReturns);
  const expectedReturns = calculateExpectedReturns(data.historicalReturns);
  const weights = optimizeWeights(expectedReturns, covarianceMatrix, data.riskTolerance);
  
  // Calculate portfolio metrics
  const expectedReturn = weights.reduce((sum, weight, i) => sum + (weight * expectedReturns[i]), 0);
  const expectedVolatility = Math.sqrt(
    weights.reduce((sum, wi, i) => 
      sum + weights.reduce((innerSum, wj, j) => 
        innerSum + (wi * wj * covarianceMatrix[i][j]), 0), 0)
  );
  
  const strategy: OptimizationStrategy = {
    targetWeights: weights,
    expectedReturn,
    expectedVolatility,
    sharpeRatio: (expectedReturn - data.riskFreeRate) / expectedVolatility,
    rebalanceActions: calculateRebalanceActions(data.positions, weights),
    estimatedCost: calculateRebalanceCost(data.positions, weights, data.transactionCosts)
  };
  
  // Context allows observability without side effects
  context.addMetadata('portfolio.expected_return', expectedReturn);
  context.addMetadata('portfolio.volatility', expectedVolatility);
  context.addMetadata('portfolio.sharpe_ratio', strategy.sharpeRatio);
  context.addMetadata('optimization.rebalance_actions', strategy.rebalanceActions.length);
  
  // Emit high-Sharpe portfolios for monitoring
  if (strategy.sharpeRatio > 2.0) {
    context.emit('HighSharpePortfolioOptimized', {
      userId: data.userId,
      sharpeRatio: strategy.sharpeRatio,
      expectedReturn: expectedReturn
    });
  }
  
  return strategy;
}
```

## Function Categories

### 1. Calculation Functions
```typescript
/**
 * Mathematical calculations with no external dependencies
 */
@EventHandler('YieldOpportunityDetected')
@ShardBy('protocol')
@ResourceRequirements({ cpu: 'medium', memory: '256MB', priority: 'high', timeout: 15000 })
export function calculateCompoundYield(
  context: EnhancedFunctionContext,
  data: YieldOpportunityData
): CompoundYieldStrategy {
  
  // Pure mathematical calculations
  const dailyRate = data.apy / 365;
  const compoundingFactor = Math.pow(1 + dailyRate, 365);
  const adjustedYield = data.apy * (1 - (data.riskScore / 10));
  
  // Risk-adjusted Kelly criterion for position sizing
  const kellyPercentage = calculateKellyCriterion(adjustedYield, data.volatility);
  const maxAllocation = Math.min(kellyPercentage, 0.25); // Max 25% of portfolio
  
  context.logger.debug('Compound yield calculated', {
    daily_rate: dailyRate,
    compounding_factor: compoundingFactor,
    kelly_percentage: kellyPercentage
  });
  
  return {
    protocol: 'compound',
    allocation: maxAllocation,
    expectedApy: adjustedYield,
    compoundingFrequency: 'daily',
    confidence: calculateConfidence(data.historicalPerformance),
    riskMetrics: {
      volatility: data.volatility,
      maxDrawdown: calculateMaxDrawdown(data.historicalPerformance),
      sharpeRatio: adjustedYield / data.volatility
    }
  };
}
```

### 2. Analysis Functions
```typescript
/**
 * Complex analysis and pattern detection
 */
@EventHandler('MarketDataUpdated')
@ShardBy('trading_pair')
@ResourceRequirements({ cpu: 'high', memory: '1GB', priority: 'medium', timeout: 30000 })
export function analyzeMarketTrends(
  context: EnhancedFunctionContext,
  data: MarketTrendData
): TrendAnalysis {
  
  context.logger.info('Analyzing market trends', {
    trading_pair: data.tradingPair,
    data_points: data.priceHistory.length,
    time_window: data.timeWindow
  });
  
  // Technical analysis calculations
  const movingAverages = {
    sma20: calculateSMA(data.priceHistory, 20),
    sma50: calculateSMA(data.priceHistory, 50),
    ema12: calculateEMA(data.priceHistory, 12),
    ema26: calculateEMA(data.priceHistory, 26)
  };
  
  const technicalIndicators = {
    rsi: calculateRSI(data.priceHistory, 14),
    macd: calculateMACD(movingAverages.ema12, movingAverages.ema26),
    bollinger: calculateBollingerBands(data.priceHistory, 20, 2),
    support: findSupportLevels(data.priceHistory),
    resistance: findResistanceLevels(data.priceHistory)
  };
  
  // Trend classification
  const trendDirection = classifyTrend(movingAverages, technicalIndicators);
  const trendStrength = calculateTrendStrength(technicalIndicators);
  const volatilityRegime = classifyVolatilityRegime(data.priceHistory);
  
  const analysis: TrendAnalysis = {
    tradingPair: data.tradingPair,
    trendDirection,
    trendStrength,
    volatilityRegime,
    technicalIndicators,
    signals: generateTradingSignals(technicalIndicators),
    confidence: calculateAnalysisConfidence(technicalIndicators),
    timeHorizon: determineBestTimeHorizon(trendDirection, volatilityRegime)
  };
  
  // Add detailed metadata for debugging
  context.addMetadata('trend.direction', trendDirection);
  context.addMetadata('trend.strength', trendStrength);
  context.addMetadata('volatility.regime', volatilityRegime);
  context.addMetadata('indicators.rsi', technicalIndicators.rsi);
  
  // Emit strong trend signals
  if (trendStrength > 0.8 && analysis.confidence > 0.9) {
    context.emit('StrongTrendDetected', {
      tradingPair: data.tradingPair,
      direction: trendDirection,
      strength: trendStrength,
      confidence: analysis.confidence
    });
  }
  
  return analysis;
}
```

### 3. Strategy Functions
```typescript
/**
 * Trading strategy generation and optimization
 */
@EventHandler('TrendAnalysisCompleted')
@EventHandler('YieldOpportunityAnalyzed')  
@ShardBy('user_id') // User-specific strategy functions
@ResourceRequirements({ cpu: 'critical', memory: '2GB', priority: 'high', timeout: 60000 })
export function generateOptimalTradingStrategy(
  context: EnhancedFunctionContext,
  data: StrategyGenerationData
): TradingStrategy {
  
  context.logger.info('Generating optimal trading strategy', {
    user_id: data.userId,
    available_capital: data.availableCapital,
    risk_profile: data.riskProfile,
    market_conditions: data.marketConditions.length
  });
  
  // Strategy generation algorithm
  const strategies = [
    generateYieldFarmingStrategy(data.yieldOpportunities, data.riskProfile),
    generateArbitrageStrategy(data.arbitrageOpportunities, data.availableCapital),
    generateTrendFollowingStrategy(data.trendAnalysis, data.riskProfile),
    generateMeanReversionStrategy(data.marketData, data.volatilityTargets)
  ];
  
  // Multi-objective optimization
  const optimizedStrategies = strategies.map(strategy => ({
    ...strategy,
    score: calculateStrategyScore(strategy, data.objectives),
    risk: assessStrategyRisk(strategy, data.marketConditions),
    allocation: optimizeAllocation(strategy, data.availableCapital)
  }));
  
  // Portfolio construction with Modern Portfolio Theory
  const optimalPortfolio = constructOptimalPortfolio(
    optimizedStrategies,
    data.availableCapital,
    data.riskProfile
  );
  
  // Risk management overlay
  const riskManagedStrategy = applyRiskManagement(
    optimalPortfolio,
    data.riskLimits,
    data.regulatoryConstraints
  );
  
  const finalStrategy: TradingStrategy = {
    userId: data.userId,
    strategies: riskManagedStrategy.components,
    totalAllocation: riskManagedStrategy.totalAllocation,
    expectedReturn: riskManagedStrategy.expectedReturn,
    expectedRisk: riskManagedStrategy.expectedRisk,
    maxDrawdown: riskManagedStrategy.maxDrawdown,
    executionPlan: generateExecutionPlan(riskManagedStrategy),
    monitoringRules: generateMonitoringRules(riskManagedStrategy),
    exitConditions: generateExitConditions(riskManagedStrategy)
  };
  
  // Rich metadata for analysis
  context.addMetadata('strategy.component_count', finalStrategy.strategies.length);
  context.addMetadata('strategy.total_allocation', finalStrategy.totalAllocation);
  context.addMetadata('strategy.expected_return', finalStrategy.expectedReturn);
  context.addMetadata('strategy.risk_score', finalStrategy.expectedRisk);
  
  // Emit exceptional strategies for monitoring
  if (finalStrategy.expectedReturn > 0.15 && finalStrategy.expectedRisk < 0.1) {
    context.emit('ExceptionalStrategyGenerated', {
      userId: data.userId,
      expectedReturn: finalStrategy.expectedReturn,
      risk: finalStrategy.expectedRisk,
      sharpeRatio: finalStrategy.expectedReturn / finalStrategy.expectedRisk
    });
  }
  
  return finalStrategy;
}
```

### 4. Validation Functions
```typescript
/**
 * Data validation and constraint checking
 */
@EventHandler('TradeExecutionRequested')
@ShardBy('risk_level')
@ResourceRequirements({ cpu: 'low', memory: '128MB', priority: 'critical', timeout: 5000 })
export function validateTradeExecution(
  context: EnhancedFunctionContext,
  data: TradeExecutionData
): TradeValidationResult {
  
  context.logger.debug('Validating trade execution', {
    trading_pair: data.tradingPair,
    amount_usd: data.amountUsd,
    trade_type: data.tradeType
  });
  
  const validationRules = [
    // Amount validation
    {
      name: 'minimum_amount',
      check: () => data.amountUsd >= 10, // Minimum $10 trade
      severity: 'error',
      message: 'Trade amount below minimum threshold'
    },
    
    // Maximum position size
    {
      name: 'maximum_position',  
      check: () => data.amountUsd <= data.maxPositionSize,
      severity: 'error',
      message: 'Trade exceeds maximum position size'
    },
    
    // Risk limits
    {
      name: 'risk_limits',
      check: () => data.riskLevel <= data.maxRiskLevel,
      severity: 'warning',
      message: 'Trade exceeds recommended risk level'
    },
    
    // Regulatory compliance
    {
      name: 'regulatory_compliance',
      check: () => validateRegulatory(data.tradingPair, data.userJurisdiction),
      severity: 'error',
      message: 'Trade violates regulatory constraints'
    },
    
    // Liquidity check
    {
      name: 'liquidity_check',
      check: () => data.availableLiquidity >= data.amountUsd * 1.1, // 110% coverage
      severity: 'warning', 
      message: 'Insufficient liquidity may cause slippage'
    }
  ];
  
  // Execute all validation rules
  const violations = validationRules
    .map(rule => ({
      ...rule,
      passed: rule.check(),
      timestamp: Date.now()
    }))
    .filter(result => !result.passed);
  
  const hasErrors = violations.some(v => v.severity === 'error');
  const hasWarnings = violations.some(v => v.severity === 'warning');
  
  const result: TradeValidationResult = {
    isValid: !hasErrors,
    hasWarnings,
    violations,
    riskScore: calculateValidationRiskScore(violations),
    recommendedActions: generateRecommendations(violations),
    alternativeStrategies: hasErrors ? generateAlternatives(data) : []
  };
  
  // Context metadata for monitoring
  context.addMetadata('validation.errors', violations.filter(v => v.severity === 'error').length);
  context.addMetadata('validation.warnings', violations.filter(v => v.severity === 'warning').length);
  context.addMetadata('validation.risk_score', result.riskScore);
  
  // Emit validation failures for monitoring
  if (hasErrors) {
    context.emit('TradeValidationFailed', {
      tradingPair: data.tradingPair,
      amountUsd: data.amountUsd,
      violations: violations.filter(v => v.severity === 'error'),
      userId: data.userId
    });
  }
  
  return result;
}
```

## Advanced Pure Function Patterns

### 1. Composition and Pipelines
```typescript
/**
 * Complex analysis using function composition
 */
@EventHandler('ComprehensiveAnalysisRequested')
@ShardBy('analysis_type')
@ResourceRequirements({ cpu: 'critical', memory: '4GB', priority: 'high', timeout: 120000 })
export function executeComprehensiveAnalysis(
  context: EnhancedFunctionContext,
  data: ComprehensiveAnalysisData
): ComprehensiveAnalysisResult {
  
  context.logger.info('Starting comprehensive analysis pipeline', {
    analysis_id: data.analysisId,
    trading_pairs: data.tradingPairs.length,
    protocols: data.protocols.length
  });
  
  // Function composition pipeline
  const pipeline = [
    // Stage 1: Data preprocessing
    (input: ComprehensiveAnalysisData) => {
      context.logger.debug('Pipeline stage 1: Data preprocessing');
      return {
        normalizedData: normalizeMultiSourceData(input.marketData),
        filteredPairs: filterValidTradingPairs(input.tradingPairs),
        validProtocols: validateProtocols(input.protocols)
      };
    },
    
    // Stage 2: Individual analysis
    (input: any) => {
      context.logger.debug('Pipeline stage 2: Individual analysis');
      return {
        yieldAnalysis: input.validProtocols.map((protocol: string) => 
          analyzeProtocolYield(input.normalizedData, protocol)
        ),
        arbitrageAnalysis: input.filteredPairs.map((pair: string) =>
          analyzeArbitrageOpportunity(input.normalizedData, pair)
        ),
        trendAnalysis: input.filteredPairs.map((pair: string) =>
          analyzeTrendPattern(input.normalizedData, pair)
        )
      };
    },
    
    // Stage 3: Cross-analysis correlation
    (input: any) => {
      context.logger.debug('Pipeline stage 3: Cross-analysis correlation');
      return {
        correlatedOpportunities: findCorrelatedOpportunities(
          input.yieldAnalysis,
          input.arbitrageAnalysis
        ),
        trendYieldCorrelation: correlateTrendsWithYield(
          input.trendAnalysis,
          input.yieldAnalysis
        ),
        riskCorrelation: analyzeRiskCorrelations(input)
      };
    },
    
    // Stage 4: Optimization
    (input: any) => {
      context.logger.debug('Pipeline stage 4: Strategy optimization');
      return optimizeIntegratedStrategy(
        input.correlatedOpportunities,
        data.constraints,
        data.objectives
      );
    }
  ];
  
  // Execute pipeline with monitoring
  let pipelineInput: any = data;
  
  for (let i = 0; i < pipeline.length; i++) {
    const stage = pipeline[i];
    const stageName = `stage_${i + 1}`;
    
    const stageStartTime = Date.now();
    pipelineInput = stage(pipelineInput);
    const stageExecutionTime = Date.now() - stageStartTime;
    
    context.addMetadata(`pipeline.${stageName}.execution_time`, stageExecutionTime);
    context.logger.debug(`Pipeline stage completed`, {
      stage: i + 1,
      execution_time_ms: stageExecutionTime
    });
  }
  
  const finalResult: ComprehensiveAnalysisResult = pipelineInput;
  
  context.addMetadata('analysis.total_opportunities', finalResult.totalOpportunities);
  context.addMetadata('analysis.expected_return', finalResult.expectedReturn);
  context.addMetadata('analysis.confidence', finalResult.overallConfidence);
  
  return finalResult;
}
```

### 2. State Machines as Pure Functions
```typescript
/**
 * State machine implementation as pure function
 */
@EventHandler('StrategyStateTransition')
@ShardBy('user_id')
@ResourceRequirements({ cpu: 'low', memory: '128MB', priority: 'medium', timeout: 10000 })
export function processStrategyStateMachine(
  context: EnhancedFunctionContext,
  data: StrategyStateData
): StrategyStateResult {
  
  context.logger.debug('Processing strategy state transition', {
    current_state: data.currentState,
    trigger_event: data.triggerEvent,
    strategy_id: data.strategyId
  });
  
  // State machine definition (pure data)
  const stateMachine = {
    states: {
      'ANALYZING': {
        transitions: {
          'ANALYSIS_COMPLETED': 'READY',
          'ANALYSIS_FAILED': 'ERROR',
          'MARKET_CONDITIONS_CHANGED': 'ANALYZING'
        }
      },
      'READY': {
        transitions: {
          'EXECUTE_STRATEGY': 'EXECUTING', 
          'MARKET_CONDITIONS_CHANGED': 'ANALYZING',
          'RISK_THRESHOLD_EXCEEDED': 'PAUSED'
        }
      },
      'EXECUTING': {
        transitions: {
          'EXECUTION_COMPLETED': 'COMPLETED',
          'EXECUTION_FAILED': 'ERROR',
          'EMERGENCY_STOP': 'STOPPED'
        }
      },
      'PAUSED': {
        transitions: {
          'RISK_CONDITIONS_NORMALIZED': 'READY',
          'MANUAL_RESUME': 'READY',
          'TIMEOUT': 'ERROR'
        }
      }
    }
  };
  
  // Pure state transition logic
  const currentStateConfig = stateMachine.states[data.currentState];
  if (!currentStateConfig) {
    throw new Error(`Invalid state: ${data.currentState}`);
  }
  
  const newState = currentStateConfig.transitions[data.triggerEvent];
  if (!newState) {
    // No transition available - stay in current state
    return {
      previousState: data.currentState,
      newState: data.currentState,
      transitionValid: false,
      actions: [],
      reason: `No transition for ${data.triggerEvent} in state ${data.currentState}`
    };
  }
  
  // Generate actions based on state transition
  const actions = generateStateActions(data.currentState, newState, data);
  
  const result: StrategyStateResult = {
    previousState: data.currentState,
    newState,
    transitionValid: true,
    actions,
    metadata: {
      triggerEvent: data.triggerEvent,
      transitionTime: Date.now(),
      strategyId: data.strategyId
    }
  };
  
  context.addMetadata('state.previous', data.currentState);
  context.addMetadata('state.new', newState);
  context.addMetadata('state.actions', actions.length);
  
  // Emit state transition events for monitoring
  context.emit('StrategyStateTransitioned', {
    strategyId: data.strategyId,
    from: data.currentState,
    to: newState,
    trigger: data.triggerEvent,
    actions: actions.length
  });
  
  return result;
}
```

### 3. Data Transformation Functions
```typescript
/**
 * Data transformation and enrichment
 */
@EventHandler('RawMarketDataReceived')
@ShardBy('data_source')
@ResourceRequirements({ cpu: 'medium', memory: '512MB', priority: 'high', timeout: 20000 })
export function enrichMarketData(
  context: EnhancedFunctionContext,
  data: RawMarketDataBatch
): EnrichedMarketData {
  
  context.logger.debug('Enriching market data batch', {
    source: data.source,
    records: data.records.length,
    trading_pairs: new Set(data.records.map(r => r.tradingPair)).size
  });
  
  // Data enrichment pipeline
  const enrichedRecords = data.records.map(record => {
    
    // Calculate derived metrics
    const spreadPercentage = ((record.ask - record.bid) / record.price) * 100;
    const volumeWeightedPrice = (record.price * record.volume);
    const priceChange24h = calculatePriceChange(record, data.historical24h);
    
    // Technical indicators
    const rsi = calculateInstantaneousRSI(record, data.historical24h);
    const volatility = calculateRollingVolatility(record, data.historical24h, 24);
    
    // Liquidity metrics
    const liquidityScore = calculateLiquidityScore(record.volume, record.spread);
    const marketDepth = calculateMarketDepth(record.orderBook);
    
    // Quality score
    const dataQuality = assessDataQuality(record, data.source);
    
    return {
      ...record,
      derived: {
        spreadPercentage,
        volumeWeightedPrice,
        priceChange24h,
        rsi,
        volatility,
        liquidityScore,
        marketDepth,
        dataQuality
      },
      enrichmentMetadata: {
        enrichedAt: Date.now(),
        source: data.source,
        version: '2.0'
      }
    };
  });
  
  // Aggregate statistics
  const aggregateStats = {
    totalVolume: enrichedRecords.reduce((sum, r) => sum + r.volume, 0),
    avgSpread: enrichedRecords.reduce((sum, r) => sum + r.derived.spreadPercentage, 0) / enrichedRecords.length,
    avgVolatility: enrichedRecords.reduce((sum, r) => sum + r.derived.volatility, 0) / enrichedRecords.length,
    avgLiquidity: enrichedRecords.reduce((sum, r) => sum + r.derived.liquidityScore, 0) / enrichedRecords.length,
    dataQualityScore: enrichedRecords.reduce((sum, r) => sum + r.derived.dataQuality, 0) / enrichedRecords.length
  };
  
  const result: EnrichedMarketData = {
    source: data.source,
    timestamp: Date.now(),
    records: enrichedRecords,
    aggregateStats,
    enrichmentVersion: '2.0'
  };
  
  context.addMetadata('enrichment.records_processed', enrichedRecords.length);
  context.addMetadata('enrichment.avg_quality_score', aggregateStats.dataQualityScore);
  context.addMetadata('enrichment.total_volume', aggregateStats.totalVolume);
  
  // Emit data quality alerts
  if (aggregateStats.dataQualityScore < 0.7) {
    context.emit('LowDataQualityDetected', {
      source: data.source,
      qualityScore: aggregateStats.dataQualityScore,
      recordCount: enrichedRecords.length
    });
  }
  
  return result;
}
```

## Helper Functions (Pure Utilities)

### Mathematical Utilities
```typescript
/**
 * Pure mathematical functions used by business logic
 */

export function calculateSMA(prices: number[], period: number): number {
  if (prices.length < period) return prices[prices.length - 1] || 0;
  
  const recentPrices = prices.slice(-period);
  return recentPrices.reduce((sum, price) => sum + price, 0) / recentPrices.length;
}

export function calculateEMA(prices: number[], period: number): number {
  if (prices.length === 0) return 0;
  if (prices.length === 1) return prices[0];
  
  const multiplier = 2 / (period + 1);
  let ema = prices[0];
  
  for (let i = 1; i < prices.length; i++) {
    ema = (prices[i] * multiplier) + (ema * (1 - multiplier));
  }
  
  return ema;
}

export function calculateRSI(prices: number[], period: number): number {
  if (prices.length < period + 1) return 50; // Neutral RSI
  
  const changes = prices.slice(1).map((price, i) => price - prices[i]);
  const gains = changes.filter(change => change > 0);
  const losses = changes.filter(change => change < 0).map(loss => Math.abs(loss));
  
  const avgGain = gains.length > 0 ? gains.reduce((sum, gain) => sum + gain, 0) / gains.length : 0;
  const avgLoss = losses.length > 0 ? losses.reduce((sum, loss) => sum + loss, 0) / losses.length : 0;
  
  if (avgLoss === 0) return 100; // No losses = max RSI
  
  const rs = avgGain / avgLoss;
  return 100 - (100 / (1 + rs));
}

export function calculateKellyCriterion(expectedReturn: number, volatility: number): number {
  // Kelly percentage = (expected return - risk-free rate) / variance
  const riskFreeRate = 0.02; // 2% risk-free rate
  const excessReturn = expectedReturn - riskFreeRate;
  const variance = volatility * volatility;
  
  const kellyPercentage = excessReturn / variance;
  
  // Cap at 25% for safety (fractional Kelly)
  return Math.min(Math.max(kellyPercentage, 0), 0.25);
}
```

### Business Logic Utilities
```typescript
/**
 * Pure business logic functions
 */

export function calculatePositionSize(
  portfolioValue: number,
  riskPerTrade: number,
  stopLossDistance: number
): number {
  // Position sizing based on risk management
  const riskAmount = portfolioValue * riskPerTrade;
  return riskAmount / stopLossDistance;
}

export function determineOptimalEntry(
  currentPrice: number,
  technicalLevels: TechnicalLevels,
  volatility: number
): EntryStrategy {
  
  const supportLevel = technicalLevels.support;
  const resistanceLevel = technicalLevels.resistance;
  const priceRange = resistanceLevel - supportLevel;
  
  // Calculate optimal entry based on price position within range
  const pricePosition = (currentPrice - supportLevel) / priceRange;
  
  if (pricePosition < 0.2) {
    return {
      strategy: 'aggressive_buy',
      targetPrice: currentPrice,
      stopLoss: supportLevel * 0.98,
      takeProfit: supportLevel + (priceRange * 0.618), // Golden ratio target
      positionSize: 1.0 // Full position
    };
  } else if (pricePosition > 0.8) {
    return {
      strategy: 'aggressive_sell',
      targetPrice: currentPrice,
      stopLoss: resistanceLevel * 1.02,
      takeProfit: resistanceLevel - (priceRange * 0.618),
      positionSize: 1.0
    };
  } else {
    return {
      strategy: 'wait_for_breakout',
      targetPrice: pricePosition > 0.5 ? resistanceLevel : supportLevel,
      stopLoss: pricePosition > 0.5 ? currentPrice : supportLevel * 0.98,
      takeProfit: pricePosition > 0.5 ? resistanceLevel * 1.05 : resistanceLevel,
      positionSize: 0.5 // Reduced position in range
    };
  }
}

export function calculateRiskMetrics(
  positions: Position[],
  marketData: MarketData[],
  correlations: CorrelationMatrix
): PortfolioRiskMetrics {
  
  // Value at Risk calculation
  const portfolioValue = positions.reduce((sum, pos) => sum + pos.valueUsd, 0);
  const weights = positions.map(pos => pos.valueUsd / portfolioValue);
  
  // Portfolio volatility using correlation matrix
  const portfolioVolatility = calculatePortfolioVolatility(weights, correlations);
  
  // 95% VaR (1-day)
  const valueAtRisk = portfolioValue * portfolioVolatility * 1.645; // 95th percentile
  
  // Maximum drawdown based on historical data
  const historicalReturns = calculatePortfolioReturns(positions, marketData);
  const maxDrawdown = calculateMaxDrawdown(historicalReturns);
  
  // Concentration risk
  const concentrationRisk = calculateConcentrationRisk(weights);
  
  // Liquidity risk
  const liquidityRisk = calculateLiquidityRisk(positions, marketData);
  
  return {
    portfolioValue,
    portfolioVolatility,
    valueAtRisk,
    maxDrawdown,
    concentrationRisk,
    liquidityRisk,
    diversificationRatio: calculateDiversificationRatio(weights, correlations),
    riskBudget: {
      used: valueAtRisk / portfolioValue,
      available: 0.15 - (valueAtRisk / portfolioValue) // 15% max risk budget
    }
  };
}
```

## Testing Pure Functions

### Unit Testing
```typescript
describe('Pure Function Core', () => {
  
  describe('calculateYieldStrategy', () => {
    it('should calculate conservative strategy for low APY', () => {
      // Arrange
      const context = createMockContext();
      const data: YieldOpportunityData = {
        protocol: 'aave',
        apy: 0.05,
        tvl: 1000000,
        riskScore: 7,
        historicalPerformance: [0.04, 0.05, 0.06, 0.05, 0.04]
      };
      
      // Act
      const result = calculateYieldStrategy(context, data);
      
      // Assert
      expect(result.allocation).toBeLessThan(0.2);
      expect(result.protocol).toBe('aave');
      expect(result.confidence).toBeGreaterThan(0);
      expect(context.logger.debug).toHaveBeenCalledWith(
        expect.stringContaining('Yield calculation completed'),
        expect.objectContaining({ original_apy: 0.05 })
      );
    });
    
    it('should calculate aggressive strategy for high APY', () => {
      const context = createMockContext();
      const data: YieldOpportunityData = {
        protocol: 'aave',
        apy: 0.25,
        tvl: 10000000,
        riskScore: 2,
        historicalPerformance: [0.24, 0.25, 0.26, 0.25, 0.24]
      };
      
      const result = calculateYieldStrategy(context, data);
      
      expect(result.allocation).toBeGreaterThan(0.3);
      expect(result.confidence).toBeGreaterThan(0.8);
    });
  });
  
  describe('validateTradeExecution', () => {
    it('should reject trades below minimum amount', () => {
      const context = createMockContext();
      const data: TradeExecutionData = {
        tradingPair: 'ETH-USDC',
        amountUsd: 5, // Below $10 minimum
        maxPositionSize: 100000,
        riskLevel: 'low',
        maxRiskLevel: 'medium'
      };
      
      const result = validateTradeExecution(context, data);
      
      expect(result.isValid).toBe(false);
      expect(result.violations).toContainEqual(
        expect.objectContaining({
          name: 'minimum_amount',
          severity: 'error'
        })
      );
    });
  });
});

function createMockContext(): EnhancedFunctionContext {
  return {
    correlationId: 'test-correlation',
    businessContext: createMockBusinessContext(),
    logger: createMockLogger(),
    shard: createMockShardInfo(),
    metrics: createMockMetrics(),
    emit: jest.fn(),
    addMetadata: jest.fn()
  };
}
```

### Integration Testing
```typescript
describe('Function Integration', () => {
  
  it('should execute complete yield analysis pipeline', async () => {
    // Arrange
    const functionManager = new YieldFunctionManager('test-shard', []);
    const event: DomainEvent<YieldOpportunityData> = createYieldEvent();
    
    // Act
    await functionManager.processIncomingEvent(event);
    
    // Assert
    expect(mockEventStore.append).toHaveBeenCalledWith(
      expect.stringContaining('yield-'),
      expect.arrayContaining([
        expect.objectContaining({
          eventType: 'YieldStrategyCalculated'
        })
      ])
    );
    
    expect(mockMessageBus.publish).toHaveBeenCalledWith(
      'lucent.events.domain.yield_farming.YieldStrategyCalculated',
      expect.anything()
    );
  });
});
```

This pure function core pattern provides **complete functional purity** while maintaining **full integration** with the I/O shell infrastructure through enhanced context. Functions remain **easily testable** and **side-effect free** while getting **enterprise-grade observability** and **dynamic sharding capabilities**.