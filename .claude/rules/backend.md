# Coding Rules for Python Backend Applications

---

## 1. Project Structure and Organization

### 1.1 Top-Level Layout
- Keep the project root flat: `main.py` (or `app.py`) as the entry point, `config.py` at root for configuration, `requirements.txt`, `env-example`, and a `src/` package for all application code.
- Place the configuration module (`config.py`) at the repository root, NOT inside `src/`.
- Place any one-off bootstrap or maintenance scripts (e.g., `populate_db.py`) at the repository root, NOT inside `src/`.
- Ship an `env-example` file at the repository root listing every required environment variable with empty values. Never commit a real `.env`.

### 1.2 The `src/` Package
- Organize `src/` by *type* at the top level, NOT by feature. Use subpackages such as `src/models/`, `src/services/`, `src/handlers/` (or `src/views/`), `src/schemas/`, `src/parsers/`. Place cross-cutting modules at the package root: `src/errors.py`, `src/messages.py`.
- Within each type-directory, use one file per entity or feature (e.g., `src/models/user.py`, `src/models/budget.py`, `src/services/user.py`).
- Place reusable, cross-cutting code (auth helpers, shared mixins, queue clients) under `src/common/`.

### 1.3 File Naming
- Snake_case for all Python module names.
- Match file name to its primary entity or feature name (e.g., the `Budget` model lives in `src/models/budget.py`, the budget service in `src/services/budget.py`).
- Test files mirror the structure of the code under test (e.g., `tests/test_services/test_budget.py` for `src/services/budget.py`), prefixed with `test_`.

---

## 2. Application Entry Point and Factory

### 2.1 Entry Point
- The repository-root entry module (`main.py` or `app.py`) must be minimal: load config, build the app via a factory, and run it.
- For async applications, wrap the entry coroutine and dispatch it via `asyncio.run(main())` inside the `if __name__ == '__main__':` guard.

### 2.2 Application Factory
- Construct the application via a `create_app(config)` (or `create_<thing>(config)`) factory function exposed from `src/__init__.py` or `src/<thing>.py`. The factory must accept the config object as an argument — never read configuration inside the factory directly.
- For frameworks with a router/dispatcher concept (web blueprints, bot dispatchers, queue consumers), define a parallel factory (e.g., `create_dispatcher()`) that wires up routers/handlers and returns the configured object.
- Register all route groups, blueprints, or handler routers inside the factory. Do not perform side-effectful registration at module import time.

### 2.3 Router/Handler Grouping
- Group routes/handlers per feature into their own module under `src/handlers/<feature>.py` (or `src/views/<feature>.py`).
- Instantiate the router with `name=__name__` (or the framework's equivalent) so the module path becomes the router identifier.
- Export one router per file; import and register them from the factory.

---

## 3. Configuration Management

### 3.1 Library
- Read environment variables with `environs` (`from environs import Env`). Call `env.read_env()` once at module top.

### 3.2 Config Object
- Define a single `Config` class decorated with `@dataclass` listing every config field with its precise type annotation (`str`, `int`, `bool`, etc.).
- Instantiate the dataclass once at module bottom and export it: `config = Config(KEY=env.str('KEY'), ...)`. Importers use `from config import config`.
- Use `env.str`, `env.int`, `env.bool`, etc. — never call `os.environ.get` directly.

### 3.3 Naming
- Use ALL_CAPS for config field names.
- Prefix project-specific env vars with the project name (e.g., `MYAPP_DATABASE_URL`) to avoid collisions; unprefixed names are acceptable only for unambiguous, broadly understood keys (e.g., `DATABASE_URL`).

### 3.4 Test Configuration
- For tests, define a separate `TestConfig` class (in `tests/conftest.py` or a dedicated test config module) with hardcoded values pointing at the test database, test Redis instance, etc. Do not call `env.read_env()` for tests.

---

## 4. Data Models (SQLAlchemy)

### 4.1 Library and Version
- Use SQLAlchemy 2.0+ ORM style with `DeclarativeBase`, `Mapped`, and `mapped_column`. Do not use legacy `Column`-based declarations.

### 4.2 Async by Default
- Prefer the async SQLAlchemy engine: `create_async_engine` + `async_sessionmaker(engine, expire_on_commit=False)`. Use the sync engine only when an external constraint (e.g., framework integration) requires it.

### 4.3 Base Class and Session
- Define a single `BaseModel(DeclarativeBase)` in `src/models/base.py` (or `src/common/models.py`). Document it with a one-line docstring (e.g., `"""Base entity."""`).
- Create the engine and `Session` factory in the same module that defines the base, exporting both.

### 4.4 One Model Per File
- Place each model class in its own file under `src/models/` (e.g., `src/models/user.py`, `src/models/budget.py`). Do NOT group multiple unrelated models in a single file.
- Each model class must include a one-line docstring describing the entity (e.g., `"""Telegram user entity."""`).

### 4.5 Column Conventions
- Always declare a surrogate integer primary key: `id: Mapped[int] = mapped_column(primary_key=True)`.
- Specify explicit length for string columns: `String(64)` for short identifiers, `String(2048)` for URLs, `Text` for unbounded text.
- Mark optional columns with `Mapped[Optional[T]]` (or `Mapped[T | None]` on Python 3.10+).
- Foreign-key columns must always have `index=True`.
- Unique business-key columns (usernames, slugs, tickers, external IDs) must have `unique=True` AND `index=True`.
- Use server-side defaults (`server_default=text('false')`, `server_default=func.now()`) over Python-side defaults whenever the database can express the default.

### 4.6 Timestamps Mixin
- For any entity that needs created/updated tracking, mix in a `TimestampedMixin` that defines `created_at` and `updated_at` via `server_default=func.now()` and `server_onupdate=func.now()`. Place the mixin in `src/common/models.py`.
- Apply it as the first base in the inheritance list: `class User(TimestampedMixin, BaseModel): ...`.

### 4.7 Enumerations
- Use `enum.StrEnum` (Python 3.11+) for string-valued enumerations stored in the database. Document the enum class with a one-line docstring (e.g., `"""Budget type choices."""`).
- Store the value as a `String` column sized to the longest member; validate the enum at the application boundary (parser/schema), not in the column type.

### 4.8 Relationships
- Declare relationships with `Mapped[list['Other']] = relationship(back_populates='this')` on the "one" side and `Mapped['Other'] = relationship(back_populates='others')` on the "many" side.
- Always set `back_populates` on both sides — do NOT use the single-sided `backref` shorthand.

---

## 5. Service Layer

### 5.1 Functions, Not Classes
- Write service code as module-level functions, not as classes or `*Service` objects. Group related functions in `src/services/<feature>.py`.

### 5.2 Session Ownership
- A service function manages its own database session via the configured `Session` factory imported from `src/models/base.py`. Do NOT require callers to pass a session in.
- For async services: `async with Session() as session: async with session.begin(): ...` for write operations; `async with Session() as session: return await session.scalar(query)` for reads.

### 5.3 Visibility
- Functions intended for outside callers are public (no underscore prefix). Internal helpers used only within the same service module are prefixed with `_` (e.g., `_create_user`).
- Compose public service functions out of smaller private helpers when an operation has multiple steps.

### 5.4 Idempotent and Composite Operations
- When an operation is naturally idempotent ("create if not exists", "upsert"), expose it as a single public function returning a boolean or the resulting entity. Example: `create_user_if_not_exist(user_id) -> bool`.

### 5.5 Return Types
- Annotate all service return types. Use `Model | None` (or `Optional[Model]`) for single-entity lookups that may miss.
- Use `Sequence[Model]` for list returns; resolve with `.scalars(...).all()`.

### 5.6 Query Style
- Build queries with `select(Model).where(...)` and execute via `session.scalar(query)`, `session.scalars(query).all()`, or the async equivalents. Do NOT use the legacy `Query` API.
- For eager loading, attach `.options(joinedload(...))` to the `select` statement.

---

## 6. Request / Response Schemas

### 6.1 Schemas Are Not Models
- Define request and response schemas in a separate module from ORM models. Use `src/schemas/<feature>.py` or `src/<feature>/schemas.py` per the project's chosen structure.
- Schemas describe the wire format; models describe the storage format. Never reuse one for the other.

### 6.2 Naming
- Suffix input schemas with `In`, output schemas with `Out`, and partial-update schemas with `UpdateIn`. Examples: `UserIn`, `UserOut`, `UserUpdateIn`.
- Use `*Preview` for stripped-down list-view schemas (e.g., `CoinPreview` exposes only `name`, `ticker`, `web_slug`, `image`).

### 6.3 Validation
- Mark required input fields with `required=True`.
- Apply length and format validators at the field level (`Length(0, 64)`, `Email()`, `URL()`).
- For multi-field rules, use a schema-level validator (e.g., `@validates_schema`) that raises `ValidationError` with a descriptive message.

### 6.4 Command/Argument Parsing
- For text-command interfaces (bots, CLIs), wrap `argparse.ArgumentParser` in a subclass that overrides `error()` to raise a domain-specific exception instead of writing to stderr and exiting. Place the base parser in `src/parsers/base.py` and one parser instance per command in `src/parsers/<command>.py`.

---

## 7. Routing and Views / Handlers

### 7.1 Thin Controllers
- View/handler functions are thin: parse input, call services, shape the response. They must NOT contain business logic or direct queries.

### 7.2 HTTP Status Codes
- Always import status codes from the stdlib `http.HTTPStatus` enum. Never write raw integer literals (`200`, `404`) in view code.
- Pass status codes explicitly to the response/output decorator (`status_code=HTTPStatus.CREATED`).

### 7.3 Error Translation
- Wrap write operations in `try / except` for known domain errors (e.g., `IntegrityError`). On the except branch: roll back the session, then `abort` with the appropriate `HTTPStatus` and a human-readable message.
- For not-found lookups, return the result or `abort(HTTPStatus.NOT_FOUND)` inline: `return result or abort(HTTPStatus.NOT_FOUND)`.

### 7.4 View Function Naming
- Suffix view/handler function names with `_view` or `_handler` (e.g., `create_user_view`, `start_handler`) to distinguish them from service functions.

### 7.5 Authentication Decorators
- Apply authentication decorators above input/output decorators on view functions. For routes accepting multiple auth schemes, compose them via the framework's multi-auth helper.

---

## 8. Custom Exceptions

### 8.1 Module Location
- Define all custom exception classes in `src/errors.py`.

### 8.2 Documentation
- Each custom exception class must include a one-line docstring stating where or how it is used. Example:
  ```python
  class TelegramCommandArgumentParserError(Exception):
      """Exception used in parsers module."""
      pass
  ```

### 8.3 Catch at the Boundary
- Catch custom exceptions at the boundary (view/handler) and translate them into user-facing responses. Do NOT let them propagate to the framework.

---

## 9. User-Facing Text and Messages

### 9.1 Centralization
- Centralize all user-facing strings (welcome messages, help text, success/error responses) in `src/messages.py` as module-level UPPER_CASE constants.
- Never inline user-facing strings in handlers, services, or schemas. The only inline text allowed is exception messages intended for developers and validation error messages that include dynamic field names.

### 9.2 Multi-Line Messages
- Use triple-quoted strings for multi-line messages. Keep the leading `"""` flush with the assignment to avoid leading indentation in the rendered text.

---

## 10. Authentication

### 10.1 Module Location
- Place authentication helpers under `src/common/auth.py` for shared primitives (token generation, token verification). Feature-specific verification logic (e.g., password check for the users feature) lives in `src/<feature>/auth.py` or `src/users/auth.py`.

### 10.2 Token Generation
- Implement token generation as a small pure function: `generate_jwt_token(payload: dict, secret_key: str, expires_in: int = 3600) -> str`. The function adds the `exp` claim with `datetime.now(UTC) + timedelta(seconds=expires_in)` and encodes with `HS256`.
- Never hardcode the algorithm at multiple call sites — keep it inside the helper.

### 10.3 Token Verification
- Verification helpers must `abort(HTTPStatus.UNAUTHORIZED)` for: missing token, blacklisted token, decode failure (`jwt.PyJWTError`). Return the decoded payload on success.

### 10.4 Token Revocation
- Store revoked tokens in Redis with a TTL equal to the remaining time until the token's natural expiry. Key format: `auth_token:<token>`. Verifiers MUST check this blocklist before decoding.
- Compute remaining TTL from the token's `exp` claim minus current epoch time; skip insertion when TTL is non-positive.

### 10.5 Password Storage
- Hash passwords with the framework's password-hashing helper (e.g., `werkzeug.security.generate_password_hash`). Store the hash in a `password_hash` column. Never store plaintext.
- On password update, verify the old password via `check_password_hash` before re-hashing the new value.

---

## 11. Asynchronous Tasks and Messaging

### 11.1 Queue
- For background work (email delivery, slow third-party calls), publish to RabbitMQ via `pika`. The producer lives in `src/common/<task>.py` (e.g., `src/common/email.py`) and exposes a single function that serializes the event with `json.dumps` and publishes to a named, durable queue.

### 11.2 Connection Lifecycle
- Open the RabbitMQ connection inside the app factory and attach it to the app context (e.g., `app.rabbitmq = pika.BlockingConnection(...)`). Gate connection setup on the presence of the config key so test environments without RabbitMQ still bring the app up:
  ```python
  if 'RABBITMQ_URL' in app.config:
      app.rabbitmq = pika.BlockingConnection(pika.URLParameters(app.config['RABBITMQ_URL']))
  else:
      app.rabbitmq = None
  ```

### 11.3 Consumer Process
- Implement consumers as separate processes under `src/<topic>/consumer.py` with a `main()` entry point that builds the app via the factory and starts `channel.start_consuming()` inside `app.app_context()`.
- On message handling: ack on success, nack and `stop_consuming()` then re-raise on failure. Do not silently swallow exceptions.

### 11.4 Queue Declaration
- Declare queues as `durable=True` on both producer and consumer sides. Both sides must call `queue_declare` with identical arguments — declaration is idempotent and ensures the queue exists.

---

## 12. Database Migrations and Seeding

### 12.1 Migrations
- Use Alembic for schema migrations (via the framework's integration when one exists). Commit the `migrations/` directory, including `env.py`, `script.py.mako`, and `versions/`.

### 12.2 Bootstrap Script
- Provide a `populate_db.py` script at the repository root that drops and recreates all tables for local development:
  ```python
  async def main():
      async with engine.begin() as conn:
          await conn.run_sync(BaseModel.metadata.drop_all)
          await conn.run_sync(BaseModel.metadata.create_all)
  ```
- This script is for local development only. Production schema changes must go through Alembic.

---

## 13. Python Conventions

### 13.1 Type Hints
- Annotate every function signature: parameters and return type. Use the modern union syntax `X | None` on Python 3.10+ in preference to `Optional[X]`.

### 13.2 Imports
- Group imports in three blocks separated by blank lines: stdlib, third-party, first-party (`src.*`, `config`, `tests.*`). Within a block, alphabetize.
- Use absolute imports (`from src.services.user import ...`) — never relative imports across packages.

### 13.3 String Style
- Use single quotes for short string literals; reserve double quotes for strings that contain a single quote. Use triple double quotes for docstrings.

### 13.4 Docstrings
- Every module-level public class and function intended to be called from outside its module must have at least a one-line docstring.
- Docstrings describe purpose, not implementation. Keep them short and declarative.

### 13.5 Dataclasses
- Prefer `@dataclass` over hand-written `__init__` for simple data containers (config, DTOs, command-parsing results). Do not use dataclasses for ORM models or schemas — those have their own base classes.

### 13.6 Async Style
- Functions performing I/O in an async context must be `async def`. Compose async work with `async with` for resource management and `await` for results — never mix blocking calls into the async path.
- Use an IIFE pattern only when a sync API demands a sync callback; otherwise propagate `async` up to the entry point.
