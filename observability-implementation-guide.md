# Lucent Observability Implementation Guide

## Overview

Lucent already has a comprehensive observability framework built into the infrastructure package. **No boilerplate needed!** Just extend the existing service base classes and your metrics will automatically appear in our Grafana dashboards.

## Quick Start

### 1. Install the Infrastructure Package

```bash
npm install @lucent-financial-group/lucent-infrastructure
```

### 2. Set Environment Variables

```bash
# Required for proper tagging and metrics
SERVICE_NAME=wallet-service  # or user-service, play-service, etc.
ENVIRONMENT=prod            # or dev, staging
HOST_NAME=lucent-host-01    # or lucent-host-02

# OpenTelemetry endpoint
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318/v1/traces
```

### 3. Extend the Service Base Classes

```typescript
// Your existing WalletService.ts
import { LucentServiceBase } from '@lucent-financial-group/lucent-infrastructure';
import { CryptoBusinessContext } from '@lucent-financial-group/lucent-infrastructure';

export class WalletService extends LucentServiceBase {
  constructor(infrastructure: IInfrastructureProvider) {
    super(infrastructure, 'wallet-service'); // This sets all the tags automatically
  }

  async createWallet(userId: string, network: string): Promise<Wallet> {
    const businessContext: CryptoBusinessContext = {
      domain: 'wallet-management',
      boundedContext: 'wallets',
      aggregateType: 'Wallet',
      aggregateId: `wallet-${network}-${userId}`,
      userId,
      priority: 'normal',
      crypto: {
        protocol: network, // This becomes blockchain_network tag
        strategyType: 'wallet-creation' // This becomes wallet_operation tag
      }
    };

    // This automatically creates metrics with all our standardized tags!
    return this.executeBusinessOperation('wallet_creation', businessContext, async () => {
      // Your existing wallet creation logic here
      const wallet = await this.generateWallet(network);
      return wallet;
    });
  }
}
```

### 4. That's It!

The framework automatically:
- ✅ Creates OpenTelemetry spans with proper trace IDs
- ✅ **NEW: Generates Prometheus metrics** with our standardized dashboard tags
- ✅ Handles errors and records them with proper severity
- ✅ Manages timing and duration metrics automatically
- ✅ Publishes events to KurrentDB with context
- ✅ **NEW: Maps business context** to `service_name`, `blockchain_network`, `wallet_operation`, etc.

## Available Base Classes

### `LucentServiceBase`
Generic base for any service. Auto-creates metrics with these tags:
- `service_name` 
- `environment`
- `host_name`
- `business.domain`
- `business.aggregate_type`
- `user_id`
- `protocol_type`

### `CryptoTradingServiceBase`
Specialized for DeFi operations. Adds crypto-specific tags:
- `blockchain_network`
- `crypto_symbol` 
- `defi_protocol`
- `defi_operation`
- `play_strategy`
- `risk_level`

### `UserServiceBase`
Specialized for user operations. Adds user-specific context:
- `auth_method`
- `user_role`

## Real Examples from Existing Services

### User Service
```typescript
export class AuthService extends UserServiceBase {
  async authenticateUser(authData: AuthRequest): Promise<AuthResult> {
    // This creates metrics with auth_method, user_role tags automatically
    return this.executeUserOperation('authentication', authData.userId, async () => {
      return await this.validateCredentials(authData);
    });
  }
}
```

### Wallet Service  
```typescript
export class WalletService extends LucentServiceBase {
  async getBalance(userId: string, network: NetworkType): Promise<number> {
    const businessContext: CryptoBusinessContext = {
      domain: 'wallet-management',
      boundedContext: 'wallets', 
      aggregateType: 'Wallet',
      aggregateId: userId,
      userId,
      crypto: { protocol: network.toLowerCase() }
    };

    return this.executeBusinessOperation('balance_check', businessContext, async () => {
      return await blockchainService.getBalance(walletAddress, network);
    });
  }
}
```

### Play Execution Service
```typescript
export class PlayService extends CryptoTradingServiceBase {  
  async executePlay(playData: PlayRequest): Promise<PlayResult> {
    // This automatically tags with defi_protocol, play_strategy, etc.
    return this.executeTrade(
      playData.strategy,        // -> play_strategy tag
      playData.tradingPair,     // -> crypto.trading_pair tag  
      playData.exchange,        // -> defi_protocol tag
      playData.amountUsd,       // -> crypto.amount_usd tag
      playData.userId,
      async () => {
        return await this.defiAdapter.execute(playData);
      }
    );
  }
}
```

## WebSocket Integration

For real-time WebSocket metrics:

```typescript
export class WebSocketService extends LucentServiceBase {
  onConnection(socket: WebSocket, channel: string) {
    // Automatically creates websocket metrics with ws_channel, ws_event_type tags
    this.publishDomainEvent(
      'websocket.connected',
      `ws-${channel}`, 
      { channel, connectionId: socket.id },
      {
        domain: 'real-time-communication',
        boundedContext: 'websockets',
        aggregateType: 'WebSocketConnection',
        aggregateId: socket.id,
        priority: 'normal'
      }
    );
  }
}
```

## Important Notes

### Environment Variables Are Critical
The framework uses these environment variables to create proper metric labels:

```bash
# These become the core tags in ALL metrics
SERVICE_NAME=wallet-service  # → service_name="wallet-service" 
ENVIRONMENT=prod            # → environment="prod"
HOST_NAME=lucent-host-01    # → host_name="lucent-host-01"

# OpenTelemetry endpoint
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318/v1/traces
```

**Without these environment variables, your metrics won't appear correctly in dashboards!**

## Testing Your Implementation

Run the synthetic data generator to verify your dashboards:

```bash
cd lucent-infrastructure
npm run synthetic-data
```

This will generate realistic metrics using the same patterns your services should use. The script runs TypeScript directly without needing to compile first.

## What Gets Created Automatically

When you use `executeBusinessOperation()`, the framework automatically creates:

1. **OpenTelemetry Span** with business context
2. **NEW: Prometheus Metrics** with standardized labels that match our dashboards:
   - `lucent_requests_total{service_name="wallet-service", blockchain_network="ethereum", wallet_operation="create", response_status="2xx"}`
   - `lucent_request_duration_seconds{service_name="play-service", play_strategy="leverage", defi_protocol="uniswap"}`
   - `lucent_errors_total{service_name="user-service", error_type="validation", error_severity="medium"}`
   - `up{service_name="your-service", environment="prod", host_name="lucent-host-01"}`
3. **KurrentDB Events** with full trace context  
4. **Structured Logs** with correlation IDs

### Automatic Tag Mapping

The framework intelligently maps your business context to dashboard tags:

| Business Context | Dashboard Tag | Example |
|------------------|---------------|---------|
| `crypto.protocol` | `blockchain_network` | `"ethereum"` |
| `crypto.strategyType` | `play_strategy` | `"leverage"` |
| `crypto.exchange` | `defi_protocol` | `"uniswap"` |
| `operationName` containing "wallet" | `wallet_operation` | `"create"` |
| `domain` | `http_endpoint` | `"/wallets"` |
| Error analysis | `error_type` | `"business"` |
| High USD amounts | `error_severity` | `"critical"` |

## Dashboard Queries Will Work

Your metrics will automatically work with our dashboard queries like:
- `rate(lucent_requests_total{service_name="wallet-service"}[5m])`
- `histogram_quantile(0.95, lucent_request_duration_seconds_bucket{blockchain_network="ethereum"})`
- `sum by (play_strategy) (lucent_requests_total{service_name="play-service"})`

## Need Help?

- Check the synthetic data generator for examples: `lucent-infrastructure/scripts/generate-dashboard-test-data.ts`
- Look at existing service implementations in the infrastructure package
- All the complex OpenTelemetry setup is already done for you!

The key insight: **Don't write observability code - just use the business context patterns and everything works automatically!**