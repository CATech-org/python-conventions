# 3. The dependency chain

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

## 3.1 Background jobs are the one exception

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
  does not open a transaction of its own. §4.2 (in `architecture/repository-layer.md`) applies
  unchanged.
- **All repositories in one job run share one session.** Passing the same session to each is what
  makes a multi-row write one transaction.

Scheduling lives in `src/core/scheduler.py`: the APScheduler instance, its start and stop, and
the schedule itself. Nothing else. If queue rules end up in `core/`, the layer split is broken.

**One process, one scheduler.** Running the app with more than one worker gives one scheduler per
worker, and each will pick up the same pending row. Processing is sequential, one review at a
time. Either the app runs with a single worker, or the job needs a lock. Decide before the
scheduler is switched on, not after.
