---
name: labrodev-testing
description: "Use when writing or reviewing Pest tests in a Labrodev Laravel project: Unit tests mirroring the code tree (Actions, Rules, Services, Data validation), end-to-end Feature tests through controllers, domain-state helpers in tests/Pest.php, or the mandatory Pest architecture test suite that mechanically enforces the playbook."
license: MIT
metadata:
  author: labrodev
---

# Testing: Pest strategy and architecture tests

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-testing` guideline** (musts, must-nots); the per-file checklist is `rules/tests.md`. This skill holds the craft: anatomy, canonical templates, and edge cases.

## What to test at which level

| Component | When to test | Assert |
|---|---|---|
| Action | Always, when it mutates state / enforces rules / coordinates objects | Resulting state changes, return value, thrown **named domain exception** on failed guards |
| Pipeline-orchestrating Service | When it exists | Final workflow outcome, side effects (state, dispatched events) — not internal step order |
| Rule | Always — ideal unit tests | Boolean decisions across edge cases; avoid DB access when possible |
| Service | When logic is non-trivial | Inputs → outputs; pure Services testable without Laravel bootstrapping |
| Pipeline | When step order or payload transforms matter | Final payload state only |
| Data | When validation rules are non-trivial | `ValidationException` on invalid raw input via `validateAndCreate()` |
| Job | When retry/backoff/dispatch matters | Delegation + queue configuration, nothing else |
| Event/Listener | Listeners with important side effects | Correct Action is called |
| Controller (Feature) | Few, high-level, end-to-end | Status/redirect, auth/authorization wiring (403s from the can-trio), record persisted |

## File layout — two suites mirroring the code tree

`tests/Unit/**` mirrors the full path of the class under test; `tests/Feature/**` mirrors the delivery structure and holds end-to-end scenarios through controllers. Every folder and subfolder aligns with the code it tests.

```
tests/
├── Pest.php                                                # suite binding + global domain-state helpers
├── ArchTest.php                                            # architecture enforcement (template below)
├── Unit/
│   ├── Core/Domain/Booking/
│   │   ├── Actions/BookingCreateTest.php                   # Action tests
│   │   ├── Rules/BookingRuleTest.php                       # Rule tests
│   │   └── Data/BookingDataTest.php                        # Data validation tests
│   └── App/Layer/Dashboard/Booking/
│       └── ViewModels/BookingIndexViewModelTest.php
└── Feature/
    └── App/Layer/Dashboard/Booking/
        └── BookingTest.php                                 # end-to-end controller scenarios
```

## Template: tests/Pest.php with domain-state helpers

Helper naming pattern: `create{Model}(array $attributes = []): {Model}` — one global helper per aggregate the suite needs. Each helper delegates to the Core create Action so every fixture is a legal domain object.

```php
<?php

declare(strict_types=1);

use Core\Domain\Booking\Actions\BookingCreate;
use Core\Domain\Booking\Data\BookingData;
use Core\Domain\Booking\Models\Booking;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Carbon;
use Tests\TestCase;

pest()->extend(TestCase::class)
    ->use(RefreshDatabase::class)
    ->in('Feature', 'Unit');

/**
 * Build domain state through the Core Action — never through raw model
 * factories — so invariants (uuid, initial status, rule guards) hold.
 *
 * @param  array<string, mixed>  $attributes
 */
function createBooking(array $attributes = []): Booking
{
    $bookingData = BookingData::from([
        'reference' => fake()->unique()->numerify('BK-####'),
        'starts_at' => Carbon::now()->addDay()->toDateTimeString(),
        'ends_at' => Carbon::now()->addDay()->addHours(2)->toDateTimeString(),
        'guest_count' => 2,
        ...$attributes,
    ]);

    $bookingCreate = app(BookingCreate::class);

    return $bookingCreate(bookingData: $bookingData);
}
```

## Template: Action test (happy / unhappy / edge)

```php
<?php

declare(strict_types=1);

use Core\Domain\Booking\Actions\BookingCreate;
use Core\Domain\Booking\Data\BookingData;
use Core\Domain\Booking\Enums\BookingStatus;
use Core\Domain\Booking\Exceptions\BookingPeriodUnavailableException;
use Core\Domain\Booking\Models\Booking;
use Illuminate\Support\Carbon;

it('creates a booking with valid data', function (): void {
    $bookingData = BookingData::from([
        'reference' => 'BK-1001',
        'starts_at' => Carbon::now()->addDay()->toDateTimeString(),
        'ends_at' => Carbon::now()->addDay()->addHours(2)->toDateTimeString(),
        'guest_count' => 2,
    ]);

    $bookingCreate = app(BookingCreate::class);

    $booking = $bookingCreate(bookingData: $bookingData);

    expect($booking)->toBeInstanceOf(Booking::class)
        ->and($booking->uuid)->not->toBeEmpty()   // UUID assigned by the create Action
        ->and($booking->status)->toBe(BookingStatus::Pending);

    $this->assertDatabaseHas('bookings', ['uuid' => $booking->uuid]);
});

// Failed business gate → the named domain exception. Never ValidationException.
it('fails when the period is not available', function (): void {
    createBooking(attributes: [
        'starts_at' => Carbon::now()->addDay()->toDateTimeString(),
        'ends_at' => Carbon::now()->addDay()->addHours(2)->toDateTimeString(),
    ]);

    $bookingData = BookingData::from([
        'reference' => 'BK-2002',
        'starts_at' => Carbon::now()->addDay()->toDateTimeString(),
        'ends_at' => Carbon::now()->addDay()->addHours(2)->toDateTimeString(),
        'guest_count' => 2,
    ]);

    $bookingCreate = app(BookingCreate::class);

    expect(fn (): Booking => $bookingCreate(bookingData: $bookingData))
        ->toThrow(BookingPeriodUnavailableException::class);
});
```

State-based ineligibility (the can-trio) is **not** an Action concern any more — it is enforced at Policy level. Cover it with a Rule unit test plus a Feature test asserting 403 (template below).

## Template: Rule unit test

`tests/Unit/Core/Domain/Booking/Rules/BookingRuleTest.php` — pure in-memory checks, no database:

```php
<?php

declare(strict_types=1);

use Core\Domain\Booking\Enums\BookingStatus;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Rules\BookingRule;

it('allows updating a draft booking', function (): void {
    $booking = new Booking();
    $booking->status = BookingStatus::Draft;

    expect(BookingRule::canUpdate($booking))->toBeTrue();
});

it('forbids removing a confirmed booking', function (): void {
    $booking = new Booking();
    $booking->status = BookingStatus::Confirmed;

    expect(BookingRule::canRemove($booking))->toBeFalse();
});
```

## Template: Data validation test

`::from()` skips validation; `::validateAndCreate()` runs `rules()`. Validation tests must use `validateAndCreate()` with raw input keys.

```php
<?php

declare(strict_types=1);

use Core\Domain\Booking\Data\BookingData;
use Illuminate\Support\Carbon;
use Illuminate\Validation\ValidationException;

it('rejects a start time in the past', function (): void {
    expect(fn () => BookingData::validateAndCreate([
        'reference' => 'BK-1001',
        'starts_at' => Carbon::now()->subDay()->toDateTimeString(),
        'ends_at' => Carbon::now()->addHours(2)->toDateTimeString(),
        'guest_count' => 2,
    ]))->toThrow(ValidationException::class);
});

it('rejects a zero guest count', function (): void {
    expect(fn () => BookingData::validateAndCreate([
        'reference' => 'BK-1001',
        'starts_at' => Carbon::now()->addDay()->toDateTimeString(),
        'ends_at' => Carbon::now()->addDay()->addHours(2)->toDateTimeString(),
        'guest_count' => 0,
    ]))->toThrow(ValidationException::class);
});
```

## Template: Feature test (end-to-end through controllers)

`tests/Feature/App/Layer/Dashboard/Booking/BookingTest.php` — the scenario enters through HTTP, exercises the controller → Data → Action chain, and asserts the response plus persisted state. The 403 case is where Policy-level can-trio enforcement gets covered.

```php
<?php

declare(strict_types=1);

use App\Models\User;
use Illuminate\Support\Carbon;

it('stores a booking through the dashboard route', function (): void {
    $user = User::factory()->create();

    $response = $this->actingAs($user)->post(route('bookings.store'), [
        'reference' => 'BK-1001',
        'starts_at' => Carbon::now()->addDay()->toDateTimeString(),
        'ends_at' => Carbon::now()->addDay()->addHours(2)->toDateTimeString(),
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
    $user = User::factory()->create();

    $this->actingAs($user)
        ->post(route('bookings.store'), [])
        ->assertForbidden();
});
```

`App\Models\User` with its factory is the framework fixture for `actingAs()` — the one sanctioned factory use; the last test's user simply carries no permission, which is what makes the 403 assertion meaningful. Routes bind `{model:uuid}`, so route parameters take the model's uuid → controller/route anatomy in the labrodev-controller skill; the permission contract behind the 403 → see the labrodev-authorization skill. That's the whole HTTP suite for a domain: connected, delegating, guarded. Nothing more.

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

arch('core never imports the app side at all')
    ->expect('Core')
    ->not->toUse('App'); // includes App\Layer, App\Http, App\Models\User — Core uses its mirror User model

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

arch('actions never throw validation exceptions')
    ->expect('Core\Domain')
    ->not->toUse('Illuminate\Validation\ValidationException'); // business gates throw named domain exceptions

// Add one block per {Layer}/{Domain} controllers namespace as domains appear:
arch('dashboard booking controllers are final invokables')
    ->expect('App\Layer\Dashboard\Booking\Controllers')
    ->classes()
    ->toBeFinal()
    ->toBeInvokable();
```

## Edge cases

- **Helper collisions**: default helper values must survive repeated calls in one test (unique constraints) — use `fake()->unique()` for identifying fields and pass `$attributes` overrides for scenario-specific state.
- **Preconditions beyond create**: build them by chaining Actions (`createBooking()` then `$bookingStatusUpdate(booking: $booking, bookingStatus: BookingStatus::Cancelled)`), never by writing model attributes directly in the test.
- **Time-sensitive Rules**: freeze time with `$this->travelTo(...)` rather than widening date assertions; datetime conventions (`Illuminate\Support\Carbon`, no `CarbonImmutable`) → see the labrodev-core skill.
- **Queued side effects**: `Queue::fake()` in Action tests to assert a Job was pushed; the Job's own test asserts delegation and retry/backoff config only.
- **Events**: `Event::fake()` to assert a fact was dispatched; do not fake events when a listener's side effect is the behavior under test.
- **Pure Rules/Services**: when they take plain values or in-memory models, test them without the database — build models via `new Booking()` with explicit attributes *only* for pure in-memory checks that never persist; anything persisted goes through Actions.
- **Testing Data with relations**: pass resolved model instances directly to `::from(['service' => $service, ...])` — caster pass-through makes this work; caster behavior → see the labrodev-data skill.
- **Pipeline-orchestrating Service tests**: assert the final outcome and observable side effects; asserting intermediate step order is allowed only when behavior depends on it.

## Cross-skill pointers

- Action anatomy, gate levels (Policy can-trio vs in-Action invariant guards throwing domain exceptions), transactions, UUID assignment → see the labrodev-action skill.
- Data classes, `rules()`, `prepareForPipeline()`, UUID casters → see the labrodev-data skill.
- Controller anatomy, routes, `{model:uuid}` binding, `Inertia::flash` + `to_route()` → see the labrodev-controller skill.
- Policies and the permission-constant contract behind authorization assertions → see the labrodev-authorization skill.
- Naming rules and named-argument invocation style used in every test → see the labrodev-naming skill.
- Pint/PHPStan/Rector configuration and the Rector → Pint → PHPStan gate → see the labrodev-static-analysis skill.
