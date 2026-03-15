# API Design: .NET Core Web API

> Authoritative API design standards for .NET Core Web API applications.

## Purpose

Establish consistent API design patterns for .NET Core applications following Microsoft's recommended practices and industry standards.

## Core Principles

1. **RESTful by default** - Follow REST conventions unless there's a compelling reason not to
2. **Strongly typed** - Use DTOs and response models for all endpoints
3. **Validation everywhere** - Use data annotations and FluentValidation
4. **Versioned APIs** - Support API versioning from day one
5. **OpenAPI documentation** - Auto-generate Swagger documentation
6. **Problem Details standard** - Use RFC 7807 for error responses

## Response Format Standards

### Success Response Models

```csharp
// Common/Models/ApiResponse.cs
public class ApiResponse<T>
{
    public T Data { get; set; } = default!;
}

public class PagedApiResponse<T>
{
    public IEnumerable<T> Items { get; set; } = Enumerable.Empty<T>();
    public PaginationInfo Pagination { get; set; } = new();
}

public class PaginationInfo
{
    public int Page { get; set; }
    public int Limit { get; set; }
    public int Total { get; set; }
    public int TotalPages { get; set; }
    public bool HasNext { get; set; }
    public bool HasPrev { get; set; }
}

public class DeleteResponse
{
    public string Message { get; set; } = string.Empty;
}
```

### DTOs

```csharp
// Features/Users/DTOs/UserDto.cs
public record UserDto
{
    public Guid Id { get; init; }
    public string Email { get; init; } = string.Empty;
    public string Name { get; init; } = string.Empty;
    public DateTime CreatedAt { get; init; }
    public DateTime? UpdatedAt { get; init; }
}

// Features/Users/DTOs/CreateUserRequest.cs
public record CreateUserRequest
{
    [Required]
    [EmailAddress]
    [MaxLength(256)]
    public string Email { get; init; } = string.Empty;

    [Required]
    [MinLength(2)]
    [MaxLength(100)]
    public string Name { get; init; } = string.Empty;

    [Required]
    [MinLength(8)]
    public string Password { get; init; } = string.Empty;
}

// Features/Users/DTOs/UpdateUserRequest.cs
public record UpdateUserRequest
{
    [MinLength(2)]
    [MaxLength(100)]
    public string? Name { get; init; }

    [EmailAddress]
    [MaxLength(256)]
    public string? Email { get; init; }
}

// Features/Users/DTOs/GetUsersRequest.cs
public record GetUsersRequest
{
    [Range(1, int.MaxValue)]
    public int Page { get; init; } = 1;

    [Range(1, 100)]
    public int Limit { get; init; } = 20;

    public string? Search { get; init; }

    public string Sort { get; init; } = "createdAt";

    [RegularExpression("^(asc|desc)$")]
    public string Order { get; init; } = "desc";
}
```

## Controller Implementation

### Full Controller Example

```csharp
// Controllers/UsersController.cs
[ApiController]
[Route("api/v{version:apiVersion}/users")]
[ApiVersion("1.0")]
[Produces("application/json")]
public class UsersController : ControllerBase
{
    private readonly IMediator _mediator;
    private readonly ILogger<UsersController> _logger;

    public UsersController(IMediator mediator, ILogger<UsersController> logger)
    {
        _mediator = mediator;
        _logger = logger;
    }

    /// <summary>
    /// Get all users with pagination
    /// </summary>
    [HttpGet]
    [ProducesResponseType(typeof(PagedApiResponse<UserDto>), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetUsers(
        [FromQuery] GetUsersRequest request,
        CancellationToken cancellationToken)
    {
        _logger.LogInformation("Getting users with page {Page}, limit {Limit}",
            request.Page, request.Limit);

        var query = new GetUsersQuery(request);
        var result = await _mediator.Send(query, cancellationToken);

        return Ok(result);
    }

    /// <summary>
    /// Get user by ID
    /// </summary>
    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetUser(
        Guid id,
        CancellationToken cancellationToken)
    {
        var query = new GetUserByIdQuery(id);
        var result = await _mediator.Send(query, cancellationToken);

        return Ok(new ApiResponse<UserDto> { Data = result });
    }

    /// <summary>
    /// Create a new user
    /// </summary>
    [HttpPost]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status409Conflict)]
    public async Task<IActionResult> CreateUser(
        [FromBody] CreateUserRequest request,
        CancellationToken cancellationToken)
    {
        var command = new CreateUserCommand(request);
        var result = await _mediator.Send(command, cancellationToken);

        return CreatedAtAction(
            nameof(GetUser),
            new { id = result.Id },
            new ApiResponse<UserDto> { Data = result });
    }

    /// <summary>
    /// Update an existing user
    /// </summary>
    [HttpPut("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> UpdateUser(
        Guid id,
        [FromBody] UpdateUserRequest request,
        CancellationToken cancellationToken)
    {
        var command = new UpdateUserCommand(id, request);
        var result = await _mediator.Send(command, cancellationToken);

        return Ok(new ApiResponse<UserDto> { Data = result });
    }

    /// <summary>
    /// Delete a user
    /// </summary>
    [HttpDelete("{id:guid}")]
    [ProducesResponseType(typeof(DeleteResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> DeleteUser(
        Guid id,
        CancellationToken cancellationToken)
    {
        var command = new DeleteUserCommand(id);
        await _mediator.Send(command, cancellationToken);

        return Ok(new DeleteResponse { Message = "User deleted successfully" });
    }
}
```

## Error Handling

### Problem Details Configuration

```csharp
// Program.cs
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Extensions["traceId"] =
            Activity.Current?.Id ?? context.HttpContext.TraceIdentifier;
        context.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow;
    };
});
```

### Global Exception Handler

```csharp
// Middleware/GlobalExceptionHandler.cs
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "Exception occurred: {Message}", exception.Message);

        var problemDetails = exception switch
        {
            NotFoundException e => new ProblemDetails
            {
                Status = StatusCodes.Status404NotFound,
                Title = "Not Found",
                Detail = e.Message,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.4"
            },
            ConflictException e => new ProblemDetails
            {
                Status = StatusCodes.Status409Conflict,
                Title = "Conflict",
                Detail = e.Message,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.8"
            },
            ValidationException e => new ValidationProblemDetails(
                e.Errors.GroupBy(x => x.PropertyName)
                    .ToDictionary(
                        g => g.Key,
                        g => g.Select(x => x.ErrorMessage).ToArray()))
            {
                Status = StatusCodes.Status400BadRequest,
                Title = "Validation Error",
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1"
            },
            _ => new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Title = "Server Error",
                Detail = "An unexpected error occurred",
                Type = "https://tools.ietf.org/html/rfc7231#section-6.6.1"
            }
        };

        problemDetails.Extensions["traceId"] =
            Activity.Current?.Id ?? httpContext.TraceIdentifier;
        problemDetails.Extensions["timestamp"] = DateTime.UtcNow;

        httpContext.Response.StatusCode = problemDetails.Status ?? 500;
        await httpContext.Response.WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}
```

## API Versioning

### Configuration

```csharp
// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
})
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});
```

### Usage

```csharp
[ApiController]
[Route("api/v{version:apiVersion}/users")]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
public class UsersController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IActionResult GetUsersV1() { /* ... */ }

    [HttpGet]
    [MapToApiVersion("2.0")]
    public IActionResult GetUsersV2() { /* ... */ }
}
```

## Swagger/OpenAPI Configuration

```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "API documentation"
    });

    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });

    // Include XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});
```

## FluentValidation Integration

```csharp
// Features/Users/Validators/CreateUserRequestValidator.cs
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MaximumLength(256);

        RuleFor(x => x.Name)
            .NotEmpty()
            .MinimumLength(2)
            .MaximumLength(100);

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .Matches("[A-Z]").WithMessage("Password must contain at least one uppercase letter")
            .Matches("[a-z]").WithMessage("Password must contain at least one lowercase letter")
            .Matches("[0-9]").WithMessage("Password must contain at least one digit");
    }
}

// Program.cs
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();
builder.Services.AddFluentValidationAutoValidation();
```

## HTTP Status Codes

| Operation | Success Code | Error Codes |
| --------- | ------------ | ----------- |
| GET (collection) | 200 OK | 400, 401, 403 |
| GET (single) | 200 OK | 400, 401, 403, 404 |
| POST | 201 Created | 400, 401, 403, 409 |
| PUT | 200 OK | 400, 401, 403, 404 |
| PATCH | 200 OK | 400, 401, 403, 404 |
| DELETE | 200 OK | 401, 403, 404 |

## Checklist

### Project Setup

- [ ] API versioning configured
- [ ] Swagger/OpenAPI configured
- [ ] Problem Details enabled
- [ ] Global exception handler registered
- [ ] CORS configured

### Controllers

- [ ] All controllers inherit from ControllerBase
- [ ] Route follows `/api/v{version}/{resource}` pattern
- [ ] ProducesResponseType attributes on all actions
- [ ] XML documentation comments
- [ ] CancellationToken passed through

### Validation

- [ ] DTOs use data annotations or FluentValidation
- [ ] All required fields marked
- [ ] Length constraints defined
- [ ] Custom validation where needed

### Documentation

- [ ] XML comments enabled in project
- [ ] All endpoints documented
- [ ] Authentication documented
- [ ] Example requests/responses

## References

- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [ASP.NET Core Web API Documentation](https://learn.microsoft.com/en-us/aspnet/core/web-api/)
- [RFC 7807 - Problem Details](https://tools.ietf.org/html/rfc7807)
- [Swashbuckle Documentation](https://github.com/domaindrivendev/Swashbuckle.AspNetCore)
- [FluentValidation Documentation](https://docs.fluentvalidation.net/)
