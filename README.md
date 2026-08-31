# python-conventions

**Purpose.** The binding conventions for Python in this project's backend — FastAPI over async
SQLAlchemy 2.0 over MSSQL, on a router → service → repository layer split.

**Read this before writing any code.** These files say *how* the code must be written. A rule
here is not a preference. Code that breaks one is wrong even if it runs and even if the
acceptance check passes.

The conventions are grouped by the layer or technology they constrain.

## SQLAlchemy

- [async.md](SQLAlchemy/async.md) — everything is async
- [query-style.md](SQLAlchemy/query-style.md) — query style, SQLAlchemy 2.0 only
- [mssql-gotchas.md](SQLAlchemy/mssql-gotchas.md) — things that fail silently

## FastAPI

- [dependency-chain.md](FastAPI/dependency-chain.md) — the dependency chain
- [router.md](FastAPI/router.md) — the router layer

## architecture

- [repository-layer.md](architecture/repository-layer.md) — the repository layer
- [service-layer.md](architecture/service-layer.md) — the service layer

## style

- [docstrings.md](style/docstrings.md) — docstrings
- [pep8.md](style/pep8.md) — style

## logging

- [conventions.md](logging/conventions.md) — logging
