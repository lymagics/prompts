# Backend Engineering Rules

These rules govern how you write backend code in this repository. Follow them exactly.
When a general convention conflicts with a rule here, the rule here wins.

The goals are **strict layering**, **a framework-free core**, and **persistence and HTTP
concerns kept at the edges**. Every file has one job. Dependencies point inward.

---

## 1. Architecture

The project is a **modular monolith**. Each business capability is a self-contained
**module** under `src/`. Shared infrastructure lives in `src/shared/`.

Each module follows an **onion / ports-and-adapters** layout. Dependencies point inward
only — an inner layer must never import an outer layer.

```
api  →  service  →  repository (port)  →  domain
                          │
                    repository (adapter) → models (ORM)
```

- **domain** is the center. It depends on nothing but the standard library.
- **service** orchestrates use cases. It depends only on `domain` and repository *ports*. It does not manage transactions.
- **repository** is a *port* (abstract) paired with an *adapter* (concrete) that talks to the database through `models`.
- **api** is the outermost layer. It wires the request together and translates between HTTP and the domain.

**Hard rule:** `domain.py` never imports SQLAlchemy, `apiquart`, or anything under `api/`.

### Cross-module boundaries

A module may import from `shared` and from another module's `domain` or `service` only.
Never reach into another module's `repository`, `models`, `unit_of_work`, or `api`.

---

## 2. Module layout

```
src/
    shared/
        auth.py              # authentication helpers / decorators
        models.py            # SQLAlchemy DeclarativeBase only
        unit_of_work.py      # abstract UnitOfWork (port)
    <module>/
        api/
            routes.py        # HTTP handlers; composition root for the request
            schemas.py       # request/response DTOs (validation + serialization)
        domain.py            # business entities, framework-free
        models.py            # SQLAlchemy ORM mapping only
        repository.py        # repository port + SQLAlchemy adapter
        unit_of_work.py      # SQLAlchemy UnitOfWork implementation
        service.py           # use-case orchestration, owns the transaction
        exceptions.py        # module-specific domain exceptions
```

---

## 3. Domain (`domain.py`)

Domain entities are pure Python. They model the business, not the database or the wire.

- **Framework-free.** No SQLAlchemy, no schema library, no HTTP types.
- **Immutable.** Use `@dataclass(frozen=True, slots=True)` (or hand-written immutable objects).
- Entities may carry behavior that enforces their **own** invariants. Logic that spans
  multiple entities or external systems belongs in the service, not here.
- Constructors only assign; no I/O, no side effects.

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class User:
    id: int
    username: str
```

---

## 4. Persistence models (`models.py`)

ORM mapping **only**. No validation, no business logic, no methods beyond the mapping.

- Inherit from the shared `BaseModel`.
- Prefix ORM classes with `Sqla` so they never get confused with domain entities
  (`SqlaUser` vs domain `User`).
- These classes never leave the repository. Nothing above the repository layer may
  import or reference a `Sqla*` type.

```python
from sqlalchemy.orm import Mapped, mapped_column

from shared.models import BaseModel


class SqlaUser(BaseModel):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(unique=True)
```

Shared base:

```python
from sqlalchemy.orm import DeclarativeBase


class BaseModel(DeclarativeBase):
    pass
```

---

## 5. Repository (`repository.py`)

The repository is the boundary between the domain and persistence. Define an **abstract
port** and a **SQLAlchemy adapter**.

Rules:

- The port takes a `UnitOfWork` and declares async methods.
- Methods accept and return **domain entities**, never ORM models. Mapping `Sqla* ↔ domain`
  happens here and nowhere else.
- No business logic — only retrieval, persistence, and translation.
- The concrete adapter inherits the **port**, not `abc.ABC`.

```python
import abc
from typing import Optional

from sqlalchemy import select

from shared.unit_of_work import UnitOfWork
from .domain import User
from .models import SqlaUser


class UserRepository(abc.ABC):
    def __init__(self, unit_of_work: UnitOfWork) -> None:
        self.uow = unit_of_work

    @abc.abstractmethod
    async def get(self, user_id: int) -> Optional[User]:
        raise NotImplementedError

    @abc.abstractmethod
    async def add(self, user: User) -> None:
        raise NotImplementedError


class SqlaUserRepository(UserRepository):
    async def get(self, user_id: int) -> Optional[User]:
        stmt = select(SqlaUser).where(SqlaUser.id == user_id)
        row = (await self.uow.session.execute(stmt)).scalar_one_or_none()
        if row is None:
            return None
        return User(id=row.id, username=row.username)

    async def add(self, user: User) -> None:
        self.uow.session.add(SqlaUser(id=user.id, username=user.username))
```

> Note: the SQLAlchemy adapter assumes a SQLAlchemy-backed unit of work (it uses
> `uow.session`). Under `mypy --strict`, either type the adapter's `uow` as the concrete
> `SqlaUnitOfWork`, or expose `session` through a `Protocol`.

---

## 6. Unit of Work (`unit_of_work.py`)

The unit of work owns the transaction boundary. The abstract port lives in `shared`; each
module provides a concrete implementation.

Rules:

- The engine and session factory are created **once at application startup** and reused.
  **Never create an engine per request** — an engine owns a connection pool.
- The UoW receives the **session factory**, not a database URL.
- It is an async context manager. The session is opened in `__aenter__` and closed in `__aexit__`.
- `__aexit__` rolls back unconditionally. After an explicit `commit()`, the rollback is a
  no-op; on any exception or early exit, it cleans up.

Abstract port (`shared/unit_of_work.py`):

```python
import abc


class UnitOfWork(abc.ABC):
    @abc.abstractmethod
    async def commit(self) -> bool:
        raise NotImplementedError

    @abc.abstractmethod
    async def rollback(self) -> bool:
        raise NotImplementedError
```

SQLAlchemy implementation (`<module>/unit_of_work.py`):

```python
from typing import Self

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

from shared.unit_of_work import UnitOfWork


class SqlaUnitOfWork(UnitOfWork):
    def __init__(self, session_factory: async_sessionmaker[AsyncSession]) -> None:
        self._session_factory = session_factory

    async def __aenter__(self) -> Self:
        self.session = self._session_factory()
        return self

    async def __aexit__(self, *exc) -> None:
        await self.rollback()
        await self.session.close()

    async def commit(self) -> bool:
        await self.session.commit()
        return True

    async def rollback(self) -> bool:
        await self.session.rollback()
        return True
```

The session factory is built at startup and stored on the app:

```python
engine = create_async_engine(app.config["DATABASE_URL"])
app.extensions["session_factory"] = async_sessionmaker(engine, expire_on_commit=False)
```

---

## 7. Service (`service.py`)

The service is the use-case layer. It enforces business rules and coordinates repositories.
It does **not** manage transactions.

Rules:

- Depends on repository **ports** only. Never on the `UnitOfWork`, concrete adapters,
  SQLAlchemy, or HTTP.
- **Never commits or rolls back.** The transaction is the caller's responsibility — the
  route opens the `UnitOfWork` scope and commits after a successful write (see §9).
- Raises **domain exceptions** from `exceptions.py`. Returns domain entities.
- Constructors only assign dependencies.

```python
from .domain import User
from .repository import UserRepository
from .exceptions import UserNotFoundError


class UserService:
    def __init__(self, user_repository: UserRepository) -> None:
        self._users = user_repository

    async def get_user(self, user_id: int) -> User:
        user = await self._users.get(user_id)
        if user is None:
            raise UserNotFoundError(f"User with {user_id=} not found")
        return user

    async def register_user(self, user: User) -> User:
        await self._users.add(user)
        return user
```

---

## 8. Exceptions (`exceptions.py`)

- Each module defines a **base exception**; specific exceptions inherit it.
- Services and the domain raise these. They are plain exceptions — no HTTP status codes,
  no framework coupling.
- Only the **api layer** catches them and maps them to HTTP responses.

```python
class UserError(Exception):
    """Base error for the user module."""


class UserNotFoundError(UserError):
    pass
```

---

## 9. API layer (`api/`)

### `schemas.py`

Request and response DTOs as **pydantic** models (apiquart uses pydantic). They validate
and serialize at the HTTP boundary **only**.

- Schemas never leak below the api layer. The service, repository, and domain must not
  import them.
- No business logic in schemas.

```python
from pydantic import BaseModel


class UserIn(BaseModel):
    username: str


class UserOut(BaseModel):
    id: int
    username: str
```

### `routes.py`

Thin HTTP handlers. The route is the **composition root for the request**: it opens the
unit of work, builds the adapter repository, builds the service, and translates domain
exceptions into HTTP errors. The route **owns the transaction**.

Rules:

- No business logic in routes — delegate to the service.
- Open the UoW with `async with` so rollback/cleanup is automatic.
- After a successful **mutating** use case, call `await uow.commit()` inside the UoW scope.
  Read-only routes never commit.
- Catch **domain exceptions** here and convert them with `abort`. Do this nowhere else.
- Return domain entities; let the output schema serialize them.

Read (no commit):

```python
from apiquart import APIBlueprint, HTTPStatus, abort
from quart import current_app

from .schemas import UserOut
from ..service import UserService
from ..repository import SqlaUserRepository
from ..unit_of_work import SqlaUnitOfWork
from ..exceptions import UserNotFoundError

bp = APIBlueprint(__name__, tags=["Users"])


@bp.get("/users/<int:user_id>")
@bp.output(UserOut)
async def get_user(user_id: int):
    session_factory = current_app.extensions["session_factory"]
    try:
      async with SqlaUnitOfWork(session_factory) as uow:
          repository = SqlaUserRepository(uow)
          service = UserService(repository)
          return await service.get_user(user_id)
    except UserNotFoundError as error:
        abort(HTTPStatus.NOT_FOUND, message=str(error))
```

Write (route commits after the service call succeeds):

```python
@bp.post("/users")
@bp.input(UserIn)
@bp.output(UserOut, status_code=HTTPStatus.CREATED)
async def create_user(data: dict):
    session_factory = current_app.extensions["session_factory"]
    async with SqlaUnitOfWork(session_factory) as uow:
        repository = SqlaUserRepository(uow)
        service = UserService(repository)
        user = await service.register_user(User(**data))
        await uow.commit()
        return user
```

> If the wiring repeats across many routes, extract a small per-module factory
> (e.g. `build_user_service(uow) -> UserService`) to assemble the service. Keep the UoW
> scope in the route.

---

## 10. Shared module (`shared/`)

- `auth.py` — authentication helpers and decorators.
- `models.py` — the SQLAlchemy `DeclarativeBase` (`BaseModel`) only.
- `unit_of_work.py` — the abstract `UnitOfWork` port.

`shared` must not depend on any module.

---

## 11. Request lifecycle (mental model)

1. Request hits the **route**.
2. The input schema validates the incoming data.
3. The route opens the **UnitOfWork**, builds the **adapter repository**, builds the **service**.
4. The **service** executes the use case against repository **ports**, enforces business
   rules, and raises domain exceptions. It does not commit.
5. The **repository** maps `Sqla* ↔ domain`.
6. The **route** commits the UnitOfWork after a successful write, catches domain exceptions
   → HTTP errors, and the output schema serializes the returned domain entity.

---

## 12. Dependency rules (quick reference)

| File              | May import                                                              |
| ----------------- | ----------------------------------------------------------------------- |
| `domain.py`       | standard library only                                                   |
| `models.py`       | `shared.models`, SQLAlchemy                                             |
| `repository.py`   | `shared.unit_of_work`, `.domain`, `.models`, SQLAlchemy                 |
| `unit_of_work.py` | `shared.unit_of_work`, SQLAlchemy                                       |
| `exceptions.py`   | standard library only                                                   |
| `service.py`      | `.domain`, `.repository` (ports), `.exceptions`                          |
| `api/schemas.py`  | pydantic (apiquart's schema layer)                                      |
| `api/routes.py`   | the whole module + `shared` + apiquart                                  |

---

## 13. Code quality

- **Type everything.** Code must pass `mypy --strict`. No bare `Any`, no untyped defs.
- **Async all the way down.** No blocking/sync database calls in the request path.
- **Immutable domain objects.** Prefer frozen dataclasses.
- **Constructors only assign** dependencies — no logic, no I/O in `__init__`.
- **Composition over inheritance.** Inheritance is reserved for ports → adapters and
  declared base classes.
- **One public responsibility per class.**
- **Name by role:** domain `User`, ORM `SqlaUser`, port `UserRepository`, adapter `SqlaUserRepository`.
- `Optional` is acceptable at the repository boundary for "not found"; in the **service**,
  prefer raising a domain exception over returning `None`.
- Keep functions small and intention-revealing.

---

## 14. Anti-patterns — reject these

- Returning a `Sqla*` ORM model from a service or route.
- Business logic or validation in `models.py` or `schemas.py`.
- Using a SQLAlchemy `session` anywhere above the repository.
- Creating an engine (or session factory) per request.
- `domain.py` importing a framework.
- Catching domain exceptions outside the api layer.
- Importing another module's `repository`, `models`, or `api`.
- A service that imports or commits the `UnitOfWork`.
- A write route that doesn't commit the unit of work (the change is silently rolled back on exit).
