---
paths:
  - database/migrations/**
---

# Migrations — playbook checks

- Universal: file opens with `declare(strict_types=1)`.
- Table names are the consistent snake_plural of the singular model name (`Booking` → `bookings`).
- Any table whose model appears in URLs or needs a stable external identifier has a `uuid` column, unique and indexed — the value is assigned in the create Action, never by the database or model.
- Columns backing enums use a column type matching the enum's backing type.
- Schema stays predictable — no implicit behavior, no magic columns; every column added is mirrored in the model's `casts()` when applicable.
- `timestamps()` is present; `softDeletes()` is added when the model uses `SoftDeletes`.
- The migration matches the model, and the ide-helper mixin is regenerated after every schema change.

Full anatomy and templates → labrodev-model skill.
