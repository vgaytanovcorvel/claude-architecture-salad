# C# Coding Style

> This file extends common/coding-style.md with C# specific content.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Immutability (CRITICAL)

ALWAYS use `record` types for DTOs and immutable models. NEVER mutate objects in-place:

```csharp
// CORRECT: Using records with 'with' expressions
public record UserDto(int Id, string Name, string Email);

var updated = original with { Email = "new@example.com" };

// WRONG: Mutable class with setters
public class UserDto 
{
    public int Id { get; set; }
    public string Name { get; set; }
}
original.Email = "new@example.com"; // ❌ Mutation
```

Prefer:
- `record` for DTOs, value objects, and immutable models
- `with` expressions for creating modified copies
- `readonly` collections (`IReadOnlyList<T>`, `IReadOnlyCollection<T>`)
- Collection expressions for initialization (C# 12+)

## Validation Strategy (CRITICAL)

ALWAYS use **FluentValidation** for request/DTO validation. NEVER use Data Annotation attributes on records — they clutter positional syntax and violate separation of concerns.

This keeps records clean and positional while making validation testable and composable:

```csharp
// CORRECT: Clean positional record + separate FluentValidation validator
public record CreateUserRequest(string Name, string Email, int Age, string Phone);

public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Email).NotEmpty().EmailAddress().MaximumLength(255);
        RuleFor(x => x.Age).InclusiveBetween(18, 120);
        RuleFor(x => x.Phone).NotEmpty().Matches(@"^\+\d{10,15}$");
    }
}

// WRONG: Data Annotations on positional record — hard to read, not testable
public record CreateUserRequest(
    [Required][MaxLength(100)] string Name,
    [Required][EmailAddress][MaxLength(255)] string Email,
    [Range(18, 120)] int Age,
    [Required][RegularExpression(@"^\+\d{10,15}$")] string Phone
);
```

Register FluentValidation in DI:
```csharp
// Program.cs
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();
```

## Naming Conventions

ALWAYS follow these C# naming standards:

- **PascalCase**: Classes, interfaces, methods, properties, public fields, namespaces
- **camelCase**: Local variables, parameters, private methods
- **_camelCase**: Private instance fields
- **IPascalCase**: Interfaces (prefix with `I`)
- **TPascalCase**: Type parameters (prefix with `T`)
- **SCREAMING_CASE**: Constants only when representing fixed values like magic numbers

```csharp
// CORRECT
public class UserService
{
    private readonly ILogger<UserService> _logger;
    private readonly DbContext _dbContext;
    
    public async Task<UserDto> GetUserAsync(int userId, CancellationToken cancellationToken)
    {
        var user = await _dbContext.Users.FindAsync(new object[] { userId }, cancellationToken);
        return MapToDto(user);
    }
}

// WRONG
public class userService  // ❌ Should be PascalCase
{
    private ILogger logger;  // ❌ Missing _ prefix
    
    public async Task<UserDto> get_user(int UserID)  // ❌ Should be PascalCase, camelCase param
    {
        // ...
    }
}
```

## Nullable Reference Types

ALWAYS enable nullable reference types in all projects:

```xml
<!-- In .csproj -->
<PropertyGroup>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

NEVER use null-forgiving operator (`!`) to suppress warnings. Fix the root cause instead:

```csharp
// CORRECT: Proper null handling
public string? FindUserEmail(int userId)
{
    var user = _dbContext.Users.Find(userId);
    return user?.Email;  // Returns null if user not found
}

// CORRECT: Validation with exception
public string GetUserEmail(int userId)
{
    var user = _dbContext.Users.Find(userId) 
        ?? throw new NotFoundException($"User {userId} not found");
    return user.Email;
}

// WRONG: Suppressing warning
public string GetUserEmail(int userId)
{
    var user = _dbContext.Users.Find(userId);
    return user!.Email;  // ❌ Null-forgiving operator hides potential bug
}
```

## LINQ Patterns

ALWAYS prefer LINQ over manual loops. Use method syntax over query syntax:

```csharp
// CORRECT: LINQ method syntax with clear intent
var activeUsers = await _dbContext.Users
    .Where(u => u.IsActive)
    .OrderBy(u => u.LastName)
    .ThenBy(u => u.FirstName)
    .Select(u => new UserDto(u.Id, u.Name, u.Email))
    .ToListAsync(cancellationToken);

// ACCEPTABLE: Query syntax for complex joins
var result = from user in _dbContext.Users
             join order in _dbContext.Orders on user.Id equals order.UserId
             where user.IsActive
             select new { user.Name, order.Total };

// WRONG: Manual loop
var activeUsers = new List<UserDto>();
foreach (var user in _dbContext.Users)  // ❌ Use LINQ Where()
{
    if (user.IsActive)
    {
        activeUsers.Add(new UserDto(user.Id, user.Name, user.Email));
    }
}
```

Always materialize with:
- `ToListAsync()` / `ToList()` for lists
- `FirstOrDefaultAsync()` / `FirstOrDefault()` for single items
- `AnyAsync()` / `Any()` for existence checks
- `CountAsync()` / `Count()` for counts

NEVER iterate deferred queries multiple times.

## Async/Await Patterns

ALWAYS use `async Task` or `async Task<T>`. NEVER use `async void` except for event handlers:

```csharp
// CORRECT: Async method returning Task<T>
public async Task<UserDto> GetUserAsync(int userId, CancellationToken cancellationToken)
{
    var user = await _dbContext.Users
        .FirstOrDefaultAsync(u => u.Id == userId, cancellationToken);
    
    return user is null 
        ? throw new NotFoundException($"User {userId} not found")
        : MapToDto(user);
}

// CORRECT: Event handler (only acceptable async void)
private async void OnButtonClick(object sender, EventArgs e)
{
    try
    {
        await ProcessDataAsync(default);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error processing data");
    }
}

// WRONG: Async void method
public async void ProcessUser(int userId)  // ❌ Should return Task
{
    await _dbContext.SaveChangesAsync();
}

// WRONG: Blocking on async
public UserDto GetUser(int userId)
{
    return GetUserAsync(userId).Result;  // ❌ Can cause deadlocks
}
```

ConfigureAwait guidance:
- Library code: Use `ConfigureAwait(false)` to avoid capturing context
- Application code (ASP.NET Core, console apps): No need for `ConfigureAwait`

## File Organization

ALWAYS use file-scoped namespaces and organize usings:

```csharp
// CORRECT: File-scoped namespace (C# 10+)
using Microsoft.EntityFrameworkCore;
using MyApp.Domain.Entities;

namespace MyApp.Infrastructure.Repositories;

public class UserRepository : IUserRepository
{
    // Implementation
}

// WRONG: Block-scoped namespace
using Microsoft.EntityFrameworkCore;

namespace MyApp.Infrastructure.Repositories
{  // ❌ Unnecessary nesting
    public class UserRepository : IUserRepository
    {
        // Implementation
    }
}
```

Use global usings in a dedicated file for common namespaces:

```csharp
// GlobalUsings.cs
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading.Tasks;
global using Microsoft.Extensions.Logging;
```

## No Console.WriteLine in Production

ALWAYS use `ILogger<T>` for logging. NEVER use `Console.WriteLine`:

```csharp
// CORRECT: Structured logging
public class UserService
{
    private readonly ILogger<UserService> _logger;

    public UserService(ILogger<UserService> logger)
    {
        _logger = logger;
    }

    public async Task<UserDto> GetUserAsync(int userId)
    {
        _logger.LogInformation("Fetching user {UserId}", userId);
        
        var user = await _dbContext.Users.FindAsync(userId);
        
        if (user is null)
        {
            _logger.LogWarning("User {UserId} not found", userId);
            throw new NotFoundException($"User {userId} not found");
        }
        
        return MapToDto(user);
    }
}

// WRONG: Console output in production code
public async Task<UserDto> GetUserAsync(int userId)
{
    Console.WriteLine($"Fetching user {userId}");  // ❌ Use ILogger
    // ...
}
```

## Pattern Matching

Use pattern matching for cleaner, more expressive code:

```csharp
// CORRECT: Switch expressions
public decimal CalculateDiscount(Customer customer) => customer.Tier switch
{
    CustomerTier.Gold => 0.20m,
    CustomerTier.Silver => 0.10m,
    CustomerTier.Bronze => 0.05m,
    _ => 0m
};

// CORRECT: Is patterns with type testing
if (result is UserDto { IsActive: true } user)
{
    _logger.LogInformation("Active user: {UserName}", user.Name);
}

// CORRECT: Null checking patterns
if (user is not null && user.Age >= 18)
{
    // Process adult user
}
```

## Collection Expressions (C# 12+)

Use collection expressions for cleaner initialization:

```csharp
// CORRECT: Collection expressions
int[] numbers = [1, 2, 3, 4, 5];
List<string> names = ["Alice", "Bob", "Charlie"];

// Spreading
int[] moreNumbers = [..numbers, 6, 7, 8];

// ACCEPTABLE: Traditional initialization (pre-C# 12)
var numbers = new[] { 1, 2, 3, 4, 5 };
var names = new List<string> { "Alice", "Bob", "Charlie" };
```

## File Size and Organization

Same as common/coding-style.md:
- 200-400 lines typical, 800 max
- Extract utilities from large classes
- Organize by feature/domain, not by type
- One public class per file (nested private classes allowed)
- Limit size of methods by splitting into smaller ones (over 8 lines needs justification)

## Code Quality Checklist

Before marking C# work complete:
- [ ] Nullable reference types enabled, no null-forgiving operators
- [ ] `record` types used for immutable models (positional syntax)
- [ ] FluentValidation used for request validation (no Data Annotations on records)
- [ ] LINQ used instead of manual loops
- [ ] `async Task` methods (not `async void`)
- [ ] `ILogger<T>` used (no `Console.WriteLine`)
- [ ] File-scoped namespaces
- [ ] Proper naming conventions (PascalCase, _camelCase)
- [ ] Pattern matching used where appropriate
- [ ] No deep nesting (>4 levels)
- [ ] Files focused (<800 lines)
