# 7. Docstrings

Google style, on every class and every public function or method. `Args:` and `Returns:` are
required where they apply; `Raises:` where the function raises deliberately.

```python
async def approve(self, ref: str) -> ApprovalResult:
    """Approve a pending proposal.

    Args:
        ref: Proposal reference, ``product:{code}`` or ``variant:{id}``.

    Returns:
        The reference and its new status.

    Raises:
        InvalidOperationError: The reference is malformed.
        NotFoundError: No such proposal.
        ConflictError: The proposal is not in ``proposed`` state, or its parent
            is not approved.
    """
```

This is enforced by ruff's `D` rules with `convention = "google"`. A missing docstring fails the
pre-commit hook.
