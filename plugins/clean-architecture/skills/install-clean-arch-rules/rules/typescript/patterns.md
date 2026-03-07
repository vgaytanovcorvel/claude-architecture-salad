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
  statusCode: number  // REQUIRED — HTTP status code (e.g., 200, 404, 500)
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

**Naming Convention** (Repositories only — does NOT apply to Services): Methods MUST start with entity type for grouping (e.g., `userSingleById`, `userCreate`). Service interfaces use natural application-level naming (e.g., `getUserById`, `createUser`).

**Single vs SingleOrDefault**: `Single` methods throw `NotFoundException` when the entity is not found. `SingleOrDefault` methods return `null`. Use `Single` when absence is exceptional; use `SingleOrDefault` when absence is a valid outcome.

**Immutability**: Repository methods MUST NOT modify passed-in objects - always return new instances.

```typescript
// Generic repository interface (for cross-cutting operations)
interface Repository<T> {
  findAll(filters?: Filters): Promise<readonly T[]>
  create(data: CreateDto): Promise<T>
  update(id: string, data: UpdateDto): Promise<T>
  delete(id: string): Promise<void>
}

// Entity-specific repository with entity-first naming
// Method names start with entity type for IDE autocomplete grouping
// Single throws NotFoundException; SingleOrDefault returns null
interface UserRepository extends Repository<User> {
  userSingleById(id: string): Promise<User>
  userSingleOrDefaultById(id: string): Promise<User | null>
  userSingleByEmail(email: string): Promise<User>
  userSingleOrDefaultByEmail(email: string): Promise<User | null>
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

  // Single — throws NotFoundException when not found
  async userSingleById(id: string): Promise<User> {
    const user = await this.userSingleOrDefaultById(id)
    if (!user) throw new NotFoundException(`User not found (UserId: ${id}).`)
    return user
  }

  async userSingleByEmail(email: string): Promise<User> {
    const user = await this.userSingleOrDefaultByEmail(email)
    if (!user) throw new NotFoundException(`User not found (Email: ${email}).`)
    return user
  }

  // SingleOrDefault — returns null when not found
  async userSingleOrDefaultById(id: string): Promise<User | null> {
    return await this.prisma.user.findUnique({ where: { id } })
  }

  async userSingleOrDefaultByEmail(email: string): Promise<User | null> {
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
- Easy to discover all User-related methods: `user...` shows all options (e.g., `userSingle...`, `userCreate`, `userUpdate`)
- Clear separation between different entity operations
