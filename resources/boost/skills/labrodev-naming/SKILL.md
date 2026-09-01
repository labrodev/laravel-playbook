---
name: labrodev-naming
description: "Use when naming anything in a Labrodev Laravel project — domains, classes (Actions, Services, Controllers, Data, Queries, Policies, Enums, ...), methods, variables — and whenever invoking an Action/Service or declaring typed parameters. Governs the class-suffix pattern table, the named-arguments callable invocation contract, and the typed-parameter mirror rule (BookingData $bookingData, never $data)."
license: MIT
metadata:
  author: labrodev
---

# Labrodev Naming & Named Arguments

Part of the Labrodev playbook. **The law for this skill lives in the always-on `labrodev-naming` guideline** (musts, must-nots, the naming pattern table, the invocation contract) — this skill holds the craft: worked invocation examples, the mirror rule applied to real code, and edge cases.

## The invocation contract (worked examples)

Actions and Services are injected as `__invoke()` parameters and invoked as **callables with named arguments**. The typed Data parameter mirrors the class short name in camelCase.

Call-site pattern in a controller (`{Model}{Action}Controller`; here Model = `Booking`, Action = `Store`):

```php
public function __invoke(
    BookingData $bookingData, // mirrors class short name — never $data
    BookingCreate $bookingCreate,         // mirrors class short name — never $action
): RedirectResponse {
    // Action invoked as a CALLABLE with a NAMED argument — never ->execute()/->handle()
    $bookingCreate(
        bookingData: $bookingData,
    );
    // ...
}
```

Multi-argument invocation (update Action — arguments named, consistent order):

```php
$bookingUpdate(
    booking: $booking,
    bookingData: $bookingData,
);
```

Single-argument calls may stay positional:

```php
$bookingDelete($booking);
```

Declaration side — the Action exposes exactly one public entry point, `__invoke()`, with mirror-named typed parameters:

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Actions;

use Core\Domain\Booking\Data\BookingData;
use Core\Domain\Booking\Models\Booking;

final readonly class BookingCreate
{
    public function __invoke(BookingData $bookingData): Booking
    {
        // body owned by the labrodev-action skill
    }
}
```

For everything inside the Action body (transactions, UUID assignment, Rule guards, domain-exception guards): → see the labrodev-action skill.
For controller anatomy, routing, and the toast/redirect idiom: → see the labrodev-controller skill.

## Typed value objects and local variables

The mirror rule applies everywhere a concrete named type appears — parameters, closure parameters, and local variables:

```php
// Parameters
public function __invoke(BookingEnvelope $bookingEnvelope): void {}   // not $booking, not $b

// Local variables holding a *Data / envelope instance
$customerAddressData = $bookingEvaluationPayload->customerAddressData; // not $addr, not $addressData

// Two parameters of the same type — disambiguate with clear prefixes
usort(
    array: $bookingEnvelopes,
    callback: fn (BookingEnvelope $firstBookingEnvelope, BookingEnvelope $secondBookingEnvelope): int =>
        $firstBookingEnvelope->startTimestamp <=> $secondBookingEnvelope->startTimestamp,
);
```

The variable name reflects the **declared type**, not a nickname — regardless of whether the value comes from an argument, a payload property, or a factory.

## Edge cases

- **`__invoke()` exceptions**: `Job::handle()` and `Pipeline::handle()` keep their conventional named entry points; everything else invokable uses `__invoke()`.
- **Named-argument order**: alphabetical by default; if the project has an established local order for a call site family, follow it consistently.
- **Cross-domain collisions**: same concept in two domains stays scoped and prefixed — `BookingConfirmedEvent` in `Core/Domain/Booking/Events`, `PaymentConfirmedEvent` in `Core/Domain/Payment/Events`. Prefer explicit over short.
- **Ubiquitous language override**: if the business domain has an established term that conflicts with a pattern here, the domain term wins — document it and apply it everywhere.
- **Abbreviations**: allowed only when universally understood in the codebase (`Pdf`, `Crm`, `Uuid`); never invent ad-hoc shortenings.
- **snake_case boundary**: only Model attributes and Data class attributes are snake_case; everything else (locals, non-persistence properties, parameters) is camelCase — including mirror-named Data variables (`$bookingData` holds an object whose own attributes are snake_case).
- **Multi-role Services**: when a Service genuinely has several operations, use explicit verbal method names (`send()`, `sendBatch()`) instead of forcing `__invoke()` — but still invoke multi-argument methods with named arguments.
- **Legacy zone**: vendor/starter code under `app/Http`, `app/Models`, `app/Actions/Fortify` is exempt → `labrodev-core` guideline.
