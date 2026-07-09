---
name: labrodev-testing
description: "Use when writing or reviewing Pest tests in a Labrodev Laravel project: testing Actions, Orchestrators, Rules, Services, or Data validation, adding route-level Feature tests, creating domain-state helpers in tests/Pest.php, or adding/maintaining the Pest architecture test suite that mechanically enforces the playbook."
license: MIT
metadata:
  author: labrodev
---

# Testing: Pest strategy and architecture tests

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

Tests exist to protect behavior, enable refactoring, and make intent explicit — not to satisfy coverage metrics. A good test explains what the system does and fails only when behavior changes. If a test breaks during refactoring but behavior did not change, the test is wrong.

## Musts

- Test behavior, not implementation. Assert resulting state, returned values, and thrown exceptions — never internal method calls or step ordering.
- Follow the pyramid, most valuable first: **(1)** Action/Orchestrator tests, **(2)** Rule/Service/Pipeline unit tests, **(3)** Job tests, **(4)** route-level Feature tests (few, wiring-only).
- Test business behavior in Core — call Actions directly as callables with named arguments (`$bookingCreate(bookingCreateData: $bookingCreateData);`), never through controllers.
- Every Action that mutates state, enforces business rules, or coordinates domain objects gets tests covering the happy path, the unhappy path, and edge cases.
- Unhappy-path tests must match the Action's failure mode: user-fixable violations → assert `ValidationException` is thrown; state-based ineligibility → assert the silent no-op (state unchanged) → failure-mode definitions in the labrodev-action skill.
- Test validation at the Data level with `{Model}{Operation}Data::validateAndCreate([...])` + `ValidationException` assertions — never via Actions or HTTP.
- Construct Data objects explicitly in tests (`BookingCreateData::from([...])`); never pass raw arrays into Actions.
- Build domain state needed as setup **through Core Actions**, via global helpers in `tests/Pest.php` (`Data::from()` + named-argument invocation) — so invariants (UUID assignment, initial status, guarded transitions) hold in fixtures exactly as in production.
- Mirror the domain structure: `tests/Feature/{Domain}/` (e.g. `tests/Feature/Booking/BookingCreateTest.php`).
- Test names are descriptive and behavior-focused: `it('creates a booking with valid data')`, `it('fails when the period is not available')`.
- Keep an architecture test suite (`pest-plugin-arch`, ships with Pest 4) that mechanically enforces the playbook — template below.
- In tests, obtain Actions via `app(BookingCreate::class)` or `new BookingCreate()` — the "no `app()`/`resolve()`" rule applies to controllers, not tests → see the labrodev-controller skill.

## Must-nots

- Never build domain state with raw model factories (`Booking::factory()->create()`) — factories bypass Rules, UUID assignment, and status transitions. Factories are allowed only for framework-level fixtures with no domain invariants (e.g. `User::factory()` for `actingAs()`).
- Never re-test business rules in HTTP tests, and never put complex domain setup in them — they verify wiring only (route connected, auth/authorization wiring, request-to-Action delegation).
- Never mock domain logic: no mocking Actions, Rules, Services under test, or Eloquent. Mocks are for external services (mail, HTTP APIs), time/UUID generation when required, and infrastructure adapters. If heavy mocking is required, the design is wrong — fix the design.
- Never test getters/setters, trivial accessors, casts, plain Eloquent relationship definitions, or framework behavior. If a test only proves Laravel works, it should not exist.
- Never test Jobs as business units — assert delegation to the Action/Orchestrator and retry/backoff/queue configuration only; do not re-test Action behavior inside Job tests.
- Never name tests `test1`, `handle_test`, `process_product` — the name must state behavior.

## What to test at which level

| Component | When to test | Assert |
|---|---|---|
| Action | Always, when it mutates state / enforces rules / coordinates objects | Resulting state changes, return value, thrown `ValidationException` or silent no-op |
| Orchestrator | When it exists | Final workflow outcome, side effects (state, dispatched events) — not internal step order |
| Rule | Always — ideal unit tests | Boolean decisions across edge cases; avoid DB access when possible |
| Service | When logic is non-trivial | Inputs → outputs; pure Services testable without Laravel bootstrapping |
| Pipeline | When step order or payload transforms matter | Final payload state only |
| Data | When validation rules are non-trivial | `ValidationException` on invalid raw input via `validateAndCreate()` |
| Job | When retry/backoff/dispatch matters | Delegation + queue configuration, nothing else |
| Event/Listener | Listeners with important side effects | Correct Action/Orchestrator is called |
| Route (Feature) | Few, high-level | Status/redirect, auth/authorization wiring, record persisted |

## File layout

```
tests/
├── Pest.php                          # test-case binding + global domain-state helpers
├── ArchTest.php                      # architecture enforcement (template below)
└── Feature/
    └── Booking/                      # mirrors Core/Domain/Booking
        ├── BookingCreateTest.php     # Action tests
        ├── BookingCreateDataTest.php # Data validation tests
        └── BookingRouteTest.php      # wiring-only HTTP tests
```

## Template: tests/Pest.php with domain-state helpers

Helper naming pattern: `create{Model}(array $attributes = []): {Model}` — one global helper per aggregate the suite needs. Each helper delegates to the Core create Action so every fixture is a legal domain object.

```php
<?php

declare(strict_types=1);

use Core\Domain\Booking\Actions\BookingCreate;
use Core\Domain\Booking\Data\BookingCreateData;
use Core\Domain\Booking\Models\Booking;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

pest()->extend(TestCase::class)
    ->use(RefreshDatabase::class)
    ->in('Feature');

/**
 * Build domain state through the Core Action — never through raw model
 * factories — so invariants (uuid, initial status, rule guards) hold.
 *
 * @param  array<string, mixed>  $attributes
 */
function createBooking(array $attributes = []): Booking
{
    $bookingCreateData = BookingCreateData::from([
        'reference' => fake()->unique()->numerify('BK-####'),
        'starts_at' => now()->addDay()->toDateTimeString(),
        'ends_at' => now()->addDay()->addHours(2)->toDateTimeString(),
        'guest_count' => 2,
        ...$attributes,
    ]);

    $bookingCreate = app(BookingCreate::class);

    return $bookingCreate(bookingCreateData: $bookingCreateData);
}
```

## Template: Action test (happy / unhappy / edge)

```php
<?php

declare(strict_types=1);

use Core\Domain\Booking\Actions\BookingCancel;
use Core\Domain\Booking\Actions\BookingCreate;
use Core\Domain\Booking\Actions\BookingUpdate;
use Core\Domain\Booking\Data\BookingCreateData;
use Core\Domain\Booking\Data\BookingUpdateData;
use Core\Domain\Booking\Enums\BookingStatus;
use Core\Domain\Booking\Models\Booking;
use Illuminate\Validation\ValidationException;

it('creates a booking with valid data', function (): void {
    $bookingCreateData = BookingCreateData::from([
        'reference' => 'BK-1001',
        'starts_at' => now()->addDay()->toDateTimeString(),
        'ends_at' => now()->addDay()->addHours(2)->toDateTimeString(),
        'guest_count' => 2,
    ]);

    $bookingCreate = app(BookingCreate::class);

    $booking = $bookingCreate(bookingCreateData: $bookingCreateData);

    expect($booking)->toBeInstanceOf(Booking::class)
        ->and($booking->uuid)->not->toBeEmpty()   // UUID assigned by the create Action
        ->and($booking->status)->toBe(BookingStatus::Pending);

    $this->assertDatabaseHas('bookings', ['uuid' => $booking->uuid]);
});

// Failure mode 1: user-fixable violation → ValidationException.
it('fails when the period is not available', function (): void {
    createBooking(attributes: [
        'starts_at' => now()->addDay()->toDateTimeString(),
        'ends_at' => now()->addDay()->addHours(2)->toDateTimeString(),
    ]);

    $bookingCreateData = BookingCreateData::from([
        'reference' => 'BK-2002',
        'starts_at' => now()->addDay()->toDateTimeString(),
        'ends_at' => now()->addDay()->addHours(2)->toDateTimeString(),
        'guest_count' => 2,
    ]);

    $bookingCreate = app(BookingCreate::class);

    expect(fn (): Booking => $bookingCreate(bookingCreateData: $bookingCreateData))
        ->toThrow(ValidationException::class);
});

// Failure mode 2: state-based ineligibility → silent no-op.
it('leaves a cancelled booking unchanged on update', function (): void {
    $booking = createBooking();

    $bookingCancel = app(BookingCancel::class);
    $bookingCancel(booking: $booking);           // precondition built via an Action too

    $bookingUpdateData = BookingUpdateData::from([
        'reference' => 'BK-CHANGED',
        'starts_at' => $booking->starts_at->toDateTimeString(),
        'ends_at' => $booking->ends_at->toDateTimeString(),
        'guest_count' => 4,
    ]);

    $bookingUpdate = app(BookingUpdate::class);
    $bookingUpdate(booking: $booking, bookingUpdateData: $bookingUpdateData);

    expect($booking->refresh()->reference)->not->toBe('BK-CHANGED');
});
```

## Template: Data validation test

`::from()` skips validation; `::validateAndCreate()` runs `rules()`. Validation tests must use `validateAndCreate()` with raw input keys.

```php
<?php

declare(strict_types=1);

use Core\Domain\Booking\Data\BookingCreateData;
use Illuminate\Validation\ValidationException;

it('rejects a start time in the past', function (): void {
    expect(fn () => BookingCreateData::validateAndCreate([
        'reference' => 'BK-1001',
        'starts_at' => now()->subDay()->toDateTimeString(),
        'ends_at' => now()->addHours(2)->toDateTimeString(),
        'guest_count' => 2,
    ]))->toThrow(ValidationException::class);
});

it('rejects a zero guest count', function (): void {
    expect(fn () => BookingCreateData::validateAndCreate([
        'reference' => 'BK-1001',
        'starts_at' => now()->addDay()->toDateTimeString(),
        'ends_at' => now()->addDay()->addHours(2)->toDateTimeString(),
        'guest_count' => 0,
    ]))->toThrow(ValidationException::class);
});
```

## Template: route Feature test (wiring only)

```php
<?php

declare(strict_types=1);

use App\Models\User; // framework fixture — factory is fine here

it('stores a booking through the dashboard route', function (): void {
    $user = User::factory()->create();

    $response = $this->actingAs($user)->post(route('bookings.store'), [
        'reference' => 'BK-1001',
        'starts_at' => now()->addDay()->toDateTimeString(),
        'ends_at' => now()->addDay()->addHours(2)->toDateTimeString(),
        'guest_count' => 2,
    ]);

    $response->assertRedirect(route('bookings.index'));
    $this->assertDatabaseHas('bookings', ['reference' => 'BK-1001']);
});

it('shows a booking by uuid', function (): void {
    $user = User::factory()->create();
    $booking = createBooking();

    $this->actingAs($user)
        ->get(route('bookings.show', ['booking' => $booking->uuid]))
        ->assertOk();
});

it('forbids storing a booking without permission', function (): void {
    $user = User::factory()->create(); // no permission granted

    $this->actingAs($user)
        ->post(route('bookings.store'), [])
        ->assertForbidden();
});
```

Routes bind `{model:uuid}`, so route parameters take the model's uuid → controller/route anatomy in the labrodev-controller skill; the permission contract behind the 403 → see the labrodev-authorization skill. That's the whole HTTP suite for a domain: connected, delegating, guarded. Nothing more.

## Template: architecture tests (paste-ready)

`tests/ArchTest.php` — pest-plugin-arch ships with Pest 4, no extra dependency. These tests turn playbook conventions into CI failures. Adjust the `Core` prefix if Core lives in a package namespace, and extend the `ignoring()` lists as the legacy zone dictates → zone definition in the labrodev-core skill.

```php
<?php

declare(strict_types=1);

arch('no debug or die calls anywhere')
    ->expect(['dd', 'dump', 'ray', 'var_dump', 'die', 'exit'])
    ->not->toBeUsed();

arch('core declares strict types everywhere')
    ->expect('Core')
    ->toUseStrictTypes();

arch('the app layer declares strict types everywhere')
    ->expect('App')
    ->toUseStrictTypes();

arch('core classes are final')
    ->expect('Core')
    ->classes()
    ->toBeFinal()
    ->ignoring([
        'Core\Shared\Models\BaseModel', // abstract shared bases are the only exception
    ]);

arch('core never imports the delivery layer')
    ->expect('Core')
    ->not->toUse('App\Layer');

arch('core never depends on legacy starter code')
    ->expect('Core')
    ->not->toUse(['App\Http', 'App\Actions\Fortify']);

arch('no form requests outside the legacy zone')
    ->expect('Illuminate\Foundation\Http\FormRequest')
    ->not->toBeUsed()
    ->ignoring('App\Http'); // legacy vendor/starter zone → labrodev-core skill

arch('the delivery layer never writes or validates on its own')
    ->expect('App\Layer')
    ->not->toUse([
        'Illuminate\Support\Facades\DB',        // transactions/writes live in Core Actions
        'Illuminate\Support\Facades\Validator', // validation lives in Data classes
    ]);

// Add one block per {Layer}/{Domain} controllers namespace as domains appear:
arch('dashboard booking controllers are final invokables')
    ->expect('App\Layer\Dashboard\Booking\Controllers')
    ->classes()
    ->toBeFinal()
    ->toBeInvokable();
```

## Edge cases

- **Helper collisions**: default helper values must survive repeated calls in one test (unique constraints) — use `fake()->unique()` for identifying fields and pass `$attributes` overrides for scenario-specific state.
- **Preconditions beyond create**: build them by chaining Actions (`createBooking()` then `$bookingCancel(booking: $booking)`), never by writing model attributes directly in the test.
- **Time-sensitive Rules**: freeze time with `$this->travelTo(...)` rather than widening date assertions; datetime conventions (`Illuminate\Support\Carbon`, no `CarbonImmutable`) → see the labrodev-core skill.
- **Queued side effects**: `Queue::fake()` in Action tests to assert a Job was pushed; the Job's own test asserts delegation and retry/backoff config only.
- **Events**: `Event::fake()` to assert a fact was dispatched; do not fake events when a listener's side effect is the behavior under test.
- **Pure Rules/Services**: when they take plain values or in-memory models, test them without the database — build models via `new Booking()` with explicit attributes *only* for pure in-memory checks that never persist; anything persisted goes through Actions.
- **Testing Data with relations**: pass resolved model instances directly to `::from(['service' => $service, ...])` — caster pass-through makes this work; caster behavior → see the labrodev-data skill.
- **Orchestrator tests**: assert the final outcome and observable side effects; asserting intermediate step order is allowed only when behavior depends on it.

## Cross-skill pointers

- Action anatomy, the two failure modes, transactions, UUID assignment → see the labrodev-action skill.
- Data classes, `rules()`, `prepareForPipeline()`, UUID casters → see the labrodev-data skill.
- Controller anatomy, routes, `{model:uuid}` binding, `Inertia::flash` + `to_route()` → see the labrodev-controller skill.
- Policies and the permission-constant contract behind authorization assertions → see the labrodev-authorization skill.
- Naming rules and named-argument invocation style used in every test → see the labrodev-naming skill.
- Pint/PHPStan/Rector and when the suite runs in the workflow → see the labrodev-workflow skill.

## Review checklist

1. Is every state-mutating Action covered by direct Action tests (happy, unhappy, edge), invoked as a callable with named arguments — never through a controller?
2. Do unhappy-path tests match the Action's failure mode — `toThrow(ValidationException::class)` for user-fixable violations, unchanged-state assertions for silent no-ops?
3. Are validation failures tested at the Data level with `validateAndCreate()` and raw input keys — not via Actions or HTTP?
4. Is all persisted domain state in setup built through Core Actions (via `tests/Pest.php` helpers), with factories reserved for framework fixtures like `User`?
5. Are route Feature tests few and wiring-only (status/redirect, auth/authorization, persistence) with no business-rule assertions or complex domain setup?
6. Is mocking limited to external services, time/UUID, and infrastructure — no mocked Actions, Rules, or Services under test?
7. Does `tests/ArchTest.php` exist and pass — strict types everywhere, final classes, Core never imports `App\Layer`, no FormRequest outside the legacy zone, no DB/Validator facades in the delivery layer?
8. Are Jobs tested as wrappers (delegation + retry/backoff config) and never as business units?
9. Are there zero tests that merely prove Laravel works (relations, casts, accessors)?
10. Do all test names read as behavior statements, and do test files mirror `tests/Feature/{Domain}/`?
