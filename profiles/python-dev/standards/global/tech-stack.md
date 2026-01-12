## Python Tech Stack for LLM & ML Development

Define your Python technical stack below. Optimized for LLM applications, Jupyter notebooks, and graph databases with container-first deployment.

### Language & Runtime
- **Python Version:** Python 3.12+ (latest stable - required for newest type hints and performance improvements)
- **Package Manager:** Poetry (modern dependency management with lock files) or pip with requirements.txt
- **Virtual Environment:** Poetry, venv, or conda (for complex ML dependencies)

### LLM & AI Frameworks
- **LLM Orchestration:** LangChain (primary framework for LLM applications)
- **LLM Providers:**
  - OpenAI SDK (`openai`) - GPT-4, GPT-3.5, embeddings
  - Anthropic SDK (`anthropic`) - Claude models
  - Google Generative AI (`google-generativeai`) - Gemini models
  - Ollama (`ollama`) - Local LLM deployment
- **LLM Extensions:**
  - `langchain-openai` - OpenAI integration
  - `langchain-experimental` - Experimental features
  - `langchain-chroma` - ChromaDB integration
  - `tiktoken` - Token counting and optimization
- **Vector Databases:**
  - ChromaDB (`chromadb`) - Primary vector store
  - DocArray (`langchain[docarray]`) - Multi-backend vector search
- **Embeddings:** Sentence Transformers (`sentence_transformers`) - Local embeddings

### Machine Learning & Data Science
- **Deep Learning:** PyTorch (`torch`) - Primary ML framework
- **Transformers:** HuggingFace Transformers (`transformers`) - Pre-trained models
- **Model Optimization:**
  - `accelerate` - Distributed training and inference
  - `bitsandbytes` - Quantization and optimization
  - `sentencepiece` - Tokenization
- **Data Processing:**
  - NumPy (`numpy`) - Numerical computing
  - Pandas (`pandas`) - Data manipulation
  - SciPy (`scipy`) - Scientific computing
  - Scikit-learn (`scikit-learn`) - Machine learning utilities
- **Datasets:** HuggingFace Datasets (`datasets`) - Dataset loading and processing
- **NLP:** Gensim (`gensim`) - Topic modeling and document similarity

### Interactive Development
- **Notebooks:** JupyterLab (`jupyterlab`) - Interactive development environment
- **Notebook Widgets:** ipywidgets (`ipywidgets`) - Interactive UI components
- **Dashboards:**
  - Jupyter Dash (`jupyter-dash`) - Plotly Dash in Jupyter
  - Gradio (`gradio`) - Quick ML demo interfaces

### Web Frameworks & APIs
- **API Framework:** FastAPI (async, OpenAPI 3.1, auto-documentation)
- **MCP Server Support:** FastAPI with custom MCP protocol handlers
- **ASGI Server:** Uvicorn (production) or Hypercorn
- **Frontend/UI:** Gradio (ML demos and interfaces) or Streamlit
- **OpenAPI:** OpenAPI 3.1+ specification with FastAPI auto-generation

### Database & Storage
- **Primary Database:** PostgreSQL 16+ (direct SQL, no ORM)
- **Database Driver:** asyncpg (async PostgreSQL driver) or psycopg3
- **Graph Database:** Neo4j (with neo4j-driver) or Neptune
- **Vector Storage:** ChromaDB, Pinecone, or Weaviate
- **Caching:** Redis (with redis-py or aioredis for async)

### Data Validation & Serialization
- **Validation:** Pydantic v2 (type-safe data validation and settings management)
- **API Serialization:** Pydantic models with FastAPI
- **Configuration:** python-dotenv (environment variables) + Pydantic Settings

### Code Quality & Formatting
- **Formatter:** Black (uncompromising code formatter)
- **Linter:** Ruff (extremely fast Python linter, replaces flake8, isort, pylint)
- **Type Checker:** mypy (static type checking) or pyright
- **Import Sorting:** Ruff (built-in)
- **Pre-commit Hooks:** pre-commit (automate code quality checks)

### Testing & Quality Assurance
- **Test Framework:** pytest (with pytest-asyncio for async tests)
- **Coverage:** pytest-cov (code coverage reporting)
- **Mocking:** pytest-mock or unittest.mock
- **HTTP Testing:** httpx (async-compatible) or requests
- **Notebook Testing:** nbval or papermill for testing notebooks

### Development Tools
- **HTTP Client:** requests (synchronous) or httpx (async-compatible)
- **CLI Tools:** Click or Typer (type-safe CLI with Pydantic)
- **Progress Bars:** tqdm (progress tracking for ML workflows)
- **System Monitoring:** psutil (system and process utilities)
- **Network Testing:** speedtest-cli (network diagnostics)

### Data Parsing & Extraction
- **Document Processing:** unstructured (PDF, HTML, Word parsing)
- **Web Scraping:** BeautifulSoup4 (`beautifulsoup4`) - HTML/XML parsing
- **Feed Parsing:** feedparser (RSS/Atom feeds)
- **Audio Processing:** pydub (audio manipulation)

### Visualization & Plotting
- **Static Plots:** Matplotlib (`matplotlib`) - Primary plotting library
- **Interactive Plots:** Plotly (`plotly`) - Interactive visualizations
- **Notebook Integration:** Built-in Jupyter/IPython display

### Monitoring & Logging
- **Logging:** structlog (structured logging) or standard logging with JSON formatter
- **Error Tracking:** Sentry (sentry-sdk) or custom error handling
- **Metrics:** Custom metrics with PostgreSQL or Prometheus client

### Container & Deployment
- **Containerization:** Docker with multi-stage builds (primary deployment method)
- **Container Orchestration:** Docker Compose (development) or Kubernetes (production)
- **Serverless:** Modal (`modal`) - Serverless GPU compute
- **Cloud Notebooks:** Google Colab compatible (use pip install, avoid system dependencies)
- **ASGI Server:** Uvicorn or Hypercorn (containerized)
- **Hosting:**
  - Docker containers on GCP, AWS, or Azure
  - Google Colab for notebook development
  - Modal for GPU workloads
  - Railway or Fly.io for quick deployments

### API Documentation
- **Specification:** OpenAPI 3.1+ (latest specification)
- **Auto-Generation:** FastAPI automatic OpenAPI schema generation
- **Interactive Docs:**
  - Swagger UI (built-in with FastAPI at `/docs`)
  - ReDoc (built-in with FastAPI at `/redoc`)
- **Schema Validation:** Pydantic models ensure OpenAPI schema accuracy

### Development Setup Requirements
- **Core:** python-dotenv, setuptools
- **Minimal Install:** Notebooks should work in Google Colab with pip install
- **Container-First:** All services should be containerizable with Docker
- **Environment Agnostic:** Code should run locally, in containers, and in cloud notebooks
