## Docker & Container Best Practices

### Container Philosophy
- **Container-First**: Design applications to run in containers from day one
- **Immutable Infrastructure**: Containers should be immutable and reproducible
- **12-Factor App**: Follow 12-factor app principles
- **Multi-Environment**: Same container should run in dev, staging, and production

### Dockerfile Best Practices
- **Multi-Stage Builds**: Use multi-stage builds to reduce image size
- **Layer Caching**: Optimize layer caching for faster builds
- **Security**: Run as non-root user, scan for vulnerabilities
- **Python 3.12+**: Use latest Python version

```dockerfile
# Multi-stage build for Python LLM application
FROM python:3.12-slim AS builder

# Set build arguments
ARG POETRY_VERSION=1.7.0

# Install system dependencies
RUN apt-get update && apt-get install -y \
    --no-install-recommends \
    build-essential \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

# Install Poetry
RUN pip install --no-cache-dir poetry==${POETRY_VERSION}

# Set working directory
WORKDIR /app

# Copy dependency files
COPY pyproject.toml poetry.lock ./

# Install dependencies (without dev dependencies)
RUN poetry config virtualenvs.create false \
    && poetry install --no-dev --no-interaction --no-ansi

# Final stage
FROM python:3.12-slim

# Install runtime dependencies only
RUN apt-get update && apt-get install -y \
    --no-install-recommends \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Set working directory
WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

# Copy application code
COPY --chown=appuser:appuser . /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Run application
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### .dockerignore
- **Exclude Unnecessary Files**: Reduce build context size
- **Security**: Don't include secrets or credentials

```.dockerignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
*.egg-info/
dist/
build/

# Virtual environments
venv/
env/
ENV/

# IDE
.vscode/
.idea/
*.swp
*.swo

# Jupyter
.ipynb_checkpoints/
*.ipynb

# Data and models (too large)
data/
models/
*.pkl
*.h5
*.pt
*.bin

# Git
.git/
.gitignore

# Environment files
.env
.env.local
*.secret

# Logs
logs/
*.log

# Test and coverage
.pytest_cache/
.coverage
htmlcov/

# Documentation
docs/
*.md
!README.md
```

### Docker Compose for Development
- **Multi-Service**: Define all services (app, database, vector DB)
- **Volumes**: Use volumes for development hot-reload
- **Environment**: Manage environment variables

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/mydb
      - REDIS_URL=redis://redis:6379/0
      - NEO4J_URI=bolt://neo4j:7687
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    volumes:
      - ./app:/app/app  # Hot-reload in development
      - ./data:/app/data
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  postgres:
    image: pgvector/pgvector:pg16
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  neo4j:
    image: neo4j:5.15
    environment:
      - NEO4J_AUTH=neo4j/password
      - NEO4J_PLUGINS=["graph-data-science"]
    ports:
      - "7474:7474"  # HTTP
      - "7687:7687"  # Bolt
    volumes:
      - neo4j_data:/data

  chromadb:
    image: chromadb/chroma:latest
    ports:
      - "8001:8000"
    volumes:
      - chroma_data:/chroma/chroma
    environment:
      - ALLOW_RESET=true

volumes:
  postgres_data:
  redis_data:
  neo4j_data:
  chroma_data:
```

### Google Colab / Jupyter Container
- **Jupyter Ready**: Container optimized for Jupyter notebooks
- **GPU Support**: Include CUDA if needed
- **Pre-installed Packages**: Include all ML/LLM dependencies

```dockerfile
FROM python:3.12-slim

# Install Jupyter and ML dependencies
RUN pip install --no-cache-dir \
    jupyterlab \
    ipywidgets \
    numpy \
    pandas \
    matplotlib \
    plotly \
    openai \
    anthropic \
    langchain \
    chromadb \
    sentence-transformers \
    gradio

# Create workspace
WORKDIR /workspace

# Expose Jupyter port
EXPOSE 8888

# Create non-root user
RUN useradd -m -s /bin/bash jupyter

USER jupyter

# Start JupyterLab
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--port=8888", "--no-browser", "--allow-root"]
```

### Environment Configuration
- **12-Factor Config**: Store config in environment variables
- **.env Files**: Use .env files for local development
- **Docker Secrets**: Use secrets for sensitive data in production

```python
# settings.py using Pydantic
from pydantic_settings import BaseSettings
from typing import Optional

class Settings(BaseSettings):
    """Application settings from environment variables."""

    # Application
    app_name: str = "LLM API"
    debug: bool = False
    log_level: str = "INFO"

    # Database
    database_url: str
    db_pool_size: int = 20

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Neo4j
    neo4j_uri: str = "bolt://localhost:7687"
    neo4j_user: str = "neo4j"
    neo4j_password: str

    # LLM APIs
    openai_api_key: str
    anthropic_api_key: Optional[str] = None
    google_api_key: Optional[str] = None

    # Vector DB
    chroma_host: str = "localhost"
    chroma_port: int = 8000

    class Config:
        env_file = ".env"
        case_sensitive = False

# Singleton instance
settings = Settings()
```

### Health Checks
- **Container Health**: Implement health check endpoints
- **Dependency Checks**: Check database, cache, external APIs
- **Graceful Degradation**: Return partial health status

```python
from fastapi import FastAPI, status
from fastapi.responses import JSONResponse
import asyncpg
import asyncio

app = FastAPI()

async def check_postgres_health() -> bool:
    """Check PostgreSQL connection."""
    try:
        conn = await asyncpg.connect(settings.database_url, timeout=5)
        await conn.execute("SELECT 1")
        await conn.close()
        return True
    except Exception:
        return False

async def check_redis_health() -> bool:
    """Check Redis connection."""
    try:
        import redis.asyncio as redis
        r = redis.from_url(settings.redis_url)
        await r.ping()
        await r.close()
        return True
    except Exception:
        return False

async def check_llm_api_health() -> bool:
    """Check LLM API availability."""
    try:
        from openai import AsyncOpenAI
        client = AsyncOpenAI(api_key=settings.openai_api_key, timeout=5)
        # Quick test call
        await client.models.list()
        return True
    except Exception:
        return False

@app.get("/health")
async def health_check():
    """Comprehensive health check."""
    checks = {
        "postgres": await check_postgres_health(),
        "redis": await check_redis_health(),
        "llm_api": await check_llm_api_health(),
    }

    all_healthy = all(checks.values())
    status_code = status.HTTP_200_OK if all_healthy else status.HTTP_503_SERVICE_UNAVAILABLE

    return JSONResponse(
        status_code=status_code,
        content={
            "status": "healthy" if all_healthy else "unhealthy",
            "checks": checks
        }
    )

@app.get("/ready")
async def readiness_check():
    """Readiness probe (minimal check)."""
    return {"status": "ready"}

@app.get("/live")
async def liveness_check():
    """Liveness probe (very basic)."""
    return {"status": "alive"}
```

### Container Orchestration (Kubernetes)
- **Deployment**: Define Kubernetes deployment
- **Services**: Expose services with LoadBalancer or Ingress
- **ConfigMaps/Secrets**: Manage configuration

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: llm-api
  template:
    metadata:
      labels:
        app: llm-api
    spec:
      containers:
      - name: api
        image: myregistry/llm-api:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: url
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: openai
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /live
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: llm-api-service
spec:
  selector:
    app: llm-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

### GPU Support for ML Workloads
- **NVIDIA Container Runtime**: Use NVIDIA runtime for GPU access
- **CUDA Base Images**: Use CUDA-enabled base images

```dockerfile
# GPU-enabled Dockerfile for ML workloads
FROM nvidia/cuda:12.2.0-runtime-ubuntu22.04

# Install Python
RUN apt-get update && apt-get install -y \
    python3.12 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Install PyTorch with CUDA support
RUN pip3 install --no-cache-dir \
    torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121

# Install other ML dependencies
RUN pip3 install --no-cache-dir \
    transformers \
    accelerate \
    bitsandbytes \
    sentence-transformers

WORKDIR /app
COPY . /app

CMD ["python3", "train.py"]
```

```yaml
# docker-compose with GPU
version: '3.8'

services:
  ml-trainer:
    build:
      context: .
      dockerfile: Dockerfile.gpu
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - ./models:/app/models
      - ./data:/app/data
```

### Container Registry & CI/CD
- **Image Tagging**: Use semantic versioning for images
- **Multi-Arch Builds**: Build for multiple architectures
- **Security Scanning**: Scan images for vulnerabilities

```yaml
# .github/workflows/docker-build.yml
name: Build and Push Docker Image

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.meta.outputs.tags }}
          format: 'sarif'
          output: 'trivy-results.sarif'
```

### Development vs Production
- **Development**: Hot-reload, debug mode, local volumes
- **Production**: Optimized, minimal, health checks

```dockerfile
# Dockerfile.dev (development)
FROM python:3.12-slim

WORKDIR /app

# Install development dependencies
RUN pip install --no-cache-dir poetry

# Copy dependencies
COPY pyproject.toml poetry.lock ./
RUN poetry config virtualenvs.create false && \
    poetry install --with dev

# Application code is mounted as volume
EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

### Logging in Containers
- **stdout/stderr**: Log to stdout/stderr for container logging
- **JSON Logs**: Use JSON format for structured logging
- **Log Aggregation**: Use logging drivers or external systems

```python
import logging
import sys
from pythonjsonlogger import jsonlogger

# Configure JSON logging for containers
logger = logging.getLogger()
handler = logging.StreamHandler(sys.stdout)
formatter = jsonlogger.JsonFormatter(
    '%(asctime)s %(name)s %(levelname)s %(message)s'
)
handler.setFormatter(formatter)
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Usage
logger.info("Application started", extra={"version": "1.0.0", "env": "production"})
```

### Resource Limits
- **Memory Limits**: Set appropriate memory limits
- **CPU Limits**: Limit CPU usage
- **Production**: Always set resource limits

```yaml
# docker-compose with resource limits
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 1G
```
