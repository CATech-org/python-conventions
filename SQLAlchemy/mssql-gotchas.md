# 9. Things that fail silently

These produce working-looking code that is wrong:

- **`NVARCHAR` for all text, never `VARCHAR`.** Review text is not always Latin. `VARCHAR` corrupts
  non-Latin characters on MSSQL without any error.
- **`DATETIME2`, never `DATETIME`.**
- **Enum columns use `_NVEnum`.** Plain `SAEnum` renders as `VARCHAR` on MSSQL.
- **The filtered unique index on `Variant.sku` is MSSQL-only.** It is declared with
  `mssql_where`; every other database silently drops the condition and makes it unconditional.
  Uniqueness among approved rows must also be checked in code.
- **Taxonomy lives in tables.** No product code, variant name, or category list as a
  Python constant, anywhere.
