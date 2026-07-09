---
name: labrodev-enum
description: "Use when creating, reviewing, or naming PHP enums in a Labrodev Laravel project (BookingStatus, ProductType, OrderState — any status/type/mode field with a finite value set), or when wiring an enum across a boundary: label() presentation helpers, Rule::enum validation in Data classes, enum casts in a model's casts(), value + *_label emission in Resources, or EnumMapper select/filter options in ViewModels."
license: MIT
metadata:
  author: labrodev
---

# Enums (the full enum contract)

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

Enums represent constrained domain values: statuses, types, modes — any field with a finite allowed set and matching logic. When an enum crosses a boundary, the **full enum contract** applies. It has four parts, and all four are mandatory:

| # | Boundary | Rule |
|---|----------|------|
| 1 | Enum class | Backed scalar enum, model-prefixed name, `label(): string` wrapping `trans()` |
| 2 | Data class (write side) | Typed enum property (implicit Spatie cast) + `Rule::enum(EnumClass::class)` |
| 3 | Model (persistence) | Enum cast declared in the `casts()` **method** |
| 4 | Frontend (Inertia props) | Resources emit `value` + `*_label` pairs; ViewModels build options via `EnumMapper::keyValues()` |

## Musts

- Enums live in `Core\Domain\{Domain}\Enums` and are always **backed** enums with a scalar backing type (`int` or `string`). The backing type is chosen for domain meaning and persistence needs, not storage convenience.
- If an enum corresponds to a specific model, its name MUST start with the model name: `BookingStatus`, `ProductType`, `OrderState`. Generic names (`Status`, `Type`, `State`) are forbidden.
- The presentation helper is ALWAYS named `label(): string` and wraps `trans()`. EnumMapper and Resources rely on this exact name (`'status_label' => $model->status->label()`).
- `label()` uses an exhaustive `match ($this)` over all cases — no `default` arm. A new case must fail loudly until it gets a label.
- All enum helper methods (`label()`, and optional `color()`, `icon()`, `shortLabel()`) must be **pure**: no side effects, no Actions/Services/Jobs, no external APIs, no business workflows.
- Static domain lookups are allowed as named constructors wrapping `tryFrom()` — they return `?self` and never throw.
- Data classes type-hint the enum directly (`public BookingStatus $status`); Spatie Data casts the raw scalar implicitly — no `#[WithCast]` needed in the common case. The same raw input key is validated with `Rule::enum(BookingStatus::class)` — ALWAYS.
- Every enum that backs a model field MUST be cast in that model's `casts()` method: `'status' => BookingStatus::class`.
- Resources emit enum fields as value + label pairs: `'status' => $this->status->value`, `'status_label' => $this->status->label()`. Labels are resolved on the backend.
- Select/filter option maps are built in ViewModels with `EnumMapper::keyValues(BookingStatus::cases(), 'label')` (package `labrodev/php-enum-mapper`) — never hardcoded in the frontend.
- The frontend renders `*_label` for display and uses the raw `value` for logic/filters; TypeScript literal unions mirror the enum values.
- Unknown enum values on the frontend must show an explicit fallback (the raw value), never silently fail or render nothing.

## Must-nots

- Never use pure (unbacked) enums for domain values that cross a boundary — persistence and validation need the scalar.
- Never validate enums with `in:` lists or raw value arrays — always `Rule::enum(EnumClass::class)`.
- Never declare the enum cast in a `$casts` property — enum fields must be cast in the `casts()` method. The project-wide `$casts` prohibition → see the labrodev-model skill.
- Never resolve enum labels in the frontend via `t()` — enum labels arrive pre-translated from the backend (`*_label` fields, EnumMapper option maps). This is the one deliberate exception to frontend-only copy → see the labrodev-inertia-react skill.
- Never emit only the raw value to the frontend and let React map value → text — that duplicates the label source of truth.
- Never put workflows, state transitions, or cross-entity logic in an enum. Deciding *whether* a status may change belongs to domain Rules and Actions → see the labrodev-action skill.
- Never call Actions, Services, Jobs, queries, or external APIs from enum methods.
- Never name a presentation helper anything other than `label()` (`getLabel()`, `title()`, `name()` break the EnumMapper/Resource convention).

## 1) Enum anatomy — canonical template

Naming pattern: `{Model}{Aspect}` (e.g. `BookingStatus`, `BookingChannel`, `ProductType`), namespace `Core\Domain\{Domain}\Enums`. Full class/method naming rules → see the labrodev-naming skill.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Enums;

enum BookingStatus: int
{
    case Pending = 1;
    case Confirmed = 2;
    case Cancelled = 3;

    // THE presentation helper. Always named label(), always wraps trans(),
    // always an exhaustive match — no default arm.
    public function label(): string
    {
        return match ($this) {
            self::Pending => trans('Pending'),
            self::Confirmed => trans('Confirmed'),
            self::Cancelled => trans('Cancelled'),
        };
    }

    // Optional: static named constructor wrapping tryFrom() for domain lookups.
    // Returns ?self, never throws.
    public static function fromValue(int $value): ?self
    {
        return self::tryFrom($value);
    }

    // Optional additional pure presentation helpers: color(), icon(), shortLabel().
    // Same constraints as label(): pure, no side effects, no Actions/Services/Jobs.
}
```

Notes:
- Enums are not classes, so `final` does not apply; the `declare(strict_types=1)` header contract still does → see the labrodev-core skill.
- `trans()` keys follow the readable-English-string style backed by Laravel `lang/*.json` → see the labrodev-inertia-react skill.

## 2) Data boundary (write side)

The enum contract at the validation boundary has two fixed parts inside the Data class — a typed property and a `Rule::enum` rule on the same raw input key. Surrounding Data class anatomy (`rules()`, `attributes()`, `prepareForPipeline()`, UUID casters) → see the labrodev-data skill.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Data;

use Core\Domain\Booking\Enums\BookingStatus;
use Illuminate\Validation\Rule;
use Spatie\LaravelData\Data;

final class BookingUpdateData extends Data
{
    public function __construct(
        // Typed enum property — Spatie Data casts the raw scalar implicitly.
        public BookingStatus $status,
        // Nullable variant when the field is optional:
        // public ?BookingStatus $status = null,
    ) {
    }

    /**
     * @return array<string, array<int, mixed>>
     */
    public static function rules(): array
    {
        return [
            // ALWAYS Rule::enum — never 'in:1,2,3', never a raw value array.
            'status' => ['required', Rule::enum(BookingStatus::class)],
            // Nullable variant:
            // 'status' => ['nullable', Rule::enum(BookingStatus::class)],
        ];
    }
}
```

Validation targets the **raw input key** (`status` as a scalar); casting to the enum instance happens after validation.

## 3) Model boundary (persistence)

Any model field backed by an enum MUST be cast in the model's `casts()` method — never a `$casts` property. Full model anatomy (attributes, relations, `$visible`, observers) → see the labrodev-model skill.

```php
// Core\Domain\Booking\Models\Booking

/**
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'status' => BookingStatus::class,
    ];
}
```

After this, `$booking->status` is always a `BookingStatus` instance in PHP — Actions assign it as `$booking->status = BookingStatus::Pending;` (explicit assignment, no mass assignment → see the labrodev-model skill).

## 4) Frontend contract (Resources + ViewModels + React)

### Resource: emit value + label pairs

Field grammar and allowlisting rules for Resources → see the labrodev-viewmodel-resource skill. The enum-specific rule:

```php
// Core\Domain\Booking\Resources\BookingResource::toArray()

'status' => $this->status->value,
'status_label' => $this->status->label(),

// Nullable enum field:
// 'status' => $this->status?->value,
// 'status_label' => $this->status?->label(),
```

Both keys, always: `value` for logic, `*_label` for display. Never one without the other when the frontend renders the field.

### ViewModel: select/filter options via EnumMapper

ViewModel anatomy → see the labrodev-viewmodel-resource skill. The enum-specific rule — option maps come from `EnumMapper::keyValues()` (package `labrodev/php-enum-mapper`), never hardcoded arrays:

```php
// App\Layer\Dashboard\Booking\ViewModels\BookingFormViewModel

use Core\Domain\Booking\Enums\BookingStatus;
use Labrodev\PhpEnumMapper\EnumMapper;

public function statusOptions(): array
{
    return EnumMapper::keyValues(BookingStatus::cases(), 'label');
}
```

The frontend receives this as its `{ value, label }` option set for selects and filters.

### React: render label, use value for logic

Page structure and props typing → see the labrodev-inertia-react skill. Enum-specific rules:

- TypeScript literal unions mirror the enum values: `type BookingStatusValue = 1 | 2 | 3;`
- Display uses `booking.status_label`; conditions and filters use `booking.status` (the raw value).
- Unknown values get an explicit fallback — show the raw value, never silently render nothing:

```tsx
<Badge>{booking.status_label ?? String(booking.status)}</Badge>
```

- Never map value → text in React; the backend `label()` is the single source of truth.

## Edge cases

- **Enum not backed by a model field** (e.g. a mode submitted only in a Data class): parts 1, 2, and 4 of the contract still apply; part 3 (model cast) does not — there is no column.
- **Enum used only internally in Core** (never crosses a boundary): only part 1 applies; `label()` remains optional until the enum is presented anywhere.
- **Backing type choice**: `int` for ordered/stateful sets persisted as integers (`BookingStatus`), `string` for values whose stored form is itself meaningful (`BookingChannel: string { case Web = 'web'; ... }`). Pick per domain, then keep the database column type in sync.
- **Renaming or removing a case**: it is a data migration concern — existing rows hold the old backing value. Migrate the column before removing the case; `tryFrom()`/named constructors return `null` for orphaned values, and the frontend fallback (raw value) makes them visible instead of crashing.
- **Adding a case**: the exhaustive `match` in `label()` (no `default`) makes every unlabeled new case throw `\UnhandledMatchError` — that is intentional. Add the label and the `lang/*.json` entry in the same change.
- **Legacy zone**: vendor/starter code under `app/Http`, `app/Models`, `app/Actions/Fortify` is exempt → see the labrodev-core skill.

## Review checklist

- [ ] Is the enum a backed scalar enum (`int` or `string`) in `Core\Domain\{Domain}\Enums`?
- [ ] Is the name model-prefixed (`BookingStatus`), not generic (`Status`, `Type`, `State`)?
- [ ] Does `label(): string` exist, wrap `trans()`, and use an exhaustive `match` with no `default` arm?
- [ ] Are all enum methods pure — no Actions, Services, Jobs, queries, external APIs, or workflows?
- [ ] Does every Data class touching this enum use a typed enum property AND `Rule::enum(EnumClass::class)` (no `in:` lists)?
- [ ] Is every model field backed by this enum cast in the `casts()` method (never a `$casts` property)?
- [ ] Do Resources emit both `'field' => ->value` and `'field_label' => ->label()` (null-safe for nullable fields)?
- [ ] Do select/filter options come from `EnumMapper::keyValues(Enum::cases(), 'label')` in a ViewModel, never hardcoded in React?
- [ ] Does the frontend render `*_label` for display, use the raw value for logic, and show an explicit raw-value fallback for unknown values?
- [ ] Are static lookups implemented as named constructors wrapping `tryFrom()` returning `?self` (never throwing)?
