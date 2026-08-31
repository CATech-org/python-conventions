# 4. The repository layer

## 4.1 The base class

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

## 4.2 Repositories own the transaction

**Every database operation, including `commit()`, happens in the repository layer.** A service
never touches a session.

- A repository method that writes ends with `await self.session.commit()`.
- On failure it calls `await self.session.rollback()` before the exception escapes.
- **One repository method per atomic unit of work.** If two rows must change together or not at
  all, that is *one* method with *one* commit — not two calls from the service.

  Placing an order writes the order and its lines together, so it is one method:

  ```python
  async def create_order_with_lines(self, order: Order, lines: list[OrderLine]) -> None:
  ```

  Not a loop of `create()` calls from the service. A line without its order is a corrupt ledger.

## 4.3 Deletes are soft

Every table carries `deleted_at` through `TimestampMixin`.

- `delete()` sets `deleted_at` to the current UTC time. It commits.
- **`session.delete()` is forbidden anywhere in `src/`.** It destroys the audit trail the project
  exists to produce.
- **Every read filters `deleted_at.is_(None)`.** No exceptions. A soft-deleted row does not exist.

## 4.4 `update()` must not touch the approval gate

`update(id, data: dict)` **rejects `status` and `deleted_at`**. Every concrete `update()` calls
`self._reject_protected(data)` before writing anything; the base class raises
`InvalidOperationError` if either field appears.

Without that guard, `await repo.update("SOME_CODE", {"status": "approved"})` makes a row visible to
the agent without any human approving it. That breaks a design invariant.

Status changes happen only through named, intent-revealing methods — `set_status()`,
`approve_and_supersede()` — which the approval service calls after running its checks.

## 4.5 Approval-gated reads

Repositories expose reads by approval state, by name:

- `list_approved(...)` — anything on the agent path.
- `list_proposed(...)` — the approval API only.

**Never write `list_all()`** or any other unfiltered listing method. Its absence is what makes the
rule "the agent cannot read its own proposals" greppable.

## 4.6 A repository touches only its own module's models

A repository imports, queries, and writes **only the models defined in its own
`src/modules/<module>/models.py`**. It may own more than one model when they belong to the same
module — a `OrderRepository` that also writes `Membership` is fine, both are
billing models. It must never `from src.modules.<other>.models import ...`.

This is a module-boundary rule, not a style preference. A repository that reaches into another
module's tables couples two domains at the layer that is meant to be the smallest, most
substitutable unit, and it hides that coupling from the dependency chain in §3
(`FastAPI/dependency-chain.md`). It is wrong even if the query is correct and the test passes.

- **A write that needs a value from another module takes it as an argument.** The caller passes
  it in; the repository does not go and fetch it.
- **A read that needs another module's data goes through that module's repository**, injected
  into the service with `Depends` (§3, `FastAPI/dependency-chain.md`) like any other collaborator
  — not by importing the model and writing a `select()`.
- **A model written by one module and only read by another belongs to the writer.** Move the
  model and its repository into the module that maintains its invariants; the reading module
  consumes it through the owning module's repository. `CustomerSummary` is billing state (a
  projection of the order ledger), not customer configuration — it lives in `billing/`.
