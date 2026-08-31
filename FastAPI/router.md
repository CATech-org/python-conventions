# 6. The router layer

- Parse the request, call **one** service method, return the result.
- No `try` / `except`. Error translation happens once, in `src/main.py`.
- No business logic, no status checks, no building response models by hand.
- Never imports `src.models`, `src.repositories`, or `AsyncSession`.
