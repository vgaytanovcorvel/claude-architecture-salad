# C# Solution Modularization

## Overview

This document defines the standard assembly structure for C# solutions. Each assembly should follow the naming convention `[ProjectNamespace].[AssemblyType]`, where `[ProjectNamespace]` represents the root namespace of the project.

**Note**: Assemblies should only be created if applicable to the project's requirements.

## Internal Folder Organization

Organize files within each assembly **by type** into dedicated folders. Each type category gets its own folder at the assembly root:

| Folder | Contains |
|---|---|
| `Constants/` | Constant values and static configuration |
| `Configurations/` | EF Core entity type configurations (`IEntityTypeConfiguration<T>`) |
| `Contexts/` | DbContext classes |
| `Controllers/` | MVC/API controllers |
| `Entities/` | ORM entity classes (with navigation properties, attributes) |
| `Enums/` | Enumerations |
| `Exceptions/` | Custom exception types |
| `Extensions/` | Extension methods and DI registration helpers |
| `Filters/` | Action filters, result filters, exception filters |
| `Interfaces/` | Service and repository interfaces (contracts) |
| `Middleware/` | Custom middleware components |
| `Migrations/` | EF Core database migrations |
| `ModelBinders/` | Custom model binders |
| `Models/` | DTOs, domain models, request/response types |
| `Repositories/` | Repository implementations |
| `Services/` | Service implementations |
| `Utilities/` | Helper and utility classes |
| `Validators/` | FluentValidation validators |
| `ValueObjects/` | Value objects |

Create only the folders applicable to each assembly. For example, `Enums/` and `Exceptions/` belong in Abstractions but not in Repository; `Entities/` and `Migrations/` belong in Repository but not in Implementation.

## Assembly Structure

### 1. [ProjectNamespace].Common

**Purpose**: Shared code between client and backend

**Contains**:
- Shared DTOs (Data Transfer Objects)
- Common constants and enumerations
- Shared utilities used by both client and server
- Cross-cutting concerns applicable to both tiers

**Dependencies**: Minimal external dependencies

**When to create**: When you have code that needs to be shared between client and server projects

---

### 2. [ProjectNamespace].Client

**Purpose**: Client-side library to communicate with backend API

**Contains**:
- HTTP client implementations
- API client interfaces and implementations
- Request/response models specific to API communication
- Client-side proxy classes

**Dependencies**:
- [ProjectNamespace].Common
- [ProjectNamespace].Abstractions (for contracts)

**When to create**: When building a client library or SDK for consuming the backend API

---

### 3. [ProjectNamespace].Abstractions

**Purpose**: Define contracts and lightweight abstractions

**Contains**:
- Domain models (persistence-ignorant — no ORM attributes or navigation properties)
- Service interfaces (contracts)
- Repository interfaces
- Custom exception types
- Lightweight helper classes
- Domain enumerations
- Value objects

**Dependencies**: Minimal; should avoid heavy framework dependencies

**Best Practices**:
- Keep this assembly lightweight
- No implementation details
- No framework-specific code when possible
- Should be safe to reference from any other assembly

---

### 4. [ProjectNamespace].Implementation

**Purpose**: Implement business logic and service interfaces

**Contains**:
- Service implementations
- Business logic layer
- Domain service implementations
- Application services
- Validation logic
- Business rules enforcement

**Dependencies**:
- [ProjectNamespace].Abstractions
- [ProjectNamespace].Common
- Third-party libraries as needed

**Best Practices**:
- Implement interfaces defined in .Abstractions
- Keep repository concerns separate (use .Repository)
- Focus on business logic, not data access

---

### 5. [ProjectNamespace].Repository

**Purpose**: Implement data access layer

**Contains**:
- Repository implementations
- ORM entity classes with navigation properties (e.g., `UserEntity`, `TodoItemEntity`)
- Mapping between domain models and ORM entities
- Data access code (Entity Framework, Dapper, etc.)
- Database context classes
- Data migrations
- Query specifications

**Dependencies**:
- [ProjectNamespace].Abstractions
- [ProjectNamespace].Common
- ORM frameworks (EF Core, etc.)

**Rationale**: Separated into its own assembly to:
- Simplify integration testing
- Allow easy mocking in unit tests
- Enable swapping persistence strategies
- Maintain clean architecture boundaries

**Best Practices**:
- Implement repository interfaces from .Abstractions
- Keep database-specific logic isolated
- ORM entity classes (with navigation properties, `[Table]` attributes) belong here, not in Abstractions
- Use dependency injection for registration

---

### 6. [ProjectNamespace].Web.Core

**Purpose**: Core web functionality for ASP.NET applications

**Contains**:
- Controllers (MVC/API)
- Web-specific services
- Filters and middleware
- Model binders
- Action filters and result filters
- Web-specific dependency injection configuration

**Dependencies**:
- [ProjectNamespace].Abstractions
- [ProjectNamespace].Implementation
- [ProjectNamespace].Common
- ASP.NET Core packages

**Best Practices**:
- Keep controllers thin
- Delegate business logic to services from .Implementation
- Use attribute routing
- Implement proper error handling

---

### 7. [ProjectNamespace].Web.Server

**Purpose**: Web application that serves/drives the frontend

**Contains**:
- Program.cs and Startup.cs/Program configuration
- Frontend asset serving configuration
- SPA hosting setup
- Application entry point
- Configuration and middleware pipeline setup

**Dependencies**:
- [ProjectNamespace].Web.Core
- [ProjectNamespace].Implementation
- [ProjectNamespace].Repository
- Static file middleware
- SPA middleware (for Angular, React, etc.)

**When to create**: When building a server-side rendered application or hosting a SPA

**Best Practices**:
- Keep this project minimal
- Focus on application bootstrapping
- Configure dependency injection container
- Set up middleware pipeline

---

### Angular SPA Frontend

The Angular SPA is a **separate sibling `.esproj` project** — see `typescript/angular.md` for full structure, folder responsibilities, `.esproj` configuration, proxy setup, and constraints.

---

### 8. [ProjectNamespace].Web.Api

**Purpose**: Web API host not directly meant for frontend consumption

**Contains**:
- Program.cs and API configuration
- API-specific middleware setup
- Swagger/OpenAPI configuration
- API versioning setup
- Authentication/authorization policies

**Dependencies**:
- [ProjectNamespace].Web.Core
- [ProjectNamespace].Implementation
- [ProjectNamespace].Repository

**When to create**: When you need a separate API endpoint for:
- Third-party integrations
- B2B APIs
- Internal service-to-service communication
- Public API endpoints

**Best Practices**:
- Configure proper API documentation (Swagger)
- Implement API versioning
- Use appropriate authentication (JWT, OAuth, etc.)

---

## Test Projects

### Naming Convention

For each assembly, create a corresponding test project:
- `[ProjectNamespace].[AssemblyType].Tests`

### Examples:
- [ProjectNamespace].Common.Tests
- [ProjectNamespace].Abstractions.Tests
- [ProjectNamespace].Implementation.Tests
- [ProjectNamespace].Repository.Tests
- [ProjectNamespace].Web.Core.Tests
- [ProjectNamespace].Web.Server.Tests
- [ProjectNamespace].Web.Api.Tests
- [ProjectNamespace].Client.Tests

### Test Project Structure

**Contains**:
- Unit tests for the corresponding assembly
- Integration tests (where applicable)
- Test fixtures and helpers
- Mock/fake implementations

**Dependencies**:
- The assembly being tested
- Testing frameworks (xUnit, NUnit, MSTest)
- Mocking libraries (Moq, NSubstitute)
- Assertion libraries (FluentAssertions)

**Best Practices**:
- Mirror the folder structure of the tested assembly
- Use meaningful test names (Method_Scenario_ExpectedResult)
- Implement AAA pattern (Arrange, Act, Assert)
- Keep tests isolated and independent
- Use test fixtures for shared setup
- Mock external dependencies

### Special Considerations for Repository Tests

**[ProjectNamespace].Repository.Tests**:
- Use in-memory database for integration tests
- Consider using TestContainers for real database testing
- Test actual database queries and data access logic
- Verify migrations and schema consistency

---

## Solution Organization Example

```
Solution: MyProject
│
├── src/
│   ├── myproject.client/               ← Angular SPA (.esproj)
│   │   ├── myproject.client.esproj
│   │   ├── angular.json
│   │   ├── package.json
│   │   ├── proxy.conf.js
│   │   └── src/
│   │       └── app/
│   │           ├── core/               ← Singleton services, guards, interceptors
│   │           ├── features/           ← Feature-sliced lazy modules
│   │           ├── shared/             ← Reusable components, pipes, directives
│   │           └── layout/             ← Shell (header, sidebar, footer)
│   ├── MyProject.Common/
│   ├── MyProject.Abstractions/
│   ├── MyProject.Implementation/
│   ├── MyProject.Repository/
│   ├── MyProject.Client/
│   ├── MyProject.Web.Core/
│   ├── MyProject.Web.Server/           ← References myproject.client.esproj
│   └── MyProject.Web.Api/
│
└── tests/
    ├── MyProject.Common.Tests/
    ├── MyProject.Abstractions.Tests/
    ├── MyProject.Implementation.Tests/
    ├── MyProject.Repository.Tests/
    ├── MyProject.Client.Tests/
    ├── MyProject.Web.Core.Tests/
    ├── MyProject.Web.Server.Tests/
    └── MyProject.Web.Api.Tests/
```

## Central Package Management (CRITICAL)

Every solution MUST use Central Package Management via `Directory.Packages.props` at the solution root. Individual `.csproj` files use `<PackageReference>` without `Version`.

---

## Dependency Flow Guidelines

### Allowed Dependencies (Bottom-Up):

1. **Common** ← No project dependencies
2. **Abstractions** ← Common
3. **Implementation** ← Abstractions, Common
4. **Repository** ← Abstractions, Common
5. **Client** ← Abstractions, Common
6. **Web.Core** ← Abstractions, Implementation, Common
7. **Web.Server** ← Web.Core, Implementation, Repository, client `.esproj` (ReferenceOutputAssembly=false)
8. **Web.Api** ← Web.Core, Implementation, Repository

The Angular `.esproj` has no .NET assembly dependencies — it communicates with the backend exclusively via HTTP at runtime.

### Forbidden Dependencies:

- **Abstractions** should NOT reference Implementation or Repository
- **Implementation** should NOT reference Repository or Web.*
- **Repository** should NOT reference Implementation or Web.*
- **Common** should NOT reference any project assemblies
- **Web.Server** and **Web.Api** should NOT reference each other
- **Angular `.esproj`** should NOT reference any .NET assembly (it is a build-only reference)

---

## Benefits of This Structure

1. **Separation of Concerns**: Each assembly has a clear, single responsibility
2. **Testability**: Easy to write unit and integration tests with proper boundaries
3. **Maintainability**: Changes are isolated to specific assemblies
4. **Reusability**: Core logic can be reused across different hosts
5. **Deployment Flexibility**: Can deploy API and Server separately if needed
6. **Team Scalability**: Different teams can work on different assemblies
7. **Clean Architecture**: Follows dependency inversion principle

---

## Migration Strategy

When refactoring an existing monolithic project:

1. Start by creating .Abstractions and moving interfaces and models
2. Create .Common and move shared utilities
3. Extract .Implementation and move business logic
4. Separate .Repository and move data access code
5. Create .Web.Core and move controllers/web services
6. Split into .Web.Server and/or .Web.Api as needed
7. Add test projects incrementally as you modularize

---

## Notes

- Not all projects require all assemblies
- Start simple and add assemblies as complexity grows
- Avoid over-engineering for small projects
- Consider monorepo vs multi-repo based on deployment needs
- Use internal visibility appropriately to control API surface
