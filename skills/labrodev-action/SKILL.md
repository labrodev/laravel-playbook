---
name: labrodev-action
description: "Use when creating, reviewing, or refactoring write-side domain classes in a Labrodev Laravel project: Actions (e.g. BookingCreate, BookingUpdate, BookingRemove), domain Services, {Model}Rule classes, or Orchestrator/Pipeline/Payload workflows — including guard ordering, DB::transaction wrapping, UUID assignment, and choosing between throwing ValidationException and a silent no-op."
license: MIT
metadata:
  author: labrodev
---

# Actions, Services, Rules, Orchestrators (write side)

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

This skill owns the **write side** of a domain: Actions (business use cases that mutate state), Services (reusable domain logic), domain Rules (pure can-I checks), and Orchestrators/Pipelines/Payloads (multi-step workflows).

## Musts

- Actions live **only** in `Core/Domain/{Domain}/Actions/`. `App/Layer` must never define Action classes. Controllers map input into Data objects and delegate mutations to Core Actions.
- Every Action is `final readonly` with a **single public `__invoke()`** entry point. Private helper methods are allowed.
- **Rule-then-mutate ordering**: business-condition guards run BEFORE any mutation, and those guards delegate to static `{Model}Rule` methods — never inline the condition in the Action.
- Create Actions wrap the write in `DB::transaction(...)`; the closure is typed and **returns the model**. Any Action performing multiple writes must also be transaction-wrapped.
- Create Actions assign the UUID explicitly: `$booking->uuid = (string) Str::uuid();` — never in the model, `boot()`, or a trait.
- Attributes are assigned **explicitly, one by one**, from the typed Data object. Never `fill()`, `create()`, `update([...])`, or any mass assignment.
- Exactly **one Rule class per model** (`BookingRule` for `Booking`). It is the single source of truth for "can I do X with this model?" — shared by Policies, Actions, Observers, and Orchestrators.
- Rule methods are `public static`, pure checks: return booleans or small decision values; they may run complex conditions and queries.
- Services express domain language, live in `Core/Domain/{Domain}/Services/`, and use one public `__invoke()` for a single operation, or explicit named methods for multiple operations — never `execute()` / `handle()` as generic method names on a Service.
- Orchestrators expose a single `execute($input)` method, communicate between Pipeline steps via a Payload object, and verify the pipeline result with an `instanceof` check that throws `PipelinePayloadIncorrect::make(Payload::class)` on mismatch.
- Pipelines mutate the Payload and pass it on via `return $next($payload);`. Payloads carry flow state only.

## Must-nots

- No mutation before guards; no guards after `save()`.
- Rules must not mutate state, trigger Actions, Jobs, workflows, or any side effects.
- Do not add `fetchRuleClass()` (or similar indirection) on the model — import `{Model}Rule` directly where needed.
- Services must not mutate state when that mutation is a business use case — delegate it to an Action.
- Actions must not call Orchestrators (Orchestrators call Actions, not the reverse).
- Payloads must not contain behavior (data + a `make()` factory only). No anonymous arrays flowing between pipeline steps.
- No presentation concerns (Resources, Inertia, HTTP) anywhere on the write side.
- App-specific ownership/scoping concerns never appear in playbook Actions → see the labrodev-core skill.

## Two failure modes (choose deliberately)

1. **Plan/ownership violation** — the caller is attempting something the business forbids and must be told why. Throw a validation error with a translated message:

   ```php
   if (! BookingRule::canCreate(bookingCreateData: $bookingCreateData)) {
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

Action classes: `{Model}{Verb}` — `BookingCreate`, `BookingUpdate`, `BookingRemove`, `BookingConfirm`. Rule classes: `{Model}Rule`. Services: actor/noun names ending in a clear role (`BookingPriceCalculator`, `EmailSender`) — avoid `Manager`, `Handler`, `Processor`. Callers invoke Actions as callables with named arguments; typed Data parameters mirror the class short name (`BookingCreateData $bookingCreateData`, never `$data`) → full rules in the labrodev-naming skill.

Namespaces below use `Core\Domain\...` as the logical module; the physical root namespace follows the project's Core package → see the labrodev-core skill.

## Template: create Action

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Actions;

use Core\Domain\Booking\Data\BookingCreateData;
use Core\Domain\Booking\Enums\BookingStatus;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Rules\BookingRule;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;
use Illuminate\Validation\ValidationException;

final readonly class BookingCreate
{
    public function __invoke(BookingCreateData $bookingCreateData): Booking
    {
        // Guard clauses first — delegate business conditions to the Rule class.
        if (! BookingRule::canCreate(bookingCreateData: $bookingCreateData)) {
            throw ValidationException::withMessages([
                'starts_at' => [trans('A booking cannot be created for this period.')],
            ]);
        }

        return DB::transaction(function () use ($bookingCreateData): Booking {
            $booking = new Booking();

            // UUID is assigned explicitly in the create Action.
            $booking->uuid = (string) Str::uuid();

            // Assign attributes explicitly — no mass assignment, no fill(), no create().
            $booking->reference = $bookingCreateData->reference;
            $booking->starts_at = $bookingCreateData->startsAt;
            $booking->ends_at = $bookingCreateData->endsAt;
            $booking->status = BookingStatus::Draft;

            $booking->save();

            return $booking;
        });
    }
}
```

Invocation from a controller: `$bookingCreate(bookingCreateData: $bookingCreateData);` — wiring and response idioms → see the labrodev-controller skill.

## Template: update Action

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Actions;

use Core\Domain\Booking\Data\BookingUpdateData;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Rules\BookingRule;

final readonly class BookingUpdate
{
    public function __invoke(
        Booking $booking,
        BookingUpdateData $bookingUpdateData
    ): Booking {
        // State-based ineligibility: silent no-op, guarded by a Rule.
        if (! BookingRule::isEditable(booking: $booking)) {
            return $booking;
        }

        $booking->reference = $bookingUpdateData->reference;
        $booking->starts_at = $bookingUpdateData->startsAt;
        $booking->ends_at = $bookingUpdateData->endsAt;

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

use Core\Domain\Booking\Data\BookingCreateData;
use Core\Domain\Booking\Enums\BookingStatus;
use Core\Domain\Booking\Models\Booking;

final class BookingRule
{
    public static function canCreate(BookingCreateData $bookingCreateData): bool
    {
        return $bookingCreateData->startsAt->lessThan($bookingCreateData->endsAt);
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

Services may depend on Queries, Collections, Models, Rules, Utilities, and other Services; they may be used inside Actions, Orchestrators, Pipelines, Jobs, and Listeners. They can be stateless helpers, dependency-injected coordinators, or stateful calculators bound to a model via a `make(Model $model): self` named constructor.

**Promotion rule:** the moment a Service starts coordinating multiple Actions into a coherent business flow, it is no longer a Service — promote it to an Orchestrator. Services = reusable domain logic; Orchestrators = explicit business scenarios.

## Templates: Orchestrator, Pipeline, Payload

Use this trio when a use case is a staged workflow. The Orchestrator triggers the pipeline with a Payload; each Pipeline step does one atomic thing; the Payload carries flow state. Data objects entering a pipeline may prepare themselves via `prepareForPipeline()` → see the labrodev-data skill.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Orchestrators;

use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Payloads\BookingConfirmationPayload;
use Core\Domain\Booking\Pipelines\BookingConfirmationOrchestrator\ConfirmBooking;
use Core\Domain\Booking\Pipelines\BookingConfirmationOrchestrator\RecalculateBookingTotals;
use Core\Shared\Exceptions\Pipelines\PipelinePayloadIncorrect;
use Illuminate\Pipeline\Pipeline;

final readonly class BookingConfirmationOrchestrator
{
    /**
     * @throws PipelinePayloadIncorrect
     */
    // The parameter name $input is fixed by contract — callers invoke ->execute(input: ...).
    public function execute(Booking $input): Booking
    {
        // Optional pre-checks / guard clauses here (e.g. already confirmed → no-op via Rule).

        $payload = BookingConfirmationPayload::make(booking: $input);

        $resultPayload = app(Pipeline::class)
            ->send($payload)
            ->through([
                ConfirmBooking::class,
                RecalculateBookingTotals::class,
            ])
            ->thenReturn();

        if (! $resultPayload instanceof BookingConfirmationPayload) {
            throw PipelinePayloadIncorrect::make(BookingConfirmationPayload::class);
        }

        return $resultPayload->booking;
    }
}
```

`PipelinePayloadIncorrect` is a `Core/Shared` exception with a `make()` named constructor — create it alongside the project's first Orchestrator.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Pipelines\BookingConfirmationOrchestrator;

use Closure;
use Core\Domain\Booking\Actions\BookingConfirm;
use Core\Domain\Booking\Payloads\BookingConfirmationPayload;

final readonly class ConfirmBooking
{
    public function __construct(
        private BookingConfirm $bookingConfirm,
    ) {}

    /**
     * One atomic step: read from payload, do a single responsibility, mutate payload.
     */
    public function handle(BookingConfirmationPayload $payload, Closure $next): mixed
    {
        $payload->booking = ($this->bookingConfirm)(booking: $payload->booking);

        return $next($payload);
    }
}
```

Pipeline steps live in a subfolder named after their Orchestrator: `Pipelines/{Orchestrator}/`.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Payloads;

use Illuminate\Support\Carbon;

final class BookingConfirmationPayload
{
    // Flow state (mutable) — declare only what this flow actually needs.
    public Carbon $confirmedAt;

    public float $total;

    public function __construct(
        public \Core\Domain\Booking\Models\Booking $booking,
    ) {}

    /**
     * Factory that prepares the minimal valid payload.
     * May resolve required relations and throw a Shared ObjectMissed exception.
     */
    public static function make(\Core\Domain\Booking\Models\Booking $booking): self
    {
        return new self(booking: $booking);
    }
}
```

Payloads are deliberately `final class` (not `readonly`): their public properties are mutable flow state written by pipeline steps. This is the one sanctioned mutability exception on the write side → immutability rules in the labrodev-core skill.

## Edge cases

- **Jobs**: a Job is a thin async wrapper that delegates to an Action or Orchestrator; if a Job contains logic that matters, move it into an Action/Orchestrator/Rule/Service.
- **Shared-behavior Actions across domains**: generic, domain-term-free Actions may live in `Core/Shared/Actions`; anything named after a domain model stays in that domain.
- **Multiple layers need the same mutation**: the Action stays in Core and both layers call it — never duplicate write logic per layer.
- **Observers** may reuse `{Model}Rule` checks to enforce invariants, but workflows stay in Actions/Orchestrators → see the labrodev-model skill.
- **Datetime values** on the write side use `Illuminate\Support\Carbon`, not `CarbonImmutable`.
- **Legacy code** under `app/Http`, `app/Models`, `app/Actions/Fortify` is exempt → see the labrodev-core skill.
- Testing Actions (unit) and routes (feature) → see the labrodev-testing skill.

## Review checklist

1. Does the Action live in `Core/Domain/{Domain}/Actions/`, is it `final readonly`, and does it expose exactly one public `__invoke()`?
2. Do all business-condition guards run before any mutation, and do they delegate to static `{Model}Rule` methods?
3. Is the correct failure mode used — `ValidationException::withMessages` with `trans()` for plan/ownership violations, silent early-return no-op for state-based ineligibility?
4. Is the create Action wrapped in `DB::transaction` with a typed closure returning the model, and is the UUID assigned via `(string) Str::uuid()` inside it?
5. Are all attributes assigned explicitly from the typed Data object — no `fill()`, `create()`, `update([...])`, or mass assignment?
6. Is there exactly one Rule class for the model, with static, pure, side-effect-free methods?
7. Are Services free of business-use-case mutations, and is any Service coordinating multiple Actions promoted to an Orchestrator?
8. Does the Orchestrator `execute()` verify the pipeline result with `instanceof` and throw `PipelinePayloadIncorrect::make(...)` on mismatch?
9. Do Pipeline steps each do one atomic thing, mutate the Payload, and `return $next($payload)` — with steps housed under `Pipelines/{Orchestrator}/`?
10. Is the Payload data-only (public flow-state properties + `make()` factory), with no behavior and no anonymous arrays between steps?
