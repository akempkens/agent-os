## Python Error Handling Best Practices

### Exception Handling Principles
- **Be Specific**: Catch specific exception types rather than using bare `except:` or catching `Exception`
- **Fail Fast**: Validate input early and raise exceptions immediately when preconditions aren't met
- **Don't Silence Errors**: Never use empty `except:` blocks; at minimum, log the error
- **Exception Chaining**: Use `raise ... from ...` to preserve the original exception context
- **Let It Crash**: Don't catch exceptions you can't handle; let them propagate to higher levels

### Python Exception Best Practices
```python
# Good: Specific exception handling
try:
    user = await get_user(user_id)
except UserNotFoundError as e:
    logger.warning(f"User {user_id} not found", extra={"user_id": user_id})
    raise HTTPException(status_code=404, detail="User not found") from e
except DatabaseError as e:
    logger.error(f"Database error: {e}", exc_info=True)
    raise HTTPException(status_code=500, detail="Service unavailable") from e

# Bad: Bare except
try:
    user = get_user(user_id)
except:  # Never do this!
    pass
```

### Custom Exceptions
- **Domain-Specific Exceptions**: Create custom exception classes for your domain logic
- **Exception Hierarchy**: Organize exceptions in a hierarchy for easier catching
- **Meaningful Names**: Use descriptive exception names ending with "Error"
- **Include Context**: Add attributes to exceptions to provide debugging context

```python
class ProjectBaseError(Exception):
    """Base exception for project-specific errors."""
    pass

class ValidationError(ProjectBaseError):
    """Raised when data validation fails."""
    def __init__(self, message: str, field: str | None = None):
        self.field = field
        super().__init__(message)

class ResourceNotFoundError(ProjectBaseError):
    """Raised when a requested resource doesn't exist."""
    def __init__(self, resource_type: str, resource_id: str | int):
        self.resource_type = resource_type
        self.resource_id = resource_id
        super().__init__(f"{resource_type} with id {resource_id} not found")
```

### HTTP Exception Handling (FastAPI)
- **HTTPException**: Use FastAPI's `HTTPException` for API error responses
- **Status Codes**: Return appropriate HTTP status codes (400, 401, 403, 404, 422, 500)
- **Error Details**: Provide clear, actionable error messages without exposing internal details
- **Error Models**: Define Pydantic models for error responses for consistent structure

```python
from fastapi import HTTPException, status

# Good: Clear, user-friendly error
if not user:
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail="User not found"
    )

# Good: Validation error with details
if age < 0:
    raise HTTPException(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        detail="Age must be a positive number"
    )
```

### Error Response Format
- **Consistent Structure**: Use consistent error response format across all endpoints
- **Error Codes**: Include machine-readable error codes for client handling
- **User-Friendly Messages**: Provide clear messages for end users
- **Hide Internal Details**: Never expose stack traces, database errors, or internal paths to users

```python
# Recommended error response structure
{
    "error": {
        "code": "USER_NOT_FOUND",
        "message": "The requested user could not be found",
        "status": 404,
        "timestamp": "2024-01-12T10:30:00Z"
    }
}
```

### Logging Errors
- **Log Before Raising**: Log errors with full context before converting to HTTP responses
- **Include Context**: Add relevant context (user_id, request_id, etc.) to log messages
- **Use exc_info**: Pass `exc_info=True` to logging to include full traceback
- **Structured Logging**: Use structured logging (structlog) for machine-readable error logs

```python
import structlog

logger = structlog.get_logger()

try:
    result = await process_payment(user_id, amount)
except PaymentError as e:
    logger.error(
        "Payment processing failed",
        user_id=user_id,
        amount=amount,
        error=str(e),
        exc_info=True
    )
    raise HTTPException(
        status_code=status.HTTP_402_PAYMENT_REQUIRED,
        detail="Payment could not be processed"
    ) from e
```

### Resource Cleanup
- **Context Managers**: Always use context managers (`with` statements) for resource management
- **Async Context Managers**: Use `async with` for async resources (database connections, HTTP clients)
- **Finally Blocks**: Use `finally` only when context managers aren't available
- **No Manual Cleanup**: Avoid manual `.close()` calls; let context managers handle it

```python
# Good: Context manager handles cleanup
async with httpx.AsyncClient() as client:
    response = await client.get(url)

# Good: File handling with context manager
with open("file.txt", "r") as f:
    content = f.read()
```

### Retry Logic
- **Exponential Backoff**: Use exponential backoff for retrying failed operations
- **Max Retries**: Set reasonable maximum retry limits (typically 3-5 attempts)
- **Idempotency**: Only retry idempotent operations
- **Retry Libraries**: Use libraries like `tenacity` for sophisticated retry logic

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
async def fetch_external_data(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()
```

### Validation Errors
- **Pydantic Validation**: Use Pydantic for automatic validation and error generation
- **Custom Validators**: Create custom validators for complex business rules
- **Clear Error Messages**: Provide field-specific error messages that help users fix their input

```python
from pydantic import BaseModel, field_validator, ValidationError

class UserCreate(BaseModel):
    email: str
    age: int

    @field_validator('age')
    def validate_age(cls, v):
        if v < 18:
            raise ValueError('User must be at least 18 years old')
        if v > 120:
            raise ValueError('Invalid age value')
        return v
```

### Error Monitoring
- **Sentry Integration**: Use Sentry for error tracking and alerting
- **Error Rates**: Monitor error rates and set up alerts for spikes
- **Context Capture**: Include user context, request data, and breadcrumbs in error reports
- **Error Grouping**: Configure proper error grouping to avoid alert fatigue

### Global Exception Handlers
- **Centralized Handling**: Use FastAPI exception handlers for consistent error responses
- **Catch-All Handler**: Implement a catch-all handler for unexpected errors
- **Security**: Never expose internal error details in production

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

@app.exception_handler(ValidationError)
async def validation_exception_handler(request: Request, exc: ValidationError):
    return JSONResponse(
        status_code=422,
        content={"error": {"code": "VALIDATION_ERROR", "message": str(exc)}}
    )

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error("Unhandled exception", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={"error": {"code": "INTERNAL_ERROR", "message": "An unexpected error occurred"}}
    )
```

### Graceful Degradation
- **Fallback Mechanisms**: Implement fallbacks for non-critical service failures
- **Circuit Breakers**: Use circuit breakers for external service calls
- **Timeouts**: Set appropriate timeouts for all external calls
- **Feature Flags**: Use feature flags to disable problematic features without deployment
