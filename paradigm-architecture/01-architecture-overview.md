# Lucent Services Paradigm Architecture Overview

**Document Version:** 1.0  
**Date:** 2025-09-03  
**Context:** Functional Core + Imperative Shell + Dynamic Function Registry

## Architectural Philosophy

Lucent Services implements a sophisticated **Functional Core, Imperative Shell** architecture enhanced with **Dynamic Function Registry** for load-balanced distributed computing. This approach combines the benefits of functional programming (testability, determinism) with object-oriented infrastructure (tracing, resilience) and dynamic systems (hot-swapping, sharding).

## Core Principles

### 1. Functional Core (Pure Functions)
- **No side effects**: Functions only transform input to output
- **Deterministic**: Same input always produces same output  
- **Testable**: Easy to unit test without mocks
- **Composable**: Functions can be combined and reused
- **Type-safe**: Full TypeScript validation

### 2. Imperative Shell (OOP Infrastructure)
- **Handles I/O**: External API calls, database operations
- **Event publishing**: Domain event management
- **Cross-service communication**: Command/response patterns
- **Resource management**: Connections, transactions, cleanup
- **Observability**: Tracing, logging, metrics

### 3. Dynamic Function Registry (Load Distribution)
- **Hot-swappable functions**: Deploy new strategies without downtime
- **Intelligent sharding**: Route functions to optimal nodes
- **Load balancing**: Distribute execution based on resource usage
- **A/B testing**: Enable/disable strategies dynamically
- **Resource optimization**: Match function requirements to available capacity

### 4. CQRS Pattern Integration
- **Command handlers**: Synchronous writes with immediate validation
- **Query handlers**: Fast reads from Redis projections
- **Saga handlers**: Long-running workflows with compensation
- **Event processing**: Asynchronous projections and notifications
- **Clear separation**: Commands (writes) vs Queries (reads) vs Events (async)

### 5. I/O Event Scheduling
- **Time-based scheduling**: Cron-like scheduled I/O operations
- **Event-triggered scheduling**: I/O operations triggered by domain events
- **Conditional scheduling**: Execute I/O when specific conditions are met
- **External event waiting**: Wait for blockchain confirmations or API responses
- **Fallback handling**: Graceful timeout and error recovery for I/O operations

## Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    HTTP/API Layer (NestJS)                  │ 
│  Controllers, Guards, Interceptors, Validation              │
└─────────────────────┬───────────────────────────────────────┘
                      │ 
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    CQRS Layer                               │
│  Commands (Sync) │ Queries (Fast) │ Sagas (Long-Running)   │
│  - Validation    │ - Redis Cache  │ - Multi-Step Workflows │
│  - Immediate     │ - Sub-ms reads │ - Compensation Logic   │
│    Response      │               │ - State Persistence     │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              Imperative Shell (OOP Service Classes)        │
│  - Service Classes (extend CryptoTradingServiceBase)       │
│  - I/O Operations (EventStore, Redis, APIs)               │
│  - Command/Query Dispatchers                               │
│  - I/O Event Scheduling (time, condition, external)       │
│  - Function Manager Communication                          │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│            Function Manager (Dynamic Orchestration)        │
│  - Event Processing    - Load Balancing    - Sharding     │
│  - Function Selection  - Result Handling   - Projections  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                Functional Core (Pure Functions)            │
│  - Command Handlers  - Query Handlers   - Saga Steps      │
│  - Business Logic    - Calculations     - Validations     │
│  - Risk Assessments  - Strategies       - Projections     │ 
│  NO SIDE EFFECTS - FULLY TESTABLE - DYNAMICALLY ROUTED    │
└─────────────────────────────────────────────────────────────┘
```

## Event Flow Architecture

### 1. Service to Function Manager Flow
```
HTTP Request → Service Class → EventStore/APIs → Function Manager → Pure Function → Result
     ↓              ↓              ↓                  ↓              ↓           ↓
I/O Layer    Business Context   Data Gathering   Load Balancing   Computation  EventStore + Redis
```

### 2. Cross-Shard Function Distribution
```
Service A → Function Manager Shard 1 → Pure Functions (Aave, Yield Farming)
Service B → Function Manager Shard 2 → Pure Functions (Arbitrage, Risk)
Service C → Function Manager Shard 3 → Pure Functions (Analytics, Reporting)
            ↓                           ↓
       Load Balancer              Function Registry
            ↓                           ↓  
      Health Monitor             Type-Safe Execution
```

### 3. Dynamic Function Distribution
```
Function Registry Database
         ↓
Load Balancer Analyzes:
  - Current shard CPU/memory usage
  - Function execution times
  - Business priority levels
         ↓
Routes Functions to Optimal Shards:
  - High-value trades → Whale shard (high CPU/memory)
  - Aave strategies → Aave shard (protocol expertise)
  - Arbitrage detection → Arbitrage shard (low latency)
```

## Integration with Infrastructure

### Base Classes Provide:
- **Business Context Management**: Automatic tracing with crypto-specific metadata
- **Event Sourcing Integration**: Domain event publishing to EventStore
- **Cross-Service Communication**: Type-safe command/response via NATS
- **Resilience Patterns**: Circuit breakers, rate limiting, chaos engineering
- **Observability**: Structured logging, distributed tracing, health monitoring

### Pure Functions Receive:
- **Enhanced Context**: Access to logging, tracing, event emission
- **Business Metadata**: User ID, trading pairs, risk levels, workflow IDs
- **Shard Information**: Current load, available resources, peer shards
- **Execution Environment**: Correlation IDs, timeout management, retry logic

## Key Benefits

### Development Experience
- **Pure functions**: Easy to write, test, and reason about
- **Base classes**: Infrastructure handled automatically
- **Type safety**: Full TypeScript validation throughout
- **Hot deployment**: Update strategies without service restart

### Operational Excellence  
- **Load balancing**: Functions execute on optimal nodes
- **Resource efficiency**: Match function requirements to available capacity
- **Fault tolerance**: Failed functions don't cascade to other shards
- **Observability**: Trace function execution across distributed nodes

### Business Agility
- **A/B testing**: Deploy competing strategies simultaneously
- **Risk management**: Disable high-risk functions instantly  
- **Performance optimization**: Route heavy calculations to powerful nodes
- **Compliance**: Enable/disable functions per jurisdiction

## File Organization

Each service follows this structure:
```
crypto-service/
├── src/
│   ├── domain/                    # Functional Core
│   │   ├── yield-calculations.ts  # Pure functions with decorators
│   │   ├── risk-assessments.ts    # Pure functions with decorators
│   │   └── arbitrage-detection.ts # Pure functions with decorators
│   ├── managers/                  # Imperative Shell
│   │   ├── yield-function-manager.ts    # Extends LucentServiceBase
│   │   └── arbitrage-function-manager.ts # Extends LucentServiceBase  
│   ├── controllers/               # HTTP/API Layer
│   │   └── crypto.controller.ts   # NestJS controllers
│   └── registry/                  # Generated
│       └── function-registry.ts   # Compile-time generated
```

## Next Steps

Refer to the following documents for detailed implementation guidance:

- **02-decorator-pattern.md**: How to use decorators for function registration
- **03-function-manager.md**: Implementing function managers with base classes
- **04-io-shell-pattern.md**: OOP at the edges for infrastructure integration
- **05-pure-function-core.md**: Writing pure functions with enhanced context
- **06-type-safe-registry.md**: Maintaining TypeScript safety in dynamic systems
- **07-load-balancing.md**: Implementing intelligent function distribution
- **08-debugging-guide.md**: Using distributed tracing for function debugging

This architecture enables **distributed functional programming** with **enterprise-grade infrastructure** - the perfect foundation for high-performance crypto trading systems.