---
name: labrodev-exception
description: "Use when creating or reviewing any custom Exception class in a Labrodev Laravel project — deciding which folder an exception belongs in (Domain, Shared, Infrastructure, Feature, or a delivery Layer/App), naming it, or wiring the `::make()` static-constructor throw contract."
license: MIT
metadata:
  author: labrodev
---

# Exceptions: named unhappy paths, placed by origin

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

An Exception is a **named, explicit unhappy path** — not a generic `throw new \Exception('...')`. Every thrown exception in a Labrodev project is its own `final` class, placed by where the failure *originates*, and constructed through a single static `make()` factory so the throw site reads as a sentence.

## Musts

- Every exception is its own `final` class — never a bare `\Exception`, `\RuntimeException`, or `\InvalidArgumentException` thrown directly.
- Every exception class exposes exactly one static named constructor, `make()`, and nothing else public besides inherited SPL-exception methods. The `__construct()` is `private` (or `protected` if the exception is deliberately designed to be extended).
- `make()` accepts whatever typed context the message needs (a model, an id, a reason string) as **named arguments** when there is more than one → see the labrodev-naming skill, and builds the message internally. Callers never build the message string themselves.
- Throw sites read as a sentence: `throw BookingOverlapException::make(booking: $booking);`
- Extend the SPL base that matches the failure's nature: `\RuntimeException` for failures only known at runtime (external call failed, state conflict), `\LogicException` (or `\InvalidArgumentException`/`\OutOfRangeException`) for programmer-caused contract violations. Default to `\RuntimeException` when unsure.
- Naming follows the standing rule: `{DescriptiveCondition}Exception` — `BookingOverlapException`, `PaymentDeclinedException`, `InvalidWebhookSignatureException` → see the labrodev-naming skill.
- Placement follows **origin, not consumer** — see the placement table below. An exception lives where the failure condition is detected, not wherever it happens to be caught.

## Must-nots

- No `throw new \Exception('some string')` anywhere in Core or App/Layer — every throw is a named class via `::make()`.
- No public `__construct()` on an exception class — callers must go through `make()`, never `new BookingOverlapException(...)`.
- No business decision logic inside the exception class itself — an exception formats a message and carries context; it does not decide *whether* to throw (that belongs to the calling Action/Service/Rule) → see the labrodev-action skill.
- No domain-named exception in `Core/Shared/Exceptions` — a class name containing a domain noun (`Booking`, `Invoice`) never belongs in Shared → see the labrodev-core skill.
- No `Core/Infrastructure` exception leaking a vendor SDK's own exception type past the adapter boundary — wrap it in a Labrodev exception before it crosses into the Domain.
- No catching-and-swallowing without a stated reason; a caught exception is either re-thrown, translated into a `ValidationException` at the Action boundary (→ see the labrodev-action skill), or logged with explicit justification — never a silent empty `catch {}`.

## Placement decision

| Failure origin | Home | Example |
|---|---|---|
| A single business domain's invariant | `Core/Domain/{Domain}/Exceptions` | `Core/Domain/Booking/Exceptions/BookingOverlapException.php` |
| Generic, domain-free technical condition reused across domains | `Core/Shared/Exceptions` | `Core/Shared/Exceptions/InvalidUuidException.php` |
| An external system/integration adapter | `Core/Infrastructure/{Integration}/Exceptions` | `Core/Infrastructure/Paddle/Exceptions/PaddleChargeFailedException.php` |
| A cross-domain workflow living in `Core/Feature` | `Core/Feature/{Feature}/Exceptions` | `Core/Feature/BookingRegistration/Exceptions/RegistrationAlreadyCompletedException.php` |
| A single delivery surface only (HTTP boundary, request shape, routing) | `App/Layer/{Layer}/{Domain}/Exceptions`, or `App/Exceptions` when it applies to every Layer | `App/Layer/Api/Booking/Exceptions/UnsupportedApiVersionException.php`, `App/Exceptions/UnauthenticatedRequestException.php` |

If a failure could originate in more than one Domain, it is not domain-specific — either it is generic (`Core/Shared`) or it belongs to the `Core/Feature` slice that coordinates those domains.

## Template: Domain exception

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Exceptions;

use Core\Domain\Booking\Models\Booking;
use RuntimeException;

final class BookingOverlapException extends RuntimeException
{
    private function __construct(string $message)
    {
        parent::__construct($message);
    }

    public static function make(Booking $booking): self
    {
        return new self(
            sprintf('Booking %s overlaps with an existing reservation.', $booking->uuid),
        );
    }
}
```

Throw site, inside a Domain Action or Service:

```php
if (! BookingRule::canConfirm(booking: $booking)) {
    throw BookingOverlapException::make(booking: $booking);
}
```

## Template: Infrastructure exception (wrapping a vendor failure)

```php
<?php

declare(strict_types=1);

namespace Core\Infrastructure\Paddle\Exceptions;

use RuntimeException;
use Throwable;

final class PaddleChargeFailedException extends RuntimeException
{
    private function __construct(string $message, ?Throwable $previous = null)
    {
        parent::__construct($message, previous: $previous);
    }

    public static function make(string $reason, ?Throwable $previous = null): self
    {
        return new self(
            message: sprintf('Paddle charge failed: %s', $reason),
            previous: $previous,
        );
    }
}
```

The adapter catches the vendor SDK's exception and re-throws this one — the Domain never sees a vendor exception type → see the labrodev-infrastructure skill.

## Template: Shared (generic) exception

```php
<?php

declare(strict_types=1);

namespace Core\Shared\Exceptions;

final class InvalidUuidException extends \InvalidArgumentException
{
    private function __construct(string $message)
    {
        parent::__construct($message);
    }

    public static function make(string $value): self
    {
        return new self(sprintf('"%s" is not a valid UUID.', $value));
    }
}
```

## Edge cases

- **Multiple contextual arguments**: `make()` follows the same named-argument rule as any multi-argument call — alphabetical order unless a stronger local convention exists → see the labrodev-naming skill.
- **Wrapping an underlying exception**: accept `?Throwable $previous` in `make()` and pass it to `parent::__construct(..., previous: $previous)` so the original stack trace is preserved — do this whenever an Infrastructure or Shared exception wraps a caught SDK/library exception.
- **Turning an exception into a user-facing validation message**: an Action may catch a Domain exception and re-throw it as `ValidationException::withMessages(...)` at the boundary — the two failure modes are distinct and owned by the labrodev-action skill; the exception class itself stays a plain carrier of context.
- **HTTP rendering**: an `App/Exceptions` exception may implement Laravel's `render()`/`report()` conventions or be mapped in the exception handler — that wiring is delivery-surface concern, not a reason to move the class out of `App/Exceptions`.
- **Reused across Layers**: if two Layers need the same exception, it is not Layer-specific — move it to the Core module (Domain/Shared/Feature) that actually detects the failure.
- **Legacy zone**: vendor/starter code under `app/Http`, `app/Models`, `app/Actions/Fortify` is exempt → see the labrodev-core skill.

## Review checklist

1. Is the exception its own `final` class with a `private` (or deliberately `protected`) constructor and a single static `make()` factory?
2. Does every throw site call `::make()` with named arguments (when more than one) instead of `new SomeException(...)` or a bare SPL exception?
3. Is it placed by **origin** — Domain, Shared (domain-noun-free), Infrastructure `{Integration}`, Feature `{Feature}`, or Layer/App — per the placement table?
4. Does the name follow `{DescriptiveCondition}Exception`?
5. Do Infrastructure exceptions wrap vendor SDK exceptions (with `previous`) before they cross into the Domain?
6. Is the exception free of business decision logic — it formats and carries context; the caller decides whether to throw?
7. Are there no silent empty `catch {}` blocks?
