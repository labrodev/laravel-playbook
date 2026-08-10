---
name: labrodev-enum
description: "Use when creating, reviewing, or naming PHP enums in a Labrodev Laravel project (BookingStatus, ProductType, OrderState — any status/type/mode field with a finite value set), or when wiring an enum across a boundary: label() presentation helpers, Rule::enum validation in Data classes, enum casts in a model's casts(), value + *_label emission in Resources, or EnumMapper select/filter options in ViewModels."
license: MIT
metadata:
  author: labrodev
---

# Enums (the full enum contract)

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-enum` guideline** (musts, must-nots); the per-file checklist is `rules/enums.md`. This skill holds the craft: anatomy, canonical templates, and edge cases.

Enums represent constrained domain values: statuses, types, modes — any field with a finite allowed set and matching logic. When an enum crosses a boundary, the **full enum contract** applies (the law → `labrodev-enum` guideline). Its four parts map onto the sections below:

| # | Boundary | Contract |
|---|----------|------|
| 1 | Enum class | Backed scalar enum, model-prefixed name, `label(): string` wrapping `trans()` |
| 2 | Data class (write side) | Typed enum property (implicit Spatie cast) + `Rule::enum(EnumClass::class)` |
| 3 | Model (persistence) | Enum cast declared in the `casts()` **method** |
| 4 | Frontend (Inertia props) | Resources emit `value` + `*_label` pairs; ViewModels build options via `EnumMapper::keyValues()` |

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

final class BookingData extends Data
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
// App\Layer\Dashboard\Booking\Resources\BookingResource::toArray()

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
