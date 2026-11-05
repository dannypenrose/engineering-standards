# Python API Design Standards

> Authoritative API design standards for Python web frameworks, aligned with OpenAPI best practices.

## Purpose

Establish consistent API design patterns for Python backends using FastAPI, Django REST Framework, or Flask, ensuring clean, well-documented, and maintainable APIs.

## Core Principles

1. **Contract-first** - Define API contracts before implementation
2. **Consistent conventions** - Follow REST principles consistently
3. **Self-documenting** - OpenAPI specs generated from code
4. **Validation at boundaries** - Validate all input data
5. **Explicit error handling** - Clear, actionable error responses

## FastAPI API Design

### Router Organization

```python
# app/api/v1/__init__.py
from fastapi import APIRouter

from .routes import health, users, products

api_router = APIRouter(prefix="/api/v1")
api_router.include_router(health.router, tags=["health"])
api_router.include_router(users.router, prefix="/users", tags=["users"])
api_router.include_router(products.router, prefix="/products", tags=["products"])
```

### Endpoint Patterns

```python
# app/api/v1/routes/users.py
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, Path, Query, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.api.deps import get_current_user, get_db
from app.models.user import User
from app.schemas.user import (
    UserCreate,
    UserResponse,
    UserUpdate,
    PaginatedUsers,
)
from app.services.user import UserService

router = APIRouter()


@router.get(
    "/",
    response_model=PaginatedUsers,
    summary="List users",
    description="Get a paginated list of all users.",
)
async def list_users(
    db: Annotated[AsyncSession, Depends(get_db)],
    page: Annotated[int, Query(ge=1)] = 1,
    page_size: Annotated[int, Query(ge=1, le=100)] = 20,
    search: str | None = None,
) -> PaginatedUsers:
    """List all users with pagination and optional search."""
    service = UserService(db)
    return await service.list(page=page, page_size=page_size, search=search)


@router.get(
    "/{user_id}",
    response_model=UserResponse,
    summary="Get user",
    responses={
        404: {"description": "User not found"},
    },
)
async def get_user(
    db: Annotated[AsyncSession, Depends(get_db)],
    user_id: Annotated[int, Path(ge=1)],
) -> UserResponse:
    """Get a specific user by ID."""
    service = UserService(db)
    user = await service.get_by_id(user_id)
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found",
        )
    return user


@router.post(
    "/",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Create user",
    responses={
        409: {"description": "Email already registered"},
    },
)
async def create_user(
    db: Annotated[AsyncSession, Depends(get_db)],
    user_in: UserCreate,
) -> UserResponse:
    """Create a new user account."""
    service = UserService(db)
    existing = await service.get_by_email(user_in.email)
    if existing:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="Email already registered",
        )
    return await service.create(user_in)


@router.patch(
    "/{user_id}",
    response_model=UserResponse,
    summary="Update user",
    responses={
        404: {"description": "User not found"},
    },
)
async def update_user(
    db: Annotated[AsyncSession, Depends(get_db)],
    user_id: Annotated[int, Path(ge=1)],
    user_in: UserUpdate,
    current_user: Annotated[User, Depends(get_current_user)],
) -> UserResponse:
    """Update a user's information."""
    service = UserService(db)
    user = await service.update(user_id, user_in)
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found",
        )
    return user


@router.delete(
    "/{user_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="Delete user",
    responses={
        404: {"description": "User not found"},
    },
)
async def delete_user(
    db: Annotated[AsyncSession, Depends(get_db)],
    user_id: Annotated[int, Path(ge=1)],
    current_user: Annotated[User, Depends(get_current_user)],
) -> None:
    """Delete a user."""
    service = UserService(db)
    deleted = await service.delete(user_id)
    if not deleted:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found",
        )
```

### Request/Response Schemas

```python
# app/schemas/user.py
from datetime import datetime

from pydantic import BaseModel, EmailStr, Field, ConfigDict


class UserBase(BaseModel):
    """Base user schema with common fields."""

    email: EmailStr
    name: str = Field(min_length=1, max_length=100)


class UserCreate(UserBase):
    """Schema for creating a user."""

    password: str = Field(min_length=8, max_length=128)


class UserUpdate(BaseModel):
    """Schema for updating a user. All fields optional."""

    email: EmailStr | None = None
    name: str | None = Field(None, min_length=1, max_length=100)


class UserResponse(UserBase):
    """Schema for user response."""

    id: int
    created_at: datetime
    updated_at: datetime

    model_config = ConfigDict(from_attributes=True)


class PaginatedUsers(BaseModel):
    """Paginated response for users."""

    items: list[UserResponse]
    total: int
    page: int
    page_size: int
    pages: int
```

### Error Handling

```python
# app/core/exceptions.py
from typing import Any

from fastapi import HTTPException, Request, status
from fastapi.responses import JSONResponse
from pydantic import ValidationError


class AppError(Exception):
    """Base application error."""

    def __init__(
        self,
        message: str,
        code: str = "APP_ERROR",
        status_code: int = status.HTTP_500_INTERNAL_SERVER_ERROR,
        details: dict[str, Any] | None = None,
    ) -> None:
        self.message = message
        self.code = code
        self.status_code = status_code
        self.details = details or {}
        super().__init__(message)


class NotFoundError(AppError):
    """Resource not found error."""

    def __init__(self, resource: str, identifier: str | int) -> None:
        super().__init__(
            message=f"{resource} not found",
            code="NOT_FOUND",
            status_code=status.HTTP_404_NOT_FOUND,
            details={"resource": resource, "id": str(identifier)},
        )


class ConflictError(AppError):
    """Resource conflict error."""

    def __init__(self, message: str, field: str | None = None) -> None:
        super().__init__(
            message=message,
            code="CONFLICT",
            status_code=status.HTTP_409_CONFLICT,
            details={"field": field} if field else {},
        )


# Error response schema
class ErrorResponse(BaseModel):
    """Standard error response format."""

    error: str
    code: str
    details: dict[str, Any] = {}


# Exception handlers
async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
    """Handle application errors."""
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": exc.message,
            "code": exc.code,
            "details": exc.details,
        },
    )


# Register handlers
def register_exception_handlers(app: FastAPI) -> None:
    app.add_exception_handler(AppError, app_error_handler)
```

### Authentication

```python
# app/api/deps.py
from typing import Annotated

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer

from app.core.security import decode_token
from app.models.user import User
from app.services.user import UserService

security = HTTPBearer()


async def get_current_user(
    credentials: Annotated[HTTPAuthorizationCredentials, Depends(security)],
    db: Annotated[AsyncSession, Depends(get_db)],
) -> User:
    """Get current authenticated user from JWT token."""
    token_data = decode_token(credentials.credentials)
    if token_data is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired token",
            headers={"WWW-Authenticate": "Bearer"},
        )

    service = UserService(db)
    user = await service.get_by_id(token_data.user_id)
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found",
        )
    return user


async def get_current_active_user(
    current_user: Annotated[User, Depends(get_current_user)],
) -> User:
    """Ensure current user is active."""
    if not current_user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Inactive user",
        )
    return current_user
```

### OpenAPI Documentation

```python
# app/main.py
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

from app.api.v1 import api_router


def custom_openapi() -> dict:
    """Generate custom OpenAPI schema."""
    if app.openapi_schema:
        return app.openapi_schema

    openapi_schema = get_openapi(
        title="My API",
        version="1.0.0",
        description="""
## Overview

This API provides access to...

## Authentication

All endpoints require Bearer token authentication.

## Rate Limiting

- 100 requests per minute for authenticated users
- 10 requests per minute for unauthenticated requests
        """,
        routes=app.routes,
    )

    # Add security scheme
    openapi_schema["components"]["securitySchemes"] = {
        "HTTPBearer": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT",
        }
    }

    app.openapi_schema = openapi_schema
    return app.openapi_schema


app = FastAPI(
    title="My API",
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc",
)
app.openapi = custom_openapi
app.include_router(api_router)
```

## Authorisation and Data Scope

Authentication says who the caller is; authorisation decides what they may see and change. Apply the [Authorisation and Data Scope](/standards/governance/authorisation-and-data-scope) standard to every endpoint. The rules most often missed:

- **One shared permission policy** evaluates every setting (tenant, record visibility, per-action levels, account-wide flags). Endpoints call it; none re-implement a rule.
- **Scope the query, not the response.** Lists, single records, related lists, counts, search, exports, reports and bulk actions all start from the policy's scoped set. Checking only the detail endpoint leaves the same records exposed through the list, the search box and the export.
- **Updates and deletes check the stored record's owners**, loaded through the scoped set, not the owners sent in the request.
- **Ownership matches every identity the user has** (for example a legacy id and a current id, or a primary and secondary owner).
- **Never take the tenant or acting user from input.** Derive them from the authenticated session. Resolve every id in the request, including ids nested in payloads, through the scoped set; narrow bulk actions to the ids the caller may act on.
- **Bypassing global query filters re-applies the tenant condition explicitly.**
- **Deny by default.** Unset permissions are the most restrictive value; a user whose record cannot be loaded gets nothing.
- **Status codes:** a hidden record is `404`; a refused action is `403` with a readable reason naming the setting; never `401`, which clients treat as signed out.
- **Exports need the export permission**, settings exports included.
- **Identifiers from configuration used in SQL** (column names) are validated against a strict allow-list; every value is parameterised.
- **Enforced by tests:** policy tests, enforcement tests with a restricted user, a two-tenant isolation suite generated from the route list and run against the real database engine, and architecture tests that fail the build when a route lacks its gate.

### Example: Scoped FastAPI Routes

The policy comes from `Depends(get_current_user)`, never from the request; every query starts from its scoped `select`, and the export route depends on the export permission.

```python
# app/api/v1/routes/contacts.py
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.api.deps import get_current_user, get_db, require_permission
from app.core.permissions import Permission, PermissionPolicy
from app.models.contact import Contact
from app.models.user import User
from app.schemas.contact import ContactResponse, ContactUpdate, PaginatedContacts

router = APIRouter()


async def get_policy(
    user: Annotated[User, Depends(get_current_user)],
    db: Annotated[AsyncSession, Depends(get_db)],
) -> PermissionPolicy:
    """Build the caller's policy from the session and a fresh read of their permissions."""
    return await PermissionPolicy.for_user(db, user)


@router.get("/", response_model=PaginatedContacts)
async def list_contacts(
    db: Annotated[AsyncSession, Depends(get_db)],
    policy: Annotated[PermissionPolicy, Depends(get_policy)],
    page: int = 1,
    page_size: int = 20,
) -> PaginatedContacts:
    """List the contacts the caller may see."""
    # Scope first, then filters, sorting and paging
    query = policy.visible(select(Contact))
    total = await db.scalar(select(func.count()).select_from(query.subquery()))
    rows = await db.scalars(query.offset((page - 1) * page_size).limit(page_size))
    return PaginatedContacts(items=list(rows), total=total or 0)


@router.put(
    "/{contact_id}",
    response_model=ContactResponse,
    responses={403: {"description": "Not allowed"}, 404: {"description": "Not found"}},
)
async def update_contact(
    contact_id: int,
    payload: ContactUpdate,
    db: Annotated[AsyncSession, Depends(get_db)],
    policy: Annotated[PermissionPolicy, Depends(get_policy)],
) -> ContactResponse:
    """Update a contact the caller may see and edit."""
    contact = await db.scalar(policy.visible(select(Contact)).where(Contact.id == contact_id))
    if contact is None:
        # Hidden and missing look the same
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Contact not found")

    # Check the stored owners, not the owners in the request
    if not policy.can_edit(contact):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail=(
                "Your permissions do not allow you to edit this contact. "
                "Ask an administrator to switch on Edit contacts."
            ),
        )

    contact.apply(payload)
    await db.commit()
    return ContactResponse.model_validate(contact)


@router.get(
    "/export",
    dependencies=[Depends(require_permission(Permission.EXPORT_RECORDS))],
)
async def export_contacts(
    db: Annotated[AsyncSession, Depends(get_db)],
    policy: Annotated[PermissionPolicy, Depends(get_policy)],
):
    """Export the contacts the caller may see."""
    rows = await db.scalars(policy.visible(select(Contact)))
    return csv_response(rows, filename="contacts.csv")
```

## Django REST Framework Patterns

### ViewSet Pattern

```python
# views/users.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

from .models import User
from .serializers import UserSerializer, UserCreateSerializer


class UserViewSet(viewsets.ModelViewSet):
    """ViewSet for user operations."""

    queryset = User.objects.all()
    permission_classes = [IsAuthenticated]

    def get_serializer_class(self):
        if self.action == 'create':
            return UserCreateSerializer
        return UserSerializer

    @action(detail=False, methods=['get'])
    def me(self, request):
        """Get current user."""
        serializer = self.get_serializer(request.user)
        return Response(serializer.data)
```

## Checklist

### API Design

- [ ] RESTful resource naming
- [ ] Proper HTTP methods used
- [ ] Appropriate status codes
- [ ] Pagination on list endpoints
- [ ] Filtering and sorting support

### Documentation

- [ ] OpenAPI spec generated
- [ ] Endpoint summaries and descriptions
- [ ] Request/response examples
- [ ] Error response documentation
- [ ] Authentication documented

### Security

- [ ] Input validation on all endpoints
- [ ] Authentication required where needed
- [ ] Authorization checks implemented
- [ ] Rate limiting configured
- [ ] CORS properly configured

### Authorisation and Data Scope

- [ ] Every endpoint applies the shared permission policy (lists, details, related lists, counts, search, exports, reports, bulk actions)
- [ ] Updates and deletes check the stored record's owners through the scoped set
- [ ] No tenant or user id taken from input; nested ids resolved through the scoped set
- [ ] Hidden records `404`, refused actions `403` with a reason, never `401`
- [ ] Export routes gated on the export permission
- [ ] Two-tenant isolation tests generated from the route list, run against the real database

## References

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [Django REST Framework](https://www.django-rest-framework.org/)
- [REST API Design](https://restfulapi.net/)
