---
name: labrodev-data
description: "Use when creating, reviewing, or naming Spatie Data classes (*Data), UUID/collection casters (*UuidCaster, *CollectionCaster), or any write-side input validation in a Labrodev Laravel project — including rules(), attributes(), prepareForPipeline(), relation-via-UUID mapping, and nested Data. Also use when someone reaches for a FormRequest: Data classes are the only validation layer."
license: MIT
metadata:
  author: labrodev
---

# Data and Validation (Spatie Laravel Data)

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-data` guideline** (musts, must-nots); the per-file checklist is `rules/data.md`. This skill holds the craft: anatomy, canonical templates, and edge cases.

This skill covers the full write-side surface: the Data class template, the four-part UUID relation contract, object and collection casters, and where validation may and may not reach.

## Data class template

Naming pattern: `{Model}Data` — the shared create/update default and when a split into `{Model}CreateData` / `{Model}UpdateData` is legitimate → `labrodev-data` guideline and "Edge cases" below. Namespace: `Core\Domain\{Domain}\Data`. Full naming rules and the mirror-variable rule (`BookingData $bookingData`, never `$data`) → see the labrodev-naming skill.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Data;

use Core\Domain\Booking\Enums\BookingChannel;
use Core\Domain\Extra\Casts\ExtraCollectionCaster;
use Core\Domain\Extra\Collections\ExtraCollection;
use Core\Domain\Service\Casts\ServiceUuidCaster;
use Core\Domain\Service\Models\Service;
use Illuminate\Validation\Rule;
use Spatie\LaravelData\Attributes\DataCollectionOf;
use Spatie\LaravelData\Attributes\WithCast;
use Spatie\LaravelData\Data;
use Spatie\LaravelData\DataCollection;

final class BookingData extends Data
{
    /**
     * @param  DataCollection<int, BookingGuestData>  $guests
     */
    public function __construct(
        #[WithCast(ServiceUuidCaster::class)]
        public Service $service,

        #[WithCast(ExtraCollectionCaster::class)]
        public ExtraCollection $extras,

        public BookingChannel $channel,

        public string $starts_at,
        public int $guest_count,

        #[DataCollectionOf(BookingGuestData::class)]
        public DataCollection $guests,

        public ?string $notes = null,
    ) {}

    /**
     * @param  array<string, mixed>  $properties
     * @return array<string, mixed>
     */
    public static function prepareForPipeline(array $properties): array
    {
        if (isset($properties['notes']) && is_string($properties['notes'])) {
            $properties['notes'] = trim($properties['notes']);
        }

        return $properties;
    }

    public static function rules(): array
    {
        return [
            'service' => ['required', 'string', 'uuid', 'exists:services,uuid'],
            'extras' => ['nullable', 'array'],
            'extras.*' => ['string', 'uuid', 'exists:extras,uuid'],
            'channel' => ['required', Rule::enum(BookingChannel::class)],
            'starts_at' => ['required', 'date', 'after:now'],
            'guest_count' => ['required', 'integer', 'min:1'],
            'guests' => ['required', 'array', 'min:1'],
            'guests.*.name' => ['required', 'string', 'max:255'],
            'guests.*.email' => ['nullable', 'string', 'email'],
            'notes' => ['nullable', 'string', 'max:2000'],
        ];
    }

    /**
     * @return array<string,string>
     */
    public static function attributes(): array
    {
        return [
            'service' => trans('Service'),
            'extras' => trans('Extras'),
            'channel' => trans('Channel'),
            'starts_at' => trans('Start time'),
            'guest_count' => trans('Guest count'),
            'guests' => trans('Guests'),
            'notes' => trans('Notes'),
        ];
    }
}
```

The nested item is a plain Data class in the same folder; its rules live in the parent via wildcard keys as shown above:

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Data;

use Spatie\LaravelData\Data;

final class BookingGuestData extends Data
{
    public function __construct(
        public string $name,
        public ?string $email = null,
    ) {}
}
```

## Relation via UUID — the fixed contract

Data classes refer to related models by **entity name**, never by internal ID. The contract has four parts that must all use the SAME key:

1. **Input key** — the client sends a UUID string under the entity name (`service`, not `service_id` / `service_uuid`).
2. **Property** — typed model property with the same name: `public Service $service` (or `public ?Service $service = null` when the business allows it, plus a `nullable` rule).
3. **Cast** — `#[WithCast(ServiceUuidCaster::class)]` resolves the validated UUID into the model.
4. **Validation** — the raw key gets `['required', 'string', 'uuid', 'exists:services,uuid']` (`nullable` instead of `required` for optional relations).

Property name = validation key = caster input. Consistency between these three is mandatory.

The caster lives in the **related model's** domain: `Core\Domain\{Domain}\Casts\{Model}UuidCaster`.

## Object caster template

Naming pattern: `{Model}UuidCaster` (or `{Model}IdCaster` when resolving by another identifier), namespace `Core\Domain\{Domain}\Casts` of the model's own domain.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Service\Casts;

use Core\Domain\Service\Models\Service;
use Core\Domain\Service\Queries\ServiceQuery;
use Spatie\LaravelData\Casts\Cast;
use Spatie\LaravelData\Casts\Uncastable;
use Spatie\LaravelData\Contracts\BaseData;
use Spatie\LaravelData\Support\Creation\CreationContext;
use Spatie\LaravelData\Support\DataProperty;

final readonly class ServiceUuidCaster implements Cast
{
    /**
     * @param  array<string, mixed>  $properties
     * @param  CreationContext<BaseData<mixed, mixed, array-key>>  $context
     */
    public function cast(
        DataProperty $property,
        mixed $value,
        array $properties,
        CreationContext $context
    ): Service|Uncastable {
        if ($value instanceof Service) {
            return $value;
        }

        if (! is_string($value) || $value === '') {
            return Uncastable::create();
        }

        return resolve(ServiceQuery::class)
            ->byUuid($value)
            ->first() ?? Uncastable::create();
    }
}
```

Behavior rules baked into this template:

- Models are resolved via the Domain Query class (`ServiceQuery`), never directly via Eloquent → see the labrodev-query skill.
- Already-a-model values pass through unchanged — the `instanceof` arm exists so Data can be constructed directly in code and tests (`BookingData::from(['service' => $service, ...])`).
- Missing/invalid values return `Uncastable::create()` — **never throw**. The `exists:services,uuid` rule on the same key rejects bad input with a proper message.

## Collection caster template

Naming pattern: `{Model}CollectionCaster`, same domain and namespace as the object caster. Return type is the domain's typed Eloquent collection → see the labrodev-model skill for Collections.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Extra\Casts;

use Core\Domain\Extra\Collections\ExtraCollection;
use Core\Domain\Extra\Queries\ExtraQuery;
use Spatie\LaravelData\Casts\Cast;
use Spatie\LaravelData\Contracts\BaseData;
use Spatie\LaravelData\Support\Creation\CreationContext;
use Spatie\LaravelData\Support\DataProperty;

final readonly class ExtraCollectionCaster implements Cast
{
    /**
     * @param  array<string, mixed>  $properties
     * @param  CreationContext<BaseData<mixed, mixed, array-key>>  $context
     */
    public function cast(
        DataProperty $property,
        mixed $value,
        array $properties,
        CreationContext $context
    ): ExtraCollection {
        if ($value instanceof ExtraCollection) {
            return $value;
        }

        $uuids = collect((array) $value)
            ->filter(fn (mixed $item): bool => is_string($item) && $item !== '')
            ->values();

        if ($uuids->isEmpty()) {
            return new ExtraCollection();
        }

        return resolve(ExtraQuery::class)
            ->byUuids($uuids->all())
            ->get();
    }
}
```

Validation pair for the same key: `'extras' => ['required', 'array']` (or `nullable`) plus `'extras.*' => ['string', 'uuid', 'exists:extras,uuid']`. UUIDs that resolve to nothing simply yield a smaller collection — the wildcard `exists` rule is what rejects them, never the caster.

## What validation may and may not enforce

Allowed: required/nullable, primitive types and formats, enum constraints, existence checks (`exists:<table>,uuid`), value ranges.

Not allowed: business rules, dependence on domain state beyond simple existence, deciding whether an operation is permitted. Those live in Core Actions/Rules → see the labrodev-action skill.

## Enums in Data

Full enum contract → see the labrodev-enum skill. Example at this boundary: `public BookingChannel $channel` with `'channel' => ['required', Rule::enum(BookingChannel::class)]`.

## Edge cases

- **Optional relation**: `public ?Service $service = null` with `['nullable', 'string', 'uuid', 'exists:services,uuid']`. Only make it optional when the business genuinely allows it.
- **Optional collection**: validate `nullable` + `array`; the collection caster turns missing/empty input into an empty typed collection, so the property stays non-nullable.
- **Update uses the SAME `BookingData` class as create** — that is the default deal. The model being updated arrives via route binding, not via the Data class → see the labrodev-controller skill.
- **Splitting into `{Model}CreateData` / `{Model}UpdateData`**: allowed ONLY when create and update genuinely accept different fields (e.g. a field settable once at creation, or update-only fields). Never split pre-emptively "for symmetry" — two classes with identical fields are drift waiting to happen.
- **Constructing Data in code/tests**: pass model instances or typed collections directly (`BookingData::from(['service' => $service, ...])`) — the pass-through branches in the casters make this work without touching the database.
- **Uncastable is not an error path**: a caster returning `Uncastable::create()` leaves rejection to the validation rule on the same key. If you feel the urge to throw inside a caster, the missing piece is a validation rule.
- **Custom Cast classes** are Core citizens: no dependence on controllers, HTTP, or UI. They may be reused by Data casts, Eloquent casts, and internal domain logic.
- **Data properties are not fillable-bait**: Actions read named properties off the Data object; they never spread `->toArray()` into `Model::create()` blindly → see the labrodev-action skill.

## Cross-skill pointers

- Controller anatomy, `{model:uuid}` binding, how Data reaches `__invoke()` → see the labrodev-controller skill.
- Read-side queries (`byUuid`, `byUuids`, IndexQueries) → see the labrodev-query skill.
- Actions/Services consuming Data, UUID assignment on create, transactions → see the labrodev-action skill.
- Model `casts()` method (never `$casts`), relations, Collections → see the labrodev-model skill.
- Full enum contract → see the labrodev-enum skill.
- Class/property/variable naming, named-argument invocation, mirror parameter names → see the labrodev-naming skill.
- File header contract (`declare(strict_types=1)`, `final`), dependency direction, legacy two-zone policy → see the labrodev-core skill.
