# Labrodev Testing

Always-on law. Tests exist to protect behavior, enable refactoring, and make intent explicit — not to satisfy coverage metrics. A good test explains what the system does and fails only when behavior changes; if a test breaks during refactoring but behavior did not change, the test is wrong. Anatomy and templates → the `labrodev-testing` skill; per-file checks → `rules/tests.md`.

## Musts

- **Test behavior, not implementation.** Assert resulting state, returned values, and thrown exceptions — never internal method calls or step ordering.
- **Follow the pyramid**, most valuable first: **(1)** Action tests, **(2)** Rule/Service/Pipeline unit tests, **(3)** Job tests, **(4)** route-level Feature tests (few, wiring-only).
- **Test business behavior in Core** — call Actions directly as callables with named arguments (`$bookingCreate(bookingData: $bookingData);`), never through controllers.
- **Every Action** that mutates state, enforces business rules, or coordinates domain objects gets happy-path, unhappy-path, and edge-case tests.
- **Unhappy-path tests match the Action's failure mode**: user-fixable violations → assert `ValidationException` is thrown; state-based ineligibility → assert the silent no-op (state unchanged) (→ labrodev-action).
- **Test validation at the Data level** with `{Model}{Operation}Data::validateAndCreate([...])` + `ValidationException` assertions — never via Actions or HTTP.
- **Construct Data objects explicitly** in tests (`BookingData::from([...])`); never pass raw arrays into Actions.
- **Build domain state needed as setup through Core Actions**, via global helpers in `tests/Pest.php` — so invariants (UUID assignment, initial status, guarded transitions) hold in fixtures exactly as in production.
- **Mirror the domain structure**: `tests/Feature/{Domain}/` (e.g. `tests/Feature/Booking/BookingCreateTest.php`).
- **Test names are descriptive and behavior-focused**: `it('creates a booking with valid data')`.
- **Keep an architecture test suite** (`pest-plugin-arch`) that mechanically enforces the playbook — paste-ready template in the skill.
- **In tests, obtain Actions via `app(BookingCreate::class)` or `new BookingCreate()`** — the "no `app()`/`resolve()`" rule applies to controllers, not tests (→ labrodev-controller).

## Must-nots

- Never build domain state with raw model factories (`Booking::factory()->create()`) — factories bypass Rules, UUID assignment, and status transitions. Factories only for framework-level fixtures with no domain invariants (e.g. `User::factory()` for `actingAs()`).
- Never re-test business rules in HTTP tests, and never put complex domain setup in them — they verify wiring only (route connected, auth/authorization wiring, request-to-Action delegation).
- Never mock domain logic: no mocking Actions, Rules, Services under test, or Eloquent. Mocks are for external services (mail, HTTP APIs), time/UUID generation when required, and infrastructure adapters. If heavy mocking is required, the design is wrong — fix the design.
- Never test getters/setters, trivial accessors, casts, plain Eloquent relationship definitions, or framework behavior. If a test only proves Laravel works, it should not exist.
- Never test Jobs as business units — assert delegation to the Action and retry/backoff/queue configuration only; do not re-test Action behavior inside Job tests.
- Never name tests `test1`, `handle_test`, `process_product` — the name must state behavior.
