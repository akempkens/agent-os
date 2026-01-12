## Python Database Query Best Practices

### SQLAlchemy 2.0+ Query Patterns
- **Use select()**: Use SQLAlchemy 2.0 select() syntax for all queries
- **Async Queries**: Always await async queries
- **Type Safety**: Use type hints for query results
- **Session Management**: Use async context managers for sessions

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

async def get_user_by_id(session: AsyncSession, user_id: int) -> User | None:
    """Get user by ID."""
    result = await session.execute(
        select(User).where(User.id == user_id)
    )
    return result.scalar_one_or_none()

async def get_users_by_status(
    session: AsyncSession,
    status: str,
    limit: int = 100
) -> list[User]:
    """Get users by status with limit."""
    result = await session.execute(
        select(User)
        .where(User.status == status)
        .limit(limit)
    )
    return list(result.scalars().all())
```

### N+1 Query Prevention
- **Eager Loading**: Use selectinload() or joinedload() to prevent N+1 queries
- **selectinload()**: Use for collections (separate query)
- **joinedload()**: Use for single relationships (JOIN)
- **Profile Queries**: Monitor query counts in development

```python
from sqlalchemy.orm import selectinload, joinedload

# Bad: N+1 query problem
async def get_users_with_posts_bad(session: AsyncSession) -> list[User]:
    result = await session.execute(select(User))
    users = result.scalars().all()
    for user in users:
        # This triggers a query for each user (N+1 problem!)
        posts = user.posts
    return users

# Good: Eager loading with selectinload
async def get_users_with_posts(session: AsyncSession) -> list[User]:
    result = await session.execute(
        select(User).options(selectinload(User.posts))
    )
    return list(result.scalars().all())

# Good: Use joinedload for single relationships
async def get_posts_with_authors(session: AsyncSession) -> list[Post]:
    result = await session.execute(
        select(Post).options(joinedload(Post.author))
    )
    return list(result.unique().scalars().all())
```

### Filtering and Conditions
- **where() Clauses**: Use clear, readable where conditions
- **Multiple Conditions**: Chain conditions or use and_/or_
- **IN Queries**: Use in_() for list conditions
- **LIKE Queries**: Use like() or ilike() for pattern matching

```python
from sqlalchemy import and_, or_

# Simple filter
async def get_active_users(session: AsyncSession) -> list[User]:
    result = await session.execute(
        select(User).where(User.is_active == True)
    )
    return list(result.scalars().all())

# Multiple conditions
async def get_filtered_users(
    session: AsyncSession,
    min_age: int,
    roles: list[str]
) -> list[User]:
    result = await session.execute(
        select(User).where(
            and_(
                User.age >= min_age,
                User.role.in_(roles),
                User.is_active == True
            )
        )
    )
    return list(result.scalars().all())

# Pattern matching
async def search_users(session: AsyncSession, query: str) -> list[User]:
    search_pattern = f"%{query}%"
    result = await session.execute(
        select(User).where(
            or_(
                User.username.ilike(search_pattern),
                User.email.ilike(search_pattern)
            )
        )
    )
    return list(result.scalars().all())
```

### Ordering and Limiting
- **order_by()**: Use order_by() for sorting
- **Multiple Order**: Chain multiple order_by() for complex sorting
- **Descending**: Use desc() for descending order
- **limit() and offset()**: Use for pagination

```python
from sqlalchemy import desc

async def get_recent_posts(
    session: AsyncSession,
    limit: int = 10
) -> list[Post]:
    """Get most recent posts."""
    result = await session.execute(
        select(Post)
        .order_by(desc(Post.created_at))
        .limit(limit)
    )
    return list(result.scalars().all())

async def get_paginated_users(
    session: AsyncSession,
    page: int = 1,
    page_size: int = 20
) -> list[User]:
    """Get paginated users sorted by username."""
    offset = (page - 1) * page_size
    result = await session.execute(
        select(User)
        .order_by(User.username)
        .offset(offset)
        .limit(page_size)
    )
    return list(result.scalars().all())
```

### Aggregations and Counts
- **func.count()**: Use for counting records
- **group_by()**: Use for grouping
- **Aggregation Functions**: Use func.sum(), func.avg(), func.max(), func.min()

```python
from sqlalchemy import func

async def count_users(session: AsyncSession) -> int:
    """Count total users."""
    result = await session.execute(
        select(func.count()).select_from(User)
    )
    return result.scalar()

async def count_users_by_role(
    session: AsyncSession
) -> dict[str, int]:
    """Count users grouped by role."""
    result = await session.execute(
        select(User.role, func.count(User.id))
        .group_by(User.role)
    )
    return dict(result.all())

async def get_user_statistics(session: AsyncSession) -> dict:
    """Get user statistics."""
    result = await session.execute(
        select(
            func.count(User.id),
            func.avg(User.age),
            func.max(User.created_at)
        )
    )
    total, avg_age, latest_signup = result.one()
    return {
        "total_users": total,
        "average_age": float(avg_age) if avg_age else 0,
        "latest_signup": latest_signup
    }
```

### Joins
- **Explicit Joins**: Use join() for explicit joins
- **Relationship Joins**: SQLAlchemy can auto-join via relationships
- **Left Joins**: Use outerjoin() for left outer joins
- **Join Conditions**: Specify join conditions when needed

```python
from sqlalchemy import join

async def get_users_with_post_count(session: AsyncSession) -> list[tuple]:
    """Get users with their post count."""
    result = await session.execute(
        select(User.username, func.count(Post.id).label("post_count"))
        .join(Post, User.id == Post.author_id, isouter=True)
        .group_by(User.id, User.username)
        .order_by(desc("post_count"))
    )
    return list(result.all())
```

### Transactions
- **Automatic Commit**: Use session.commit() to save changes
- **Rollback**: Use session.rollback() on errors
- **Context Manager**: Use async with for automatic transaction management
- **Savepoints**: Use nested transactions for complex operations

```python
async def create_user_with_profile(
    session: AsyncSession,
    user_data: UserCreate,
    profile_data: ProfileCreate
) -> User:
    """Create user and profile in a transaction."""
    try:
        # Create user
        user = User(**user_data.model_dump())
        session.add(user)
        await session.flush()  # Get user.id without committing

        # Create profile linked to user
        profile = Profile(**profile_data.model_dump(), user_id=user.id)
        session.add(profile)

        # Commit transaction
        await session.commit()
        await session.refresh(user)
        return user

    except Exception as e:
        # Rollback on error
        await session.rollback()
        raise
```

### Bulk Operations
- **Bulk Insert**: Use session.add_all() for multiple records
- **Bulk Update**: Use update() for bulk updates
- **Bulk Delete**: Use delete() for bulk deletes
- **Performance**: Bulk operations are much faster for large datasets

```python
async def create_users_bulk(
    session: AsyncSession,
    users_data: list[UserCreate]
) -> list[User]:
    """Create multiple users in bulk."""
    users = [User(**data.model_dump()) for data in users_data]
    session.add_all(users)
    await session.commit()
    return users

async def deactivate_inactive_users(
    session: AsyncSession,
    days_inactive: int
) -> int:
    """Bulk update: deactivate users inactive for X days."""
    from datetime import datetime, timedelta

    cutoff_date = datetime.utcnow() - timedelta(days=days_inactive)

    result = await session.execute(
        update(User)
        .where(User.last_login < cutoff_date)
        .values(is_active=False)
    )
    await session.commit()
    return result.rowcount
```

### Raw SQL (When Necessary)
- **Prefer ORM**: Use ORM queries whenever possible
- **Complex Queries**: Use raw SQL only for very complex queries
- **Parameterization**: Always use parameterized queries to prevent SQL injection
- **text()**: Use SQLAlchemy's text() for raw SQL

```python
from sqlalchemy import text

async def execute_complex_query(
    session: AsyncSession,
    min_posts: int
) -> list[dict]:
    """Execute complex raw SQL query."""
    query = text("""
        SELECT u.id, u.username, COUNT(p.id) as post_count
        FROM users u
        LEFT JOIN posts p ON u.id = p.author_id
        GROUP BY u.id, u.username
        HAVING COUNT(p.id) >= :min_posts
        ORDER BY post_count DESC
    """)

    result = await session.execute(query, {"min_posts": min_posts})
    return [dict(row._mapping) for row in result]
```

### Query Performance
- **EXPLAIN ANALYZE**: Use EXPLAIN to analyze query performance
- **Indexes**: Ensure frequently queried columns have indexes
- **Select Only Needed**: Use load_only() to select specific columns
- **Pagination**: Always paginate large result sets

```python
from sqlalchemy.orm import load_only

async def get_user_emails_only(
    session: AsyncSession,
    limit: int = 1000
) -> list[str]:
    """Get only email addresses (not full user objects)."""
    result = await session.execute(
        select(User)
        .options(load_only(User.email))
        .limit(limit)
    )
    return [user.email for user in result.scalars().all()]
```

### Query Organization (Repository Pattern)
- **Repository Classes**: Organize queries in repository classes
- **Separation of Concerns**: Keep data access logic separate from business logic
- **Reusable Queries**: Create reusable query methods

```python
class UserRepository:
    """Repository for user data access."""

    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, user_id: int) -> User | None:
        """Get user by ID."""
        result = await self.session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def get_by_email(self, email: str) -> User | None:
        """Get user by email."""
        result = await self.session.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def create(self, user_data: UserCreate) -> User:
        """Create new user."""
        user = User(**user_data.model_dump())
        self.session.add(user)
        await self.session.commit()
        await self.session.refresh(user)
        return user

    async def list_active(
        self,
        offset: int = 0,
        limit: int = 100
    ) -> list[User]:
        """List active users with pagination."""
        result = await self.session.execute(
            select(User)
            .where(User.is_active == True)
            .order_by(User.created_at.desc())
            .offset(offset)
            .limit(limit)
        )
        return list(result.scalars().all())
```

### Database Connection Pooling
- **Pool Size**: Configure appropriate pool size for production
- **Pool Pre-Ping**: Enable pool pre-ping to handle stale connections
- **Pool Recycling**: Configure connection recycling for long-running connections

```python
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_size=20,           # Max number of connections in pool
    max_overflow=10,        # Additional connections beyond pool_size
    pool_pre_ping=True,     # Test connections before using
    pool_recycle=3600,      # Recycle connections after 1 hour
    echo=False              # Disable SQL logging in production
)
```
