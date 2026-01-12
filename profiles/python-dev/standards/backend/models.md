## Python Database Models Best Practices

### SQLAlchemy 2.0+ Models
- **Declarative Models**: Use SQLAlchemy's declarative base for all models
- **Type Hints**: Add type hints to all model attributes
- **Mapped Attributes**: Use `Mapped` and `mapped_column` (SQLAlchemy 2.0 syntax)
- **Relationships**: Define relationships explicitly with proper lazy loading

```python
from datetime import datetime
from sqlalchemy import String, Text, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    """Base class for all database models."""
    pass

class User(Base):
    __tablename__ = "users"

    # Primary key
    id: Mapped[int] = mapped_column(primary_key=True)

    # Required fields
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(String(255))

    # Optional fields (nullable)
    bio: Mapped[str | None] = mapped_column(Text, nullable=True)
    avatar_url: Mapped[str | None] = mapped_column(String(500), nullable=True)

    # Timestamps
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(
        default=datetime.utcnow,
        onupdate=datetime.utcnow
    )

    # Relationships
    posts: Mapped[list["Post"]] = relationship(back_populates="author", lazy="selectin")

    def __repr__(self) -> str:
        return f"<User(id={self.id}, username={self.username})>"
```

### Model Naming Conventions
- **Table Names**: Use lowercase with underscores: `users`, `blog_posts`
- **Class Names**: Use singular PascalCase: `User`, `BlogPost`
- **Column Names**: Use lowercase with underscores: `created_at`, `is_active`
- **Foreign Keys**: Name as `{related_table}_id`: `user_id`, `post_id`

### Field Types and Constraints
- **Appropriate Types**: Use correct SQLAlchemy types for data:
  - `String(n)`: Variable-length strings with max length
  - `Text`: Large text fields
  - `Integer`: Integer numbers
  - `Float`: Floating point numbers
  - `Boolean`: True/False values
  - `DateTime`: Date and time
  - `JSON`: JSON data (PostgreSQL JSONB)
- **Constraints**: Define constraints at database level:
  - `unique=True`: Unique constraint
  - `nullable=False`: Not null constraint
  - `index=True`: Create index
  - `default`: Default value

```python
from sqlalchemy import Boolean, Integer, Enum as SQLEnum
from enum import Enum

class UserRole(str, Enum):
    USER = "user"
    ADMIN = "admin"
    MODERATOR = "moderator"

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    role: Mapped[UserRole] = mapped_column(SQLEnum(UserRole), default=UserRole.USER)
    login_count: Mapped[int] = mapped_column(Integer, default=0)
```

### Relationships
- **Explicit Relationships**: Define all relationships explicitly
- **back_populates**: Use `back_populates` for bidirectional relationships
- **Lazy Loading**: Configure lazy loading strategy appropriately:
  - `lazy="selectin"`: Good for small collections (N+1 safe)
  - `lazy="joined"`: Use for required relationships
  - `lazy="dynamic"`: Use for large collections that need filtering
- **Cascade Options**: Set appropriate cascade behavior

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    posts: Mapped[list["Post"]] = relationship(
        back_populates="author",
        lazy="selectin",
        cascade="all, delete-orphan"
    )

class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text)

    # Foreign key
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))

    # Relationship
    author: Mapped["User"] = relationship(back_populates="posts", lazy="joined")
```

### Many-to-Many Relationships
- **Association Tables**: Use association tables for many-to-many relationships
- **Association Objects**: Use association objects when extra data is needed
- **Naming**: Name association tables as `{table1}_{table2}`: `users_groups`

```python
from sqlalchemy import Table, Column, ForeignKey

# Simple many-to-many without extra data
user_group_association = Table(
    "user_groups",
    Base.metadata,
    Column("user_id", ForeignKey("users.id"), primary_key=True),
    Column("group_id", ForeignKey("groups.id"), primary_key=True)
)

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    groups: Mapped[list["Group"]] = relationship(
        secondary=user_group_association,
        back_populates="members"
    )

class Group(Base):
    __tablename__ = "groups"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    members: Mapped[list["User"]] = relationship(
        secondary=user_group_association,
        back_populates="groups"
    )

# Many-to-many with extra data (use association object)
class UserGroupMembership(Base):
    __tablename__ = "user_group_memberships"

    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), primary_key=True)
    group_id: Mapped[int] = mapped_column(ForeignKey("groups.id"), primary_key=True)
    joined_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    role: Mapped[str] = mapped_column(String(50), default="member")

    user: Mapped["User"] = relationship(back_populates="group_memberships")
    group: Mapped["Group"] = relationship(back_populates="user_memberships")
```

### Timestamps and Audit Fields
- **Created/Updated**: Add created_at and updated_at to all models
- **UTC Timestamps**: Always use UTC for timestamps
- **Automatic Updates**: Use `onupdate` for updated_at
- **Soft Deletes**: Implement soft deletes with deleted_at field when needed

```python
from datetime import datetime, timezone

class TimestampMixin:
    """Mixin for timestamp fields."""
    created_at: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(timezone.utc)
    )
    updated_at: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc)
    )

class SoftDeleteMixin:
    """Mixin for soft delete functionality."""
    deleted_at: Mapped[datetime | None] = mapped_column(nullable=True)

    @property
    def is_deleted(self) -> bool:
        return self.deleted_at is not None

class User(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)
```

### Indexes
- **Index Important Columns**: Add indexes to frequently queried columns
- **Foreign Keys**: Foreign keys automatically get indexes
- **Unique Indexes**: Use unique indexes for uniqueness constraints
- **Composite Indexes**: Create composite indexes for multi-column queries

```python
from sqlalchemy import Index

class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    status: Mapped[str] = mapped_column(String(20), index=True)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    # Composite index for common query pattern
    __table_args__ = (
        Index("idx_author_status", "author_id", "status"),
        Index("idx_status_created", "status", "created_at"),
    )
```

### Model Methods
- **Business Logic**: Add business logic methods to models when appropriate
- **Property Decorators**: Use @property for computed fields
- **No Database Operations**: Don't put database queries in model methods
- **Validation**: Add validation logic for data integrity

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    first_name: Mapped[str] = mapped_column(String(50))
    last_name: Mapped[str] = mapped_column(String(50))
    email: Mapped[str] = mapped_column(String(255), unique=True)
    hashed_password: Mapped[str] = mapped_column(String(255))

    @property
    def full_name(self) -> str:
        """Get user's full name."""
        return f"{self.first_name} {self.last_name}"

    def verify_password(self, password: str) -> bool:
        """Verify password against hash."""
        from passlib.hash import bcrypt
        return bcrypt.verify(password, self.hashed_password)

    def set_password(self, password: str) -> None:
        """Hash and set user password."""
        from passlib.hash import bcrypt
        self.hashed_password = bcrypt.hash(password)
```

### JSON and JSONB Fields
- **Use JSONB**: Use PostgreSQL JSONB for flexible data (faster and indexed)
- **Pydantic Integration**: Use Pydantic models for JSON field validation
- **Type Hints**: Use proper type hints for JSON fields

```python
from sqlalchemy.dialects.postgresql import JSONB
from typing import Any

class Product(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    metadata: Mapped[dict[str, Any]] = mapped_column(JSONB, default=dict)
    tags: Mapped[list[str]] = mapped_column(JSONB, default=list)
```

### Enums
- **Python Enums**: Use Python Enum with SQLAlchemy Enum type
- **String Enums**: Prefer string enums for better database readability
- **Explicit Values**: Explicitly set enum values for database stability

```python
from enum import Enum
from sqlalchemy import Enum as SQLEnum

class OrderStatus(str, Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[OrderStatus] = mapped_column(
        SQLEnum(OrderStatus),
        default=OrderStatus.PENDING,
        nullable=False
    )
```

### Model Organization
- **One File per Model**: Keep large models in separate files
- **Related Models Together**: Group related models in same file for small projects
- **Base Classes**: Use mixins for shared functionality
- **Models Module**: Organize models in `models/` directory

```
src/project_name/models/
├── __init__.py      # Import all models
├── base.py          # Base and mixins
├── user.py          # User model
├── post.py          # Post model
└── product.py       # Product model
```

### Async SQLAlchemy (2.0+)
- **AsyncSession**: Use AsyncSession for async operations
- **Async Queries**: All queries must use await
- **Async Engine**: Configure async engine properly

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

# Create async engine
engine = create_async_engine(
    "postgresql+asyncpg://user:password@localhost/dbname",
    echo=True,
    pool_pre_ping=True
)

# Create async session factory
async_session = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

# Usage in dependency
async def get_db_session() -> AsyncSession:
    async with async_session() as session:
        yield session
```

### Django ORM (Alternative)
If using Django instead of SQLAlchemy:

```python
from django.db import models
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    """Custom user model."""
    bio = models.TextField(blank=True, null=True)
    avatar_url = models.URLField(max_length=500, blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "users"
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["email"]),
            models.Index(fields=["username"]),
        ]

    def __str__(self) -> str:
        return self.username
```
