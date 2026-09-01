---
name: labrodev-exception
description: "Use when creating or reviewing any custom Exception class in a Labrodev Laravel project — deciding which folder an exception belongs in (Domain, Shared, Infrastructure, Feature, or a delivery Layer/App), naming it, or wiring the `::make()` static-constructor throw contract."
license: MIT
metadata:
  author: labrodev
---

# Exceptions: named unhappy paths, placed by origin

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-exception` guideline** (musts, must-nots); the per-file checklist is `rules/exceptions.md`. This skill holds the craft: anatomy, canonical templates, and edge cases.

An Exception is a **named, explicit unhappy path** — not a generic `throw new \Exception('...')`. Every thrown exception in a Labrodev project is its own `final` class, placed by where the failure *originates*, and constructed through a single static `make()` factory so the throw site reads as a sentence.

Placement law (origin, not consumer — the Domain/Shared/Infrastructure/Feature/Layer table) → `labrodev-exception` guideline.

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
- **Surfacing a domain exception to the user**: never by converting it into `ValidationException` — that class is the Data validation layer's alone (→ labrodev-data). The exception stays a plain carrier of context; the delivery layer decides its user-facing shape (redirect + error toast, JSON error, status code) in Laravel's exception handler (`bootstrap/app.php` `->withExceptions(...)` or a `render()` on an App-layer exception).
- **HTTP rendering**: an `App/Exceptions` exception may implement Laravel's `render()`/`report()` conventions or be mapped in the exception handler — that wiring is delivery-surface concern, not a reason to move the class out of `App/Exceptions`.
- **Reused across Layers**: if two Layers need the same exception, it is not Layer-specific — move it to the Core module (Domain/Shared/Feature) that actually detects the failure.
- **Legacy zone**: vendor/starter code under `app/Http`, `app/Models`, `app/Actions/Fortify` is exempt → see the labrodev-core skill.
