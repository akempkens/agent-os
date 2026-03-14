## Python Validation Best Practices

### Pydantic for Validation
- **Use Pydantic Models**: Use Pydantic for all data validation, serialization, and settings management
- **Type Safety**: Leverage Python type hints with Pydantic for automatic validation
- **Automatic Coercion**: Pydantic automatically coerces compatible types (e.g., string to int)
- **FastAPI Integration**: FastAPI uses Pydantic models for automatic request/response validation

### Input Validation Principles
- **Validate Early**: Validate all input at the system boundaries (API endpoints, CLI inputs)
- **Never Trust Input**: Always validate user input; never trust client-side validation alone
- **Fail Fast**: Reject invalid data immediately with clear error messages
- **Allowlists Over Blocklists**: Define what is allowed rather than trying to block everything invalid

### Pydantic Model Examples
```python
from pydantic import BaseModel, EmailStr, Field, field_validator
from datetime import datetime

class UserCreate(BaseModel):
    """Schema for creating a new user."""
    email: EmailStr  # Validates email format
    username: str = Field(..., min_length=3, max_length=50, pattern="^[a-zA-Z0-9_-]+$")
    age: int = Field(..., ge=18, le=120)  # Greater than or equal to 18
    bio: str | None = Field(None, max_length=500)

    @field_validator('username')
    @classmethod
    def username_alphanumeric(cls, v: str) -> str:
        if not v.replace('_', '').replace('-', '').isalnum():
            raise ValueError('Username must be alphanumeric with optional _ or -')
        return v.lower()  # Normalize to lowercase

class UserResponse(BaseModel):
    """Schema for user responses."""
    id: int
    email: EmailStr
    username: str
    created_at: datetime

    model_config = {"from_attributes": True}  # Enable ORM mode for SQLAlchemy
```

### Field Validation
- **Built-in Constraints**: Use Field constraints: `min_length`, `max_length`, `ge`, `le`, `gt`, `lt`, `regex`
- **Custom Validators**: Create custom validators with `@field_validator` for complex rules
- **Multiple Fields**: Use `@model_validator` for validation that depends on multiple fields
- **Type Annotations**: Always use proper type hints for automatic type validation

```python
from pydantic import BaseModel, model_validator

class DateRange(BaseModel):
    start_date: datetime
    end_date: datetime

    @model_validator(mode='after')
    def validate_date_range(self) -> 'DateRange':
        if self.end_date <= self.start_date:
            raise ValueError('end_date must be after start_date')
        return self
```

### API Request Validation
- **FastAPI Automatic Validation**: FastAPI automatically validates requests using Pydantic models
- **Path Parameters**: Validate path parameters with type hints and constraints
- **Query Parameters**: Use Pydantic models or Query() for complex query parameter validation
- **Request Body**: Always define Pydantic models for request bodies

```python
from fastapi import FastAPI, Query, Path, Body
from pydantic import BaseModel, Field

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(
    user_id: int = Path(..., ge=1, description="User ID must be positive"),
    include_posts: bool = Query(False, description="Include user's posts")
):
    # FastAPI validates user_id is int >= 1
    pass

@app.post("/users")
async def create_user(user: UserCreate = Body(...)):
    # FastAPI validates request body matches UserCreate schema
    pass
```

### Response Validation
- **Response Models**: Define Pydantic models for all API responses
- **response_model**: Use FastAPI's `response_model` parameter for automatic validation
- **Exclude Sensitive Data**: Use `response_model_exclude` or separate schemas to hide sensitive fields

```python
@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):
    user = await db.get_user(user_id)
    return user  # FastAPI validates response matches UserResponse
```

### Data Sanitization
- **HTML Escaping**: Escape HTML in user-generated content to prevent XSS
- **SQL Injection**: Use parameterized queries (ORM handles this automatically)
- **Path Traversal**: Validate file paths to prevent directory traversal attacks
- **Unicode Normalization**: Normalize unicode strings to prevent homograph attacks

```python
import html
from pathlib import Path

def sanitize_html(text: str) -> str:
    """Escape HTML special characters."""
    return html.escape(text)

def validate_filename(filename: str) -> Path:
    """Validate filename to prevent path traversal."""
    path = Path(filename)
    if path.is_absolute() or '..' in path.parts:
        raise ValueError("Invalid filename")
    return path
```

### Database Input Validation
- **ORM Protection**: Use ORM (SQLAlchemy, Django ORM) for automatic SQL injection protection
- **Parameterized Queries**: If using raw SQL, always use parameterized queries
- **Type Checking**: Validate types before database operations
- **Unique Constraints**: Enforce uniqueness at database level, not just application level

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

async def get_user_by_email(session: AsyncSession, email: str) -> User | None:
    # SQLAlchemy automatically parameterizes this query
    result = await session.execute(
        select(User).where(User.email == email)
    )
    return result.scalar_one_or_none()
```

### Settings and Configuration Validation
- **Pydantic Settings**: Use Pydantic's `BaseSettings` for environment variable validation
- **Required Settings**: Mark required settings without defaults
- **Type Validation**: Ensure environment variables are parsed to correct types
- **Secrets**: Use SecretStr for sensitive configuration values

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field, PostgresDsn, SecretStr

class Settings(BaseSettings):
    """Application settings loaded from environment variables."""
    model_config = SettingsConfigDict(env_file='.env', case_sensitive=False)

    # Required settings (no default)
    database_url: PostgresDsn
    secret_key: SecretStr

    # Optional settings with defaults
    debug: bool = False
    log_level: str = Field(default="INFO", pattern="^(DEBUG|INFO|WARNING|ERROR|CRITICAL)$")
    max_connections: int = Field(default=10, ge=1, le=100)

settings = Settings()
```

### File Upload Validation
- **File Type**: Validate file MIME types and extensions
- **File Size**: Enforce maximum file size limits
- **Content Validation**: Validate file content, not just extension
- **Virus Scanning**: Scan uploaded files for malware in production

```python
from fastapi import UploadFile, HTTPException
import magic

ALLOWED_MIME_TYPES = {"image/jpeg", "image/png", "image/gif"}
MAX_FILE_SIZE = 5 * 1024 * 1024  # 5 MB

async def validate_image_upload(file: UploadFile) -> bytes:
    """Validate uploaded image file."""
    # Read file content
    content = await file.read()

    # Check file size
    if len(content) > MAX_FILE_SIZE:
        raise HTTPException(status_code=413, detail="File too large")

    # Validate MIME type using python-magic (reads file header)
    mime_type = magic.from_buffer(content, mime=True)
    if mime_type not in ALLOWED_MIME_TYPES:
        raise HTTPException(status_code=415, detail="Invalid file type")

    return content
```

### Business Rule Validation
- **Domain Logic**: Validate business rules at the service layer
- **Database Constraints**: Enforce constraints at database level when possible
- **Consistent Validation**: Apply same validation rules across all entry points
- **Async Validation**: Use async validators for database-dependent validation

```python
from sqlalchemy.ext.asyncio import AsyncSession

class UserService:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create_user(self, user_data: UserCreate) -> User:
        # Business rule: email must be unique
        existing_user = await self.get_user_by_email(user_data.email)
        if existing_user:
            raise ValueError("Email already registered")

        # Business rule: username must be unique
        existing_username = await self.get_user_by_username(user_data.username)
        if existing_username:
            raise ValueError("Username already taken")

        # Create user
        user = User(**user_data.model_dump())
        self.session.add(user)
        await self.session.commit()
        return user
```

### Error Messages
- **Clear and Specific**: Provide clear, field-specific error messages
- **Actionable**: Tell users how to fix the error
- **Security**: Don't expose internal details or system information
- **Localization**: Support internationalization for error messages

```python
from pydantic import BaseModel, field_validator

class PasswordChange(BaseModel):
    current_password: str
    new_password: str = Field(..., min_length=12)

    @field_validator('new_password')
    @classmethod
    def validate_password_strength(cls, v: str) -> str:
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain at least one uppercase letter')
        if not any(c.islower() for c in v):
            raise ValueError('Password must contain at least one lowercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain at least one digit')
        if not any(c in '!@#$%^&*()_+-=[]{}|;:,.<>?' for c in v):
            raise ValueError('Password must contain at least one special character')
        return v
```

### Validation Testing
- **Test Invalid Input**: Write tests for validation errors, not just happy paths
- **Boundary Testing**: Test boundary conditions (min/max values)
- **Type Testing**: Test that wrong types are rejected
- **Error Messages**: Verify error messages are clear and helpful

```python
import pytest
from pydantic import ValidationError

def test_user_create_validation():
    # Test valid data
    user = UserCreate(email="test@example.com", username="testuser", age=25)
    assert user.email == "test@example.com"

    # Test invalid email
    with pytest.raises(ValidationError) as exc_info:
        UserCreate(email="invalid-email", username="testuser", age=25)
    assert "email" in str(exc_info.value)

    # Test age validation
    with pytest.raises(ValidationError) as exc_info:
        UserCreate(email="test@example.com", username="testuser", age=15)
    assert "age" in str(exc_info.value)
```
