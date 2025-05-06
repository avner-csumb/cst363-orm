---
# theme: default
title: Welcome to Slidev
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)

drawings:
  persist: false

# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# open graph

---

# Object Relational Mappers

CST 363

---


## What is an ORM?

<br>

- Description of how an object model "maps" to a relational data model
    - Class to table(s)
    - Object properties (attributes) to columns or relations

<br>


<img src="/orm_overview.png" width=500>


---

## Why ORMs Exist

<br>

- **Impedance mismatch** --- Python, Java, etc. work with *objects*, Postgres, MySQL, etc. store *rows*.  
  Bridging the two manually means endless `cursor.execute()` calls, hand‑written SQL strings, and error‑prone type conversions.

- **Raise the abstraction** --- focus on domain models (`User`, `Order`, `Product`) instead of SQL plumbing.

- **Database portability** --- the *same* Python code can speak to PostgreSQL in production and SQLite during unit‑tests.

- **Still SQL‑centered** --- an ORM *generates* SQL; you can drop down to raw SQL any time.


<!-- conceptual and structural differences between the way data is represented and manipulated in object-oriented code versus relational databases -->

---

## Trade‑offs at a Glance

<br>

| **Strengths** | **Watch out for...** |
|--------------|---------------|
| Faster development & fewer lines of code | Hidden queries → *N + 1* problems |
| Compile‑time model validation | Harder to squeeze the last % of performance |
| Safe parameter binding (no SQL‑injection) | Learning curve: sessions, identity map |
| Easy migrations via Alembic | May tempt you to ignore good schema design |


<!-- 
Parameter binding: binding actual values to those placeholders when the query is executed

SELECT * FROM users WHERE id = ?

the ? is a parameter (or placeholder)
 -->


---

## Aside: SQL Injection

<br>

```python
def login():
    username = request.args.get('username', '')
    password = request.args.get('password', '')

    conn = get_db()
    cur = conn.cursor()

    # Unsafe: directly interpolating user input into SQL
    query = (
        f"SELECT id, username "
        f"FROM users "
        f"WHERE username = '{username}' "
        f"  AND password = '{password}';"
    )
    cur.execute(query)
    user = cur.fetchone()
    conn.close()
```


---

## Why this is dangerous?

An attacker can craft a URL like:

```sql
/login?username=admin'--&password=anything
```

which makes the SQL look like:

```sql
SELECT id, username
FROM users
WHERE username = 'admin'--'
  AND password = 'anything';
```

<br>

- Everything after `--` is treated as a comment, so the `AND password = ...` check is skipped. 
    - effectively logs them in as “admin” without knowing the password.


---

## Demonstration of a Classic Injection

Bypass authentication:

```sql
/login?username=' OR '1'='1&password=' OR '1'='1
```

<br>

```sql
SELECT id, username
FROM users
WHERE username = '' OR '1'='1'
  AND password = '' OR '1'='1';
```

Since `'1'='1'` is always true, the query returns the first user.


---


## Destructive commands

<br>

```sql
/login?username=anything'; DROP TABLE users;--&password=x
```

<br>

```sql
SELECT id, username
FROM users
WHERE username = 'anything';
DROP TABLE users;--'
  AND password = 'x';
```

<br>

which would delete your entire users table.


---

## How an ORM can help here

- Built-in Parameterization
    - Query API parameters are automatically bound for you
        - you never string-interpolate user input into SQL. 
    Injection risks become effectively zero.

- Domain-centric Code
    - Instead of hand-writing raw SQL, you work with, e.g., Python classes and objects.
    
- Cross-Database Portability
    - You write the same ORM-based code whether you're on SQLite, PostgreSQL, MySQL, or Oracle. 
        - No conditional SQL syntax in your application logic.


<!-- 
- When you write queries using an ORM, you don’t manually construct SQL strings with user input. Instead, you pass values as parameters, and the ORM safely binds them behind the scenes.

- interact with your database through objects and classes that represent your application’s domain—like User, Order, Comment—instead of raw SQL.

 -->

---


## More Benefits

- Maintainability & Readability
    - Complex joins, transactions, and migrations become declarative
        - you can reason about your data model in one place (your Python classes) rather than scattered SQL strings.

- Automatic Migrations & Schema Management
    - With extensions like Alembic, you can version and evolve your schema alongside your code, instead of manually writing `ALTER TABLE` scripts.




---



## SQLAlchemy: Two Layers

<br>

1. **Core** – thin, explicit SQL construction layer.  
   - Think “Pythonic SQL string‑builder.”
   - Includes schema management functions, SQL query builder, and schema inspection utilities
   

<br>

2. **ORM** – builds on Core, adds identity map, sessions, and Python classes that map to tables.

<br>

> We’ll use the 2.0 style API (released 2023), which is declarative‑by‑default and async‑friendly.

<br><br>

<img src="/sqlalchemy.jpg" width="300">

---


## Getting Set Up

<br>

With virtual environment activated: `pip install sqlalchemy psycopg[binary] alembic`

- Postgres must be running 

- Create a database


---

## Connecting

<br>

```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg://postgres:ott3r@localhost:5431/my_db",
    echo=True
)
```

<br>

- This is based on our existing PostgreSQL Docker container and a database called `my_db`
- `engine` is **cheap**; create once, reuse everywhere.

<br>



---

## Declaring Models (1 of 3)

<br>


```python
from typing import List
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import ForeignKey, Text


class Base(DeclarativeBase):
    pass
```
<br>

- `DeclarativeBase` --- to define your base class --- a single shared foundation for all models 
- `Mapped[...]`  --- marks an attribute as a mapped column or relationship
- `mapped_column(...)` --- defines a column 
- `relationship(...)` --- defines ORM relationships between tables


---

## Declaring Models (2 of 3)

<br>

```python
class User(Base):
    __tablename__ = 'users'

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(nullable=False)
    email_address: Mapped[str] = mapped_column(nullable=True)
    comments: Mapped[List["Comment"]] = relationship(back_populates="user")

    def __repr__(self) -> str:
        return f"<User username={self.username!r}>"
```

<br>


- `back_populates` creates a bidirectional relationship between `User` and `Comment`
    - It connects both sides of a relationship explicitly, so changes on one side are reflected on the other

---

## Declaring Models (3 of 3)

<br>

```python
class Comment(Base):
    __tablename__ = 'comments'

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), nullable=False)
    text: Mapped[str] = mapped_column(Text, nullable=False)
    user: Mapped["User"] = relationship(back_populates="comments")

    def __repr__(self) -> str:
        return f"<Comment text={self.text!r} by {self.user.username!r}>"
```

---


##  Creating the Schema

<br>

 Generates and runs `CREATE TABLE ... SQL`

```python
Base.metadata.create_all(bind=engine)     
```

---

## Unit of Work: Sessions

<br>

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    # create a user plus a comment in one go
    new_user    = User(username="joe", email_address="joe@example.com")
    new_comment = Comment(text="First post!", user=new_user)

    session.add(new_user)   # new_comment is cascaded via the relationship
    session.commit()        # both INSERTs happen here

```

Key ideas:

1. **Identity map** – within a session, each row is represented by *one* Python object.
2. **Lazy vs. eager loading** – accessing `customer.orders` may trigger a SELECT.
3. **Transactions** – `Session` wraps every flush in a database transaction.


---


## Querying

<br>

```python
from sqlalchemy import select

with Session(engine) as session:
    stmt = (
        select(Comment)
        .join(Comment.user)
        .where(User.username == "joe")
        .order_by(Comment.id.desc())
        .limit(5)
    )
    recent_comments = session.scalars(stmt).all()
    # → gives you a List[Comment]

```

<br>

* `select()` → SQL is built safely.
* `.scalars()` flattens the `Row` objects to the first column (the `Order`).


---


## Lazy, Eager, and "N + 1"


The "N + 1" problem is a common performance pitfall when using an ORM with lazy-loaded relationships.

- One query to load the parent objects

- N additional queries to load each child collection

<br>

> Hence, 1 + N queries total—every time you iterate, you pay the cost of a separate round-trip to the database.

<!--

Demonstrate with echo logs:

```python
with Session(engine) as session:
    users = session.scalars(select(User).limit(3)).all()
    for u in users:
        print([c.text for c in u.comments])
        # ← each access to u.comments issues its own SELECT (N extra queries)

``` -->

---


## How it happens

```sql
with Session(engine) as session:
    users = session.scalars(select(User).limit(3)).all()
    for u in users:
        # Lazy: when you first access `u.comments`, SQLAlchemy issues:
        #   SELECT * FROM comments WHERE comments.user_id = <u.id>
        print([c.text for c in u.comments])
```

1. Query #1                    <br> `SELECT * FROM users LIMIT 3;`

2. Query #2 (for user 1)        <br> `SELECT * FROM comments WHERE user_id = 1;`

3. Query #3 (for user 2)        <br> `SELECT * FROM comments WHERE user_id = 2;`

4. Query #4 (for user 3)        <br> `SELECT * FROM comments WHERE user_id = 3;`




---


# N + 1 (cont.)

<br>

That's four queries to get three users and their comments.

- Round-trip latency multiplies with each extra query.
- Under load, the database sees a storm of tiny queries rather than a few bigger ones.
- Becomes especially painful when N (number of parents) grows.

<br>

>Use eager loading so that related rows are fetched in bulk

<!-- 
- Eager loading in an ORM refers to the practice of loading related data up front, in a single query (or a small set of queries), instead of deferring it until it's accessed (which would be lazy loading).

 -->



---

## Mitigate the N + 1:

<br>

```python
from sqlalchemy.orm import selectinload

stmt = (
    select(User)
    .options(selectinload(User.comments))
    .limit(3)
)

with Session(engine) as session:
    users = session.scalars(stmt).all()
    for u in users:
        print([c.text for c in u.comments])
        # ← no extra queries: comments are loaded in bulk

```
---

## SQLAlchemy + Pandas ❤️‍🔥

<br> 

- A Pandas `DataFrame` is a 2D labeled data structure in Python (like a spreadsheet or SQL table)
    - a collection of rows and columns with mixed data types (int, float, string, etc.).
    - Built on top of NumPy, optimized for performance and memory.
    - Commonly used for data analysis, cleaning, filtering, grouping, and visualization prep.
    - Can be created from: Lists or dictionaries, CSV, Excel, SQL databases, etc.

- Common use cases of this powerful combo:
    - Read SQL query results into a `DataFrame`
    - Write a `DataFrame` into a database table
    - Run analytical queries directly with Pandas on top of your database

---

## Example 1: Read from database into DataFrame

<br>

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql+psycopg2://user:pass@localhost/dbname")

df = pd.read_sql("SELECT * FROM users", con=engine)
```

- Under the hood, Pandas uses SQLAlchemy to connect.

- You can also pass a full select() object from SQLAlchemy Core.


---

## Example 2: Write DataFrame to database

<br>

```python
df.to_sql("users", con=engine, if_exists="replace", index=False)
```

- This creates or replaces a table called `users`.

- `if_exists="append"` adds rows to the table instead.

---

## Example 3: Use ORM to get data, then convert to DataFrame

<br>

```python
from sqlalchemy.orm import Session
from models import User

with Session(engine) as session:
    users = session.query(User).all()

# Convert list of ORM objects to DataFrame
df = pd.DataFrame([{
    "id": user.id,
    "username": user.username,
    "email": user.email_address
} for user in users])
```

- This is useful if you're using the ORM and want full control over fields.


---


## Migrations with Alembic (Brief)

<br>

1. `alembic init alembic` <br>

2. Edit `alembic.ini` to point at the same URL. <br>

3. `alembic revision --autogenerate -m "add order tables"` <br>

4. `alembic upgrade head`

<br>

By putting every schema change into a migration script and committing it, you:

- Have a clear, chronological history of how your schema evolved.
- Can reproduce the same database structure on any machine (dev, test, CI, production).
- Avoid accidental data loss or “works on my machine” drift.


---


##  When to *Skip* the ORM

<br>


- Use the ORM for everyday create-read-update-delete and straightforward query work—its object-mapping and identity-tracking save you boilerplate. 

-  When you need full control of SQL, want bulk-friendly operations, or are squeezing out every last microsecond of performance, reach for Core (or plain SQL strings) instead.



---

## Further Reading & Exercises

<br>

**Reading** --- SQLAlchemy 2.0 tutorial sections: <a href="https://docs.sqlalchemy.org/en/20/orm/quickstart.html">ORM Quick Start</a>, Relationship Patterns.  <br>

<!-- 2. **Hands‑on** – extend the e‑commerce schema with a `Coupon` model and demonstrate a one‑to‑many relationship (`Coupon.redemptions`).  <br>
3. **Challenge** – profile an *N + 1* situation and refactor with `selectinload`. -->

<!-- ---

## Key Takeaways

<br>

* ORMs like SQLAlchemy improve productivity **without discarding SQL**.
* Understand the **Session lifecycle**, **lazy loading**, and **transactions** to avoid surprises.
* Use the **right abstraction level** for every problem: ORM ↔ Core ↔ raw SQL. -->
