# Python Development Profile for LLM & ML Applications

A comprehensive Agent OS profile for modern Python development focused on LLM applications, machine learning, Jupyter notebooks, and graph databases.

## 🚀 Quick Start for AI Agents

**Start here**: [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) - Compact 350-line guide with all essential standards and code patterns.

**Detailed examples**: See individual standard files in `standards/` directory only when needed.

## Overview

This profile provides coding standards, conventions, and best practices for Python development using a modern LLM/ML tech stack. It's designed to guide AI agents and development teams in building production-quality LLM applications, working with vector databases, and deploying in container environments.

## Tech Stack

This profile is optimized for:

### Core
- **Python 3.12+** - Latest Python with modern type hints and performance
- **Poetry** - Modern dependency management with lock files
- **Docker** - Container-first deployment strategy
- **PostgreSQL 16+** - Direct SQL with asyncpg (no ORM)

### LLM & AI
- **OpenAI SDK** - GPT-4, GPT-3.5, embeddings
- **Anthropic SDK** - Claude models
- **Google Generative AI** - Gemini models
- **LangChain** - LLM orchestration framework
- **Ollama** - Local LLM deployment

### Machine Learning
- **PyTorch** - Primary ML framework
- **HuggingFace Transformers** - Pre-trained models
- **Sentence Transformers** - Local embeddings
- **Accelerate** - Distributed training and inference

### Data & Databases
- **PostgreSQL + pgvector** - Vector similarity search
- **ChromaDB** - Vector database for embeddings
- **Neo4j** - Graph database for relationships
- **Redis** - Caching and message queuing

### Development
- **JupyterLab** - Interactive development
- **Gradio** - ML demo interfaces
- **FastAPI** - Async API framework with OpenAPI 3.1
- **MCP Server Support** - Model Context Protocol integration

### Code Quality
- **Black** - Code formatting
- **Ruff** - Fast linting
- **mypy** - Static type checking
- **pytest** - Testing framework

## Profile Structure

```
python-dev/
├── README.md                           # This file
├── QUICK_REFERENCE.md                  # ⭐ Compact guide for AI agents (start here!)
└── standards/
    ├── global/                         # Global standards for all Python projects
    │   ├── tech-stack.md              # Complete LLM/ML tech stack
    │   ├── coding-style.md            # PEP 8, Black, Ruff, type hints
    │   ├── conventions.md             # Python development conventions
    │   ├── error-handling.md          # Exception handling, logging
    │   ├── validation.md              # Pydantic validation patterns
    │   └── commenting.md              # Docstrings, type hints as documentation
    ├── backend/                        # Backend-specific standards
    │   ├── api.md                     # FastAPI + MCP + Gradio integration
    │   ├── database.md                # PostgreSQL direct SQL patterns
    │   ├── queries.md                 # Direct SQL queries with asyncpg
    │   └── graph-database.md          # Neo4j graph database patterns
    ├── ml/                             # Machine Learning standards
    │   ├── notebooks.md               # Jupyter notebook best practices
    │   └── llm-integration.md         # LLM provider integration patterns
    ├── deployment/                     # Deployment standards
    │   └── docker.md                  # Docker & container best practices
    └── testing/                        # Testing standards
        └── test-writing.md            # pytest, async testing, coverage
```

## Key Standards

### LLM Integration
- **Multi-Provider Support**: OpenAI, Anthropic, Google, Ollama
- **Async Operations**: All LLM calls use async/await
- **Error Handling**: Retry logic, rate limit handling, fallbacks
- **Token Management**: Track and optimize token usage
- **Streaming**: Server-Sent Events for streaming responses

### API Development
- **FastAPI + OpenAPI 3.1**: Modern async API with latest OpenAPI spec
- **MCP Server Support**: Implement Model Context Protocol servers
- **Gradio Integration**: Quick ML demo interfaces
- **Vector Search API**: Endpoints for embeddings and semantic search
- **Direct PostgreSQL**: Use asyncpg for direct SQL (no ORM)

### Notebook Development
- **JupyterLab**: Interactive development environment
- **Google Colab Compatible**: Notebooks work in Colab
- **Progress Tracking**: Use tqdm for long-running operations
- **Visualization**: Matplotlib, Plotly for rich visualizations
- **Version Control**: Clear outputs before committing

### Database
- **PostgreSQL + pgvector**: Vector similarity search with SQL
- **Direct SQL**: Write clear SQL queries with asyncpg
- **No ORM**: Direct database access for performance and control
- **Graph Database**: Neo4j for relationship-heavy data
- **Vector Databases**: ChromaDB for embeddings

### Container Deployment
- **Docker-First**: All services containerized
- **Multi-Stage Builds**: Optimized container images
- **Health Checks**: Proper health/readiness/liveness probes
- **Docker Compose**: Local multi-service development
- **Kubernetes Ready**: Production-ready deployments

## Using This Profile

### For New LLM Projects

1. Set up Poetry with LLM dependencies:
   ```bash
   poetry init
   poetry add fastapi uvicorn openai anthropic langchain
   poetry add chromadb asyncpg pydantic python-dotenv
   poetry add --group dev black ruff mypy pytest jupyterlab
   ```

2. Create `.env` file for API keys:
   ```env
   OPENAI_API_KEY=your_key
   ANTHROPIC_API_KEY=your_key
   DATABASE_URL=postgresql://user:pass@localhost/db
   ```

3. Set up Docker development environment:
   ```bash
   docker-compose up -d
   ```

4. Follow the standards in this profile for all code

### For Notebook Development

1. Install Jupyter dependencies:
   ```bash
   poetry add jupyterlab ipywidgets gradio
   poetry add numpy pandas matplotlib plotly
   ```

2. Launch JupyterLab:
   ```bash
   poetry run jupyter lab
   ```

3. Follow notebook best practices from `ml/notebooks.md`

## Standards Overview

### Global Standards
- **tech-stack.md**: Complete LLM/ML tech stack with all libraries
- **coding-style.md**: PEP 8, Black, type hints, naming conventions
- **conventions.md**: Project structure, containers, Jupyter compatibility
- **error-handling.md**: Exception handling, logging, error responses
- **validation.md**: Pydantic validation patterns for LLM inputs/outputs
- **commenting.md**: Docstrings, type hints, documentation

### Backend Standards
- **api.md**: FastAPI + OpenAPI 3.1, MCP servers, Gradio integration, vector search
- **database.md**: PostgreSQL direct SQL patterns with asyncpg
- **queries.md**: Direct SQL queries, vector search, JSONB operations
- **graph-database.md**: Neo4j patterns for graph data

### ML Standards
- **notebooks.md**: Jupyter best practices, Colab compatibility, visualization
- **llm-integration.md**: OpenAI, Anthropic, LangChain, RAG patterns

### Deployment Standards
- **docker.md**: Container best practices, Docker Compose, Kubernetes, GPU support

### Testing Standards
- **test-writing.md**: pytest patterns, async testing, mocking, coverage

## Philosophy

This profile emphasizes:

1. **LLM-First**: Designed for building LLM-powered applications
2. **Type Safety**: Comprehensive type hints for better tooling
3. **Async Operations**: Async/await for LLM calls and I/O operations
4. **Container-Native**: Docker-first development and deployment
5. **Direct SQL**: No ORM overhead for maximum performance
6. **Notebook-Friendly**: Works seamlessly in Jupyter and Google Colab
7. **Production-Ready**: Standards for deploying LLM apps at scale

## When to Use This Profile

This profile is ideal for:

- ✅ LLM-powered applications (chatbots, agents, RAG systems)
- ✅ Machine learning model deployment
- ✅ Jupyter notebook development
- ✅ Vector search and semantic similarity
- ✅ Graph database applications
- ✅ Container-based deployments
- ✅ Projects using multiple LLM providers
- ✅ Google Colab development
- ✅ FastAPI + MCP server development

Consider alternatives for:
- ❌ Traditional CRUD web applications (use general Python profile)
- ❌ Django projects (Django has its own conventions)
- ❌ Legacy Python 2 projects
- ❌ Simple scripts (may be overkill)

## Example Project Structure

```
my-llm-project/
├── .env                                # Environment variables (not in git)
├── .gitignore                          # Python + Jupyter gitignore
├── docker-compose.yml                  # Local development services
├── Dockerfile                          # Production container
├── pyproject.toml                      # Poetry dependencies
├── poetry.lock                         # Locked dependencies
├── README.md                           # Project documentation
├── app/                                # Application code
│   ├── __init__.py
│   ├── main.py                        # FastAPI app
│   ├── api/                           # API routes
│   │   ├── llm.py                     # LLM endpoints
│   │   ├── embeddings.py              # Embedding endpoints
│   │   └── mcp.py                     # MCP server endpoints
│   ├── services/                      # Business logic
│   │   ├── llm_client.py              # LLM client
│   │   ├── embeddings.py              # Embedding service
│   │   └── vector_search.py           # Vector search
│   ├── db/                            # Database
│   │   ├── postgres.py                # PostgreSQL connection
│   │   └── neo4j.py                   # Neo4j connection
│   └── models/                        # Pydantic models
│       └── schemas.py                 # Request/response schemas
├── notebooks/                          # Jupyter notebooks
│   ├── exploration.ipynb              # Data exploration
│   └── experiments.ipynb              # LLM experiments
├── tests/                             # Tests
│   ├── test_api.py
│   ├── test_llm.py
│   └── test_embeddings.py
└── data/                              # Data files (gitignored)
```

## Quick Start

1. **Clone and setup**:
   ```bash
   poetry install
   cp .env.example .env  # Add your API keys
   ```

2. **Start services**:
   ```bash
   docker-compose up -d  # Start PostgreSQL, Redis, Neo4j, ChromaDB
   ```

3. **Run development server**:
   ```bash
   poetry run uvicorn app.main:app --reload
   ```

4. **Open Jupyter**:
   ```bash
   poetry run jupyter lab
   ```

5. **View API docs**:
   - Swagger UI: http://localhost:8000/docs
   - ReDoc: http://localhost:8000/redoc

## Resources

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [LangChain Documentation](https://python.langchain.com/)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [Anthropic Claude Documentation](https://docs.anthropic.com/)
- [ChromaDB Documentation](https://docs.trychroma.com/)
- [Neo4j Python Driver](https://neo4j.com/docs/python-manual/current/)

## Contributing

This profile is designed to be a living document. As LLM technology and best practices evolve, these standards should be updated accordingly.

## License

This profile is part of the Agent OS project and follows its licensing terms.
