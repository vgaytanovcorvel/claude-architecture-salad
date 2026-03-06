# C# Design Patterns

> This file extends common/patterns.md with C# specific content.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Repository Pattern

Encapsulate data access behind a consistent interface. Repository interfaces operate on **domain models** (persistence-ignorant classes defined in Abstractions), NOT on EF Core entity classes.

**Naming Convention** (Repositories only — does NOT apply to Services): Methods MUST start with entity type for grouping (e.g., `UserGetByIdAsync`, `UserAddAsync`). Service interfaces use natural application-level naming (e.g., `GetUserByIdAsync`, `CreateUserAsync`).

**Immutability**: Repository methods MUST NOT modify passed-in domain models - always return new instances.

**Domain vs Entity Separation**: ORM entity classes (with navigation properties, `[Table]` attributes, etc.) are defined in the Repository project as an implementation detail. Repository implementations map between domain models and ORM entities internally.

```csharp
// Repository interface — operates on domain models, NOT EF Core entities
// Standalone interface (no generic IRepository<T> base — mapping makes that impractical)
// Method names start with entity type for IDE grouping
public interface IUserRepository
{
    Task<User?> UserGetByIdAsync(int id, CancellationToken cancellationToken);
    Task<IReadOnlyList<User>> UserGetAllAsync(CancellationToken cancellationToken);
    Task<User?> UserGetByEmailAsync(string email, CancellationToken cancellationToken);
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
public class UserRepository : IUserRepository
{
    private readonly ApplicationDbContext _dbContext;

    public UserRepository(ApplicationDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task<User?> UserGetByIdAsync(int id, CancellationToken cancellationToken)
    {
        var entity = await _dbContext.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == id, cancellationToken);

        return entity is null ? null : MapToDomain(entity);
    }

    public async Task<User?> UserGetByEmailAsync(string email, CancellationToken cancellationToken)
    {
        var entity = await _dbContext.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Email == email, cancellationToken);

        return entity is null ? null : MapToDomain(entity);
    }

    public async Task<User> UserAddAsync(User user, CancellationToken cancellationToken)
    {
        var entity = MapToEntity(user);
        var entry = await _dbContext.Users.AddAsync(entity, cancellationToken);
        await _dbContext.SaveChangesAsync(cancellationToken);
        return MapToDomain(entry.Entity);
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

DI Registration:
```csharp
// Program.cs
builder.Services.AddScoped<IUserRepository, UserRepository>();
```

## API Response Format

Use a consistent envelope for all API responses:

**CRITICAL**: `ErrorCode` is ALWAYS required. Use `HttpStatusCode` enum values. The `Ok()` factory methods default to `HttpStatusCode.OK`. The `Fail()` factory method requires an explicit error code.

```csharp
// Response envelope record — ErrorCode is always required
public record ApiResponse<T>
{
    public bool Success { get; init; }
    public T? Data { get; init; }
    public string? Error { get; init; }
    public HttpStatusCode ErrorCode { get; init; }
    public ResponseMetadata? Meta { get; init; }

    // Success overloads (ErrorCode defaults to HttpStatusCode.OK)
    public static ApiResponse<T> Ok(T data) =>
        new() { Success = true, Data = data, ErrorCode = HttpStatusCode.OK };

    public static ApiResponse<T> Ok(T data, ResponseMetadata meta) =>
        new() { Success = true, Data = data, ErrorCode = HttpStatusCode.OK, Meta = meta };

    // Failure — ErrorCode is mandatory (no overload without it)
    public static ApiResponse<T> Fail(string error, HttpStatusCode errorCode) =>
        new() { Success = false, Error = error, ErrorCode = errorCode };
}

public record ResponseMetadata(int Total, int Page, int Limit);

// Usage in controller
[HttpGet("{id}")]
public async Task<ActionResult<ApiResponse<UserDto>>> GetUser(int id, CancellationToken cancellationToken)
{
    try
    {
        // Using entity-first method naming
        var user = await _userRepository.GetByIdAsync(id, cancellationToken);

        if (user is null)
            return NotFound(ApiResponse<UserDto>.Fail($"User {id} not found", HttpStatusCode.NotFound));

        var dto = _mapper.Map<UserDto>(user);
        return Ok(ApiResponse<UserDto>.Ok(dto));
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error fetching user {UserId}", id);
        return StatusCode(500, ApiResponse<UserDto>.Fail("Internal server error", HttpStatusCode.InternalServerError));
    }
}

// Paginated response
[HttpGet]
public async Task<ActionResult<ApiResponse<IReadOnlyList<UserDto>>>> GetUsers(
    [FromQuery] int page,
    [FromQuery] int limit,
    CancellationToken cancellationToken)
{
    var users = await _userRepository.GetAllAsync(cancellationToken);
    var total = users.Count;

    var paginatedUsers = users
        .Skip((page - 1) * limit)
        .Take(limit)
        .Select(u => _mapper.Map<UserDto>(u))
        .ToList();

    var meta = new ResponseMetadata(total, page, limit);
    return Ok(ApiResponse<IReadOnlyList<UserDto>>.Ok(paginatedUsers, meta));
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

// Registration in Program.cs
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection(DatabaseOptions.SectionName));
    
builder.Services.Configure<EmailOptions>(
    builder.Configuration.GetSection(EmailOptions.SectionName));

// Usage in service
public class EmailService
{
    private readonly EmailOptions _emailOptions;
    private readonly ILogger<EmailService> _logger;

    public EmailService(
        IOptions<EmailOptions> emailOptions,
        ILogger<EmailService> logger)
    {
        _emailOptions = emailOptions.Value;
        _logger = logger;
    }

    public async Task SendEmailAsync(string to, string subject, string body, CancellationToken cancellationToken)
    {
        _logger.LogInformation(
            "Sending email to {To} via {SmtpServer}:{Port}", 
            to, 
            _emailOptions.SmtpServer, 
            _emailOptions.Port);
        
        // Send email using _emailOptions
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

// Register validator
builder.Services.AddSingleton<IValidateOptions<DatabaseOptions>, DatabaseOptionsValidator>();
```

## Service Layer Pattern

Separate business logic from controllers and data access:

```csharp
// Service interface
public interface IUserService
{
    Task<UserDto?> GetUserByIdAsync(int id, CancellationToken cancellationToken);
    Task<UserDto> CreateUserAsync(CreateUserRequest request, CancellationToken cancellationToken);
    Task<UserDto> UpdateUserAsync(int id, UpdateUserRequest request, CancellationToken cancellationToken);
    Task DeleteUserAsync(int id, CancellationToken cancellationToken);
}

// Service implementation
public class UserService : IUserService
{
    private readonly IUserRepository _userRepository;
    private readonly IMapper _mapper;
    private readonly ILogger<UserService> _logger;

    public UserService(
        IUserRepository userRepository,
        IMapper mapper,
        ILogger<UserService> logger)
    {
        _userRepository = userRepository;
        _mapper = mapper;
        _logger = logger;
    }

    public async Task<UserDto?> GetUserByIdAsync(int id, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Fetching user {UserId}", id);
        
        var user = await _userRepository.GetByIdAsync(id, cancellationToken);
        return user is null ? null : _mapper.Map<UserDto>(user);
    }

    public async Task<UserDto> CreateUserAsync(CreateUserRequest request, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Creating user with email {Email}", request.Email);
        
        // Business logic: Check for duplicate email
        var existingUser = await _userRepository.GetByEmailAsync(request.Email, cancellationToken);
        if (existingUser is not null)
            throw new DuplicateEmailException($"Email {request.Email} already exists");
        
        var user = _mapper.Map<User>(request);
        var created = await _userRepository.AddAsync(user, cancellationToken);
        
        return _mapper.Map<UserDto>(created);
    }

    public async Task<UserDto> UpdateUserAsync(int id, UpdateUserRequest request, CancellationToken cancellationToken)
    {
        var user = await _userRepository.GetByIdAsync(id, cancellationToken)
            ?? throw new NotFoundException($"User {id} not found");
        
        _mapper.Map(request, user);
        await _userRepository.UpdateAsync(user, cancellationToken);
        
        return _mapper.Map<UserDto>(user);
    }

    public async Task DeleteUserAsync(int id, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Deleting user {UserId}", id);
        await _userRepository.DeleteAsync(id, cancellationToken);
    }
}
```

## Extension Method Patterns for Service Registration

Create extension methods to organize service registration:

```csharp
// Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Database
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection")));
        
        // Repositories
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IOrderRepository, OrderRepository>();
        
        return services;
    }
}

// Application/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        // Services
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<IOrderService, OrderService>();
        
        // AutoMapper
        services.AddAutoMapper(Assembly.GetExecutingAssembly());
        
        // MediatR (if using CQRS)
        services.AddMediatR(cfg => 
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));
        
        return services;
    }
}

// Program.cs
builder.Services
    .AddApplication()
    .AddInfrastructure(builder.Configuration);
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

// Usage
public async Task<Result<UserDto>> CreateUserAsync(CreateUserRequest request)
{
    var existingUser = await _userRepository.GetByEmailAsync(request.Email);
    if (existingUser is not null)
        return Result<UserDto>.Failure($"Email {request.Email} already exists");
    
    var user = _mapper.Map<User>(request);
    var created = await _userRepository.AddAsync(user);
    var dto = _mapper.Map<UserDto>(created);
    
    return Result<UserDto>.Success(dto);
}

// In controller
[HttpPost]
public async Task<ActionResult<ApiResponse<UserDto>>> CreateUser(CreateUserRequest request)
{
    var result = await _userService.CreateUserAsync(request);

    return result.IsSuccess
        ? Ok(ApiResponse<UserDto>.Ok(result.Value!))
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

public class BaseSpecification<T> : ISpecification<T>
{
    public Expression<Func<T, bool>> Criteria { get; }
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public Expression<Func<T, object>>? OrderBy { get; private set; }

    protected BaseSpecification(Expression<Func<T, bool>> criteria)
    {
        Criteria = criteria;
    }

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
