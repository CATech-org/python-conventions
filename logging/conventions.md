# 10. Logging

Logging is configured once, in `src/core/log_setup.py`, and nowhere else.

- Modules use `logging.getLogger(__name__)`. Nothing outside `log_setup.py` calls
  `addHandler`, `basicConfig`, or `dictConfig`. `logging.getLogger(name)` returns the same
  object every call, so a handler added anywhere else piles up on every later call and can
  open two handles on one file.
- Lazy formatting only: `logger.info("job %d done", job_id)`, never an f-string. An
  f-string is built even when the level is switched off; the `%`-style arguments are not.
  See §10.1.
- **Never log a filename, review text, summary text, or anything from `.env`.** A
  filename in this project can identify a customer. Log the job id instead; it
  resolves to the file through the database when a human needs it.
- `logger.exception` only inside an `except` block. Outside one, there is no traceback to
  attach and the call is misleading.

**`@logged` from `src/core/log_context.py` goes on the method you want a record of.** It
logs how long the call took, logs and re-raises on failure, and sets the job id
for everything below it when the method takes a `job` argument. Method level is
deliberate: it keeps the body flat, at the cost of not timing the individual steps inside.

## 10.1 Why lazy, not f-strings

The cost is **when** the string is built, not how. An f-string is evaluated the moment
Python reaches the call, before `logger.debug` runs and whether or not the record is
emitted. The `%`-style defers: the values go in as `args` and are only formatted inside
`record.getMessage()`, which `Formatter.format` calls only once a record passes the level
check and is written.

```python
# eager — formatted even if DEBUG is off
logger.debug(f"order {order_id} for user {user_id}")

# lazy — formatted only if the record is actually emitted
logger.debug("order %s for user %s", order_id, user_id)
```

It is the logging module's design, not a preference. `LogRecord` carries an `args` tuple
for this, and the level check happens in `Logger._log`, before `makeRecord`, so a disabled
level means no `getMessage` and no formatting. An f-string cannot join that deferral.

That is where it shows up. `DEBUG` is off in production. An f-string in `logger.debug(...)`
still pays the interpolation cost even though the message is thrown away, and in a hot path
that emits hundreds of debug calls a second, all filtered out, the wasted formatting adds up.

The one catch: a bare `%` in the literal with no `args` makes `logging` treat it as
formatting and raise `ValueError: not all arguments converted during string formatting`.
Escape it or keep the literal free of `%`.
