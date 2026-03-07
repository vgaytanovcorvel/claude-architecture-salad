# C# - ASP.NET Core Backend Rules

> This file extends [csharp/coding-style.md](coding-style.md) and [csharp/patterns.md](patterns.md) with ASP.NET Core backend specifics.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Controller-Based APIs

Use the `[ApiController]` attribute to enable automatic model validation and binding behaviors.

### ApiController Attribute Benefits

**Automatically enabled features:**
- Model validation errors return 400 Bad Request automatically
- Binding source parameter inference (`[FromBody]`, `[FromQuery]`, etc.)
- Multipart/form-data request inference
- Problem details for error responses

**Example: CRUD Controller**

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController(
    IUserService userService,
    ILogger<UsersController> logger) : ControllerBase
{
    [HttpGet]
    [ProducesResponseType(typeof(ApiResponse<IReadOnlyList<UserDto>>), StatusCodes.Status200OK)]
    public async Task<ActionResult<ApiResponse<IReadOnlyList<UserDto>>>> GetUsers(
        [FromQuery] int page,
        [FromQuery] int limit,
        CancellationToken cancellationToken)
    {
        var users = await userService.GetUsersAsync(page, limit, cancellationToken);
        return Ok(ApiResponse<IReadOnlyList<UserDto>>.Ok(users));
    }

    [HttpGet("{id}")]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ApiResponse<UserDto>>> GetUser(
        int id,
        CancellationToken cancellationToken)
    {
        var user = await userService.GetUserByIdAsync(id, cancellationToken);

        if (user is null)
            return NotFound(ApiResponse<UserDto>.Fail($"User not found (UserId: {id}).", HttpStatusCode.NotFound));

        return Ok(ApiResponse<UserDto>.Ok(user));
    }

    [HttpPost]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<ApiResponse<UserDto>>> CreateUser(
        [FromBody] CreateUserRequest request,
        CancellationToken cancellationToken)
    {
        var user = await userService.CreateUserAsync(request, cancellationToken);

        return CreatedAtAction(
            nameof(GetUser),
            new { id = user.Id },
            ApiResponse<UserDto>.Ok(user));
    }

    [HttpPut("{id}")]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ApiResponse<UserDto>>> UpdateUser(
        int id,
        [FromBody] UpdateUserRequest request,
        CancellationToken cancellationToken)
    {
        var user = await userService.UpdateUserAsync(id, request, cancellationToken);
        return Ok(ApiResponse<UserDto>.Ok(user));
    }

    [HttpDelete("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    public async Task<ActionResult> DeleteUser(int id, CancellationToken cancellationToken)
    {
        await userService.DeleteUserAsync(id, cancellationToken);
        return NoContent();
    }
}
```

## Minimal APIs

Lightweight alternative to controllers for simple APIs and microservices.

### Endpoint Mapping

**Example: RESTful Minimal API**

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Middleware configuration
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

// Route groups for organization
var usersApi = app.MapGroup("/api/users")
    .WithTags("Users")
    .RequireAuthorization();

usersApi.MapGet("/", async (IUserService userService, int page, int limit) =>
{
    var users = await userService.GetUsersAsync(page, limit);
    return Results.Ok(ApiResponse<IReadOnlyList<UserDto>>.Ok(users));
})
.WithName("GetUsers")
.Produces<ApiResponse<IReadOnlyList<UserDto>>>(StatusCodes.Status200OK);

usersApi.MapGet("/{id}", async (int id, IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);
    
    return user is null
        ? Results.NotFound(ApiResponse<UserDto>.Fail($"User not found (UserId: {id}).", HttpStatusCode.NotFound))
        : Results.Ok(ApiResponse<UserDto>.Ok(user));
})
.WithName("GetUserById")
.Produces<ApiResponse<UserDto>>(StatusCodes.Status200OK)
.Produces<ApiResponse<UserDto>>(StatusCodes.Status404NotFound);

usersApi.MapPost("/", async (CreateUserRequest request, IUserService userService) =>
{
    var user = await userService.CreateUserAsync(request);
    
    return Results.CreatedAtRoute(
        "GetUserById",
        new { id = user.Id },
        ApiResponse<UserDto>.Ok(user));
})
.WithName("CreateUser")
.Produces<ApiResponse<UserDto>>(StatusCodes.Status201Created)
.Produces(StatusCodes.Status400BadRequest);

usersApi.MapPut("/{id}", async (int id, UpdateUserRequest request, IUserService userService) =>
{
    var user = await userService.UpdateUserAsync(id, request);
    return Results.Ok(ApiResponse<UserDto>.Ok(user));
})
.WithName("UpdateUser")
.Produces<ApiResponse<UserDto>>(StatusCodes.Status200OK);

usersApi.MapDelete("/{id}", async (int id, IUserService userService) =>
{
    await userService.DeleteUserAsync(id);
    return Results.NoContent();
})
.WithName("DeleteUser")
.Produces(StatusCodes.Status204NoContent);

app.Run();
```

### Request Validation with Filters

```csharp
// Custom validation filter
public class ValidationFilter<T>(IValidator<T> validator) : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var request = context.Arguments.OfType<T>().FirstOrDefault();

        if (request is null)
            return Results.BadRequest("Invalid request");

        var validationResult = await validator.ValidateAsync(request);

        if (!validationResult.IsValid)
        {
            return Results.BadRequest(new
            {
                Errors = validationResult.Errors.Select(e => new { e.PropertyName, e.ErrorMessage })
            });
        }

        return await next(context);
    }
}

// Usage
usersApi.MapPost("/", async (CreateUserRequest request, IUserService userService) =>
{
    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/api/users/{user.Id}", user);
})
.AddEndpointFilter<ValidationFilter<CreateUserRequest>>();
```

## When to Use Each

### Controllers

**Use for:**
- Large APIs with many endpoints (10+)
- Complex routing requirements
- Action filters and custom attributes
- Traditional MVC architecture familiarity
- Teams experienced with controller-based patterns

**Example scenario:** Enterprise resource management API with complex authorization rules

### Minimal APIs

**Use for:**
- Microservices with focused functionality
- Simple CRUD operations (5-10 endpoints)
- Performance-critical scenarios (lower overhead)
- Serverless/containerized deployments with small footprint
- Rapid prototyping

**Example scenario:** Lightweight product catalog API for an e-commerce platform

## EF Core Patterns

### DbContext Configuration

```csharp
public class ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
    : DbContext(options)
{
    public DbSet<User> Users => Set<User>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Entity configuration
        modelBuilder.Entity<User>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Email).IsRequired().HasMaxLength(255);
            entity.HasIndex(e => e.Email).IsUnique();
            
            // Relationships
            entity.HasMany(e => e.Orders)
                .WithOne(e => e.User)
                .HasForeignKey(e => e.UserId)
                .OnDelete(DeleteBehavior.Cascade);
        });

        // Apply all configurations from assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
    }
}

// Registration in Program.cs
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorNumbersToAdd: null);
            sqlOptions.CommandTimeout(30);
        }));
```

### Migration Workflow

```powershell
# Create migration
dotnet ef migrations add AddUserEmail

# Update database
dotnet ef database update

# Rollback to specific migration
dotnet ef database update PreviousMigration

# Remove last migration (if not applied)
dotnet ef migrations remove

# Generate SQL script
dotnet ef migrations script
```

### Query Performance Patterns

```csharp
// CORRECT: AsNoTracking for read-only queries
public async Task<IReadOnlyList<UserDto>> GetUsersAsync()
{
    return await _dbContext.Users
        .AsNoTracking()  // No change tracking overhead
        .Select(u => new UserDto(u.Id, u.Name, u.Email))
        .ToListAsync();
}

// CORRECT: Eager loading to avoid N+1 queries
public async Task<User?> GetUserWithOrdersAsync(int userId)
{
    return await _dbContext.Users
        .Include(u => u.Orders)
            .ThenInclude(o => o.Items)
        .FirstOrDefaultAsync(u => u.Id == userId);
}

// CORRECT: Projection to select only needed columns
public async Task<IReadOnlyList<UserSummaryDto>> GetUserSummariesAsync()
{
    return await _dbContext.Users
        .Select(u => new UserSummaryDto
        {
            Id = u.Id,
            Name = u.Name,
            OrderCount = u.Orders.Count
        })
        .ToListAsync();
}

// WRONG: Loading all data then filtering in memory
public async Task<IReadOnlyList<User>> GetActiveUsersWrong()
{
    var allUsers = await _dbContext.Users.ToListAsync();  // ❌ Loads all users
    return allUsers.Where(u => u.IsActive).ToList();  // ❌ Filters in memory
}
```

### Transaction Handling

```csharp
public async Task<Order> CreateOrderWithInventoryUpdateAsync(CreateOrderRequest request)
{
    using var transaction = await _dbContext.Database.BeginTransactionAsync();
    
    try
    {
        // Create order
        var order = new Order { UserId = request.UserId, Total = request.Total };
        _dbContext.Orders.Add(order);
        await _dbContext.SaveChangesAsync();
        
        // Update inventory
        var product = await _dbContext.Products.FindAsync(request.ProductId)
            ?? throw new NotFoundException($"Product not found (ProductId: {request.ProductId}).");
        
        product.Stock -= request.Quantity;
        await _dbContext.SaveChangesAsync();
        
        await transaction.CommitAsync();
        return order;
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

## Middleware Pipeline

**CRITICAL:** Middleware order matters. Use this sequence:

```csharp
var app = builder.Build();

// 1. Exception handling (first to catch all errors)
app.UseExceptionHandler("/error");
app.UseHsts();  // Production only

// 2. HTTPS redirection
app.UseHttpsRedirection();

// 3. Static files (before routing)
app.UseStaticFiles();

// 4. Routing
app.UseRouting();

// 5. CORS (after routing, before auth)
app.UseCors("AllowSpecificOrigin");

// 6. Authentication (before authorization)
app.UseAuthentication();

// 7. Authorization
app.UseAuthorization();

// 8. Response caching
app.UseResponseCaching();

// 9. Endpoints (last)
app.MapControllers();
// or app.MapGroup("/api/users");

app.Run();
```

### Custom Middleware Pattern

```csharp
public class RequestLoggingMiddleware(
    RequestDelegate next,
    ILogger<RequestLoggingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        logger.LogInformation(
            "Request started (Method: {Method}, Path: {Path}).",
            context.Request.Method,
            context.Request.Path);

        try
        {
            await next(context);
        }
        finally
        {
            stopwatch.Stop();
            logger.LogInformation(
                "Request completed (Method: {Method}, Path: {Path}, ElapsedMs: {ElapsedMs}, StatusCode: {StatusCode}).",
                context.Request.Method,
                context.Request.Path,
                stopwatch.ElapsedMilliseconds,
                context.Response.StatusCode);
        }
    }
}

// Extension method for registration
public static class RequestLoggingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestLoggingMiddleware>();
    }
}

// Usage in Program.cs
app.UseRequestLogging();
```

### Global Exception Handler

All unhandled exceptions are caught here, logged, and returned as a consistent `ApiResponse.Fail(...)` envelope. Controllers should **not** use try/catch for response shaping — let exceptions bubble up to this handler (see [common/logging.md](../common/logging.md)).

```csharp
// Program.cs
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exceptionHandlerFeature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionHandlerFeature?.Error;

        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
        logger.LogError(exception, "Unhandled exception.");

        var statusCode = exception switch
        {
            NotFoundException => StatusCodes.Status404NotFound,
            UnauthorizedAccessException => StatusCodes.Status401Unauthorized,
            _ => StatusCodes.Status500InternalServerError
        };

        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/json";

        var message = statusCode == StatusCodes.Status500InternalServerError
            ? "Internal server error"
            : exception?.Message ?? "An error occurred";

        await context.Response.WriteAsJsonAsync(
            ApiResponse<object>.Fail(message, (HttpStatusCode)statusCode));
    });
});
```

## Background Services

### IHostedService Pattern

```csharp
public class DataCleanupService(
    IServiceProvider serviceProvider,
    ILogger<DataCleanupService> logger) : BackgroundService
{
    private readonly TimeSpan _interval = TimeSpan.FromHours(24);

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("Data cleanup service started.");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await CleanupExpiredDataAsync(stoppingToken);
                await Task.Delay(_interval, stoppingToken);
            }
            catch (OperationCanceledException)
            {
                // Expected when stopping
                break;
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Error in data cleanup service.");
                await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);  // Retry after delay
            }
        }

        logger.LogInformation("Data cleanup service stopped.");
    }

    private async Task CleanupExpiredDataAsync(CancellationToken cancellationToken)
    {
        // Create scope to resolve scoped services (DbContext)
        using var scope = serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();

        var cutoffDate = DateTime.UtcNow.AddDays(-30);
        
        var expiredRecords = await dbContext.TempData
            .Where(t => t.CreatedAt < cutoffDate)
            .ToListAsync(cancellationToken);

        dbContext.TempData.RemoveRange(expiredRecords);
        await dbContext.SaveChangesAsync(cancellationToken);

        logger.LogInformation("Expired records cleaned up (Count: {Count}).", expiredRecords.Count);
    }
}

// Registration
builder.Services.AddHostedService<DataCleanupService>();
```

## Health Checks

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>()
    .AddUrlGroup(new Uri("https://api.example.com/health"), "External API")
    .AddCheck<CustomHealthCheck>("Custom Check");

// Custom health check
public class CustomHealthCheck(IUserRepository userRepository) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken)
    {
        try
        {
            var userCount = await userRepository.GetCountAsync(cancellationToken);
            
            return userCount > 0
                ? HealthCheckResult.Healthy($"Database has {userCount} users")
                : HealthCheckResult.Degraded("Database is empty");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database check failed", ex);
        }
    }
}

// Endpoint mapping
app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

## API Versioning

```csharp
// Install: Asp.Versioning.Http
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
});

// Controllers (no primary constructor needed when there are no dependencies)
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
public class UsersV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetUsers() => Ok(new[] { "User1", "User2" });
}

[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("2.0")]
public class UsersV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetUsers() => Ok(new[] { new { Id = 1, Name = "User1" } });
}
```

## Response Caching

```csharp
// Output caching (.NET 7+)
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => builder.Cache());
    options.AddPolicy("Expire30s", builder => builder.Expire(TimeSpan.FromSeconds(30)));
});

app.UseOutputCache();

// Usage
app.MapGet("/api/users", async (IUserService userService) =>
{
    var users = await userService.GetUsersAsync();
    return Results.Ok(users);
})
.CacheOutput("Expire30s");
```

## Swagger/OpenAPI

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "A sample ASP.NET Core Web API"
    });

    // Include XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

Enable XML documentation in `.csproj`:
```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn>
</PropertyGroup>
```

## Containerization

**Multi-stage Dockerfile:**

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj and restore dependencies
COPY ["MyApi/MyApi.csproj", "MyApi/"]
RUN dotnet restore "MyApi/MyApi.csproj"

# Copy everything else and build
COPY . .
WORKDIR "/src/MyApi"
RUN dotnet build "MyApi.csproj" -c Release -o /app/build

# Publish stage
FROM build AS publish
RUN dotnet publish "MyApi.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app

# Create non-root user
RUN adduser --disabled-password --gecos "" appuser && chown -R appuser /app
USER appuser

COPY --from=publish /app/publish .

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet", "MyApi.dll"]
```

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=MyDb;User=sa;Password=YourStrong@Passw0rd;
    depends_on:
      - db

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong@Passw0rd
    ports:
      - "1433:1433"
    volumes:
      - sqldata:/var/opt/mssql

volumes:
  sqldata:
```

## Backend Development Checklist

Before committing ASP.NET Core backend code:
- [ ] Controllers use `[ApiController]` attribute or minimal APIs use route groups
- [ ] All endpoints have proper `[ProducesResponseType]` attributes or `.Produces()` calls
- [ ] Database queries use `AsNoTracking()` for read-only operations
- [ ] Eager loading used to avoid N+1 queries
- [ ] Middleware pipeline in correct order
- [ ] Background services properly inject scoped services via `IServiceProvider`
- [ ] Health checks configured for database and external dependencies
- [ ] Swagger/OpenAPI documentation enabled in Development
- [ ] Multi-stage Dockerfile optimized for production
- [ ] No hardcoded connection strings (use `IConfiguration`)
- [ ] Global exception handler configured
- [ ] CancellationToken passed to all async methods
