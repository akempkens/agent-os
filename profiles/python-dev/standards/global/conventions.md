## Python Development Conventions

### Project Structure
- **Standard Layout**: Follow standard Python project structure:
  ```
  project_name/
  ├── pyproject.toml          # Poetry configuration and dependencies
  ├── README.md               # Project documentation
  ├── .gitignore              # Python-specific gitignore
  ├── .pre-commit-config.yaml # Pre-commit hooks
  ├── src/                    # Source code (use src layout)
  │   └── project_name/       # Main package
  │       ├── __init__.py
  │       ├── main.py
  │       ├── api/            # API routes/endpoints
  │       ├── models/         # Database models
  │       ├── services/       # Business logic
  │       ├── schemas/        # Pydantic models/serializers
  │       └── utils/          # Utility functions
  ├── tests/                  # Test directory (mirrors src structure)
  │   ├── __init__.py
  │   ├── conftest.py         # Pytest fixtures
  │   └── test_*.py           # Test modules
  ├── alembic/                # Database migrations (if using Alembic)
  └── docs/                   # Documentation
  ```
- **Src Layout**: Use `src/` layout to avoid import confusion during development
- **Package Names**: Use short, lowercase package names without underscores when possible

### Dependency Management
- **Poetry for Everything**: Use Poetry for dependency management, virtual environments, and packaging
- **Separate Dependencies**: Define dependencies in groups: `main`, `dev`, `test`, `docs`
- **Lock Files**: Always commit `poetry.lock` to ensure reproducible builds
- **Version Constraints**: Use caret (`^`) for compatible versions: `fastapi = "^0.104.0"`
- **Security Updates**: Regularly run `poetry update` and `poetry audit` for security vulnerabilities

### Environment Configuration
- **Environment Variables**: Use environment variables for all configuration (database URLs, API keys, etc.)
- **Pydantic Settings**: Use Pydantic's `BaseSettings` for type-safe configuration management
- **env Files**: Use `.env` files for local development (never commit to version control)
- **Separate Configs**: Maintain separate config files for development, staging, and production
- **No Secrets in Code**: Never hardcode secrets, API keys, or credentials in source code

### Version Control Best Practices
- **Gitignore**: Use comprehensive Python `.gitignore` (include `__pycache__/`, `.venv/`, `.env`, etc.)
- **Clear Commit Messages**: Write descriptive commit messages following conventional commits format
- **Feature Branches**: Use feature branches for new development; never commit directly to main
- **Pull Requests**: Require code review through pull requests before merging
- **Pre-commit Hooks**: Set up pre-commit hooks to run Black, Ruff, mypy before committing

### Code Review Process
- **Type Safety**: Ensure all new code has proper type hints and passes mypy checks
- **Test Coverage**: New features must include tests; aim for >80% coverage on new code
- **Documentation**: Public APIs must have docstrings and usage examples
- **Security Review**: Check for common vulnerabilities (SQL injection, XSS, secrets in code)
- **Performance**: Consider performance implications for database queries and API endpoints

### Testing Requirements
- **Test Everything New**: All new features and bug fixes must include tests
- **Fast Unit Tests**: Keep unit tests fast (< 100ms each) by mocking external dependencies
- **Integration Tests**: Write integration tests for critical paths and API endpoints
- **Test Isolation**: Each test should be independent and runnable in any order
- **Fixtures**: Use pytest fixtures to share setup code and test data

### Database Best Practices
- **Migrations**: Use migrations for all database schema changes (Alembic or Django migrations)
- **Migration Naming**: Use descriptive names for migrations: `add_user_email_index`
- **Reversible Migrations**: Ensure migrations can be rolled back safely
- **No Direct SQL**: Prefer ORM queries over raw SQL unless performance requires it
- **Connection Pooling**: Use connection pooling for production deployments

### API Development Conventions
- **RESTful Design**: Follow REST principles with resource-based URLs
- **Versioning**: Version APIs from day one (e.g., `/api/v1/users`)
- **Pydantic Schemas**: Use Pydantic models for request/response validation
- **Automatic Documentation**: Leverage FastAPI's automatic OpenAPI documentation
- **HTTP Status Codes**: Return appropriate status codes (200, 201, 400, 404, 500, etc.)

### Logging Standards
- **Structured Logging**: Use structlog for structured, machine-readable logs
- **Log Levels**: Use appropriate log levels: DEBUG, INFO, WARNING, ERROR, CRITICAL
- **Correlation IDs**: Include request/correlation IDs for tracing requests through system
- **No Sensitive Data**: Never log passwords, tokens, or sensitive user data
- **JSON Format**: Output logs in JSON format for production environments

### Async/Await Best Practices
- **Async When Needed**: Use async/await for I/O-bound operations (database, HTTP requests)
- **Consistent Async**: Keep async code paths separate from sync code; don't mix unnecessarily
- **Proper Awaits**: Always await async functions; use `asyncio.gather()` for concurrent operations
- **Resource Management**: Use `async with` for async context managers

### Security Standards
- **Input Validation**: Validate and sanitize all user input using Pydantic
- **SQL Injection**: Use parameterized queries (ORM handles this automatically)
- **XSS Protection**: Escape output and use CSP headers
- **Authentication**: Use established libraries (FastAPI-Users, Django-Allauth) instead of rolling your own
- **HTTPS Only**: Require HTTPS in production; redirect HTTP to HTTPS
- **CORS Configuration**: Configure CORS properly; never use `allow_origins=["*"]` in production

### Documentation Standards
- **README**: Maintain up-to-date README with setup instructions, requirements, and usage examples
- **API Docs**: Keep OpenAPI/Swagger documentation current (auto-generated with FastAPI)
- **Architecture Docs**: Document major architectural decisions and system design
- **Changelog**: Maintain CHANGELOG.md following Keep a Changelog format
- **Code Comments**: Use comments sparingly; prefer self-documenting code with clear names

### Performance Considerations
- **Database Queries**: Use `select_related()` and `prefetch_related()` to avoid N+1 queries
- **Caching**: Implement caching for expensive operations (Redis)
- **Pagination**: Always paginate list endpoints
- **Background Tasks**: Use Celery or background tasks for long-running operations
- **Profiling**: Profile slow endpoints with tools like py-spy or Django Debug Toolbar
