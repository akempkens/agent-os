## PostgreSQL Query Best Practices (Direct SQL with asyncpg)

### Query Writing Philosophy
- **Raw SQL**: Write clear, readable SQL queries directly
- **Parameterized Queries**: Always use parameterized queries for security
- **Performance First**: Optimize queries for performance from the start
- **Type Safety**: Use Pydantic models to validate query results

### Basic Query Patterns
- **fetchrow()**: Fetch single row (returns Record or None)
- **fetch()**: Fetch multiple rows (returns list of Records)
- **fetchval()**: Fetch single value from single row
- **execute()**: Execute query without returning results

```python
import asyncpg
from typing import Optional, List

# Fetch single row
async def get_document(conn: asyncpg.Connection, doc_id: int) -> Optional[dict]:
    """Get single document by ID."""
    row = await conn.fetchrow(
        "SELECT id, title, content, created_at FROM documents WHERE id = $1",
        doc_id
    )
    return dict(row) if row else None

# Fetch multiple rows
async def list_all_documents(conn: asyncpg.Connection) -> List[dict]:
    """Get all documents."""
    rows = await conn.fetch(
        "SELECT id, title, content FROM documents ORDER BY created_at DESC"
    )
    return [dict(row) for row in rows]

# Fetch single value
async def count_documents(conn: asyncpg.Connection) -> int:
    """Count total documents."""
    count = await conn.fetchval("SELECT COUNT(*) FROM documents")
    return count

# Execute without result
async def mark_document_processed(conn: asyncpg.Connection, doc_id: int):
    """Mark document as processed."""
    await conn.execute(
        "UPDATE documents SET processed = true WHERE id = $1",
        doc_id
    )
```

### Filtering and Conditions
- **WHERE Clauses**: Use clear WHERE conditions with parameters
- **Multiple Conditions**: Combine with AND/OR
- **IN Clauses**: Use ANY for list conditions
- **Pattern Matching**: Use LIKE/ILIKE for text search

```python
# Simple filter
async def get_documents_by_status(
    conn: asyncpg.Connection,
    status: str
) -> List[dict]:
    """Get documents by status."""
    rows = await conn.fetch(
        "SELECT * FROM documents WHERE status = $1",
        status
    )
    return [dict(row) for row in rows]

# Multiple conditions
async def get_recent_documents(
    conn: asyncpg.Connection,
    days: int,
    status: str
) -> List[dict]:
    """Get recent documents with specific status."""
    rows = await conn.fetch(
        """
        SELECT * FROM documents
        WHERE created_at > NOW() - INTERVAL '$1 days'
          AND status = $2
        ORDER BY created_at DESC
        """,
        days,
        status
    )
    return [dict(row) for row in rows]

# IN clause using ANY
async def get_documents_by_ids(
    conn: asyncpg.Connection,
    doc_ids: List[int]
) -> List[dict]:
    """Get multiple documents by IDs."""
    rows = await conn.fetch(
        "SELECT * FROM documents WHERE id = ANY($1::int[])",
        doc_ids
    )
    return [dict(row) for row in rows]

# Pattern matching
async def search_documents(
    conn: asyncpg.Connection,
    search_term: str
) -> List[dict]:
    """Search documents by title or content."""
    pattern = f"%{search_term}%"
    rows = await conn.fetch(
        """
        SELECT * FROM documents
        WHERE title ILIKE $1 OR content ILIKE $1
        ORDER BY created_at DESC
        """,
        pattern
    )
    return [dict(row) for row in rows]
```

### Pagination
- **OFFSET/LIMIT**: Standard pagination
- **Cursor-Based**: For large datasets
- **Include Total Count**: Return total count for UI

```python
from typing import Tuple

async def paginate_documents(
    conn: asyncpg.Connection,
    page: int = 1,
    page_size: int = 20
) -> Tuple[List[dict], int]:
    """Paginate documents with total count."""
    offset = (page - 1) * page_size

    # Get total count
    total = await conn.fetchval("SELECT COUNT(*) FROM documents")

    # Get page of results
    rows = await conn.fetch(
        """
        SELECT id, title, content, created_at
        FROM documents
        ORDER BY created_at DESC
        OFFSET $1 LIMIT $2
        """,
        offset,
        page_size
    )

    return [dict(row) for row in rows], total

# Cursor-based pagination (better for large datasets)
async def cursor_paginate_documents(
    conn: asyncpg.Connection,
    cursor: Optional[int] = None,
    limit: int = 20
) -> List[dict]:
    """Cursor-based pagination using ID."""
    if cursor is None:
        # First page
        rows = await conn.fetch(
            """
            SELECT id, title, content, created_at
            FROM documents
            ORDER BY id DESC
            LIMIT $1
            """,
            limit
        )
    else:
        # Subsequent pages
        rows = await conn.fetch(
            """
            SELECT id, title, content, created_at
            FROM documents
            WHERE id < $1
            ORDER BY id DESC
            LIMIT $2
            """,
            cursor,
            limit
        )

    return [dict(row) for row in rows]
```

### Aggregations
- **COUNT, SUM, AVG**: Standard aggregations
- **GROUP BY**: Group and aggregate
- **HAVING**: Filter grouped results

```python
# Simple aggregation
async def get_document_stats(conn: asyncpg.Connection) -> dict:
    """Get document statistics."""
    row = await conn.fetchrow(
        """
        SELECT
            COUNT(*) as total,
            COUNT(*) FILTER (WHERE status = 'published') as published,
            AVG(LENGTH(content)) as avg_length
        FROM documents
        """
    )
    return dict(row)

# Group by
async def count_by_category(conn: asyncpg.Connection) -> List[dict]:
    """Count documents by category."""
    rows = await conn.fetch(
        """
        SELECT
            metadata->>'category' as category,
            COUNT(*) as count
        FROM documents
        GROUP BY metadata->>'category'
        ORDER BY count DESC
        """
    )
    return [dict(row) for row in rows]

# Group by with HAVING
async def popular_tags(conn: asyncpg.Connection, min_count: int = 5) -> List[dict]:
    """Get tags with minimum usage count."""
    rows = await conn.fetch(
        """
        SELECT tag, COUNT(*) as count
        FROM document_tags
        GROUP BY tag
        HAVING COUNT(*) >= $1
        ORDER BY count DESC
        """,
        min_count
    )
    return [dict(row) for row in rows]
```

### Joins
- **INNER JOIN**: Match records in both tables
- **LEFT JOIN**: All records from left table
- **Explicit ON**: Always use explicit join conditions

```python
# Inner join
async def get_documents_with_authors(
    conn: asyncpg.Connection
) -> List[dict]:
    """Get documents with author information."""
    rows = await conn.fetch(
        """
        SELECT
            d.id,
            d.title,
            d.content,
            u.username as author,
            u.email as author_email
        FROM documents d
        INNER JOIN users u ON d.author_id = u.id
        ORDER BY d.created_at DESC
        """
    )
    return [dict(row) for row in rows]

# Left join (include documents without tags)
async def get_documents_with_tags(
    conn: asyncpg.Connection
) -> List[dict]:
    """Get documents with their tags (if any)."""
    rows = await conn.fetch(
        """
        SELECT
            d.id,
            d.title,
            ARRAY_AGG(dt.tag) FILTER (WHERE dt.tag IS NOT NULL) as tags
        FROM documents d
        LEFT JOIN document_tags dt ON d.id = dt.document_id
        GROUP BY d.id, d.title
        ORDER BY d.created_at DESC
        """
    )
    return [dict(row) for row in rows]
```

### Vector Similarity Search (pgvector)
- **Cosine Distance**: Use <-> for cosine distance
- **L2 Distance**: Use <=> for Euclidean distance
- **Indexes**: Use IVFFlat or HNSW indexes for performance

```python
from typing import List

async def vector_search(
    conn: asyncpg.Connection,
    query_embedding: List[float],
    limit: int = 10,
    threshold: float = 0.8
) -> List[dict]:
    """Semantic search using vector embeddings."""
    rows = await conn.fetch(
        """
        SELECT
            id,
            title,
            content,
            1 - (embedding <-> $1::vector) AS similarity
        FROM documents
        WHERE 1 - (embedding <-> $1::vector) > $2
        ORDER BY embedding <-> $1::vector
        LIMIT $3
        """,
        query_embedding,
        threshold,
        limit
    )
    return [dict(row) for row in rows]

async def hybrid_search(
    conn: asyncpg.Connection,
    query_text: str,
    query_embedding: List[float],
    limit: int = 10
) -> List[dict]:
    """Hybrid search: full-text + vector similarity."""
    rows = await conn.fetch(
        """
        SELECT
            id,
            title,
            content,
            ts_rank(search_vector, query) as text_rank,
            1 - (embedding <-> $2::vector) as vector_similarity,
            (ts_rank(search_vector, query) * 0.3 +
             (1 - (embedding <-> $2::vector)) * 0.7) as combined_score
        FROM documents,
             to_tsquery('english', $1) query
        WHERE search_vector @@ query
        ORDER BY combined_score DESC
        LIMIT $3
        """,
        query_text,
        query_embedding,
        limit
    )
    return [dict(row) for row in rows]
```

### JSONB Queries
- **-> Operator**: Get JSON object/array
- **->> Operator**: Get JSON value as text
- **@> Operator**: Contains JSON
- **? Operator**: Key exists

```python
# Query by JSONB field
async def get_documents_by_metadata_field(
    conn: asyncpg.Connection,
    key: str,
    value: any
) -> List[dict]:
    """Get documents where metadata contains key-value pair."""
    rows = await conn.fetch(
        """
        SELECT id, title, metadata
        FROM documents
        WHERE metadata @> $1::jsonb
        """,
        {key: value}
    )
    return [dict(row) for row in rows]

# Check if key exists
async def get_documents_with_key(
    conn: asyncpg.Connection,
    key: str
) -> List[dict]:
    """Get documents where metadata has specific key."""
    rows = await conn.fetch(
        """
        SELECT id, title, metadata
        FROM documents
        WHERE metadata ? $1
        """,
        key
    )
    return [dict(row) for row in rows]

# Extract and filter by nested JSON
async def get_documents_by_nested_field(
    conn: asyncpg.Connection,
    category: str
) -> List[dict]:
    """Get documents by nested metadata field."""
    rows = await conn.fetch(
        """
        SELECT id, title, metadata->'details'->>'category' as category
        FROM documents
        WHERE metadata->'details'->>'category' = $1
        """,
        category
    )
    return [dict(row) for row in rows]
```

### Window Functions
- **ROW_NUMBER()**: Assign unique numbers
- **RANK()**: Ranking with gaps
- **PARTITION BY**: Group for window functions

```python
async def get_top_documents_per_category(
    conn: asyncpg.Connection,
    top_n: int = 3
) -> List[dict]:
    """Get top N documents per category based on views."""
    rows = await conn.fetch(
        """
        WITH ranked_docs AS (
            SELECT
                id,
                title,
                metadata->>'category' as category,
                view_count,
                ROW_NUMBER() OVER (
                    PARTITION BY metadata->>'category'
                    ORDER BY view_count DESC
                ) as rank
            FROM documents
        )
        SELECT id, title, category, view_count
        FROM ranked_docs
        WHERE rank <= $1
        ORDER BY category, rank
        """,
        top_n
    )
    return [dict(row) for row in rows]
```

### Common Table Expressions (CTEs)
- **WITH Clause**: Create temporary result sets
- **Readability**: Break complex queries into steps
- **Reusability**: Reference CTE multiple times

```python
async def get_document_engagement_metrics(
    conn: asyncpg.Connection,
    doc_id: int
) -> dict:
    """Get engagement metrics for a document using CTEs."""
    row = await conn.fetchrow(
        """
        WITH doc_stats AS (
            SELECT
                id,
                title,
                view_count,
                like_count
            FROM documents
            WHERE id = $1
        ),
        comment_stats AS (
            SELECT
                document_id,
                COUNT(*) as comment_count,
                AVG(sentiment_score) as avg_sentiment
            FROM comments
            WHERE document_id = $1
            GROUP BY document_id
        ),
        recent_activity AS (
            SELECT
                document_id,
                COUNT(*) as recent_views
            FROM document_views
            WHERE document_id = $1
              AND viewed_at > NOW() - INTERVAL '7 days'
            GROUP BY document_id
        )
        SELECT
            ds.id,
            ds.title,
            ds.view_count,
            ds.like_count,
            COALESCE(cs.comment_count, 0) as comments,
            COALESCE(cs.avg_sentiment, 0) as avg_sentiment,
            COALESCE(ra.recent_views, 0) as recent_views
        FROM doc_stats ds
        LEFT JOIN comment_stats cs ON ds.id = cs.document_id
        LEFT JOIN recent_activity ra ON ds.id = ra.document_id
        """,
        doc_id
    )
    return dict(row) if row else {}
```

### Transactions
- **BEGIN/COMMIT**: Wrap multiple operations
- **ROLLBACK**: Automatic on error in context manager
- **SAVEPOINT**: Nested transactions

```python
async def transfer_document_ownership(
    pool: asyncpg.Pool,
    doc_id: int,
    from_user_id: int,
    to_user_id: int
) -> bool:
    """Transfer document ownership with transaction."""
    async with pool.acquire() as conn:
        async with conn.transaction():
            # Verify current owner
            current_owner = await conn.fetchval(
                "SELECT author_id FROM documents WHERE id = $1",
                doc_id
            )

            if current_owner != from_user_id:
                raise ValueError("Document not owned by from_user")

            # Update document
            await conn.execute(
                "UPDATE documents SET author_id = $1 WHERE id = $2",
                to_user_id,
                doc_id
            )

            # Log transfer
            await conn.execute(
                """
                INSERT INTO ownership_transfers (document_id, from_user_id, to_user_id)
                VALUES ($1, $2, $3)
                """,
                doc_id,
                from_user_id,
                to_user_id
            )

    return True
```

### Bulk Operations
- **executemany()**: Batch inserts/updates
- **COPY**: Very fast bulk loads
- **UNNEST**: Insert array values

```python
# Batch insert
async def bulk_insert_embeddings(
    conn: asyncpg.Connection,
    embeddings: List[Tuple[int, List[float]]]
) -> int:
    """Bulk update document embeddings."""
    # embeddings: [(doc_id, embedding), ...]
    await conn.executemany(
        """
        UPDATE documents
        SET embedding = $2::vector
        WHERE id = $1
        """,
        embeddings
    )
    return len(embeddings)

# Insert from array with UNNEST
async def bulk_insert_tags(
    conn: asyncpg.Connection,
    doc_id: int,
    tags: List[str]
) -> int:
    """Insert multiple tags for a document."""
    result = await conn.execute(
        """
        INSERT INTO document_tags (document_id, tag)
        SELECT $1, unnest($2::text[])
        ON CONFLICT DO NOTHING
        """,
        doc_id,
        tags
    )
    return len(tags)
```

### Query Optimization
- **EXPLAIN ANALYZE**: Analyze query performance
- **Indexes**: Ensure proper indexes exist
- **LIMIT**: Always limit large result sets
- **Avoid SELECT ***: Select only needed columns

```python
async def explain_query(
    conn: asyncpg.Connection,
    query: str,
    *args
) -> str:
    """Get query execution plan."""
    explain_query = f"EXPLAIN ANALYZE {query}"
    rows = await conn.fetch(explain_query, *args)
    return "\n".join(row[0] for row in rows)

# Example usage:
# plan = await explain_query(
#     conn,
#     "SELECT * FROM documents WHERE status = $1",
#     "published"
# )
# print(plan)
```

### Full-Text Search
- **to_tsvector()**: Convert text to searchable vector
- **to_tsquery()**: Parse search query
- **ts_rank()**: Rank search results

```python
async def fulltext_search(
    conn: asyncpg.Connection,
    search_query: str,
    limit: int = 10
) -> List[dict]:
    """Full-text search with ranking."""
    rows = await conn.fetch(
        """
        SELECT
            id,
            title,
            content,
            ts_rank(search_vector, query) as rank
        FROM documents,
             to_tsquery('english', $1) query
        WHERE search_vector @@ query
        ORDER BY rank DESC
        LIMIT $2
        """,
        search_query,
        limit
    )
    return [dict(row) for row in rows]
```

### Error Handling
- **Try/Except**: Catch specific exceptions
- **Unique Violations**: Handle duplicate keys
- **Foreign Key Violations**: Handle referential integrity

```python
import asyncpg.exceptions as pg_exceptions

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

    except pg_exceptions.UniqueViolationError as e:
        print(f"Duplicate document: {e}")
        return None

    except pg_exceptions.ForeignKeyViolationError as e:
        print(f"Invalid reference: {e}")
        return None

    except pg_exceptions.PostgresError as e:
        print(f"Database error: {e}")
        raise
```

### Type Conversion
- **asyncpg Automatic**: asyncpg auto-converts most types
- **Custom Types**: Register custom type codecs if needed
- **Arrays**: PostgreSQL arrays map to Python lists

```python
# Arrays are automatic
async def get_document_tags(
    conn: asyncpg.Connection,
    doc_id: int
) -> List[str]:
    """Get tags as array."""
    tags = await conn.fetchval(
        """
        SELECT ARRAY_AGG(tag)
        FROM document_tags
        WHERE document_id = $1
        """,
        doc_id
    )
    return tags or []
```
