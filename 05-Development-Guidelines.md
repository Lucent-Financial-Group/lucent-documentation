# Development Guidelines for Lucent Platform

## Overview

This document establishes development standards, code organization patterns, API design guidelines, and best practices for building the Lucent DeFi platform. These guidelines ensure consistency, maintainability, and scalability across the entire development team.

## Code Organization Standards

### Backend Structure (NestJS)

#### Service-Oriented Directory Structure

```
apps/
├── api-gateway/                # Main API Gateway
│   ├── src/
│   │   ├── gateway/           # Gateway configuration
│   │   ├── middleware/        # Global middleware
│   │   └── main.ts
│   └── package.json
│
├── user-service/              # User microservice
│   ├── src/
│   │   ├── domain/           # Domain layer
│   │   │   ├── entities/     # Business entities
│   │   │   ├── value-objects/ # Value objects
│   │   │   └── repositories/ # Repository interfaces
│   │   ├── application/      # Application layer
│   │   │   ├── services/     # Application services
│   │   │   ├── dto/          # Data transfer objects
│   │   │   ├── commands/     # CQRS commands
│   │   │   ├── queries/      # CQRS queries
│   │   │   └── handlers/     # Command/Query handlers
│   │   ├── infrastructure/   # Infrastructure layer
│   │   │   ├── database/     # Database implementations
│   │   │   ├── external/     # External service clients
│   │   │   └── messaging/    # Event bus implementations
│   │   ├── presentation/     # Presentation layer
│   │   │   ├── controllers/  # HTTP controllers
│   │   │   ├── graphql/      # GraphQL resolvers
│   │   │   └── websocket/    # WebSocket handlers
│   │   └── shared/           # Shared utilities
│   └── package.json
│
└── shared/                    # Shared libraries
    ├── common/               # Common utilities
    ├── events/               # Event definitions
    ├── types/                # Shared TypeScript types
    └── decorators/           # Custom decorators
```

### Frontend Structure (React/NextJS)

```
src/
├── app/                      # Next.js App Router
│   ├── (auth)/              # Route groups
│   ├── (dashboard)/         # Protected routes
│   ├── api/                 # API routes
│   ├── globals.css          # Global styles
│   ├── layout.tsx           # Root layout
│   └── page.tsx             # Home page
│
├── components/               # Reusable components
│   ├── ui/                  # Base UI components (shadcn/ui)
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   ├── card.tsx
│   │   └── index.ts         # Barrel exports
│   ├── forms/               # Form-specific components
│   │   ├── LoginForm.tsx
│   │   ├── UserProfileForm.tsx
│   │   └── PlayConfigForm.tsx
│   ├── charts/              # Chart components
│   │   ├── PortfolioChart.tsx
│   │   ├── PriceChart.tsx
│   │   └── PerformanceChart.tsx
│   ├── layout/              # Layout components
│   │   ├── Header.tsx
│   │   ├── Sidebar.tsx
│   │   ├── Navigation.tsx
│   │   └── Footer.tsx
│   └── business/            # Business-specific components
│       ├── WalletCard.tsx
│       ├── PlayCard.tsx
│       └── TaskList.tsx
│
├── hooks/                   # Custom React hooks
│   ├── useAuth.ts
│   ├── useApi.ts
│   ├── useWebSocket.ts
│   └── usePortfolio.ts
│
├── lib/                     # Utility libraries
│   ├── api.ts              # API client
│   ├── auth.ts             # Auth utilities
│   ├── utils.ts            # General utilities
│   ├── validations.ts      # Zod schemas
│   └── constants.ts        # App constants
│
├── store/                   # State management
│   ├── slices/             # Redux Toolkit slices
│   │   ├── authSlice.ts
│   │   ├── userSlice.ts
│   │   ├── walletSlice.ts
│   │   └── portfolioSlice.ts
│   ├── middleware/         # Custom middleware
│   └── index.ts            # Store configuration
│
├── types/                   # TypeScript definitions
│   ├── api.ts
│   ├── user.ts
│   ├── wallet.ts
│   └── index.ts
│
└── styles/                  # Styling
    ├── globals.css
    ├── components.css
    └── utils.css
```

## Naming Conventions

### File and Directory Naming

#### Backend (NestJS)
```typescript
// Files
user.controller.ts          // Controllers
user.service.ts            // Services
user.repository.ts         // Repositories
user.entity.ts            // Entities
user.dto.ts               // DTOs
create-user.command.ts    // Commands
get-user.query.ts         // Queries
user-created.event.ts     // Events

// Directories
kebab-case                // All directories use kebab-case
user-service/
wallet-service/
portfolio-tracking/
```

#### Frontend (React/NextJS)
```typescript
// Components (PascalCase)
UserProfile.tsx
WalletCard.tsx
PortfolioChart.tsx

// Hooks (camelCase starting with 'use')
useAuth.ts
usePortfolio.ts
useWebSocket.ts

// Utilities (camelCase)
apiClient.ts
authUtils.ts
formatters.ts

// Types (PascalCase)
User.ts
Wallet.ts
ApiResponse.ts
```

### Code Naming Standards

```typescript
// Variables and functions: camelCase
const userData = await fetchUserData();
const isAuthenticated = checkAuthStatus();

// Classes and interfaces: PascalCase
class UserService {}
interface IPaymentProcessor {}

// Constants: SCREAMING_SNAKE_CASE
const API_BASE_URL = 'https://api.lucent.com';
const MAX_RETRY_ATTEMPTS = 3;

// Enums: PascalCase
enum UserRole {
  USER = 'user',
  ADMIN = 'admin',
  SUPER_ADMIN = 'super_admin'
}

// Generic types: Single uppercase letter or PascalCase
interface Repository<T> {}
interface ApiResponse<TData> {}
```

## API Design Guidelines

### RESTful API Standards

#### URL Structure
```
# Resource-based URLs
GET    /api/v1/users                    # Get all users
GET    /api/v1/users/{userId}           # Get specific user
POST   /api/v1/users                    # Create user
PUT    /api/v1/users/{userId}           # Update user
DELETE /api/v1/users/{userId}           # Delete user

# Nested resources
GET    /api/v1/users/{userId}/wallets   # Get user's wallets
POST   /api/v1/users/{userId}/wallets   # Create wallet for user
GET    /api/v1/wallets/{walletId}       # Get specific wallet

# Query parameters for filtering, sorting, pagination
GET /api/v1/users?role=admin&sort=createdAt&page=1&limit=20

# Action-based endpoints (use sparingly)
POST /api/v1/wallets/{walletId}/refresh-balance
POST /api/v1/plays/{playId}/execute
POST /api/v1/users/{userId}/reset-password
```

#### HTTP Status Code Standards

```typescript
// Success responses
200 OK              // Successful GET, PUT, PATCH
201 Created         // Successful POST
202 Accepted        // Async operation started
204 No Content      // Successful DELETE

// Client error responses
400 Bad Request     // Invalid request format
401 Unauthorized    // Authentication required
403 Forbidden       // Insufficient permissions
404 Not Found       // Resource doesn't exist
409 Conflict        // Resource conflict (duplicate)
422 Unprocessable   // Validation errors

// Server error responses
500 Internal Error  // Generic server error
503 Service Unavailable // Service temporarily down
```

#### Response Format Standards

```typescript
// Success response format
interface SuccessResponse<T = any> {
  success: true;
  data: T;
  message?: string;
  meta?: {
    timestamp: string;
    requestId: string;
    version: string;
  };
}

// Error response format
interface ErrorResponse {
  success: false;
  error: {
    code: string;
    message: string;
    details?: any;
  };
  meta: {
    timestamp: string;
    requestId: string;
    version: string;
  };
}

// Paginated response format
interface PaginatedResponse<T> {
  success: true;
  data: T[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    pages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
  meta?: {
    timestamp: string;
    requestId: string;
  };
}

// Example implementations
@Controller('api/v1/users')
export class UserController {
  @Get()
  async getUsers(@Query() query: GetUsersQueryDto): Promise<PaginatedResponse<User>> {
    const result = await this.userService.getUsers(query);
    
    return {
      success: true,
      data: result.users,
      pagination: {
        page: query.page,
        limit: query.limit,
        total: result.total,
        pages: Math.ceil(result.total / query.limit),
        hasNext: query.page * query.limit < result.total,
        hasPrev: query.page > 1
      },
      meta: {
        timestamp: new Date().toISOString(),
        requestId: this.requestContext.requestId
      }
    };
  }

  @Post()
  async createUser(@Body() createUserDto: CreateUserDto): Promise<SuccessResponse<User>> {
    const user = await this.userService.createUser(createUserDto);
    
    return {
      success: true,
      data: user,
      message: 'User created successfully',
      meta: {
        timestamp: new Date().toISOString(),
        requestId: this.requestContext.requestId,
        version: '1.0.0'
      }
    };
  }
}
```

### GraphQL Standards (Optional)

```typescript
// Schema organization
type User {
  id: ID!
  web3AuthId: String!
  email: String
  name: String
  role: UserRole!
  wallets: [Wallet!]!
  portfolio: Portfolio
  createdAt: DateTime!
}

type Query {
  # Singular resource queries
  user(id: ID!): User
  wallet(id: ID!): Wallet
  
  # Plural resource queries with filtering
  users(
    filter: UserFilter
    sort: UserSort
    pagination: PaginationInput
  ): UserConnection!
  
  # Nested queries
  userWallets(userId: ID!): [Wallet!]!
}

type Mutation {
  # CRUD operations
  createUser(input: CreateUserInput!): UserPayload!
  updateUser(id: ID!, input: UpdateUserInput!): UserPayload!
  deleteUser(id: ID!): DeletePayload!
  
  # Action-based mutations
  executePlay(playId: ID!, userId: ID!): ExecutePlayPayload!
  refreshWalletBalance(walletId: ID!): RefreshBalancePayload!
}

# Consistent payload types
type UserPayload {
  success: Boolean!
  user: User
  errors: [Error!]
}
```

## Error Handling Standards

### Backend Error Handling

```typescript
// Custom exception hierarchy
export abstract class BaseException extends Error {
  abstract readonly code: string;
  abstract readonly statusCode: number;
  
  constructor(
    message: string,
    public readonly details?: any
  ) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class ValidationException extends BaseException {
  readonly code = 'VALIDATION_ERROR';
  readonly statusCode = 422;
}

export class NotFoundResourceException extends BaseException {
  readonly code = 'RESOURCE_NOT_FOUND';
  readonly statusCode = 404;
}

export class InsufficientPermissionsException extends BaseException {
  readonly code = 'INSUFFICIENT_PERMISSIONS';
  readonly statusCode = 403;
}

export class BusinessLogicException extends BaseException {
  readonly code = 'BUSINESS_LOGIC_ERROR';
  readonly statusCode = 400;
}

// Global exception filter
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  constructor(private readonly logger: Logger) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let statusCode = 500;
    let code = 'INTERNAL_ERROR';
    let message = 'An unexpected error occurred';
    let details = null;

    if (exception instanceof BaseException) {
      statusCode = exception.statusCode;
      code = exception.code;
      message = exception.message;
      details = exception.details;
    } else if (exception instanceof HttpException) {
      statusCode = exception.getStatus();
      const response = exception.getResponse();
      message = typeof response === 'string' ? response : (response as any).message;
    }

    const errorResponse: ErrorResponse = {
      success: false,
      error: {
        code,
        message,
        ...(details && { details })
      },
      meta: {
        timestamp: new Date().toISOString(),
        requestId: request.headers['x-request-id'] as string,
        version: '1.0.0'
      }
    };

    this.logger.error(
      `${request.method} ${request.url} - ${statusCode} - ${message}`,
      {
        error: exception,
        request: {
          method: request.method,
          url: request.url,
          headers: request.headers,
          body: request.body
        }
      }
    );

    response.status(statusCode).json(errorResponse);
  }
}

// Usage in services
@Injectable()
export class UserService {
  async getUserById(userId: string): Promise<User> {
    const user = await this.userRepository.findById(userId);
    
    if (!user) {
      throw new NotFoundResourceException(
        `User with ID ${userId} not found`,
        { userId }
      );
    }
    
    return user;
  }

  async updateUser(userId: string, updates: UpdateUserDto): Promise<User> {
    // Validation
    if (updates.email && !this.isValidEmail(updates.email)) {
      throw new ValidationException(
        'Invalid email format',
        { field: 'email', value: updates.email }
      );
    }

    // Business logic validation
    if (updates.role === 'super_admin' && !this.currentUser.isSuperAdmin()) {
      throw new InsufficientPermissionsException(
        'Only super admins can assign super admin role'
      );
    }

    const user = await this.getUserById(userId);
    return await this.userRepository.update(userId, updates);
  }
}
```

### Frontend Error Handling

```typescript
// Error boundary component
interface ErrorBoundaryState {
  hasError: boolean;
  error?: Error;
}

class ErrorBoundary extends Component<PropsWithChildren, ErrorBoundaryState> {
  constructor(props: PropsWithChildren) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    
    // Send to error reporting service
    this.reportError(error, errorInfo);
  }

  private reportError(error: Error, errorInfo: ErrorInfo) {
    // Send to monitoring service (e.g., Sentry)
    if (process.env.NODE_ENV === 'production') {
      // Sentry.captureException(error, { contexts: { react: errorInfo } });
    }
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-boundary">
          <h2>Something went wrong</h2>
          <p>We're sorry for the inconvenience. Please try refreshing the page.</p>
          <button onClick={() => window.location.reload()}>
            Refresh Page
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

// API error handling hook
export const useApiError = () => {
  const showToast = useToast();

  const handleError = useCallback((error: unknown) => {
    if (axios.isAxiosError(error)) {
      const apiError = error.response?.data;
      
      if (apiError?.error?.code === 'VALIDATION_ERROR') {
        showToast({
          title: 'Validation Error',
          description: apiError.error.message,
          variant: 'destructive'
        });
      } else if (apiError?.error?.code === 'INSUFFICIENT_PERMISSIONS') {
        showToast({
          title: 'Access Denied',
          description: 'You don\'t have permission to perform this action',
          variant: 'destructive'
        });
      } else {
        showToast({
          title: 'Error',
          description: apiError?.error?.message || 'An unexpected error occurred',
          variant: 'destructive'
        });
      }
    } else {
      showToast({
        title: 'Error',
        description: 'An unexpected error occurred',
        variant: 'destructive'
      });
    }
  }, [showToast]);

  return { handleError };
};

// Usage in components
const UserProfile = () => {
  const { handleError } = useApiError();
  const [isLoading, setIsLoading] = useState(false);

  const handleSubmit = async (data: UpdateUserDto) => {
    try {
      setIsLoading(true);
      await UserAPI.updateProfile(data);
      // Success handling
    } catch (error) {
      handleError(error);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <ErrorBoundary>
      {/* Component content */}
    </ErrorBoundary>
  );
};
```

## Testing Standards

### Backend Testing (Jest + Supertest)

```typescript
// Unit test example
describe('UserService', () => {
  let service: UserService;
  let repository: MockRepository<User>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: getRepositoryToken(User),
          useClass: MockRepository
        }
      ]
    }).compile();

    service = module.get<UserService>(UserService);
    repository = module.get(getRepositoryToken(User));
  });

  describe('getUserById', () => {
    it('should return user when found', async () => {
      // Arrange
      const userId = 'user-123';
      const expectedUser = { id: userId, name: 'John Doe' };
      repository.findById.mockResolvedValue(expectedUser);

      // Act
      const result = await service.getUserById(userId);

      // Assert
      expect(result).toEqual(expectedUser);
      expect(repository.findById).toHaveBeenCalledWith(userId);
    });

    it('should throw NotFoundResourceException when user not found', async () => {
      // Arrange
      const userId = 'nonexistent-user';
      repository.findById.mockResolvedValue(null);

      // Act & Assert
      await expect(service.getUserById(userId))
        .rejects
        .toThrow(NotFoundResourceException);
    });
  });
});

// Integration test example
describe('UserController (Integration)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule]
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('POST /api/v1/users', () => {
    it('should create user successfully', () => {
      const createUserDto = {
        web3AuthId: 'web3-123',
        email: 'test@example.com',
        name: 'Test User'
      };

      return request(app.getHttpServer())
        .post('/api/v1/users')
        .send(createUserDto)
        .expect(201)
        .expect(res => {
          expect(res.body.success).toBe(true);
          expect(res.body.data.email).toBe(createUserDto.email);
        });
    });

    it('should return validation error for invalid email', () => {
      const createUserDto = {
        web3AuthId: 'web3-123',
        email: 'invalid-email',
        name: 'Test User'
      };

      return request(app.getHttpServer())
        .post('/api/v1/users')
        .send(createUserDto)
        .expect(422)
        .expect(res => {
          expect(res.body.success).toBe(false);
          expect(res.body.error.code).toBe('VALIDATION_ERROR');
        });
    });
  });
});
```

### Frontend Testing (Jest + React Testing Library)

```typescript
// Component test example
describe('UserProfile', () => {
  const mockUser = {
    id: '1',
    name: 'John Doe',
    email: 'john@example.com',
    role: 'user'
  };

  beforeEach(() => {
    // Mock API calls
    jest.spyOn(UserAPI, 'getProfile').mockResolvedValue(mockUser);
    jest.spyOn(UserAPI, 'updateProfile').mockResolvedValue(mockUser);
  });

  it('should display user information', async () => {
    render(<UserProfile />);

    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
      expect(screen.getByText('john@example.com')).toBeInTheDocument();
    });
  });

  it('should update profile when form is submitted', async () => {
    render(<UserProfile />);

    // Wait for component to load
    await waitFor(() => {
      expect(screen.getByDisplayValue('John Doe')).toBeInTheDocument();
    });

    // Update name field
    const nameInput = screen.getByLabelText('Name');
    fireEvent.change(nameInput, { target: { value: 'Jane Doe' } });

    // Submit form
    const submitButton = screen.getByText('Update Profile');
    fireEvent.click(submitButton);

    // Verify API call
    await waitFor(() => {
      expect(UserAPI.updateProfile).toHaveBeenCalledWith({
        name: 'Jane Doe'
      });
    });
  });
});

// Hook test example
describe('useAuth', () => {
  it('should return authenticated user', () => {
    const mockUser = { id: '1', name: 'John' };
    
    // Mock the store
    const mockStore = {
      user: mockUser,
      isLoading: false,
      login: jest.fn(),
      logout: jest.fn()
    };

    jest.spyOn(require('@/store/slices/userSlice'), 'useUserStore')
      .mockReturnValue(mockStore);

    const { result } = renderHook(() => useAuth());

    expect(result.current.user).toEqual(mockUser);
    expect(result.current.isAuthenticated).toBe(true);
    expect(result.current.isLoading).toBe(false);
  });
});
```

## Code Quality Standards

### Linting Configuration

```json
// .eslintrc.json
{
  "extends": [
    "@nestjs/eslint-config",
    "plugin:@typescript-eslint/recommended",
    "plugin:prettier/recommended"
  ],
  "rules": {
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/explicit-function-return-type": "warn",
    "@typescript-eslint/no-explicit-any": "warn",
    "prefer-const": "error",
    "no-var": "error",
    "no-console": "warn"
  }
}
```

### Pre-commit Hooks

```json
// package.json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
    }
  },
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write",
      "git add"
    ],
    "*.{js,jsx,json,md}": [
      "prettier --write",
      "git add"
    ]
  }
}
```

### Commit Message Standards

```bash
# Format: <type>(<scope>): <subject>

# Types:
feat:     # New feature
fix:      # Bug fix  
docs:     # Documentation changes
style:    # Code style changes (formatting, etc.)
refactor: # Code refactoring
test:     # Adding or updating tests
chore:    # Maintenance tasks

# Examples:
feat(auth): add Web3Auth integration
fix(wallet): resolve balance calculation issue
docs(api): update user endpoint documentation
refactor(user): extract validation logic to separate service
test(play): add unit tests for play execution
chore(deps): update dependencies to latest versions
```

This comprehensive development guidelines document ensures consistent, maintainable, and high-quality code across the entire Lucent platform development lifecycle.