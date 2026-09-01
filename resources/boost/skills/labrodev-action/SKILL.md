---
name: labrodev-action
description: "Use when creating, reviewing, or refactoring write-side domain classes in a Labrodev Laravel project: Actions (e.g. BookingCreate, BookingUpdate, BookingRemove), domain Services, or {Model}Rule classes — including guard ordering, domain exceptions on failed gates, DB::transaction wrapping, and UUID assignment. Staged multi-step workflows live in labrodev-pipeline."
license: MIT
metadata:
  author: labrodev
---

# Actions, Services, Rules (write side)

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-action` guideline** (musts, must-nots); the per-file checklist is `rules/actions.md`. This skill holds the craft: anatomy, canonical templates, and edge cases.

This skill owns the **write side** of a domain: Actions (business use cases that mutate state), Services (reusable domain logic), and domain Rules (pure can-I checks). Staged multi-step workflows (Pipelines, Payloads, orchestrating Services) → see the labrodev-pipeline skill.

## Where gates run (two levels, one owner each)

1. **Policy level — the can-trio.** `{Model}Rule::canCreate` / `canUpdate` / `canRemove` are called from the corresponding Policy methods (`create()`, `update()`, `remove()`), so an ineligible request is rejected with 403 **before** the Action ever runs → see the labrodev-authorization skill. Actions never re-run these checks.

2. **Action level — domain invariants the Policy cannot see.** Conditions that depend on submitted input or cross-record state at write time (an overlapping period, a plan limit against the requested quantity) are guarded inside the Action, delegated to a `{Model}Rule` method, and **throw a dedicated domain exception** on failure:

   ```php
   if (! BookingRule::periodIsAvailable(bookingData: $bookingData)) {
       throw BookingPeriodUnavailableException::make(bookingData: $bookingData);
   }
   ```

   Never `ValidationException` — that class belongs exclusively to the Data validation layer (→ labrodev-data), and a failed business gate is a business failure with its own exception class (→ labrodev-exception). Never a silent early-return no-op either: silent skips hide bugs from callers outside HTTP (pipelines, jobs, console).

Invalid input *shape* is neither of these — it is handled at the Data validation boundary → see the labrodev-data skill. How a domain exception becomes a user-facing response (toast, JSON error) is the delivery layer's concern via the exception handler — never the Action's.

## Placement and naming pattern

Action classes always end in one of the three verbs: `{Model}Create`, `{Model}Update`, `{Model}Remove` — with an aspect infix for partial/state updates (`BookingStatusUpdate`), never a bespoke verb (`BookingConfirm` is not a class name; confirmation is a `BookingStatusUpdate`). One Action = one singular mutation of one model object; anything touching several objects is a Service calling Actions. Rule classes: `{Model}Rule`; gate methods mirror the operation verb (`can{Verb}`), invariant checks get descriptive names (`periodIsAvailable`). Services: `-er` agent nouns naming the manipulation inside (`BookingPriceCalculator`, `EmailSender`, `BookingSearcher`) — never `Manager`, `Handler`, `Processor`. Callers invoke Actions as callables with named arguments; typed Data parameters mirror the class short name (`BookingData $bookingData`, never `$data`) → full rules in the labrodev-naming skill.

Namespaces below use `Core\Domain\...` as the logical module; the physical root namespace follows the project's Core package → see the labrodev-core skill.

## Template: create Action

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Actions;

use Core\Domain\Booking\Data\BookingData;
use Core\Domain\Booking\Enums\BookingStatus;
use Core\Domain\Booking\Exceptions\BookingPeriodUnavailableException;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Rules\BookingRule;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;

final readonly class BookingCreate
{
    public function __invoke(BookingData $bookingData): Booking
    {
        if (! BookingRule::periodIsAvailable(bookingData: $bookingData)) {
            throw BookingPeriodUnavailableException::make(bookingData: $bookingData);
        }

        return DB::transaction(function () use ($bookingData): Booking {
            $booking = new Booking();

            $booking->uuid = (string) Str::uuid();

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

Anatomy, in order: invariant guard (Rule condition, named domain exception), then the transaction-wrapped write with the UUID assigned explicitly and every attribute assigned one by one — no mass assignment, no `fill()`, no `create()`. The template carries no comments: the no-comments law applies to every class (→ labrodev-core), and the structure is the explanation.

Invocation from a controller: `$bookingCreate(bookingData: $bookingData);` — wiring and response idioms → see the labrodev-controller skill.

## Template: update Action

Eligibility (`BookingRule::canUpdate`) was already enforced by the Policy — the Action receives an editable model and mutates it. Only input-dependent invariants are re-guarded here.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Actions;

use Core\Domain\Booking\Data\BookingData;
use Core\Domain\Booking\Exceptions\BookingPeriodUnavailableException;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Rules\BookingRule;

final readonly class BookingUpdate
{
    public function __invoke(
        Booking $booking,
        BookingData $bookingData
    ): Booking {
        if (! BookingRule::periodIsAvailable(bookingData: $bookingData, ignoreBooking: $booking)) {
            throw BookingPeriodUnavailableException::make(bookingData: $bookingData);
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

Eligibility (`BookingRule::canRemove`) is Policy territory; with no input-dependent invariant left, the Action is pure mutation. Whether `delete()` is a soft or hard delete is the model's concern.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Actions;

use Core\Domain\Booking\Models\Booking;

final readonly class BookingRemove
{
    public function __invoke(Booking $booking): void
    {
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
use Core\Domain\Booking\Queries\BookingQuery;

final class BookingRule
{
    public static function canCreate(): bool
    {
        return true;
    }

    public static function canUpdate(Booking $booking): bool
    {
        return $booking->status === BookingStatus::Draft;
    }

    public static function canRemove(Booking $booking): bool
    {
        return $booking->status !== BookingStatus::Confirmed;
    }

    public static function periodIsAvailable(BookingData $bookingData, ?Booking $ignoreBooking = null): bool
    {
        return ! resolve(BookingQuery::class)
            ->overlappingPeriod(startsAt: $bookingData->startsAt, endsAt: $bookingData->endsAt)
            ->when($ignoreBooking !== null, fn ($query) => $query->whereKeyNot($ignoreBooking->getKey()))
            ->exists();
    }
}
```

`canCreate()` is the project-specific slot for create-eligibility visible without submitted input (plan limits, account state, feature windows) — the template's `return true` marks the slot, real projects fill it with a real condition. `periodIsAvailable()` shows the read discipline: DB conditions route through the model's Query class via `resolve(BookingQuery::class)` — never inline `Booking::query()` (→ labrodev-query) — and callers chain further conditions on the returned Builder.

Rules are not request validation (→ see the labrodev-data skill) and not Policies — Policies call the can-trio directly (`BookingRule::canRemove($booking)`) as the enforcement point → see the labrodev-authorization skill.

## Template: Service

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Services;

use Core\Domain\Booking\Models\Booking;

final class BookingPriceCalculator
{
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

**Batch modifications are Services, not Actions.** An Action mutates exactly one model object; when a use case modifies many (bulk archive, sync a collection, re-price every booking of a day), write a Service that iterates and invokes the singular Action per object — the invariants stay enforced in one place and the batch class stays a coordinator.

**Staged workflows:** the moment a use case grows beyond one atomic mutation — several distinct side-effecting steps (create + mail + log + external CRM) — stage it through Pipelines: a `{Workflow}Payload` carries flow state, verb-first step classes under `Pipelines/{Workflow}/` do one thing each, and a Service in `Services/` plays the orchestrator role. Anatomy, templates, and the Domain-vs-Feature placement rule → see the labrodev-pipeline skill.

## Edge cases

- **Callers outside HTTP** (pipeline steps, jobs, console commands) bypass Policies — they consult the same `{Model}Rule` gates before invoking an Action when eligibility is in question; the Action's own invariant guards still fail loud for them.
- **Jobs**: a Job is a thin async wrapper that delegates to an Action or Service; if a Job contains logic that matters, move it into an Action/Rule/Service. Queue settings are declared as class attributes (`#[OnQueue('notifications')]`, `#[OnConnection('redis')]`, `#[WithoutRelations]`) — never as public properties or constructor wiring.
- **Notifications**: always the chain Event → Listener → queued Job → Notification. The Action dispatches `BookingConfirmedEvent`; a Listener dispatches `SendBookingConfirmationJob`; the Job sends the Notification. Never `->notify()` inline on the write side.
- **Shared-behavior Actions across domains**: generic, domain-term-free Actions may live in `Core/Shared/Actions`; anything named after a domain model stays in that domain.
- **Multiple layers need the same mutation**: the Action stays in Core and both layers call it — never duplicate write logic per layer.
- **Observers** may reuse `{Model}Rule` checks as a persistence-level safety net, but workflows stay in Actions and pipelines → see the labrodev-model skill.
- **Datetime values** on the write side use `Illuminate\Support\Carbon`, not `CarbonImmutable`.
- **Legacy code** under `app/Http`, `app/Models`, `app/Actions/Fortify` is exempt → see the labrodev-core skill.
- Testing Actions (unit) and routes (feature) → see the labrodev-testing skill.
