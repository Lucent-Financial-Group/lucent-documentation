# SOLID Principles Application Guidelines for Lucent Platform

## Overview

This document provides concrete examples and implementation guidelines for applying SOLID principles throughout the Lucent DeFi platform. Each principle is explained with real-world examples from the financial crypto domain, showing both problematic and corrected implementations.

## 1. Single Responsibility Principle (SRP)

**Definition**: A class should have one, and only one, reason to change. Each class should have a single responsibility or concern.

### ❌ Violation Example

```typescript
// BAD: UserService handling multiple responsibilities
class UserService {
  // User management responsibility
  async createUser(userData: CreateUserDto): Promise<User> {
    const user = new User(userData);
    return await this.userRepository.save(user);
  }

  // Email responsibility (should be separate)
  async sendWelcomeEmail(user: User): Promise<void> {
    const emailContent = `Welcome ${user.name}!`;
    await this.emailProvider.send(user.email, 'Welcome', emailContent);
  }

  // Wallet generation responsibility (should be separate)
  async generateWalletForUser(userId: string, network: string): Promise<Wallet> {
    const privateKey = this.cryptoUtils.generatePrivateKey();
    const publicKey = this.cryptoUtils.derivePublicKey(privateKey);
    const address = this.cryptoUtils.deriveAddress(publicKey, network);
    
    return await this.walletRepository.save({
      userId,
      network,
      address,
      publicKey,
      privateKey
    });
  }

  // Portfolio calculation responsibility (should be separate)
  async calculatePortfolioBalance(userId: string): Promise<number> {
    const wallets = await this.walletRepository.findByUserId(userId);
    let totalBalance = 0;
    
    for (const wallet of wallets) {
      const balance = await this.blockchainService.getBalance(wallet.address);
      totalBalance += balance * await this.priceService.getUSDPrice(wallet.network);
    }
    
    return totalBalance;
  }
}
```

### ✅ Corrected Implementation

```typescript
// GOOD: Separate services with single responsibilities

// User management only
@Injectable()
class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly eventBus: EventBus
  ) {}

  async createUser(userData: CreateUserDto): Promise<User> {
    const user = await this.userRepository.create(userData);
    
    // Emit event for other services to handle their responsibilities
    await this.eventBus.emit('user.created', { userId: user.id, userData });
    
    return user;
  }

  async updateUser(userId: string, updates: UpdateUserDto): Promise<User> {
    return await this.userRepository.update(userId, updates);
  }
}

// Email management only
@Injectable()
class NotificationService {
  constructor(private readonly emailProvider: EmailProvider) {}

  @EventHandler('user.created')
  async handleUserCreated(event: UserCreatedEvent): Promise<void> {
    await this.sendWelcomeEmail(event.userData);
  }

  async sendWelcomeEmail(userData: any): Promise<void> {
    const emailContent = `Welcome ${userData.name}!`;
    await this.emailProvider.send(userData.email, 'Welcome', emailContent);
  }
}

// Wallet operations only
@Injectable()
class WalletService {
  constructor(
    private readonly walletRepository: WalletRepository,
    private readonly keyVaultService: KeyVaultService
  ) {}

  @EventHandler('user.created')
  async handleUserCreated(event: UserCreatedEvent): Promise<void> {
    // Generate wallets for all supported networks
    const networks = ['EVM', 'Solana', 'Bitcoin'];
    for (const network of networks) {
      await this.generateWallet(event.userId, network);
    }
  }

  async generateWallet(userId: string, network: string): Promise<Wallet> {
    const keyPair = this.generateKeyPair(network);
    const address = this.deriveAddress(keyPair.publicKey, network);
    
    // Store private key securely
    const secretName = await this.keyVaultService.storeSecret(keyPair.privateKey);
    
    return await this.walletRepository.create({
      userId,
      network,
      address,
      publicKey: keyPair.publicKey,
      privateKeySecretName: secretName
    });
  }
}

// Portfolio calculations only
@Injectable()
class PortfolioService {
  constructor(
    private readonly portfolioRepository: PortfolioRepository,
    private readonly walletService: WalletService,
    private readonly priceService: PriceService
  ) {}

  async calculateUserPortfolio(userId: string): Promise<Portfolio> {
    const wallets = await this.walletService.getUserWallets(userId);
    const calculations = await this.calculateTotalBalance(wallets);
    
    return await this.portfolioRepository.updateOrCreate(userId, calculations);
  }
}
```

## 2. Open/Closed Principle (OCP)

**Definition**: Software entities should be open for extension but closed for modification. You should be able to add new functionality without changing existing code.

### ❌ Violation Example

```typescript
// BAD: Hard to extend for new blockchain networks
class WalletGenerator {
  async generateWallet(network: string): Promise<WalletData> {
    if (network === 'EVM') {
      return this.generateEVMWallet();
    } else if (network === 'Solana') {
      return this.generateSolanaWallet();
    } else if (network === 'Bitcoin') {
      return this.generateBitcoinWallet();
    } else {
      throw new Error(`Unsupported network: ${network}`);
    }
  }

  // Adding Cardano would require modifying this class
  private generateEVMWallet(): WalletData { /* ... */ }
  private generateSolanaWallet(): WalletData { /* ... */ }
  private generateBitcoinWallet(): WalletData { /* ... */ }
}
```

### ✅ Corrected Implementation

```typescript
// GOOD: Open for extension, closed for modification

// Abstract interface that defines the contract
interface IBlockchainWalletGenerator {
  readonly network: string;
  generateWallet(): Promise<WalletData>;
  validateAddress(address: string): boolean;
  deriveAddress(publicKey: string): string;
}

// Base implementation
abstract class BaseWalletGenerator implements IBlockchainWalletGenerator {
  abstract readonly network: string;
  abstract generateWallet(): Promise<WalletData>;
  abstract validateAddress(address: string): boolean;
  abstract deriveAddress(publicKey: string): string;

  protected generateEntropy(): Buffer {
    return crypto.randomBytes(32);
  }
}

// Specific implementations - can be added without modifying existing code
@Injectable()
class EVMWalletGenerator extends BaseWalletGenerator {
  readonly network = 'EVM';

  async generateWallet(): Promise<WalletData> {
    const entropy = this.generateEntropy();
    const wallet = ethers.Wallet.fromMnemonic(
      ethers.utils.entropyToMnemonic(entropy)
    );
    
    return {
      network: this.network,
      privateKey: wallet.privateKey,
      publicKey: wallet.publicKey,
      address: wallet.address
    };
  }

  validateAddress(address: string): boolean {
    return ethers.utils.isAddress(address);
  }

  deriveAddress(publicKey: string): string {
    return ethers.utils.computeAddress(publicKey);
  }
}

@Injectable()
class SolanaWalletGenerator extends BaseWalletGenerator {
  readonly network = 'Solana';

  async generateWallet(): Promise<WalletData> {
    const keypair = Keypair.generate();
    
    return {
      network: this.network,
      privateKey: bs58.encode(keypair.secretKey),
      publicKey: keypair.publicKey.toBase58(),
      address: keypair.publicKey.toBase58()
    };
  }

  validateAddress(address: string): boolean {
    try {
      new PublicKey(address);
      return true;
    } catch {
      return false;
    }
  }

  deriveAddress(publicKey: string): string {
    return publicKey; // In Solana, public key is the address
  }
}

// Factory pattern to manage generators
@Injectable()
class WalletGeneratorFactory {
  private generators = new Map<string, IBlockchainWalletGenerator>();

  constructor(
    private readonly evmGenerator: EVMWalletGenerator,
    private readonly solanaGenerator: SolanaWalletGenerator
    // New generators can be injected here without modifying existing code
  ) {
    this.generators.set('EVM', this.evmGenerator);
    this.generators.set('Solana', this.solanaGenerator);
  }

  getGenerator(network: string): IBlockchainWalletGenerator {
    const generator = this.generators.get(network);
    if (!generator) {
      throw new Error(`No generator available for network: ${network}`);
    }
    return generator;
  }

  // Easy to add new networks
  registerGenerator(generator: IBlockchainWalletGenerator): void {
    this.generators.set(generator.network, generator);
  }
}

// Adding Cardano support doesn't require changing existing code
@Injectable()
class CardanoWalletGenerator extends BaseWalletGenerator {
  readonly network = 'Cardano';

  async generateWallet(): Promise<WalletData> {
    // Cardano-specific implementation
    // ...
  }
  
  // ... other methods
}
```

## 3. Liskov Substitution Principle (LSP)

**Definition**: Objects of a superclass should be replaceable with objects of its subclasses without breaking the application. Derived classes must be substitutable for their base classes.

### ❌ Violation Example

```typescript
// BAD: Derived class changes the behavior expectations
abstract class PlayExecutor {
  abstract executePlay(userPlay: UserPlay): Promise<ExecutionResult>;
  
  // Base expectation: all plays should support pausing
  async pausePlay(userPlayId: string): Promise<void> {
    // Default implementation
    await this.updatePlayStatus(userPlayId, 'paused');
  }
}

class LeveragePlayExecutor extends PlayExecutor {
  async executePlay(userPlay: UserPlay): Promise<ExecutionResult> {
    return await this.executeLeverageStrategy(userPlay);
  }
}

class StakingPlayExecutor extends PlayExecutor {
  async executePlay(userPlay: UserPlay): Promise<ExecutionResult> {
    return await this.executeStakingStrategy(userPlay);
  }

  // VIOLATION: Changes the contract by throwing an error
  async pausePlay(userPlayId: string): Promise<void> {
    throw new Error('Staking plays cannot be paused once started');
  }
}

// This will break when using StakingPlayExecutor
class PlayService {
  async pauseUserPlay(userPlayId: string, executor: PlayExecutor): Promise<void> {
    // This will throw an error for StakingPlayExecutor, violating LSP
    await executor.pausePlay(userPlayId);
  }
}
```

### ✅ Corrected Implementation

```typescript
// GOOD: Consistent behavior across all implementations

abstract class PlayExecutor {
  abstract executePlay(userPlay: UserPlay): Promise<ExecutionResult>;
  abstract canBePaused(): boolean;
  
  async pausePlay(userPlayId: string): Promise<PauseResult> {
    if (!this.canBePaused()) {
      return {
        success: false,
        reason: 'This play type cannot be paused'
      };
    }
    
    await this.updatePlayStatus(userPlayId, 'paused');
    return { success: true };
  }

  protected abstract updatePlayStatus(userPlayId: string, status: string): Promise<void>;
}

class LeveragePlayExecutor extends PlayExecutor {
  canBePaused(): boolean {
    return true; // Leverage plays can be paused
  }

  async executePlay(userPlay: UserPlay): Promise<ExecutionResult> {
    return await this.executeLeverageStrategy(userPlay);
  }

  protected async updatePlayStatus(userPlayId: string, status: string): Promise<void> {
    await this.userPlayRepository.update(userPlayId, { status });
  }
}

class StakingPlayExecutor extends PlayExecutor {
  canBePaused(): boolean {
    return false; // Staking plays cannot be paused
  }

  async executePlay(userPlay: UserPlay): Promise<ExecutionResult> {
    return await this.executeStakingStrategy(userPlay);
  }

  protected async updatePlayStatus(userPlayId: string, status: string): Promise<void> {
    // Staking-specific status update logic
    await this.stakingRepository.updateStatus(userPlayId, status);
  }
}

// Now the service works consistently with all executors
class PlayService {
  async pauseUserPlay(userPlayId: string, executor: PlayExecutor): Promise<PauseResult> {
    // This will work consistently for all executors
    return await executor.pausePlay(userPlayId);
  }
}
```

## 4. Interface Segregation Principle (ISP)

**Definition**: No client should be forced to depend on methods it does not use. Many client-specific interfaces are better than one general-purpose interface.

### ❌ Violation Example

```typescript
// BAD: Fat interface forcing unnecessary dependencies
interface IWalletService {
  // Basic wallet operations
  createWallet(userId: string, network: string): Promise<Wallet>;
  getWallet(walletId: string): Promise<Wallet>;
  
  // Balance operations
  getBalance(walletId: string): Promise<number>;
  refreshBalance(walletId: string): Promise<number>;
  
  // Transaction operations
  signTransaction(walletId: string, transaction: any): Promise<string>;
  broadcastTransaction(signedTx: string): Promise<TransactionResult>;
  
  // Admin-only operations
  getPrivateKey(walletId: string): Promise<string>;
  rotateKeys(walletId: string): Promise<Wallet>;
  
  // Analytics operations
  getTransactionHistory(walletId: string): Promise<Transaction[]>;
  generateReport(walletId: string): Promise<Report>;
  
  // Backup operations
  exportWallet(walletId: string): Promise<BackupData>;
  importWallet(backupData: BackupData): Promise<Wallet>;
}

// Regular users are forced to implement admin methods they can't use
class UserWalletService implements IWalletService {
  // Users don't need these admin methods but are forced to implement them
  async getPrivateKey(walletId: string): Promise<string> {
    throw new Error('Unauthorized'); // Always throws!
  }

  async rotateKeys(walletId: string): Promise<Wallet> {
    throw new Error('Unauthorized'); // Always throws!
  }
  
  // ... forced to implement all other methods
}
```

### ✅ Corrected Implementation

```typescript
// GOOD: Segregated interfaces for specific use cases

// Core wallet operations
interface IWalletManager {
  createWallet(userId: string, network: string): Promise<Wallet>;
  getWallet(walletId: string): Promise<Wallet>;
  deleteWallet(walletId: string): Promise<void>;
}

// Balance-related operations
interface IWalletBalanceService {
  getBalance(walletId: string): Promise<number>;
  refreshBalance(walletId: string): Promise<number>;
  getBalanceHistory(walletId: string): Promise<BalanceSnapshot[]>;
}

// Transaction operations
interface IWalletTransactionService {
  signTransaction(walletId: string, transaction: any): Promise<string>;
  broadcastTransaction(signedTx: string): Promise<TransactionResult>;
  getTransactionStatus(txHash: string): Promise<TransactionStatus>;
}

// Admin-only operations
interface IWalletAdminService {
  getPrivateKey(walletId: string): Promise<string>;
  rotateKeys(walletId: string): Promise<Wallet>;
  auditWallet(walletId: string): Promise<AuditResult>;
}

// Analytics operations
interface IWalletAnalyticsService {
  getTransactionHistory(walletId: string): Promise<Transaction[]>;
  generateReport(walletId: string, type: ReportType): Promise<Report>;
  getUsageStatistics(walletId: string): Promise<UsageStats>;
}

// Backup operations
interface IWalletBackupService {
  exportWallet(walletId: string): Promise<BackupData>;
  importWallet(backupData: BackupData): Promise<Wallet>;
  validateBackup(backupData: BackupData): Promise<boolean>;
}

// Implementations only implement what they need
@Injectable()
class UserWalletService implements IWalletManager, IWalletBalanceService, IWalletTransactionService {
  constructor(
    private readonly walletRepository: WalletRepository,
    private readonly blockchainService: BlockchainService
  ) {}

  async createWallet(userId: string, network: string): Promise<Wallet> {
    // Implementation for regular users
  }

  async getBalance(walletId: string): Promise<number> {
    // Implementation for balance checking
  }

  async signTransaction(walletId: string, transaction: any): Promise<string> {
    // Implementation for transaction signing
  }

  // No need to implement admin methods
}

@Injectable()
class AdminWalletService implements IWalletManager, IWalletAdminService, IWalletAnalyticsService {
  constructor(
    private readonly walletRepository: WalletRepository,
    private readonly keyVaultService: KeyVaultService
  ) {}

  async createWallet(userId: string, network: string): Promise<Wallet> {
    // Admin implementation with additional logging
  }

  async getPrivateKey(walletId: string): Promise<string> {
    // Admin-only implementation
    return await this.keyVaultService.getSecret(walletId);
  }

  async generateReport(walletId: string, type: ReportType): Promise<Report> {
    // Analytics implementation
  }

  // No need to implement backup methods if not needed
}
```

## 5. Dependency Inversion Principle (DIP)

**Definition**: High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details; details should depend on abstractions.

### ❌ Violation Example

```typescript
// BAD: High-level service directly depends on low-level implementations
class PlayExecutionService {
  private emailSender: NodemailerEmailSender; // Direct dependency
  private database: MongoDatabase; // Direct dependency
  private priceAPI: CoinGeckoAPI; // Direct dependency

  constructor() {
    // Tightly coupled to specific implementations
    this.emailSender = new NodemailerEmailSender();
    this.database = new MongoDatabase();
    this.priceAPI = new CoinGeckoAPI();
  }

  async executeUserPlay(userPlayId: string): Promise<void> {
    // Directly using concrete classes
    const userPlay = await this.database.findUserPlay(userPlayId);
    const currentPrice = await this.priceAPI.getPrice(userPlay.tokenSymbol);
    
    if (this.shouldExecute(userPlay, currentPrice)) {
      await this.executeStrategy(userPlay);
      
      // Directly calling email service
      await this.emailSender.send(
        userPlay.userEmail,
        'Play Executed',
        `Your ${userPlay.playName} has been executed at $${currentPrice}`
      );
    }
  }
}
```

### ✅ Corrected Implementation

```typescript
// GOOD: Depend on abstractions, not concretions

// Abstractions (interfaces)
interface IEmailService {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
}

interface IPriceService {
  getPrice(symbol: string): Promise<number>;
  getPriceHistory(symbol: string, days: number): Promise<PricePoint[]>;
}

interface IUserPlayRepository {
  findById(id: string): Promise<UserPlay>;
  update(id: string, data: Partial<UserPlay>): Promise<UserPlay>;
}

interface INotificationService {
  notifyUser(userId: string, notification: NotificationData): Promise<void>;
}

// High-level service depends only on abstractions
@Injectable()
class PlayExecutionService {
  constructor(
    private readonly userPlayRepository: IUserPlayRepository,
    private readonly priceService: IPriceService,
    private readonly notificationService: INotificationService,
    private readonly logger: ILogger
  ) {}

  async executeUserPlay(userPlayId: string): Promise<ExecutionResult> {
    try {
      // Using abstractions, not concrete implementations
      const userPlay = await this.userPlayRepository.findById(userPlayId);
      const currentPrice = await this.priceService.getPrice(userPlay.tokenSymbol);
      
      if (this.shouldExecute(userPlay, currentPrice)) {
        const result = await this.executeStrategy(userPlay, currentPrice);
        
        // Using notification abstraction
        await this.notificationService.notifyUser(userPlay.userId, {
          type: 'play_executed',
          title: 'Play Executed Successfully',
          message: `Your ${userPlay.playName} has been executed at $${currentPrice}`,
          data: { userPlayId, executionPrice: currentPrice, result }
        });

        return result;
      }

      return { success: false, reason: 'Execution conditions not met' };
      
    } catch (error) {
      await this.logger.error('Play execution failed', { userPlayId, error });
      throw error;
    }
  }
}

// Concrete implementations of abstractions
@Injectable()
class NodemailerEmailService implements IEmailService {
  constructor(private readonly transporter: any) {}

  async sendEmail(to: string, subject: string, body: string): Promise<void> {
    await this.transporter.sendMail({ to, subject, html: body });
  }
}

@Injectable()
class SendGridEmailService implements IEmailService {
  constructor(private readonly sendGridClient: any) {}

  async sendEmail(to: string, subject: string, body: string): Promise<void> {
    await this.sendGridClient.send({ to, subject, html: body });
  }
}

@Injectable()
class CoinGeckoPriceService implements IPriceService {
  async getPrice(symbol: string): Promise<number> {
    // CoinGecko implementation
  }

  async getPriceHistory(symbol: string, days: number): Promise<PricePoint[]> {
    // CoinGecko implementation
  }
}

@Injectable()
class ChainlinkPriceService implements IPriceService {
  async getPrice(symbol: string): Promise<number> {
    // Chainlink price feed implementation
  }

  async getPriceHistory(symbol: string, days: number): Promise<PricePoint[]> {
    // Chainlink historical data implementation
  }
}

// MongoDB implementation
@Injectable()
class MongoUserPlayRepository implements IUserPlayRepository {
  constructor(
    @InjectModel(UserPlay.name) private userPlayModel: Model<UserPlayDocument>
  ) {}

  async findById(id: string): Promise<UserPlay> {
    return await this.userPlayModel.findById(id).exec();
  }

  async update(id: string, data: Partial<UserPlay>): Promise<UserPlay> {
    return await this.userPlayModel.findByIdAndUpdate(id, data, { new: true }).exec();
  }
}

// Module configuration with dependency injection
@Module({
  imports: [
    MongooseModule.forFeature([
      { name: UserPlay.name, schema: UserPlaySchema }
    ])
  ],
  providers: [
    PlayExecutionService,
    {
      provide: 'IUserPlayRepository',
      useClass: MongoUserPlayRepository
    },
    {
      provide: 'IPriceService',
      useClass: process.env.PRICE_SERVICE === 'chainlink' 
        ? ChainlinkPriceService 
        : CoinGeckoPriceService
    },
    {
      provide: 'IEmailService',
      useClass: process.env.EMAIL_SERVICE === 'sendgrid'
        ? SendGridEmailService
        : NodemailerEmailService
    }
  ]
})
export class PlayModule {}
```

## SOLID Principles Checklist

### ✅ Before Code Review

- [ ] **SRP**: Does each class have a single, well-defined responsibility?
- [ ] **OCP**: Can I add new features without modifying existing code?
- [ ] **LSP**: Can I substitute any derived class for its base class without breaking functionality?
- [ ] **ISP**: Are interfaces focused and not forcing clients to implement unnecessary methods?
- [ ] **DIP**: Do high-level modules depend on abstractions rather than concrete implementations?

### ✅ Code Quality Indicators

- [ ] Classes have clear, single purposes
- [ ] New features are added through extension, not modification
- [ ] Polymorphic code works correctly with all implementations
- [ ] Interfaces are cohesive and client-specific
- [ ] Dependencies are injected and based on interfaces
- [ ] Code is testable and mockable
- [ ] Business logic is separated from infrastructure concerns

## Benefits of Following SOLID Principles

1. **Maintainability**: Easier to understand, modify, and extend code
2. **Testability**: Dependencies can be easily mocked and tested
3. **Flexibility**: New features can be added without breaking existing functionality
4. **Reusability**: Components can be reused in different contexts
5. **Reduced Coupling**: Components are loosely coupled and highly cohesive
6. **Better Team Collaboration**: Clear responsibilities make code ownership easier

By following these SOLID principles, the Lucent platform will be more maintainable, extensible, and robust, enabling rapid development while maintaining code quality.