# .NET Standards

Implementation standards for .NET Core projects, including ASP.NET Core APIs and Entity Framework.

## Purpose

These standards provide concrete implementations of universal engineering principles using .NET-specific patterns, tools, and best practices.

## Standards Index

| Standard | Description |
| -------- | ----------- |
| [Coding Standards](./coding-standards) | C# coding conventions, project structure, patterns |
| [API Design](./api-design) | ASP.NET Core API design, controllers, validation |

## Tech Stack Coverage

### Core Framework

- **.NET 8+** - Latest LTS runtime
- **ASP.NET Core** - Web framework
- **C# 12+** - Latest language features
- **Entity Framework Core** - Database ORM

### API Development

- **Minimal APIs / Controllers** - Endpoint patterns
- **FluentValidation** - Request validation
- **MediatR** - CQRS pattern
- **AutoMapper** - Object mapping

### Database

- **Entity Framework Core** - ORM
- **PostgreSQL / SQL Server** - Database engines
- **Dapper** - Micro ORM for complex queries

### Tooling

- **dotnet CLI** - Build and run
- **NuGet** - Package management
- **xUnit** - Testing framework
- **Moq / NSubstitute** - Mocking
- **StyleCop** - Code analysis

## Universal Standards Reference

These .NET implementations build upon:

- [Security Guidelines](../governance/security-guidelines) - Authentication, authorization, input validation
- [Database Standards](../governance/database-standards) - Schema design, migrations, indexing, naming conventions
- [Testing Strategy](../quality/testing-strategy) - Unit, integration, E2E testing patterns
- [Observability Standards](../reliability/observability-standards) - Logging, metrics, tracing
- [Performance Budgets](../quality/performance-budgets) - API latency, response times
- [CI/CD Pipelines](../development/ci-cd-pipelines) - .NET deploy workflow template, NuGet caching

## Quick Reference

### Solution Structure

```text
src/
├── Company.Project.Api/           # ASP.NET Core Web API
├── Company.Project.Application/   # Business logic, CQRS
├── Company.Project.Domain/        # Domain entities, interfaces
├── Company.Project.Infrastructure/# Database, external services
└── Company.Project.Shared/        # Cross-cutting concerns

tests/
├── Company.Project.Api.Tests/
├── Company.Project.Application.Tests/
└── Company.Project.Integration.Tests/
```

### Controller Pattern

```csharp
[ApiController]
[Route("api/v1/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IMediator _mediator;

    public UsersController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(UserDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(Guid id)
    {
        var result = await _mediator.Send(new GetUserQuery(id));
        return result is null ? NotFound() : Ok(result);
    }
}
```
