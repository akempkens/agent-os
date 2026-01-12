## Python Tech Stack

Define your Python technical stack below. This serves as a reference for all team members and helps maintain consistency across the project.

### Language & Runtime
- **Python Version:** Python 3.11+ (or 3.12+ for latest features)
- **Package Manager:** Poetry (modern dependency management with lock files)
- **Virtual Environment:** Poetry (automatically managed) or venv

### Web Frameworks
- **API Framework:** FastAPI (modern, async, with automatic OpenAPI docs) or Django REST Framework
- **Full-Stack Framework:** Django (batteries included) or Flask (lightweight)
- **ASGI Server:** Uvicorn (for async frameworks) or Hypercorn

### Backend & Data
- **Database:** PostgreSQL (primary choice) or MySQL
- **ORM:** SQLAlchemy 2.0+ (with async support) or Django ORM
- **Database Migrations:** Alembic (SQLAlchemy) or Django Migrations
- **Async Database Driver:** asyncpg (PostgreSQL) or aiomysql (MySQL)
- **Caching:** Redis (with redis-py or aioredis for async)
- **Task Queue:** Celery (distributed tasks) or Dramatiq

### Data Validation & Serialization
- **Validation:** Pydantic v2 (type-safe data validation and settings management)
- **API Serialization:** Pydantic models (FastAPI) or Django REST Framework serializers

### Code Quality & Formatting
- **Formatter:** Black (uncompromising code formatter)
- **Linter:** Ruff (extremely fast Python linter, replaces flake8, isort, pylint)
- **Type Checker:** mypy (static type checking) or pyright
- **Import Sorting:** Ruff (built-in) or isort
- **Pre-commit Hooks:** pre-commit (automate code quality checks)

### Testing & Quality Assurance
- **Test Framework:** pytest (with pytest-asyncio for async tests)
- **Coverage:** pytest-cov (code coverage reporting)
- **Mocking:** pytest-mock or unittest.mock
- **HTTP Testing:** httpx (async-compatible) or requests-mock
- **Database Testing:** pytest-postgresql or factory_boy (test fixtures)

### Development Tools
- **Environment Variables:** python-dotenv or Pydantic settings
- **CLI Tools:** Click or Typer (type-safe CLI with Pydantic)
- **Documentation:** Sphinx or MkDocs with autodoc
- **API Documentation:** Auto-generated with FastAPI or drf-spectacular (Django)

### Authentication & Security
- **Authentication:** FastAPI-Users, Django-Allauth, or Auth0
- **JWT:** PyJWT or python-jose
- **Password Hashing:** passlib with bcrypt or argon2-cffi
- **Security Headers:** fastapi-security or django-csp

### Monitoring & Logging
- **Logging:** structlog (structured logging) or standard logging with JSON formatter
- **Error Tracking:** Sentry (sentry-sdk)
- **APM:** Datadog, New Relic, or OpenTelemetry
- **Metrics:** Prometheus client or StatsD

### Deployment & Infrastructure
- **Containerization:** Docker with multi-stage builds
- **WSGI/ASGI Server:** Gunicorn (WSGI) or Uvicorn/Hypercorn (ASGI)
- **Hosting:** AWS (ECS/Lambda), Google Cloud Run, Railway, or Fly.io
- **CI/CD:** GitHub Actions, GitLab CI, or CircleCI

### Additional Libraries
- **Date/Time:** pendulum (better datetime) or arrow
- **HTTP Client:** httpx (async-compatible) or requests
- **Background Jobs:** Celery, Dramatiq, or FastAPI BackgroundTasks
- **Email:** emails, fastapi-mail, or django-anymail
- **File Storage:** boto3 (S3), django-storages, or minio
