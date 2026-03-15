# Python Standards

Implementation standards for Python projects, including FastAPI backends and Django applications.

## Purpose

These standards provide concrete implementations of universal engineering principles using Python-specific patterns, tools, and best practices aligned with PEP guidelines.

## Standards Index

| Standard | Description |
| -------- | ----------- |
| [Coding Standards](./coding-standards) | Python coding conventions, project structure, patterns |
| [API Design](./api-design) | FastAPI/Django API design, validation, serialization |

## Tech Stack Coverage

### Web Frameworks

- **FastAPI** - Modern async API framework
- **Django** - Full-stack web framework
- **Flask** - Lightweight microframework

### Data & Validation

- **Pydantic** - Data validation and serialization
- **SQLAlchemy** - Database ORM
- **Alembic** - Database migrations
- **Django ORM** - Django's built-in ORM

### Async & Performance

- **asyncio** - Async programming
- **uvicorn** - ASGI server
- **Celery** - Task queues

### Tooling

- **Poetry / pip** - Package management
- **Ruff** - Fast linter (replaces flake8, isort, black)
- **mypy** - Static type checking
- **pytest** - Testing framework
- **pytest-cov** - Coverage reporting

## Universal Standards Reference

These Python implementations build upon:

- [Security Guidelines](../governance/security-guidelines) - Authentication, authorization, input validation
- [Database Standards](../governance/database-standards) - Schema design, migrations, indexing, naming conventions
- [Testing Strategy](../quality/testing-strategy) - Unit, integration, E2E testing patterns
- [Observability Standards](../reliability/observability-standards) - Logging, metrics, tracing
- [Performance Budgets](../quality/performance-budgets) - API latency, response times
- [CI/CD Pipelines](../development/ci-cd-pipelines) - Deploy workflow templates, dependency caching

## Quick Reference

### FastAPI Project Structure

```text
src/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI application
│   ├── config.py            # Configuration
│   ├── dependencies.py      # Dependency injection
│   ├── api/
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── routes/
│   │   │   └── schemas/
│   │   └── deps.py
│   ├── core/
│   │   ├── __init__.py
│   │   ├── security.py
│   │   └── exceptions.py
│   ├── models/              # SQLAlchemy models
│   ├── schemas/             # Pydantic schemas
│   └── services/            # Business logic
├── tests/
├── alembic/                 # Migrations
├── pyproject.toml
└── README.md
```

### FastAPI Endpoint Pattern

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.api.deps import get_db
from app.schemas.user import UserCreate, UserResponse
from app.services.user import UserService

router = APIRouter(prefix="/users", tags=["users"])


@router.post(
    "/",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
)
async def create_user(
    user_in: UserCreate,
    db: AsyncSession = Depends(get_db),
) -> UserResponse:
    """Create a new user account."""
    service = UserService(db)
    user = await service.create(user_in)
    return UserResponse.model_validate(user)


@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
) -> UserResponse:
    """Get user by ID."""
    service = UserService(db)
    user = await service.get_by_id(user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found",
        )
    return UserResponse.model_validate(user)
```
