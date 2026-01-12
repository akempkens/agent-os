## Python API Standards and Conventions

### RESTful API Design
- **Resource-Based URLs**: Use nouns for resources, not verbs: `/users`, `/posts`, not `/getUsers`
- **HTTP Methods**: Use appropriate HTTP methods for operations:
  - GET: Retrieve resources
  - POST: Create new resources
  - PUT: Replace entire resource
  - PATCH: Partial update
  - DELETE: Remove resource
- **Plural Nouns**: Use plural nouns for collections: `/users`, `/products`
- **Nested Resources**: Limit nesting to 2-3 levels: `/users/123/posts/456`

### FastAPI Best Practices
- **Router Organization**: Organize endpoints using APIRouter for modularity
- **Dependency Injection**: Use FastAPI's dependency injection for database sessions, authentication
- **Automatic Documentation**: Leverage FastAPI's automatic OpenAPI/Swagger documentation
- **Async by Default**: Use async functions for I/O-bound operations

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter(prefix="/api/v1/users", tags=["users"])

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    session: AsyncSession = Depends(get_db_session)
) -> UserResponse:
    """Retrieve a user by ID."""
    user = await user_service.get_by_id(session, user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    return user
```

### API Versioning
- **URL Versioning**: Version APIs in URL path: `/api/v1/users`, `/api/v2/users`
- **Version from Day One**: Include versioning from the beginning
- **Semantic Versioning**: Use semantic versioning for API versions
- **Deprecation Notices**: Announce deprecation well in advance with sunset dates

### Request/Response Models
- **Pydantic Models**: Use Pydantic models for all request and response validation
- **Separate Schemas**: Use different models for create, update, and response operations
- **response_model**: Always specify `response_model` to validate and filter responses
- **Consistent Structure**: Maintain consistent JSON structure across endpoints

```python
from pydantic import BaseModel, EmailStr, Field

class UserCreate(BaseModel):
    """Schema for creating a new user."""
    email: EmailStr
    username: str = Field(..., min_length=3, max_length=50)
    password: str = Field(..., min_length=12)

class UserUpdate(BaseModel):
    """Schema for updating user information."""
    email: EmailStr | None = None
    username: str | None = Field(None, min_length=3, max_length=50)
    bio: str | None = Field(None, max_length=500)

class UserResponse(BaseModel):
    """Schema for user responses (excludes password)."""
    id: int
    email: EmailStr
    username: str
    created_at: datetime

    model_config = {"from_attributes": True}
```

### HTTP Status Codes
- **Appropriate Status Codes**: Use correct HTTP status codes for responses:
  - 200 OK: Successful GET, PUT, PATCH
  - 201 Created: Successful POST
  - 204 No Content: Successful DELETE
  - 400 Bad Request: Invalid input
  - 401 Unauthorized: Not authenticated
  - 403 Forbidden: Authenticated but not authorized
  - 404 Not Found: Resource doesn't exist
  - 422 Unprocessable Entity: Validation error
  - 500 Internal Server Error: Server error
- **Consistent Usage**: Apply status codes consistently across all endpoints

### Pagination
- **Always Paginate Lists**: Paginate all list endpoints to prevent performance issues
- **Cursor or Offset**: Use cursor-based pagination for large datasets, offset for smaller
- **Pagination Metadata**: Include total count, page info in response
- **Configurable Page Size**: Allow clients to specify page size with reasonable limits

```python
from pydantic import BaseModel

class PaginatedResponse(BaseModel):
    """Generic paginated response."""
    items: list[UserResponse]
    total: int
    page: int
    page_size: int
    total_pages: int

@router.get("/", response_model=PaginatedResponse)
async def list_users(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    session: AsyncSession = Depends(get_db_session)
) -> PaginatedResponse:
    """List users with pagination."""
    offset = (page - 1) * page_size
    users = await user_service.list_users(session, offset=offset, limit=page_size)
    total = await user_service.count_users(session)

    return PaginatedResponse(
        items=users,
        total=total,
        page=page,
        page_size=page_size,
        total_pages=(total + page_size - 1) // page_size
    )
```

### Filtering and Sorting
- **Query Parameters**: Use query parameters for filtering: `/users?status=active&role=admin`
- **Consistent Naming**: Use consistent parameter names across endpoints
- **Sort Parameter**: Support sorting with `sort` parameter: `?sort=-created_at` (minus for descending)
- **Validate Filters**: Validate filter values to prevent injection attacks

```python
from enum import Enum

class SortOrder(str, Enum):
    ASC = "asc"
    DESC = "desc"

@router.get("/")
async def list_users(
    status: UserStatus | None = None,
    role: UserRole | None = None,
    sort_by: str = Query("created_at", regex="^(created_at|username|email)$"),
    sort_order: SortOrder = SortOrder.DESC,
    session: AsyncSession = Depends(get_db_session)
):
    """List users with filtering and sorting."""
    filters = {}
    if status:
        filters["status"] = status
    if role:
        filters["role"] = role

    users = await user_service.list_users(
        session,
        filters=filters,
        sort_by=sort_by,
        sort_order=sort_order
    )
    return users
```

### Error Handling and Responses
- **Consistent Error Format**: Use consistent error response structure
- **Meaningful Messages**: Provide clear, actionable error messages
- **Error Codes**: Include machine-readable error codes
- **Don't Expose Internals**: Never expose stack traces, SQL, or internal details

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

class APIError(BaseModel):
    """Standard API error response."""
    error: str
    message: str
    status_code: int
    details: dict[str, any] | None = None

@app.exception_handler(ValidationError)
async def validation_exception_handler(request: Request, exc: ValidationError):
    return JSONResponse(
        status_code=422,
        content={
            "error": "VALIDATION_ERROR",
            "message": "Input validation failed",
            "status_code": 422,
            "details": exc.errors()
        }
    )
```

### Authentication & Authorization
- **JWT Tokens**: Use JWT tokens for stateless authentication
- **OAuth2**: Implement OAuth2 with FastAPI's security utilities
- **Dependency Injection**: Use dependencies for authentication checks
- **Role-Based Access Control**: Implement RBAC for authorization

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    session: AsyncSession = Depends(get_db_session)
) -> User:
    """Verify JWT token and return current user."""
    token = credentials.credentials
    payload = verify_jwt_token(token)
    if not payload:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials"
        )

    user = await user_service.get_by_id(session, payload["user_id"])
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found"
        )
    return user

@router.get("/me", response_model=UserResponse)
async def get_current_user_profile(
    current_user: User = Depends(get_current_user)
) -> UserResponse:
    """Get current authenticated user's profile."""
    return current_user
```

### Rate Limiting
- **Implement Rate Limits**: Protect APIs with rate limiting
- **Rate Limit Headers**: Include rate limit info in response headers
- **Per-User Limits**: Apply different limits based on user tier
- **Library**: Use libraries like slowapi for rate limiting

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@router.get("/search")
@limiter.limit("10/minute")
async def search_users(request: Request, query: str):
    """Search users (rate limited to 10 requests per minute)."""
    pass
```

### CORS Configuration
- **Explicit Origins**: Never use `allow_origins=["*"]` in production
- **Credentials**: Set `allow_credentials=True` only if needed
- **Methods and Headers**: Explicitly allow only required methods and headers

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourdomain.com"],  # Explicit origins
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
)
```

### API Documentation
- **OpenAPI/Swagger**: FastAPI generates this automatically
- **Endpoint Descriptions**: Add descriptions to all endpoints
- **Tag Organization**: Group related endpoints with tags
- **Examples**: Provide request/response examples in schemas

```python
@router.post(
    "/",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Create a new user",
    description="Register a new user account with email and password",
    responses={
        201: {"description": "User created successfully"},
        400: {"description": "Invalid input"},
        409: {"description": "Email already registered"}
    }
)
async def create_user(
    user_data: UserCreate,
    session: AsyncSession = Depends(get_db_session)
) -> UserResponse:
    """Create a new user account."""
    pass
```

### Background Tasks
- **FastAPI BackgroundTasks**: Use for simple tasks that don't need reliability
- **Celery for Complex**: Use Celery for complex, distributed background jobs
- **Don't Block Requests**: Never block request handlers with slow operations
- **Task Status**: Provide endpoints to check background task status

```python
from fastapi import BackgroundTasks

def send_welcome_email(email: str):
    """Send welcome email to new user."""
    # Email sending logic
    pass

@router.post("/", status_code=201)
async def create_user(
    user_data: UserCreate,
    background_tasks: BackgroundTasks,
    session: AsyncSession = Depends(get_db_session)
):
    """Create user and send welcome email in background."""
    user = await user_service.create(session, user_data)

    # Send email in background
    background_tasks.add_task(send_welcome_email, user.email)

    return user
```

### Request Validation
- **Automatic Validation**: FastAPI validates requests automatically with Pydantic
- **Custom Validators**: Add custom validation for complex business rules
- **Query Parameters**: Validate query params with Field constraints
- **Path Parameters**: Validate path params with constraints

### API Testing
- **Test All Endpoints**: Write tests for all API endpoints
- **Use TestClient**: Use FastAPI's TestClient for synchronous tests
- **httpx for Async**: Use httpx.AsyncClient for async endpoint tests
- **Test Error Cases**: Test validation errors, authentication, and edge cases
