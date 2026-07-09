---
name: labrodev-pipeline
description: "Use when a business use case is a staged workflow rather than one atomic mutation — e.g. booking creation that also creates a customer, sends mails, writes logs, and pushes data to an external CRM. Covers Pipeline step classes, the Payload flow-state object, the orchestrating Service that drives them, and where the trio lives (Core/Domain/{Domain} vs Core/Feature for cross-domain workflows)."
license: MIT
metadata:
  author: labrodev
---

# Pipelines: staged workflows with a Payload and an orchestrating Service

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

A pipeline workflow has exactly three parts:

| Part | Class | Lives in |
|---|---|---|
| Flow state | `{Workflow}Payload` | `Core/Domain/{Domain}/Payloads/` |
| Atomic steps | verb-first step classes, no suffix | `Core/Domain/{Domain}/Pipelines/{Workflow}/` |
| Orchestrator | `{Workflow}Service` — a **Service** plays the orchestrator role | `Core/Domain/{Domain}/Services/` |

There is **no separate Orchestrator class type**. The orchestrating class is a Service and follows every Service rule: single public `__invoke()`, injected and invoked as a callable with named arguments (→ see the labrodev-naming skill).

## When to use a pipeline

Reach for this trio when a use case is a **staged workflow** — several distinct steps, each with its own side effect, that must run as one coherent business scenario. Canonical example: registering a booking is not just creating the booking row — it also creates the customer, sends confirmation mail, writes an audit log, and pushes the data to an external CRM.

Do NOT use a pipeline for:

- a single atomic mutation → a plain Action (→ see the labrodev-action skill);
- two steps where the second is a persistence-adjacent side effect → Action + Observer (→ see the labrodev-model skill);
- read-side composition → ViewModels/Queries, never pipelines.

**Placement rule:** when every step belongs to one domain, the trio lives in that domain (`Core/Domain/Booking/...`). When the workflow genuinely spans domains (Booking + Customer + external CRM), it is cross-domain logic and lives in a **Feature module**: `Core/Feature/{FeatureName}/` with the same internal `Payloads/`, `Pipelines/{Workflow}/`, `Services/` structure. Features may depend on multiple Domains and on Infrastructure contracts; Domains must not depend on Features (→ see the labrodev-core skill).

## Musts

- The orchestrating Service exposes a single public `__invoke(...)`, builds the Payload via `{Workflow}Payload::make(...)`, runs `app(Pipeline::class)->send($payload)->through([...])->thenReturn()`, validates the result with an `instanceof` check throwing `PipelinePayloadIncorrect::make(...)`, and returns the final value from the Payload.
- Each Pipeline step does **one atomic thing**, mutates the Payload, and returns `$next($payload)`. Steps are `final readonly` with a single `handle({Workflow}Payload $payload, Closure $next): mixed` method.
- Steps are named **verb-first, no suffix** (`CreateCustomer`, `CreateBooking`, `SendBookingConfirmationMail`, `PushBookingToCrm`, `LogBookingRegistration`) and live under `Pipelines/{Workflow}/` — a subfolder named after the workflow.
- **Mutations happen through Actions.** A step that persists something calls the domain Action (`BookingCreate`, `CustomerCreate`) — it never writes models directly. Each Action manages its own transaction (→ see the labrodev-action skill).
- **Persistence before external side effects.** Order the steps so DB-mutating steps run first; mail, CRM pushes, and other external calls run after the domain state is safely persisted. Never wrap the whole pipeline in one `DB::transaction()` — external calls do not belong inside a transaction.
- External integrations are reached through Infrastructure contracts (a `CrmClient` interface from `Core/Infrastructure/...`), never called inline with HTTP code in a step.
- The Payload is a mutable `final class` (NOT readonly — it is the one sanctioned mutability exception) with public flow-state properties and a static `make()` constructor that validates prerequisites.

## Must-nots

- No business branching in the orchestrating Service beyond guard clauses and step selection — logic lives in the steps and the Actions they call.
- No step that does two things ("create customer and send mail") — split it.
- No pipelines inside pipelines; if a step needs its own staged workflow, that workflow is its own Service the step calls.
- No swallowing step failures: a failing step throws; slow/unreliable external steps may dispatch a queued Job instead of calling synchronously.
- No Payload reuse across workflows — one Payload class per workflow, named after it.

## Worked example: booking registration

### Payload — `Core/Domain/Booking/Payloads/BookingRegistrationPayload.php`

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Payloads;

use Core\Domain\Booking\Data\BookingData;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Customer\Models\Customer;

final class BookingRegistrationPayload
{
    public Booking $booking;

    public Customer $customer;

    public static function make(BookingData $bookingData): self
    {
        $payload = new self();
        $payload->bookingData = $bookingData;

        return $payload;
    }

    public function __construct(
        public ?BookingData $bookingData = null,
    ) {}
}
```

### Steps — `Core/Domain/Booking/Pipelines/BookingRegistration/`

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Pipelines\BookingRegistration;

use Closure;
use Core\Domain\Booking\Actions\BookingCreate;
use Core\Domain\Booking\Payloads\BookingRegistrationPayload;

final readonly class CreateBooking
{
    public function __construct(
        private BookingCreate $bookingCreate,
    ) {}

    public function handle(BookingRegistrationPayload $payload, Closure $next): mixed
    {
        $payload->booking = ($this->bookingCreate)(
            bookingData: $payload->bookingData,
        );

        return $next($payload);
    }
}
```

Sibling steps follow the same shape: `CreateCustomer` calls the `CustomerCreate` Action and sets `$payload->customer`; `SendBookingConfirmationMail` dispatches the notification; `PushBookingToCrm` calls the `CrmClient` Infrastructure contract; `LogBookingRegistration` writes the audit log entry.

### Orchestrating Service — `Core/Domain/Booking/Services/BookingRegistrationService.php`

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Services;

use Core\Domain\Booking\Data\BookingData;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Payloads\BookingRegistrationPayload;
use Core\Domain\Booking\Pipelines\BookingRegistration\CreateBooking;
use Core\Domain\Booking\Pipelines\BookingRegistration\CreateCustomer;
use Core\Domain\Booking\Pipelines\BookingRegistration\LogBookingRegistration;
use Core\Domain\Booking\Pipelines\BookingRegistration\PushBookingToCrm;
use Core\Domain\Booking\Pipelines\BookingRegistration\SendBookingConfirmationMail;
use Core\Shared\Exceptions\Pipelines\PipelinePayloadIncorrect;
use Illuminate\Pipeline\Pipeline;

final readonly class BookingRegistrationService
{
    public function __invoke(BookingData $bookingData): Booking
    {
        $payload = BookingRegistrationPayload::make(bookingData: $bookingData);

        $resultPayload = app(Pipeline::class)
            ->send($payload)
            ->through([
                CreateCustomer::class,
                CreateBooking::class,
                SendBookingConfirmationMail::class,
                PushBookingToCrm::class,
                LogBookingRegistration::class,
            ])
            ->thenReturn();

        if (! $resultPayload instanceof BookingRegistrationPayload) {
            throw PipelinePayloadIncorrect::make(BookingRegistrationPayload::class);
        }

        return $resultPayload->booking;
    }
}
```

`PipelinePayloadIncorrect` is a `Core/Shared/Exceptions/Pipelines` exception with a `make()` named constructor — create it alongside the project's first pipeline.

### Call site

The controller injects the Service and invokes it as a callable with named arguments, exactly like an Action:

```php
public function __invoke(
    BookingData $bookingData,
    BookingRegistrationService $bookingRegistrationService,
): RedirectResponse {
    $bookingRegistrationService(bookingData: $bookingData);
    // toast + to_route → see the labrodev-controller skill
}
```

Data objects entering a pipeline may normalize themselves via `prepareForPipeline()` → see the labrodev-data skill.

## Cross-domain variant (Feature module)

The booking-registration example already touches the Customer domain and CRM infrastructure — in a real project that is the signal to move it to `Core/Feature/BookingRegistration/`:

```
Core/Feature/BookingRegistration/
├── Payloads/BookingRegistrationPayload.php
├── Pipelines/BookingRegistration/{CreateCustomer, CreateBooking, ...}.php
└── Services/BookingRegistrationService.php
```

Same classes, same rules — only the namespace root changes. Promote a Feature to this structure the moment its steps import models or Actions from more than one domain.

## Review checklist

1. Is this genuinely a staged workflow (3+ distinct side-effecting steps) — not an atomic mutation that belongs in a plain Action?
2. Does the trio live in the right place: single-domain → `Core/Domain/{Domain}`, cross-domain → `Core/Feature/{FeatureName}`?
3. Is the orchestrator a Service in `Services/` with a single `__invoke()`, invoked as a callable with named arguments?
4. Does the Service validate the pipeline result with `instanceof` + `PipelinePayloadIncorrect::make(...)`?
5. Does every step do exactly one thing, mutate the Payload, and `return $next($payload)`?
6. Are steps verb-first with no suffix, housed under `Pipelines/{Workflow}/`?
7. Do mutating steps delegate to Actions (own transactions), with persistence steps ordered before mail/CRM/external steps?
8. Are external integrations reached through Infrastructure contracts, with slow/unreliable calls pushed to queued Jobs?
9. Is the Payload a mutable `final class` dedicated to this one workflow, with a `make()` constructor?
