# Coding Standards: .NET Core Backend

> Authoritative coding standards for .NET Core backend applications.

## Purpose

Establish consistent coding practices for .NET Core applications following Microsoft's recommended patterns and industry best practices.

## Core Principles

1. **Clean Architecture** - Separate concerns into layers (Domain, Application, Infrastructure, Presentation)
2. **Dependency Injection** - Use built-in DI container for all dependencies
3. **Async by default** - Use async/await for all I/O operations
4. **Strong typing** - Leverage C#'s type system, avoid dynamic types
5. **Immutability** - Prefer immutable objects and records where appropriate
6. **SOLID principles** - Follow Single Responsibility, Open/Closed, etc.

## Project Structure

### Clean Architecture Layout

```
Solution/
├── src/
│   ├── Domain/                    # Enterprise business rules
│   │   ├── Entities/
│   │   ├── ValueObjects/
│   │   ├── Enums/
│   │   └── Exceptions/
│   ├── Application/               # Application business rules
│   │   ├── Common/
│   │   │   ├── Interfaces/
│   │   │   ├── Behaviors/
│   │   │   └── Exceptions/
│   │   ├── Features/
│   │   │   └── Users/
│   │   │       ├── Commands/
│   │   │       ├── Queries/
│   │   │       └── DTOs/
│   │   └── DependencyInjection.cs
│   ├── Infrastructure/            # External concerns
│   │   ├── Persistence/
│   │   │   ├── Configurations/
│   │   │   └── ApplicationDbContext.cs
│   │   ├── Services/
│   │   └── DependencyInjection.cs
│   └── WebApi/                    # Presentation layer
│       ├── Controllers/
│       ├── Filters/
│       ├── Middleware/
│       └── Program.cs
├── tests/
│   ├── Domain.UnitTests/
│   ├── Application.UnitTests/
│   ├── Infrastructure.IntegrationTests/
│   └── WebApi.IntegrationTests/
└── Solution.sln
```

## Standards Reference Table

| Category | Standard | Example |
| -------- | -------- | ------- |
| Namespaces | PascalCase, match folder structure | `MyApp.Application.Features.Users` |
| Classes | PascalCase, noun | `UserService`, `OrderController` |
| Interfaces | PascalCase with I prefix | `IUserRepository` |
| Methods | PascalCase, verb | `GetUserAsync`, `CreateOrder` |
| Properties | PascalCase | `FirstName`, `IsActive` |
| Private fields | camelCase with underscore | `_userRepository` |
| Parameters | camelCase | `userId`, `cancellationToken` |
| Constants | PascalCase | `MaxRetryCount` |
| Async methods | Suffix with Async | `GetUsersAsync` |

## Entity Framework Core

### Entity Configuration

```csharp
// Domain/Entities/User.cs
public class User
{
    public Guid Id { get; private set; }
    public string Email { get; private set; } = string.Empty;
    public string Name { get; private set; } = string.Empty;
    public DateTime CreatedAt { get; private set; }
    public DateTime? UpdatedAt { get; private set; }

    private User() { } // For EF Core

    public static User Create(string email, string name)
    {
        return new User
        {
            Id = Guid.NewGuid(),
            Email = email.ToLowerInvariant(),
            Name = name,
            CreatedAt = DateTime.UtcNow
        };
    }

    public void Update(string name)
    {
        Name = name;
        UpdatedAt = DateTime.UtcNow;
    }
}

// Infrastructure/Persistence/Configurations/UserConfiguration.cs
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.HasKey(u => u.Id);

        builder.Property(u => u.Email)
            .IsRequired()
            .HasMaxLength(256);

        builder.Property(u => u.Name)
            .IsRequired()
            .HasMaxLength(100);

        builder.HasIndex(u => u.Email)
            .IsUnique();
    }
}
```

### Repository Pattern

```csharp
// Application/Common/Interfaces/IUserRepository.cs
public interface IUserRepository
{
    Task<User?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<User?> GetByEmailAsync(string email, CancellationToken cancellationToken = default);
    Task<(IReadOnlyList<User> Users, int Total)> GetPagedAsync(
        int page,
        int pageSize,
        string? search = null,
        CancellationToken cancellationToken = default);
    Task AddAsync(User user, CancellationToken cancellationToken = default);
    void Update(User user);
    void Remove(User user);
}

// Infrastructure/Persistence/Repositories/UserRepository.cs
public class UserRepository : IUserRepository
{
    private readonly ApplicationDbContext _context;

    public UserRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<User?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        return await _context.Users
            .FirstOrDefaultAsync(u => u.Id == id, cancellationToken);
    }

    public async Task<(IReadOnlyList<User> Users, int Total)> GetPagedAsync(
        int page,
        int pageSize,
        string? search = null,
        CancellationToken cancellationToken = default)
    {
        var query = _context.Users.AsQueryable();

        if (!string.IsNullOrWhiteSpace(search))
        {
            query = query.Where(u =>
                u.Name.Contains(search) ||
                u.Email.Contains(search));
        }

        var total = await query.CountAsync(cancellationToken);
        var users = await query
            .OrderBy(u => u.Name)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(cancellationToken);

        return (users, total);
    }

    public async Task AddAsync(User user, CancellationToken cancellationToken = default)
    {
        await _context.Users.AddAsync(user, cancellationToken);
    }

    public void Update(User user)
    {
        _context.Users.Update(user);
    }

    public void Remove(User user)
    {
        _context.Users.Remove(user);
    }
}
```

## API Controllers

### Controller Pattern

```csharp
// WebApi/Controllers/UsersController.cs
[ApiController]
[Route("api/v1/[controller]")]
[Produces("application/json")]
public class UsersController : ControllerBase
{
    private readonly IMediator _mediator;

    public UsersController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpGet]
    [ProducesResponseType(typeof(GetUsersResponse), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetUsers(
        [FromQuery] GetUsersQuery query,
        CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(query, cancellationToken);
        return Ok(result);
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(GetUserResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetUser(
        Guid id,
        CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(new GetUserQuery(id), cancellationToken);
        return Ok(result);
    }

    [HttpPost]
    [ProducesResponseType(typeof(CreateUserResponse), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<IActionResult> CreateUser(
        [FromBody] CreateUserCommand command,
        CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(command, cancellationToken);
        return CreatedAtAction(nameof(GetUser), new { id = result.Id }, result);
    }

    [HttpPut("{id:guid}")]
    [ProducesResponseType(typeof(UpdateUserResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> UpdateUser(
        Guid id,
        [FromBody] UpdateUserCommand command,
        CancellationToken cancellationToken)
    {
        command = command with { Id = id };
        var result = await _mediator.Send(command, cancellationToken);
        return Ok(result);
    }

    [HttpDelete("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> DeleteUser(
        Guid id,
        CancellationToken cancellationToken)
    {
        await _mediator.Send(new DeleteUserCommand(id), cancellationToken);
        return Ok(new { message = "User deleted successfully" });
    }
}
```

## Error Handling

### Custom Exceptions

```csharp
// Domain/Exceptions/DomainException.cs
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
}

// Domain/Exceptions/NotFoundException.cs
public class NotFoundException : DomainException
{
    public NotFoundException(string entityName, object key)
        : base($"{entityName} with key '{key}' was not found.") { }
}

// Domain/Exceptions/ConflictException.cs
public class ConflictException : DomainException
{
    public ConflictException(string message) : base(message) { }
}
```

### Global Exception Handler

```csharp
// WebApi/Middleware/GlobalExceptionHandler.cs
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "An exception occurred: {Message}", exception.Message);

        var (statusCode, response) = exception switch
        {
            NotFoundException e => (StatusCodes.Status404NotFound, CreateResponse(e)),
            ConflictException e => (StatusCodes.Status409Conflict, CreateResponse(e)),
            ValidationException e => (StatusCodes.Status400BadRequest, CreateValidationResponse(e)),
            _ => (StatusCodes.Status500InternalServerError, CreateResponse(exception))
        };

        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(response, cancellationToken);

        return true;
    }

    private static object CreateResponse(Exception exception) => new
    {
        statusCode = 500,
        message = new[] { exception.Message },
        error = exception.GetType().Name,
        timestamp = DateTime.UtcNow
    };
}
```

## Logging Standards

```csharp
// Structured logging with semantic parameters
_logger.LogInformation("User {UserId} logged in from {IpAddress}", userId, ipAddress);

// Log levels
_logger.LogDebug("Processing request with parameters: {@Parameters}", parameters);
_logger.LogInformation("User {UserId} created successfully", userId);
_logger.LogWarning("Rate limit exceeded for {IpAddress}", ipAddress);
_logger.LogError(exception, "Failed to process payment for order {OrderId}", orderId);
_logger.LogCritical("Database connection lost");

// Never log sensitive data
// BAD: _logger.LogInformation("User login: {@User}", user);
// GOOD: _logger.LogInformation("User {UserId} logged in", user.Id);
```

## Test Coverage Requirements

| Layer | Minimum Coverage | Critical Path Coverage |
| ----- | ---------------- | ---------------------- |
| Domain | 90% | 95% |
| Application | 80% | 90% |
| Infrastructure | 60% | 80% |
| WebApi | 50% | 70% |

## Checklist

### Project Setup

- [ ] Solution structure follows Clean Architecture
- [ ] Dependency injection configured in each layer
- [ ] Global exception handling implemented
- [ ] Logging configured with structured logging
- [ ] Health checks configured

### Code Quality

- [ ] Nullable reference types enabled
- [ ] EditorConfig configured for consistent formatting
- [ ] Code analyzers enabled (StyleCop, SonarAnalyzer)
- [ ] Async methods use CancellationToken

### Database

- [ ] Entity configurations use Fluent API
- [ ] Migrations version controlled
- [ ] Repository pattern implemented
- [ ] Unit of Work pattern (if needed)

### API

- [ ] Controllers follow REST conventions
- [ ] Request validation implemented
- [ ] Response DTOs defined
- [ ] API versioning configured

## References

- [Microsoft .NET Documentation](https://learn.microsoft.com/en-us/dotnet/)
- [Clean Architecture by Jason Taylor](https://github.com/jasontaylordev/CleanArchitecture)
- [C# Coding Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions)
- [ASP.NET Core Best Practices](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/best-practices)
- [Entity Framework Core Documentation](https://learn.microsoft.com/en-us/ef/core/)
