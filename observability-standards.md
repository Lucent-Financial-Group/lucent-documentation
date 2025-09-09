# Lucent Platform Observability Standards

## Overview

This document defines the observability architecture for the Lucent DeFi platform, implementing OpenTelemetry standards with SOLID principles for scalable monitoring and debugging capabilities using KurrentDB and WebSocket communications.

## Dashboard Architecture

### Five-Layer Hierarchical Structure

#### Layer 1: Executive/Business Overview
- **Platform Health KPIs Dashboard**
  - Overall system availability and performance
  - Critical business metrics and SLA compliance
  - Cost tracking and resource utilization trends

- **Business Metrics Dashboard**  
  - User growth and engagement trends
  - Financial transaction volumes
  - Revenue and performance indicators

#### Layer 2: Service Architecture Overview
- **Service Dependency Dashboard**
  - Service dependency mapping and health matrix
  - Inter-service communication patterns
  - Event-driven architecture monitoring
  - Cross-service error correlation

#### Layer 3: Individual Service Dashboards
- **User Service Dashboard**
- **Wallet Service Dashboard** 
- **Play Service Dashboard**
- **Task Service Dashboard**
- **Portfolio Service Dashboard**
- **Notification Service Dashboard**

*Each dashboard follows standardized layout with SOLID principle compliance monitoring*

#### Layer 4: Infrastructure & Platform
- **Internal Infrastructure Dashboard**
  - Host health, resource utilization, scaling metrics
  
- **KurrentDB & Storage Dashboard**
  - KurrentDB performance, stream health, event processing analytics
  
- **WebSocket & Networking Dashboard**
  - WebSocket connection health, message throughput, real-time communication patterns

#### Layer 5: Business Process Flows
- **User Journey Dashboard**
  - End-to-end user registration → wallet → first play
  - Authentication flow success rates
  
- **DeFi Operations Dashboard**
  - Play execution workflows
  - Transaction processing pipelines
  
- **Admin Workflow Dashboard**
  - Task completion tracking
  - Administrative operation monitoring

## Standardized Tagging Taxonomy

### Core Tags (Required on all metrics)
```yaml
# Service Identification
service.name: "user-service" | "wallet-service" | "play-service" | "task-service" | "portfolio-service" | "notification-service"
service.version: "1.0.0"
environment: "dev" | "staging" | "prod"

# Internal Infrastructure
host.name: "lucent-host-01" | "lucent-host-02"
# region: "us-east" # Future: when multi-region deployment
```

### WebSocket Communication Tags
```yaml
# WebSocket Operations
ws.event_type: "connection" | "message" | "error" | "close"
ws.channel: "user-updates" | "wallet-events" | "play-status" | "task-notifications"
ws.message_type: "subscribe" | "unsubscribe" | "data" | "heartbeat"
ws.client_id: "client-uuid"
ws.connection_status: "connected" | "disconnected" | "reconnecting"
```

### KurrentDB Specific Tags
```yaml
# Database Operations
kurrentdb.stream: "users" | "wallets" | "plays" | "tasks" | "portfolios" 
kurrentdb.operation: "append" | "read" | "subscribe" | "checkpoint"
kurrentdb.partition: "0" | "1" | "2"
event.type: "user.created" | "wallet.generated" | "play.executed"
```

### Service-Specific Tags

#### Authentication & User Management
```yaml
auth.method: "web3auth" | "jwt"
auth.provider: "metamask" | "walletconnect" | "coinbase"
user.role: "user" | "admin" | "super-admin"
```

#### Wallet Operations
```yaml
blockchain.network: "ethereum" | "polygon" | "solana" | "bitcoin"
wallet.operation: "create" | "import" | "sign" | "balance-check"
crypto.symbol: "ETH" | "SOL" | "BTC" | "USDC"
```

#### DeFi Operations
```yaml
play.strategy: "leverage" | "staking" | "liquidity" | "farming"
play.status: "pending" | "executing" | "completed" | "failed"
defi.protocol: "uniswap" | "compound" | "aave" | "curve"
defi.operation: "swap" | "stake" | "lend" | "farm"
```

### Error & Performance Tags
```yaml
# Error Classification
error.type: "business" | "infrastructure" | "external" | "validation" | "websocket"
error.severity: "critical" | "high" | "medium" | "low"

# Communication Context
protocol.type: "websocket" | "http" | "kurrentdb"
request.method: "GET" | "POST" | "PUT" | "DELETE" | "WS_MESSAGE"
response.status: "2xx" | "4xx" | "5xx" | "ws_connected" | "ws_error"

# Performance Classification
performance.tier: "critical" | "important" | "normal"
sli.type: "availability" | "latency" | "throughput" | "error-rate"
```

## SOLID Principles Application

### Single Responsibility Principle (SRP)
- Each dashboard serves one specific monitoring need
- Metrics grouped by single business function
- Clear separation between infrastructure and business metrics

### Open/Closed Principle (OCP)
- Dashboard templates allow adding new services without modification
- Standardized tagging enables extension without changing existing dashboards
- Plugin-based approach for new metric types

### Liskov Substitution Principle (LSP)
- All service dashboards follow identical interface patterns
- Consistent metric naming across services
- Interchangeable dashboard components

### Interface Segregation Principle (ISP)
- Role-based dashboard access (DevOps vs Business views)
- Focused dashboards avoiding metric overload
- Specific interfaces for different stakeholder needs

### Dependency Inversion Principle (DIP)
- Dashboards depend on standardized tagging taxonomy
- Abstract metric interfaces instead of concrete implementations
- Configuration-driven dashboard generation

## Implementation Guidelines

### Metric Naming Conventions
```yaml
# Counter Metrics
lucent_requests_total{service.name="user-service", protocol.type="websocket"}
lucent_errors_total{service.name="wallet-service", error.type="external"}

# Histogram Metrics
lucent_request_duration_seconds{service.name="play-service", defi.operation="swap"}
lucent_kurrentdb_operation_duration_seconds{kurrentdb.stream="portfolios", kurrentdb.operation="append"}

# Gauge Metrics
lucent_websocket_connections_active{service.name="notification-service"}
lucent_wallet_balance_usd{blockchain.network="ethereum", crypto.symbol="ETH"}
```

### Dashboard Organization Standards
1. **Consistent Layout**: All dashboards follow identical grid structure
2. **Standardized Colors**: Red (errors), Yellow (warnings), Green (success), Blue (info)
3. **Time Windows**: Default 15min, with 1h, 6h, 24h, 7d options
4. **Drill-Down**: Each panel links to relevant detailed dashboards

### Alert Configuration
- **Critical**: System-wide failures, security breaches, WebSocket infrastructure down
- **High**: Service-specific failures, SLA violations, KurrentDB stream issues
- **Medium**: Performance degradation, elevated error rates, connection drops
- **Low**: Resource warnings, maintenance notifications

This observability architecture provides comprehensive monitoring while maintaining debugging focus and following enterprise standards for KurrentDB and WebSocket-based systems.