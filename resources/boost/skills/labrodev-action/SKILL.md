---
name: labrodev-action
description: "Use when creating, reviewing, or refactoring write-side domain classes in a Labrodev Laravel project: Actions (e.g. BookingCreate, BookingUpdate, BookingRemove), domain Services, or {Model}Rule classes — including guard ordering, DB::transaction wrapping, UUID assignment, and choosing between throwing ValidationException and a silent no-op. Staged multi-step workflows live in labrodev-pipeline."
license: MIT
metadata:
  author: labrodev
---

# Actions, Services, Rules (write side)

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-action` guideline** (musts, must-nots); the per-file checklist is `rules/actions.md`. This skill holds the craft: anatomy, canonical templates, and edge cases.

This skill owns the **write side** of a domain: Actions (business use cases that mutate state), Services (reusable domain logic), and domain Rules (pure can-I checks). Staged multi-step workflows (Pipelines, Payloads, orchestrating Services) → see the labrodev-pipeline skill.

## Two failure modes (choose deliberately)

1. **Plan/ownership violation** — the caller is attempting something the business forbids and must be told why. Throw a validation error with a translated message:

   ```php
   if (! BookingRule::canCreate(bookingData: $bookingData)) {
       throw ValidationException::withMessages([
           'starts_at' => [trans('A booking cannot be created for this period.')],
       ]);
   }
   ```

2. **State-based ineligibility** — the model simply is not in a state where the operation applies (already confirmed, not editable, already removed). Silently no-op with an early return, guarded by a Rule:

   ```php
   if (! BookingRule::isEditable(booking: $booking)) {
       return $booking;
   }
   ```

Invalid input *shape* is neither of these — it is handled at the Data validation boundary → see the labrodev-data skill.

## Placement and naming pattern

Action classes: `{Model}{Verb}` — `BookingCreate`, `BookingUpdate`, `BookingRemove`, `BookingConfirm`. Rule classes: `{Model}Rule`. Services: actor/noun names ending in a clear role (`BookingPriceCalculator`, `EmailSender`) — avoid `Manager`, `Handler`, `Processor`. Callers invoke Actions as callables with named arguments; typed Data parameters mirror the class short name (`BookingData $bookingData`, never `$data`) → full rules in the labrodev-naming skill.

Namespaces below use `Core\Domain\...` as the logical module; the physical root namespace follows the project's Core package → see the labrodev-core skill.

## Template: create Action

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Actions;

use Core\Domain\Booking\Data\BookingData;
use Core\Domain\Booking\Enums\BookingStatus;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Rules\BookingRule;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;
use Illuminate\Validation\ValidationException;

final readonly class BookingCreate
{
    public function __invoke(BookingData $bookingData): Booking
    {
        // Guard clauses first — delegate business conditions to the Rule class.
        if (! BookingRule::canCreate(bookingData: $bookingData)) {
            throw ValidationException::withMessages([
                'starts_at' => [trans('A booking cannot be created for this period.')],
            ]);
        }

        return DB::transaction(function () use ($bookingData): Booking {
            $booking = new Booking();

            // UUID is assigned explicitly in the create Action.
            $booking->uuid = (string) Str::uuid();

            // Assign attributes explicitly — no mass assignment, no fill(), no create().
            $booking->reference = $bookingData->reference;
            $booking->starts_at = $bookingData->startsAt;
            $booking->ends_at = $bookingData->endsAt;
            $booking->status = BookingStatus::Draft;

            $booking->save();

            return $booking;
        });
    }
}
```

Invocation from a controller: `$bookingCreate(bookingData: $bookingData);` — wiring and response idioms → see the labrodev-controller skill.

## Template: update Action

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Actions;

use Core\Domain\Booking\Data\BookingData;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Rules\BookingRule;

final readonly class BookingUpdate
{
    public function __invoke(
        Booking $booking,
        BookingData $bookingData
    ): Booking {
        // State-based ineligibility: silent no-op, guarded by a Rule.
        if (! BookingRule::isEditable(booking: $booking)) {
            return $booking;
        }

        $booking->reference = $bookingData->reference;
        $booking->starts_at = $bookingData->startsAt;
        $booking->ends_at = $bookingData->endsAt;

        $booking->save();

        return $booking;
    }
}
```

Wrap the body in `DB::transaction` as soon as the update touches more than one write (e.g. syncing relations).

## Template: remove Action

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Actions;

use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Rules\BookingRule;

final readonly class BookingRemove
{
    public function __invoke(Booking $booking): void
    {
        if (! BookingRule::canBeDeleted(booking: $booking)) {
            return;
        }

        // Soft or hard delete is a model concern.
        $booking->delete();
    }
}
```

## Template: domain Rule

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Rules;

use Core\Domain\Booking\Data\BookingData;
use Core\Domain\Booking\Enums\BookingStatus;
use Core\Domain\Booking\Models\Booking;

final class BookingRule
{
    public static function canCreate(BookingData $bookingData): bool
    {
        return $bookingData->startsAt->lessThan($bookingData->endsAt);
    }

    public static function isEditable(Booking $booking): bool
    {
        return $booking->status === BookingStatus::Draft;
    }

    public static function canBeDeleted(Booking $booking): bool
    {
        return $booking->status !== BookingStatus::Confirmed;
    }
}
```

Rules are not request validation (→ see the labrodev-data skill) and not Policies — Policies call these Rule methods directly (`BookingRule::canBeDeleted($booking)`) → see the labrodev-authorization skill.

## Template: Service

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Services;

use Core\Domain\Booking\Models\Booking;

final class BookingPriceCalculator
{
    // Single clear operation → one public __invoke() as the only entry method.
    public function __invoke(Booking $booking): float
    {
        return $this->nightCount(booking: $booking) * $booking->nightly_rate;
    }

    private function nightCount(Booking $booking): int
    {
        return (int) $booking->starts_at->diffInDays($booking->ends_at);
    }
}
```

Services may depend on Queries, Collections, Models, Rules, Utilities, and other Services; they may be used inside Actions, Pipeline steps, Jobs, and Listeners. They can be stateless helpers, dependency-injected coordinators, or stateful calculators bound to a model via a `make(Model $model): self` named constructor.

**Staged workflows:** the moment a use case grows beyond one atomic mutation — several distinct side-effecting steps (create + mail + log + external CRM) — stage it through Pipelines: a `{Workflow}Payload` carries flow state, verb-first step classes under `Pipelines/{Workflow}/` do one thing each, and a Service in `Services/` plays the orchestrator role. Anatomy, templates, and the Domain-vs-Feature placement rule → see the labrodev-pipeline skill.

## Edge cases

- **Jobs**: a Job is a thin async wrapper that delegates to an Action or Service; if a Job contains logic that matters, move it into an Action/Rule/Service.
- **Shared-behavior Actions across domains**: generic, domain-term-free Actions may live in `Core/Shared/Actions`; anything named after a domain model stays in that domain.
- **Multiple layers need the same mutation**: the Action stays in Core and both layers call it — never duplicate write logic per layer.
- **Observers** may reuse `{Model}Rule` checks to enforce invariants, but workflows stay in Actions and pipelines → see the labrodev-model skill.
- **Datetime values** on the write side use `Illuminate\Support\Carbon`, not `CarbonImmutable`.
- **Legacy code** under `app/Http`, `app/Models`, `app/Actions/Fortify` is exempt → see the labrodev-core skill.
- Testing Actions (unit) and routes (feature) → see the labrodev-testing skill.
