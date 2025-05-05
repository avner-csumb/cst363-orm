---
# You can also start simply with 'default'
# theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: Welcome to Slidev
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
# transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# open graph
# seoMeta:
#  ogImage: https://cover.sli.dev
---

# Object Relational Mapper
CST 363



---

## 1  Why ORMs Exist

* **Impedance mismatch** – Python works with *objects*, Postgres stores *rows*.  
  Bridging the two manually means endless `cursor.execute()` calls, hand‑written SQL strings, and error‑prone type conversions.

* **Raise the abstraction** – let instructors and students focus on domain models (`User`, `Order`, `Product`) instead of SQL plumbing.

* **Database portability** – the *same* Python code can speak to Postgres in production and SQLite during unit‑tests.

* **Still SQL‑centred** – an ORM *generates* SQL; you can drop down to raw SQL any time.

---

## 2  Trade‑offs at a Glance

| ✅ Strengths | ⚠️ Watch‑outs |
|--------------|---------------|
| Faster development & fewer lines of code | Hidden queries → *N + 1* problems |
| Compile‑time model validation | Harder to squeeze the last % of performance |
| Safe parameter binding (no SQL‑injection) | Learning curve: sessions, identity map |
| Easy migrations via Alembic | May tempt you to ignore good schema design |


---

## 3  SQLAlchemy: Two Layers

1. **Core** – thin, explicit SQL construction layer.  
   *Think “Pythonic SQL string‑builder.”*

2. **ORM** – builds on Core, adds identity map, sessions, and Python classes that map to tables.

> **We’ll teach with the 2.0 style API (released 2023), which is declarative‑by‑default and async‑friendly.**

---


## 4  Getting Set Up

```bash
python -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate
pip install sqlalchemy psycopg[binary] alembic
```

*Postgres must be running (Docker‑compose, Homebrew, etc.).*  
Create a database and user:

```bash
psql -U postgres
CREATE DATABASE ecommerce;
CREATE USER shop WITH PASSWORD 'secret';
GRANT ALL ON DATABASE ecommerce TO shop;
```


---

## 5  Connecting

```python
from sqlalchemy import create_engine
engine = create_engine(
    "postgresql+psycopg://shop:secret@localhost:5432/ecommerce",
    echo=True,      # echo SQL to stdout for teaching moments
)
```

`engine` is **cheap**; create once, reuse everywhere.

---

## 6  Declaring Models (E‑commerce Example)


```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import ForeignKey
from datetime import datetime

class Base(DeclarativeBase):
    pass
```

---

```python
class Customer(Base):
    __tablename__ = "customers"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    email: Mapped[str] = mapped_column(unique=True)
    created_at: Mapped[datetime] = mapped_column(server_default="now()")

    orders: Mapped[list["Order"]] = relationship(back_populates="customer")
```

---

```python

class Product(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    price_cents: Mapped[int]
```

---

```python
class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    customer_id: Mapped[int] = mapped_column(ForeignKey("customers.id"))
    placed_at: Mapped[datetime] = mapped_column(server_default="now()")

    customer: Mapped["Customer"] = relationship(back_populates="orders")
    items:    Mapped[list["OrderItem"]] = relationship(back_populates="order")
```

---

```python
class OrderItem(Base):
    __tablename__ = "order_items"

    order_id:   Mapped[int] = mapped_column(ForeignKey("orders.id"), primary_key=True)
    product_id: Mapped[int] = mapped_column(ForeignKey("products.id"), primary_key=True)
    qty:        Mapped[int]

    order:   Mapped["Order"] = relationship(back_populates="items")
```


---


## 7  Creating the Schema

```python
Base.metadata.create_all(engine)      # Generates and runs CREATE TABLE … SQL
```

---

## 8  Unit of Work: Sessions

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    new_customer = Customer(email="avner@example.com")
    session.add(new_customer)         # staged
    session.commit()                  # INSERT happens here
```

Key ideas to hammer home:

1. **Identity map** – within a session, each row is represented by *one* Python object.
2. **Lazy vs. eager loading** – accessing `customer.orders` may trigger a SELECT.
3. **Transactions** – `Session` wraps every flush in a database transaction.


---


## 9  Querying

```python
from sqlalchemy import select

with Session(engine) as session:
    stmt = (
        select(Order)
        .join(Order.customer)
        .where(Customer.email == "avner@example.com")
        .order_by(Order.placed_at.desc())
        .limit(5)
    )
    recent_orders = session.scalars(stmt).all()
```

Explain:

* `select()` → SQL is built safely.
* `.scalars()` flattens the `Row` objects to the first column (the `Order`).


---


## 10  Lazy, Eager, and “N + 1”

Demonstrate with echo logs:

```python
orders = session.scalars(select(Order).limit(3)).all()
for o in orders:
    print([item.qty for item in o.items])   # may emit 3 extra queries!
```

Mitigate:

```python
from sqlalchemy.orm import selectinload

stmt = select(Order).options(selectinload(Order.items)).limit(3)
```
---

## 11  Migrations with Alembic (Brief)

1. `alembic init alembic`
2. Edit `alembic.ini` to point at the same URL.
3. `alembic revision --autogenerate -m "add order tables"`
4. `alembic upgrade head`

Teach that **DDL belongs in version control**; never `DROP TABLE` without a migration.

---

## 12  Performance Tips

* Keep an eye on **query count**; integrate `pytest-sqlalchemy` or a custom echo logger in unit tests.
* Prefer **bulk operations** (`session.execute()` with raw SQL) for mass updates.
* Tune **statement caching** (`engine = create_engine(..., pool_size=10, max_overflow=20)`).

---

## 13  When to *Skip* the ORM

| Scenario | Why raw SQL / Core is better |
|----------|------------------------------|
| Ad‑hoc analytics / data science | ORMs obscure GROUP BY, CTEs, window functions |
| Heavy batch ETL | `COPY`, `INSERT … SELECT` need Core or SQL strings |
| Ultra‑high‑perf hot code paths | Python object overhead dominates |

---

## 14  Bridging Back to Theory

* Relational algebra ↔ SQL ↔ SQLAlchemy Core → ORM  
  (Make students translate the same query across the three.)
* Emphasise that **good schema design** remains essential – the ORM cannot fix poor normalization.

---

## 15  Further Reading & Exercises

1. **Reading** – SQLAlchemy 2.0 tutorial sections: ORM Quick Start, Relationship Patterns.  
2. **Hands‑on** – extend the e‑commerce schema with a `Coupon` model and demonstrate a one‑to‑many relationship (`Coupon.redemptions`).  
3. **Challenge** – profile an *N + 1* situation and refactor with `selectinload`.

---

## 16  Key Takeaways

* ORMs like SQLAlchemy improve productivity **without discarding SQL**.  
* Understand the **Session lifecycle**, **lazy loading**, and **transactions** to avoid surprises.  
* Use the **right abstraction level** for every problem: ORM ↔ Core ↔ raw SQL.
