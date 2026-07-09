---
name: labrodev-model
description: "Use when creating, reviewing, or modifying Eloquent models, model Collections, Observers, or migrations in a Labrodev Laravel project — including declaring casts()/relations/$visible, wiring #[Table]/#[ObservedBy]/#[CollectedBy]/#[UsePolicy]/#[UseFactory] attributes, or deciding what logic belongs (or does not belong) in a model."
license: MIT
metadata:
  author: labrodev
---

# Models, Collections, and Observers

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

Models are persistence objects only: **state + casts + relations**. They are not the domain itself and never coordinate workflows. Every model has a Collection; Observers are optional and handle persistence-adjacent side effects only.

## Musts

- Every domain model lives in `Core/Domain/{Domain}/Models/{Model}.php` and **extends `Core/Shared/Models/BaseModel`**.
- `BaseModel` sets `$guarded = ['*']` — mass assignment is forbidden architecture-wide. Attributes are assigned explicitly, row by row, in Actions (never `fill()` / `create()`).
- Declare the table with the **`#[Table('actual_table')]` class attribute** when the table name is not Laravel's default snake_plural — never `protected $table`.
- Wire observer, collection, policy, and factory with **class attributes** in this conventional order: `#[ObservedBy]`, `#[CollectedBy]`, `#[UsePolicy]`, `#[UseFactory]`. Omit `#[UseFactory]` (and its import) until the factory exists.
- **No `@property` lines and no comments in models.** Column/attribute metadata for PHPStan/IDE comes from **barryvdh/laravel-ide-helper** generated mixins (`php artisan ide-helper:models --nowrite` → `_ide_helper_models.php`), regenerated after every schema change — never written into the model file. No class docblock, no human comments.
- Casts go in the **`casts()` method — never the `$casts` property**. Cast every field that needs it: enums, `datetime`, `float`, `int`, `array` (JSON). Every enum backing a model field MUST be cast here.
- Every relation method declares the correct native Relation return type AND carries the PHPStan generics docblock (`@return BelongsTo<User, $this>`) — this is the ONE docblock kind allowed in a model.
- Every model defines **`$visible` explicitly** to control serialization. `BaseModel` hides audit columns via `$hidden`; concrete models use `$visible`, not `$hidden` overrides.
- Every model has a corresponding `{Model}Collection` in `Core/Domain/{Domain}/Collections/` and declares it with `#[CollectedBy(...)]`.
- Observers, when present, live in `Core/Domain/{Domain}/Observers/{Model}Observer.php`, are `final readonly`, and are wired only via `#[ObservedBy(...)]`.
- Traits on models must contain persistence-level logic only (actor/audit metadata, soft deletes); domain-specific traits live in the domain and are named accordingly.

## Must-nots

- Never define `$fillable`. Never use `$model->fill()` or `Model::create()`.
- Never write `@property`/`@property-read` lists, class docblocks, or explanatory comments in a model file. `ide-helper:models --write` (writing into model files) is equally forbidden — column metadata lives in the generated mixin file only. (Relation `@return` generics are the sole exception and are required.)
- Never put business workflows, queries, scopes, cross-entity coordination, complex calculations, or UI formatting in a model. → see the labrodev-action skill (workflows) and the labrodev-query skill (reads).
- Never generate UUIDs in the model, in `boot()`, or via a trait. UUIDs are assigned explicitly in the create Action → see the labrodev-action skill.
- Never override `getRouteKeyName()` for URL binding. Keep the default route key; routes bind by UUID with `{booking:uuid}`. → see the labrodev-controller skill.
- Never register observers from `boot()` via `Model::observe()` — it causes recursive boot. `#[ObservedBy]` is the only wiring.
- Never override `newCollection()` — `#[CollectedBy]` replaces it (Laravel 13+).
- Object-state gates live in `{Model}Rule`, never on the model → see the labrodev-action skill.
- Observers must not change domain state, call mutating Actions/Services, or contain logic the main business flow depends on.

Class/method/variable naming rules → see the labrodev-naming skill.
File header contract (`declare(strict_types=1)`, `final`) and dependency direction → see the labrodev-core skill.

## BaseModel (shared, one per project)

Location: `Core/Shared/Models/BaseModel.php`. All domain models extend it.

Responsibilities: blocks mass assignment for every model (`$guarded = ['*']`), centralizes hidden fields, enables `Model::factory()` for models declaring `#[UseFactory]`. Business logic never lives here; concrete models use `$visible` instead of overriding `$hidden`.

```php
<?php

declare(strict_types=1);

namespace Core\Shared\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

abstract class BaseModel extends Model
{
    /** @use HasFactory<\Illuminate\Database\Eloquent\Factories\Factory> */
    use HasFactory;

    protected $guarded = ['*'];

    protected $hidden = [
        'created_by',
        'updated_by',
        'deleted_at',
        'created_at',
        'updated_at',
    ];
}
```

(The single `@use HasFactory` generic on the shared base is the only annotation allowed — it cannot be generated and lives in one file, not in each model.)

## Model template

Naming pattern: `Core/Domain/{Domain}/Models/{Model}.php` — singular model name, e.g. `Booking` in `Core/Domain/Booking/Models/Booking.php`.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Models;

use App\Models\User;
use Core\Domain\Booking\Collections\BookingCollection;
use Core\Domain\Booking\Enums\BookingStatus;
use Core\Domain\Booking\Factories\BookingFactory;
use Core\Domain\Booking\Observers\BookingObserver;
use Core\Domain\Booking\Policies\BookingPolicy;
use Core\Shared\Models\BaseModel;
use Illuminate\Database\Eloquent\Attributes\CollectedBy;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;
use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Attributes\UseFactory;
use Illuminate\Database\Eloquent\Attributes\UsePolicy;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

#[Table('bookings')]
#[ObservedBy(BookingObserver::class)]
#[CollectedBy(BookingCollection::class)]
#[UsePolicy(BookingPolicy::class)]
#[UseFactory(BookingFactory::class)]
final class Booking extends BaseModel
{
    protected $visible = [
        'uuid',
        'number',
        'status',
        'guests_count',
        'total',
        'confirmed_at',
    ];

    protected function casts(): array
    {
        return [
            'status' => BookingStatus::class,
            'guests_count' => 'int',
            'total' => 'float',
            'confirmed_at' => 'datetime',
        ];
    }

    /**
     * @return BelongsTo<User, $this>
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    /**
     * @return HasMany<BookingItem, $this>
     */
    public function items(): HasMany
    {
        return $this->hasMany(BookingItem::class, 'booking_id');
    }
}
```

This is the whole file — no class docblock, no `@property` lists, no comments. The relation `@return` generics are the only docblocks a model carries. That emptiness is the convention, not an omission.

## PHPStan / IDE metadata (ide-helper, never inline)

- Install `barryvdh/laravel-ide-helper` as a dev dependency.
- Generate model metadata into the mixin file, never into models: `php artisan ide-helper:models --nowrite` (writes `_ide_helper_models.php` with `@mixin` metadata for every model).
- Regenerate after every migration/schema change — a stale mixin is a PHPStan lie.
- The mixin covers column `@property` metadata for PHPStan and the IDE; relation return types are typed inline via the `@return` generics on the relation methods themselves.
- Never run `ide-helper:models --write` (it injects docblocks into model files) and never hand-write `@property` lists — both violate the clean-model rule.

Notes:
- Declare only real relations — do not scaffold placeholder relation methods.
- Accessors/mutators are allowed only when they are genuinely persistence-level (never business workflows).
- The `User` model is the vendor-starter exception living in `App\Models` — legacy two-zone policy → see the labrodev-core skill.
- Enum cast is one leg of the full enum contract (label(), Rule::enum, frontend value+label) → see the labrodev-enum skill.

## Collection template

Naming pattern: `Core/Domain/{Domain}/Collections/{Model}Collection.php` — one per model, always, even if initially empty.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Collections;

use Core\Domain\Booking\Enums\BookingStatus;
use Core\Domain\Booking\Models\Booking;
use Illuminate\Database\Eloquent\Collection;

/**
 * @extends Collection<int,Booking>
 */
final class BookingCollection extends Collection
{
    // Collection-level logic only: aggregations, sorting/filtering,
    // derived fields across multiple in-memory models.
    // Methods must be READ-ONLY — never mutate persisted state.

    public function confirmed(): static
    {
        return $this->filter(
            fn (Booking $booking): bool => $booking->status === BookingStatus::Confirmed,
        )->values();
    }

    public function totalAmount(): float
    {
        return (float) $this->sum(
            fn (Booking $booking): float => $booking->total,
        );
    }
}
```

Rules:
- Filtering helpers return `static` and end with `->values()` to re-index keys.
- Methods operate only on in-memory models and express business intent by name.
- Do not put query building here → see the labrodev-query skill.

## Observer template

Naming pattern: `Core/Domain/{Domain}/Observers/{Model}Observer.php`, class is `final readonly`. Create one only when there is a real side effect to run; logging lifecycle events is recommended by default.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Observers;

use Core\Domain\Booking\Models\Booking;
use Illuminate\Support\Facades\Log;

final readonly class BookingObserver
{
    // Persistence-adjacent side effects only:
    // - logging (recommended by default)
    // - cache invalidation
    // - emitting events handled asynchronously (queues/listeners)
    //
    // Anything here must be safe to delay, retry, or fail without
    // affecting the user-facing or core business flow.

    public function created(Booking $booking): void
    {
        Log::channel('observer')->info(sprintf(
            '%s %s has been created by %s (%s)',
            Booking::class,
            $booking->uuid ?? $booking->id,
            auth()->user()->name ?? 'system',
            auth()->user()->id ?? 'noid',
        ));
    }

    public function updated(Booking $booking): void
    {
        Log::channel('observer')->info(sprintf(
            '%s %s has been updated by %s (%s)',
            Booking::class,
            $booking->uuid ?? $booking->id,
            auth()->user()->name ?? 'system',
            auth()->user()->id ?? 'noid',
        ));

        // If emitting events, they must be non-blocking and asynchronous:
        // event(new BookingUpdated($booking));
    }

    public function deleted(Booking $booking): void
    {
        Log::channel('observer')->info(sprintf(
            '%s %s has been deleted by %s (%s)',
            Booking::class,
            $booking->uuid ?? $booking->id,
            auth()->user()->name ?? 'system',
            auth()->user()->id ?? 'noid',
        ));
    }
}
```

If observer logic becomes workflow-like (creates/cancels/approves things, mutates other entities, is required for the main flow to succeed), it belongs in Actions (or a pipeline-orchestrating Service for staged workflows → see the labrodev-pipeline skill) → see the labrodev-action skill.

## Migrations basics

- Clear, singular model names; consistent snake_plural table names (`Booking` → `bookings`).
- Add a `uuid` column (unique, indexed) on any table whose model appears in URLs or needs a stable external identifier — the value is assigned in the create Action, not by the database or model.
- Keep schema predictable: no implicit behavior, no magic columns. Every column added must be mirrored in `casts()` when applicable, and the ide-helper mixin regenerated.
- Include `timestamps()`; add `softDeletes()` when the model uses `SoftDeletes`.

```php
Schema::create('bookings', function (Blueprint $table): void {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->string('number')->nullable();
    $table->unsignedTinyInteger('status'); // column type must match the enum backing type
    $table->unsignedInteger('guests_count')->default(0);
    $table->decimal('total', 10, 2)->default(0);
    $table->timestamp('confirmed_at')->nullable();
    $table->foreignId('user_id')->nullable()->constrained();
    $table->timestamps();
    $table->softDeletes();
});
```

## Edge cases

- **No factory yet**: omit `#[UseFactory(...)]` and the factory import entirely; add both once the factory exists (needed for tests/seeders).
- **Default table name matches**: `#[Table]` is only required when the table differs from Laravel's snake_plural default; adding it anyway for explicitness is acceptable, `protected $table` never is.
- **No observer needed**: omit `#[ObservedBy]`; do not create empty observers. The Collection, by contrast, is mandatory for every model.
- **Pivot/morph relations**: same rules — correct native Relation return type plus the generics docblock (`@return BelongsToMany<Role, $this>`, `@return MorphMany<Comment, $this>`).
- **Typed collection in signatures**: query results for `Booking` are `BookingCollection` thanks to `#[CollectedBy]`; type hints downstream (outside the model) should use `BookingCollection`, not the generic `Collection`.

## Review checklist

1. Does the model extend `BaseModel` and contain only state, `casts()`, and relations — no workflows, queries, or scopes?
2. Are `#[Table]`, `#[ObservedBy]`, `#[CollectedBy]`, `#[UsePolicy]` (and `#[UseFactory]` when a factory exists) declared as class attributes, with no `protected $table`, no `boot()` wiring, no `newCollection()` override?
3. Is the model free of `@property` lists, class docblocks, and comments — with column metadata delegated to the regenerated ide-helper mixin?
4. Are casts declared in the `casts()` method (never `$casts`), covering every enum, date, float, int, and JSON field?
5. Does every relation method declare the correct native Relation return type plus the `@return` generics docblock?
6. Is `$visible` explicitly defined, with no `$fillable` anywhere?
7. Does a `{Model}Collection` exist with `@extends Collection<int,{Model}>`, read-only helpers returning `static` with `->values()`?
8. Is the observer (if any) `final readonly` and limited to logging, cache invalidation, or async events — nothing the main flow depends on?
9. Is UUID generation absent from the model, `boot()`, and traits (assigned in the create Action instead), and is `getRouteKeyName()` not overridden?
10. Does the migration match the model (`uuid` unique column where needed, timestamps, soft deletes when used, casts covering new columns) — and was the ide-helper mixin regenerated after the schema change?
