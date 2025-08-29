# Lucent Core Services Design Specification

## Overview

This document defines the detailed specifications for each microservice in the Lucent DeFi platform, including data models, API contracts, business logic, and inter-service communication patterns.

## 1. User Service

### Domain Responsibility
Authentication, authorization, user profile management, and role-based access control.

### MongoDB Schema Design

```typescript
// users collection
interface UserDocument {
  _id: ObjectId;
  web3AuthId: string; // Unique identifier from Web3Auth
  email?: string;
  name?: string;
  profileImage?: string;
  authProvider: string; // 'web3auth', 'metamask', etc.
  role: 'user' | 'admin' | 'super_admin';
  isActive: boolean;
  isEmailVerified: boolean;
  permissions: string[]; // Granular permissions array
  metadata: Record<string, any>; // Flexible additional data
  playSettings: {
    leverage: number; // 0-75 LTV percentage
    riskTolerance: 'low' | 'medium' | 'high';
    autoRebalance: boolean;
  };
  portfolioBalance: number; // USD value
  createdAt: Date;
  updatedAt: Date;
  lastLoginAt?: Date;
  bannedAt?: Date;
  banReason?: string;
}

// sessions collection (for JWT blacklisting)
interface SessionDocument {
  _id: ObjectId;
  userId: ObjectId;
  jti: string; // JWT ID for token invalidation
  issuedAt: Date;
  expiresAt: Date;
  isRevoked: boolean;
  deviceInfo?: {
    userAgent: string;
    ipAddress: string;
    location?: string;
  };
}
```

### API Contract

```typescript
// Authentication endpoints
POST /auth/login
POST /auth/refresh
POST /auth/logout
POST /auth/verify-token

// User management endpoints
GET /users/profile
PUT /users/profile
PUT /users/settings
GET /users/{userId} // Admin only
PUT /users/{userId}/role // Super admin only
POST /users/{userId}/ban // Admin only
DELETE /users/{userId}/ban // Admin only
```

### Service Interface

```typescript
interface IUserService {
  // Authentication
  authenticateWithWeb3Auth(web3AuthData: Web3AuthPayload): Promise<AuthResult>;
  refreshToken(refreshToken: string): Promise<AuthResult>;
  validateToken(token: string): Promise<UserDocument>;
  revokeToken(jti: string): Promise<void>;
  
  // User management
  getUserById(userId: string): Promise<UserDocument>;
  updateUserProfile(userId: string, updates: Partial<UserDocument>): Promise<UserDocument>;
  updateUserSettings(userId: string, settings: UserPlaySettings): Promise<UserDocument>;
  
  // Admin operations
  listUsers(filters: UserFilters): Promise<PaginatedResult<UserDocument>>;
  updateUserRole(userId: string, role: UserRole): Promise<UserDocument>;
  banUser(userId: string, reason: string): Promise<UserDocument>;
  unbanUser(userId: string): Promise<UserDocument>;
}
```

### Business Logic Rules

1. **Authentication Flow**: Web3Auth → JWT generation → Session tracking
2. **Role Hierarchy**: Super Admin > Admin > User
3. **Permission Inheritance**: Higher roles inherit lower role permissions
4. **Session Management**: Track active sessions, allow revocation
5. **Profile Validation**: Email format, name length, image URL validation

---

## 2. Wallet Service

### Domain Responsibility
Multi-chain wallet generation, private key management, address derivation, and balance tracking.

### MongoDB Schema Design

```typescript
// wallets collection
interface WalletDocument {
  _id: ObjectId;
  userId: ObjectId; // Reference to user
  network: 'EVM' | 'Solana' | 'Bitcoin';
  address: string;
  publicKey: string;
  privateKeySecretName: string; // Azure Key Vault reference
  derivationPath?: string; // HD wallet path
  isActive: boolean;
  metadata: {
    walletType: 'generated' | 'imported';
    createdBy: 'system' | 'user';
    lastBalanceUpdate: Date;
    balanceUSD: number;
    nativeBalance: number;
  };
  createdAt: Date;
  updatedAt: Date;
}

// wallet_balances collection (for historical tracking)
interface WalletBalanceDocument {
  _id: ObjectId;
  walletId: ObjectId;
  network: string;
  address: string;
  tokenBalances: {
    symbol: string;
    contractAddress?: string;
    balance: string;
    decimals: number;
    usdValue: number;
  }[];
  totalUSD: number;
  timestamp: Date;
}
```

### API Contract

```typescript
// Wallet management endpoints
GET /wallets/user/{userId}
GET /wallets/{walletId}
POST /wallets/generate
POST /wallets/import
PUT /wallets/{walletId}/activate
DELETE /wallets/{walletId}

// Balance operations
GET /wallets/{walletId}/balance
POST /wallets/{walletId}/refresh-balance
GET /wallets/{walletId}/history

// Transaction operations
POST /wallets/{walletId}/sign-transaction
POST /wallets/{walletId}/send-transaction
GET /wallets/{walletId}/transactions
```

### Service Interface

```typescript
interface IWalletService {
  // Wallet management
  generateWallet(userId: string, network: BlockchainNetwork): Promise<WalletDocument>;
  importWallet(userId: string, network: BlockchainNetwork, privateKey: string): Promise<WalletDocument>;
  getUserWallets(userId: string): Promise<WalletDocument[]>;
  getWallet(walletId: string): Promise<WalletDocument>;
  
  // Key management
  getPrivateKey(walletId: string): Promise<string>; // Admin only
  rotatePrivateKey(walletId: string): Promise<WalletDocument>;
  
  // Balance operations
  getWalletBalance(walletId: string): Promise<WalletBalanceDocument>;
  refreshBalance(walletId: string): Promise<WalletBalanceDocument>;
  getBalanceHistory(walletId: string, timeframe: string): Promise<WalletBalanceDocument[]>;
  
  // Transaction operations
  signTransaction(walletId: string, transaction: TransactionData): Promise<SignedTransaction>;
  broadcastTransaction(signedTx: SignedTransaction): Promise<TransactionResult>;
}
```

### Business Logic Rules

1. **One Wallet Per Network**: Users can have one active wallet per blockchain network
2. **Key Vault Integration**: All private keys stored in Azure Key Vault with encryption
3. **Balance Updates**: Automatic balance refresh every 5 minutes for active wallets
4. **Transaction Limits**: Rate limiting on transaction signing and broadcasting
5. **Audit Trail**: Complete transaction history with status tracking

---

## 3. Play Service

### Domain Responsibility
DeFi strategy definition, configuration management, play lifecycle, and user play instantiation.

### MongoDB Schema Design

```typescript
// plays collection
interface PlayDocument {
  _id: ObjectId;
  name: string;
  description: string;
  type: 'leverage' | 'staking' | 'liquidity' | 'farming';
  status: 'active' | 'inactive' | 'coming_soon';
  configSchema: {
    [fieldName: string]: {
      type: 'number' | 'string' | 'boolean' | 'select';
      min?: number;
      max?: number;
      options?: string[];
      default?: any;
      label: string;
      description?: string;
      required: boolean;
    };
  };
  icon?: string;
  color?: string;
  displayOrder: number;
  riskRating: 'low' | 'medium' | 'high';
  expectedApy: number;
  minInvestment: number;
  maxInvestment?: number;
  supportedNetworks: string[];
  createdAt: Date;
  updatedAt: Date;
}

// user_plays collection
interface UserPlayDocument {
  _id: ObjectId;
  userId: ObjectId;
  playId: ObjectId;
  status: 'active' | 'paused' | 'completed' | 'failed';
  configuration: Record<string, any>; // User-specific config values
  allocation: {
    percentage: number; // Percentage of portfolio
    usdAmount: number;
    lastUpdated: Date;
  };
  performance: {
    initialInvestment: number;
    currentValue: number;
    totalReturn: number;
    returnPercentage: number;
    lastCalculated: Date;
  };
  createdAt: Date;
  updatedAt: Date;
}
```

### API Contract

```typescript
// Play management endpoints
GET /plays
GET /plays/{playId}
POST /plays // Admin only
PUT /plays/{playId} // Admin only
DELETE /plays/{playId} // Admin only

// User play endpoints
GET /user-plays/user/{userId}
POST /user-plays
PUT /user-plays/{userPlayId}
DELETE /user-plays/{userPlayId}
PUT /user-plays/{userPlayId}/allocation
GET /user-plays/{userPlayId}/performance
```

### Service Interface

```typescript
interface IPlayService {
  // Play management
  getAllPlays(includeInactive?: boolean): Promise<PlayDocument[]>;
  getPlayById(playId: string): Promise<PlayDocument>;
  createPlay(playData: Partial<PlayDocument>): Promise<PlayDocument>;
  updatePlay(playId: string, updates: Partial<PlayDocument>): Promise<PlayDocument>;
  
  // User play management
  createUserPlay(userId: string, playId: string, config: Record<string, any>): Promise<UserPlayDocument>;
  getUserPlays(userId: string): Promise<UserPlayDocument[]>;
  updateUserPlayAllocation(userPlayId: string, percentage: number): Promise<UserPlayDocument>;
  calculateUserPlayPerformance(userPlayId: string): Promise<PerformanceMetrics>;
  
  // Configuration validation
  validatePlayConfiguration(playId: string, config: Record<string, any>): Promise<ValidationResult>;
}
```

---

## 4. Task Service

### Domain Responsibility
Workflow orchestration, task creation and assignment, step execution tracking, and admin task management.

### MongoDB Schema Design

```typescript
// play_tasks collection (task templates)
interface PlayTaskDocument {
  _id: ObjectId;
  playId?: ObjectId; // null for global tasks
  name: string;
  description: string;
  type: 'manual' | 'automated' | 'approval';
  frequency: 'once' | 'daily' | 'weekly' | 'monthly' | 'on_demand';
  priority: 'low' | 'medium' | 'high' | 'critical';
  estimatedDuration: number; // in minutes
  assigneeRole: 'admin' | 'super_admin' | 'system';
  steps: {
    id: string;
    name: string;
    description: string;
    type: 'form' | 'verification' | 'approval' | 'transaction';
    required: boolean;
    orderIndex: number;
    config: Record<string, any>;
  }[];
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

// user_play_tasks collection (task instances)
interface UserPlayTaskDocument {
  _id: ObjectId;
  userId: ObjectId;
  userPlayId: ObjectId;
  taskId: ObjectId; // Reference to PlayTaskDocument
  assignedTo?: ObjectId; // Admin user ID
  status: 'pending' | 'in_progress' | 'completed' | 'failed' | 'cancelled';
  priority: 'low' | 'medium' | 'high' | 'critical';
  data: Record<string, any>; // Task-specific data
  steps: {
    stepId: string;
    status: 'pending' | 'in_progress' | 'completed' | 'failed';
    completedAt?: Date;
    completedBy?: ObjectId;
    data?: Record<string, any>;
    notes?: string;
  }[];
  scheduledFor?: Date;
  startedAt?: Date;
  completedAt?: Date;
  dueDate?: Date;
  notes: string;
  comments: {
    id: string;
    authorId: ObjectId;
    text: string;
    createdAt: Date;
  }[];
  createdAt: Date;
  updatedAt: Date;
}
```

### API Contract

```typescript
// Task template management
GET /tasks/templates
POST /tasks/templates // Admin only
PUT /tasks/templates/{taskId} // Admin only
DELETE /tasks/templates/{taskId} // Admin only

// User task instances
GET /tasks/user/{userId}
GET /tasks/{taskId}
PUT /tasks/{taskId}/assign // Admin only
PUT /tasks/{taskId}/status
PUT /tasks/{taskId}/steps/{stepId}
POST /tasks/{taskId}/comments
DELETE /tasks/{taskId}/comments/{commentId}
```

### Service Interface

```typescript
interface ITaskService {
  // Template management
  createTaskTemplate(taskData: Partial<PlayTaskDocument>): Promise<PlayTaskDocument>;
  getTaskTemplates(playId?: string): Promise<PlayTaskDocument[]>;
  updateTaskTemplate(taskId: string, updates: Partial<PlayTaskDocument>): Promise<PlayTaskDocument>;
  
  // Task instance management
  createUserPlayTask(userId: string, userPlayId: string, taskId: string): Promise<UserPlayTaskDocument>;
  getUserTasks(userId: string, filters?: TaskFilters): Promise<UserPlayTaskDocument[]>;
  assignTask(taskId: string, assigneeId: string): Promise<UserPlayTaskDocument>;
  updateTaskStatus(taskId: string, status: TaskStatus): Promise<UserPlayTaskDocument>;
  updateTaskStep(taskId: string, stepId: string, data: StepUpdateData): Promise<UserPlayTaskDocument>;
  
  // Comments and collaboration
  addTaskComment(taskId: string, authorId: string, text: string): Promise<UserPlayTaskDocument>;
  deleteTaskComment(taskId: string, commentId: string): Promise<UserPlayTaskDocument>;
}
```

---

## 5. Portfolio Service

### Domain Responsibility
Investment tracking, allocation management, performance calculation, and balance aggregation.

### MongoDB Schema Design

```typescript
// user_portfolios collection
interface UserPortfolioDocument {
  _id: ObjectId;
  userId: ObjectId;
  totalBalance: number; // USD
  totalInvested: number; // USD
  totalReturn: number; // USD
  returnPercentage: number;
  allocations: {
    playId: ObjectId;
    playName: string;
    percentage: number;
    usdAmount: number;
    performance: {
      invested: number;
      currentValue: number;
      returnAmount: number;
      returnPercentage: number;
    };
  }[];
  riskMetrics: {
    healthRating: number; // 1.0 - 3.0
    diversificationScore: number; // 0-100
    volatilityRating: 'low' | 'medium' | 'high';
  };
  lastCalculated: Date;
  createdAt: Date;
  updatedAt: Date;
}

// portfolio_transactions collection
interface PortfolioTransactionDocument {
  _id: ObjectId;
  userId: ObjectId;
  playId: ObjectId;
  type: 'deposit' | 'withdrawal' | 'rebalance' | 'profit' | 'loss';
  amount: number; // USD
  tokenAmount?: number;
  tokenSymbol?: string;
  transactionHash?: string;
  network?: string;
  metadata: Record<string, any>;
  timestamp: Date;
}
```

### API Contract

```typescript
// Portfolio overview
GET /portfolio/user/{userId}
GET /portfolio/user/{userId}/performance
GET /portfolio/user/{userId}/allocations
PUT /portfolio/user/{userId}/rebalance

// Transaction tracking
GET /portfolio/user/{userId}/transactions
POST /portfolio/transactions
GET /portfolio/analytics/performance
```

### Service Interface

```typescript
interface IPortfolioService {
  // Portfolio management
  getUserPortfolio(userId: string): Promise<UserPortfolioDocument>;
  updatePortfolioBalance(userId: string): Promise<UserPortfolioDocument>;
  rebalancePortfolio(userId: string, allocations: AllocationData[]): Promise<UserPortfolioDocument>;
  
  // Performance tracking
  calculatePerformanceMetrics(userId: string): Promise<PerformanceMetrics>;
  getPortfolioHistory(userId: string, timeframe: string): Promise<PortfolioSnapshot[]>;
  
  // Transaction management
  recordTransaction(transaction: Partial<PortfolioTransactionDocument>): Promise<PortfolioTransactionDocument>;
  getTransactionHistory(userId: string, filters?: TransactionFilters): Promise<PortfolioTransactionDocument[]>;
  
  // Risk assessment
  calculateRiskMetrics(userId: string): Promise<RiskMetrics>;
  getHealthRating(userId: string): Promise<number>;
}
```

---

## 6. Notification Service

### Domain Responsibility
Real-time messaging, event broadcasting, WebSocket management, and notification delivery.

### MongoDB Schema Design

```typescript
// notifications collection
interface NotificationDocument {
  _id: ObjectId;
  userId: ObjectId;
  type: 'task_update' | 'portfolio_alert' | 'system_message' | 'transaction_status';
  title: string;
  message: string;
  data: Record<string, any>; // Additional notification data
  isRead: boolean;
  readAt?: Date;
  priority: 'low' | 'medium' | 'high';
  expiresAt?: Date;
  createdAt: Date;
}

// notification_preferences collection
interface NotificationPreferenceDocument {
  _id: ObjectId;
  userId: ObjectId;
  preferences: {
    email: boolean;
    push: boolean;
    inApp: boolean;
    sms?: boolean;
  };
  categories: {
    [category: string]: {
      enabled: boolean;
      channels: string[];
    };
  };
  updatedAt: Date;
}
```

### API Contract

```typescript
// Notification management
GET /notifications/user/{userId}
PUT /notifications/{notificationId}/read
POST /notifications/broadcast // Admin only
DELETE /notifications/{notificationId}

// WebSocket endpoints
WS /ws/connect
WS /ws/subscribe/{channel}
WS /ws/unsubscribe/{channel}

// Preferences
GET /notifications/preferences/{userId}
PUT /notifications/preferences/{userId}
```

### Service Interface

```typescript
interface INotificationService {
  // Notification management
  sendNotification(userId: string, notification: NotificationData): Promise<NotificationDocument>;
  getUserNotifications(userId: string, filters?: NotificationFilters): Promise<NotificationDocument[]>;
  markAsRead(notificationId: string): Promise<NotificationDocument>;
  
  // Real-time communication
  connectWebSocket(userId: string, connectionId: string): Promise<void>;
  disconnectWebSocket(connectionId: string): Promise<void>;
  broadcastToUser(userId: string, message: any): Promise<void>;
  broadcastToChannel(channel: string, message: any): Promise<void>;
  
  // Preferences
  updateNotificationPreferences(userId: string, preferences: NotificationPreferenceDocument): Promise<NotificationPreferenceDocument>;
  getNotificationPreferences(userId: string): Promise<NotificationPreferenceDocument>;
}
```

## Inter-Service Communication

### Event-Driven Patterns

```typescript
// Event types for cross-service communication
interface ServiceEvents {
  'user.created': { userId: string; userData: UserDocument };
  'user.updated': { userId: string; changes: Partial<UserDocument> };
  'wallet.created': { walletId: string; userId: string; network: string };
  'wallet.balance.updated': { walletId: string; balance: number };
  'play.created': { playId: string; userId: string };
  'task.assigned': { taskId: string; assigneeId: string; userId: string };
  'task.completed': { taskId: string; userId: string; result: any };
  'portfolio.updated': { userId: string; newBalance: number };
}
```

### Data Consistency Strategies

1. **Event Sourcing**: Store all state changes as events
2. **Saga Pattern**: Manage distributed transactions
3. **Eventual Consistency**: Accept temporary inconsistency
4. **Compensation Actions**: Rollback failed operations

This design provides clear service boundaries while maintaining data integrity and enabling independent development and deployment of each service.