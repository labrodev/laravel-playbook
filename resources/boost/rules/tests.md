---
paths:
  - tests/**
---

# Tests — playbook checks

- Universal: file opens with `declare(strict_types=1)`.
- Every state-mutating Action is covered by direct Action tests (happy, unhappy, edge), invoked as a callable with named arguments — never through a controller.
- Unhappy-path tests match the Action's failure mode: `toThrow(ValidationException::class)` for user-fixable violations, unchanged-state assertions for silent no-ops.
- Validation failures are tested at the Data level with `validateAndCreate()` and raw input keys — not via Actions or HTTP.
- All persisted domain state in setup is built through Core Actions (via `tests/Pest.php` helpers); factories are reserved for framework fixtures like `User`.
- Route Feature tests are few and wiring-only (status/redirect, auth/authorization, persistence) — no business-rule assertions, no complex domain setup.
- Mocking is limited to external services, time/UUID, and infrastructure — no mocked Actions, Rules, or Services under test.
- `tests/ArchTest.php` exists and passes: strict types everywhere, final classes, Core never imports `App\Layer`, no FormRequest outside the legacy zone, no DB/Validator facades in the delivery layer.
- Jobs are tested as wrappers (delegation + retry/backoff config), never as business units.
- Zero tests that merely prove Laravel works (relations, casts, accessors).
- All test names read as behavior statements; test files mirror `tests/Feature/{Domain}/`.

Full anatomy and templates → labrodev-testing skill.
