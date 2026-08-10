---
name: labrodev-model
description: "Use when creating, reviewing, or modifying Eloquent models, model Collections, Observers, or migrations in a Labrodev Laravel project — including declaring casts()/relations/$visible, wiring #[Table]/#[ObservedBy]/#[CollectedBy]/#[UsePolicy]/#[UseFactory] attributes, or deciding what logic belongs (or does not belong) in a model."
license: MIT
metadata:
  author: labrodev
---

# Models, Collections, and Observers

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-model` guideline** (musts, must-nots); the per-file checklists are `rules/models.md` and `rules/migrations.md`. This skill holds the craft: anatomy, canonical templates, and edge cases.

Models are persistence objects only: **state + casts + relations**. They are not the domain itself and never coordinate workflows. Every model has a Collection; Observers are optional and handle persistence-adjacent side effects only.

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

/** @mixin IdeHelperBooking */
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

This is the whole file — no `@property` lists, no comments. A model carries exactly two annotation kinds: the single `@mixin IdeHelper{Model}` line (written by ide-helper mixin mode) and the relation `@return` generics. That emptiness is the convention, not an omission.

## PHPStan / IDE metadata (ide-helper mixin mode, never inline)

- Install `barryvdh/laravel-ide-helper` as a dev dependency.
- Generate in **mixin mode**: `php artisan ide-helper:models -M` — writes `_ide_helper_models.php` containing an `IdeHelper{Model}` class per model, and puts the single `/** @mixin IdeHelper{Model} */` line on each model.
- Configure `config/ide-helper.php`: `model_locations` points at the domain models path (e.g. `src/Core/Domain/*/Models` — match the project layout); `write_model_magic_where` off.
- Regenerate after every migration/schema change — a stale mixin is a PHPStan lie.
- `_ide_helper_models.php` stays **committed** (CI needs it for PHPStan via `scanFiles`) and is excluded from Pint via `notPath` → see the labrodev-static-analysis skill.
- The mixin covers column `@property` metadata for PHPStan and the IDE; relation return types are typed inline via the `@return` generics on the relation methods themselves.

**Pre-save trap**: the mixin's `@property string $uuid` (NOT NULL column) describes a *persisted row* — before the first save the attribute is genuinely unset, so PHPStan flags `=== null` guards in `creating()`/`saving()` hooks as dead. Do NOT delete the guard; read via `$model->getAttribute('uuid')` (returns `mixed`) so the guard stays live and PHPStan-clean → see the labrodev-static-analysis skill.

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

Conventions:
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

## Migration template

Migration law (snake_plural tables, `uuid` columns, timestamps/soft deletes, casts kept in sync) → `labrodev-model` guideline.

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
