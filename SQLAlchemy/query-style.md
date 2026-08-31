# 2. Query style — SQLAlchemy 2.0 only

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
