# Lucent Technical Standards & Implementation Guidelines

## Overview

This document defines the technical standards, patterns, and implementation guidelines for the Lucent DeFi platform, covering NestJS backend architecture, React/NextJS frontend patterns, MongoDB integration, and development best practices.

## Backend Architecture - NestJS Standards

### Project Structure Pattern

```
src/
├── common/                 # Shared utilities and decorators
│   ├── decorators/        # Custom decorators
│   ├── filters/           # Exception filters
│   ├── guards/            # Auth guards
│   ├── interceptors/      # Request/Response interceptors
│   ├── pipes/             # Validation pipes
│   └── utils/             # Utility functions
├── config/                 # Configuration management
│   ├── database.config.ts
│   ├── auth.config.ts
│   └── app.config.ts
├── modules/                # Feature modules
│   ├── user/              # User module
│   │   ├── controllers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── dto/
│   │   ├── schemas/
│   │   └── user.module.ts
│   ├── wallet/            # Wallet module
│   ├── play/              # Play module
│   ├── task/              # Task module
│   ├── portfolio/         # Portfolio module
│   └── notification/      # Notification module
├── shared/                 # Shared business logic
│   ├── interfaces/
│   ├── enums/
│   ├── types/
│   └── constants/
└── main.ts                # Application entry point
```

### Module Design Pattern

```typescript
// user.module.ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { UserController } from './controllers/user.controller';
import { UserService } from './services/user.service';
import { UserRepository } from './repositories/user.repository';
import { User, UserSchema } from './schemas/user.schema';
import { AuthModule } from '../auth/auth.module';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: User.name, schema: UserSchema }
    ]),
    AuthModule
  ],
  controllers: [UserController],
  providers: [
    UserService,
    UserRepository,
    {
      provide: 'IUserService',
      useClass: UserService
    }
  ],
  exports: [UserService, 'IUserService']
})
export class UserModule {}
```

### Controller Standards

```typescript
// user.controller.ts
import { 
  Controller, 
  Get, 
  Post, 
  Put, 
  Delete, 
  Body, 
  Param, 
  Query,
  UseGuards,
  UseInterceptors
} from '@nestjs/common';
import { 
  ApiTags, 
  ApiOperation, 
  ApiResponse, 
  ApiBearerAuth 
} from '@nestjs/swagger';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard';
import { RoleGuard } from '../../common/guards/role.guard';
import { LoggingInterceptor } from '../../common/interceptors/logging.interceptor';
import { UserService } from '../services/user.service';
import { CreateUserDto, UpdateUserDto, UserResponseDto } from '../dto/user.dto';
import { Roles } from '../../common/decorators/roles.decorator';
import { CurrentUser } from '../../common/decorators/current-user.decorator';

@ApiTags('users')
@Controller('api/v1/users')
@UseGuards(JwtAuthGuard)
@UseInterceptors(LoggingInterceptor)
export class UserController {
  constructor(
    private readonly userService: UserService
  ) {}

  @Get('profile')
  @ApiOperation({ summary: 'Get current user profile' })
  @ApiResponse({ status: 200, type: UserResponseDto })
  @ApiBearerAuth()
  async getProfile(@CurrentUser() userId: string): Promise<UserResponseDto> {
    return await this.userService.getUserProfile(userId);
  }

  @Put('profile')
  @ApiOperation({ summary: 'Update user profile' })
  @ApiResponse({ status: 200, type: UserResponseDto })
  @ApiBearerAuth()
  async updateProfile(
    @CurrentUser() userId: string,
    @Body() updateDto: UpdateUserDto
  ): Promise<UserResponseDto> {
    return await this.userService.updateUserProfile(userId, updateDto);
  }

  @Get()
  @UseGuards(RoleGuard)
  @Roles('admin', 'super_admin')
  @ApiOperation({ summary: 'Get all users (admin only)' })
  @ApiBearerAuth()
  async getAllUsers(
    @Query() query: PaginationQueryDto
  ): Promise<PaginatedResult<UserResponseDto>> {
    return await this.userService.getAllUsers(query);
  }
}
```

### Service Layer Pattern

```typescript
// user.service.ts
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { UserRepository } from '../repositories/user.repository';
import { CreateUserDto, UpdateUserDto, UserResponseDto } from '../dto/user.dto';
import { User } from '../schemas/user.schema';
import { IUserService } from '../interfaces/user-service.interface';

@Injectable()
export class UserService implements IUserService {
  constructor(
    private readonly userRepository: UserRepository
  ) {}

  async createUser(createUserDto: CreateUserDto): Promise<UserResponseDto> {
    // Check if user already exists
    const existingUser = await this.userRepository.findByWeb3AuthId(
      createUserDto.web3AuthId
    );
    
    if (existingUser) {
      throw new ConflictException('User already exists');
    }

    // Apply business logic
    const userData = this.prepareUserData(createUserDto);
    
    // Create user
    const user = await this.userRepository.create(userData);
    
    // Return DTO
    return this.toResponseDto(user);
  }

  async getUserProfile(userId: string): Promise<UserResponseDto> {
    const user = await this.userRepository.findById(userId);
    
    if (!user) {
      throw new NotFoundException('User not found');
    }

    return this.toResponseDto(user);
  }

  private prepareUserData(dto: CreateUserDto): Partial<User> {
    return {
      web3AuthId: dto.web3AuthId,
      email: dto.email,
      name: dto.name,
      role: 'user',
      isActive: true,
      playSettings: {
        leverage: 0,
        riskTolerance: 'medium',
        autoRebalance: false
      }
    };
  }

  private toResponseDto(user: User): UserResponseDto {
    return {
      id: user._id.toString(),
      web3AuthId: user.web3AuthId,
      email: user.email,
      name: user.name,
      role: user.role,
      isActive: user.isActive,
      createdAt: user.createdAt
    };
  }
}
```

### Repository Pattern

```typescript
// user.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model, Types } from 'mongoose';
import { User, UserDocument } from '../schemas/user.schema';
import { BaseRepository } from '../../shared/repositories/base.repository';

@Injectable()
export class UserRepository extends BaseRepository<UserDocument> {
  constructor(
    @InjectModel(User.name) private userModel: Model<UserDocument>
  ) {
    super(userModel);
  }

  async findByWeb3AuthId(web3AuthId: string): Promise<UserDocument | null> {
    return await this.userModel.findOne({ web3AuthId }).exec();
  }

  async findByEmail(email: string): Promise<UserDocument | null> {
    return await this.userModel.findOne({ email }).exec();
  }

  async findActiveUsers(limit?: number): Promise<UserDocument[]> {
    const query = this.userModel.find({ isActive: true });
    
    if (limit) {
      query.limit(limit);
    }
    
    return await query.exec();
  }

  async updateLastLogin(userId: string): Promise<void> {
    await this.userModel.updateOne(
      { _id: new Types.ObjectId(userId) },
      { lastLoginAt: new Date() }
    ).exec();
  }
}

// Base repository for common operations
export abstract class BaseRepository<T extends Document> {
  constructor(private readonly model: Model<T>) {}

  async create(data: Partial<T>): Promise<T> {
    return await new this.model(data).save();
  }

  async findById(id: string): Promise<T | null> {
    return await this.model.findById(id).exec();
  }

  async findAll(filters: any = {}): Promise<T[]> {
    return await this.model.find(filters).exec();
  }

  async update(id: string, data: Partial<T>): Promise<T | null> {
    return await this.model.findByIdAndUpdate(id, data, { new: true }).exec();
  }

  async delete(id: string): Promise<boolean> {
    const result = await this.model.findByIdAndDelete(id).exec();
    return !!result;
  }

  async count(filters: any = {}): Promise<number> {
    return await this.model.countDocuments(filters).exec();
  }
}
```

### MongoDB Schema Standards

```typescript
// user.schema.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document, Types } from 'mongoose';

export type UserDocument = User & Document;

@Schema({ 
  timestamps: true,
  collection: 'users',
  toJSON: {
    transform: (doc, ret) => {
      ret.id = ret._id;
      delete ret._id;
      delete ret.__v;
      delete ret.bannedAt;
      delete ret.banReason;
      return ret;
    }
  }
})
export class User {
  @Prop({ required: true, unique: true, index: true })
  web3AuthId: string;

  @Prop({ required: false, index: true })
  email?: string;

  @Prop({ required: false })
  name?: string;

  @Prop({ required: false })
  profileImage?: string;

  @Prop({ default: 'web3auth' })
  authProvider: string;

  @Prop({ 
    enum: ['user', 'admin', 'super_admin'], 
    default: 'user',
    index: true 
  })
  role: string;

  @Prop({ default: true, index: true })
  isActive: boolean;

  @Prop({ default: false })
  isEmailVerified: boolean;

  @Prop({ type: [String], default: [] })
  permissions: string[];

  @Prop({ type: Object, default: {} })
  metadata: Record<string, any>;

  @Prop({ 
    type: Object, 
    default: () => ({ leverage: 0, riskTolerance: 'medium', autoRebalance: false })
  })
  playSettings: {
    leverage: number;
    riskTolerance: 'low' | 'medium' | 'high';
    autoRebalance: boolean;
  };

  @Prop({ type: Number, default: 0, index: true })
  portfolioBalance: number;

  @Prop({ required: false })
  lastLoginAt?: Date;

  @Prop({ required: false })
  bannedAt?: Date;

  @Prop({ required: false })
  banReason?: string;

  createdAt: Date;
  updatedAt: Date;
}

export const UserSchema = SchemaFactory.createForClass(User);

// Add indexes for performance
UserSchema.index({ web3AuthId: 1 });
UserSchema.index({ email: 1 });
UserSchema.index({ role: 1, isActive: 1 });
UserSchema.index({ createdAt: -1 });
```

### DTO Standards

```typescript
// user.dto.ts
import { 
  IsString, 
  IsEmail, 
  IsOptional, 
  IsBoolean, 
  IsEnum, 
  IsNumber,
  ValidateNested,
  Min,
  Max
} from 'class-validator';
import { Type } from 'class-transformer';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ description: 'Web3Auth unique identifier' })
  @IsString()
  web3AuthId: string;

  @ApiPropertyOptional({ description: 'User email address' })
  @IsOptional()
  @IsEmail()
  email?: string;

  @ApiPropertyOptional({ description: 'User display name' })
  @IsOptional()
  @IsString()
  name?: string;

  @ApiPropertyOptional({ description: 'Profile image URL' })
  @IsOptional()
  @IsString()
  profileImage?: string;
}

export class UpdateUserDto {
  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  name?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsEmail()
  email?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  profileImage?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @ValidateNested()
  @Type(() => UserPlaySettingsDto)
  playSettings?: UserPlaySettingsDto;
}

export class UserPlaySettingsDto {
  @ApiProperty({ minimum: 0, maximum: 75 })
  @IsNumber()
  @Min(0)
  @Max(75)
  leverage: number;

  @ApiProperty({ enum: ['low', 'medium', 'high'] })
  @IsEnum(['low', 'medium', 'high'])
  riskTolerance: 'low' | 'medium' | 'high';

  @ApiProperty()
  @IsBoolean()
  autoRebalance: boolean;
}

export class UserResponseDto {
  @ApiProperty()
  id: string;

  @ApiProperty()
  web3AuthId: string;

  @ApiPropertyOptional()
  email?: string;

  @ApiPropertyOptional()
  name?: string;

  @ApiProperty()
  role: string;

  @ApiProperty()
  isActive: boolean;

  @ApiProperty()
  createdAt: Date;
}
```

## Frontend Architecture - React/NextJS Standards

### Project Structure Pattern

```
src/
├── app/                    # Next.js 14+ app router
│   ├── (auth)/            # Route groups
│   │   ├── login/
│   │   └── register/
│   ├── dashboard/
│   ├── plays/
│   ├── portfolio/
│   ├── layout.tsx
│   ├── page.tsx
│   └── globals.css
├── components/             # Reusable UI components
│   ├── ui/                # Base UI components (shadcn/ui)
│   ├── forms/             # Form components
│   ├── charts/            # Chart components
│   └── layout/            # Layout components
├── hooks/                  # Custom React hooks
├── lib/                    # Utility libraries and configurations
│   ├── api.ts             # API client configuration
│   ├── auth.ts            # Authentication utilities
│   ├── utils.ts           # General utilities
│   └── validations.ts     # Zod validation schemas
├── store/                  # State management (Zustand/Redux Toolkit)
│   ├── slices/
│   └── index.ts
├── types/                  # TypeScript type definitions
└── constants/              # Application constants
```

### Component Standards

```typescript
// components/ui/Button.tsx
import React from 'react';
import { cn } from '@/lib/utils';
import { cva, type VariantProps } from 'class-variance-authority';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground shadow hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground shadow-sm hover:bg-destructive/90',
        outline: 'border border-input bg-background shadow-sm hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground shadow-sm hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline'
      },
      size: {
        default: 'h-9 px-4 py-2',
        sm: 'h-8 rounded-md px-3 text-xs',
        lg: 'h-10 rounded-md px-8',
        icon: 'h-9 w-9'
      }
    },
    defaultVariants: {
      variant: 'default',
      size: 'default'
    }
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    return (
      <button
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    );
  }
);
Button.displayName = 'Button';

export { Button, buttonVariants };
```

### State Management Pattern (Zustand)

```typescript
// store/slices/userSlice.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { api } from '@/lib/api';

interface User {
  id: string;
  web3AuthId: string;
  email?: string;
  name?: string;
  role: string;
  isActive: boolean;
}

interface UserState {
  user: User | null;
  isLoading: boolean;
  error: string | null;
  
  // Actions
  setUser: (user: User) => void;
  fetchUser: () => Promise<void>;
  updateProfile: (updates: Partial<User>) => Promise<void>;
  logout: () => void;
}

export const useUserStore = create<UserState>()(
  devtools(
    persist(
      (set, get) => ({
        user: null,
        isLoading: false,
        error: null,

        setUser: (user) => set({ user, error: null }),

        fetchUser: async () => {
          set({ isLoading: true, error: null });
          
          try {
            const response = await api.get('/users/profile');
            set({ user: response.data, isLoading: false });
          } catch (error) {
            set({ 
              error: error instanceof Error ? error.message : 'Failed to fetch user',
              isLoading: false 
            });
          }
        },

        updateProfile: async (updates) => {
          const { user } = get();
          if (!user) return;

          set({ isLoading: true, error: null });
          
          try {
            const response = await api.put('/users/profile', updates);
            set({ user: response.data, isLoading: false });
          } catch (error) {
            set({ 
              error: error instanceof Error ? error.message : 'Failed to update profile',
              isLoading: false 
            });
            throw error;
          }
        },

        logout: () => {
          set({ user: null, error: null });
          // Clear persisted state
          localStorage.removeItem('user-store');
        }
      }),
      {
        name: 'user-store',
        partialize: (state) => ({ user: state.user })
      }
    ),
    { name: 'user-store' }
  )
);
```

### Custom Hooks Pattern

```typescript
// hooks/useAuth.ts
import { useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { useUserStore } from '@/store/slices/userSlice';
import { Web3Auth } from '@web3auth/modal';

export const useAuth = () => {
  const router = useRouter();
  const { user, isLoading, setUser, logout: logoutUser } = useUserStore();

  const login = async (web3auth: Web3Auth) => {
    try {
      await web3auth.connect();
      const userInfo = await web3auth.getUserInfo();
      
      // Call backend to authenticate
      const response = await api.post('/auth/login', {
        web3AuthId: userInfo.sub,
        email: userInfo.email,
        name: userInfo.name
      });

      setUser(response.data.user);
      router.push('/dashboard');
    } catch (error) {
      console.error('Login failed:', error);
      throw error;
    }
  };

  const logout = async () => {
    logoutUser();
    router.push('/');
  };

  const requireAuth = () => {
    useEffect(() => {
      if (!isLoading && !user) {
        router.push('/login');
      }
    }, [user, isLoading]);
  };

  return {
    user,
    isLoading,
    isAuthenticated: !!user,
    login,
    logout,
    requireAuth
  };
};
```

### API Client Configuration

```typescript
// lib/api.ts
import axios, { AxiosResponse, AxiosError } from 'axios';
import { useUserStore } from '@/store/slices/userSlice';

const baseURL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001/api/v1';

export const api = axios.create({
  baseURL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Request interceptor to add auth token
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('auth-token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor for error handling
api.interceptors.response.use(
  (response: AxiosResponse) => response,
  (error: AxiosError) => {
    if (error.response?.status === 401) {
      // Clear auth state and redirect to login
      useUserStore.getState().logout();
      window.location.href = '/login';
    }
    
    return Promise.reject(error);
  }
);

// API service classes
export class UserAPI {
  static async getProfile() {
    const response = await api.get('/users/profile');
    return response.data;
  }

  static async updateProfile(updates: any) {
    const response = await api.put('/users/profile', updates);
    return response.data;
  }
}

export class WalletAPI {
  static async getWallets() {
    const response = await api.get('/wallets');
    return response.data;
  }

  static async generateWallet(network: string) {
    const response = await api.post('/wallets/generate', { network });
    return response.data;
  }
}
```

## MongoDB Integration Standards

### Connection Configuration

```typescript
// config/database.config.ts
import { ConfigService } from '@nestjs/config';
import { MongooseModuleOptions } from '@nestjs/mongoose';

export const getDatabaseConfig = (configService: ConfigService): MongooseModuleOptions => ({
  uri: configService.get<string>('DATABASE_URL'),
  retryWrites: true,
  retryReads: true,
  maxPoolSize: 10,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
  bufferMaxEntries: 0,
  bufferCommands: false,
  heartbeatFrequencyMS: 10000,
  // Connection options for production
  ...(process.env.NODE_ENV === 'production' && {
    ssl: true,
    sslValidate: true,
    authSource: 'admin'
  })
});
```

### Indexing Strategy

```typescript
// Database indexes for performance
export const createIndexes = async () => {
  // User indexes
  await userCollection.createIndex({ web3AuthId: 1 }, { unique: true });
  await userCollection.createIndex({ email: 1 }, { sparse: true });
  await userCollection.createIndex({ role: 1, isActive: 1 });
  await userCollection.createIndex({ createdAt: -1 });

  // Wallet indexes
  await walletCollection.createIndex({ userId: 1, network: 1 });
  await walletCollection.createIndex({ address: 1 }, { unique: true });
  await walletCollection.createIndex({ isActive: 1, network: 1 });

  // Task indexes
  await taskCollection.createIndex({ userId: 1, status: 1 });
  await taskCollection.createIndex({ assignedTo: 1, status: 1 });
  await taskCollection.createIndex({ scheduledFor: 1 });
  await taskCollection.createIndex({ createdAt: -1 });

  // Portfolio indexes
  await portfolioCollection.createIndex({ userId: 1 }, { unique: true });
  await portfolioCollection.createIndex({ 'allocations.playId': 1 });
  await portfolioCollection.createIndex({ lastCalculated: -1 });
};
```

### Transaction Patterns

```typescript
// service/transaction.service.ts
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection, ClientSession } from 'mongoose';

@Injectable()
export class TransactionService {
  constructor(
    @InjectConnection() private connection: Connection
  ) {}

  async executeTransaction<T>(
    operations: (session: ClientSession) => Promise<T>
  ): Promise<T> {
    const session = await this.connection.startSession();
    
    try {
      session.startTransaction();
      
      const result = await operations(session);
      
      await session.commitTransaction();
      return result;
      
    } catch (error) {
      await session.abortTransaction();
      throw error;
      
    } finally {
      await session.endSession();
    }
  }

  // Example usage in service
  async createUserWithWallet(userData: any, walletData: any) {
    return await this.executeTransaction(async (session) => {
      // Create user
      const user = await this.userRepository.create(userData, { session });
      
      // Create wallet
      const wallet = await this.walletRepository.create(
        { ...walletData, userId: user._id },
        { session }
      );
      
      return { user, wallet };
    });
  }
}
```

This technical standards document provides the foundation for implementing the Lucent platform with consistent patterns, proper error handling, and maintainable code architecture.