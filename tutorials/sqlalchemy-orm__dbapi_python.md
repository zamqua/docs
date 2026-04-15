+++
title = "SQLAlchemy ORM"
author = ["Mohammed Azam"]
date = 2026-04-13T16:21:00-05:00
tags = ["dbapi", "learning", "python"]
draft = false
+++

## Overview {#overview}

Source of tutorial: <https://docs.sqlalchemy.org/en/20/tutorial/dbapi_transactions.html>


### Terms {#terms}

-   Engine: The "source" of connectivity. It handles the actual talking to
    your db file or server.
-   Declarative Base
-   Session: Your "workspace". This is where you create, delete, or update
    objects before "committing" them to the database.

## Working with Transactions and the DBAPI {#working-with-transactions-and-the-dbapi}

1.  create an engine (sqlite db, use pysqlite api, store in memory, print sql
    statements)
2.  create a connection using python context manager ('with' statement), conn
    is within this context. Context Manager in Python handles the "setup" and
    "teardown" of a resource antomatically.
3.  execute the statement.. call textual select statement.


### Create an engine and connecton {#create-an-engine-and-connecton}


#### Create an engine and run hello-world {#create-an-engine-and-run-hello-world}

-   once conn is release, control goes out of 'with' statement, rollback is
    automatically called, unless we commit our changes.
-   **autocommit** is available for special case.

<!--listend-->

```python
from sqlalchemy import create_engine, text

engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)
with engine.connect() as conn:
    result = conn.execute(text("select 'hello, world'"))
    print(result.all())
```

:RESULTS:

```text
2026-04-13 16:50:26,619 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2026-04-13 16:50:26,619 INFO sqlalchemy.engine.Engine select 'hello, world'
2026-04-13 16:50:26,619 INFO sqlalchemy.engine.Engine [generated in 0.00009s] ()
[('hello, world',)]
2026-04-13 16:50:26,619 INFO sqlalchemy.engine.Engine ROLLBACK
```


#### Create an engine, make change and commit {#create-an-engine-make-change-and-commit}

```python
from sqlalchemy import create_engine, text
engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)
with engine.connect() as conn:
    conn.execute(text("CREATE TABLE some_table (x int, y int)"))
    conn.execute(text("INSERT INTO some_table (x, y) VALUES (:x, :y)"), [{"x":1, "y":1}, {"x":2, "y":4}])
    result = conn.execute(text("select * from some_table"))
    print(result.all())
    conn.commit()
```


#### Create an engine - make connect block a transactions block upfront. Commit on success. {#create-an-engine-make-connect-block-a-transactions-block-upfront-dot-commit-on-success-dot}

```python
from sqlalchemy import create_engine, text
engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)
with engine.begin() as conn:
    conn.execute(text("CREATE TABLE some_table (x int, y int)"))
    conn.execute(text("INSERT INTO some_table (x, y) VALUES (:x, :y)"), [{"x":1, "y":1}, {"x":2, "y":4}])
    result = conn.execute(text("select * from some_table"))
    print(result.all())
    print(type(result.all()))
```


### Fetching Rows {#fetching-rows}

```python
from sqlalchemy import create_engine, text
engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)
with engine.connect() as conn:
    conn.execute(text("CREATE TABLE some_table (x int, y int)"))
    conn.execute(text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
                 [{"x":1, "y":1}, {"x":2, "y":4}]
     )
    result = conn.execute(text("select * from some_table"))
    print(type(result))

    # Attribute Name
    #for row in result:
    #    print(type(row))
    #    print(f"x: {row.x} y: {row.y}")

    # Tuple Assignment
    #for x, y in result:
    #    print(f"x: {x} y: {y}")

    # Integer Index
    #for row in result:
    #    x = row[0]
    #    print(type(x))
    #    print(f"x: {x}")

    # Mapping Access
    for dict_row in result.mappings():
        print(type(dict_row))
        print(f"x: {dict_row['x']} y: {dict_row['y']}")
```

In C++, an iterator provides a way to access the elements of a container
sequentially without exposing the underlying representation. A Cursor does the
exact same for a database result set.


### Sending Parameters {#sending-parameters}

```python
from sqlalchemy import create_engine, text
engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)
with engine.connect() as conn:
    conn.execute(text("CREATE TABLE some_table (x int, y int)"))
    conn.execute(text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
                 [{"x":1, "y":1}, {"x":2, "y":4}]
     )
    # Passing bound parameters
    result = conn.execute(text("select * from some_table where y > :y"), {"y": 2})
    for row in result:
        print(f"x: {row.x} y: {row.y}")

```


## Executing with an ORM session {#executing-with-an-orm-session}

ORM: Object-Relational Mapping

| Python Side (Objects)    | Database Side (Relational) |
|--------------------------|----------------------------|
| Class (class User)       | Table (Users)              |
| Instance (user_obj)      | Row (A single record)      |
| Attribute (user_obj.name | Column (The name field)    |


### Using Session as connection {#using-session-as-connection}

```python
from sqlalchemy.orm import Session
from sqlalchemy import create_engine, text

engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)
stmt = text("select x, y from some_table where y>:y order by x, y")
with Session(engine) as session:
    session.execute(text("CREATE TABLE some_table (x int, y int)"))
    session.execute(text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
                    [{"x":1, "y":1}, {"x":2, "y":4}, {"x":10, "y":20}])
    session.execute(text("UPDATE some_table SET y=:y WHERE x=:x"),
                    [{"x":2, "y":25}, {"x":1, "y":11}])
    result = session.execute(stmt, {"y": 6})
    for row in result:
        print(f"x: {row.x} y: {row.y}")
    session.commit()
```


### Using Session - framing out a begin/commit/rollback block {#using-session-framing-out-a-begin-commit-rollback-block}

```python
from sqlalchemy.orm import Session
from sqlalchemy import create_engine, text

engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)
stmt = text("select x, y from some_table where y>:y order by x, y")

# this can be written on two separate lines also, with try/except/else.
# inner context calls session.commit(), if there were no exceptions
# outer context calls session.close()
with Session(engine) as session, session.begin():
    session.execute(text("CREATE TABLE some_table (x INT, y INT)"))
    session.execute(text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
                    [{"x":1, "y":1}, {"x":2, "y":"4re"}, {"x":10, "y":'abc'}])
    session.execute(text("UPDATE some_table SET y=:y WHERE x=:x"),
                    [{"x":2, "y":25}, {"x":1, "y":11}])
    result = session.execute(stmt, {"y": 6})
    for row in result:
        print(f"x: {row.x} y: {row.y}")
```


### Using Session - with sessionmaker {#using-session-with-sessionmaker}

The purpose of sessionmake is to provide a factory for Session objects with
a fixed configuration.

```python
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker

# an Engine, which the Session will use for connection
# resources, typically in module scope
# dialect+driver://username:password@host:port/database_name
engine = create_engine("postgresql+psycopg2://appuser:apppass@localhost:5432/appdb", echo=True)

# a sessionmaker(), also in the same scope as the engine
Session = sessionmaker(engine)

stmt = text("select x, y from some_table where y>:y order by x, y")

# we can now construct a Session() without needing to pass the engine each time
with Session() as session, session.begin():
    session.execute(text("CREATE TABLE some_table (x INT, y INT)"))
    session.execute(text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
                    [{"x":1, "y":"a1"}, {"x":2, "y":4}, {"x":10, "y":20}])
    session.execute(text("UPDATE some_table SET y=:y WHERE x=:x"),
                    [{"x":2, "y":25}, {"x":1, "y":11}])
    result = session.execute(stmt, {"y": 6})
    for row in result:
        print(f"x: {row.x} y: {row.y}")
```


### Using Session - with sessionmaker with data model and querying {#using-session-with-sessionmaker-with-data-model-and-querying}

It throw following error if duplicate field exists:

```text
psycopg2.errors.UniqueViolation: duplicate key value violates unique constraint "_x_y_uc"
DETAIL:  Key (x, y)=(1, 2) already exists.
```

```python
from sqlalchemy import create_engine, select, UniqueConstraint
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, sessionmaker

class Base(DeclarativeBase):
    pass

class Point(Base):
    __tablename__ = "point_table"

    # primary_key is required for SQLAlchemy models
    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)

    x: Mapped[int] = mapped_column()
    y: Mapped[int] = mapped_column()

    # This prevents any two rows from having the same x and y combination
    __table_args__ = (UniqueConstraint('x', 'y', name='_x_y_uc'),)

    def __init__(self, x, y):
        self.x = x
        self.y = y

points = [Point(1,13), Point(2,12)]

engine = create_engine("postgresql+psycopg2://appuser:apppass@localhost:5432/appdb", echo=True)
Session = sessionmaker(engine)

# Assuming you have defined 'Point' as we did before
# This command inspects the DB and creates missing tables
Base.metadata.create_all(engine)

with Session() as session, session.begin():
    for point in points:
        session.add(point)

with Session() as session:
    statement = select(Point).filter_by(x='1')    # query for Point objects
    point_objs = session.scalars(statement).all() # list of Point objects
    for obj in point_objs:
        print(f"x: {obj.x} y: {obj.y}")

    statement = select(Point.x)                   # query for individual columns
    rows = session.execute(statement).all()       # list of Row objects
    for row in rows:
        print(f"x: {row.x}")

```
