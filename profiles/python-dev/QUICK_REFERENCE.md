# Python LLM/ML Development - Quick Reference

**Compact reference for AI agents. For detailed examples, see individual standard files.**

## Core Stack
- **Python**: 3.12+ only
- **Package Manager**: Poetry or pip + requirements.txt
- **Database**: PostgreSQL 16+ (direct SQL with asyncpg, no ORM)
- **API**: FastAPI + OpenAPI 3.1
- **LLMs**: OpenAI, Anthropic, Google, Ollama via LangChain
- **Deployment**: Docker-first, Google Colab compatible

## Code Style (PEP 8 + Modern)
```python
# Use modern type hints (3.10+ syntax)
def process_data(items: list[str], config: dict[str, int]) -> list[dict]:
    pass

# Async for I/O operations
async def fetch_data(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()

# Pydantic for validation
from pydantic import BaseModel, Field

class UserCreate(BaseModel):
    email: str
    age: int = Field(ge=18, le=120)
```

**Tools**: Black (format), Ruff (lint), mypy (types)

## Project Structure
```
project/
├── pyproject.toml              # Poetry deps
├── .env                        # Secrets (not in git)
├── docker-compose.yml          # Local services
├── Dockerfile                  # Production container
├── app/
│   ├── main.py                # FastAPI app
│   ├── api/                   # Routes (llm.py, embeddings.py)
│   ├── services/              # Business logic
│   ├── db/                    # DB connections (postgres.py, neo4j.py)
│   └── models/                # Pydantic schemas
├── notebooks/                  # Jupyter notebooks
└── tests/                      # pytest tests
```

## Database (PostgreSQL Direct SQL)

**Connection Pool**:
```python
import asyncpg

pool = await asyncpg.create_pool(
    "postgresql://user:pass@localhost/db",
    min_size=5, max_size=20
)

async def get_doc(doc_id: int) -> dict:
    async with pool.acquire() as conn:
        row = await conn.fetchrow(
            "SELECT * FROM documents WHERE id = $1", doc_id
        )
        return dict(row) if row else None
```

**Vector Search (pgvector)**:
```python
# Semantic search
results = await conn.fetch("""
    SELECT id, title, 1 - (embedding <-> $1::vector) AS similarity
    FROM documents
    ORDER BY embedding <-> $1::vector
    LIMIT $2
""", query_embedding, limit)
```

**JSONB Queries**:
```python
# Query by metadata field
await conn.fetch(
    "SELECT * FROM documents WHERE metadata @> $1::jsonb",
    {"category": "ml"}
)
```

## FastAPI + OpenAPI 3.1

```python
from fastapi import FastAPI, Depends
from pydantic import BaseModel

app = FastAPI(
    title="LLM API",
    version="1.0.0",
    openapi_version="3.1.0"  # Latest spec
)

# Pydantic models
class PromptRequest(BaseModel):
    prompt: str
    model: str = "gpt-4"
    temperature: float = 0.7

# Async endpoint
@app.post("/generate")
async def generate(req: PromptRequest) -> dict:
    response = await llm_client.generate(req.prompt)
    return {"response": response}

# Health check
@app.get("/health")
async def health():
    return {"status": "healthy"}
```

**MCP Server Pattern**:
```python
@app.post("/mcp/v1/tools/invoke")
async def invoke_tool(tool_name: str, parameters: dict) -> dict:
    handler = get_tool_handler(tool_name)
    result = await handler.execute(parameters)
    return {"result": result}
```

## LLM Integration

**Multi-Provider Setup**:
```python
from openai import AsyncOpenAI
from anthropic import AsyncAnthropic

openai_client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))
anthropic_client = AsyncAnthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

async def generate_openai(prompt: str) -> str:
    response = await openai_client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content
```

**Error Handling + Retry**:
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=2, max=10))
async def llm_call_with_retry(prompt: str) -> str:
    return await openai_client.chat.completions.create(...)
```

**Token Counting**:
```python
import tiktoken

encoding = tiktoken.encoding_for_model("gpt-4")
token_count = len(encoding.encode(text))
```

**RAG Pattern**:
```python
# 1. Retrieve relevant docs from vector DB
docs = await vector_search(query_embedding, top_k=3)

# 2. Build context
context = "\n\n".join([doc["content"] for doc in docs])

# 3. Generate with context
prompt = f"Context:\n{context}\n\nQuestion: {query}\nAnswer:"
response = await llm_client.generate(prompt)
```

## Jupyter Notebooks

**Setup Cell**:
```python
# Standard imports
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from tqdm.notebook import tqdm

# LLM imports
from openai import AsyncOpenAI
from langchain.chains import LLMChain

# Config
%load_ext autoreload
%autoreload 2
pd.set_option('display.max_columns', None)

# Load env
from dotenv import load_dotenv
load_dotenv()
```

**Google Colab Compatibility**:
```python
try:
    import google.colab
    IN_COLAB = True
    from google.colab import drive
    drive.mount('/content/drive')
    !pip install -q openai langchain
except:
    IN_COLAB = False
```

**Progress Tracking**:
```python
results = []
for item in tqdm(data, desc="Processing"):
    result = await process_item(item)
    results.append(result)
```

## Graph Database (Neo4j)

```python
from neo4j import AsyncGraphDatabase

driver = AsyncGraphDatabase.driver(
    "bolt://localhost:7687",
    auth=("neo4j", "password")
)

# Create node
async def create_node(doc_id: int, title: str):
    async with driver.session() as session:
        await session.run(
            "CREATE (d:Document {id: $id, title: $title})",
            id=doc_id, title=title
        )

# Create relationship
async def create_citation(from_id: int, to_id: int):
    async with driver.session() as session:
        await session.run("""
            MATCH (from:Document {id: $from_id})
            MATCH (to:Document {id: $to_id})
            CREATE (from)-[:CITES]->(to)
        """, from_id=from_id, to_id=to_id)

# Query with pattern
async def find_related(doc_id: int):
    async with driver.session() as session:
        result = await session.run("""
            MATCH (d:Document {id: $id})-[:CITES]->(cited:Document)
            RETURN cited
        """, id=doc_id)
        return [dict(record["cited"]) async for record in result]
```

## Docker Deployment

**Multi-Stage Dockerfile**:
```dockerfile
FROM python:3.12-slim AS builder
RUN pip install poetry
WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN poetry install --no-dev

FROM python:3.12-slim
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
WORKDIR /app
COPY . .
RUN useradd -r appuser && chown -R appuser:appuser /app
USER appuser
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**docker-compose.yml**:
```yaml
services:
  app:
    build: .
    ports: ["8000:8000"]
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/db
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      - postgres
      - redis

  postgres:
    image: pgvector/pgvector:pg16
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  postgres_data:
```

## Testing

```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_api_endpoint(client: AsyncClient):
    response = await client.post(
        "/generate",
        json={"prompt": "test", "model": "gpt-4"}
    )
    assert response.status_code == 200
    assert "response" in response.json()

# Fixtures
@pytest.fixture
async def db_conn():
    pool = await asyncpg.create_pool("postgresql://...")
    async with pool.acquire() as conn:
        yield conn
    await pool.close()
```

## Environment Configuration

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    openai_api_key: str
    anthropic_api_key: str | None = None
    neo4j_uri: str = "bolt://localhost:7687"

    class Config:
        env_file = ".env"

settings = Settings()
```

## Key Principles

1. **Type Safety**: Full type hints, mypy strict mode
2. **Async First**: Use async/await for all I/O operations
3. **Direct SQL**: No ORM, use asyncpg directly
4. **Container Native**: Docker for all services
5. **Token Awareness**: Track and optimize LLM token usage
6. **Error Handling**: Retry logic, graceful degradation
7. **Environment Config**: All secrets in .env (not in code)
8. **Health Checks**: Implement /health, /ready, /live endpoints

## Common Patterns

**Batch LLM Processing**:
```python
from asyncio import Semaphore

async def process_batch(prompts: list[str], max_concurrent: int = 5):
    sem = Semaphore(max_concurrent)

    async def process_one(prompt):
        async with sem:
            return await llm_client.generate(prompt)

    return await asyncio.gather(*[process_one(p) for p in prompts])
```

**Streaming Response**:
```python
from fastapi.responses import StreamingResponse

async def stream_llm(prompt: str):
    async for chunk in llm_client.stream(prompt):
        yield f"data: {chunk}\n\n"

@app.post("/stream")
async def stream_endpoint(prompt: str):
    return StreamingResponse(stream_llm(prompt), media_type="text/event-stream")
```

**Vector + Text Hybrid Search**:
```python
# Combine full-text search with vector similarity
results = await conn.fetch("""
    SELECT id, title,
           ts_rank(search_vector, query) * 0.3 +
           (1 - (embedding <-> $1::vector)) * 0.7 as score
    FROM documents,
         to_tsquery('english', $2) query
    WHERE search_vector @@ query
    ORDER BY score DESC
    LIMIT 10
""", query_embedding, query_text)
```

---

**For detailed examples and patterns, see:**
- `ml/llm-integration.md` - LLM provider integration details
- `ml/notebooks.md` - Jupyter notebook best practices
- `backend/database.md` - PostgreSQL patterns
- `backend/queries.md` - Advanced SQL queries
- `backend/graph-database.md` - Neo4j graph patterns
- `deployment/docker.md` - Container deployment details
