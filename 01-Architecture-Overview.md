# Lucent Financial Crypto Application - Architecture Overview

## Executive Summary

This document outlines the microservices architecture for the Lucent DeFi platform, transitioning from a monolithic proof-of-concept to a production-ready, scalable system. The architecture follows SOLID principles and domain-driven design patterns, utilizing NestJS for backend services, React/NextJS for frontend, and MongoDB for data persistence.

## Current State Analysis

### Existing Architecture Problems
- **Monolithic Structure**: Single Express.js application handling all concerns
- **Tight Coupling**: Controllers directly accessing multiple services and databases
- **Mixed Responsibilities**: Authentication, wallet management, and business logic intertwined
- **Single Database**: All entities in one SQL Server database creating bottlenecks
- **Limited Scalability**: Cannot scale individual components independently

### Business Domain Understanding
The Lucent platform manages DeFi investment strategies ("Plays") across multiple blockchain networks:
- **User Management**: Web3Auth integration, role-based access control
- **Multi-Chain Wallets**: EVM, Solana, Bitcoin wallet generation and management
- **DeFi Strategies**: Configurable investment plays (leverage, staking, liquidity, farming)
- **Task Management**: Admin workflow system for strategy execution and monitoring
- **Portfolio Tracking**: Real-time balance monitoring and allocation management

## Target Microservices Architecture

### Core Design Principles

#### 1. SOLID Principles Application
- **Single Responsibility**: Each service handles one business domain
- **Open/Closed**: Services extensible through interfaces, closed for modification
- **Liskov Substitution**: Implementation interchangeability through contracts
- **Interface Segregation**: Focused, minimal service interfaces
- **Dependency Inversion**: Services depend on abstractions, not concretions

#### 2. Domain-Driven Design
- **Bounded Contexts**: Clear service boundaries aligned with business domains
- **Ubiquitous Language**: Consistent terminology across services
- **Aggregate Design**: Proper entity relationships within service boundaries

### Microservices Breakdown

#### 1. User Service
**Domain**: Authentication, authorization, user profile management
**Database**: MongoDB users collection
**Responsibilities**:
- Web3Auth integration and JWT management
- User registration, profile updates
- Role-based access control (RBAC)
- User settings and preferences

#### 2. Wallet Service  
**Domain**: Multi-chain wallet operations and key management
**Database**: MongoDB wallets collection
**Responsibilities**:
- Wallet generation for EVM, Solana, Bitcoin
- Azure Key Vault integration for private key storage
- Address derivation and public key management
- Wallet balance tracking and updates

#### 3. Play Service
**Domain**: DeFi strategy definition and configuration
**Database**: MongoDB plays collection  
**Responsibilities**:
- Play creation and management (leverage, staking, etc.)
- Configuration schema definition
- Play status and availability management
- Strategy metadata and display configuration

#### 4. Task Service
**Domain**: Workflow orchestration and task management
**Database**: MongoDB tasks collection
**Responsibilities**:
- Admin task creation and assignment
- Task step definition and execution tracking
- User play task instantiation
- Workflow state management

#### 5. Portfolio Service
**Domain**: Investment tracking and allocation management  
**Database**: MongoDB portfolio collection
**Responsibilities**:
- User portfolio balance calculation
- Play allocation percentage tracking
- Performance metrics and health ratings
- Transaction history and audit trails

#### 6. Notification Service
**Domain**: Real-time messaging and event broadcasting
**Database**: MongoDB notifications collection
**Responsibilities**:
- WebSocket connection management
- Real-time task updates and status changes
- Email and push notification delivery
- Event logging and audit

### Service Communication Patterns

#### 1. API Gateway Pattern
- **Kong or AWS API Gateway** for request routing and rate limiting
- **Authentication middleware** for JWT validation
- **Request/Response transformation** for client compatibility
- **Load balancing** across service instances

#### 2. Event-Driven Architecture
- **MongoDB Change Streams** for database event triggers
- **Redis Pub/Sub** for inter-service messaging
- **Event Sourcing** for audit trails and state reconstruction
- **Saga Pattern** for distributed transactions

#### 3. Database per Service
- **MongoDB Collections per Service**: Users, Wallets, Plays, Tasks, Portfolio, Notifications
- **Data Ownership**: Each service owns its data completely
- **Eventual Consistency**: Accept temporary inconsistency for scalability
- **Data Synchronization**: Event-driven updates between services

## MongoDB Data Architecture

### Collection Design Patterns

#### Document Structure Guidelines
- **Embedded Documents**: For one-to-few relationships (e.g., user preferences)
- **References**: For one-to-many relationships (e.g., user to wallets)
- **Denormalization**: Strategic duplication for read performance
- **Indexing Strategy**: Compound indexes for query optimization

#### Schema Evolution
- **Versioning Strategy**: Schema version fields in documents
- **Migration Patterns**: Lazy migration during reads
- **Backward Compatibility**: Support multiple schema versions

### Data Consistency Patterns
- **Single Document Transactions**: For atomic operations within service
- **Multi-Document Transactions**: For complex operations requiring ACID
- **Eventually Consistent Reads**: Accept stale data for performance
- **Compensation Actions**: Rollback strategies for failed operations

## Security Architecture

### Authentication & Authorization
- **JWT Tokens**: Stateless authentication with refresh mechanisms
- **Role-Based Access Control**: User, Admin, Super Admin roles
- **Permission-Based Access**: Granular permissions for fine control
- **Web3Auth Integration**: Wallet-based authentication flow

### Key Management
- **Azure Key Vault**: Centralized private key storage
- **Encryption at Rest**: MongoDB encryption for sensitive data
- **Encryption in Transit**: TLS for all service communication
- **Key Rotation**: Regular rotation policies for security keys

### API Security
- **Rate Limiting**: Request throttling per user/IP
- **Input Validation**: Comprehensive request validation
- **SQL Injection Prevention**: Parameterized queries only
- **CORS Configuration**: Restricted cross-origin access

## Scalability Considerations

### Horizontal Scaling
- **Stateless Services**: No server-side session storage
- **Load Balancing**: Distribute requests across service instances
- **Database Sharding**: Partition data across MongoDB replica sets
- **Caching Strategy**: Redis for frequently accessed data

### Performance Optimization
- **Connection Pooling**: Database connection management
- **Query Optimization**: Proper indexing and query patterns
- **Lazy Loading**: Load data on-demand to reduce memory usage
- **Background Processing**: Async operations for heavy tasks

## Deployment Architecture

### Containerization
- **Docker**: Service containerization with multi-stage builds
- **Kubernetes**: Orchestration and service mesh
- **Helm Charts**: Deployment configuration management
- **Service Discovery**: Automatic service registration and discovery

### Environment Management
- **Development**: Local Docker Compose setup
- **Staging**: Kubernetes cluster with production data simulation
- **Production**: Multi-region deployment with failover capabilities
- **Configuration Management**: Environment-specific configurations

## Monitoring and Observability

### Logging Strategy
- **Centralized Logging**: ELK Stack or similar aggregation
- **Structured Logging**: JSON format for searchability
- **Correlation IDs**: Request tracing across services
- **Log Levels**: Appropriate logging levels for debugging

### Metrics and Monitoring
- **Application Metrics**: Response times, error rates, throughput
- **Business Metrics**: User engagement, transaction volumes
- **Infrastructure Metrics**: CPU, memory, disk, network usage
- **Alerting**: Proactive notifications for critical issues

## Migration Strategy

### Phased Approach
1. **Phase 1**: Extract User Service and implement API Gateway
2. **Phase 2**: Extract Wallet Service with key vault integration
3. **Phase 3**: Extract Play Service and Task Service
4. **Phase 4**: Implement Portfolio and Notification services
5. **Phase 5**: Frontend migration to React/NextJS

### Risk Mitigation
- **Feature Flags**: Toggle between old and new implementations
- **Database Migration**: Gradual data migration with validation
- **Rollback Strategy**: Quick reversion to previous state if needed
- **Load Testing**: Comprehensive performance validation

## Conclusion

This architecture provides a foundation for building a scalable, maintainable DeFi platform that can evolve with business requirements. The microservices approach enables independent development, deployment, and scaling of individual components while maintaining system reliability and security.

The MongoDB-based data architecture provides the flexibility needed for evolving schemas while maintaining performance and consistency requirements for financial applications.