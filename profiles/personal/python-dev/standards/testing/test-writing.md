## Python Testing Best Practices

### Testing Philosophy
- **Test Core Flows**: Focus on testing critical paths and primary user workflows
- **Minimal During Development**: Write strategic tests at logical completion points, not for every change
- **Test Behavior**: Test what the code does, not how it does it
- **Fast Tests**: Keep unit tests fast (milliseconds) to encourage frequent running

### Pytest Framework
- **Use Pytest**: Use pytest as the primary testing framework
- **Fixtures**: Use pytest fixtures for test setup and teardown
- **Parametrize**: Use @pytest.mark.parametrize for testing multiple scenarios
- **Async Support**: Use pytest-asyncio for async tests

```python
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

# Simple test
def test_user_full_name():
    """Test User.full_name property."""
    user = User(first_name="John", last_name="Doe")
    assert user.full_name == "John Doe"

# Async test
@pytest.mark.asyncio
async def test_get_user_by_id(db_session: AsyncSession):
    """Test retrieving user by ID."""
    # Create test user
    user = User(email="test@example.com", username="testuser")
    db_session.add(user)
    await db_session.commit()

    # Test retrieval
    found_user = await get_user_by_id(db_session, user.id)
    assert found_user is not None
    assert found_user.email == "test@example.com"

# Parametrized test
@pytest.mark.parametrize("age,expected", [
    (17, False),
    (18, True),
    (25, True),
    (121, False),
])
def test_age_validation(age: int, expected: bool):
    """Test age validation with multiple values."""
    if expected:
        user = UserCreate(email="test@example.com", username="test", age=age)
        assert user.age == age
    else:
        with pytest.raises(ValidationError):
            UserCreate(email="test@example.com", username="test", age=age)
```

### Test Organization
- **Mirror Source Structure**: Organize tests to mirror source code structure
- **Test File Naming**: Name test files with `test_` prefix: `test_user.py`, `test_api.py`
- **Test Function Naming**: Use descriptive names: `test_create_user_with_valid_data`
- **Group Related Tests**: Use classes to group related tests

```
tests/
├── conftest.py                  # Shared fixtures
├── test_models/
│   ├── test_user.py
│   └── test_post.py
├── test_services/
│   ├── test_user_service.py
│   └── test_auth_service.py
└── test_api/
    ├── test_users.py
    └── test_auth.py
```

### Fixtures
- **conftest.py**: Define shared fixtures in conftest.py
- **Scope**: Use appropriate fixture scope (function, class, module, session)
- **Cleanup**: Use yield for fixtures that need cleanup
- **Dependency Injection**: Use fixtures to inject dependencies

```python
# conftest.py
import pytest
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture(scope="session")
def event_loop():
    """Create event loop for async tests."""
    import asyncio
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture(scope="function")
async def db_session():
    """Create a test database session."""
    # Create test engine
    engine = create_async_engine(
        "postgresql+asyncpg://test:test@localhost/test_db",
        echo=False
    )

    # Create all tables
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    # Create session
    async_session = sessionmaker(
        engine, class_=AsyncSession, expire_on_commit=False
    )

    async with async_session() as session:
        yield session

    # Drop all tables after test
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

    await engine.dispose()

@pytest.fixture
async def test_user(db_session: AsyncSession):
    """Create a test user."""
    user = User(
        email="test@example.com",
        username="testuser",
        hashed_password="hashed_password"
    )
    db_session.add(user)
    await db_session.commit()
    await db_session.refresh(user)
    return user
```

### API Testing with FastAPI
- **TestClient**: Use FastAPI's TestClient for testing endpoints
- **AsyncClient**: Use httpx.AsyncClient for async endpoint tests
- **Mock Dependencies**: Override dependencies for testing
- **Test All Status Codes**: Test success, validation errors, and error cases

```python
import pytest
from httpx import AsyncClient
from fastapi import FastAPI

@pytest.mark.asyncio
async def test_create_user_success(client: AsyncClient):
    """Test successful user creation."""
    response = await client.post(
        "/api/v1/users",
        json={
            "email": "newuser@example.com",
            "username": "newuser",
            "password": "SecurePass123!"
        }
    )
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "newuser@example.com"
    assert data["username"] == "newuser"
    assert "password" not in data  # Password should not be in response

@pytest.mark.asyncio
async def test_create_user_duplicate_email(client: AsyncClient, test_user):
    """Test user creation with duplicate email."""
    response = await client.post(
        "/api/v1/users",
        json={
            "email": test_user.email,  # Duplicate email
            "username": "anotheruser",
            "password": "SecurePass123!"
        }
    )
    assert response.status_code == 400
    assert "email" in response.json()["detail"].lower()

@pytest.mark.asyncio
async def test_get_user_not_found(client: AsyncClient):
    """Test getting non-existent user."""
    response = await client.get("/api/v1/users/99999")
    assert response.status_code == 404
```

### Mocking
- **Mock External Services**: Mock external APIs, databases, and services
- **pytest-mock**: Use pytest-mock for easier mocking
- **unittest.mock**: Use unittest.mock for standard mocking
- **Don't Mock What You Own**: Test your own code, mock external dependencies

```python
import pytest
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_send_welcome_email(mocker):
    """Test sending welcome email."""
    # Mock email service
    mock_send = mocker.patch("app.services.email.send_email", new=AsyncMock())

    # Call function that sends email
    user = User(email="test@example.com", username="test")
    await send_welcome_email(user)

    # Verify email was sent
    mock_send.assert_called_once()
    call_args = mock_send.call_args
    assert call_args[0][0] == user.email
    assert "welcome" in call_args[1]["subject"].lower()

@pytest.mark.asyncio
@patch("app.services.payment.stripe.Charge.create")
async def test_process_payment(mock_stripe_charge):
    """Test payment processing with mocked Stripe."""
    # Mock Stripe response
    mock_stripe_charge.return_value = {
        "id": "ch_123",
        "status": "succeeded",
        "amount": 1000
    }

    # Process payment
    result = await process_payment(user_id=1, amount=10.00)

    # Verify
    assert result["status"] == "succeeded"
    mock_stripe_charge.assert_called_once()
```

### Database Testing
- **Test Database**: Use separate test database
- **Transaction Rollback**: Roll back transactions after each test
- **Factory Pattern**: Use factory_boy for creating test data
- **Isolation**: Ensure tests don't affect each other

```python
import pytest
from factory import Factory, Faker, SubFactory
from factory.alchemy import SQLAlchemyModelFactory

# Test data factories
class UserFactory(SQLAlchemyModelFactory):
    """Factory for creating test users."""
    class Meta:
        model = User
        sqlalchemy_session = None  # Set in fixture

    email = Faker("email")
    username = Faker("user_name")
    first_name = Faker("first_name")
    last_name = Faker("last_name")

class PostFactory(SQLAlchemyModelFactory):
    """Factory for creating test posts."""
    class Meta:
        model = Post
        sqlalchemy_session = None

    title = Faker("sentence")
    content = Faker("text")
    author = SubFactory(UserFactory)

# Usage in tests
@pytest.mark.asyncio
async def test_user_posts(db_session: AsyncSession):
    """Test user with multiple posts."""
    # Set factory session
    UserFactory._meta.sqlalchemy_session = db_session
    PostFactory._meta.sqlalchemy_session = db_session

    # Create test data
    user = UserFactory.create()
    posts = PostFactory.create_batch(3, author=user)

    await db_session.commit()

    # Test
    result = await db_session.execute(
        select(User).where(User.id == user.id).options(selectinload(User.posts))
    )
    retrieved_user = result.scalar_one()
    assert len(retrieved_user.posts) == 3
```

### Test Coverage
- **pytest-cov**: Use pytest-cov for coverage reporting
- **Aim for >80%**: Target >80% coverage for new code
- **Coverage is Not Quality**: High coverage doesn't mean good tests
- **Focus on Critical Paths**: Prioritize coverage of critical functionality

```bash
# Run tests with coverage
pytest --cov=src --cov-report=html --cov-report=term

# Coverage configuration in pyproject.toml
[tool.coverage.run]
source = ["src"]
omit = ["*/tests/*", "*/migrations/*"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
]
```

### Testing Best Practices
- **AAA Pattern**: Arrange, Act, Assert
- **One Assertion Per Test**: Generally test one thing per test
- **Clear Test Names**: Names should describe what's being tested
- **Fast Tests**: Keep unit tests fast; use integration tests for slow operations
- **Independent Tests**: Tests should not depend on each other
- **Test Edge Cases**: Test boundary conditions and edge cases

```python
@pytest.mark.asyncio
async def test_password_validation_too_short():
    """Test password validation rejects short passwords."""
    # Arrange
    user_data = {
        "email": "test@example.com",
        "username": "testuser",
        "password": "Short1!"  # Too short (< 12 chars)
    }

    # Act & Assert
    with pytest.raises(ValidationError) as exc_info:
        UserCreate(**user_data)

    # Verify error message
    errors = exc_info.value.errors()
    assert any("password" in str(error) for error in errors)
```

### Integration Tests
- **Test Real Interactions**: Test actual interactions between components
- **Use Test Database**: Use real database for integration tests
- **Slower but Valuable**: Integration tests are slower but catch more issues
- **Mark as Integration**: Mark with @pytest.mark.integration

```python
@pytest.mark.integration
@pytest.mark.asyncio
async def test_user_registration_flow(client: AsyncClient, db_session: AsyncSession):
    """Test complete user registration flow."""
    # Register user
    response = await client.post(
        "/api/v1/auth/register",
        json={
            "email": "newuser@example.com",
            "username": "newuser",
            "password": "SecurePass123!"
        }
    )
    assert response.status_code == 201
    user_data = response.json()

    # Verify user in database
    result = await db_session.execute(
        select(User).where(User.email == "newuser@example.com")
    )
    user = result.scalar_one()
    assert user is not None
    assert user.username == "newuser"

    # Login with new user
    login_response = await client.post(
        "/api/v1/auth/login",
        json={
            "email": "newuser@example.com",
            "password": "SecurePass123!"
        }
    )
    assert login_response.status_code == 200
    assert "access_token" in login_response.json()
```

### Error Testing
- **Test Failure Cases**: Test error conditions, not just happy paths
- **Validate Error Messages**: Verify error messages are clear and helpful
- **Test All Error Types**: Test validation, authentication, authorization errors

```python
@pytest.mark.asyncio
async def test_invalid_email_format():
    """Test user creation with invalid email format."""
    with pytest.raises(ValidationError) as exc_info:
        UserCreate(
            email="not-an-email",
            username="testuser",
            password="SecurePass123!"
        )
    assert "email" in str(exc_info.value)

@pytest.mark.asyncio
async def test_unauthorized_access(client: AsyncClient):
    """Test accessing protected endpoint without authentication."""
    response = await client.get("/api/v1/users/me")
    assert response.status_code == 401
```

### Test Configuration
```python
# pytest.ini or pyproject.toml
[tool.pytest.ini_options]
minversion = "7.0"
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
asyncio_mode = "auto"
markers = [
    "unit: Unit tests",
    "integration: Integration tests",
    "slow: Slow tests",
]
# Don't capture output for better debugging
addopts = "-v --tb=short --strict-markers"
```

### Running Tests
```bash
# Run all tests
pytest

# Run specific test file
pytest tests/test_user.py

# Run specific test
pytest tests/test_user.py::test_create_user

# Run tests by marker
pytest -m unit          # Only unit tests
pytest -m integration   # Only integration tests
pytest -m "not slow"    # Exclude slow tests

# Run with coverage
pytest --cov=src --cov-report=html

# Run in parallel (requires pytest-xdist)
pytest -n auto

# Run with verbose output
pytest -vv

# Stop on first failure
pytest -x
```
