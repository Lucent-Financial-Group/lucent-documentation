# Lucent Financial Crypto Platform - Design Documentation

## Overview

This repository contains the complete design documentation for rewriting the Lucent DeFi platform from a monolithic proof-of-concept to a production-ready microservices architecture. The documentation follows SOLID principles and provides concrete guidelines for AI-assisted development.

## 🏗️ Architecture Summary

The Lucent platform is a financial crypto application that manages DeFi investment strategies ("Plays") across multiple blockchain networks including EVM, Solana, and Bitcoin. The new architecture transforms the existing Angular + Express.js monolith into a scalable microservices system.

### Key Technologies
- **Backend**: NestJS with TypeScript
- **Frontend**: React/NextJS 
- **Database**: MongoDB (replacing SQL Server)
- **Key Management**: Azure Key Vault
- **Infrastructure**: Docker + Kubernetes
- **State Management**: Event-driven architecture with Redis

## 📚 Documentation Structure

### [01 - Architecture Overview](./01-Architecture-Overview.md)
- Microservices design principles and domain boundaries
- Service communication patterns and data architecture
- MongoDB integration strategy and migration approach
- Security architecture and scalability considerations

### [02 - Core Services Design](./02-Core-Services-Design.md)  
- Detailed specifications for 6 core microservices:
  - **User Service**: Authentication, authorization, profile management
  - **Wallet Service**: Multi-chain wallet operations and key management  
  - **Play Service**: DeFi strategy definition and user play instantiation
  - **Task Service**: Workflow orchestration and admin task management
  - **Portfolio Service**: Investment tracking and allocation management
  - **Notification Service**: Real-time messaging and event broadcasting
- MongoDB schema designs and API contracts
- Inter-service communication patterns

### [03 - Technical Standards](./03-Technical-Standards.md)
- NestJS backend architecture patterns and module organization
- React/NextJS frontend standards with component design
- MongoDB integration with transaction patterns and indexing
- Repository patterns, DTOs, and service interfaces
- State management with Zustand and API client configuration

### [04 - SOLID Principles Guidelines](./04-SOLID-Principles-Guidelines.md)
- Concrete examples of each SOLID principle applied to the financial domain:
  - **Single Responsibility**: Service separation with proper boundaries
  - **Open/Closed**: Extensible blockchain wallet generators
  - **Liskov Substitution**: Consistent play executor behaviors  
  - **Interface Segregation**: Focused service interfaces
  - **Dependency Inversion**: Abstract dependencies with proper injection
- Code quality checklist and refactoring guidelines

### [05 - Development Guidelines](./05-Development-Guidelines.md)
- Project structure patterns for both backend and frontend
- Naming conventions and code organization standards
- RESTful API design with consistent response formats
- Comprehensive error handling strategies
- Testing standards with Jest and React Testing Library
- Code quality tools and pre-commit hooks

### [06 - Deployment & DevOps Standards](./06-Deployment-DevOps-Standards.md)
- Docker containerization with multi-stage builds
- Kubernetes deployment configurations for production
- CI/CD pipeline with GitHub Actions
- Blue-green deployment strategy
- Monitoring with Prometheus and centralized logging
- Security policies and network configurations

## 🎯 Key Architectural Decisions

### Database Strategy
- **MongoDB** instead of SQL Server for better scalability and flexible schemas
- **Database per service** pattern with eventual consistency
- **Event sourcing** for audit trails and state reconstruction
- **Change streams** for real-time data synchronization

### Service Boundaries
Services are organized by domain with clear responsibilities:
- User management separated from wallet operations  
- DeFi strategy definition separated from execution
- Task orchestration as a dedicated workflow service
- Portfolio calculations as a specialized analytics service

### Security Approach
- **Azure Key Vault** for private key storage (no keys in database)
- **JWT-based authentication** with refresh token rotation
- **Role-based access control** with granular permissions
- **Rate limiting** and comprehensive input validation

### Scalability Design
- **Horizontal scaling** with stateless services
- **Event-driven communication** to reduce coupling
- **Caching strategies** with Redis for performance
- **Background job processing** for heavy operations

## 🚀 Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
- Set up NestJS API Gateway and User Service
- Implement authentication and authorization
- MongoDB setup with user collections
- Basic CI/CD pipeline

### Phase 2: Core Services (Weeks 3-5)  
- Implement Wallet Service with Azure Key Vault
- Build Play Service for strategy management
- Create Task Service for workflow orchestration
- Event bus implementation with Redis

### Phase 3: Advanced Features (Weeks 6-8)
- Portfolio Service with real-time calculations
- Notification Service with WebSocket support
- Frontend migration to React/NextJS
- Comprehensive testing suite

### Phase 4: Production Readiness (Weeks 9-10)
- Kubernetes deployment configuration
- Monitoring and logging implementation
- Security hardening and audit
- Performance optimization and load testing

## 🔧 AI Development Guidelines

This documentation is specifically designed for AI-assisted development:

### For Code Generation
- Use the service interfaces and DTOs as contracts
- Follow the repository and controller patterns shown
- Implement proper error handling with custom exceptions
- Include comprehensive logging and monitoring

### For Architecture Decisions
- Maintain service boundaries as defined
- Use event-driven communication between services
- Follow SOLID principles in all implementations
- Prioritize security and auditability

### For Testing
- Write unit tests for all business logic
- Include integration tests for API endpoints  
- Implement end-to-end tests for critical user flows
- Mock external dependencies appropriately

## 📋 Validation Checklist

Before considering the rewrite complete, ensure:

- [ ] All 6 microservices are implemented and tested
- [ ] MongoDB collections match the defined schemas
- [ ] API contracts follow the documented standards
- [ ] Security measures are properly implemented
- [ ] CI/CD pipeline deploys successfully
- [ ] Monitoring and logging are operational
- [ ] Performance meets or exceeds current system
- [ ] All existing functionality has been migrated
- [ ] Documentation is updated with any changes

## 🔗 Related Resources

- [Current Lucent Client](../lucent_client/) - Existing Angular application
- [Current Lucent Server](../Lucent_Server/) - Existing Express.js backend
- [NestJS Documentation](https://docs.nestjs.com/)
- [MongoDB Best Practices](https://www.mongodb.com/developer/products/mongodb/mongodb-schema-design-best-practices/)
- [Azure Key Vault Integration](https://docs.microsoft.com/en-us/azure/key-vault/)

---

This documentation provides the complete blueprint for transforming Lucent from a proof-of-concept to a production-ready, scalable DeFi platform. Each document builds upon the previous one, creating a comprehensive guide for systematic development and deployment.