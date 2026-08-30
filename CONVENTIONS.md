# CONVENTIONS.md — how code is written in this project

**Read this before writing any code.** This file says *how* the code must be written.

A rule here is not a preference. Code that breaks one is wrong even if it runs and even if the
acceptance check passes.

---

## 1. Everything is async

The database driver is `aioodbc` (`mssql+aioodbc`). There is no sync path.

- Repository methods, service methods, and route handlers are all `async def`.
- The session type is `AsyncSession`, never `Session`.
- Queries are awaited: `await session.execute(...)`, `await session.scalars(...)`.
- `await session.commit()`, `await session.rollback()`, `await session.flush()`.

The one exception is `migrations/env.py`, which Alembic drives itself.

**Never call a blocking database function from async code.** No `session.query()`, no
`engine.connect()` without `async with`, no `asyncio.run()` inside a request.

## 2. Query style — SQLAlchemy 2.0 only

Use `select()` with `await session.scalars(...)` or `await session.execute(...)`.

```python
stmt = (
    select(Product)
    .where(
        Product.status == ProductStatus.DRAFT,
        Product.deleted_at.is_(None),
    )
    .order_by(Product.created_at)
)
rows = list(await session.scalars(stmt))
```

**`session.query(...)` is forbidden.** It is the legacy 1.x API, it has no async form, and it will
not work against an `AsyncSession`.

## 3. The dependency chain

Every dependency is supplied by FastAPI `Depends`. Nothing constructs its own collaborators.

```
router  ──Depends──▶  service  ──Depends──▶  repository  ──Depends──▶  AsyncSession
```

Concretely (names are illustrative — do not build these):

```python
# db layer
async def get_db() -> AsyncGenerator[AsyncSession]:
    async with SessionLocal() as session:
        yield session


# repository layer
async def get_foo_repository(
    session: AsyncSession = Depends(get_db),  # noqa: B008
) -> FooRepository:
    return FooRepository(session)


# service layer
async def get_bar_service(
    foos: FooRepository = Depends(get_foo_repository),  # noqa: B008
    bazs: BazRepository = Depends(get_baz_repository),  # noqa: B008
) -> BarService:
    return BarService(foos, bazs)


# router layer
@router.get("", response_model=SomeListOut)
async def list_items(
    service: BarService = Depends(get_bar_service),  # noqa: B008
) -> SomeListOut:
    return await service.list_items()
```

Rules:

- **A service never builds a repository.** It receives them. `BarService(session)` is wrong;
  `BarService(foos, bazs)` is right.
- **A router never builds a service**, and never sees an `AsyncSession` at all.
- **A repository never builds another repository.**
- The provider function for each layer lives in that layer's module, next to the class.
- `# noqa: B008` on `Depends(...)` in a default is expected. It is the FastAPI pattern; ruff's
  B008 does not know that.

### 3.1 Background jobs are the one exception

The chain above is built by FastAPI, once per request. Code that runs without a request has no
`Depends` and must build the chain itself. Today that means scheduled jobs. It means nothing
else.

```python
# a scheduled job — the only place allowed to do this
async def process_pending_reviews() -> None:
    async with SessionLocal() as session:
        service = ReviewService(
            ReviewJobRepository(session),
            ReviewRepository(session),
        )
        await service.process_next()
```

Rules:

- **One session per job run.** Open it when the run starts, close it when the run ends. Never
  hold a session between runs, never create one at import time, and never share one across two
  jobs.
- **The job function contains no business logic.** It opens the session, builds the chain, and
  calls one service method. Deciding what to process, in what order, and what to do on failure is
  the service's work.
- **The exception stops at the job function.** Services and repositories still receive their
  collaborators. A service that builds its own repository is wrong inside a job for the same
  reason it is wrong inside a request.
- **The repository still owns the transaction.** A job does not commit, does not roll back, and
  does not open a transaction of its own. §4.2 applies unchanged.
- **All repositories in one job run share one session.** Passing the same session to each is what
  makes a multi-row write one transaction.

Scheduling lives in `src/core/scheduler.py`: the APScheduler instance, its start and stop, and
the schedule itself. Nothing else. If queue rules end up in `core/`, the layer split is broken.

**One process, one scheduler.** Running the app with more than one worker gives one scheduler per
worker, and each will pick up the same pending row. Processing is sequential, one review at a time. Either the app runs with a single worker, or the job needs
a lock. Decide before the scheduler is switched on, not after.

## 4. The repository layer

### 4.1 The base class

Every repository inherits from `BaseRepository`, in `src/core/repository.py`:

```python
from abc import ABC, abstractmethod
from typing import Any, Generic, TypeVar

from sqlalchemy.ext.asyncio import AsyncSession

from src.core.base import Base

T = TypeVar("T", bound=Base)


class BaseRepository(ABC, Generic[T]):
    """Base class for all repositories.

    Owns the database session and every write to it, including the commit.

    Args:
        session: Async session for this request.
        model: SQLAlchemy model class this repository is responsible for.
    """

    PROTECTED_FIELDS: frozenset[str] = frozenset({"status", "deleted_at"})

    def __init__(self, session: AsyncSession, model: type[T]) -> None:
        self.session = session
        self.model = model

    def _reject_protected(self, data: dict) -> None:
        """Refuse an update that would write an approval-gated field.

        Args:
            data: Field-value mapping about to be written.

        Raises:
            InvalidOperationError: ``data`` contains a protected field.
        """
        blocked = self.PROTECTED_FIELDS & data.keys()
        if blocked:
            raise InvalidOperationError(
                f"Fields {sorted(blocked)} cannot be changed through update()"
            )

    @abstractmethod
    async def create(self, data: T) -> T:
        """Persist a new row."""
        ...

    @abstractmethod
    async def read(self, id: Any) -> T | None:
        """Read one row by primary key, or None."""
        ...

    @abstractmethod
    async def update(self, id: Any, data: dict) -> bool:
        """Update non-protected fields on one row."""
        ...

    @abstractmethod
    async def delete(self, id: Any) -> bool:
        """Soft-delete one row."""
        ...
```

A concrete repository passes its model to `super().__init__`, implements all four abstract
methods, and adds whatever domain methods it needs:

```python
class ProductRepository(BaseRepository[Product]):
    def __init__(self, session: AsyncSession) -> None:
        super().__init__(session, Product)
```

`read(id: Any)` is deliberately `Any`: `Product`'s primary key is `code`, an
`NVARCHAR(50)` string, while `Variant`'s is an `int`. One signature has to cover both.

### 4.2 Repositories own the transaction

**Every database operation, including `commit()`, happens in the repository layer.** A service
never touches a session.

- A repository method that writes ends with `await self.session.commit()`.
- On failure it calls `await self.session.rollback()` before the exception escapes.
- **One repository method per atomic unit of work.** If two rows must change together or not at
  all, that is *one* method with *one* commit — not two calls from the service.

  Approving a merge changes the winner and every loser together, so it is one method:

  ```python
  async def approve_and_supersede(self, winner_id: int, loser_ids: list[int]) -> None:
  ```

  Not a loop of `set_status()` calls from the service. A partial merge is a corrupt taxonomy.

### 4.3 Deletes are soft

Every table carries `deleted_at` through `TimestampMixin`.

- `delete()` sets `deleted_at` to the current UTC time. It commits.
- **`session.delete()` is forbidden anywhere in `src/`.** It destroys the audit trail the project
  exists to produce.
- **Every read filters `deleted_at.is_(None)`.** No exceptions. A soft-deleted row does not exist.

### 4.4 `update()` must not touch the approval gate

`update(id, data: dict)` **rejects `status` and `deleted_at`**. Every concrete `update()` calls
`self._reject_protected(data)` before writing anything; the base class raises
`InvalidOperationError` if either field appears.

Without that guard, `await repo.update("SOME_CODE", {"status": "approved"})` makes a row visible to
the agent without any human approving it. That breaks a design invariant.

Status changes happen only through named, intent-revealing methods — `set_status()`,
`approve_and_supersede()` — which the approval service calls after running its checks.

### 4.5 Approval-gated reads

Repositories expose reads by approval state, by name:

- `list_approved(...)` — anything on the agent path.
- `list_proposed(...)` — the approval API only.

**Never write `list_all()`** or any other unfiltered listing method. Its absence is what makes the
rule "the agent cannot read its own proposals" greppable.

### 4.6 A repository touches only its own module's models

A repository imports, queries, and writes **only the models defined in its own
`src/modules/<module>/models.py`**. It may own more than one model when they belong to the same
module — a `OrderRepository` that also writes `Membership` is fine, both are
billing models. It must never `from src.modules.<other>.models import ...`.

This is a module-boundary rule, not a style preference. A repository that reaches into another
module's tables couples two domains at the layer that is meant to be the smallest, most
substitutable unit, and it hides that coupling from the dependency chain in §3. It is wrong even
if the query is correct and the test passes.

- **A write that needs a value from another module takes it as an argument.** The caller passes
  it in; the repository does not go and fetch it.
- **A read that needs another module's data goes through that module's repository**, injected
  into the service with `Depends` (§3) like any other collaborator — not by importing the model
  and writing a `select()`.
- **A model written by one module and only read by another belongs to the writer.** Move the
  model and its repository into the module that maintains its invariants; the reading module
   consumes it through the owning module's repository. `CustomerSummary` is billing state (a
   projection of the order ledger), not customer configuration — it lives in `billing/`.

## 5. The service layer

Business logic only.

- **No `AsyncSession`.** A service never receives, holds, or touches a session.
- **No `select()`, no `session.execute`, no `commit`, no `rollback`.** All of it goes through
  repositories.
- **No `fastapi`, no `starlette`, no `HTTPException`.** Services raise the types in
  `src/exceptions.py`; `src/main.py` maps those to status codes.
- Validate what the database cannot, before calling anything that writes. See §5.1 for what that
  does and does not cover.

### 5.1 A bad request never surfaces as a 500

A caller that sends a `review_id` which does not exist must get a **404 with a message
naming it** — never a bare `Internal Server Error`. A caller that asks for something the data
will not allow must get a 409, and one that contradicts itself must get a 400.

That does **not** mean checking every foreign key before every write. The database already
enforces them, and an extra `SELECT` per write to defend against input the frontend does not
produce is a cost with no buyer. What it means is that the failure must arrive in the project's
own vocabulary — see §5.2 for how, and for the two places a check really does have to happen in
code.

Checks that cost no query always run in the service, before anything is written: argument
validation, self-contradictory requests, anything answerable from the arguments alone.

`src/exceptions.py` holds the whole vocabulary, and `src/main.py` already maps all three:

| Raise | Status | For |
|---|---|---|
| `NotFoundError` | 404 | The row does not exist, or is soft-deleted — including a referenced parent row |
| `ConflictError` | 409 | The row exists but is in the wrong state, or the write collides with an existing row |
| `InvalidOperationError` | 400 | The request contradicts itself — a merge naming the same row as winner and loser |

Adding a status code means adding an exception type and a handler. It never means raising
`HTTPException` from a service.

### 5.2 `IntegrityError` is a bug, not a status code

**There is no global `IntegrityError` handler and there must not be one.** `IntegrityError` is
four unrelated failures wearing one name — a foreign-key violation (404), a unique violation
(409), a `NOT NULL` violation (422), a check-constraint violation (400). Mapping them all to one
code answers most requests wrongly, and telling them apart means pattern-matching MSSQL error
numbers out of the driver's message string, which is fragile.

So: an unexpected `IntegrityError` stays a 500. That is the correct outcome. It means the code
wrote something it never checked, and it should be loud enough to find.

**Translate it at the call site instead.** A handler cannot know which constraint fired, but a
repository method can — it wrote the statement. `ProductRepository.create` inserts one row whose
only reachable constraint is the `country_id` foreign key, so it catches `IntegrityError`, rolls
back, and raises `NotFoundError(f"Country {id} not found")`. Correct status, specific message,
**zero extra queries**. The repository owns the transaction, so it owns the failure.

A call-site translation is legitimate only where both hold:

1. It sits in the repository method that issued the statement, never in a handler.
2. The constraint being attributed is **named in a comment**, along with why the attribution
   holds. If more than one constraint could have fired, say which ones and why the others are
   unreachable.

Point 2 is the whole safeguard, and the attribution is usually an inference about the *caller*,
not a fact about the exception. `products` can also violate `NOT NULL` on `name`; what makes
`Country {id} not found` safe is that `ProductCreate` types `name` as a required `str`. When that
stops being true the message becomes a lie, and the comment is what lets the next reader notice.

Catch `sqlalchemy.exc.IntegrityError` itself — never `Exception`, `SQLAlchemyError`, or
`DBAPIError` — and never inspect `exc.orig`, the driver message, or MSSQL error numbers. If a
statement has two constraints you cannot tell apart, that is when you pre-check, not when you
start parsing driver strings.

### 5.3 What the database cannot tell you

Two things a constraint will never catch, so they are checked in code:

- **A request that contradicts itself.** A merge naming the same row as winner and loser commits
  cleanly and leaves a row that supersedes itself — no exception, no log line. Argument-level
  checks like this cost no query and run in the service, first, before any repository call.
- **A soft-deleted parent.** A foreign key does not reject one: the row still exists,
  `deleted_at` is merely set. Only a read filtering `deleted_at.is_(None)` catches it. There are
  no DELETE endpoints today, so nothing is exposed to this yet — **when the first one is added,
  every write that references a soft-deletable parent needs a pre-read**, and that read finally
  earns its round trip.

## 6. The router layer

- Parse the request, call **one** service method, return the result.
- No `try` / `except`. Error translation happens once, in `src/main.py`.
- No business logic, no status checks, no building response models by hand.
- Never imports `src.models`, `src.repositories`, or `AsyncSession`.

## 7. Docstrings

Google style, on every class and every public function or method. `Args:` and `Returns:` are
required where they apply; `Raises:` where the function raises deliberately.

```python
async def approve(self, ref: str) -> ApprovalResult:
    """Approve a pending proposal.

    Args:
        ref: Proposal reference, ``product:{code}`` or ``variant:{id}``.

    Returns:
        The reference and its new status.

    Raises:
        InvalidOperationError: The reference is malformed.
        NotFoundError: No such proposal.
        ConflictError: The proposal is not in ``proposed`` state, or its parent
            is not approved.
    """
```

This is enforced by ruff's `D` rules with `convention = "google"`. A missing docstring fails the
pre-commit hook.

## 8. Style

- PEP 8, enforced by ruff.
- **Line length 100.** This overrides PEP 8's 79 and is the project's setting.
- Imports at module top, sorted by ruff's isort rules. **Never import inside a function body**
  unless breaking a genuine circular import, and say so in a comment when you do.
- `from __future__ import annotations` at the top of every module that has type annotations.
- Type-annotate every parameter and return value.
- Modern syntax: `X | None`, not `Optional[X]`. `list[X]`, not `List[X]`.
- No commented-out code, no `TODO` without a task number, no dead branches.

## 9. Things that fail silently

These produce working-looking code that is wrong:

- **`NVARCHAR` for all text, never `VARCHAR`.** Review text is not always Latin. `VARCHAR` corrupts
   non-Latin characters on MSSQL without any error.
- **`DATETIME2`, never `DATETIME`.**
- **Enum columns use `_NVEnum`.** Plain `SAEnum` renders as `VARCHAR` on MSSQL.
- **The filtered unique index on `Variant.sku` is MSSQL-only.** It is declared with
  `mssql_where`; every other database silently drops the condition and makes it unconditional.
  Uniqueness among approved rows must also be checked in code.
- **Taxonomy lives in tables.** No product code, variant name, or category list as a
  Python constant, anywhere.

## 10. Logging

Logging is configured once, in `src/core/log_setup.py`, and nowhere else.

- Modules use `logging.getLogger(__name__)`. Nothing outside `log_setup.py` calls
  `addHandler`, `basicConfig`, or `dictConfig`. `logging.getLogger(name)` returns the same
  object every call, so a handler added anywhere else piles up on every later call and can
  open two handles on one file.
- Lazy formatting only: `logger.info("job %d done", job_id)`, never an f-string. An
  f-string is built even when the level is switched off; the `%`-style arguments are not.
- **Never log a filename, review text, summary text, or anything from `.env`.** A
  filename in this project can identify a customer. Log the job id instead; it
  resolves to the file through the database when a human needs it.
- `logger.exception` only inside an `except` block. Outside one, there is no traceback to
  attach and the call is misleading.

**`@logged` from `src/core/log_context.py` goes on the method you want a record of.** It
logs how long the call took, logs and re-raises on failure, and sets the job id
for everything below it when the method takes a `job` argument. Method level is
deliberate: it keeps the body flat, at the cost of not timing the individual steps inside. 
