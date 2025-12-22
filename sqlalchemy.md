# SQLAlchemy: Ultimate Interview Crash Course (1–3 Days)

---

## 1. Core Concepts You Must Know Cold

### 1.1 What is SQLAlchemy?

SQLAlchemy is Python's most popular SQL toolkit and Object-Relational Mapper (ORM). It provides two distinct APIs:

- **Core**: Low-level SQL abstraction layer (SQL Expression Language)
- **ORM**: High-level object-relational mapping

```python
# SQLAlchemy 2.0 style (current standard - KNOW THIS)
from sqlalchemy import create_engine, text
from sqlalchemy.orm import Session, DeclarativeBase, Mapped, mapped_column

# Engine: The starting point - manages connection pool
engine = create_engine("postgresql://user:pass@localhost/db", echo=True)

# Execute raw SQL via Core
with engine.connect() as conn:
    result = conn.execute(text("SELECT * FROM users WHERE id = :id"), {"id": 1})
    print(result.fetchone())
```

**Interview Point**: Always clarify SQLAlchemy 1.x vs 2.0 syntax. 2.0 uses `Mapped[]`, `mapped_column()`, and `Session.execute()` instead of `Query`.

---

### 1.2 Engine & Connection Pool

The `Engine` is the core interface to the database. It maintains a **connection pool** for efficiency.

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

# Production-grade engine configuration
engine = create_engine(
    "postgresql+psycopg2://user:pass@localhost:5432/mydb",
    pool_size=5,           # Number of connections to keep open
    max_overflow=10,       # Extra connections allowed beyond pool_size
    pool_timeout=30,       # Seconds to wait for available connection
    pool_recycle=1800,     # Recycle connections after 30 min (avoid stale)
    pool_pre_ping=True,    # Test connection health before using
    echo=False,            # Set True for SQL logging (debug only)
)

# Connection URL format:
# dialect+driver://username:password@host:port/database
# Examples:
# - SQLite:      sqlite:///./local.db (file) or sqlite:///:memory: (in-memory)
# - PostgreSQL:  postgresql+psycopg2://user:pass@host/db
# - MySQL:       mysql+pymysql://user:pass@host/db
```

**Interview Point**: Explain `pool_pre_ping` — it sends a lightweight query (like `SELECT 1`) before using a connection to detect disconnections. Critical for long-running apps.

---

### 1.3 Declarative ORM Models (SQLAlchemy 2.0)

Models map Python classes to database tables.

```python
from datetime import datetime
from typing import List, Optional
from sqlalchemy import String, ForeignKey, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

# Base class for all models
class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    
    # Primary key with auto-increment
    id: Mapped[int] = mapped_column(primary_key=True)
    
    # Required string with max length
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    
    # Optional field (nullable)
    name: Mapped[Optional[str]] = mapped_column(String(100))
    
    # Server-side default (database generates value)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    
    # Python-side default
    is_active: Mapped[bool] = mapped_column(default=True)
    
    # One-to-Many relationship
    posts: Mapped[List["Post"]] = relationship(back_populates="author", lazy="selectin")
    
    def __repr__(self) -> str:
        return f"User(id={self.id}, email={self.email})"


class Post(Base):
    __tablename__ = "posts"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str]
    
    # Foreign key constraint
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id", ondelete="CASCADE"))
    
    # Many-to-One relationship (reverse of User.posts)
    author: Mapped["User"] = relationship(back_populates="posts")


# Create all tables
Base.metadata.create_all(engine)
```

**Key Differences: 1.x vs 2.0**

| Feature | SQLAlchemy 1.x | SQLAlchemy 2.0 |
|---------|----------------|----------------|
| Column definition | `Column(Integer, primary_key=True)` | `mapped_column(primary_key=True)` |
| Type hints | Optional | Required via `Mapped[type]` |
| Query API | `session.query(User).filter()` | `session.execute(select(User).where())` |
| Result access | `query.all()` returns objects | `result.scalars().all()` returns objects |

---

### 1.4 Session & Unit of Work Pattern

The `Session` is the ORM's "holding zone" for objects. It implements the **Unit of Work** pattern: track changes, batch them, commit atomically.

```python
from sqlalchemy.orm import Session, sessionmaker

# Create session factory (bind to engine)
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)

# ==================== CRUD Operations ====================

# CREATE
with Session(engine) as session:
    new_user = User(email="alice@example.com", name="Alice")
    session.add(new_user)           # Stage for insert
    session.add_all([user1, user2]) # Bulk add
    session.commit()                # Persist to DB
    print(new_user.id)              # ID populated after commit

# READ (SQLAlchemy 2.0 style)
from sqlalchemy import select

with Session(engine) as session:
    # Get by primary key (most efficient)
    user = session.get(User, 1)
    
    # Select with filter
    stmt = select(User).where(User.email == "alice@example.com")
    user = session.execute(stmt).scalar_one_or_none()  # Returns User or None
    
    # Select multiple
    stmt = select(User).where(User.is_active == True).order_by(User.created_at.desc())
    users = session.execute(stmt).scalars().all()  # Returns List[User]

# UPDATE
with Session(engine) as session:
    user = session.get(User, 1)
    user.name = "Alice Updated"  # Just modify the object
    session.commit()             # Changes tracked and persisted
    
    # Bulk update (efficient - single SQL statement)
    from sqlalchemy import update
    stmt = update(User).where(User.is_active == False).values(is_active=True)
    session.execute(stmt)
    session.commit()

# DELETE
with Session(engine) as session:
    user = session.get(User, 1)
    session.delete(user)
    session.commit()
    
    # Bulk delete
    from sqlalchemy import delete
    stmt = delete(User).where(User.email.like("%@spam.com"))
    session.execute(stmt)
    session.commit()
```

**Session States** (frequently asked):

```
Transient  →  Pending  →  Persistent  →  Detached
   ↓            ↓            ↓              ↓
 new()       add()      commit()      close()/expunge()
 
- Transient: Python object, not associated with session
- Pending: Added to session, not yet flushed to DB
- Persistent: In session AND in database (has identity)
- Detached: Was persistent, session closed/expired
```

---

### 1.5 Relationships & Loading Strategies

```python
from sqlalchemy.orm import relationship, joinedload, selectinload, lazyload

# ==================== Relationship Types ====================

# One-to-Many (User has many Posts)
class User(Base):
    posts: Mapped[List["Post"]] = relationship(back_populates="author")

# Many-to-One (Post belongs to User)
class Post(Base):
    author: Mapped["User"] = relationship(back_populates="posts")

# Many-to-Many (with association table)
post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", ForeignKey("posts.id"), primary_key=True),
    Column("tag_id", ForeignKey("tags.id"), primary_key=True),
)

class Post(Base):
    tags: Mapped[List["Tag"]] = relationship(secondary=post_tags, back_populates="posts")

class Tag(Base):
    posts: Mapped[List["Post"]] = relationship(secondary=post_tags, back_populates="tags")

# One-to-One (use uselist=False)
class User(Base):
    profile: Mapped["Profile"] = relationship(back_populates="user", uselist=False)


# ==================== Loading Strategies (CRITICAL) ====================

# 1. LAZY (default) - N+1 problem!
# Loads related objects only when accessed
users = session.execute(select(User)).scalars().all()
for user in users:
    print(user.posts)  # Each access = separate query = N+1 problem!

# 2. JOINED (joinedload) - Single query with JOIN
# Best for: One-to-One, Many-to-One
stmt = select(User).options(joinedload(User.profile))
users = session.execute(stmt).unique().scalars().all()

# 3. SELECTIN (selectinload) - Two queries: main + IN clause
# Best for: One-to-Many, Many-to-Many (avoids cartesian product)
stmt = select(User).options(selectinload(User.posts))
users = session.execute(stmt).scalars().all()
# Query 1: SELECT * FROM users
# Query 2: SELECT * FROM posts WHERE author_id IN (1, 2, 3, ...)

# 4. SUBQUERY (subqueryload) - Two queries using subquery
# Similar to selectin, sometimes better for complex queries
stmt = select(User).options(subqueryload(User.posts))

# 5. RAISE (raiseload) - Explicitly prevent lazy loading
# Use in production to catch N+1 bugs
stmt = select(User).options(raiseload(User.posts))
```

**Interview Point**: Always be ready to explain the N+1 problem and how to solve it with `selectinload` or `joinedload`.

---

### 1.6 Transactions & Error Handling

```python
from sqlalchemy.exc import IntegrityError, SQLAlchemyError

# ==================== Basic Transaction ====================

with Session(engine) as session:
    try:
        user = User(email="existing@email.com")
        session.add(user)
        session.commit()
    except IntegrityError:
        session.rollback()  # MUST rollback on error
        print("Duplicate email!")
    except SQLAlchemyError as e:
        session.rollback()
        print(f"Database error: {e}")

# ==================== Nested Transactions (Savepoints) ====================

with Session(engine) as session:
    session.add(User(email="user1@test.com"))
    
    # Create savepoint
    with session.begin_nested():
        try:
            session.add(User(email="user1@test.com"))  # Duplicate!
            session.commit()  # This commits the nested transaction
        except IntegrityError:
            pass  # Rolled back to savepoint, outer transaction safe
    
    session.commit()  # user1 still committed

# ==================== Context Manager Pattern (Recommended) ====================

from contextlib import contextmanager

@contextmanager
def get_session():
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# Usage
with get_session() as session:
    session.add(User(email="safe@example.com"))
    # Auto-commits on success, auto-rollbacks on error
```

---

### 1.7 Querying Deep Dive

```python
from sqlalchemy import select, and_, or_, func, desc, asc, case, distinct

# ==================== Filtering ====================

# Basic comparisons
stmt = select(User).where(User.id == 1)
stmt = select(User).where(User.name != None)  # IS NOT NULL
stmt = select(User).where(User.email.like("%@gmail.com"))
stmt = select(User).where(User.email.ilike("%@GMAIL.COM"))  # Case-insensitive
stmt = select(User).where(User.id.in_([1, 2, 3]))
stmt = select(User).where(User.id.between(1, 100))

# AND / OR
stmt = select(User).where(and_(User.is_active == True, User.name != None))
stmt = select(User).where(or_(User.email.like("%@a.com"), User.email.like("%@b.com")))

# Chaining (implicit AND)
stmt = select(User).where(User.is_active == True).where(User.name != None)

# ==================== Ordering & Pagination ====================

stmt = (
    select(User)
    .order_by(desc(User.created_at))
    .offset(20)
    .limit(10)
)

# ==================== Aggregations ====================

# Count
stmt = select(func.count()).select_from(User)
count = session.execute(stmt).scalar()

# Count with condition
stmt = select(func.count()).select_from(User).where(User.is_active == True)

# Group By
stmt = (
    select(User.is_active, func.count(User.id))
    .group_by(User.is_active)
)
results = session.execute(stmt).all()  # [(True, 50), (False, 10)]

# Having
stmt = (
    select(Post.author_id, func.count(Post.id).label("post_count"))
    .group_by(Post.author_id)
    .having(func.count(Post.id) > 5)
)

# ==================== Joins ====================

# Implicit join (via relationship)
stmt = select(User, Post).join(User.posts)

# Explicit join
stmt = select(User, Post).join(Post, User.id == Post.author_id)

# Left outer join
stmt = select(User, Post).outerjoin(Post, User.id == Post.author_id)

# ==================== Subqueries ====================

# Scalar subquery
subq = select(func.count(Post.id)).where(Post.author_id == User.id).correlate(User).scalar_subquery()
stmt = select(User.name, subq.label("post_count"))

# Subquery as table
subq = select(Post.author_id, func.count(Post.id).label("cnt")).group_by(Post.author_id).subquery()
stmt = select(User.name, subq.c.cnt).join(subq, User.id == subq.c.author_id)

# ==================== CASE / Conditional ====================

stmt = select(
    User.name,
    case(
        (User.is_active == True, "Active"),
        (User.is_active == False, "Inactive"),
        else_="Unknown"
    ).label("status")
)
```

---

### 1.8 Migrations with Alembic

Alembic is SQLAlchemy's migration tool. **You WILL be asked about this.**

```bash
# Install
pip install alembic

# Initialize (creates alembic/ directory)
alembic init alembic

# Configure alembic.ini
# sqlalchemy.url = postgresql://user:pass@localhost/db
```

```python
# alembic/env.py - Set target metadata
from myapp.models import Base
target_metadata = Base.metadata
```

```bash
# Generate migration from model changes
alembic revision --autogenerate -m "Add users table"

# Apply migrations
alembic upgrade head

# Rollback one version
alembic downgrade -1

# Show current version
alembic current

# Show history
alembic history
```

```python
# Example migration file (alembic/versions/xxxx_add_users.py)
def upgrade():
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('email', sa.String(255), unique=True, nullable=False),
        sa.Column('created_at', sa.DateTime(), server_default=sa.func.now()),
    )
    op.create_index('ix_users_email', 'users', ['email'])

def downgrade():
    op.drop_index('ix_users_email')
    op.drop_table('users')
```

---

### 1.9 Performance Concepts

```python
# ==================== Bulk Operations ====================

# SLOW: Individual inserts (N round trips)
for i in range(10000):
    session.add(User(email=f"user{i}@test.com"))
session.commit()

# FAST: Bulk insert (single round trip)
from sqlalchemy import insert
session.execute(insert(User), [{"email": f"user{i}@test.com"} for i in range(10000)])
session.commit()

# FASTEST: Core bulk insert (bypasses ORM overhead)
engine.execute(User.__table__.insert(), [{"email": f"user{i}@test.com"} for i in range(10000)])

# ==================== Query Optimization ====================

# Select only needed columns (reduces data transfer)
stmt = select(User.id, User.email)  # Instead of select(User)

# Use exists() for existence checks
from sqlalchemy import exists
stmt = select(exists().where(User.email == "test@test.com"))
exists_flag = session.execute(stmt).scalar()

# Use count() efficiently
stmt = select(func.count()).select_from(User).where(User.is_active == True)

# ==================== Indexing (in model) ====================

class User(Base):
    email: Mapped[str] = mapped_column(String(255), index=True)  # Single index
    
    __table_args__ = (
        Index('ix_user_name_email', 'name', 'email'),  # Composite index
    )
```

---

## 2. Most Frequently Asked Interview Questions & Topics

### Easy (Warm-up / Phone Screen)

| Question | Key Points |
|----------|------------|
| What is an ORM? Why use SQLAlchemy? | Abstraction over SQL, DB-agnostic, prevents SQL injection, Python objects ↔ rows |
| Explain `Session` and when to use it | Unit of Work pattern, tracks changes, manages transactions |
| How do you create a model? | `DeclarativeBase`, `Mapped[]`, `mapped_column()`, `__tablename__` |
| What's the difference between `add()` and `commit()`? | `add()` stages object, `commit()` persists to DB |
| How do you handle nullable fields? | `Mapped[Optional[str]]` or `mapped_column(nullable=True)` |

### Medium (Technical Screen / Onsite)

| Question | Key Points |
|----------|------------|
| Explain the N+1 query problem and solutions | Lazy loading triggers query per object; use `selectinload`/`joinedload` |
| What are relationship loading strategies? | `lazy`, `joined`, `selectin`, `subquery`, `raise` - explain tradeoffs |
| How do transactions work? What about nested? | `session.commit()`, `session.rollback()`, `begin_nested()` for savepoints |
| Explain connection pooling | `pool_size`, `max_overflow`, `pool_recycle`, `pool_pre_ping` |
| How would you do bulk inserts efficiently? | `insert()` with list of dicts, or Core `execute()` bypassing ORM |
| What's the difference between Core and ORM? | Core = SQL Expression Language; ORM = object mapping layer on top |
| How do you handle schema migrations? | Alembic: `autogenerate`, `upgrade`, `downgrade`, version control |

### Hard (Senior / System Design)

| Question | Key Points |
|----------|------------|
| Design a multi-tenant database layer | Schema-per-tenant, `search_path`, or discriminator column with events |
| How would you implement soft deletes? | Add `deleted_at` column, use events or `@hybrid_property` for filtering |
| Explain session scoping in web apps | `scoped_session`, request-scoped sessions, FastAPI `Depends()` |
| How do you handle read replicas? | Multiple engines, routing via custom session or `execution_options` |
| What are SQLAlchemy events? How would you use them? | `@event.listens_for` - audit logging, validation, auto-timestamps |
| Optimistic vs pessimistic locking | Optimistic: version column, detect conflicts. Pessimistic: `FOR UPDATE` |
| How do you test code that uses SQLAlchemy? | In-memory SQLite, `pytest` fixtures, transaction rollback per test |

---

## 3. One-Page Cheat Sheet

### Essential Imports
```python
from sqlalchemy import create_engine, select, insert, update, delete, func, and_, or_, desc, asc
from sqlalchemy.orm import Session, DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy.orm import joinedload, selectinload, lazyload
from sqlalchemy import String, Integer, ForeignKey, DateTime, Boolean, Text, Index
```

### Quick Reference Table

| Operation | SQLAlchemy 2.0 Syntax |
|-----------|----------------------|
| Create engine | `engine = create_engine("postgresql://user:pass@host/db")` |
| Create session | `with Session(engine) as session:` |
| Define model | `class User(Base): id: Mapped[int] = mapped_column(primary_key=True)` |
| Insert | `session.add(obj); session.commit()` |
| Select one | `session.get(User, 1)` or `session.execute(select(User).where(...)).scalar_one_or_none()` |
| Select all | `session.execute(select(User)).scalars().all()` |
| Update | Modify object directly, then `commit()`, or `session.execute(update(User).where(...).values(...))` |
| Delete | `session.delete(obj)` or `session.execute(delete(User).where(...))` |
| Filter | `.where(User.name == "Alice")` |
| Join | `.join(User.posts)` or `.join(Post, User.id == Post.author_id)` |
| Eager load | `.options(selectinload(User.posts))` |
| Order | `.order_by(desc(User.created_at))` |
| Paginate | `.offset(20).limit(10)` |
| Count | `select(func.count()).select_from(User)` |
| Group | `.group_by(User.status)` |
| Transaction | `try: ... commit() except: rollback()` |

### Relationship Loading Cheat Sheet

| Strategy | SQL Queries | Best For | Pitfall |
|----------|-------------|----------|---------|
| `lazy` (default) | N+1 | Rarely accessed | N+1 problem |
| `joinedload` | 1 (JOIN) | One-to-One, Many-to-One | Cartesian explosion |
| `selectinload` | 2 (SELECT + IN) | One-to-Many, Many-to-Many | Large IN clauses |
| `subqueryload` | 2 (SELECT + subquery) | Complex queries | Query complexity |
| `raiseload` | Raises error | Detecting N+1 in prod | Breaks if accessed |

### Connection String Formats
```
SQLite:       sqlite:///./app.db  |  sqlite:///:memory:
PostgreSQL:   postgresql+psycopg2://user:pass@host:5432/db
MySQL:        mysql+pymysql://user:pass@host:3306/db
```

---

## 4. Best Practices & Common Mistakes

### ✅ What Senior Engineers Do

```python
# 1. Always use context managers for sessions
with Session(engine) as session:
    # Work here
    session.commit()
# Session automatically closed

# 2. Explicit loading strategies (no surprise N+1)
stmt = select(User).options(selectinload(User.posts))

# 3. Use get() for primary key lookups (cached)
user = session.get(User, user_id)

# 4. Bulk operations for large datasets
session.execute(insert(User), list_of_dicts)

# 5. Index frequently queried columns
email: Mapped[str] = mapped_column(String(255), index=True)

# 6. Use expire_on_commit=False when needed
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)

# 7. Connection pool configuration for production
engine = create_engine(url, pool_pre_ping=True, pool_recycle=1800)

# 8. Type hints everywhere (SQLAlchemy 2.0)
id: Mapped[int] = mapped_column(primary_key=True)

# 9. Alembic for ALL schema changes
# Never use Base.metadata.create_all() in production
```

### ❌ Common Mistakes That Get You Rejected

```python
# 1. Forgetting to commit/rollback
session.add(user)
# Forgot session.commit() - nothing saved!

# 2. Ignoring N+1 queries
users = session.execute(select(User)).scalars().all()
for user in users:
    print(user.posts)  # N+1 queries! Use selectinload

# 3. Not handling IntegrityError
session.add(User(email="duplicate@test.com"))
session.commit()  # Crashes! Wrap in try/except

# 4. Using create_all() in production
Base.metadata.create_all(engine)  # BAD - use Alembic migrations

# 5. Accessing detached objects
with Session(engine) as session:
    user = session.get(User, 1)
user.posts  # ERROR! Session closed, can't lazy load

# 6. Not closing sessions (connection leak)
session = Session(engine)
# ... work ...
# Forgot session.close()!

# 7. String concatenation in queries (SQL injection)
session.execute(text(f"SELECT * FROM users WHERE id = {user_id}"))  # DANGER!
# Use: session.execute(text("SELECT * FROM users WHERE id = :id"), {"id": user_id})

# 8. Modifying objects outside session
user = session.get(User, 1)
session.close()
user.name = "New"  # Change not tracked, never saved

# 9. Using synchronous SQLAlchemy in async code
# Use sqlalchemy.ext.asyncio for async applications
```

---

## 5. Minimal but Complete Code Examples

### Example 1: Complete CRUD API Model

```python
"""
Production-ready SQLAlchemy setup for a FastAPI/Flask application.
Covers: Models, Session management, CRUD operations.
"""

from datetime import datetime
from typing import Optional, List
from sqlalchemy import create_engine, String, ForeignKey, func, select
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, relationship,
    Session, sessionmaker, selectinload
)

# ============ Configuration ============
DATABASE_URL = "postgresql+psycopg2://user:pass@localhost/myapp"

engine = create_engine(
    DATABASE_URL,
    pool_size=5,
    max_overflow=10,
    pool_pre_ping=True,
    pool_recycle=1800,
)

SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)

# ============ Models ============
class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[Optional[str]] = mapped_column(String(100))
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    
    posts: Mapped[List["Post"]] = relationship(
        back_populates="author",
        cascade="all, delete-orphan",
        lazy="selectin"
    )

class Post(Base):
    __tablename__ = "posts"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str]
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id", ondelete="CASCADE"))
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    
    author: Mapped["User"] = relationship(back_populates="posts")

# ============ Repository Pattern ============
class UserRepository:
    def __init__(self, session: Session):
        self.session = session
    
    def get_by_id(self, user_id: int) -> Optional[User]:
        return self.session.get(User, user_id)
    
    def get_by_email(self, email: str) -> Optional[User]:
        stmt = select(User).where(User.email == email)
        return self.session.execute(stmt).scalar_one_or_none()
    
    def get_all(self, skip: int = 0, limit: int = 100) -> List[User]:
        stmt = select(User).offset(skip).limit(limit)
        return list(self.session.execute(stmt).scalars().all())
    
    def create(self, email: str, name: Optional[str] = None) -> User:
        user = User(email=email, name=name)
        self.session.add(user)
        self.session.flush()  # Get ID without committing
        return user
    
    def delete(self, user: User) -> None:
        self.session.delete(user)

# ============ Usage ============
def main():
    # Create tables (use Alembic in production!)
    Base.metadata.create_all(engine)
    
    with SessionLocal() as session:
        repo = UserRepository(session)
        
        # Create
        user = repo.create(email="alice@example.com", name="Alice")
        session.commit()
        print(f"Created user with ID: {user.id}")
        
        # Read
        found_user = repo.get_by_email("alice@example.com")
        print(f"Found: {found_user}")
        
        # Update
        found_user.name = "Alice Smith"
        session.commit()
        
        # Delete
        repo.delete(found_user)
        session.commit()

if __name__ == "__main__":
    main()
```

---

### Example 2: Complex Queries with Joins and Aggregations

```python
"""
Advanced querying: Joins, subqueries, aggregations, window functions.
"""

from sqlalchemy import select, func, desc, case, and_, or_
from sqlalchemy.orm import aliased

# Setup (assume User and Post models from Example 1)

with SessionLocal() as session:
    
    # ========== JOIN: Users with their post count ==========
    stmt = (
        select(User.name, func.count(Post.id).label("post_count"))
        .outerjoin(Post)
        .group_by(User.id)
        .order_by(desc("post_count"))
    )
    results = session.execute(stmt).all()
    # [('Alice', 10), ('Bob', 5), ('Charlie', 0)]
    
    # ========== SUBQUERY: Users with above-average posts ==========
    avg_posts_subq = (
        select(func.avg(func.count(Post.id)))
        .select_from(Post)
        .group_by(Post.author_id)
        .scalar_subquery()
    )
    
    post_count_subq = (
        select(Post.author_id, func.count(Post.id).label("cnt"))
        .group_by(Post.author_id)
        .subquery()
    )
    
    stmt = (
        select(User)
        .join(post_count_subq, User.id == post_count_subq.c.author_id)
        .where(post_count_subq.c.cnt > avg_posts_subq)
    )
    prolific_users = session.execute(stmt).scalars().all()
    
    # ========== CASE: Categorize users by activity ==========
    stmt = select(
        User.name,
        case(
            (func.count(Post.id) == 0, "Inactive"),
            (func.count(Post.id) < 5, "Casual"),
            (func.count(Post.id) < 20, "Active"),
            else_="Power User"
        ).label("category")
    ).outerjoin(Post).group_by(User.id)
    
    categorized = session.execute(stmt).all()
    
    # ========== EXISTS: Find users with recent posts ==========
    from sqlalchemy import exists
    from datetime import datetime, timedelta
    
    recent_cutoff = datetime.utcnow() - timedelta(days=7)
    
    recent_post_exists = (
        exists()
        .where(Post.author_id == User.id)
        .where(Post.created_at > recent_cutoff)
    )
    
    stmt = select(User).where(recent_post_exists)
    active_users = session.execute(stmt).scalars().all()
    
    # ========== Self-join: Find users who commented on each other ==========
    # (Assuming a Comment model with commenter_id and post_id)
    """
    UserA = aliased(User)
    UserB = aliased(User)
    
    stmt = (
        select(UserA.name, UserB.name)
        .select_from(Comment)
        .join(Post, Comment.post_id == Post.id)
        .join(UserA, Comment.commenter_id == UserA.id)
        .join(UserB, Post.author_id == UserB.id)
        .where(UserA.id != UserB.id)
        .distinct()
    )
    """
```

---

### Example 3: FastAPI Integration with Dependency Injection

```python
"""
Production pattern for FastAPI + SQLAlchemy.
Covers: Dependency injection, request-scoped sessions, Pydantic integration.
"""

from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel, EmailStr
from typing import Generator, List
from sqlalchemy.orm import Session
from sqlalchemy import select

# Assume engine, SessionLocal, User, Post from Example 1

app = FastAPI()

# ============ Dependency Injection ============
def get_db() -> Generator[Session, None, None]:
    """Yield a session per request, auto-close after."""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# ============ Pydantic Schemas ============
class UserCreate(BaseModel):
    email: EmailStr
    name: str | None = None

class UserResponse(BaseModel):
    id: int
    email: str
    name: str | None
    is_active: bool
    
    class Config:
        from_attributes = True  # Enable ORM mode (was orm_mode in Pydantic v1)

class UserWithPosts(UserResponse):
    posts: List["PostResponse"] = []

class PostResponse(BaseModel):
    id: int
    title: str
    content: str
    
    class Config:
        from_attributes = True

# ============ Endpoints ============
@app.post("/users/", response_model=UserResponse)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    # Check for existing email
    stmt = select(User).where(User.email == user.email)
    existing = db.execute(stmt).scalar_one_or_none()
    if existing:
        raise HTTPException(status_code=400, detail="Email already registered")
    
    db_user = User(email=user.email, name=user.name)
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

@app.get("/users/{user_id}", response_model=UserWithPosts)
def get_user(user_id: int, db: Session = Depends(get_db)):
    # Eager load posts to avoid N+1
    stmt = select(User).where(User.id == user_id).options(selectinload(User.posts))
    user = db.execute(stmt).scalar_one_or_none()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.get("/users/", response_model=List[UserResponse])
def list_users(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    stmt = select(User).offset(skip).limit(limit)
    users = db.execute(stmt).scalars().all()
    return users

@app.delete("/users/{user_id}")
def delete_user(user_id: int, db: Session = Depends(get_db)):
    user = db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    db.delete(user)
    db.commit()
    return {"message": "User deleted"}
```

---

### Example 4: Events, Soft Deletes, and Audit Logging

```python
"""
Advanced patterns: Events, soft deletes, automatic timestamps, audit trail.
"""

from datetime import datetime
from typing import Optional
from sqlalchemy import event, String, DateTime, Boolean, func, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session

class Base(DeclarativeBase):
    pass

# ============ Mixin for common fields ============
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(),
        onupdate=func.now()
    )

class SoftDeleteMixin:
    deleted_at: Mapped[Optional[datetime]] = mapped_column(default=None)
    
    @property
    def is_deleted(self) -> bool:
        return self.deleted_at is not None
    
    def soft_delete(self):
        self.deleted_at = datetime.utcnow()

# ============ Model with mixins ============
class User(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "users"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)
    name: Mapped[Optional[str]] = mapped_column(String(100))

# ============ Audit Log ============
class AuditLog(Base):
    __tablename__ = "audit_logs"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    table_name: Mapped[str] = mapped_column(String(100))
    record_id: Mapped[int]
    action: Mapped[str] = mapped_column(String(20))  # INSERT, UPDATE, DELETE
    old_values: Mapped[Optional[str]]  # JSON string
    new_values: Mapped[Optional[str]]  # JSON string
    timestamp: Mapped[datetime] = mapped_column(server_default=func.now())

# ============ Event Listeners ============
import json

@event.listens_for(User, "after_insert")
def log_insert(mapper, connection, target):
    connection.execute(
        AuditLog.__table__.insert().values(
            table_name="users",
            record_id=target.id,
            action="INSERT",
            new_values=json.dumps({"email": target.email, "name": target.name})
        )
    )

@event.listens_for(User, "after_update")
def log_update(mapper, connection, target):
    # Get changed attributes
    changes = {}
    for attr in ["email", "name", "deleted_at"]:
        hist = getattr(target, attr)
        # In real implementation, use inspect(target).attrs[attr].history
        changes[attr] = getattr(target, attr)
    
    connection.execute(
        AuditLog.__table__.insert().values(
            table_name="users",
            record_id=target.id,
            action="UPDATE",
            new_values=json.dumps(changes)
        )
    )

# ============ Query helper for soft deletes ============
def get_active_users(session: Session):
    """Only return non-deleted users."""
    stmt = select(User).where(User.deleted_at == None)
    return session.execute(stmt).scalars().all()

# ============ Usage ============
"""
with SessionLocal() as session:
    user = User(email="test@test.com", name="Test")
    session.add(user)
    session.commit()  # Triggers after_insert event
    
    user.name = "Updated"
    session.commit()  # Triggers after_update event
    
    user.soft_delete()
    session.commit()  # Soft delete instead of hard delete
    
    # Query only active users
    active = get_active_users(session)
"""
```

---

### Example 5: Async SQLAlchemy (SQLAlchemy 2.0)

```python
"""
Async SQLAlchemy for high-concurrency applications (FastAPI async).
"""

import asyncio
from typing import List, Optional
from sqlalchemy import String, select
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

# ============ Async Engine Setup ============
# Note: Use async driver (asyncpg for PostgreSQL, aiosqlite for SQLite)
ASYNC_DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/myapp"

async_engine = create_async_engine(
    ASYNC_DATABASE_URL,
    pool_size=5,
    max_overflow=10,
    echo=False,
)

AsyncSessionLocal = async_sessionmaker(
    bind=async_engine,
    class_=AsyncSession,
    expire_on_commit=False,
)

# ============ Model ============
class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)
    name: Mapped[Optional[str]] = mapped_column(String(100))

# ============ Async CRUD ============
async def create_user(session: AsyncSession, email: str, name: str = None) -> User:
    user = User(email=email, name=name)
    session.add(user)
    await session.commit()
    await session.refresh(user)
    return user

async def get_user(session: AsyncSession, user_id: int) -> Optional[User]:
    return await session.get(User, user_id)

async def get_all_users(session: AsyncSession) -> List[User]:
    stmt = select(User)
    result = await session.execute(stmt)
    return list(result.scalars().all())

async def update_user(session: AsyncSession, user_id: int, name: str) -> Optional[User]:
    user = await session.get(User, user_id)
    if user:
        user.name = name
        await session.commit()
    return user

async def delete_user(session: AsyncSession, user_id: int) -> bool:
    user = await session.get(User, user_id)
    if user:
        await session.delete(user)
        await session.commit()
        return True
    return False

# ============ FastAPI Async Dependency ============
"""
from fastapi import FastAPI, Depends

app = FastAPI()

async def get_async_db():
    async with AsyncSessionLocal() as session:
        yield session

@app.get("/users/{user_id}")
async def read_user(user_id: int, db: AsyncSession = Depends(get_async_db)):
    user = await get_user(db, user_id)
    if not user:
        raise HTTPException(status_code=404)
    return user
"""

# ============ Main (for testing) ============
async def main():
    # Create tables
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    
    async with AsyncSessionLocal() as session:
        user = await create_user(session, "async@test.com", "Async User")
        print(f"Created: {user.id}")
        
        all_users = await get_all_users(session)
        print(f"All users: {all_users}")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 6. Recommended Problems to Solve Today

Since SQLAlchemy is a library (not algorithmic), practice these **practical exercises**:

| # | Problem | Difficulty | Focus Area |
|---|---------|------------|------------|
| 1 | Build a Blog API with Users, Posts, Comments (CRUD) | Easy | Basic ORM, relationships |
| 2 | Implement pagination with cursor-based and offset-based | Medium | Querying, performance |
| 3 | Add full-text search using PostgreSQL `tsvector` | Medium | Raw SQL, hybrid properties |
| 4 | Implement soft deletes with query filtering | Medium | Events, mixins |
| 5 | Create a multi-tenant system (schema per tenant) | Hard | Advanced configuration |
| 6 | Build an audit log using SQLAlchemy events | Medium | Events, triggers |
| 7 | Implement optimistic locking with version column | Hard | Concurrency |
| 8 | Write a migration that adds a column with backfill | Medium | Alembic |
| 9 | Set up read replica routing (writes→primary, reads→replica) | Hard | Multiple engines |
| 10 | Create async endpoints for high-concurrency service | Medium | Async SQLAlchemy |

**System Design Exercises:**
- Design a rate limiter using SQLAlchemy
- Design a job queue with status tracking
- Design a notification system with read/unread state

---

## 7. 30-Minute Quiz

Test yourself! Answers at the bottom.

### Questions

**Q1.** What is the correct SQLAlchemy 2.0 way to define a nullable string column?
```python
# A
name = Column(String(100), nullable=True)
# B
name: Mapped[Optional[str]] = mapped_column(String(100))
# C
name: Mapped[str] = mapped_column(String(100), nullable=True)
# D
name: Optional[Mapped[str]] = mapped_column(String(100))
```

**Q2.** What's wrong with this code?
```python
with Session(engine) as session:
    user = session.get(User, 1)
print(user.posts)  # Accessing relationship here
```

**Q3.** Which loading strategy prevents the N+1 problem for a One-to-Many relationship most efficiently?
- A) `lazy="joined"`
- B) `lazy="selectin"`
- C) `lazy="dynamic"`
- D) `lazy="raise"`

**Q4.** What does `pool_pre_ping=True` do?

**Q5.** How do you perform a bulk insert of 10,000 rows efficiently?
```python
# A
for item in items:
    session.add(Model(**item))
session.commit()

# B
session.execute(insert(Model), items)
session.commit()

# C
session.bulk_insert_mappings(Model, items)
session.commit()

# D
Both B and C are efficient
```

**Q6.** What's the difference between `session.flush()` and `session.commit()`?

**Q7.** In this query, what does `.unique()` do and why is it needed?
```python
stmt = select(User).options(joinedload(User.posts))
users = session.execute(stmt).unique().scalars().all()
```

**Q8.** How do you implement a composite primary key?
```python
class PostTag(Base):
    __tablename__ = "post_tags"
    # Fill in the blanks
```

**Q9.** What happens if you forget `session.rollback()` after an `IntegrityError`?

**Q10.** Write the SQLAlchemy 2.0 query for: "Find all users who have more than 5 posts"

---

### Answers

<details>
<summary>Click to reveal answers</summary>

**A1.** **B** is correct.
- `Mapped[Optional[str]]` is the 2.0 way to indicate nullable
- A is 1.x syntax
- C would work but is less idiomatic
- D is incorrect syntax

**A2.** The session is closed when accessing `user.posts`. Since `posts` is lazy-loaded by default, accessing it outside the session raises `DetachedInstanceError`. Solutions:
- Use `expire_on_commit=False`
- Eager load with `selectinload(User.posts)`
- Access `.posts` inside the `with` block

**A3.** **B) `lazy="selectin"`** — Uses 2 queries (main + IN clause), avoids cartesian product from JOINs with multiple rows.

**A4.** Before using a connection from the pool, it sends a lightweight query (`SELECT 1`) to verify the connection is alive. Prevents errors from stale/dropped connections.

**A5.** **D** — Both B (`insert()` with list) and C (`bulk_insert_mappings`) are efficient. Option A is slow (N round trips). B is preferred in 2.0.

**A6.** 
- `flush()`: Sends pending changes to DB but doesn't commit transaction. Useful for getting auto-generated IDs.
- `commit()`: Calls flush() then commits the transaction, making changes permanent.

**A7.** `joinedload` creates a JOIN that can return duplicate parent rows (one per child). `.unique()` deduplicates them so you get each User once.

**A8.**
```python
class PostTag(Base):
    __tablename__ = "post_tags"
    post_id: Mapped[int] = mapped_column(ForeignKey("posts.id"), primary_key=True)
    tag_id: Mapped[int] = mapped_column(ForeignKey("tags.id"), primary_key=True)
```

**A9.** The session enters an invalid state. All subsequent operations will fail with `InvalidRequestError`. The session cannot be used until rolled back or closed.

**A10.**
```python
from sqlalchemy import select, func

stmt = (
    select(User)
    .join(Post)
    .group_by(User.id)
    .having(func.count(Post.id) > 5)
)
users = session.execute(stmt).scalars().all()
```

</details>

---

## 8. Further Learning

### Top Resource #1: Tutorial
**SQLAlchemy 2.0 Tutorial** (Official)  
https://docs.sqlalchemy.org/en/20/tutorial/index.html

*The canonical guide. Read the ORM sections thoroughly. Takes ~3-4 hours.*

### Top Resource #2: Documentation Reference
**SQLAlchemy ORM Documentation**  
https://docs.sqlalchemy.org/en/20/orm/index.html

*Use as reference for specific topics: relationships, loading strategies, events, session lifecycle.*

---

## Quick Review: What Interviewers Actually Test

1. **Can you avoid N+1 queries?** → Know loading strategies cold
2. **Do you understand session lifecycle?** → States, commit/rollback, context managers
3. **Can you write complex queries?** → Joins, subqueries, aggregations
4. **Do you know production concerns?** → Connection pooling, migrations, bulk operations
5. **Can you integrate with web frameworks?** → Dependency injection, async patterns

**Good luck! You've got this.** 🚀
