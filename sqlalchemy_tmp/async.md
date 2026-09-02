# 1. Everything is async

- Repository methods, service methods, and route handlers are all `async def`.
- The session type is `AsyncSession`, never `Session`.
- Queries are awaited: `await session.execute(...)`, `await session.scalars(...)`.
- `await session.commit()`, `await session.rollback()`, `await session.flush()`.

**Never call a blocking database function from async code.** No `session.query()`, no
`engine.connect()` without `async with`, no `asyncio.run()` inside a request.
