---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# TypeScript/JavaScript Patterns

> This file extends [common/patterns.md](../common/patterns.md) with TypeScript/JavaScript specific content.

## API Response Format

```typescript
interface ApiResponse<T> {
  success: boolean
  data?: T
  error?: string
  errorCode: number  // REQUIRED — HTTP status code (e.g., 200, 404, 500)
  meta?: {
    total: number
    page: number
    limit: number
  }
}
```

## Custom Hooks Pattern

```typescript
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}
```

## Repository Pattern

**Naming Convention**: Methods MUST start with entity type for grouping (e.g., `userFindById`, `userCreate`).

**Immutability**: Repository methods MUST NOT modify passed-in objects - always return new instances.

```typescript
// Generic repository interface (for cross-cutting operations)
interface Repository<T> {
  findAll(filters?: Filters): Promise<readonly T[]>
  findById(id: string): Promise<T | null>
  create(data: CreateDto): Promise<T>
  update(id: string, data: UpdateDto): Promise<T>
  delete(id: string): Promise<void>
}

// Entity-specific repository with entity-first naming
// Method names start with entity type for IDE autocomplete grouping
interface UserRepository extends Repository<User> {
  userFindByEmail(email: string): Promise<User | null>
  userFindActive(): Promise<readonly User[]>
  userCreate(data: CreateUserDto): Promise<User>
  userUpdate(id: string, data: UpdateUserDto): Promise<User>
}

// Implementation example with Prisma
class PrismaUserRepository implements UserRepository {
  constructor(private prisma: PrismaClient) {}

  async findAll(filters?: Filters): Promise<readonly User[]> {
    // Returns new array - no side effects
    return await this.prisma.user.findMany({ where: filters })
  }

  async findById(id: string): Promise<User | null> {
    return await this.prisma.user.findUnique({ where: { id } })
  }

  async userFindByEmail(email: string): Promise<User | null> {
    return await this.prisma.user.findUnique({ where: { email } })
  }

  async userFindActive(): Promise<readonly User[]> {
    return await this.prisma.user.findMany({ where: { isActive: true } })
  }

  async create(data: CreateUserDto): Promise<User> {
    // Returns new entity - no side effects on input
    return await this.prisma.user.create({ data })
  }

  async userCreate(data: CreateUserDto): Promise<User> {
    return await this.create(data)
  }

  async update(id: string, data: UpdateUserDto): Promise<User> {
    // Returns new entity - no side effects on input
    return await this.prisma.user.update({ where: { id }, data })
  }

  async userUpdate(id: string, data: UpdateUserDto): Promise<User> {
    return await this.update(id, data)
  }

  async delete(id: string): Promise<void> {
    await this.prisma.user.delete({ where: { id } })
  }
}
```

**Benefits of entity-first naming**:
- Methods grouped by entity in IDE autocomplete
- Easy to discover all User-related methods: `user...` shows all options
- Clear separation between different entity operations
