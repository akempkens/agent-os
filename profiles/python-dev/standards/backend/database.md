## PostgreSQL Database Standards (Direct SQL, No ORM)

### Database Philosophy
- **Direct SQL**: Use raw SQL queries with asyncpg (no ORM abstraction)
- **Type Safety**: Use Pydantic models for validation, not database mapping
- **Performance**: Direct SQL provides better performance and control
- **Simplicity**: Fewer layers of abstraction, easier debugging

### Database Connection with asyncpg
- **Connection Pooling**: Always use connection pools in production
- **Async Operations**: Use asyncpg for async PostgreSQL operations
- **Configuration**: Store connection details in environment variables

```python
import asyncpg
import os
from typing import Optional

# Global connection pool
_db_pool: Optional[asyncpg.Pool] = None

async def init_db_pool() -> asyncpg.Pool:
    """Initialize database connection pool."""
    global _db_pool

    if _db_pool is None:
        _db_pool = await asyncpg.create_pool(
            host=os.getenv("DB_HOST", "localhost"),
            port=int(os.getenv("DB_PORT", "5432")),
            database=os.getenv("DB_NAME", "mydb"),
            user=os.getenv("DB_USER", "postgres"),
            password=os.getenv("DB_PASSWORD"),
            min_size=5,
            max_size=20,
            command_timeout=60
        )

    return _db_pool

async def get_db_pool() -> asyncpg.Pool:
    """Get existing database pool or create new one."""
    if _db_pool is None:
        return await init_db_pool()
    return _db_pool

async def close_db_pool():
    """Close database connection pool."""
    global _db_pool
    if _db_pool:
        await _db_pool.close()
        _db_pool = None
```

### Schema Design
- **Use Migrations**: Use migration tools (e.g., alembic, flyway) for schema changes
- **Descriptive Names**: Use clear, descriptive table and column names
- **Snake Case**: Use snake_case for table and column names
- **Primary Keys**: Always use primary keys (prefer SERIAL or UUID)
- **Indexes**: Index frequently queried columns

```sql
-- Example: Documents table with vector embeddings
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    content TEXT NOT NULL,
    embedding vector(1536),  -- pgvector for embeddings
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Indexes for common queries
CREATE INDEX idx_documents_created_at ON documents(created_at DESC);
CREATE INDEX idx_documents_metadata ON documents USING GIN(metadata);

-- Vector similarity search index (using pgvector)
CREATE INDEX idx_documents_embedding ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

### pgvector for Vector Embeddings
- **Install Extension**: Use pgvector extension for vector operations
- **Vector Columns**: Store embeddings as vector type
- **Similarity Search**: Use distance operators for semantic search

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Example: Semantic search using cosine distance
SELECT id, title, content,
       embedding <-> $1::vector AS distance
FROM documents
ORDER BY distance
LIMIT 10;
```

```python
import asyncpg
from typing import List

async def search_similar_documents(
    conn: asyncpg.Connection,
    query_embedding: List[float],
    limit: int = 10
) -> List[dict]:
    """Search for similar documents using vector embeddings."""
    # Convert Python list to PostgreSQL vector
    results = await conn.fetch(
        """
        SELECT id, title, content,
               embedding <-> $1::vector AS distance
        FROM documents
        ORDER BY distance
        LIMIT $2
        """,
        query_embedding,
        limit
    )

    return [dict(row) for row in results]
```

### CRUD Operations
- **Parameterized Queries**: Always use parameterized queries ($1, $2, etc.)
- **RETURNING Clause**: Use RETURNING to get inserted/updated data
- **Error Handling**: Handle database errors appropriately

```python
import asyncpg
from datetime import datetime
from typing import Optional, List

# CREATE
async def create_document(
    conn: asyncpg.Connection,
    title: str,
    content: str,
    embedding: List[float],
    metadata: dict = None
) -> dict:
    """Create a new document."""
    row = await conn.fetchrow(
        """
        INSERT INTO documents (title, content, embedding, metadata)
        VALUES ($1, $2, $3::vector, $4)
        RETURNING id, title, content, created_at
        """,
        title,
        content,
        embedding,
        metadata or {}
    )
    return dict(row)

# READ
async def get_document_by_id(
    conn: asyncpg.Connection,
    doc_id: int
) -> Optional[dict]:
    """Get document by ID."""
    row = await conn.fetchrow(
        """
        SELECT id, title, content, metadata, created_at, updated_at
        FROM documents
        WHERE id = $1
        """,
        doc_id
    )
    return dict(row) if row else None

async def list_documents(
    conn: asyncpg.Connection,
    offset: int = 0,
    limit: int = 100
) -> List[dict]:
    """List documents with pagination."""
    rows = await conn.fetch(
        """
        SELECT id, title, content, created_at
        FROM documents
        ORDER BY created_at DESC
        OFFSET $1 LIMIT $2
        """,
        offset,
        limit
    )
    return [dict(row) for row in rows]

# UPDATE
async def update_document(
    conn: asyncpg.Connection,
    doc_id: int,
    title: Optional[str] = None,
    content: Optional[str] = None
) -> Optional[dict]:
    """Update document fields."""
    # Build dynamic update query
    updates = []
    params = []
    param_count = 1

    if title is not None:
        updates.append(f"title = ${param_count}")
        params.append(title)
        param_count += 1

    if content is not None:
        updates.append(f"content = ${param_count}")
        params.append(content)
        param_count += 1

    if not updates:
        return await get_document_by_id(conn, doc_id)

    updates.append(f"updated_at = NOW()")
    params.append(doc_id)

    query = f"""
        UPDATE documents
        SET {', '.join(updates)}
        WHERE id = ${param_count}
        RETURNING id, title, content, updated_at
    """

    row = await conn.fetchrow(query, *params)
    return dict(row) if row else None

# DELETE
async def delete_document(
    conn: asyncpg.Connection,
    doc_id: int
) -> bool:
    """Delete document by ID."""
    result = await conn.execute(
        "DELETE FROM documents WHERE id = $1",
        doc_id
    )
    # Returns "DELETE N" where N is number of deleted rows
    return result.split()[-1] != "0"
```

### Transactions
- **Use Transactions**: Wrap multiple operations in transactions for atomicity
- **async with**: Use async context managers for automatic rollback
- **Savepoints**: Use savepoints for nested transactions

```python
async def create_document_with_tags(
    pool: asyncpg.Pool,
    title: str,
    content: str,
    tags: List[str]
) -> dict:
    """Create document and associated tags in a transaction."""
    async with pool.acquire() as conn:
        async with conn.transaction():
            # Insert document
            doc_row = await conn.fetchrow(
                """
                INSERT INTO documents (title, content)
                VALUES ($1, $2)
                RETURNING id, title, content, created_at
                """,
                title,
                content
            )
            doc_id = doc_row["id"]

            # Insert tags
            if tags:
                await conn.executemany(
                    """
                    INSERT INTO document_tags (document_id, tag)
                    VALUES ($1, $2)
                    """,
                    [(doc_id, tag) for tag in tags]
                )

            return dict(doc_row)
```

### Bulk Operations
- **executemany**: Use for bulk inserts
- **COPY**: Use COPY for very large bulk operations
- **Batch Processing**: Process large datasets in batches

```python
async def bulk_insert_documents(
    conn: asyncpg.Connection,
    documents: List[tuple]
) -> int:
    """Bulk insert documents."""
    # documents is list of tuples: [(title, content), ...]
    result = await conn.executemany(
        """
        INSERT INTO documents (title, content)
        VALUES ($1, $2)
        """,
        documents
    )
    return len(documents)

async def copy_documents_from_csv(
    conn: asyncpg.Connection,
    csv_data: str
) -> int:
    """Bulk import using COPY (fastest for large datasets)."""
    result = await conn.copy_to_table(
        'documents',
        source=csv_data,
        columns=['title', 'content'],
        format='csv'
    )
    return result
```

### JSON/JSONB Operations
- **JSONB**: Use JSONB for flexible metadata storage
- **Indexing**: Use GIN indexes for JSONB queries
- **Operators**: Use PostgreSQL's rich JSONB operators

```python
async def query_by_metadata(
    conn: asyncpg.Connection,
    key: str,
    value: any
) -> List[dict]:
    """Query documents by metadata field."""
    rows = await conn.fetch(
        """
        SELECT id, title, content, metadata
        FROM documents
        WHERE metadata @> $1::jsonb
        """,
        {key: value}  # Automatically converted to JSONB
    )
    return [dict(row) for row in rows]

async def update_metadata(
    conn: asyncpg.Connection,
    doc_id: int,
    new_fields: dict
) -> Optional[dict]:
    """Update metadata fields (merge with existing)."""
    row = await conn.fetchrow(
        """
        UPDATE documents
        SET metadata = metadata || $1::jsonb
        WHERE id = $2
        RETURNING id, metadata
        """,
        new_fields,
        doc_id
    )
    return dict(row) if row else None
```

### Complex Queries with CTEs
- **Common Table Expressions**: Use CTEs for complex queries
- **Readability**: CTEs improve query readability
- **Performance**: CTEs can improve query planning

```python
async def get_document_stats(conn: asyncpg.Connection) -> dict:
    """Get document statistics using CTE."""
    row = await conn.fetchrow(
        """
        WITH doc_stats AS (
            SELECT
                COUNT(*) as total_docs,
                COUNT(DISTINCT metadata->>'category') as categories,
                AVG(LENGTH(content)) as avg_content_length
            FROM documents
        ),
        recent_docs AS (
            SELECT COUNT(*) as recent_count
            FROM documents
            WHERE created_at > NOW() - INTERVAL '7 days'
        )
        SELECT
            ds.total_docs,
            ds.categories,
            ds.avg_content_length,
            rd.recent_count
        FROM doc_stats ds, recent_docs rd
        """
    )
    return dict(row)
```

### Full-Text Search
- **tsvector**: Use PostgreSQL's full-text search
- **Indexes**: Create GIN indexes on tsvector columns
- **Language**: Specify language for proper stemming

```sql
-- Add tsvector column for full-text search
ALTER TABLE documents
ADD COLUMN search_vector tsvector
GENERATED ALWAYS AS (
    to_tsvector('english', title || ' ' || content)
) STORED;

-- Create GIN index for fast full-text search
CREATE INDEX idx_documents_search ON documents USING GIN(search_vector);
```

```python
async def search_documents_fulltext(
    conn: asyncpg.Connection,
    query: str,
    limit: int = 10
) -> List[dict]:
    """Full-text search on documents."""
    rows = await conn.fetch(
        """
        SELECT id, title, content,
               ts_rank(search_vector, query) AS rank
        FROM documents,
             to_tsquery('english', $1) query
        WHERE search_vector @@ query
        ORDER BY rank DESC
        LIMIT $2
        """,
        query,
        limit
    )
    return [dict(row) for row in rows]
```

### Database Migrations
- **Use Migration Tools**: Use Alembic or similar for schema changes
- **Version Control**: Keep migrations in version control
- **Reversible**: Make migrations reversible when possible

```python
# alembic/versions/001_initial.py
"""Initial schema

Revision ID: 001
Create Date: 2024-01-01 00:00:00
"""

def upgrade():
    """Create initial tables."""
    op.execute("""
        CREATE TABLE documents (
            id SERIAL PRIMARY KEY,
            title VARCHAR(500) NOT NULL,
            content TEXT NOT NULL,
            created_at TIMESTAMP DEFAULT NOW()
        )
    """)

def downgrade():
    """Drop initial tables."""
    op.execute("DROP TABLE documents")
```

### Error Handling
- **Specific Exceptions**: Catch specific asyncpg exceptions
- **Unique Violations**: Handle unique constraint violations
- **Retry Logic**: Implement retry logic for transient errors

```python
import asyncpg

async def create_document_safe(
    conn: asyncpg.Connection,
    title: str,
    content: str
) -> Optional[dict]:
    """Create document with error handling."""
    try:
        row = await conn.fetchrow(
            """
            INSERT INTO documents (title, content)
            VALUES ($1, $2)
            RETURNING id, title, content, created_at
            """,
            title,
            content
        )
        return dict(row)

    except asyncpg.UniqueViolationError:
        # Handle duplicate title or other unique constraint violation
        print(f"Document with title '{title}' already exists")
        return None

    except asyncpg.PostgresError as e:
        # Handle other PostgreSQL errors
        print(f"Database error: {e}")
        raise
```

### Connection Management in FastAPI
- **Dependency Injection**: Use FastAPI dependencies for connections
- **Cleanup**: Ensure connections are returned to pool

```python
from fastapi import Depends
from typing import AsyncGenerator

async def get_db_connection() -> AsyncGenerator[asyncpg.Connection, None]:
    """FastAPI dependency for database connection."""
    pool = await get_db_pool()
    async with pool.acquire() as connection:
        yield connection
```

### Performance Best Practices
- **Use Indexes**: Index all frequently queried columns
- **EXPLAIN ANALYZE**: Use EXPLAIN ANALYZE to optimize slow queries
- **Connection Pooling**: Always use connection pools
- **Prepared Statements**: asyncpg automatically uses prepared statements
- **Batch Operations**: Use bulk operations for multiple inserts/updates

```python
# Check query performance
async def explain_query(conn: asyncpg.Connection, query: str, *args):
    """Run EXPLAIN ANALYZE on a query."""
    explain_query = f"EXPLAIN ANALYZE {query}"
    results = await conn.fetch(explain_query, *args)
    for row in results:
        print(row[0])
```
