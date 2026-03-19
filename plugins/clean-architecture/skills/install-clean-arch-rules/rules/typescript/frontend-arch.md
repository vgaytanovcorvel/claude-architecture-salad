---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# Frontend Clean Architecture

> Framework-agnostic layering rules for React and Angular web applications. See [angular.md](angular.md) for Angular-specific implementation and [react.md](react.md) for React-specific implementation.

## The Problem

Default frontend code puts data fetching, business logic, validation, state management, and rendering all inside components. The result: components that are hard to test, expensive to change, and impossible to reuse in isolation.

## Layer Model

```
Domain ← Repository ← Service ← State ← Presentation
```

Each layer depends only on layers to its left. Domain has zero framework dependencies. Presentation has zero data-fetching code.

| Layer | Responsibility | May import framework? |
|---|---|---|
| Domain | Types, interfaces, domain errors, `Result<T>` | No |
| Repository | HTTP/storage access, DTO → domain mapping | No |
| Service | Business logic, validation, orchestration | No |
| State | Reactive wrappers around services | Yes |
| Presentation | Rendering, user events | Yes |

**Forbidden dependencies:**
- Domain MUST NOT import from any other layer
- Repository MUST NOT import from Service or Presentation
- Service MUST NOT import from State or Presentation
- Presentational components MUST NOT call services or repositories directly

---

## Domain Layer

Pure TypeScript — zero imports from React, Angular, HTTP libraries, or ORMs.

```typescript
// domain/models/user.ts
export interface User {
  id: string
  email: string
  name: string
  role: UserRole
  isActive: boolean
  createdAt: Date
}

export type UserRole = 'admin' | 'member' | 'guest'
```

**Rules:**
- Domain models are plain interfaces or types — no class decorators, no framework annotations
- No validation attributes on domain types — validation lives in Zod schemas in the Service layer
- Repository interfaces are defined here (contracts), implementations live in the Repository layer
- Service interfaces are defined here, implementations live in the Service layer

### Result Pattern

Use `Result<T>` for operations where failure is an expected outcome, not an exception:

```typescript
// domain/result.ts
export type Result<T> =
  | { success: true; value: T }
  | { success: false; error: string }

export const Result = {
  ok: <T>(value: T): Result<T> => ({ success: true, value }),
  fail: <T>(error: string): Result<T> => ({ success: false, error }),
}
```

### Domain Errors

```typescript
// domain/errors.ts
export class AppError extends Error {
  constructor(message: string, public readonly code: string) {
    super(message)
    this.name = 'AppError'
  }
}

export class NotFoundException extends AppError {
  constructor(message: string) { super(message, 'NOT_FOUND') }
}

export class ValidationError extends AppError {
  constructor(message: string) { super(message, 'VALIDATION_ERROR') }
}

export class UnauthorizedError extends AppError {
  constructor(message: string) { super(message, 'UNAUTHORIZED') }
}
```

---

## Repository Layer

The repository is the **only layer that knows HTTP or storage exists**. It owns endpoints, response shapes, query parameters, and maps API DTOs to domain models. Services and components are persistence-ignorant.

```typescript
// domain/interfaces/i-user-repository.ts — interface defined in Domain
export interface IUserRepository {
  userSingleById(id: string): Promise<User>
  userSingleOrDefaultById(id: string): Promise<User | null>
  userSingleOrDefaultByEmail(email: string): Promise<User | null>
  userFindAll(filters?: UserFilters): Promise<readonly User[]>
  userCreate(data: CreateUserDto): Promise<User>
  userUpdate(id: string, data: UpdateUserDto): Promise<User>
  userDelete(id: string): Promise<void>
}
```

See [typescript/patterns.md](patterns.md) for full naming conventions (`userSingleById`, `userFindAll`, etc.) and Single vs SingleOrDefault semantics.

**Rules:**
- All HTTP calls happen here and only here — nothing above the repository calls `fetch`, `axios`, or `HttpClient` directly
- Maps API response DTOs to domain models before returning — callers never see raw API shapes
- No framework imports in the interface or base implementation (Angular `@Injectable` is the only exception — see [angular.md](angular.md))
- Mock implementations (`MockUserRepository`) live alongside real ones and are injected in tests

```typescript
// repositories/http-user-repository.ts
export class HttpUserRepository implements IUserRepository {
  constructor(private readonly baseUrl: string) {}

  async userSingleById(id: string): Promise<User> {
    const user = await this.userSingleOrDefaultById(id)
    if (!user) throw new NotFoundException(`User not found (UserId: ${id})`)
    return user
  }

  async userSingleOrDefaultById(id: string): Promise<User | null> {
    const res = await fetch(`${this.baseUrl}/users/${id}`)
    if (res.status === 404) return null
    if (!res.ok) throw new AppError(`API error ${res.status}`, 'API_ERROR')
    return this.mapToDomain(await res.json() as UserDto)
  }

  async userCreate(data: CreateUserDto): Promise<User> {
    const res = await fetch(`${this.baseUrl}/users`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    })
    if (!res.ok) throw new AppError(`Create failed ${res.status}`, 'API_ERROR')
    return this.mapToDomain(await res.json())
  }

  // All DTO → domain mapping isolated in one private method
  private mapToDomain(dto: UserDto): User {
    return {
      id: dto.user_id,
      email: dto.email_address,
      name: dto.display_name,
      role: dto.role as UserRole,
      isActive: dto.is_active,
      createdAt: new Date(dto.created_at),
    }
  }
}
```

---

## Service Layer

Business logic, validation, and orchestration. Services depend on repository **interfaces**, not concrete implementations.

```typescript
// domain/interfaces/i-user-service.ts — interface defined in Domain
export interface IUserService {
  getUserById(id: string): Promise<User | null>
  getActiveUsers(): Promise<readonly User[]>
  createUser(data: CreateUserRequest): Promise<Result<User>>
  updateUser(id: string, data: UpdateUserRequest): Promise<User>
  deleteUser(id: string): Promise<void>
}
```

```typescript
// services/user-service.ts
import { z } from 'zod'

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  role: z.enum(['admin', 'member', 'guest']),
})

export class UserService implements IUserService {
  constructor(private readonly userRepo: IUserRepository) {}

  getUserById(id: string) {
    return this.userRepo.userSingleOrDefaultById(id)
  }

  getActiveUsers() {
    return this.userRepo.userFindAll({ isActive: true })
  }

  async createUser(data: CreateUserRequest): Promise<Result<User>> {
    const parsed = createUserSchema.safeParse(data)
    if (!parsed.success) return Result.fail(parsed.error.issues[0].message)

    const existing = await this.userRepo.userSingleOrDefaultByEmail(data.email)
    if (existing) return Result.fail(`Email ${data.email} is already in use`)

    const user = await this.userRepo.userCreate(parsed.data)
    return Result.ok(user)
  }

  async deleteUser(id: string): Promise<void> {
    const user = await this.userRepo.userSingleById(id)  // throws NotFoundException if absent
    if (user.role === 'admin') throw new ValidationError('Admin users cannot be deleted')
    await this.userRepo.userDelete(id)
  }
}
```

**Rules:**
- Services use natural application-level method names (`getUserById`, not `userSingleById`)
- Validation via Zod schemas in the service file — never attributes on domain types
- Services throw domain errors (`NotFoundException`, `ValidationError`) — never HTTP status codes
- No framework imports, no browser APIs, no UI concerns
- Service methods return domain models or `Result<T>` — never raw DTOs or HTTP responses

---

## Feature Slices

Organize by feature, not by type. Each feature is a self-contained module.

```
src/
  domain/                     ← interfaces, models, errors, Result<T>
  repositories/               ← HTTP implementations + mock implementations
  services/                   ← business logic services
  features/
    users/
      components/             ← presentational (dumb) components
      containers/             ← smart components (React) or pages/ (Angular)
      state/                  ← feature-scoped hooks (React) or signal stores (Angular)
      index.ts                ← public API — export only what other features need
    orders/
      ...
  shared/
    components/               ← reusable presentational UI (Button, Modal, Spinner)
    hooks/                    ← shared utility hooks/composables
  core/
    providers.ts              ← composition root (see below)
```

**Feature boundary rules:**
- Features import from `shared/` and `domain/` but NOT from other features' internal paths
- Cross-feature navigation is done via the router, not direct component imports
- `index.ts` is the only public surface of a feature — treat internal paths as private

---

## Composition Root

Instantiate and wire all dependencies in one place. This is the only place where concrete classes are constructed.

```typescript
// core/providers.ts
import { HttpUserRepository } from '../repositories/http-user-repository'
import { UserService } from '../services/user-service'

const userRepository = new HttpUserRepository(import.meta.env.VITE_API_BASE_URL)
export const userService = new UserService(userRepository)
```

**Rules:**
- The composition root is the ONLY place that calls `new` on repository and service classes
- All other code receives dependencies via constructor, DI framework, or Context
- Tests create their own composition with mock implementations — no global `fetch` patching

---

## Smart vs Presentational Components

**Smart (Container) components:**
- Obtain data from the State layer (hooks, signals, stores)
- Handle routing and navigation
- Pass data down to presentational components via props/inputs
- Contain minimal rendering logic — primarily orchestration and composition

**Presentational (Dumb) components:**
- Receive all data via props/inputs
- Communicate up via callbacks/outputs
- No data fetching, no service injection for data concerns
- Fully testable in isolation with static props
- Freely reusable across different containers

**Rule:** If a component exceeds ~150 lines or reaches into more than one service/hook for data, evaluate a split into container + presentational components.
