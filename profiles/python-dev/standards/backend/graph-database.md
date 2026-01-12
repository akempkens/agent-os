## Graph Database Best Practices (Neo4j)

### Graph Database Philosophy
- **Relationships First**: Model data with relationships as first-class citizens
- **Cypher Queries**: Use Cypher query language for graph operations
- **Direct Driver**: Use official Neo4j Python driver (no ORM)
- **Property Graphs**: Leverage property graphs for rich data modeling

### Neo4j Connection Setup
- **Async Driver**: Use async Neo4j driver for async operations
- **Connection Pool**: Use connection pooling for performance
- **Environment Config**: Store credentials in environment variables

```python
from neo4j import AsyncGraphDatabase
import os

class Neo4jConnection:
    """Async Neo4j connection manager."""

    def __init__(self):
        self.uri = os.getenv("NEO4J_URI", "bolt://localhost:7687")
        self.user = os.getenv("NEO4J_USER", "neo4j")
        self.password = os.getenv("NEO4J_PASSWORD")
        self.driver = None

    async def connect(self):
        """Initialize connection."""
        self.driver = AsyncGraphDatabase.driver(
            self.uri,
            auth=(self.user, self.password),
            max_connection_lifetime=3600,
            max_connection_pool_size=50,
            connection_acquisition_timeout=60
        )

    async def close(self):
        """Close connection."""
        if self.driver:
            await self.driver.close()

    async def execute_query(self, query: str, parameters: dict = None):
        """Execute Cypher query."""
        async with self.driver.session() as session:
            result = await session.run(query, parameters or {})
            return [record async for record in result]

# Usage
db = Neo4jConnection()
await db.connect()
```

### Node Operations (CREATE, READ, UPDATE, DELETE)
- **Labeled Nodes**: Always use node labels
- **Properties**: Store metadata as node properties
- **MERGE**: Use MERGE to avoid duplicates

```python
# CREATE node
async def create_document_node(
    driver,
    doc_id: int,
    title: str,
    content: str,
    category: str
) -> dict:
    """Create document node."""
    query = """
    CREATE (d:Document {
        id: $doc_id,
        title: $title,
        content: $content,
        category: $category,
        created_at: datetime()
    })
    RETURN d
    """
    async with driver.session() as session:
        result = await session.run(
            query,
            doc_id=doc_id,
            title=title,
            content=content,
            category=category
        )
        record = await result.single()
        return dict(record["d"])

# READ node
async def get_document_by_id(driver, doc_id: int) -> dict:
    """Get document node by ID."""
    query = """
    MATCH (d:Document {id: $doc_id})
    RETURN d
    """
    async with driver.session() as session:
        result = await session.run(query, doc_id=doc_id)
        record = await result.single()
        return dict(record["d"]) if record else None

# UPDATE node
async def update_document_properties(
    driver,
    doc_id: int,
    properties: dict
) -> dict:
    """Update document properties."""
    query = """
    MATCH (d:Document {id: $doc_id})
    SET d += $properties, d.updated_at = datetime()
    RETURN d
    """
    async with driver.session() as session:
        result = await session.run(query, doc_id=doc_id, properties=properties)
        record = await result.single()
        return dict(record["d"])

# DELETE node
async def delete_document(driver, doc_id: int):
    """Delete document and all its relationships."""
    query = """
    MATCH (d:Document {id: $doc_id})
    DETACH DELETE d
    """
    async with driver.session() as session:
        await session.run(query, doc_id=doc_id)

# MERGE (create or update)
async def merge_document(
    driver,
    doc_id: int,
    title: str,
    content: str
) -> dict:
    """Create document if not exists, otherwise update."""
    query = """
    MERGE (d:Document {id: $doc_id})
    ON CREATE SET d.title = $title, d.content = $content, d.created_at = datetime()
    ON MATCH SET d.title = $title, d.content = $content, d.updated_at = datetime()
    RETURN d
    """
    async with driver.session() as session:
        result = await session.run(query, doc_id=doc_id, title=title, content=content)
        record = await result.single()
        return dict(record["d"])
```

### Relationship Operations
- **Typed Relationships**: Use descriptive relationship types
- **Relationship Properties**: Store metadata on relationships
- **Bidirectional**: Consider query patterns when choosing direction

```python
# CREATE relationship
async def create_citation_relationship(
    driver,
    from_doc_id: int,
    to_doc_id: int,
    context: str = None
):
    """Create citation relationship between documents."""
    query = """
    MATCH (from:Document {id: $from_id})
    MATCH (to:Document {id: $to_id})
    CREATE (from)-[r:CITES {
        created_at: datetime(),
        context: $context
    }]->(to)
    RETURN r
    """
    async with driver.session() as session:
        result = await session.run(
            query,
            from_id=from_doc_id,
            to_id=to_doc_id,
            context=context
        )
        return await result.single()

# QUERY relationships
async def get_cited_documents(driver, doc_id: int) -> list[dict]:
    """Get all documents cited by this document."""
    query = """
    MATCH (d:Document {id: $doc_id})-[:CITES]->(cited:Document)
    RETURN cited
    """
    async with driver.session() as session:
        result = await session.run(query, doc_id=doc_id)
        return [dict(record["cited"]) async for record in result]

async def get_documents_citing(driver, doc_id: int) -> list[dict]:
    """Get all documents that cite this document."""
    query = """
    MATCH (citing:Document)-[:CITES]->(d:Document {id: $doc_id})
    RETURN citing
    """
    async with driver.session() as session:
        result = await session.run(query, doc_id=doc_id)
        return [dict(record["citing"]) async for record in result]
```

### Pattern Matching
- **Variable Length Paths**: Use variable-length patterns
- **Optional Matches**: Use OPTIONAL MATCH for nullable patterns
- **WHERE Clauses**: Filter with WHERE for complex conditions

```python
# Find paths between nodes
async def find_citation_path(
    driver,
    from_doc_id: int,
    to_doc_id: int,
    max_depth: int = 5
) -> list[dict]:
    """Find citation path between two documents."""
    query = """
    MATCH path = (from:Document {id: $from_id})
                 -[:CITES*1..${max_depth}]->
                 (to:Document {id: $to_id})
    RETURN path
    ORDER BY length(path)
    LIMIT 10
    """
    async with driver.session() as session:
        result = await session.run(
            query,
            from_id=from_doc_id,
            to_id=to_doc_id,
            max_depth=max_depth
        )
        return [record async for record in result]

# Find related documents (collaborative filtering)
async def find_related_documents(
    driver,
    doc_id: int,
    min_shared_citations: int = 3
) -> list[dict]:
    """Find documents with shared citations."""
    query = """
    MATCH (d:Document {id: $doc_id})-[:CITES]->(cited:Document)
    MATCH (related:Document)-[:CITES]->(cited)
    WHERE d <> related
    WITH related, COUNT(cited) as shared_citations
    WHERE shared_citations >= $min_shared
    RETURN related, shared_citations
    ORDER BY shared_citations DESC
    LIMIT 10
    """
    async with driver.session() as session:
        result = await session.run(
            query,
            doc_id=doc_id,
            min_shared=min_shared_citations
        )
        return [
            {"document": dict(record["related"]), "shared": record["shared_citations"]}
            async for record in result
        ]
```

### Aggregations and Analytics
- **COUNT, SUM, AVG**: Use aggregation functions
- **COLLECT**: Aggregate results into lists
- **Graph Algorithms**: Use Neo4j Graph Data Science library

```python
# Citation count
async def get_citation_stats(driver, doc_id: int) -> dict:
    """Get citation statistics for a document."""
    query = """
    MATCH (d:Document {id: $doc_id})
    OPTIONAL MATCH (d)-[:CITES]->(cited)
    OPTIONAL MATCH (citing)-[:CITES]->(d)
    RETURN
        COUNT(DISTINCT cited) as citations_made,
        COUNT(DISTINCT citing) as citations_received
    """
    async with driver.session() as session:
        result = await session.run(query, doc_id=doc_id)
        record = await result.single()
        return {
            "citations_made": record["citations_made"],
            "citations_received": record["citations_received"]
        }

# Top cited documents
async def get_most_cited_documents(driver, limit: int = 10) -> list[dict]:
    """Get most cited documents."""
    query = """
    MATCH (d:Document)
    OPTIONAL MATCH (citing)-[:CITES]->(d)
    WITH d, COUNT(citing) as citation_count
    WHERE citation_count > 0
    RETURN d, citation_count
    ORDER BY citation_count DESC
    LIMIT $limit
    """
    async with driver.session() as session:
        result = await session.run(query, limit=limit)
        return [
            {"document": dict(record["d"]), "citations": record["citation_count"]}
            async for record in result
        ]
```

### Vector Similarity with Graph
- **Combine Vector + Graph**: Use vector similarity with graph relationships
- **Hybrid Search**: Combine semantic search with graph traversal

```python
async def hybrid_graph_vector_search(
    driver,
    query_embedding: list[float],
    top_k: int = 10
) -> list[dict]:
    """Hybrid search: vector similarity + citation network."""
    query = """
    // This assumes vector index exists on Document.embedding
    CALL db.index.vector.queryNodes(
        'document_embeddings',
        $top_k,
        $query_embedding
    ) YIELD node as similar_doc, score

    // Expand with cited documents
    OPTIONAL MATCH (similar_doc)-[:CITES]->(cited:Document)

    RETURN
        similar_doc,
        score as similarity_score,
        COLLECT(DISTINCT cited.id) as cited_docs
    ORDER BY similarity_score DESC
    """
    async with driver.session() as session:
        result = await session.run(
            query,
            top_k=top_k,
            query_embedding=query_embedding
        )
        return [
            {
                "document": dict(record["similar_doc"]),
                "similarity": record["similarity_score"],
                "cited_docs": record["cited_docs"]
            }
            async for record in result
        ]
```

### Indexes and Constraints
- **Unique Constraints**: Ensure uniqueness where needed
- **Property Indexes**: Index frequently queried properties
- **Vector Indexes**: Create vector indexes for similarity search

```python
async def create_indexes_and_constraints(driver):
    """Create necessary indexes and constraints."""
    async with driver.session() as session:
        # Unique constraint on document ID
        await session.run("""
            CREATE CONSTRAINT document_id_unique IF NOT EXISTS
            FOR (d:Document)
            REQUIRE d.id IS UNIQUE
        """)

        # Index on document category
        await session.run("""
            CREATE INDEX document_category IF NOT EXISTS
            FOR (d:Document)
            ON (d.category)
        """)

        # Full-text index on title and content
        await session.run("""
            CREATE FULLTEXT INDEX document_fulltext IF NOT EXISTS
            FOR (d:Document)
            ON EACH [d.title, d.content]
        """)

        # Vector index for embeddings (Neo4j 5.11+)
        await session.run("""
            CREATE VECTOR INDEX document_embeddings IF NOT EXISTS
            FOR (d:Document)
            ON d.embedding
            OPTIONS {indexConfig: {
                `vector.dimensions`: 1536,
                `vector.similarity_function`: 'cosine'
            }}
        """)
```

### Batch Operations
- **UNWIND**: Use UNWIND for batch inserts
- **Transactions**: Use transactions for atomic batch operations

```python
async def batch_create_documents(driver, documents: list[dict]):
    """Batch create multiple documents."""
    query = """
    UNWIND $documents as doc
    CREATE (d:Document {
        id: doc.id,
        title: doc.title,
        content: doc.content,
        category: doc.category,
        created_at: datetime()
    })
    """
    async with driver.session() as session:
        await session.run(query, documents=documents)

async def batch_create_relationships(driver, relationships: list[dict]):
    """Batch create relationships."""
    query = """
    UNWIND $relationships as rel
    MATCH (from:Document {id: rel.from_id})
    MATCH (to:Document {id: rel.to_id})
    MERGE (from)-[r:CITES]->(to)
    SET r.created_at = datetime()
    """
    async with driver.session() as session:
        await session.run(query, relationships=relationships)
```

### Transactions
- **Explicit Transactions**: Use for multi-statement operations
- **Read/Write Transactions**: Separate read and write transactions

```python
async def create_document_with_citations_transaction(
    driver,
    document: dict,
    citations: list[int]
):
    """Create document and citation relationships in transaction."""
    async with driver.session() as session:
        async with session.begin_transaction() as tx:
            # Create document
            await tx.run(
                """
                CREATE (d:Document {
                    id: $id,
                    title: $title,
                    content: $content
                })
                """,
                id=document["id"],
                title=document["title"],
                content=document["content"]
            )

            # Create citations
            if citations:
                await tx.run(
                    """
                    MATCH (from:Document {id: $doc_id})
                    UNWIND $cited_ids as cited_id
                    MATCH (to:Document {id: cited_id})
                    CREATE (from)-[:CITES]->(to)
                    """,
                    doc_id=document["id"],
                    cited_ids=citations
                )

            await tx.commit()
```

### Graph Algorithms (Using GDS Library)
- **PageRank**: Identify influential documents
- **Community Detection**: Find document clusters
- **Centrality**: Find most central/important nodes

```python
async def compute_pagerank(driver) -> list[dict]:
    """Compute PageRank for documents."""
    query = """
    CALL gds.pageRank.stream('document_graph')
    YIELD nodeId, score
    RETURN gds.util.asNode(nodeId) as document, score
    ORDER BY score DESC
    LIMIT 20
    """
    async with driver.session() as session:
        result = await session.run(query)
        return [
            {"document": dict(record["document"]), "pagerank": record["score"]}
            async for record in result
        ]

async def detect_communities(driver) -> list[dict]:
    """Detect communities using Louvain algorithm."""
    query = """
    CALL gds.louvain.stream('document_graph')
    YIELD nodeId, communityId
    RETURN
        communityId,
        COLLECT(gds.util.asNode(nodeId).title) as documents,
        COUNT(*) as size
    ORDER BY size DESC
    """
    async with driver.session() as session:
        result = await session.run(query)
        return [dict(record) async for record in result]
```

### Performance Optimization
- **Indexed Lookups**: Use indexed properties in MATCH
- **Query Planning**: Use EXPLAIN/PROFILE to optimize queries
- **Limit Results**: Always limit large result sets

```python
# Good: Uses index
async def get_documents_by_category_optimized(driver, category: str):
    """Get documents by category (uses index)."""
    query = """
    MATCH (d:Document {category: $category})
    RETURN d
    LIMIT 100
    """
    async with driver.session() as session:
        result = await session.run(query, category=category)
        return [dict(record["d"]) async for record in result]

# Profile query performance
async def profile_query(driver, query: str, parameters: dict = None):
    """Profile query to analyze performance."""
    profile_query = f"PROFILE {query}"
    async with driver.session() as session:
        result = await session.run(profile_query, parameters or {})
        summary = await result.consume()
        return summary.profile
```
