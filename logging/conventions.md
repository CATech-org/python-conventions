# 10. Logging

Logging is configured once, in `src/core/log_setup.py`, and nowhere else.

- Modules use `logging.getLogger(__name__)`. Nothing outside `log_setup.py` calls
  `addHandler`, `basicConfig`, or `dictConfig`. `logging.getLogger(name)` returns the same
  object every call, so a handler added anywhere else piles up on every later call and can
  open two handles on one file.
- Lazy formatting only: `logger.info("job %d done", job_id)`, never an f-string. An
  f-string is built even when the level is switched off; the `%`-style arguments are not.
- **Never log a filename, review text, summary text, or anything from `.env`.** A
  filename in this project can identify a customer. Log the job id instead; it
  resolves to the file through the database when a human needs it.
- `logger.exception` only inside an `except` block. Outside one, there is no traceback to
  attach and the call is misleading.

**`@logged` from `src/core/log_context.py` goes on the method you want a record of.** It
logs how long the call took, logs and re-raises on failure, and sets the job id
for everything below it when the method takes a `job` argument. Method level is
deliberate: it keeps the body flat, at the cost of not timing the individual steps inside.
