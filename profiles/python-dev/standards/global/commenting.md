## Python Code Commenting Best Practices

### Self-Documenting Code First
- **Clear Names Over Comments**: Write code that explains itself through clear structure and naming
- **Type Hints as Documentation**: Use type hints to document expected types instead of comments
- **Small Functions**: Break complex logic into small, well-named functions instead of commenting sections
- **Docstrings for Public APIs**: Use docstrings for modules, classes, and public functions

### When to Write Comments
- **Why, Not What**: Explain why code exists, not what it does (code shows what)
- **Non-Obvious Decisions**: Comment on non-obvious design decisions or trade-offs
- **Complex Algorithms**: Explain complex algorithms or business logic that isn't immediately clear
- **Bug Workarounds**: Document temporary workarounds with issue references
- **Performance Optimizations**: Explain why code is optimized in a non-obvious way

### When NOT to Write Comments
```python
# Bad: Comment states the obvious
# Increment counter by 1
counter += 1

# Good: Self-explanatory code
counter += 1

# Bad: Comment duplicates function name
# Get user by ID
def get_user_by_id(user_id: int) -> User:
    pass

# Good: Name and type hints are sufficient
def get_user_by_id(user_id: int) -> User:
    pass
```

### Docstrings (Documentation Strings)
- **Module Docstrings**: Add docstrings to all modules explaining their purpose
- **Class Docstrings**: Document classes with their purpose and key attributes
- **Function Docstrings**: Document all public functions with Google or NumPy style
- **Don't Duplicate Type Hints**: Don't repeat type information from type hints in docstrings

#### Google Style Docstrings (Recommended)
```python
def create_user(
    email: str,
    username: str,
    age: int,
    *,
    is_admin: bool = False
) -> User:
    """Create a new user in the system.

    Creates a new user account with the provided information. The email
    address must be unique across all users. Username will be normalized
    to lowercase.

    Args:
        email: User's email address (must be unique)
        username: Desired username (3-50 characters, alphanumeric)
        age: User's age in years (must be 18 or older)
        is_admin: Whether user should have admin privileges

    Returns:
        The newly created User object with assigned ID.

    Raises:
        ValueError: If email or username already exists
        ValidationError: If any input fails validation

    Example:
        >>> user = create_user(
        ...     email="user@example.com",
        ...     username="johndoe",
        ...     age=25
        ... )
        >>> print(user.id)
        123
    """
    pass
```

#### Class Docstrings
```python
class UserRepository:
    """Repository for user data access operations.

    Provides methods for CRUD operations on User entities. All methods
    are async and use the injected database session for queries.

    Attributes:
        session: SQLAlchemy async database session
        cache: Redis cache client for caching user queries
    """

    def __init__(self, session: AsyncSession, cache: Redis):
        self.session = session
        self.cache = cache
```

#### Module Docstrings
```python
"""User authentication and authorization utilities.

This module provides functions for user authentication, JWT token
generation and validation, and permission checking. It integrates
with FastAPI-Users for session management.

Example:
    from auth import authenticate_user, create_access_token

    user = await authenticate_user(email, password)
    token = create_access_token(user.id)
"""
```

### Type Hints as Documentation
- **Always Use Type Hints**: Type hints are the best form of documentation for parameters and returns
- **Generic Types**: Use generics for collections: `list[User]`, `dict[str, int]`
- **Optional Types**: Use `Type | None` for optional values (Python 3.10+)
- **TypeAlias**: Create type aliases for complex types

```python
from typing import TypeAlias

# Type alias makes complex type readable
UserId: TypeAlias = int
UserDict: TypeAlias = dict[str, str | int | None]

def get_users_by_ids(user_ids: list[UserId]) -> list[User]:
    """Fetch users by their IDs."""
    pass

def process_user_data(data: UserDict) -> User:
    """Process raw user data into User object."""
    pass
```

### Inline Comments
- **Minimal and Helpful**: Use inline comments sparingly for large sections of logic
- **Above the Line**: Place comments on the line above the code, not at the end
- **Complete Sentences**: Write comments as complete sentences with proper punctuation
- **No Obvious Comments**: Don't state what the code obviously does

```python
# Bad: Stating the obvious
x = x + 1  # Add one to x

# Good: Explaining why
# Account for zero-based indexing when displaying to users
display_position = index + 1

# Good: Explaining complex business logic
# Apply 10% discount only for orders over $100 placed by premium members
# This is a temporary promotion for Q4 2024
if order.total > 100 and user.is_premium:
    discount = order.total * 0.10
```

### TODO Comments
- **Use TODO Tags**: Mark incomplete work with TODO comments including issue reference
- **Include Context**: Explain what needs to be done and why it's not done yet
- **Track in Issues**: Create issues for TODOs; don't rely solely on comments

```python
# TODO(#123): Implement caching for this expensive query
# Currently hits database on every request, should use Redis cache
# with 5-minute TTL once we have Redis deployed
def get_user_statistics(user_id: int) -> UserStats:
    pass
```

### Don't Comment Out Code
- **Delete, Don't Comment**: Delete unused code instead of commenting it out
- **Version Control**: Trust version control to keep history; deleted code can be recovered
- **Clutters Codebase**: Commented-out code confuses readers and clutters the codebase

```python
# Bad: Commented-out code
def process_payment(amount: float):
    # old_amount = amount * 1.05  # Old tax calculation
    # tax = old_amount * 0.08
    new_amount = calculate_with_tax(amount)
    return new_amount

# Good: Clean code
def process_payment(amount: float) -> float:
    return calculate_with_tax(amount)
```

### Evergreen Comments
- **Timeless Information**: Comments should be relevant far into the future
- **Don't Document Changes**: Don't comment on recent changes or fixes
- **No Dates or Names**: Avoid "Fixed by John on 2024-01-15" - use git history

```python
# Bad: Temporal comment that will become outdated
# Fixed bug where users couldn't login (2024-01-15)
# Changed by Jane to handle edge case

# Good: Evergreen explanation
# Users with unverified emails cannot access premium features
# as per security requirements documented in SEC-101
```

### Comments for Complex Logic
```python
def calculate_compound_interest(
    principal: float,
    rate: float,
    years: int,
    compounds_per_year: int = 12
) -> float:
    """Calculate compound interest."""
    # Formula: A = P(1 + r/n)^(nt)
    # Where:
    #   A = final amount
    #   P = principal
    #   r = annual interest rate (decimal)
    #   n = compounds per year
    #   t = time in years
    rate_decimal = rate / 100
    amount = principal * (1 + rate_decimal / compounds_per_year) ** (compounds_per_year * years)
    return amount
```

### Documentation for APIs
- **OpenAPI/Swagger**: Use FastAPI's automatic OpenAPI docs generation
- **Descriptions**: Add descriptions to Pydantic fields and endpoint parameters
- **Examples**: Provide examples in docstrings and Pydantic schemas

```python
from pydantic import BaseModel, Field

class UserCreate(BaseModel):
    """Schema for creating a new user account."""

    email: EmailStr = Field(
        ...,
        description="User's email address (must be unique)",
        example="user@example.com"
    )
    username: str = Field(
        ...,
        min_length=3,
        max_length=50,
        description="Desired username (alphanumeric, 3-50 characters)",
        example="johndoe"
    )
    age: int = Field(
        ...,
        ge=18,
        le=120,
        description="User's age in years (must be 18+)",
        example=25
    )
```

### Architecture and Design Comments
- **High-Level Overview**: Document architectural decisions in module docstrings
- **Design Patterns**: Explain which design patterns are used and why
- **Coupling**: Document intentional coupling or dependencies between modules

```python
"""User service layer.

This service implements the business logic for user management. It acts
as a mediator between the API layer and the repository layer, enforcing
business rules and coordinating transactions.

Design patterns used:
- Repository pattern for data access abstraction
- Service layer pattern for business logic encapsulation

The service layer ensures that all user operations validate business
rules (unique email, age restrictions) before persisting to database.
"""
```
