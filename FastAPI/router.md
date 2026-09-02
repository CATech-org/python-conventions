# 6. The router layer

- Parse the request, call **one** service method, return the result.
- No `try` / `except`. Error translation happens once, in `src/main.py`.
- No business logic, no status checks, no building response models by hand.
- Never imports `src.models`, `src.repositories`, or `AsyncSession`.
- Declare `responses` on every route so the Swagger UI documents every status the route can
  return, not just the happy path.

## 6.1 `responses` documents the whole contract

`response_model` names only the 2xx shape. The `responses` attribute names every status the
route can return, and the Swagger UI renders each one, so a caller can see an error shape
before hitting it. The status set here is the one `src/main.py` maps from `src/exceptions.py`
— a status in `responses` that `src/main.py` never emits is a lie in the docs, and a status
`src/main.py` emits that is not named here is one the caller cannot see.

Concretely (names are illustrative — do not build these):

```python
@router.get(
    "/{code}",
    response_model=ProductOut,
    responses={
        404: {"model": NotFoundOut, "description": "No such product"},
        409: {"model": ConflictOut, "description": "Already approved"},
    },
)
async def get_product(
    code: str,
    service: ProductService = Depends(get_product_service),  # noqa: B008
) -> Product:
    return await service.get_product(code)
```
