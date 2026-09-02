# 7. Query parameters

The schema is justified only when the query params are a **recurring, validated bundle that
crosses the router → service boundary as one object.** Otherwise it is ceremony.

The test that decides: *will this set be the argument you hand to the service, and does it
appear on more than one route?* Yes to both → schema. Otherwise inline.

## Use a query schema when

- The params are 4+ and form one logical unit (filter + paging + sort + status + cursor).
  A model is the DRY unit — the same bundle shows up on every list endpoint.
- It becomes the **single argument the service receives.** `router.md` says the router parses
  and calls one service method; the schema is that argument. That is the whole reason it
  earns its existence.
- You want argument-level validation that costs no query — `service-layer.md` §5.3's
  "checks that cost no query." `Query(ge=1)`, enum, `list[str]`, max length all live in the
  model, not re-derived per route.
- It is declared at the router's parse step, before the one service call. Declare the
  model with `Query()` and FastAPI extracts each field — no `Depends` to wire, no
  provider to keep in sync.

## Skip the schema (inline `Annotated[X, Query()]`) when

- 1–3 params, trivially obvious. Wrapping one param in a model just moves the typing around.
- The "bundle" is only used on one route and never shared.
- You are reaching for the model to pass to the service but do not actually need the
  validation — then you have added an object for the shape, not the behavior.

## Concretely (names are illustrative — do not build these)

The schema case. The model is the bundle; declare it as a query parameter and FastAPI
extracts each field. Defaults and constraints live in the model, not in a provider.

```python
from __future__ import annotations

from typing import Annotated

from fastapi import APIRouter, Depends, Query
from pydantic import BaseModel, Field

router = APIRouter()


class ItemsQuery(BaseModel):
    """One list-filter bundle, shared by every list endpoint."""

    q: str | None = Field(default=None, max_length=200)
    status: str | None = Field(default=None)
    limit: int = Field(default=20, ge=1)
    cursor: str | None = Field(default=None)


@router.get("", response_model=ItemsListOut)
async def list_items(
    query: Annotated[ItemsQuery, Query()],
    service: ItemsService = Depends(get_items_service),  # noqa: B008
) -> ItemsListOut:
    """List items. Parses the bundle, calls one service method."""
    return await service.list_items(query)
```

A `Depends` provider that builds the model also works, but it re-declares every field
the model already declares — the ceremony the Skip list is about — so the plain
`Annotated[Model, Query()]` form is the default. Extra query parameters are silently
ignored unless the model sets `model_config = {"extra": "forbid"}`. The model extraction
is supported only from FastAPI 0.115.0 on; a pinned older release will not extract the
model's fields from the query string.

The inline case. Few params, one route, nothing to share. Keep them on the signature; a
model here is typing moved to another file for no buyer.

```python
@router.get("/search", response_model=ItemsListOut)
async def search(
    q: Annotated[str, Query(min_length=1)],
    limit: Annotated[int, Query()] = 20,
    service: ItemsService = Depends(get_items_service),  # noqa: B008
) -> ItemsListOut:
    """Search by one required term. Two params, no bundle."""
    return await service.search(q, limit)
```
