# Python Coding Standards

> Authoritative Python coding standards aligned with PEP 8 and PEP 20 for consistent, readable Python code.

## Purpose

Establish consistent Python coding practices that ensure maintainability, readability, and quality across all Python projects.

## Core Principles

1. **Explicit is better than implicit** - Code should be clear about intent
2. **Readability counts** - Code is read more than written
3. **Simple is better than complex** - Prefer straightforward solutions
4. **Type safety** - Use type hints for better tooling and documentation
5. **Test everything** - Comprehensive testing is mandatory

## Style Guide

### Formatting

Use **Ruff** for linting and formatting (replaces flake8, isort, black):

```toml
# pyproject.toml
[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
    "S",   # flake8-bandit (security)
]

[tool.ruff.lint.isort]
known-first-party = ["app"]
```

### Naming Conventions

| Element | Convention | Example |
| ------- | ---------- | ------- |
| Modules | lowercase_underscore | `user_service.py` |
| Classes | PascalCase | `UserService` |
| Functions | lowercase_underscore | `get_user_by_id` |
| Variables | lowercase_underscore | `user_count` |
| Constants | UPPERCASE_UNDERSCORE | `MAX_RETRIES` |
| Private | leading_underscore | `_internal_helper` |
| Type variables | Short, PascalCase | `T`, `UserT` |

### Type Hints

Always use type hints for function signatures:

```python
from collections.abc import Sequence
from typing import TypeVar

T = TypeVar("T")


# ✅ Good: Full type annotations
def process_users(
    users: Sequence[User],
    *,
    include_inactive: bool = False,
    limit: int | None = None,
) -> list[UserResponse]:
    """Process a sequence of users and return responses."""
    ...


# ✅ Good: Using modern union syntax (Python 3.10+)
def get_user(user_id: int) -> User | None:
    ...


# ❌ Bad: Missing type hints
def process_users(users, include_inactive=False, limit=None):
    ...
```

### Type Checking

Use **mypy** for static type checking:

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_ignores = true
disallow_untyped_defs = true

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
```

## Project Structure

### Standard Layout

```text
project/
├── src/
│   └── app/
│       ├── __init__.py
│       ├── main.py
│       ├── config.py
│       ├── api/
│       │   ├── __init__.py
│       │   └── v1/
│       │       ├── __init__.py
│       │       └── routes/
│       ├── core/
│       │   ├── __init__.py
│       │   ├── exceptions.py
│       │   └── security.py
│       ├── models/
│       │   ├── __init__.py
│       │   └── user.py
│       ├── schemas/
│       │   ├── __init__.py
│       │   └── user.py
│       └── services/
│           ├── __init__.py
│           └── user.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── unit/
│   └── integration/
├── alembic/
├── pyproject.toml
├── README.md
└── Dockerfile
```

### Package Management

Use **Poetry** for dependency management:

```toml
# pyproject.toml
[project]
name = "my-project"
version = "0.1.0"
description = "Project description"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.109.0",
    "uvicorn[standard]>=0.27.0",
    "sqlalchemy>=2.0.0",
    "pydantic>=2.5.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-cov>=4.1.0",
    "pytest-asyncio>=0.23.0",
    "mypy>=1.8.0",
    "ruff>=0.1.0",
]
```

## Code Patterns

### Async/Await

Prefer async for I/O-bound operations:

```python
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine


# ✅ Good: Async database operations
async def get_user_by_id(
    db: AsyncSession,
    user_id: int,
) -> User | None:
    result = await db.execute(
        select(User).where(User.id == user_id)
    )
    return result.scalar_one_or_none()


# ✅ Good: Async context manager
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        yield session
```

### Exception Handling

Use specific exceptions with proper hierarchy:

```python
# ✅ Good: Custom exception hierarchy
class AppError(Exception):
    """Base application error."""

    def __init__(self, message: str, code: str = "APP_ERROR") -> None:
        self.message = message
        self.code = code
        super().__init__(message)


class NotFoundError(AppError):
    """Resource not found."""

    def __init__(self, resource: str, identifier: str | int) -> None:
        super().__init__(
            message=f"{resource} with id '{identifier}' not found",
            code="NOT_FOUND",
        )


class ValidationError(AppError):
    """Validation failed."""

    def __init__(self, errors: list[str]) -> None:
        self.errors = errors
        super().__init__(
            message="Validation failed",
            code="VALIDATION_ERROR",
        )


# ✅ Good: Specific exception handling
async def get_user(user_id: int) -> User:
    user = await repository.get_by_id(user_id)
    if user is None:
        raise NotFoundError("User", user_id)
    return user
```

### Dependency Injection

Use FastAPI's dependency injection:

```python
from functools import lru_cache
from typing import Annotated

from fastapi import Depends


@lru_cache
def get_settings() -> Settings:
    return Settings()


def get_db_session(
    settings: Annotated[Settings, Depends(get_settings)],
) -> AsyncGenerator[AsyncSession, None]:
    engine = create_async_engine(settings.database_url)
    async with AsyncSession(engine) as session:
        yield session


# Usage in routes
@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: Annotated[AsyncSession, Depends(get_db_session)],
) -> UserResponse:
    ...
```

### Data Validation

Use **Pydantic** for data validation:

```python
from datetime import datetime
from pydantic import BaseModel, EmailStr, Field, field_validator


class UserCreate(BaseModel):
    """Schema for creating a user."""

    email: EmailStr
    name: str = Field(min_length=1, max_length=100)
    password: str = Field(min_length=8, max_length=128)

    @field_validator("name")
    @classmethod
    def name_must_not_be_empty(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Name cannot be blank")
        return v.strip()


class UserResponse(BaseModel):
    """Schema for user response."""

    id: int
    email: str
    name: str
    created_at: datetime

    model_config = {"from_attributes": True}
```

## Testing

### Test Structure

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

from app.main import app


@pytest.fixture
async def client() -> AsyncGenerator[AsyncClient, None]:
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac


@pytest.fixture
async def db_session() -> AsyncGenerator[AsyncSession, None]:
    # Create test database session
    ...
```

### Test Patterns

```python
# tests/unit/test_user_service.py
import pytest
from unittest.mock import AsyncMock

from app.services.user import UserService
from app.schemas.user import UserCreate


class TestUserService:
    """Tests for UserService."""

    @pytest.fixture
    def mock_repository(self) -> AsyncMock:
        return AsyncMock()

    @pytest.fixture
    def service(self, mock_repository: AsyncMock) -> UserService:
        return UserService(repository=mock_repository)

    async def test_create_user_success(
        self,
        service: UserService,
        mock_repository: AsyncMock,
    ) -> None:
        # Arrange
        user_data = UserCreate(
            email="test@example.com",
            name="Test User",
            password="securepass123",
        )
        mock_repository.create.return_value = User(id=1, **user_data.model_dump())

        # Act
        result = await service.create(user_data)

        # Assert
        assert result.id == 1
        assert result.email == "test@example.com"
        mock_repository.create.assert_called_once()
```

## Checklist

### Project Setup

- [ ] Python 3.12+ required
- [ ] pyproject.toml configured
- [ ] Ruff configured for linting/formatting
- [ ] mypy configured for type checking
- [ ] pytest configured for testing
- [ ] Pre-commit hooks set up

### Code Quality

- [ ] All functions have type hints
- [ ] Docstrings on public APIs
- [ ] No unused imports
- [ ] No unused variables
- [ ] No security issues (bandit)

### Testing

- [ ] Unit tests for business logic
- [ ] Integration tests for APIs
- [ ] Test coverage > 80%
- [ ] Async tests properly configured

## References

- [PEP 8 - Style Guide](https://peps.python.org/pep-0008/)
- [PEP 20 - The Zen of Python](https://peps.python.org/pep-0020/)
- [PEP 484 - Type Hints](https://peps.python.org/pep-0484/)
- [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [Ruff Documentation](https://docs.astral.sh/ruff/)
