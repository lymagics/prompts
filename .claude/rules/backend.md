# Coding Rules for Python Backend Applications

---

## 1. Domain Model Rules

### 1.1 The Domain Model Is the Core
- The domain model is the code closest to the business. Make it easy to understand and modify.
- The domain model should have **no dependencies on infrastructure** — no ORM imports, no web framework imports, no I/O. It is pure Python objects.
- Behavior should come first and drive storage requirements. Build the domain model before thinking about persistence.
- Model business processes as **verbs**, not static data models of nouns.

### 1.2 Distinguish Entities from Value Objects
- A **Value Object** is defined entirely by its attributes and is immutable. If you change an attribute, it represents a different object. Implement value objects using `@dataclass(frozen=True)` or `NamedTuple`.
- An **Entity** has attributes that may vary over time but retains a recognizable identity. Define what uniquely identifies an entity (usually a reference or name field).
- For entities, implement `__eq__` based on the identity attribute (e.g., `.reference`), and set `__hash__` based on that same attribute or to `None`.
- For value objects, equality and hash are based on all value attributes. `@dataclass(frozen=True)` provides this for free.

```python
# Value Object
@dataclass(frozen=True)
class OrderLine:
    orderid: str
    sku: str
    qty: int

# Entity
class Batch:
    def __init__(self, ref: str, sku: str, qty: int, eta: Optional[date]):
        self.reference = ref
        self.sku = sku
        self.eta = eta
        self._purchased_quantity = qty
        self._allocations: Set[OrderLine] = set()

    def __eq__(self, other):
        if not isinstance(other, Batch):
            return False
        return other.reference == self.reference

    def __hash__(self):
        return hash(self.reference)
```

### 1.3 Not Everything Has to Be an Object
- Python is a multiparadigm language. Let the "verbs" in your code be functions.
- For every `FooManager`, `BarBuilder`, or `BazFactory`, there's often a more expressive `manage_foo()`, `build_bar()`, or `get_baz()` waiting to happen.
- Use **domain service functions** for operations that don't naturally belong to an entity or value object.

### 1.4 Use Python Magic Methods for Domain Semantics
- Implement `__gt__`, `__lt__`, etc. so domain objects work naturally with `sorted()`, `min()`, `max()`.
- Implement `__eq__` and `__hash__` to express domain equality rules.

### 1.5 Use Domain Exceptions
- Name exceptions in the ubiquitous language of the domain.
- Domain exceptions are part of the domain model, not infrastructure.
- Do **not** use exceptions for control flow when domain events can serve the same purpose.

```python
class OutOfStock(Exception):
    pass
```

### 1.6 Apply SOLID Principles
- Revisit the SOLID principles and heuristics like "has-a versus is-a" and "prefer composition over inheritance."
- Single Responsibility Principle: if you can't describe what your function does without using words like "then" or "and," you might be violating SRP.

---

## 2. Architecture and Dependency Rules

### 2.1 Dependency Inversion Principle (DIP)
- High-level modules (domain) should not depend on low-level modules (infrastructure). Both should depend on abstractions.
- The domain model is the "inside" of your architecture, and dependencies flow inward to it (onion/hexagonal/ports-and-adapters architecture).
- Your ORM should import your model, not the other way around. The domain model stays "pure" and free from infrastructure concerns.

### 2.2 Invert the ORM Dependency
- Do **not** have model classes inherit from ORM base classes (like SQLAlchemy's `declarative_base()` or Django's `models.Model`).
- Instead, define your schema separately and use explicit mapping (e.g., SQLAlchemy's classical mapping) so that the ORM imports the model.
- This gives persistence ignorance: the domain model doesn't need to know anything about how data is loaded or persisted.

```python
# WRONG: Model depends on ORM
class OrderLine(Base):
    id = Column(Integer, primary_key=True)
    sku = Column(String(250))

# RIGHT: ORM depends on Model
# model.py - pure Python
@dataclass(frozen=True)
class OrderLine:
    orderid: str
    sku: str
    qty: int

# orm.py - maps model to tables
order_lines = Table('order_lines', metadata,
    Column('id', Integer, primary_key=True, autoincrement=True),
    Column('sku', String(255)),
    Column('qty', Integer, nullable=False),
    Column('orderid', String(255)),
)
def start_mappers():
    mapper(model.OrderLine, order_lines)
```

### 2.3 Depend on Abstractions
- Service-layer functions should depend on abstract repository types, not concrete ones. Use type hints like `repo: AbstractRepository`.
- This allows tests to inject `FakeRepository` and production code to inject `SqlAlchemyRepository`.

---

## 3. Repository Pattern Rules

### 3.1 Repository as Collection Abstraction
- The Repository pattern is an abstraction over persistent storage, giving the illusion of an in-memory collection of objects.
- The simplest repository has just two methods: `add()` to put a new item in the repository, and `get()` to return a previously added item. Stick rigidly to using these methods for data access in your domain and service layer.
- Do **not** put `.save()` methods on repositories. The Unit of Work handles persistence.
- Keep `.commit()` outside of the repository — it is the caller's responsibility (or the Unit of Work's).

### 3.2 One Aggregate = One Repository
- The only repositories you should have are repositories that return aggregates.
- Repositories are the main place where you enforce the convention that aggregates are the only way into your domain model.

### 3.3 Build Fakes for Testing
- Building fakes for your abstractions is an excellent way to get design feedback: if it's hard to fake, the abstraction is probably too complicated.
- A `FakeRepository` is typically a simple wrapper around a `set` — all methods are one-liners.

```python
class FakeRepository(AbstractRepository):
    def __init__(self, products):
        super().__init__()
        self._products = set(products)

    def _add(self, product):
        self._products.add(product)

    def _get(self, sku):
        return next((p for p in self._products if p.sku == sku), None)
```

### 3.4 Repository Tracks Seen Aggregates
- The repository should track aggregates that pass through it (via a `.seen` set attribute) so that the Unit of Work can collect domain events from them after commit.

---

## 4. Service Layer Rules

### 4.1 Purpose of the Service Layer
- The service layer (also called orchestration layer or use-case layer) defines the entrypoints to your system and captures the primary use cases.
- It handles orchestration: fetching objects from repositories, performing checks, calling domain services, and persisting changes.
- It is **not** the place for business logic — that belongs in the domain model.

### 4.2 Service Layer Functions Follow a Pattern
Typical service-layer functions have these steps:
1. Fetch some objects from the repository (via the UoW).
2. Make checks or assertions about the request against the current state.
3. Call a domain service or aggregate method.
4. If all is well, commit (via the UoW).

### 4.3 Express the Service Layer API in Primitives
- Service layer functions should accept **primitive types** (strings, ints, dates), not domain objects.
- This decouples the service layer's clients (tests, API, CLI) from the domain model.
- Even better: use **Command and Event dataclasses** as the service layer's input interface.

```python
# Good: primitives
def allocate(orderid: str, sku: str, qty: int, uow: AbstractUnitOfWork) -> str:
    ...

# Even better: commands
def allocate(cmd: commands.Allocate, uow: AbstractUnitOfWork) -> str:
    line = OrderLine(cmd.orderid, cmd.sku, cmd.qty)
    ...
```

### 4.4 Keep Entrypoints Thin
- Flask/API endpoints should be thin wrappers: parse JSON, call the service layer / message bus, return HTTP responses. No business logic.
- The responsibilities of a web framework adapter are: per-request session management, parsing parameters, status codes, and JSON responses.

```python
@app.route("/allocate", methods=['POST'])
def allocate_endpoint():
    try:
        cmd = commands.Allocate(
            request.json['orderid'], request.json['sku'], request.json['qty'],
        )
        results = messagebus.handle(cmd, unit_of_work.SqlAlchemyUnitOfWork())
        batchref = results.pop(0)
    except InvalidSku as e:
        return jsonify({'message': str(e)}), 400
    return jsonify({'batchref': batchref}), 201
```

---

## 5. Unit of Work Pattern Rules

### 5.1 UoW as Atomic Operations Abstraction
- The Unit of Work (UoW) is the abstraction over atomic operations. It provides a stable snapshot of the database, a way to persist all changes at once, and a simple API to persistence.
- Implement the UoW as a **context manager** (`__enter__` / `__exit__`).
- The default behavior on exit (without commit) is to **roll back**. This makes the software safe by default.

### 5.2 Require Explicit Commits
- Prefer requiring an explicit `uow.commit()` call. Don't auto-commit on exit.
- The default behavior is to not change anything. There's only one code path that leads to changes: total success and an explicit commit.

```python
class AbstractUnitOfWork(abc.ABC):
    products: AbstractProductRepository

    def __exit__(self, *args):
        self.rollback()

    @abc.abstractmethod
    def commit(self):
        raise NotImplementedError

    @abc.abstractmethod
    def rollback(self):
        raise NotImplementedError
```

### 5.3 UoW Provides Access to Repositories
- The UoW gives access to repositories via attributes (e.g., `uow.products`).
- The service layer has only one dependency: the abstract UoW.

### 5.4 UoW Collects and Publishes Domain Events
- After committing, the UoW collects new events from all aggregates that the repository has seen, and yields them for the message bus to process.
- The UoW should have a `collect_new_events()` method that iterates over `self.products.seen` and pops events from each aggregate.

```python
def collect_new_events(self):
    for product in self.products.seen:
        while product.events:
            yield product.events.pop(0)
```

### 5.5 Use UoW to Group Multiple Operations Atomically
- If deallocate fails, don't call allocate. If allocate fails, don't commit the deallocate. The UoW ensures atomic success or failure.

---

## 6. Aggregate Pattern Rules

### 6.1 Choose the Right Aggregate
- An aggregate is a cluster of associated objects treated as a unit for data changes. It defines and enforces a consistency boundary.
- The aggregate is the boundary where every operation ends in a consistent state.
- Choose aggregates to be as small as possible for performance. You can change your mind over time.

### 6.2 Aggregates Are Entrypoints to the Domain
- Think of aggregates as the "public" classes of your model, and other entities/value objects as "private."
- Modify objects inside the aggregate only by loading the whole aggregate and calling methods on it.

### 6.3 One Aggregate Per Transaction
- Each use case should update a **single aggregate** at a time.
- If you need to modify two aggregates, use domain events to carry information between separate transactions.
- Between two aggregates (or services), accept **eventual consistency**.

### 6.4 Aggregates Record Domain Events
- Aggregates expose a `.events` attribute (a list) that records facts about what has happened using domain event objects.
- Events are raised at the place they occur, using only the language of the domain.

```python
class Product:
    def __init__(self, sku: str, batches: List[Batch], version_number: int = 0):
        self.sku = sku
        self.batches = batches
        self.version_number = version_number
        self.events: List[events.Event] = []

    def allocate(self, line: OrderLine) -> str:
        try:
            batch = next(b for b in sorted(self.batches) if b.can_allocate(line))
            batch.allocate(line)
            self.version_number += 1
            self.events.append(events.Allocated(
                orderid=line.orderid, sku=line.sku, qty=line.qty,
                batchref=batch.reference,
            ))
            return batch.reference
        except StopIteration:
            self.events.append(events.OutOfStock(line.sku))
            return None
```

### 6.5 Use Optimistic Concurrency with Version Numbers
- Add a `version_number` attribute to your aggregate. Increment it on every state change.
- Use database transaction isolation levels or `SELECT FOR UPDATE` to prevent concurrent conflicting updates.
- Handle concurrency conflicts by retrying the failed operation.

---

## 7. Events and Message Bus Rules

### 7.1 Domain Events Are Simple Dataclasses
- Events are value objects — pure data structures with no behavior.
- Name events in past tense using the domain language: `Allocated`, `OutOfStock`, `BatchQuantityChanged`.
- Events have a parent `Event` base class for type hints.

```python
@dataclass
class Allocated(Event):
    orderid: str
    sku: str
    qty: int
    batchref: str
```

### 7.2 Commands Are Imperative
- Commands represent a job the system should perform. Name them with imperative mood: `Allocate`, `CreateBatch`, `ChangeBatchQuantity`.
- A command is sent by one actor to one specific actor, expecting a particular result.
- Commands should have exactly **one handler**. Events can have **multiple handlers**.

### 7.3 Different Error Handling for Commands vs Events
- **Commands fail noisily**: if a command handler raises an exception, it bubbles up to the caller.
- **Events fail independently**: if an event handler raises an exception, log it and continue processing other handlers. Do not let a failed event handler stop the system.

```python
def handle_event(event, queue, uow):
    for handler in EVENT_HANDLERS[type(event)]:
        try:
            handler(event, uow=uow)
            queue.extend(uow.collect_new_events())
        except Exception:
            logger.exception('Exception handling event %s', event)
            continue

def handle_command(command, queue, uow):
    try:
        handler = COMMAND_HANDLERS[type(command)]
        result = handler(command, uow=uow)
        queue.extend(uow.collect_new_events())
        return result
    except Exception:
        logger.exception('Exception handling command %s', command)
        raise
```

### 7.4 The Message Bus Is the Main Entrypoint
- The message bus routes messages (commands and events) to their appropriate handlers.
- Everything in the system can be an event handler. API calls are commands; side effects are events.
- The message bus maintains a queue: after each handler finishes, new events from the UoW are added to the queue and processed in order.

### 7.5 Handlers Are Service Functions
- Handlers receive a command or event and a UoW. They perform the work needed for one use case.
- Each handler opens its own UoW context, ensuring each unit of work is small and atomic.

---

## 8. CQRS Rules

### 8.1 Separate Reads from Writes
- Functions should either modify state or answer questions, but **never both** (Command-Query Separation).
- Write operations return 201/202 with a Location header. Read operations use separate GET endpoints.
- Keep the distinction clear: event handlers modify state; views/queries return read-only data.

### 8.2 Reads Don't Need the Domain Model
- For read-only operations, it's acceptable (and often better) to use raw SQL, simple ORM queries, or denormalized read models.
- The complexity of the domain model exists to enforce rules when changing state. Reads don't need it.
- Don't force reads through repository and domain model abstractions if it makes them awkward.

### 8.3 Consider Separate Read Models
- For high-performance reads, maintain a denormalized read model (a separate table or even a different data store like Redis) updated via event handlers.
- Event handlers are a great way to manage updates to a read model. They also make it easy to change the implementation later.

```python
def add_allocation_to_read_model(event: events.Allocated, uow):
    with uow:
        uow.session.execute(
            'INSERT INTO allocations_view (orderid, sku, batchref)'
            ' VALUES (:orderid, :sku, :batchref)',
            dict(orderid=event.orderid, sku=event.sku, batchref=event.batchref)
        )
        uow.commit()
```

---

## 9. Testing Rules

### 9.1 Aim for a Healthy Test Pyramid
- **Unit tests** (against service layer with fakes): the bulk of your tests. Fast, no I/O.
- **Integration tests** (against real DB): a small number to verify repository, ORM, and UoW.
- **End-to-end tests** (against HTTP API): aim for one per feature — just happy path and one unhappy path.

### 9.2 Test at the Highest Useful Abstraction
- Write the bulk of your tests against the service layer / message bus using `FakeUnitOfWork`.
- Maintain a small core of domain model tests for the most complex business logic. Don't be afraid to delete these if the functionality is later covered at the service layer.
- Every line of code in a test is like glue, holding the system in a particular shape. The more low-level tests you have, the harder it will be to change things.

### 9.3 High Gear vs Low Gear
- **Low gear** (domain model tests): when starting a new project or tackling a gnarly problem, for better feedback and executable documentation.
- **High gear** (service layer tests): for most features and bug fixes, for lower coupling and higher coverage.

### 9.4 Service-Layer Tests Should Use Only Primitives and Services
- Service-layer tests should set up state by calling service-layer functions (e.g., `add_batch`), not by directly instantiating domain objects.
- If you need to do domain-layer stuff directly in service-layer tests, your service layer may be incomplete.

### 9.5 Prefer Fakes Over Mocks
- Use **fakes** (working in-memory implementations) rather than **mocks** (`mock.patch`).
- Mocks couple your tests to implementation details. Fakes test the same interface as production code.
- "Don't mock what you don't own" — build simple abstractions over messy subsystems and fake those instead.
- Patching out dependencies doesn't improve design. Introducing abstractions does.

### 9.6 Error Handling Counts as a Feature
- Structure your application so all errors bubble up to entrypoints and are handled uniformly.
- Test only the happy path E2E, and reserve unit tests for unhappy path edge cases.

---

## 10. Dependency Injection Rules

### 10.1 Use a Bootstrap Script
- Create a `bootstrap.py` that wires up all dependencies: UoW, notifications, event publishers, etc.
- The bootstrap script declares default (production) dependencies but allows overrides for tests.
- The bootstrap script performs initialization (e.g., `orm.start_mappers()`), injects dependencies into handlers, and returns a configured message bus.

```python
def bootstrap(
    start_orm: bool = True,
    uow: AbstractUnitOfWork = SqlAlchemyUnitOfWork(),
    notifications: AbstractNotifications = EmailNotifications(),
    publish: Callable = redis_eventpublisher.publish,
) -> MessageBus:
    if start_orm:
        orm.start_mappers()
    dependencies = {'uow': uow, 'notifications': notifications, 'publish': publish}
    injected_event_handlers = {
        event_type: [inject_dependencies(handler, dependencies) for handler in handlers]
        for event_type, handlers in EVENT_HANDLERS.items()
    }
    injected_command_handlers = {
        command_type: inject_dependencies(handler, dependencies)
        for command_type, handler in COMMAND_HANDLERS.items()
    }
    return MessageBus(uow=uow, event_handlers=injected_event_handlers,
                      command_handlers=injected_command_handlers)
```

### 10.2 Explicit Dependencies Over Implicit Imports
- Prefer declaring dependencies explicitly (as function parameters) over importing them at module level and using `mock.patch` in tests.
- Explicit is better than implicit. Declaring an explicit dependency is an example of the dependency inversion principle.

### 10.3 Use Closures or Partials for DI
- Compose handler functions with their dependencies using `functools.partial`, closures, or callable classes.
- The bootstrap script creates partially-applied handler functions that already have their dependencies injected.

### 10.4 Building Proper Adapters
Follow these steps:
1. Define your API using an ABC (or Protocol).
2. Implement the real thing.
3. Build a fake and use it for unit/service-layer/handler tests.
4. Find a less-fake version for your Docker dev environment (e.g., MailHog for email).
5. Test the less-fake "real" thing with integration tests.

---

## 11. Project Structure Rules

### 11.1 Organize by Architectural Layer
```
src/
└── allocation/
    ├── domain/          # Domain model: entities, value objects, events, commands
    │   ├── model.py
    │   ├── events.py
    │   └── commands.py
    ├── service_layer/   # Use cases, handlers, UoW, message bus
    │   ├── handlers.py
    │   ├── unit_of_work.py
    │   └── messagebus.py
    ├── adapters/        # Secondary adapters: repository, ORM, email, Redis
    │   ├── orm.py
    │   ├── repository.py
    │   ├── notifications.py
    │   └── redis_eventpublisher.py
    ├── entrypoints/     # Primary adapters: Flask API, Redis consumer, CLI
    │   ├── flask_app.py
    │   └── redis_eventconsumer.py
    ├── views.py         # Read-only query functions (CQRS)
    ├── bootstrap.py     # Dependency injection and initialization
    └── config.py        # Environment-based configuration
tests/
├── unit/                # Fast tests against fakes
├── integration/         # Tests against real DB
└── e2e/                 # Tests against HTTP API
```

### 11.2 Install Source as a Package
- Put all application code in a `src/` folder and make it pip-installable with a `setup.py`.
- This simplifies imports and keeps tests separate from source.

### 11.3 Configuration via Environment Variables
- Use environment variables for all configuration (12-factor app style).
- Centralize config access in a `config.py` module with functions (not constants), so client code can modify `os.environ` if needed.
- Provide sensible defaults for local development.

---

## 12. Event-Driven Microservices Integration Rules

### 12.1 Use Events for Inter-Service Communication
- Instead of synchronous HTTP API calls between services, use asynchronous events via a message broker (Redis pub/sub, Kafka, RabbitMQ, EventStore).
- This provides temporal decoupling: services can fail independently, improving overall reliability.
- Think in terms of verbs (ordering, allocating), not nouns (orders, batches).

### 12.2 Entrypoints Translate External Messages
- External message consumers (like a Redis subscriber) are thin adapters: deserialize JSON, create a Command, pass it to the message bus. Same pattern as Flask endpoints.

### 12.3 Distinguish Internal vs External Events
- Not all domain events should be published externally. Keep the distinction clear.
- Outbound events are one of the places to apply validation.

### 12.4 Design for Failure
- Explicitly choose small, focused transactions that can fail independently.
- Use retry logic with exponential backoff for transient failures (e.g., the `tenacity` library).
- Make handlers **idempotent** so that calling them repeatedly with the same message won't make repeated changes to state.
- You will need monitoring to know when transactions fail, and tooling to replay events.

### 12.5 Microservices as Consistency Boundaries
- Like aggregates, microservices should be consistency boundaries. Between two services, accept eventual consistency.
- Each service accepts commands from the outside world and raises events to record results. Other services listen to those events.

---

## 13. General Principles

### 13.1 Functional Core, Imperative Shell
- Separate what you want to do from how to do it.
- The "functional core" contains business logic with no side effects — pure functions that take simple data structures and return simple data structures.
- The "imperative shell" gathers inputs, calls the core, and applies outputs (I/O, DB writes, email sends).

### 13.2 Abstractions Hide Messy Details
- When choosing abstractions, ask:
  - Can I choose a familiar Python data structure to represent the state of the messy system?
  - Where can I draw a line between systems to insert an abstraction?
  - What are the dependencies, and what is the core business logic?
  - What implicit concepts can I make explicit?

### 13.3 Use the "Make the Change Easy, Then Make the Easy Change" Workflow
- Refactor first (preparatory refactoring) to make the architecture accommodate the new requirement, then implement the requirement.

### 13.4 CRUD Apps Don't Need These Patterns
- If your app is just a simple CRUD wrapper around a database, you don't need a domain model, repository pattern, service layer, or any of these patterns. Use Django and save yourself the bother.
- The more complex the domain, the more investment in separating infrastructure from business logic will pay off.

### 13.5 When Adding a New Feature
- Look for "When X, then Y" in requirements — these indicate events.
- Don't add side effects (email, logging, notifications) into the domain model or the service layer. Use domain events and separate handlers.
- Each new feature should map to a new command/event and handler, fitting into existing architectural patterns without requiring architectural changes.

### 13.6 Avoid the Big Ball of Mud
- A big ball of mud is the natural state of software, like wilderness is the natural state of a garden. It takes energy and direction to prevent collapse.
- Chaos in software is characterized by homogeneity: every part looks the same, with API handlers doing domain logic, sending email, and performing I/O; business logic classes doing I/O; everything coupled to everything.
- Maintain clear boundaries between layers. Business logic should never be spread across multiple layers.
