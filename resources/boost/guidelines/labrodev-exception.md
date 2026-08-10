# Labrodev Exceptions

Always-on law. An exception is a **named, explicit unhappy path** — its own class, placed by where the failure originates, thrown through a static factory so the throw site reads as a sentence. Anatomy and templates → the `labrodev-exception` skill; per-file checks → `rules/exceptions.md`.

## Musts

- **Every exception is its own `final` class** — never a bare `\Exception`, `\RuntimeException`, or `\InvalidArgumentException` thrown directly.
- **One public entry point**: a single static named constructor, `make()`, with `__construct()` `private` (or `protected` only when the exception is deliberately designed to be extended). Nothing else public besides inherited SPL-exception methods.
- **`make()` builds the message internally** from the typed context it accepts (a model, an id, a reason string), passed as named arguments when there is more than one (→ labrodev-naming). Callers never build the message string.
- **Throw sites read as a sentence**: `throw BookingOverlapException::make(booking: $booking);`
- **Extend the SPL base matching the failure's nature**: `\RuntimeException` for failures only known at runtime (external call failed, state conflict); `\LogicException` (or `\InvalidArgumentException`/`\OutOfRangeException`) for programmer-caused contract violations; default `\RuntimeException` when unsure.
- **Naming**: `{DescriptiveCondition}Exception` — `BookingOverlapException`, `PaymentDeclinedException`, `InvalidWebhookSignatureException` (→ labrodev-naming).
- **Placement follows origin, not consumer** — an exception lives where the failure condition is detected, not wherever it happens to be caught.

## Placement

| Failure origin | Home | Example |
|---|---|---|
| A single business domain's invariant | `Core/Domain/{Domain}/Exceptions` | `BookingOverlapException` |
| Generic, domain-free technical condition reused across domains | `Core/Shared/Exceptions` | `InvalidUuidException` |
| An external system/integration adapter | `Core/Infrastructure/{Integration}/Exceptions` | `PaddleChargeFailedException` |
| A cross-domain workflow living in `Core/Feature` | `Core/Feature/{Feature}/Exceptions` | `RegistrationAlreadyCompletedException` |
| A single delivery surface only (HTTP boundary, request shape, routing) | `App/Layer/{Layer}/{Domain}/Exceptions` — or `App/Exceptions` when it applies to every Layer | `UnsupportedApiVersionException` |

If a failure could originate in more than one Domain, it is not domain-specific — either it is generic (`Core/Shared`) or it belongs to the `Core/Feature` slice that coordinates those domains.

## Must-nots

- No `throw new \Exception('some string')` anywhere in Core or App/Layer — every throw is a named class via `::make()`.
- No public `__construct()` on an exception class — callers go through `make()`, never `new BookingOverlapException(...)`.
- No business decision logic inside the exception class — it formats a message and carries context; *whether* to throw is decided by the calling Action/Service/Rule (→ labrodev-action).
- No domain-named exception in `Core/Shared/Exceptions` — a class name containing a domain noun (`Booking`, `Invoice`) never belongs in Shared (→ labrodev-core).
- No `Core/Infrastructure` exception leaking a vendor SDK's own exception type past the adapter boundary — wrap it in a Labrodev exception before it crosses into the Domain.
- No catching-and-swallowing without a stated reason — a caught exception is re-thrown, translated into a `ValidationException` at the Action boundary (→ labrodev-action), or logged with explicit justification; never a silent empty `catch {}`.
