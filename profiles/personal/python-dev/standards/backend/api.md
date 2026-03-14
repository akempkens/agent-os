## Python API Standards for LLM & ML Applications

### FastAPI with OpenAPI 3.1+
- **OpenAPI Version**: Use OpenAPI 3.1+ specification (latest)
- **Auto-Documentation**: Leverage FastAPI's automatic schema generation
- **Type Safety**: Use Pydantic v2 models for all request/response validation
- **Async by Default**: Use async functions for all I/O-bound operations

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(
    title="LLM API",
    description="API for LLM-powered applications",
    version="1.0.0",
    openapi_version="3.1.0",  # Explicitly use OpenAPI 3.1
    docs_url="/docs",         # Swagger UI
    redoc_url="/redoc"        # ReDoc alternative
)

class PromptRequest(BaseModel):
    """Request schema for LLM prompts."""
    prompt: str
    model: str = "gpt-4"
    max_tokens: int = 1000
    temperature: float = 0.7

class PromptResponse(BaseModel):
    """Response schema for LLM completions."""
    response: str
    model: str
    tokens_used: int
```

### RESTful API Design
- **Resource-Based URLs**: Use nouns for resources: `/prompts`, `/embeddings`, `/documents`
- **HTTP Methods**: Use appropriate methods:
  - GET: Retrieve resources
  - POST: Create new resources or trigger operations (LLM completions)
  - PUT: Replace entire resource
  - PATCH: Partial update
  - DELETE: Remove resource
- **Versioning**: Version APIs from day one: `/api/v1/prompts`

### MCP (Model Context Protocol) Server Support
- **MCP Integration**: FastAPI endpoints can serve as MCP servers
- **Standard Protocol**: Implement MCP protocol for AI tool integration
- **Tool Registration**: Expose LLM tools through MCP endpoints
- **Context Management**: Handle context windows and conversation state

```python
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

# MCP Server Router
mcp_router = APIRouter(prefix="/mcp/v1", tags=["mcp"])

class MCPToolRequest(BaseModel):
    """MCP tool invocation request."""
    tool_name: str
    parameters: dict[str, any]
    context: dict[str, any] | None = None

class MCPToolResponse(BaseModel):
    """MCP tool invocation response."""
    result: any
    metadata: dict[str, any]

@mcp_router.post("/tools/invoke", response_model=MCPToolResponse)
async def invoke_mcp_tool(request: MCPToolRequest) -> MCPToolResponse:
    """Invoke an MCP tool with context."""
    # Route to appropriate tool handler
    tool_handler = get_tool_handler(request.tool_name)
    if not tool_handler:
        raise HTTPException(status_code=404, detail=f"Tool {request.tool_name} not found")

    result = await tool_handler.execute(
        parameters=request.parameters,
        context=request.context
    )

    return MCPToolResponse(
        result=result,
        metadata={"tool": request.tool_name, "execution_time": result.get("time")}
    )

@mcp_router.get("/tools", response_model=list[dict])
async def list_mcp_tools():
    """List all available MCP tools."""
    return [
        {
            "name": "search_documents",
            "description": "Search vector database for relevant documents",
            "parameters": {"query": "string", "top_k": "integer"}
        },
        {
            "name": "execute_query",
            "description": "Execute SQL query on PostgreSQL",
            "parameters": {"query": "string"}
        }
    ]
```

### Gradio Frontend Integration
- **Gradio for Demos**: Use Gradio to create quick ML demo interfaces
- **API Backend**: FastAPI serves as backend, Gradio provides frontend
- **Separation**: Keep API logic separate from Gradio UI logic
- **Mounting**: Optionally mount Gradio app within FastAPI

```python
import gradio as gr
from fastapi import FastAPI

# FastAPI backend
app = FastAPI()

@app.post("/api/v1/generate")
async def generate_text(prompt: str, max_tokens: int = 100) -> dict:
    """Generate text using LLM."""
    # Your LLM logic here
    response = await llm_client.generate(prompt=prompt, max_tokens=max_tokens)
    return {"text": response.text, "tokens": response.tokens}

# Gradio interface
def gradio_generate(prompt: str, max_tokens: int) -> str:
    """Gradio wrapper for text generation."""
    import requests
    response = requests.post(
        "http://localhost:8000/api/v1/generate",
        json={"prompt": prompt, "max_tokens": max_tokens}
    )
    return response.json()["text"]

# Create Gradio interface
demo = gr.Interface(
    fn=gradio_generate,
    inputs=[
        gr.Textbox(label="Prompt", lines=5),
        gr.Slider(minimum=10, maximum=500, value=100, label="Max Tokens")
    ],
    outputs=gr.Textbox(label="Generated Text", lines=10),
    title="LLM Text Generator",
    description="Generate text using our LLM API"
)

# Mount Gradio app in FastAPI (optional)
app = gr.mount_gradio_app(app, demo, path="/ui")

# Or run separately:
# if __name__ == "__main__":
#     demo.launch(server_port=7860)
```

### Database Integration (PostgreSQL Direct)
- **No ORM**: Use direct SQL with asyncpg for database operations
- **Connection Pooling**: Use asyncpg connection pools
- **Dependency Injection**: Inject database connections via FastAPI dependencies
- **Transactions**: Use asyncpg transactions for atomic operations

```python
import asyncpg
from fastapi import Depends
from typing import AsyncGenerator

# Database connection pool (global)
db_pool: asyncpg.Pool | None = None

async def get_db_pool() -> asyncpg.Pool:
    """Get database connection pool."""
    global db_pool
    if db_pool is None:
        db_pool = await asyncpg.create_pool(
            host="localhost",
            database="mydb",
            user="user",
            password="password",
            min_size=5,
            max_size=20
        )
    return db_pool

async def get_db_connection() -> AsyncGenerator[asyncpg.Connection, None]:
    """Dependency to get database connection."""
    pool = await get_db_pool()
    async with pool.acquire() as connection:
        yield connection

@router.post("/documents", response_model=DocumentResponse)
async def create_document(
    doc: DocumentCreate,
    conn: asyncpg.Connection = Depends(get_db_connection)
) -> DocumentResponse:
    """Create a new document."""
    # Direct SQL execution
    row = await conn.fetchrow(
        """
        INSERT INTO documents (title, content, embedding)
        VALUES ($1, $2, $3)
        RETURNING id, title, content, created_at
        """,
        doc.title,
        doc.content,
        doc.embedding
    )

    return DocumentResponse(
        id=row["id"],
        title=row["title"],
        content=row["content"],
        created_at=row["created_at"]
    )
```

### LLM API Patterns
- **Streaming Responses**: Use Server-Sent Events (SSE) for streaming LLM outputs
- **Token Counting**: Always count and return token usage
- **Error Handling**: Handle LLM provider errors gracefully
- **Rate Limiting**: Implement rate limiting for expensive LLM calls

```python
from fastapi.responses import StreamingResponse
from typing import AsyncIterator
import asyncio

async def stream_llm_response(prompt: str) -> AsyncIterator[str]:
    """Stream LLM response token by token."""
    async for chunk in llm_client.stream(prompt=prompt):
        # Yield Server-Sent Event format
        yield f"data: {chunk.text}\n\n"
        await asyncio.sleep(0.01)  # Small delay to prevent overwhelming client

@router.post("/stream")
async def stream_completion(request: PromptRequest):
    """Stream LLM completion using SSE."""
    return StreamingResponse(
        stream_llm_response(request.prompt),
        media_type="text/event-stream"
    )
```

### Vector Search & Embeddings API
- **Embedding Endpoints**: Expose endpoints for text embeddings
- **Vector Search**: Implement semantic search over vector databases
- **Batch Operations**: Support batch embedding and search

```python
from pydantic import BaseModel

class EmbeddingRequest(BaseModel):
    """Request for text embeddings."""
    texts: list[str]
    model: str = "text-embedding-ada-002"

class EmbeddingResponse(BaseModel):
    """Response with embeddings."""
    embeddings: list[list[float]]
    model: str
    tokens_used: int

@router.post("/embeddings", response_model=EmbeddingResponse)
async def create_embeddings(
    request: EmbeddingRequest
) -> EmbeddingResponse:
    """Generate embeddings for texts."""
    embeddings = await embedding_service.embed_texts(
        texts=request.texts,
        model=request.model
    )

    return EmbeddingResponse(
        embeddings=embeddings.vectors,
        model=request.model,
        tokens_used=embeddings.tokens
    )

class SearchRequest(BaseModel):
    """Vector search request."""
    query: str
    top_k: int = 5
    filters: dict[str, any] | None = None

class SearchResponse(BaseModel):
    """Vector search results."""
    results: list[dict]
    query_embedding: list[float] | None = None

@router.post("/search", response_model=SearchResponse)
async def search_documents(
    request: SearchRequest,
    conn: asyncpg.Connection = Depends(get_db_connection)
) -> SearchResponse:
    """Semantic search using vector embeddings."""
    # Generate query embedding
    query_embedding = await embedding_service.embed_text(request.query)

    # Vector search using pgvector or external vector DB
    results = await conn.fetch(
        """
        SELECT id, title, content,
               embedding <-> $1::vector AS distance
        FROM documents
        ORDER BY distance
        LIMIT $2
        """,
        query_embedding,
        request.top_k
    )

    return SearchResponse(
        results=[dict(row) for row in results],
        query_embedding=query_embedding
    )
```

### Authentication & Security
- **API Keys**: Use API key authentication for LLM endpoints
- **Rate Limiting**: Implement per-user rate limits
- **CORS**: Configure CORS for web frontends (Gradio)
- **Input Validation**: Validate and sanitize all inputs

```python
from fastapi import Header, HTTPException, Security
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key")

async def verify_api_key(api_key: str = Security(api_key_header)):
    """Verify API key."""
    # Check against database or environment variable
    valid_key = await get_valid_api_key(api_key)
    if not valid_key:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return valid_key

@router.post("/protected-endpoint")
async def protected_endpoint(
    request: PromptRequest,
    api_key: str = Depends(verify_api_key)
):
    """Protected endpoint requiring API key."""
    # Your logic here
    pass
```

### Error Handling
- **Consistent Format**: Use consistent error response format
- **LLM Errors**: Handle provider-specific errors (rate limits, token limits)
- **Graceful Degradation**: Fallback to alternative models when primary fails

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class LLMError(Exception):
    """Base exception for LLM errors."""
    def __init__(self, message: str, status_code: int = 500):
        self.message = message
        self.status_code = status_code

@app.exception_handler(LLMError)
async def llm_error_handler(request: Request, exc: LLMError):
    """Handle LLM-specific errors."""
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": "LLM_ERROR",
            "message": exc.message,
            "type": "llm_error"
        }
    )
```

### Background Tasks
- **Async Tasks**: Use FastAPI BackgroundTasks for simple async operations
- **Long Operations**: Use Celery or modal for long-running LLM tasks
- **Status Tracking**: Provide endpoints to check task status

```python
from fastapi import BackgroundTasks

async def process_document_embeddings(document_id: int):
    """Background task to generate embeddings."""
    # Fetch document, generate embeddings, store
    pass

@router.post("/documents/{doc_id}/embed")
async def trigger_embedding(
    doc_id: int,
    background_tasks: BackgroundTasks
):
    """Trigger embedding generation in background."""
    background_tasks.add_task(process_document_embeddings, doc_id)
    return {"message": "Embedding generation started", "document_id": doc_id}
```

### OpenAPI 3.1 Documentation
- **Rich Descriptions**: Add detailed descriptions to all endpoints
- **Examples**: Include request/response examples in Pydantic models
- **Tags**: Organize endpoints with tags for better documentation
- **Metadata**: Add contact info, license, and version info

```python
app = FastAPI(
    title="LLM & Vector Search API",
    description="""
    API for LLM-powered applications with vector search capabilities.

    ## Features
    * Text generation with multiple LLM providers
    * Vector embeddings and semantic search
    * Document processing and storage
    * MCP server protocol support
    """,
    version="1.0.0",
    openapi_version="3.1.0",
    contact={
        "name": "API Support",
        "email": "support@example.com"
    },
    license_info={
        "name": "MIT"
    }
)
```

### Container Deployment
- **Health Checks**: Implement health check endpoints for container orchestration
- **Graceful Shutdown**: Handle SIGTERM for graceful container shutdown
- **Environment Config**: Use environment variables for all configuration

```python
@app.get("/health")
async def health_check():
    """Health check endpoint for container orchestration."""
    # Check database connection, LLM provider, etc.
    db_healthy = await check_database_health()
    llm_healthy = await check_llm_provider()

    if not (db_healthy and llm_healthy):
        return JSONResponse(
            status_code=503,
            content={"status": "unhealthy", "database": db_healthy, "llm": llm_healthy}
        )

    return {"status": "healthy"}

@app.on_event("shutdown")
async def shutdown_event():
    """Cleanup on shutdown."""
    global db_pool
    if db_pool:
        await db_pool.close()
```
