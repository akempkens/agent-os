# Python Development Profile

A comprehensive Agent OS profile for modern Python development with best practices, modern tooling, and industry standards.

## Overview

This profile provides coding standards, conventions, and best practices for Python development using a modern tech stack. It's designed to guide AI agents and development teams in building high-quality Python applications.

## Tech Stack

This profile is optimized for:

- **Python 3.11+** - Modern Python with latest features
- **Poetry** - Modern dependency management
- **FastAPI** - Modern async web framework
- **SQLAlchemy 2.0+** - Async ORM with type hints
- **Pydantic v2** - Data validation and settings
- **Black** - Opinionated code formatting
- **Ruff** - Fast, comprehensive linting
- **mypy** - Static type checking
- **pytest** - Testing framework
- **PostgreSQL** - Primary database

## Profile Structure

```
python-dev/
├── README.md                           # This file
└── standards/
    ├── global/                         # Global standards for all Python projects
    │   ├── tech-stack.md              # Complete tech stack definition
    │   ├── coding-style.md            # PEP 8, Black, Ruff, type hints
    │   ├── conventions.md             # Python development conventions
    │   ├── error-handling.md          # Exception handling, logging
    │   ├── validation.md              # Pydantic validation patterns
    │   └── commenting.md              # Docstrings, type hints as documentation
    ├── backend/                        # Backend-specific standards
    │   ├── api.md                     # FastAPI REST API best practices
    │   ├── models.md                  # SQLAlchemy models, relationships
    │   └── queries.md                 # Database queries, N+1 prevention
    └── testing/                        # Testing standards
        └── test-writing.md            # pytest, async testing, coverage
```

## Key Standards

### Coding Style
- **PEP 8 Compliance**: Follow Python's official style guide
- **Automatic Formatting**: Black for consistent code style (88-100 chars)
- **Fast Linting**: Ruff for comprehensive, fast linting
- **Type Hints**: Full type hint coverage with mypy checking
- **Modern Syntax**: Python 3.10+ type syntax (`list[str]`, `dict[str, int]`)

### API Development
- **FastAPI First**: Async-first API development with automatic OpenAPI docs
- **Pydantic Validation**: Type-safe request/response validation
- **RESTful Design**: Resource-based URLs with appropriate HTTP methods
- **API Versioning**: Version APIs from day one (`/api/v1/`)

### Database
- **SQLAlchemy 2.0+**: Modern async ORM with declarative models
- **Type-Safe Models**: Full type hints on all model attributes
- **N+1 Prevention**: Proper eager loading with selectinload/joinedload
- **Async Operations**: All database operations use async/await

### Testing
- **pytest Framework**: Modern testing with fixtures and parametrization
- **Async Testing**: Full async/await support with pytest-asyncio
- **Test Coverage**: Aim for >80% coverage on core functionality
- **Fast Tests**: Keep unit tests fast; mock external dependencies

### Code Quality
- **Poetry**: Modern dependency management with lock files
- **Pre-commit Hooks**: Automated quality checks before commit
- **Type Checking**: Static type checking with mypy or pyright
- **Security**: Input validation, parameterized queries, no secrets in code

## Using This Profile

### For New Projects

1. Set up Poetry for dependency management:
   ```bash
   poetry init
   poetry add fastapi uvicorn sqlalchemy asyncpg pydantic
   poetry add --group dev black ruff mypy pytest pytest-asyncio
   ```

2. Configure tools in `pyproject.toml`:
   ```toml
   [tool.black]
   line-length = 88
   target-version = ['py311']

   [tool.ruff]
   line-length = 88
   target-version = "py311"

   [tool.mypy]
   python_version = "3.11"
   strict = true
   ```

3. Set up pre-commit hooks:
   ```bash
   poetry add --group dev pre-commit
   pre-commit install
   ```

4. Follow the standards in this profile for all code

### For Existing Projects

1. Review the standards in each section
2. Gradually adopt standards that fit your project
3. Update tooling configuration to match recommendations
4. Use standards as reference during code reviews

## Standards Overview

### Global Standards
- **tech-stack.md**: Complete modern Python tech stack with alternatives
- **coding-style.md**: PEP 8, Black, type hints, naming conventions
- **conventions.md**: Project structure, dependency management, configuration
- **error-handling.md**: Exception handling, logging, error responses
- **validation.md**: Pydantic validation patterns and best practices
- **commenting.md**: Docstrings, type hints, when to comment

### Backend Standards
- **api.md**: FastAPI REST API design, authentication, rate limiting
- **models.md**: SQLAlchemy models, relationships, indexes
- **queries.md**: Query patterns, N+1 prevention, performance

### Testing Standards
- **test-writing.md**: pytest patterns, fixtures, mocking, coverage

## Philosophy

This profile emphasizes:

1. **Type Safety**: Comprehensive type hints for better tooling and fewer bugs
2. **Modern Tools**: Latest Python features and best-in-class tooling
3. **Async First**: Async/await for better performance in I/O-bound applications
4. **Simplicity**: Simple, readable code over clever abstractions
5. **Automation**: Automated formatting, linting, and testing
6. **Standards**: Consistent coding standards across the project

## When to Use This Profile

This profile is ideal for:

- ✅ Web APIs and microservices
- ✅ Async I/O-bound applications
- ✅ Projects requiring type safety
- ✅ Modern Python 3.11+ projects
- ✅ Teams wanting consistent standards
- ✅ Projects using FastAPI, SQLAlchemy, or Pydantic

Consider alternatives for:
- ❌ Data science / ML projects (use specialized profiles)
- ❌ Legacy Python 2 or older Python 3 projects
- ❌ Simple scripts (may be overkill)
- ❌ Projects with different framework requirements (Django-specific, etc.)

## Contributing

This profile is designed to be a living document. As Python and its ecosystem evolve, these standards should be updated to reflect current best practices.

## License

This profile is part of the Agent OS project and follows its licensing terms.
