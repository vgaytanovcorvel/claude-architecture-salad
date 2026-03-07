# C# Design Patterns

> This file extends common/patterns.md with C# specific content.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Repository Pattern

Encapsulate data access behind a consistent interface. Repository interfaces operate on **domain models** (persistence-ignorant classes defined in Abstractions), NOT on EF Core entity classes.

**Naming Convention** (Repositories only — does NOT apply to Services): Methods MUST start with entity type for grouping (e.g., `UserSingleByIdAsync`, `UserAddAsync`). Service interfaces use natural application-level naming (e.g., `GetUserByIdAsync`, `CreateUserAsync`).

**Single vs SingleOrDefault**: `Single` methods throw `NotFoundException` when the entity is not found. `SingleOrDefault` methods return `null`. Use `Single` when absence is exceptional; use `SingleOrDefault` when absence is a valid outcome.

**Domain vs Entity Separation (CRITICAL)**: ORM entity classes (with navigation properties, `[Table]` attributes, etc.) are defined in the Repository project as an implementation detail and MUST NOT leak outside it. Repository implementations map between domain models and ORM entities internally.

**Repository Contract**: Read methods use `AsNoTracking()` and return new domain model instances. Write methods accept domain models, map to EF Core entities, persist via change tracking internally, and return new domain model instances reflecting the saved state. EF Core's change tracking and mutation are confined to the repository implementation — callers only ever see plain domain objects.

```csharp
// Repository interface — operates on domain models, NOT EF Core entities
// Standalone interface (no generic IRepository<T> base — mapping makes that impractical)
// Method names start with entity type for IDE grouping
public interface IUserRepository
{
    // Single — throws NotFoundException when not found
    Task<User> UserSingleByIdAsync(int id, CancellationToken cancellationToken);
    Task<User> UserSingleByEmailAsync(string email, CancellationToken cancellationToken);

    // SingleOrDefault — returns null when not found
    Task<User?> UserSingleOrDefaultByIdAsync(int id, CancellationToken cancellationToken);
    Task<User?> UserSingleOrDefaultByEmailAsync(string email, CancellationToken cancellationToken);

    Task<IReadOnlyList<User>> UserGetAllAsync(CancellationToken cancellationToken);
    Task<IReadOnlyList<User>> UserGetActiveAsync(CancellationToken cancellationToken);
    Task<User> UserAddAsync(User user, CancellationToken cancellationToken);
    Task<User> UserUpdateAsync(User user, CancellationToken cancellationToken);
    Task UserDeleteAsync(int id, CancellationToken cancellationToken);
}

// EF Core entity — defined in Repository project, NOT in Abstractions
// Contains ORM concerns (navigation properties, EF attributes)
public class UserEntity
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public UserRole Role { get; set; }
    public bool IsActive { get; set; }
    public DateTime CreatedAtUtc { get; set; }
    public DateTime? UpdatedAtUtc { get; set; }
    public ICollection<TodoItemEntity> Todos { get; set; } = [];  // Navigation property — ORM only
}

// Repository implementation — maps between domain models and EF Core entities
public class UserRepository(ApplicationDbContext dbContext) : IUserRepository
{
    // Single — throws NotFoundException when not found
    public async Task<User> UserSingleByIdAsync(int id, CancellationToken cancellationToken)
    {
        return await UserSingleOrDefaultByIdAsync(id, cancellationToken)
            ?? throw new NotFoundException($"User not found (UserId: {id}).");
    }

    public async Task<User> UserSingleByEmailAsync(string email, CancellationToken cancellationToken)
    {
        return await UserSingleOrDefaultByEmailAsync(email, cancellationToken)
            ?? throw new NotFoundException($"User not found (Email: {email}).");
    }

    // SingleOrDefault — returns null when not found
    public async Task<User?> UserSingleOrDefaultByIdAsync(int id, CancellationToken cancellationToken)
    {
        var entity = await dbContext.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == id, cancellationToken);

        return entity is null ? null : MapToDomain(entity);
    }

    public async Task<User?> UserSingleOrDefaultByEmailAsync(string email, CancellationToken cancellationToken)
    {
        var entity = await dbContext.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Email == email, cancellationToken);

        return entity is null ? null : MapToDomain(entity);
    }

    public async Task<User> UserAddAsync(User user, CancellationToken cancellationToken)
    {
        var entity = MapToEntity(user);
        var entry = await dbContext.Users.AddAsync(entity, cancellationToken);
        await dbContext.SaveChangesAsync(cancellationToken);
        return MapToDomain(entry.Entity);
    }

    public async Task<User> UserUpdateAsync(User user, CancellationToken cancellationToken)
    {
        // Map domain model to a new EF entity, attach, and mark modified
        var entity = MapToEntity(user);
        dbContext.Users.Update(entity);
        await dbContext.SaveChangesAsync(cancellationToken);

        // Detach so tracked state doesn't leak; return a fresh domain model
        dbContext.Entry(entity).State = EntityState.Detached;
        return MapToDomain(entity);
    }

    // ... other methods follow the same pattern

    private static User MapToDomain(UserEntity entity)
    {
        return new User
        {
            Id = entity.Id,
            Email = entity.Email,
            PasswordHash = entity.PasswordHash,
            FirstName = entity.FirstName,
            LastName = entity.LastName,
            Role = entity.Role,
            IsActive = entity.IsActive,
            CreatedAtUtc = entity.CreatedAtUtc,
            UpdatedAtUtc = entity.UpdatedAtUtc
        };
    }

    private static UserEntity MapToEntity(User user)
    {
        return new UserEntity
        {
            Id = user.Id,
            Email = user.Email,
            PasswordHash = user.PasswordHash,
            FirstName = user.FirstName,
            LastName = user.LastName,
            Role = user.Role,
            IsActive = user.IsActive,
            CreatedAtUtc = user.CreatedAtUtc,
            UpdatedAtUtc = user.UpdatedAtUtc
        };
    }
}
```

DI Registration — wire inside the assembly's `Add{Feature}` extension method (see [Extension Method Patterns](#extension-method-patterns-for-service-registration)):
```csharp
// TodoApp.Repository/Infrastructure/PersistenceServiceCollectionExtensions.cs
services.AddScoped<IUserRepository, UserRepository>();
```

## FluentValidation Pattern

Use FluentValidation to keep record DTOs clean and positional. Validation lives in dedicated validator classes — never as Data Annotation attributes on records (see [csharp/coding-style.md](coding-style.md#validation-strategy-critical)).

```csharp
// Request records — clean positional syntax
public record CreateUserRequest(string Name, string Email, int Age);
public record UpdateUserRequest(string Name, string Email);

// Validators — one per request type
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Email).NotEmpty().EmailAddress().MaximumLength(255);
        RuleFor(x => x.Age).InclusiveBetween(18, 120);
    }
}

public class UpdateUserRequestValidator : AbstractValidator<UpdateUserRequest>
{
    public UpdateUserRequestValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Email).NotEmpty().EmailAddress().MaximumLength(255);
    }
}
```

### Dependent/Async Validation Rules

```csharp
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    // FluentValidation validators keep traditional constructors — rules must be defined
    // in the constructor body, which primary constructors do not provide.
    public CreateUserRequestValidator(IUserRepository userRepository)
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MustAsync(async (email, cancellation) =>
            {
                var existing = await userRepository.UserSingleOrDefaultByEmailAsync(email, cancellation);
                return existing is null;
            })
            .WithMessage("Email already exists");
    }
}
```

### DI Registration

Wire inside the assembly's `Add{Feature}` extension method (see [Extension Method Patterns](#extension-method-patterns-for-service-registration)):

```csharp
// Inside Add{Feature} extension method
services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();
```

For Minimal APIs, use the `ValidationFilter<T>` endpoint filter (see [csharp/backend.md](backend.md#request-validation-with-filters)). For controller-based APIs, `[ApiController]` triggers automatic model state validation — wire FluentValidation into the MVC pipeline:

```csharp
// Inside Add{Feature} extension method
services.AddFluentValidationAutoValidation();
```

## API Response Format

Use a consistent envelope for all API responses:

**CRITICAL**: `StatusCode` is ALWAYS required. Use `HttpStatusCode` enum values. The `Ok()` factory method defaults to `HttpStatusCode.OK`. The `Fail()` factory method requires an explicit status code.

```csharp
// Response envelope record — StatusCode is always required
public record ApiResponse<T>
{
    public bool Success { get; init; }
    public T? Data { get; init; }
    public string? Error { get; init; }
    public HttpStatusCode StatusCode { get; init; }

    public static ApiResponse<T> Ok(T data) =>
        new() { Success = true, Data = data, StatusCode = HttpStatusCode.OK };

    // Failure — StatusCode is mandatory (no overload without it)
    public static ApiResponse<T> Fail(string error, HttpStatusCode statusCode) =>
        new() { Success = false, Error = error, StatusCode = statusCode };
}

// Usage in controller — no try/catch; exceptions bubble to global handler
// (see csharp/backend.md Global Exception Handler)
[HttpGet("{id}")]
public async Task<ActionResult<ApiResponse<UserDto>>> GetUser(int id, CancellationToken cancellationToken)
{
    // Using entity-first method naming — SingleOrDefault returns null when not found
    var user = await userRepository.UserSingleOrDefaultByIdAsync(id, cancellationToken);

    if (user is null)
        return NotFound(ApiResponse<UserDto>.Fail($"User not found (UserId: {id}).", HttpStatusCode.NotFound));

    return Ok(ApiResponse<UserDto>.Ok(MapToDto(user)));
}
```

## Options Pattern

Use the Options pattern for strongly-typed configuration:

```csharp
// Configuration class
public class DatabaseOptions
{
    public const string SectionName = "Database";
    
    public string ConnectionString { get; set; } = string.Empty;
    public int MaxRetryCount { get; set; } = 3;
    public int CommandTimeout { get; set; } = 30;
}

public class EmailOptions
{
    public const string SectionName = "Email";
    
    public string SmtpServer { get; set; } = string.Empty;
    public int Port { get; set; } = 587;
    public string FromAddress { get; set; } = string.Empty;
}

// appsettings.json
{
  "Database": {
    "ConnectionString": "Server=localhost;Database=MyApp;",
    "MaxRetryCount": 3,
    "CommandTimeout": 30
  },
  "Email": {
    "SmtpServer": "smtp.example.com",
    "Port": 587,
    "FromAddress": "noreply@example.com"
  }
}

// Inside Add{Feature} extension method
services.Configure<DatabaseOptions>(
    configuration.GetSection(DatabaseOptions.SectionName));

services.Configure<EmailOptions>(
    configuration.GetSection(EmailOptions.SectionName));

// Usage in service with primary constructor
public class EmailService(
    IOptions<EmailOptions> emailOptions,
    ILogger<EmailService> logger)
{
    public async Task SendEmailAsync(string to, string subject, string body, CancellationToken cancellationToken)
    {
        logger.LogInformation(
            "Sending email (To: {To}, SmtpServer: {SmtpServer}, Port: {Port}).",
            to,
            emailOptions.Value.SmtpServer,
            emailOptions.Value.Port);

        // Send email using emailOptions.Value
    }
}
```

For validation, use `IValidateOptions<T>`:

```csharp
public class DatabaseOptionsValidator : IValidateOptions<DatabaseOptions>
{
    public ValidateOptionsResult Validate(string? name, DatabaseOptions options)
    {
        if (string.IsNullOrWhiteSpace(options.ConnectionString))
            return ValidateOptionsResult.Fail("ConnectionString is required");
        
        if (options.MaxRetryCount < 0)
            return ValidateOptionsResult.Fail("MaxRetryCount must be >= 0");
        
        return ValidateOptionsResult.Success;
    }
}

// Inside Add{Feature} extension method
services.AddSingleton<IValidateOptions<DatabaseOptions>, DatabaseOptionsValidator>();
```

## Service Layer Pattern

Separate business logic from controllers and data access:

```csharp
// Service interface — operates on domain models, not DTOs
public interface IUserService
{
    Task<User?> GetUserByIdAsync(int id, CancellationToken cancellationToken);
    Task<User> CreateUserAsync(CreateUserRequest request, CancellationToken cancellationToken);
    Task<User> UpdateUserAsync(int id, UpdateUserRequest request, CancellationToken cancellationToken);
    Task DeleteUserAsync(int id, CancellationToken cancellationToken);
}

// Service implementation with primary constructor
public class UserService(
    IUserRepository userRepository) : IUserService
{
    public virtual async Task<User?> GetUserByIdAsync(int id, CancellationToken cancellationToken)
    {
        return await userRepository.UserSingleOrDefaultByIdAsync(id, cancellationToken);
    }

    public virtual async Task<User> CreateUserAsync(CreateUserRequest request, CancellationToken cancellationToken)
    {
        // Business logic: Check for duplicate email — SingleOrDefault since absence is expected
        var existingUser = await userRepository.UserSingleOrDefaultByEmailAsync(request.Email, cancellationToken);
        if (existingUser is not null)
            throw new DuplicateEmailException($"Email {request.Email} already exists");

        var user = new User { Name = request.Name, Email = request.Email };
        return await userRepository.UserAddAsync(user, cancellationToken);
    }

    public virtual async Task<User> UpdateUserAsync(int id, UpdateUserRequest request, CancellationToken cancellationToken)
    {
        // Single — throws NotFoundException if user doesn't exist
        var user = await userRepository.UserSingleByIdAsync(id, cancellationToken);

        var updated = user with { Name = request.Name, Email = request.Email };
        return await userRepository.UserUpdateAsync(updated, cancellationToken);
    }

    public virtual async Task DeleteUserAsync(int id, CancellationToken cancellationToken)
    {
        await userRepository.UserDeleteAsync(id, cancellationToken);
    }
}
```

## Extension Method Patterns for Service Registration

Each assembly exposes its own DI wiring via an `IServiceCollection` extension method.

**Conventions**:
- **Namespace**: Always `Microsoft.Extensions.DependencyInjection` — this is the standard .NET convention so callers discover the method without extra `using` statements
- **Method name**: `Add{Feature}(...)` — describes what capability the assembly adds (e.g., `AddPersistence`, `AddUserServices`, `AddWebCore`)
- **File location**: `Infrastructure/{Feature}ServiceCollectionExtensions.cs` inside the assembly
- **Class name**: `{Feature}ServiceCollectionExtensions`

```csharp
// TodoApp.Repository/Infrastructure/PersistenceServiceCollectionExtensions.cs
using Microsoft.EntityFrameworkCore;

namespace Microsoft.Extensions.DependencyInjection;

public static class PersistenceServiceCollectionExtensions
{
    public static IServiceCollection AddPersistence(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection")));

        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IOrderRepository, OrderRepository>();

        return services;
    }
}

// TodoApp.Implementation/Infrastructure/UserServiceCollectionExtensions.cs
namespace Microsoft.Extensions.DependencyInjection;

public static class UserServiceCollectionExtensions
{
    public static IServiceCollection AddUserServices(this IServiceCollection services)
    {
        services.AddScoped<IUserService, UserService>();

        return services;
    }
}

// Program.cs — callers compose features without extra using statements
builder.Services
    .AddUserServices()
    .AddPersistence(builder.Configuration);
```

## Result Pattern (Alternative to Exceptions)

For operations where failure is expected, use a Result type instead of exceptions:

```csharp
public record Result<T>
{
    public bool IsSuccess { get; init; }
    public T? Value { get; init; }
    public string? Error { get; init; }

    public static Result<T> Success(T value) =>
        new() { IsSuccess = true, Value = value };

    public static Result<T> Failure(string error) =>
        new() { IsSuccess = false, Error = error };
}

// Usage — service returns domain models, not DTOs
public async Task<Result<User>> CreateUserAsync(CreateUserRequest request)
{
    var existingUser = await _userRepository.UserSingleOrDefaultByEmailAsync(request.Email);
    if (existingUser is not null)
        return Result<User>.Failure($"Email {request.Email} already exists");

    var user = new User { Name = request.Name, Email = request.Email };
    var created = await _userRepository.UserAddAsync(user);

    return Result<User>.Success(created);
}

// In controller — DTO mapping happens here (presentation concern)
[HttpPost]
public async Task<ActionResult<ApiResponse<UserDto>>> CreateUser(CreateUserRequest request)
{
    var result = await _userService.CreateUserAsync(request);

    return result.IsSuccess
        ? Ok(ApiResponse<UserDto>.Ok(MapToDto(result.Value!)))
        : BadRequest(ApiResponse<UserDto>.Fail(result.Error!, HttpStatusCode.BadRequest));
}
```

## Specification Pattern (For Complex Queries)

Encapsulate query logic in reusable specifications:

```csharp
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    Expression<Func<T, object>>? OrderBy { get; }
}

public class BaseSpecification<T>(Expression<Func<T, bool>> criteria) : ISpecification<T>
{
    public Expression<Func<T, bool>> Criteria { get; } = criteria;
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public Expression<Func<T, object>>? OrderBy { get; private set; }

    protected void AddInclude(Expression<Func<T, object>> includeExpression)
    {
        Includes.Add(includeExpression);
    }

    protected void ApplyOrderBy(Expression<Func<T, object>> orderByExpression)
    {
        OrderBy = orderByExpression;
    }
}

// Example specification
// Note: Subclasses that need constructor body logic (e.g., calling AddInclude)
// keep traditional constructors since primary constructors have no body.
public class ActiveUsersSpecification : BaseSpecification<User>
{
    public ActiveUsersSpecification() : base(u => u.IsActive)
    {
        AddInclude(u => u.Orders);
        ApplyOrderBy(u => u.LastName);
    }
}

// Repository method
public async Task<IReadOnlyList<T>> GetAsync(ISpecification<T> spec)
{
    var query = _dbSet.AsQueryable();
    
    query = query.Where(spec.Criteria);
    
    query = spec.Includes.Aggregate(query, (current, include) => 
        current.Include(include));
    
    if (spec.OrderBy is not null)
        query = query.OrderBy(spec.OrderBy);
    
    return await query.ToListAsync();
}
```

These patterns provide a solid foundation for building maintainable, testable C# applications with clear separation of concerns.
