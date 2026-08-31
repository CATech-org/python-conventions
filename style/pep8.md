# 8. Style

- PEP 8, enforced by ruff.
- **Line length 100.** This overrides PEP 8's 79 and is the project's setting.
- Imports at module top, sorted by ruff's isort rules. **Never import inside a function body**
  unless breaking a genuine circular import, and say so in a comment when you do.
- `from __future__ import annotations` at the top of every module that has type annotations.
- Type-annotate every parameter and return value.
- Modern syntax: `X | None`, not `Optional[X]`. `list[X]`, not `List[X]`.
- No commented-out code, no `TODO` without a task number, no dead branches.
