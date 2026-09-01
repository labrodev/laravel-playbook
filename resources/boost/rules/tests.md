---
paths:
  - tests/**
---

# Tests — playbook checks

- Universal: file opens with `declare(strict_types=1)`.
- Two Pest suites mirror the code tree exactly: `tests/Unit/**` mirrors the class under test's full path (`tests/Unit/Core/Domain/{Domain}/Actions/...`); `tests/Feature/**` mirrors the delivery structure (`tests/Feature/App/Layer/{Layer}/{Domain}/...`) and holds end-to-end controller scenarios.
- Every state-mutating Action is covered by direct Action tests (happy, unhappy, edge), invoked as a callable with named arguments — never through a controller.
- Unhappy-path tests assert the named domain exception: `toThrow({Condition}Exception::class)` — never `ValidationException` outside Data-level validation tests.
- Can-trio ineligibility is covered by Rule unit tests plus Feature tests asserting 403 — not by Action tests.
- Validation failures are tested at the Data level with `validateAndCreate()` and raw input keys — not via Actions or HTTP.
- All persisted domain state in setup is built through Core Actions (via `tests/Pest.php` helpers); factories are reserved for framework fixtures like `User`.
- Feature tests are few and end-to-end (status/redirect, auth/authorization, persistence) — no business-rule assertions, no complex domain setup.
- Mocking is limited to external services, time/UUID, and infrastructure — no mocked Actions, Rules, or Services under test.
- `tests/ArchTest.php` exists and passes: strict types everywhere, final classes, Core never imports `App` at all, no FormRequest outside the legacy zone, no DB/Validator facades in the delivery layer, no `ValidationException` in `Core\Domain`.
- Jobs are tested as wrappers (delegation + retry/backoff config), never as business units.
- Zero tests that merely prove Laravel works (relations, casts, accessors).
- All test names read as behavior statements.

Full anatomy and templates → labrodev-testing skill.
