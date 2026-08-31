# 5. The service layer

Business logic only.

- **No `AsyncSession`.** A service never receives, holds, or touches a session.
- **No `select()`, no `session.execute`, no `commit`, no `rollback`.** All of it goes through
  repositories.
- **No `fastapi`, no `starlette`, no `HTTPException`.** Services raise the types in
  `src/exceptions.py`; `src/main.py` maps those to status codes.
- Validate what the database cannot, before calling anything that writes. See §5.1 for what that
  does and does not cover.

## 5.1 A bad request never surfaces as a 500

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

## 5.2 `IntegrityError` is a bug, not a status code

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

## 5.3 What the database cannot tell you

Two things a constraint will never catch, so they are checked in code:

- **A request that contradicts itself.** A merge naming the same row as winner and loser commits
  cleanly and leaves a row that supersedes itself — no exception, no log line. Argument-level
  checks like this cost no query and run in the service, first, before any repository call.
- **A soft-deleted parent.** A foreign key does not reject one: the row still exists,
  `deleted_at` is merely set. Only a read filtering `deleted_at.is_(None)` catches it. There are
  no DELETE endpoints today, so nothing is exposed to this yet — **when the first one is added,
  every write that references a soft-deletable parent needs a pre-read**, and that read finally
  earns its round trip.

## 5.4 A simple service

Everything above in one class. Concretely (names are illustrative — do not build these):

```python
from __future__ import annotations

from fastapi import Depends

from src.exceptions import (
    ConflictError,
    InvalidOperationError,
    NotFoundError,
)
from src.modules.catalog.models import Product, ProductStatus, Variant
from src.modules.catalog.repository import (
    ProductRepository,
    VariantRepository,
    get_product_repository,
    get_variant_repository,
)


class ProductService:
    """Owns the business rules for the catalog.

    Holds no session and writes no SQL: every read and every write goes
    through the repositories it was given.
    """

    def __init__(
        self,
        products: ProductRepository,
        variants: VariantRepository,
    ) -> None:
        self.products = products
        self.variants = variants

    async def create_product(self, data: dict) -> Product:
        """Create a new product.

        Args:
            data: Field-value mapping for the product (code, name,
                country_id, ...).

        Returns:
            The newly created Product.

        Raises:
            NotFoundError: No country with ``data["country_id"]``.
        """
        return await self.products.create(data)

    async def get_product(self, code: str) -> Product:
        """Read one product by code.

        Args:
            code: Primary key of the product.

        Returns:
            The Product instance.

        Raises:
            NotFoundError: No such product (or it is soft-deleted).
        """
        product = await self.products.read(code)
        if product is None:
            raise NotFoundError(f"Product {code} not found")
        return product

    async def list_products(self) -> list[Product]:
        """List the approved catalog.

        Returns:
            Every approved, non-deleted Product.
        """
        return await self.products.list_approved()

    async def delete_product(self, code: str) -> None:
        """Soft-delete a draft product.

        Args:
            code: Primary key of the product.

        Raises:
            NotFoundError: No such product.
            InvalidOperationError: The product is not in ``DRAFT`` status.
        """
        product = await self.get_product(code)
        if product.status is not ProductStatus.DRAFT:
            raise InvalidOperationError("Cannot delete a product that is not DRAFT")
        await self.products.delete(code)

    async def create_variant(self, product_code: str, data: dict) -> Variant:
        """Attach a new variant to a product.

        Args:
            product_code: The product the variant belongs to.
            data: Field-value mapping for the variant (sku, price, ...).

        Returns:
            The newly created Variant.

        Raises:
            NotFoundError: No product with ``product_code``.
            ConflictError: An approved variant already has that sku.
        """
        product = await self.products.read(product_code)
        if product is None:
            raise NotFoundError(f"Product {product_code} not found")
        if await self.variants.sku_in_use(data["sku"]):
            raise ConflictError(f"Variant sku {data['sku']} is already in use")
        data["product_code"] = product_code
        return await self.variants.create(data)


def get_product_service(
    products: ProductRepository = Depends(get_product_repository),  # noqa: B008
    variants: VariantRepository = Depends(get_variant_repository),  # noqa: B008
) -> ProductService:
    return ProductService(products, variants)
```
