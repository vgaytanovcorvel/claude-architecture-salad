# C# Solution Modularization

## Overview

This document defines the standard assembly structure for C# solutions. Each assembly should follow the naming convention `[ProjectNamespace].[AssemblyType]`, where `[ProjectNamespace]` represents the root namespace of the project.

**Note**: Assemblies should only be created if applicable to the project's requirements.

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

### Angular SPA Frontend (Separate `.esproj` Project)

Starting with .NET 8 and Visual Studio 2022 v17.8+, the Angular SPA is a **separate sibling project** using the JavaScript Project System (`.esproj`). It is **not** nested inside `Web.Server` as a `ClientApp/` subfolder — that is the legacy pattern.

The `.esproj` project uses the `Microsoft.VisualStudio.JavaScript.Sdk` and is referenced by `Web.Server` via a `<ProjectReference>` with `ReferenceOutputAssembly=false`.

#### Solution-Level Layout

```
src/
├── [projectnamespace].client/                 ← Angular CLI workspace (.esproj)
│   ├── [projectnamespace].client.esproj
│   ├── angular.json
│   ├── package.json
│   ├── tsconfig.json
│   ├── tsconfig.app.json
│   ├── tsconfig.spec.json
│   ├── proxy.conf.js
│   └── src/
│       ├── main.ts
│       ├── index.html
│       ├── styles.scss
│       └── app/
│           ├── app.component.ts
│           ├── app.config.ts
│           ├── app.routes.ts
│           ├── core/
│           │   ├── services/
│           │   ├── guards/
│           │   ├── interceptors/
│           │   └── models/
│           ├── features/
│           │   ├── todos/
│           │   │   ├── components/
│           │   │   ├── services/
│           │   │   ├── models/
│           │   │   └── todos.routes.ts
│           │   └── auth/
│           │       ├── components/
│           │       ├── services/
│           │       └── auth.routes.ts
│           ├── shared/
│           │   ├── components/
│           │   ├── pipes/
│           │   └── directives/
│           └── layout/
│               ├── header/
│               ├── sidebar/
│               └── footer/
│
└── [ProjectNamespace].Web.Server/
    ├── [ProjectNamespace].Web.Server.csproj    ← References .esproj
    ├── Program.cs
    └── wwwroot/                                ← Production SPA output goes here
```

**Naming convention**: The `.esproj` project uses **lowercase** naming (e.g., `corvel.todos.client`) to match npm/Angular conventions, while .NET projects use PascalCase.

#### Folder Responsibilities

| Folder | Purpose | Rules |
|---|---|---|
| `core/services/` | Singleton services (`providedIn: 'root'`), API clients | One service per file. No component-level providers. |
| `core/guards/` | Functional route guards | Must use `CanActivateFn` (not class-based guards). |
| `core/interceptors/` | HTTP interceptors | Must use `HttpInterceptorFn`. |
| `core/models/` | TypeScript interfaces and type aliases shared across features | No classes — interfaces and types only. |
| `features/<name>/` | Self-contained feature slices | Each feature has its own `components/`, `services/`, `models/`, and route file. |
| `shared/components/` | Reusable presentational (dumb) components | No service injection. Input/output only. |
| `shared/pipes/` | Reusable pipes | Must be standalone (the default in Angular 19+). |
| `shared/directives/` | Reusable directives | Must be standalone (the default in Angular 19+). |
| `layout/` | Application shell (header, sidebar, footer) | May inject Router and AuthService. |

#### The `.esproj` File

```xml
<Project Sdk="Microsoft.VisualStudio.JavaScript.Sdk/1.0">
  <PropertyGroup>
    <StartupCommand>npm start</StartupCommand>
    <JavaScriptTestRoot>src\</JavaScriptTestRoot>
    <JavaScriptTestFramework>Jasmine</JavaScriptTestFramework>
    <ShouldRunBuildScript>false</ShouldRunBuildScript>
    <PublishAssetsDirectory>$(MSBuildProjectDirectory)\dist\browser</PublishAssetsDirectory>
  </PropertyGroup>
</Project>
```

Key `.esproj` properties:

| Property | Purpose |
|---|---|
| `StartupCommand` | Dev server launch command (`npm start`) |
| `BuildCommand` | Production build command (defaults to `npm run build`) |
| `PublishAssetsDirectory` | Folder containing production output (copied to `wwwroot` on publish) |
| `ShouldRunBuildScript` | Whether `npm run build` runs on every `dotnet build` |
| `ShouldRunNpmInstall` | Whether `npm install` runs on restore (default `true`) |

#### Web.Server `.csproj` Integration

The server project references the `.esproj` — no `Microsoft.AspNetCore.SpaProxy` package is needed:

```xml
<ItemGroup>
  <ProjectReference
    Include="..\[projectnamespace].client\[projectnamespace].client.esproj">
    <ReferenceOutputAssembly>false</ReferenceOutputAssembly>
  </ProjectReference>
</ItemGroup>
```

#### Web.Server `Program.cs`

```csharp
var app = builder.Build();

app.UseHttpsRedirection();
app.UseDefaultFiles();   // Serves index.html for root
app.UseStaticFiles();    // Serves wwwroot/ contents
app.UseRouting();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller}/{action=Index}/{id?}");

app.MapFallbackToFile("index.html");  // SPA fallback route

app.Run();
```

#### Required Angular Infrastructure Files

**`src/main.ts`** — Bootstraps the application:

```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig);
```

**`src/app/app.config.ts`** — Central provider configuration:

```typescript
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';
import { routes } from './app.routes';
import { authInterceptor } from './core/interceptors/auth.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
    provideAnimationsAsync()
  ]
};
```

**`src/app/app.routes.ts`** — Top-level lazy-loaded route configuration:

```typescript
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', redirectTo: 'todos', pathMatch: 'full' },
  {
    path: 'todos',
    loadChildren: () =>
      import('./features/todos/todos.routes').then(m => m.TODOS_ROUTES)
  },
  { path: '**', redirectTo: 'todos' }
];
```

**Feature route files** (e.g., `features/todos/todos.routes.ts`):

```typescript
import { Routes } from '@angular/router';

export const TODOS_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () =>
      import('./components/todo-list/todo-list.component').then(m => m.TodoListComponent)
  },
  {
    path: ':id',
    loadComponent: () =>
      import('./components/todo-detail/todo-detail.component').then(m => m.TodoDetailComponent)
  }
];
```

#### Development Proxy Configuration

During development, both projects run simultaneously (multi-startup in VS). The Angular dev server proxies API requests to the .NET backend via `proxy.conf.js`:

```javascript
const { env } = require('process');

const target = env.ASPNETCORE_HTTPS_PORTS
  ? `https://localhost:${env.ASPNETCORE_HTTPS_PORTS}`
  : env.ASPNETCORE_URLS
    ? env.ASPNETCORE_URLS.split(';')[0]
    : 'https://localhost:7001';

const PROXY_CONFIG = [
  {
    context: ["/api"],
    target,
    secure: false
  }
];

module.exports = PROXY_CONFIG;
```

Referenced in `angular.json` under the `serve` target:

```json
"serve": {
  "configurations": {
    "development": {
      "proxyConfig": "proxy.conf.js"
    }
  }
}
```

#### Development Workflow

1. Visual Studio starts both projects via multi-startup configuration
2. The `.esproj` `StartupCommand` runs `npm start` → `ng serve` on its own port
3. The `.csproj` starts the ASP.NET Core backend on its own port
4. Angular's `proxy.conf.js` forwards `/api` requests to the .NET backend
5. The browser opens the Angular dev server URL

#### Production Build

1. `dotnet publish` on Web.Server triggers the `.esproj` build via the `<ProjectReference>`
2. The `.esproj` runs `ng build --configuration production`
3. Angular outputs AoT-compiled static files to `dist/browser/`
4. Those files are copied into Web.Server's `wwwroot/`
5. `app.MapFallbackToFile("index.html")` handles SPA client-side routing

#### Key Constraints

- **Separate project** — the Angular SPA is a sibling `.esproj`, never nested inside Web.Server
- **No NgModules** — all components, pipes, and directives are standalone by default (Angular 19+)
- **Lazy load every feature** — top-level routes use `loadChildren` or `loadComponent`
- **Core is not feature code** — `core/` contains only cross-cutting singletons (auth, HTTP, logging); feature-specific services belong in `features/<name>/services/`
- **Shared is presentation only** — `shared/` components must be stateless and reusable; no business logic or service injection
- **One component per file** — never declare multiple components in a single `.ts` file
- **Proxy for development** — all `/api` calls go through `proxy.conf.js` to the .NET backend; never hardcode backend URLs in Angular code
- **No `Microsoft.AspNetCore.SpaProxy`** — the modern template does not use SpaProxy; multi-startup replaces it

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
